# RSVP Robusto + Domínio manueale.com — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Tornar a confirmação de presença confiável de ponta a ponta (gravação verificada, reenvio, sem duplicatas, aviso por e-mail + WhatsApp) e servir o convite pelo domínio próprio manueale.com.

**Architecture:** Mantém GitHub Pages + Planilha Google + Apps Script. O Apps Script passa a fazer upsert por `id` único do convidado, notificar (e-mail confiável + WhatsApp best-effort) e devolver um `token` na consulta. O convite só mostra "confirmado" depois de reler a planilha e bater o `token`; se falhar, reenvia e, em último caso, guarda como pendente. O gerador de links cria um `id` por convidado. Domínio via CNAME + DNS no registrador.

**Tech Stack:** HTML/JS estático (sem build), Tailwind CDN, Google Apps Script (Web App), GitHub Pages, CallMeBot (WhatsApp), DNS do registrador do domínio.

## Global Constraints

- **Sem build / arquivo único:** toda lógica do convite vive em `index.html`; backoffice em `gerar-links.html`. Sem npm/bundler.
- **Validar JS após editar o `<script>`:** `awk '/<script>/{f=1;next} /<\/script>/{f=0} f' index.html > /tmp/j.js && node --check /tmp/j.js` → deve imprimir nada (sucesso).
- **Verificar no navegador (mobile 390×844)** com a ferramenta Chrome, conforme convenção do projeto; usar `eval` (getBoundingClientRect / estado do DOM) para medir, screenshot `fullpage:true` quando precisar ver.
- **Deploy:** `git -c user.name="alexkleis" -c user.email="alexkleis@gmail.com" commit` + `git push`. GitHub Pages propaga em ~1 min.
- **Apps Script:** toda mudança exige **Nova implantação / Nova versão** (acesso "Qualquer pessoa"), senão não entra no ar.
- **Chave do convidado = `id`** vindo do link (`?id=...`); fallback por nome só para links antigos.
- **WhatsApp é best-effort** (try/catch); nunca pode bloquear a gravação nem o e-mail.
- **Não mostrar falso sucesso:** "Presença confirmada!" só após o `token` bater na releitura.
- **Domínio principal:** `manueale.com` (apex); `www` redireciona. Trocar URLs base só depois do DNS resolver; manter github.io funcionando até lá.
- Spec de referência: `docs/superpowers/specs/2026-06-29-rsvp-robusto-design.md`.

---

## File Structure

- `index.html` — convite. Modifica: objeto `GUEST` (ler `id`), módulo `RSVP` (`submit` verificado, `remoteCheck` por id+token, pendentes/retry, estados de UI), e o comentário do Apps Script no fim do arquivo. Modifica meta `og:image` (Task 5, após DNS).
- `gerar-links.html` — backoffice. Modifica: geração de `id` por convidado, inclusão de `&id=` no link, base preenchida → manueale.com (Task 5, após DNS).
- `CNAME` — **novo** arquivo na raiz do repo: conteúdo `manueale.com`.
- `Codigo.gs` (no Apps Script do usuário, fora do repo) — substituído pelo script novo; também colado no comentário ao fim de `index.html` para referência/versionamento.

---

## Task 1: Domínio manueale.com no GitHub Pages

**Files:**
- Create: `CNAME` (raiz do repo, conteúdo exato: `manueale.com`)

**Interfaces:**
- Produces: o site passa a responder em `https://manueale.com` (e `https://www.manueale.com` redireciona). Outras tasks que trocam URL base dependem disso estar no ar (gate na Task 5).

- [ ] **Step 1: Criar o arquivo CNAME**

Conteúdo do arquivo `CNAME` (uma linha, sem `http`, sem barra):
```
manueale.com
```

- [ ] **Step 2: Commit e push do CNAME**

