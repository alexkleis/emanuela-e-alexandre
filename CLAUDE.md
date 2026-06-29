# Convite de Casamento — Emanuela & Alexandre

Convite digital (página única) para o casamento de **Emanuela & Alexandre**, em
**Curitiba/PR, 24/10/2026**. Bilíngue (PT padrão / EN), personalizado por convidado
via link, com confirmação de presença gravada numa planilha do Google.

## Stack & filosofia

- **HTML único, estático, sem build.** Tudo vive em `index.html`: Tailwind via CDN,
  Google Fonts (Fraunces, Cormorant Garamond, Great Vibes), Font Awesome. Não há
  npm, bundler, nem framework. Editar = abrir o arquivo.
- Hospedado no **GitHub Pages**: repo `alexkleis/emanuela-e-alexandre`,
  no ar em https://alexkleis.github.io/emanuela-e-alexandre/
- Deploy = `git push` na branch principal (propaga em ~1 min).

## Arquivos

- **`index.html`** — o convite. É o arquivo que quase sempre se edita.
- **`gerar-links.html`** — backoffice (SÓ para os noivos, não enviar a convidados):
  gera o link personalizado de cada convidado (nome, adultos, crianças, idioma,
  encurtar via is.gd).
- **`PUERTA.png`** — painéis da animação de abertura (portas que se abrem, efeito VibeFly).
- **`musica.mp3`** — música de fundo ("All of Me").
- **`*.JPG/*.jpg/*.png`** — fotos do casal (capa, história, locais, carrossel, rodapé).
  Sempre usar **JPG/PNG** — HEIC não abre no navegador (converter com
  `sips -s format jpeg in.HEIC --out out.jpg`).

## Como o `index.html` está organizado

- Todo o conteúdo é dirigido pelo objeto **`CONFIG`** (≈ linha 820 do `<script>`):
  nomes, data, locais, pais, programação, traje, hashtag, PIX, `googleSheetUrl`.
  Mudar texto/dados = mexer no CONFIG, não no HTML das seções.
- **i18n:** `LANG` vem de `?lang=en`; dicionário `TR` (PT→EN) + `translatePage()`
  troca os textos. Padrão é PT. Ao adicionar texto novo em PT, adicionar a
  tradução correspondente em `TR`.
- **Personalização por convidado:** query params
  `?convidado=Nome&adultos=X&criancas=Y&lang=en`. Lidos no objeto `GUEST`. Mostram
  o nome na abertura e pré-preenchem os contadores do RSVP; o campo "nome" do RSVP
  fica escondido quando o link já traz o nome.

## RSVP (confirmação) — importante

- Botão único **"Confirmar presença"** → grava na planilha via Apps Script
  (form POST oculto para um `<iframe name="rsvp-sink">`, evita CORS).
- **Fonte da verdade é a planilha**, não o navegador. Ao reabrir o link, o convite
  consulta a planilha por **JSONP** (`?action=check&nome=...&callback=__rsvpCheck`)
  para saber se aquele convidado já confirmou e mostrar "Você já confirmou! / Editar".
  `localStorage` é só atalho instantâneo (não persiste no navegador interno do
  WhatsApp / Safari anônimo — por isso o check remoto existe).
- O **Apps Script precisa ter `doPost` E `doGet`** (o `doGet` é o que responde o
  check). O código pronto das duas funções está no comentário no fim do `index.html`.
- ⚠️ **Toda vez que o Apps Script muda, é preciso uma NOVA implantação** (ou
  "Gerenciar implantações → editar → Versão: Nova versão"). Acesso: "Qualquer pessoa".
  Só salvar o script NÃO publica a mudança.

## Convenções de trabalho neste projeto

- **Verificar no navegador, não só no código.** Para espaçamento/layout, usar o
  Chrome (superpowers-chrome): viewport mobile 390×844, abrir o `index.html` local,
  forçar `#main` visível e os `.observe-me` (reveal on scroll começa em opacity:0),
  e usar **screenshot `fullpage:true`** (screenshot de viewport rolado volta em branco).
  Imagens `loading=lazy` às vezes não carregam no headless — forçar `eager` quando precisar ver fotos.
- **Mobile-first.** Ritmo vertical responsivo: seções `py-7 md:py-20`, títulos
  com margem menor no celular e maior no desktop (sufixo `md:`). Manter consistente
  entre todas as seções.
- **Validar o JS** após editar o `<script>`:
  `awk '/<script>/{f=1;next} /<\/script>/{f=0} f' index.html > /tmp/j.js && node --check /tmp/j.js`
- **Commits** em PT, curtos. Commitar e dar push só quando a mudança está pronta;
  avisar o usuário para atualizar com refresh forçado / aba anônima (cache do Pages).
- Idioma de conversa com o usuário: **português**.
