# Guia: Simulador WhatsApp para N8N Dev

**Data:** 2026-03-11
**Objetivo:** Evoluir o ChatTotal existente no Analise Total para simular webhooks identicos aos da WhatsApp Cloud API (Meta), apontando para o N8N dev em `76.13.172.17:5678`.

---

## PARTE 1 — Frontend Developer (app.js)

### Contexto

O componente `ChatTotal` (app.js linha 681) ja envia payloads para o N8N. O trabalho e:
- Adicionar seletor de perfil de usuario
- Adicionar seletor de tipo de mensagem (text, audio, image, document, interactive)
- Tornar webhook URL configuravel
- Manter payload 100% identico ao formato Meta

### Passo 1: Adicionar constantes de configuracao (topo do ChatTotal)

Substituir as constantes hardcoded por estado configuravel:

```js
// ANTES (linhas 692-693):
const USER_PHONE = '554391936205';
const USER_NAME = 'Luiz Felipe';

// DEPOIS:
const PROFILES = [
    { name: 'Luiz Felipe', phone: '554391936205' },
    { name: 'Usuario Teste A', phone: '5543999990001' },
    { name: 'Usuario Teste B', phone: '5543999990002' },
    { name: 'Maria Premium', phone: '5543999990003' },
];

const MSG_TYPES = [
    { value: 'text', label: 'Texto' },
    { value: 'audio', label: 'Audio' },
    { value: 'image', label: 'Imagem' },
    { value: 'document', label: 'Documento' },
    { value: 'interactive', label: 'Botao Interativo' },
];

const DEFAULT_WEBHOOK = 'http://n8n-fcwk0sw4soscgsgs08g8gssk.76.13.172.17.sslip.io/webhook/f5a55000-7947-4d20-8e7f-8fd13316f39f';

const [selectedProfile, setSelectedProfile] = useState(PROFILES[0]);
const [msgType, setMsgType] = useState('text');
const [webhookUrl, setWebhookUrl] = useState(
    localStorage.getItem('dev_webhook_url') || DEFAULT_WEBHOOK
);
const [showConfig, setShowConfig] = useState(false);
```

### Passo 2: Criar funcao `buildPayload` (substituir o payload inline)

Criar ANTES do `handleSend`. Esta funcao gera payloads **identicos** ao que a Meta envia ao webhook do WhatsApp Business:

```js
const randomId = (len) => {
    const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    return Array.from({ length: len }, () => chars[Math.floor(Math.random() * chars.length)]).join('');
};

const randomMediaId = () => Math.floor(1000000000000 + Math.random() * 9000000000000).toString();

function buildPayload(profile, type, content) {
    const messageBase = {
        from: profile.phone,
        id: `wamid.HBgM${profile.phone}FQIAEhg${randomId(20)}E`,
        timestamp: Math.floor(Date.now() / 1000).toString(),
        type: type,
    };

    switch (type) {
        case 'text':
            messageBase.text = { body: content };
            break;

        case 'audio':
            // Simula audio recebido (como Meta envia)
            messageBase.audio = {
                mime_type: "audio/ogg; codecs=opus",
                sha256: randomId(44) + "=",
                id: randomMediaId(),
                voice: true
            };
            // Se tiver texto, adiciona como contexto forwarded (audio c/ descricao)
            if (content) {
                messageBase.context = { forwarded: true };
            }
            break;

        case 'image':
            messageBase.image = {
                mime_type: "image/jpeg",
                sha256: randomId(44) + "=",
                id: randomMediaId(),
                caption: content || ''
            };
            break;

        case 'document':
            messageBase.document = {
                mime_type: "application/pdf",
                sha256: randomId(44) + "=",
                id: randomMediaId(),
                filename: "documento_teste.pdf",
                caption: content || ''
            };
            break;

        case 'interactive':
            // Simula clique em botao interativo (button_reply)
            // content deve ser { id: 'btn_id', title: 'Texto do Botao' }
            const btnData = typeof content === 'string'
                ? { id: 'btn_' + content.toLowerCase().replace(/\s/g, '_'), title: content }
                : content;
            messageBase.interactive = {
                type: "button_reply",
                button_reply: {
                    id: btnData.id,
                    title: btnData.title
                }
            };
            break;
    }

    // Retorna ARRAY — exatamente como Meta envia
    return [
        {
            messaging_product: "whatsapp",
            metadata: {
                display_phone_number: "554384983452",
                phone_number_id: "744582292082931"
            },
            contacts: [
                {
                    profile: { name: profile.name },
                    wa_id: profile.phone
                }
            ],
            messages: [messageBase],
            field: "messages"
        }
    ];
}
```