```bash
git add CNAME
git -c user.name="alexkleis" -c user.email="alexkleis@gmail.com" commit -m "Adiciona dominio personalizado manueale.com (CNAME)"
git push
```

- [ ] **Step 3: (USUÁRIO) Configurar o DNS no registrador do domínio**

No painel de DNS de manueale.com, criar:
- 4 registros **A** para o apex (`@` / manueale.com), apontando para os IPs do GitHub Pages:
  - `185.199.108.153`
  - `185.199.109.153`
  - `185.199.110.153`
  - `185.199.111.153`
- (opcional, recomendado) 4 registros **AAAA** (IPv6): `2606:50c0:8000::153`, `2606:50c0:8001::153`, `2606:50c0:8002::153`, `2606:50c0:8003::153`
- 1 registro **CNAME** para `www` → `alexkleis.github.io.`

- [ ] **Step 4: (USUÁRIO) Ativar o domínio no GitHub**

Repo `alexkleis/emanuela-e-alexandre` → **Settings → Pages → Custom domain** → digitar `manueale.com` → Save. Aguardar o check do GitHub e marcar **Enforce HTTPS** (pode levar de minutos a ~24h para o certificado).

- [ ] **Step 5: Verificar a propagação**

Run (repetir até resolver; DNS pode levar minutos/horas):
```bash
dig +short manueale.com A
curl -sI https://manueale.com | head -n 1
```
Expected: `dig` lista os 4 IPs `185.199.10x.153`; `curl` retorna `HTTP/2 200` (ou um 301 de http→https). Enquanto não resolver, seguir usando `https://alexkleis.github.io/emanuela-e-alexandre/`.

---

## Task 2: Apps Script v2 — upsert por id, notificações, token

**Files:**
- Modify: `index.html` (bloco de comentário ao fim, ~linha 1511 em diante) — substituir o código de exemplo do Apps Script pelo script novo (referência versionada).
- (USUÁRIO) cola o mesmo script no editor do Apps Script da planilha e re-implanta.

**Interfaces:**
- Produces:
  - `doPost(e)` — grava/atualiza por `id` (params: `id, nome, presenca, adultos, criancas, mensagem, evento, token`), envia e-mail aos dois noivos, dispara WhatsApp best-effort; retorna texto `OK`.
  - `doGet(e)` — `?action=check&id=<id>&callback=<cb>` retorna JSONP `cb({confirmed, presenca, adultos, criancas, mensagem, token, atualizadoEm})` ou `cb({confirmed:false})`.
- Consumes: planilha com cabeçalhos `Data(A) | Nome(B) | Presença(C) | Adultos(D) | Crianças(E) | Mensagem(F) | Evento(G) | Atualizado em(H) | Token(I) | Id(J)`.

- [ ] **Step 1: (USUÁRIO) Acrescentar cabeçalhos na planilha**

Na 1ª linha da planilha, garantir as colunas H, I, J com os títulos: `Atualizado em` (H), `Token` (I), `Id` (J).

- [ ] **Step 2: Escrever o script novo no comentário de `index.html`**

Substituir o corpo `function doPost(e){...}` (e o `doGet` atual) dentro do comentário ao fim de `index.html` por:

```javascript
// ====== CONFIG (preencher) ======
var NOIVOS_EMAIL = ['alexkleis@gmail.com', 'emanuelacsantos@gmail.com'];
var WHATS = [
  { phone: '5541999410707', apikey: 'COLOQUE_A_APIKEY_DO_ALEXANDRE' },
  { phone: '5541XXXXXXXXX', apikey: 'COLOQUE_A_APIKEY_DA_EMANUELA' }
];
// Colunas 1-based: A..J
var COL = { data:1, nome:2, presenca:3, adultos:4, criancas:5, mensagem:6, evento:7, atualizado:8, token:9, id:10 };

function sheet_() { return SpreadsheetApp.getActiveSpreadsheet().getSheets()[0]; }

function findRowById_(sh, id) {
  if (!id) return -1;
  var data = sh.getDataRange().getValues();
  for (var i = 1; i < data.length; i++) {
    if (String(data[i][COL.id - 1]).trim() === String(id).trim()) return i + 1;
  }
  return -1;
}
function findRowByName_(sh, nome) {
  if (!nome) return -1;
  var data = sh.getDataRange().getValues();
  var alvo = String(nome).trim().toLowerCase();
  for (var i = data.length - 1; i >= 1; i--) {
    if (String(data[i][COL.nome - 1]).trim().toLowerCase() === alvo) return i + 1;
  }
  return -1;
}

function doPost(e) {
  var p = (e && e.parameter) || {};
  var sh = sheet_();
  var agora = new Date();
  var row = findRowById_(sh, p.id);
  if (row < 0) row = findRowByName_(sh, p.nome); // fallback p/ links antigos sem id
  if (row < 0) {
    sh.appendRow([agora, p.nome || '', p.presenca || '', p.adultos || '', p.criancas || '',
      p.mensagem || '', p.evento || '', agora, p.token || '', p.id || '']);
  } else {
    sh.getRange(row, COL.nome).setValue(p.nome || '');
    sh.getRange(row, COL.presenca).setValue(p.presenca || '');
    sh.getRange(row, COL.adultos).setValue(p.adultos || '');
    sh.getRange(row, COL.criancas).setValue(p.criancas || '');
    sh.getRange(row, COL.mensagem).setValue(p.mensagem || '');
    sh.getRange(row, COL.evento).setValue(p.evento || '');
    sh.getRange(row, COL.atualizado).setValue(agora);
    sh.getRange(row, COL.token).setValue(p.token || '');
    if (p.id) sh.getRange(row, COL.id).setValue(p.id);
  }
  notificar_(p);
  return ContentService.createTextOutput('OK');
}

function notificar_(p) {
  var resumo = 'Nome: ' + (p.nome || '') + '\n'
    + 'Presença: ' + (p.presenca || '') + '\n'
    + 'Adultos: ' + (p.adultos || '0') + '\n'
    + 'Crianças: ' + (p.criancas || '0') + '\n'
    + 'Mensagem: ' + (p.mensagem || '(sem recado)');
  try {
    MailApp.sendEmail({
      to: NOIVOS_EMAIL.join(','),
      subject: 'RSVP ' + (p.presenca === 'Sim' ? '[SIM]' : '[NAO]') + ' — ' + (p.nome || ''),
      body: resumo
    });
  } catch (err) { Logger.log('email falhou: ' + err); }
  for (var i = 0; i < WHATS.length; i++) {
    try {
      var w = WHATS[i];
      if (!w.apikey || w.apikey.indexOf('COLOQUE') === 0) continue;
      var url = 'https://api.callmebot.com/whatsapp.php?phone=' + encodeURIComponent(w.phone)
        + '&text=' + encodeURIComponent('Nova confirmacao:\n' + resumo)
        + '&apikey=' + encodeURIComponent(w.apikey);
      UrlFetchApp.fetch(url, { muteHttpExceptions: true });
    } catch (err2) { Logger.log('whats falhou: ' + err2); }
  }
}

function doGet(e) {
  var p = (e && e.parameter) || {};
  var cb = p.callback || 'callback';
  var out = { confirmed: false };
  if (p.action === 'check') {
    var sh = sheet_();
    var row = -1;
    if (p.id) row = findRowById_(sh, p.id);
    if (row < 0 && p.nome) row = findRowByName_(sh, p.nome);
    if (row > 0) {
      var v = sh.getRange(row, 1, 1, COL.id).getValues()[0];
      out = {
        confirmed: true,
        presenca: v[COL.presenca - 1], adultos: v[COL.adultos - 1],
        criancas: v[COL.criancas - 1], mensagem: v[COL.mensagem - 1],
        token: v[COL.token - 1], atualizadoEm: v[COL.atualizado - 1]
      };
    }
  }
  return ContentService.createTextOutput(cb + '(' + JSON.stringify(out) + ')')
    .setMimeType(ContentService.MimeType.JAVASCRIPT);
}
```

