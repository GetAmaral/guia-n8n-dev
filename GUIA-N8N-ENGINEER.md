# Guia N8N Engineer — Workflow Dev com Webhook Simulado

**Data:** 2026-03-11
**N8N Dev:** `http://76.13.172.17:5678/`
**Objetivo:** Criar workflow que receba o mesmo payload da WhatsApp Cloud API via Webhook generico (sem credenciais Meta), para que o simulador do Analise Total funcione.

---

## Por que Webhook generico e nao WhatsAppTrigger?

O node `whatsAppTrigger` do N8N valida OAuth com a Meta. Como estamos simulando, usamos um `Webhook` simples que aceita o mesmo JSON. O payload e identico — entao todo o resto do workflow funciona igual.

---

## Passo 1: Criar novo workflow

1. Abrir N8N dev: `http://76.13.172.17:5678/`
2. Criar workflow novo: **"Dev - Total Assistente"**
3. Ativar o workflow (toggle ON)

---

## Passo 2: Adicionar node Webhook

| Campo | Valor |
|-------|-------|
| **Node type** | `Webhook` |
| **Node name** | `trigger-whatsapp` (MESMO NOME da producao!) |
| **HTTP Method** | POST |
| **Path** | `whatsapp-dev` |
| **Authentication** | None |
| **Response Mode** | Last Node |
| **Response Code** | 200 |

> **CRITICO:** O nome do node DEVE ser `trigger-whatsapp` porque todo o workflow referencia `$('trigger-whatsapp')` para extrair dados. Se o nome for diferente, todas as expressoes quebram.

**Depois de salvar**, copiar a URL do webhook. Sera algo como:
```
http://76.13.172.17:5678/webhook/whatsapp-dev
```

Essa URL vai para o frontend developer configurar no simulador.

---

## Passo 3: Adicionar node Code (normalizar input)

O `whatsAppTrigger` nativo extrai automaticamente o conteudo do array. O Webhook generico recebe o array raw. Precisamos de um node intermediario:

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

---

## Passo 4: Copiar nodes de processamento da producao

A partir daqui, copiar os nodes da producao **na ordem exata**:

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

> **IMPORTANTE:** Conectar a saida do `Normalize Webhook Input` na entrada do `Edit Fields`.

---

## Passo 5: Adaptar credenciais

Cada node copiado que usa credenciais precisa ser atualizado:

| Node | Credencial Producao | Acao no Dev |
|------|---------------------|-------------|
| Supabase nodes | "Total Supabase" | Criar credencial Supabase dev ou usar mesma (se mesmo DB) |
| HTTP Request (WhatsApp send) | "WhatsApp Header Auth" | **NAO COPIAR** — substituir (ver Passo 6) |
| WhatsApp send nodes | "WhatsApp account 2" | **NAO COPIAR** — substituir (ver Passo 6) |

---

## Passo 6: Substituir envio WhatsApp por gravacao no Supabase

Na producao, a resposta da IA e enviada via WhatsApp API. No dev, a resposta deve voltar para o chat do simulador. O simulador monitora a tabela `resposta_ia` via Supabase realtime.

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

---

## Passo 7: Variavel de ambiente (opcional)

No N8N dev, configurar:

```
N8N_ENV=dev
```

Nos Code nodes, pode usar:
```javascript
const isDev = $env.N8N_ENV === 'dev';
```

Permite usar o mesmo workflow com comportamento diferente (ex: nao enviar WhatsApp real em dev).

---

## Passo 8: Testar com curl

Antes de testar pelo site, validar com curl.

**Texto:**
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

**Audio:**
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

**Botao interativo:**
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

Se retornar 200 e a execucao aparecer no N8N, o webhook esta funcionando.

---

## Passo 9: Fluxo completo

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

## Referencia: Payload completo da Meta

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

---

## Checklist

- [ ] Criar workflow "Dev - Total Assistente"
- [ ] Adicionar node Webhook (nome: `trigger-whatsapp`, path: `whatsapp-dev`)
- [ ] Adicionar node Code "Normalize Webhook Input"
- [ ] Copiar nodes de processamento da producao
- [ ] Conectar `Normalize Webhook Input` → `Edit Fields`
- [ ] Atualizar credenciais Supabase para dev (se aplicavel)
- [ ] Substituir nodes de envio WhatsApp por gravacao em `resposta_ia`
- [ ] Ativar workflow
- [ ] Testar com curl (text, audio, interactive)
- [ ] Confirmar URL do webhook e passar para frontend developer
- [ ] Testar fluxo completo com simulador