> **IMPORTANTE:** Os valores `display_phone_number: "554384983452"` e `phone_number_id: "744582292082931"` sao os mesmos da producao. O N8N dev deve usar os mesmos valores para que o workflow funcione identico.

### Passo 3: Atualizar o `handleSend` (linha 794)

Substituir o bloco try/catch inteiro:

```js
const handleSend = async (ev) => {
    ev.preventDefault();
    if (!input.trim() || isSending) return;

    const userMsgText = input;

    // Optimistic UI
    setMessages(prev => [...prev, {
        id: 'opt_' + Date.now(),
        text: msgType === 'audio' ? '🎤 [Audio enviado]' :
              msgType === 'image' ? '🖼️ [Imagem enviada]' + (userMsgText ? ': ' + userMsgText : '') :
              msgType === 'document' ? '📄 [Documento enviado]' + (userMsgText ? ': ' + userMsgText : '') :
              msgType === 'interactive' ? '🔘 [Botao: ' + userMsgText + ']' :
              userMsgText,
        type: 'user',
        time: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
    }]);

    setInput('');
    setIsSending(true);

    try {
        const payload = buildPayload(selectedProfile, msgType, userMsgText);

        const response = await fetch(webhookUrl, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });

        if (!response.ok) throw new Error(`Erro n8n: ${response.status}`);

    } catch (err) {
        console.error('Erro ao enviar para n8n:', err);
        setMessages(prev => [...prev, {
            id: 'err_' + Date.now(),
            text: `Erro: ${err.message}. Verifique se o N8N dev esta ativo.`,
            type: 'ai',
            time: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
        }]);
    } finally {
        setIsSending(false);
    }
};
```

### Passo 4: Adicionar UI de controles (no return do componente)

Adicionar ANTES do form de input (linha 895), dentro do container principal:

```js
// Barra de controle do simulador (acima do input)
e('div', { className: 'simulator-controls', style: {
    display: 'flex', gap: '8px', padding: '8px 16px',
    borderTop: '1px solid hsla(var(--foreground) / 0.1)',
    alignItems: 'center', flexWrap: 'wrap'
}},
    // Seletor de perfil
    e('select', {
        value: selectedProfile.phone,
        onChange: (ev) => setSelectedProfile(PROFILES.find(p => p.phone === ev.target.value)),
        style: { background: 'hsla(var(--background) / 0.6)', border: '1px solid hsla(var(--foreground) / 0.15)',
                 borderRadius: '8px', padding: '6px 10px', fontSize: '0.75rem', color: 'hsl(var(--foreground))' }
    },
        PROFILES.map(p => e('option', { key: p.phone, value: p.phone }, `${p.name} (${p.phone.slice(-4)})`))
    ),

    // Seletor de tipo de mensagem
    e('select', {
        value: msgType,
        onChange: (ev) => setMsgType(ev.target.value),
        style: { background: 'hsla(var(--background) / 0.6)', border: '1px solid hsla(var(--foreground) / 0.15)',
                 borderRadius: '8px', padding: '6px 10px', fontSize: '0.75rem', color: 'hsl(var(--foreground))' }
    },
        MSG_TYPES.map(t => e('option', { key: t.value, value: t.value }, t.label))
    ),

    // Botao config webhook
    e('button', {
        onClick: () => setShowConfig(!showConfig),
        style: { background: 'none', border: 'none', cursor: 'pointer', opacity: 0.5, fontSize: '0.75rem', color: 'hsl(var(--foreground))' }
    }, '⚙️ Webhook')
),

// Painel de config do webhook (colapsavel)
showConfig && e('div', { style: {
    padding: '8px 16px', background: 'hsla(var(--background) / 0.3)',
    borderTop: '1px solid hsla(var(--foreground) / 0.05)'
}},
    e('input', {
        value: webhookUrl,
        onChange: (ev) => {
            setWebhookUrl(ev.target.value);
            localStorage.setItem('dev_webhook_url', ev.target.value);
        },
        placeholder: 'Webhook URL do N8N dev',
        style: { width: '100%', background: 'hsla(var(--background) / 0.6)', border: '1px solid hsla(var(--foreground) / 0.15)',
                 borderRadius: '8px', padding: '8px 12px', fontSize: '0.75rem', color: 'hsl(var(--foreground))' }
    })
),
```

