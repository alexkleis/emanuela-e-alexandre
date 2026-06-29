# Design — RSVP robusto (confirmação de presença blindada)

**Data:** 2026-06-29
**Projeto:** Convite Emanuela & Alexandre (`index.html`, GitHub Pages)
**Decisão de caminho:** manter Google (Planilha + Apps Script), porém blindado.
Nada de novo serviço/conta. Ver [[CLAUDE.md]] e [[reference-rsvp-backend]].

## Objetivo

Tornar a confirmação de presença confiável de ponta a ponta, atendendo aos quatro
pontos que o usuário priorizou:

1. **Não perder confirmação** — só mostrar "confirmado" quando a gravação foi
   verificada de verdade; reenviar se falhar; nunca mostrar falso sucesso.
2. **Avisar na hora** — e-mail automático (canal confiável) + WhatsApp (best-effort)
   a cada confirmação.
3. **Planilha limpa** — uma linha por convidado; editar atualiza a linha (sem duplicar).
4. (Decidido não migrar para fora do Google agora — Apps Script é suficiente na
   escala de um casamento; "sair do Google" custaria setup sem ganho prático real.)

## Problemas do sistema atual

- `submit()` faz POST para um `<iframe>` oculto e **nunca lê a resposta** → se a
  planilha falhar, a UI mostra "confirmado" mesmo sem ter gravado (falha silenciosa).
- Sem reenvio: falha de rede no momento = confirmação perdida.
- Cada edição cria uma **linha nova** (o check pega a última, mas a planilha acumula).
- Sem aviso aos noivos (precisam abrir a planilha).

## Arquitetura

Dois componentes, contrato simples entre eles:

### A) Apps Script (Web App, acesso "Qualquer pessoa")

- **`doPost(e)`** — grava/atualiza e notifica:
  1. **Upsert por `id`**: se já existe linha com aquele `id` (coluna J), atualiza-a;
     senão, cria nova. (Se um link antigo vier sem `id`, cai no nome como fallback.)
  2. Grava também `Atualizado em` (agora) e o `Token` recebido (ver fluxo).
  3. **E-mail** via `MailApp.sendEmail` para os dois noivos (canal confiável).
  4. **WhatsApp** via `UrlFetchApp.fetch` ao CallMeBot, para cada número
     cadastrado — **dentro de try/catch**: qualquer erro do WhatsApp é ignorado e
     **não** afeta a gravação nem o e-mail.
  5. Retorna `OK` (texto simples).
- **`doGet(e)`** — consulta (JSONP, já existe), agora **por `id`** (`?action=check&id=...`),
  e devolve também o `Token` e `atualizadoEm` da linha, para a verificação ponta a ponta.

Config no topo do script (preenchida pelo usuário, **fora do repositório público**):
```
var NOIVOS_EMAIL = ['alexkleis@gmail.com', 'emanuelacsantos@gmail.com'];
var WHATS = [
  { phone: '5541999410707', apikey: 'XXXXXX' },   // CallMeBot do Alexandre
  { phone: '5541xxxxxxxxx', apikey: 'YYYYYY' }     // CallMeBot da Emanuela
];
```

### B) Convite (módulo `RSVP` no `index.html`)

- Lê o **`id` único do convidado** do link (`?id=...`), além de `convidado`,
  `adultos`, `criancas`, `lang`. O `id` é a chave de gravação/consulta.
- `submit()` vira **envio verificado com reenvio** (ver fluxo) e inclui o `id`.
- `restore()` (já consulta a planilha) passa a consultar **por `id`**; também
  restaura confirmações **pendentes** salvas localmente e tenta reenviá-las.
- Novo estado de UI: "enviando…", "não consegui confirmar — tentar de novo"
  (com botão), além do "Presença confirmada!" e "Você já confirmou!".

### C) Gerador de links (`gerar-links.html`)

- Ao adicionar um convidado, gera um **`id` único e estável** (ex.: 8 caracteres
  aleatórios `a–z0–9`) e o inclui no link: `...&id=<id>`.
- O `id` acompanha o convidado em todos os botões (copiar/WhatsApp/encurtar).

## Modelo de dados (colunas da planilha)

`Data(A) | Nome(B) | Presença(C) | Adultos(D) | Crianças(E) | Mensagem(F) | Evento(G) | Atualizado em(H) | Token(I) | Id(J)`

- `Data` = primeira confirmação (não muda em edições).
- `Atualizado em` = última edição.
- `Token` = código único da última confirmação (usado para verificar a gravação).
- `Id` = **chave do convidado** (vem do link, gerado pelo backoffice). É por ele que
  o `doPost` faz upsert e o `doGet` consulta.