- [ ] **Step 3: Commit do comentário de referência**

```bash
git add index.html
git -c user.name="alexkleis" -c user.email="alexkleis@gmail.com" commit -m "Apps Script v2: upsert por id, e-mail+WhatsApp, token (referencia)"
git push
```

- [ ] **Step 4: (USUÁRIO) Colar no Apps Script, preencher apikeys e re-implantar**

Extensões → Apps Script → colar o script → preencher as 2 `apikey` do CallMeBot (Task 4 explica como obter) → **Implantar → Gerenciar implantações → editar → Versão: Nova versão** (acesso "Qualquer pessoa").

- [ ] **Step 5: Verificar o endpoint (GET de check)**

Run (substituir pela URL `/exec` em uso; usar um id que não existe ainda):
```bash
curl -sL --max-time 20 "https://script.google.com/macros/s/AKfycbzmSym9Cort4t5qtaJjKvmk6hK4AaVqWeGIrkkXT86u8BG7skJkQzggdK9p9oF2-X0vdg/exec?action=check&id=teste123&callback=cb" | head -c 200
```
Expected: `cb({"confirmed":false})` (e **não** mais a página de erro HTML "Script function not found: doGet"). Isso prova que a nova implantação está no ar.

---

## Task 3: Convite — ler `id`, envio verificado, reenvio e pendentes

**Files:**
- Modify: `index.html` — objeto `GUEST` (~linha 1026); módulo `RSVP` (~linhas 1335–1505): `submit`, `remoteCheck`, `restore`, novos helpers e estados de UI.
- Test: verificação no navegador (Chrome tool) + `node --check`.

**Interfaces:**
- Consumes: `doGet`/`doPost` da Task 2 (param `id`, retorno com `token`).
- Produces (funções internas do IIFE `RSVP`):
  - `makeToken()` → string única.
  - `remoteCheck(id, name)` → Promise resolvendo o objeto do `doGet` (ou `null`).
  - `trySendVerified(payload)` → Promise<boolean> (true só se o `token` bateu na releitura).
  - estados de UI: `showSending()`, `showRetry()`, além dos já existentes `showSuccess()`, `edit()`.

- [ ] **Step 1: Ler o `id` do link no objeto GUEST**

Em `index.html`, no objeto `GUEST` (~linha 1026), acrescentar o campo `id`:
```javascript
const GUEST = {
    name: (_params.get('convidado') || _params.get('c') || '').trim(),
    id: (_params.get('id') || '').trim(),
    adults: Math.max(1, _intParam(['adultos', 'a'], _intParam(['max', 'n'], 2))),
    children: _intParam(['criancas', 'k'], 0)
};
```

- [ ] **Step 2: Acrescentar o markup do estado "tentar de novo"**

Em `index.html`, dentro de `#rsvp-form`, logo após o botão `#rsvp-submit` e antes de `#rsvp-hint` (~linha 750), inserir um aviso de falha (começa oculto):
```html
<div id="rsvp-retry" style="display:none" class="mt-4 p-4 rounded text-center" style2="">
  <p class="text-sm mb-3" style="color:#b07a6a">Não consegui confirmar agora. Verifique a internet e tente de novo.</p>
  <button id="rsvp-retry-btn" onclick="RSVP.submit()" class="btn-refined">Tentar de novo</button>
</div>
```
(Observação: remover o atributo `style2` — é só para não duplicar `style`; usar uma classe. Markup final:)
```html
<div id="rsvp-retry" style="display:none" class="mt-4 p-4 rounded text-center">
  <p class="text-sm mb-3" style="color:#b07a6a">Não consegui confirmar agora. Verifique a internet e tente de novo.</p>
  <button id="rsvp-retry-btn" onclick="RSVP.submit()" class="btn-refined">Tentar de novo</button>
</div>
```