### Passo 5: Atualizar placeholder do input

Mudar o placeholder para refletir o tipo selecionado (no form, linha 896):

```js
e('input', {
    placeholder: isSending ? 'Processando...' :
        msgType === 'text' ? 'Digite uma mensagem...' :
        msgType === 'audio' ? 'Audio sera simulado (texto opcional)...' :
        msgType === 'image' ? 'Caption da imagem (opcional)...' :
        msgType === 'document' ? 'Caption do documento (opcional)...' :
        msgType === 'interactive' ? 'Texto do botao (ex: Sim, Resumir)...' :
        'Envie uma mensagem...',
    value: input,
    onChange: (v) => setInput(v.target.value),
    disabled: isSending
}),
```

### Passo 6: Adicionar CSS minimo

No `style.css`, adicionar:

```css
.simulator-controls select {
    font-family: inherit;
    cursor: pointer;
}
.simulator-controls select:focus {
    outline: 1px solid hsl(var(--primary));
}
```

### Resumo de mudancas no Frontend

| Arquivo | O que muda |
|---------|-----------|
| `app.js` linhas 692-693 | Substituir constantes por PROFILES + state |
| `app.js` linhas 812-857 | Substituir payload inline por `buildPayload()` |
| `app.js` linha 851 | URL hardcoded → `webhookUrl` do state |
| `app.js` linhas 895-905 | Adicionar controles de simulador antes do form |
| `style.css` | 6 linhas de CSS |

**Nenhum arquivo novo. Nenhuma dependencia nova.**

---

## PARTE 2 — N8N Engineer (Workflow Dev)

### Contexto

O N8N dev fica em `http://76.13.172.17:5678/`. O objetivo e criar um workflow que receba o mesmo payload da WhatsApp Cloud API, mas via **Webhook generico** (nao o WhatsAppTrigger nativo), para que o simulador funcione sem precisar de credenciais Meta.

### Por que Webhook generico e nao WhatsAppTrigger?

O node `whatsAppTrigger` do N8N valida OAuth com a Meta. Como estamos simulando, usamos um `Webhook` simples que aceita o mesmo JSON. O payload e identico — entao todo o resto do workflow funciona igual.

### Passo 1: Criar novo workflow "Dev - Total Assistente"

1. Abrir N8N dev: `http://76.13.172.17:5678/`
2. Criar workflow novo: **"Dev - Total Assistente"**
3. Ativar o workflow (toggle ON)

### Passo 2: Adicionar node Webhook (substitui o trigger-whatsapp)

| Campo | Valor |
|-------|-------|
| **Node type** | `Webhook` |
| **Node name** | `trigger-whatsapp` (MESMO NOME da producao!) |
| **HTTP Method** | POST |
| **Path** | `whatsapp-dev` (ou copiar o UUID da producao se preferir) |
| **Authentication** | None |
| **Response Mode** | Last Node |
| **Response Code** | 200 |

> **CRITICO:** O nome do node DEVE ser `trigger-whatsapp` porque todo o workflow referencia `$('trigger-whatsapp')` para extrair dados. Se o nome for diferente, todas as expressoes quebram.

**Depois de salvar**, copiar a URL do webhook. Sera algo como:
```
http://76.13.172.17:5678/webhook/whatsapp-dev
```
ou
```
http://n8n-fcwk0sw4soscgsgs08g8gssk.76.13.172.17.sslip.io/webhook/whatsapp-dev
```