## Fluxo de dados — gravação verificada

1. Convidado toca **Confirmar**. O cliente gera `token` aleatório e monta o payload
   (**id**, nome, presença, adultos, crianças, mensagem, evento, **token**).
2. Cliente **envia** (POST). Mostra "Enviando…".
3. Cliente **confere** chamando `doGet` (JSONP) **pelo `id`**: a linha existe e o
   `Token` retornado **é igual** ao `token` enviado?
4. **Sim** → mostra "Presença confirmada!", salva no `localStorage`, fim.
5. **Não** → espera curta e tenta (passos 2–3) até **3 vezes** (backoff ~1.5s, 3s).
6. Ainda falhou → salva a confirmação como **pendente** no `localStorage` e mostra
   *"Não consegui confirmar agora. Toque para tentar de novo"* (botão de retry).
   Também tenta reenviar automaticamente no próximo carregamento da página.
   **Nunca** mostra sucesso sem o token bater.

O `token` torna a verificação inequívoca: prova que **aquele** envio chegou (não uma
confirmação antiga).

## Notificações

- **E-mail (confiável):** assunto "Nova confirmação — <Nome>", corpo com presença,
  adultos, crianças e recado. Cota do Gmail comum ≈ 100/dia — folgado.
- **WhatsApp (best-effort):** CallMeBot por número cadastrado. Setup único por
  pessoa: enviar a mensagem de ativação ao número do CallMeBot e receber a `apikey`.
  Falha do CallMeBot é silenciosa (não quebra nada). É o "ping na hora"; o e-mail e a
  linha na planilha são a garantia.

## Tratamento de erros

| Situação | Comportamento |
|---|---|
| Sem internet / script fora do ar no envio | Reenvia 3×; se falhar, salva pendente + botão "tentar de novo"; retenta ao reabrir. Sem falso sucesso. |
| WhatsApp (CallMeBot) falha | Ignorado (try/catch). Gravação e e-mail seguem normais. |
| E-mail falha | Logado no Apps Script; não bloqueia a gravação. |
| Convidado edita resposta | `doPost` atualiza a mesma linha; novo `token`; `doGet` confirma. |
| Reabrir o link (outro aparelho/navegador) | `restore()` consulta a planilha **pelo `id`** e mostra "Você já confirmou!". |

## Limitações conhecidas (honestas)

- **Chave = `id` único do link.** O `id` é gerado pelo backoffice e é a identidade do
  convidado — nomes iguais não colidem mais. Um link precisa **ter** o `id` para o
  upsert funcionar; links gerados de agora em diante sempre terão. (Fallback: link
  sem `id` cai no nome, só por segurança.)
- **CallMeBot é serviço comunitário gratuito** (não oficial). Por isso é best-effort,
  com o e-mail como canal garantido.
- **Sempre re-implantar** o Apps Script após editar (Nova implantação / Nova versão),
  senão as mudanças não entram no ar.

## Plano de testes (no navegador, conforme [[feedback-verify-in-browser]])

1. **Sucesso:** confirmar via link → "Presença confirmada!" só aparece após o token
   bater; linha criada na planilha; e-mail chega; WhatsApp chega (se cadastrado).
2. **Falha de rede:** apontar `googleSheetUrl` para URL inválida (ou simular) →
   UI mostra "tentar de novo", NÃO mostra sucesso; pendente salvo; retry funciona.
3. **Edição:** editar e reconfirmar → **mesma** linha atualizada (sem duplicar);
   `Atualizado em` muda; `doGet` confirma o novo token.
4. **Reabrir:** confirmar, limpar localStorage, reabrir o mesmo link → "Você já
   confirmou!" (veio da planilha).
5. **WhatsApp fora do ar:** com apikey inválida → gravação e e-mail seguem normais.

## Passos operacionais para o usuário (uma vez)

1. Acrescentar os cabeçalhos `Atualizado em` (H), `Token` (I) e `Id` (J) na 1ª
   linha da planilha. O `doPost` escreve nessas posições fixas.
2. Colar o `doPost`+`doGet` novos no Apps Script e preencher e-mails/WhatsApp.
3. Ativar o CallMeBot (cada noivo, uma vez) e colar as apikeys.
4. **Nova implantação** do Web App (acesso "Qualquer pessoa").
5. Eu valido tudo no navegador.

## Fora de escopo (YAGNI)

- Migração para Supabase/Firebase.
- Painel/dashboard próprio (a planilha já serve).
- Lembretes automáticos para quem não respondeu.