- [ ] **Step 3: Reescrever helpers + `submit` verificado no módulo RSVP**

Substituir, no IIFE `RSVP`, a função `submit()` atual e a função `remoteCheck`/`restore` (Task 2 do código anterior) pelo conjunto abaixo. `sendToSheet` (POST oculto) permanece como está, mas passa a receber também `id` e `token`.

```javascript
function makeToken() {
    return 'tk' + Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
}
function delay(ms) { return new Promise(function (r) { setTimeout(r, ms); }); }

// Consulta a planilha por id (com fallback por nome). Resolve o objeto do doGet ou null.
function remoteCheck(id, name) {
    return new Promise(function (resolve) {
        if (!CONFIG.googleSheetUrl || (!id && !name)) { resolve(null); return; }
        const cbName = '__rsvpCheck';
        let done = false;
        const s = document.createElement('script');
        function cleanup() { try { s.remove(); } catch (e) {} window[cbName] = undefined; }
        window[cbName] = function (data) { if (done) return; done = true; resolve(data); cleanup(); };
        s.onerror = function () { if (done) return; done = true; resolve(null); cleanup(); };
        let q = CONFIG.googleSheetUrl + '?action=check&callback=' + cbName;
        if (id) q += '&id=' + encodeURIComponent(id);
        if (name) q += '&nome=' + encodeURIComponent(name);
        s.src = q;
        document.body.appendChild(s);
        setTimeout(function () { if (done) return; done = true; resolve(null); cleanup(); }, 7000);
    });
}

// Envia e confirma relendo: true só se a planilha tiver o token enviado.
function trySendVerified(payload) {
    return (async function () {
        const waits = [1500, 3000, 4000];
        for (let i = 0; i < waits.length; i++) {
            sendToSheet(payload);
            await delay(waits[i]);
            const data = await remoteCheck(payload.id, payload.nome);
            if (data && data.confirmed && String(data.token) === String(payload.token)) return true;
        }
        return false;
    })();
}

const PENDING_KEY = 'mw_rsvp_pending_' + (GUEST.id || GUEST.name || 'self').toLowerCase().replace(/\s+/g, '_');

function showSending() {
    document.getElementById('rsvp-retry').style.display = 'none';
    const btn = document.getElementById('rsvp-submit');
    btn.disabled = true; btn.textContent = t('Enviando...');
}
function showRetry() {
    document.getElementById('rsvp-retry').style.display = 'block';
    const btn = document.getElementById('rsvp-submit');
    btn.disabled = false; btn.textContent = t('Confirmar presença');
}

async function submit() {
    const name = (document.getElementById('rsvp-guest').value || '').trim();
    const note = (document.getElementById('rsvp-msg').value || '').trim();
    if (attending === null) { alert(LANG === 'en' ? 'Please choose "Yes" or "No" first.' : 'Escolha "Sim, vou comparecer" ou "Não poderei ir" primeiro.'); return; }
    if (!name) { alert(LANG === 'en' ? 'Please enter your name.' : 'Por favor, digite o seu nome.'); return; }

    const payload = {
        id: GUEST.id, evento: CONFIG.coupleNames, nome: name,
        presenca: attending ? 'Sim' : 'Não',
        adultos: attending ? adults : 0,
        criancas: attending ? children : 0,
        mensagem: note, idioma: LANG, token: makeToken(),
        enviadoEm: new Date().toLocaleString('pt-BR')
    };

    showSending();
    const ok = await trySendVerified(payload);
    if (ok) {
        try { localStorage.removeItem(PENDING_KEY); } catch (e) {}
        try { localStorage.setItem(STORE_KEY, JSON.stringify({ attending, adults, children, note, name })); } catch (e) {}
        showSuccess(false);
    } else {
        try { localStorage.setItem(PENDING_KEY, JSON.stringify(payload)); } catch (e) {}
        showRetry();
    }
}
```