Essa URL vai no frontend (Passo 1 do Frontend Developer).

### Passo 3: Tratar diferenca de formato (wrapper node)

O `whatsAppTrigger` nativo do N8N extrai automaticamente o conteudo do array e entrega um objeto. O Webhook generico recebe o array raw. Precisamos de um node intermediario:

1. Adicionar node **Code** logo depois do Webhook
2. Nome: `Normalize Webhook Input`
3. Codigo:

```javascript
// O simulador envia array (como Meta), extrair primeiro elemento
const input = $input.all();
const raw = input[0].json;

// Se veio como array (body e array), pegar primeiro item
// Se veio como objeto com body array, extrair
let payload;

if (Array.isArray(raw.body)) {
    payload = raw.body[0];
} else if (Array.isArray(raw)) {
    payload = raw[0];
} else if (raw.messaging_product) {
    payload = raw;
} else if (raw.body && raw.body.messaging_product) {
    payload = raw.body;
} else {
    payload = raw;
}

return [{ json: payload }];
```

> Isso garante que o output tenha o formato identico ao que o `whatsAppTrigger` nativo entrega: um objeto com `messaging_product`, `metadata`, `contacts`, `messages`, `field`.

### Passo 4: Copiar nodes de processamento da producao

A partir daqui, copiar os nodes da producao **na ordem exata**. A cadeia e:

```
trigger-whatsapp → Normalize Webhook Input → Edit Fields → If1 → BotGuard Normalize → ...
```

**Nodes a copiar do workflow de producao (nesta ordem):**

| # | Node | Tipo | Funcao |
|---|------|------|--------|
| 1 | `Edit Fields` | Set | Extrai `messageSet` = `messages[0].button?.text \|\| messages[0].text?.body` |
| 2 | `If1` | If | Filtra status updates (so passa mensagens reais) |
| 3 | `BotGuard Normalize` | Code | Extrai phone, body, body_hash |
| 4 | `Insert bot_events` | Supabase | Loga evento (apontar para Supabase dev se tiver) |
| 5 | `HTTP Request` (BotGuard) | HTTP | Verifica rate limit |
| 6 | `Blocked?` | If | Verifica se usuario esta bloqueado |
| 7 | `If3` | If | Detecta audio forwarded |
| 8 | `Get a row` | Supabase | Busca usuario na tabela `phones_whatsapp` |

**Para copiar:** Na producao, selecionar os nodes → Ctrl+C → No workflow dev → Ctrl+V.

### Passo 5: Adaptar credenciais e endpoints

Cada node copiado que usa credenciais precisa ser atualizado:

| Node | Credencial Producao | Acao no Dev |
|------|---------------------|-------------|
| Supabase nodes | "Total Supabase" | Criar credencial Supabase dev ou usar mesma (se mesmo DB) |
| HTTP Request (WhatsApp send) | "WhatsApp Header Auth" | **NAO COPIAR** — substituir por node que grava resposta no Supabase (ver Passo 6) |
| WhatsApp send nodes | "WhatsApp account 2" | **NAO COPIAR** — substituir (ver Passo 6) |

### Passo 6: Substituir envio WhatsApp por gravacao no Supabase

Na producao, a resposta da IA e enviada via WhatsApp API. No dev, queremos que a resposta volte para o chat do simulador. O chat monitora a tabela `resposta_ia` via Supabase realtime.

**Em CADA node que envia mensagem WhatsApp**, substituir por um node **Supabase → Insert a Row**:

| Campo | Valor |
|-------|-------|
| **Table** | `resposta_ia` |
| **Campos:** | |
| `mensagem` | `={{ $json.textBody }}` ou o texto da resposta da IA |
| `created_at` | `={{ new Date().toISOString() }}` |

Ou, se preferir manter os nodes de envio WhatsApp para referencia futura, adicionar um **node Code** ANTES deles que grava no Supabase:

```javascript
// Gravar resposta no Supabase para o simulador capturar
const resposta = $json.textBody || $json.text || $json.body || 'Resposta processada';

return [{
    json: {
        ...$json,
        _dev_response: resposta
    }
}];
```

E conectar a um **Supabase Insert** node em paralelo ao send WhatsApp.

### Passo 7: Configurar variavel de ambiente (opcional mas recomendado)

No N8N dev, configurar uma variavel de ambiente para distinguir dev de prod:

```
N8N_ENV=dev
```

E nos Code nodes, pode usar:
```javascript
const isDev = $env.N8N_ENV === 'dev';
```

Isso permite usar o mesmo workflow com comportamento diferente (ex: nao enviar WhatsApp real em dev).

### Passo 8: Testar com curl

Antes de testar pelo site, validar com curl:

```bash
curl -X POST \
  'http://76.13.172.17:5678/webhook/whatsapp-dev' \
  -H 'Content-Type: application/json' \
  -d '[{
    "messaging_product": "whatsapp",
    "metadata": {
      "display_phone_number": "554384983452",
      "phone_number_id": "744582292082931"
    },
    "contacts": [{
      "profile": { "name": "Teste Dev" },
      "wa_id": "5543999990001"
    }],
    "messages": [{
      "from": "5543999990001",
      "id": "wamid.HBgMNTU0Mzk5OTkwMDAxFQIAEhgWM0VCMDFEQUNTESTE",
      "timestamp": "1710000000",
      "type": "text",
      "text": { "body": "Ola, teste do simulador!" }
    }],
    "field": "messages"
  }]'
```

Se retornar 200 e a execucao aparecer no N8N, o webhook esta funcionando.

**Testar cada tipo:**

Audio:
```bash
curl -X POST \
  'http://76.13.172.17:5678/webhook/whatsapp-dev' \
  -H 'Content-Type: application/json' \
  -d '[{
    "messaging_product": "whatsapp",
    "metadata": {
      "display_phone_number": "554384983452",
      "phone_number_id": "744582292082931"
    },
    "contacts": [{
      "profile": { "name": "Teste Dev" },
      "wa_id": "5543999990001"
    }],
    "messages": [{
      "from": "5543999990001",
      "id": "wamid.HBgMNTU0Mzk5OTkwMDAxFQIAEhgWM0VCMDFEQUNTESTE",
      "timestamp": "1710000000",
      "type": "audio",
      "audio": {
        "mime_type": "audio/ogg; codecs=opus",
        "sha256": "mIdSs7maDEskotWCS833PF5KbYl4iY4a8rhISgHVAhk=",
        "id": "1453326049640573",
        "voice": true
      },
      "context": { "forwarded": true }
    }],
    "field": "messages"
  }]'
```

Botao interativo:
```bash
curl -X POST \
  'http://76.13.172.17:5678/webhook/whatsapp-dev' \
  -H 'Content-Type: application/json' \
  -d '[{
    "messaging_product": "whatsapp",
    "metadata": {
      "display_phone_number": "554384983452",
      "phone_number_id": "744582292082931"
    },
    "contacts": [{
      "profile": { "name": "Teste Dev" },
      "wa_id": "5543999990001"
    }],
    "messages": [{
      "from": "5543999990001",
      "id": "wamid.HBgMNTU0Mzk5OTkwMDAxFQIAEhgWM0VCMDFEQUNTESTE",
      "timestamp": "1710000000",
      "type": "interactive",
      "interactive": {
        "type": "button_reply",
        "button_reply": {
          "id": "btn_resumir",
          "title": "Resuma para mim"
        }
      }
    }],
    "field": "messages"
  }]'
```

### Passo 9: Fluxo completo no dev

```
Simulador (Analise Total)
    │
    │  POST payload Meta-identico
    ▼
Webhook generico (N8N dev: trigger-whatsapp)
    │
    ▼
Normalize Webhook Input (Code node - unwrap array)
    │
    ▼
Edit Fields (extrai messageSet)
    │
    ▼
If1 (filtra status updates)
    │
    ▼
BotGuard Normalize (extrai phone, body, hash)
    │
    ▼
[... resto do workflow identico a producao ...]
    │
    ▼
Em vez de enviar WhatsApp → Gravar em resposta_ia (Supabase)
    │
    ▼
Simulador captura via Supabase Realtime → Mostra no chat
```

---

## PARTE 3 — Checklist de Deploy