- [ ] **Step 4: Reescrever `restore()` para usar id + reenviar pendentes**

Substituir o `restore()` atual por:
```javascript
(function restore() {
    // 1) Já confirmado neste aparelho?
    let saved = null;
    try { saved = JSON.parse(localStorage.getItem(STORE_KEY)); } catch (e) {}
    if (saved && typeof saved.attending === 'boolean') {
        applyState(saved); showSuccess(true);
    }
    // 2) Há um envio pendente (falhou antes)? Tenta reenviar em segundo plano.
    let pending = null;
    try { pending = JSON.parse(localStorage.getItem(PENDING_KEY)); } catch (e) {}
    if (pending && pending.token) {
        trySendVerified(pending).then(function (ok) {
            if (ok) {
                try { localStorage.removeItem(PENDING_KEY); } catch (e) {}
                try { localStorage.setItem(STORE_KEY, JSON.stringify({ attending: pending.presenca === 'Sim', adults: +pending.adultos, children: +pending.criancas, note: pending.mensagem, name: pending.nome })); } catch (e) {}
            }
        });
    }
    // 3) Sem registro local: pergunta à planilha (por id) se já confirmou.
    if (!saved && (GUEST.id || GUEST.name)) {
        remoteCheck(GUEST.id, GUEST.name).then(function (data) {
            if (!data || !data.confirmed) return;
            const att = String(data.presenca || '').trim().toLowerCase() === 'sim';
            const st = { attending: att, adults: parseInt(data.adultos, 10) || GUEST.adults, children: parseInt(data.criancas, 10) || 0, note: data.mensagem || '', name: GUEST.name };
            applyState(st); showSuccess(true);
            try { localStorage.setItem(STORE_KEY, JSON.stringify(st)); } catch (e) {}
        });
    }
})();
```

- [ ] **Step 5: Garantir que `edit()` esconde o aviso de retry e expor `submit`**

No `edit()`, acrescentar a ocultação do aviso; conferir que o `return` do IIFE expõe `submit` (já expõe). `edit()` final:
```javascript
function edit() {
    document.getElementById('rsvp-success').style.display = 'none';
    document.getElementById('rsvp-retry').style.display = 'none';
    document.getElementById('rsvp-form').style.display = 'block';
    const btn = document.getElementById('rsvp-submit');
    btn.disabled = false;
    btn.textContent = t('Confirmar presença');
}
```

- [ ] **Step 6: Acrescentar as traduções novas (EN)**

No dicionário `TR`, adicionar:
```javascript
"Enviando...": "Sending...",
"Não consegui confirmar agora. Verifique a internet e tente de novo.": "We couldn't confirm just now. Check your connection and try again.",
"Tentar de novo": "Try again",
```

- [ ] **Step 7: Validar o JS**

Run:
```bash
awk '/<script>/{f=1;next} /<\/script>/{f=0} f' index.html > /tmp/j.js && node --check /tmp/j.js && echo "JS OK"
```
Expected: `JS OK`.

- [ ] **Step 8: Verificar o fluxo no navegador (sucesso)**

Com a Task 2 já implantada, no Chrome tool (viewport 390×844) abrir o arquivo local com um id de teste:
`file:///Users/alexandrekleis/Downloads/Wedding%20invite/index.html?convidado=Teste%20Plano&id=plano-teste-1&adultos=2`
Forçar `#main` visível, escolher "Sim", chamar `RSVP.submit()` via `eval`, aguardar ~6s e checar:
```javascript
(()=>({ form: document.getElementById('rsvp-form').style.display, success: document.getElementById('rsvp-success').style.display, retry: document.getElementById('rsvp-retry').style.display, title: document.getElementById('rsvp-success-title').textContent }))()
```
Expected: `success="block"`, `retry="none"`, `title="Presença confirmada!"`. Conferir na planilha que surgiu **uma** linha com id `plano-teste-1` e que o e-mail chegou.