### Frontend Developer

- [ ] Adicionar constantes PROFILES e MSG_TYPES
- [ ] Adicionar states (selectedProfile, msgType, webhookUrl, showConfig)
- [ ] Criar funcao `buildPayload()`
- [ ] Atualizar `handleSend` para usar `buildPayload()` e `webhookUrl`
- [ ] Adicionar UI de controles (selects + config webhook)
- [ ] Atualizar placeholder do input
- [ ] Adicionar CSS
- [ ] Testar envio de cada tipo (text, audio, image, document, interactive)
- [ ] Verificar que resposta aparece via Supabase realtime
- [ ] Commitar e push

### N8N Engineer

- [ ] Criar workflow "Dev - Total Assistente"
- [ ] Adicionar node Webhook (nome: `trigger-whatsapp`, path: `whatsapp-dev`)
- [ ] Adicionar node Code "Normalize Webhook Input"
- [ ] Copiar nodes de processamento da producao
- [ ] Atualizar credenciais Supabase para dev (se aplicavel)
- [ ] Substituir nodes de envio WhatsApp por gravacao em `resposta_ia`
- [ ] Ativar workflow
- [ ] Testar com curl (text, audio, interactive)
- [ ] Confirmar URL do webhook e passar para frontend developer
- [ ] Testar fluxo completo com simulador

---

## Referencia: Campos do Payload Meta (Completo)

```json
{
    "messaging_product": "whatsapp",
    "metadata": {
        "display_phone_number": "554384983452",
        "phone_number_id": "744582292082931"
    },
    "contacts": [{
        "profile": { "name": "Nome do Contato" },
        "wa_id": "5543XXXXXXXXX"
    }],
    "messages": [{
        "from": "5543XXXXXXXXX",
        "id": "wamid.HBgM...",
        "timestamp": "1710000000",
        "type": "text|audio|image|document|interactive|button",
        "text": { "body": "..." },
        "audio": { "mime_type": "audio/ogg; codecs=opus", "sha256": "...", "id": "...", "voice": true },
        "image": { "mime_type": "image/jpeg", "sha256": "...", "id": "...", "caption": "..." },
        "document": { "mime_type": "application/pdf", "sha256": "...", "id": "...", "filename": "...", "caption": "..." },
        "interactive": { "type": "button_reply", "button_reply": { "id": "btn_x", "title": "Texto" } },
        "button": { "text": "Texto do botao" },
        "context": { "forwarded": true, "frequently_forwarded": false }
    }],
    "field": "messages"
}
```

## Referencia: Expressoes N8N que acessam o payload

Estas expressoes sao usadas no workflow de producao e DEVEM funcionar identicamente no dev:

| Expressao | Retorna |
|-----------|---------|
| `$('trigger-whatsapp').item.json.contacts[0].wa_id` | Telefone do remetente |
| `$('trigger-whatsapp').item.json.contacts[0].profile.name` | Nome do remetente |
| `$('trigger-whatsapp').item.json.messages[0].from` | Telefone (alternativo) |
| `$('trigger-whatsapp').item.json.messages[0].type` | Tipo da mensagem |
| `$('trigger-whatsapp').item.json.messages[0].text.body` | Texto da mensagem |
| `$('trigger-whatsapp').item.json.messages[0].audio.id` | ID do audio (Meta) |
| `$('trigger-whatsapp').item.json.messages[0].image.id` | ID da imagem (Meta) |
| `$('trigger-whatsapp').item.json.messages[0].document.id` | ID do documento |
| `$('trigger-whatsapp').item.json.messages[0].document.mime_type` | MIME do documento |
| `$('trigger-whatsapp').item.json.messages[0].interactive.button_reply.id` | ID do botao clicado |
| `$('trigger-whatsapp').item.json.messages[0].interactive.button_reply.title` | Texto do botao |
| `$('trigger-whatsapp').item.json.messages[0].id` | ID unico da mensagem |
| `$('trigger-whatsapp').item.json.messages[0].context.forwarded` | Se e encaminhada |

---

*Relatorio gerado por Sherlock (analisador-n8n) — STRICT READ-ONLY*