- [ ] **Step 9: Verificar o fluxo de falha (sem falso sucesso)**

Ainda no navegador, simular endpoint inválido antes de enviar:
```javascript
CONFIG.googleSheetUrl = 'https://script.google.com/macros/s/INVALIDO/exec';
```
Escolher "Sim", `RSVP.submit()`, aguardar ~10s e checar o mesmo objeto do Step 8.
Expected: `success="none"`, `retry="block"` (apareceu "Tentar de novo"), e `localStorage` tem a chave `mw_rsvp_pending_plano-teste-1`. Confirma que **não** mostra sucesso falso.

- [ ] **Step 10: Commit**

```bash
awk '/<script>/{f=1;next} /<\/script>/{f=0} f' index.html > /tmp/j.js && node --check /tmp/j.js
git add index.html
git -c user.name="alexkleis" -c user.email="alexkleis@gmail.com" commit -m "RSVP: envio verificado por token, reenvio, pendentes e consulta por id"
git push
```

---

## Task 4: Gerador de links — `id` único por convidado

**Files:**
- Modify: `gerar-links.html` — função de adicionar convidado (`addGuest`), `buildLink`, e setup do CallMeBot na ajuda da página.

**Interfaces:**
- Consumes: nada de outras tasks.
- Produces: links no formato `...index.html?convidado=<nome>&adultos=<n>[&criancas=<k>][&lang=en]&id=<id>` com `id` estável de 8 chars.

- [ ] **Step 1: Gerar o id ao adicionar convidado**

Em `gerar-links.html`, na função `addGuest()`, gerar um id e guardá-lo no objeto do convidado:
```javascript
function genId() {
    var s = 'abcdefghijklmnopqrstuvwxyz0123456789';
    var out = '';
    for (var i = 0; i < 8; i++) out += s.charAt(Math.floor(Math.random() * s.length));
    return out;
}
```
E na criação do convidado, incluir `id: genId()`:
```javascript
guests.push({ name, adults, children: kids, lang, short: '', id: genId() });
```

- [ ] **Step 2: Incluir o id no link**

Em `buildLink(g)`, acrescentar o parâmetro `id` ao final:
```javascript
function buildLink(g) {
    const base = getBase();
    const sep = base.includes('?') ? '&' : '?';
    let link = `${base}${sep}convidado=${encodeURIComponent(g.name)}&adultos=${g.adults}`;
    if (g.children > 0) link += `&criancas=${g.children}`;
    if (g.lang === 'en') link += `&lang=en`;
    if (g.id) link += `&id=${g.id}`;
    return link;
}
```

- [ ] **Step 3: Acrescentar a dica de setup do CallMeBot**

No bloco de ajuda no topo de `gerar-links.html` (após o parágrafo de instruções), acrescentar:
```html
<p class="text-xs text-gray-500 mt-2">Para receber aviso no WhatsApp a cada confirmação, cada noivo deve, uma vez: enviar a mensagem <b>"I allow callmebot to send me messages"</b> para o número <b>+34 644 51 95 23</b> (CallMeBot) e guardar a apikey recebida — ela vai no Apps Script.</p>
```

- [ ] **Step 4: Verificar a geração de link no navegador**

No Chrome tool, abrir `file:///Users/alexandrekleis/Downloads/Wedding%20invite/gerar-links.html`, adicionar um convidado via `eval` e ler o link gerado:
```javascript
(()=>{ document.getElementById('g-name').value='Família Teste'; addGuest(); return document.querySelector('#list input').value; })()
```
Expected: a URL termina com `...&id=` seguido de 8 caracteres `a–z0–9`, e contém `convidado=Fam%C3%ADlia%20Teste&adultos=2`.

- [ ] **Step 5: Commit**

```bash
git add gerar-links.html
git -c user.name="alexkleis" -c user.email="alexkleis@gmail.com" commit -m "Gerador: id unico por convidado no link"
git push
```

---

## Task 5: Trocar URLs base para manueale.com (somente após DNS resolver)

**Files:**
- Modify: `index.html` — meta `og:image` (~linha com `og:image`).
- Modify: `gerar-links.html` — valor pré-preenchido de `#base-url`.

**Interfaces:**
- Consumes: Task 1 concluída e `https://manueale.com` respondendo 200 (Step 5 da Task 1).

- [ ] **Step 1: Confirmar que o domínio está no ar**

Run:
```bash
curl -sI https://manueale.com | head -n 1
```
Expected: `HTTP/2 200`. **Se não for 200, parar — não trocar as URLs ainda.**

- [ ] **Step 2: Atualizar a base do gerador de links**

Em `gerar-links.html`, trocar o `value` e o `placeholder` do `#base-url` para:
```
https://manueale.com/index.html
```

- [ ] **Step 3: Atualizar a og:image do convite**

Em `index.html`, trocar a URL da meta `og:image` de `https://alexkleis.github.io/emanuela-e-alexandre/IMG_3443.JPG` para:
```
https://manueale.com/IMG_3443.JPG
```

- [ ] **Step 4: Verificar**

Run:
```bash
curl -sI https://manueale.com/IMG_3443.JPG | head -n 1
awk '/<script>/{f=1;next} /<\/script>/{f=0} f' index.html > /tmp/j.js && node --check /tmp/j.js && echo "JS OK"
```
Expected: imagem retorna `HTTP/2 200`; `JS OK`.

- [ ] **Step 5: Commit**

```bash
git add index.html gerar-links.html
git -c user.name="alexkleis" -c user.email="alexkleis@gmail.com" commit -m "Usa dominio manueale.com nas URLs base (og:image e gerador)"
git push
```

---

## Self-Review (preenchido)

**Spec coverage:**
- "Não perder confirmação" → Task 3 (envio verificado por token + reenvio + pendentes; Steps 8–9 testam sucesso e falha).
- "Avisar na hora (e-mail+WhatsApp)" → Task 2 (`notificar_`: MailApp + CallMeBot best-effort).
- "Planilha limpa (upsert)" → Task 2 (`findRowById_`/`findRowByName_` + update no lugar).
- "Id único" → Task 4 (gera id) + Task 3 (lê/envia/consulta por id) + Task 2 (chave J).
- "Domínio manueale.com" → Task 1 (CNAME+DNS) + Task 5 (URLs base).

**Placeholder scan:** o markup do Step 2/Task 3 traz a versão final corrigida (sem `style2`). As `apikey: 'COLOQUE_...'` são marcadores **intencionais** que o usuário preenche no Apps Script (não no repo) e o código os ignora via `indexOf('COLOQUE')`. Nenhum "TODO/TBD".

**Type consistency:** `remoteCheck(id, name)` usado igual em `trySendVerified`, `submit` e `restore`. `trySendVerified(payload)` recebe sempre o objeto `payload` com os campos `{id, nome, presenca, adultos, criancas, mensagem, token, ...}`. `token` comparado como string nos dois lados (`String(...)`). Colunas `COL.*` consistentes entre `doPost`/`doGet`.

## Notas de operação (resumo para o usuário)

1. DNS do domínio (Task 1, Steps 3–4) e ativar no GitHub.
2. Cabeçalhos H/I/J na planilha (Task 2, Step 1).
3. Colar Apps Script v2, preencher apikeys do CallMeBot, **Nova implantação** (Task 2, Step 4).
4. Ativar CallMeBot uma vez por noivo (Task 4, Step 3).
5. O resto (convite e gerador) eu implemento e testo no navegador.
