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

## Passo 3: ~~Normalize Webhook Input~~ — NAO E MAIS NECESSARIO

O frontend foi corrigido para enviar o payload como **objeto** (nao array). Isso significa que o Webhook generico entrega `$json.messages`, `$json.contacts`, etc. diretamente — identico ao output do `whatsAppTrigger` nativo.

**Se voce ja adicionou o node `Normalize Webhook Input`, pode deletar.** Conecte o Webhook direto no `Edit Fields`.

---

## Passo 3 (atualizado): Copiar nodes de processamento da producao

Conectar o Webhook direto nos nodes copiados:

```
trigger-whatsapp (Webhook) → Edit Fields → If1 → BotGuard Normalize → ...
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

### O problema

Na producao existem **12 nodes** que enviam mensagens via WhatsApp API. No dev nao tem WhatsApp real — esses nodes vao falhar. Cada um precisa ser substituido por uma gravacao no Supabase, que o simulador captura via Realtime.

```
PRODUCAO:  N8N → WhatsApp API (graph.facebook.com) → celular do usuario
DEV:       N8N → INSERT na tabela resposta_ia (Supabase) → simulador captura via Realtime
```

### Lista completa dos 12 nodes a substituir

#### Nodes nativos WhatsApp (8 nodes)

| # | Node | Texto que envia | Node anterior |
|---|------|----------------|---------------|
| 1 | `Send message3` | "Detectei muitas mensagens em pouco tempo..." | `Blocked?` |
| 2 | `Send message2` | Transcricao de audio: `$json.text` | `Transcribe a recording` |
| 3 | `Send message4` | "Ola! Para ativar sua conta, digite seu email" | `Switch` (branch 2) |
| 4 | `Send message6` | "Foi enviado um codigo de verificacao..." | `HTTP Request1` |
| 5 | `Send message7` | "Digite o seu email correto." | `Switch1` / `Switch2` |
| 6 | `Send message8` | "Codigo incorreto ou expirado." | `Switch2` (branch 3) |
| 7 | `Send message9` | "Conta ativa com sucesso!" | `Update a row2` |
| 8 | `Send message10` | "Conta ativa com sucesso!" | `Create a row2` |

#### Nodes HTTP Request para graph.facebook.com (4 nodes)

| # | Node | O que envia | Node anterior |
|---|------|-------------|---------------|
| 9 | `HTTP Request — send agenda template1` | Template `lembretes_diarios` | `Edit Fields2` |
| 10 | `HTTP Request — send agenda text` | Texto da agenda | `HTTP Request — send agenda template1` |
| 11 | `HTTP Request — send agenda template` | Template `email_confirmacao` | `Switch` (branch 3) |
| 12 | `HTTP Request — send agenda template2` | Template `plano_inativo` | `If9` (branch 2) |

### Como substituir cada node

**Opcao A (recomendada): Criar 1 sub-workflow reutilizavel**

Criar um sub-workflow chamado **"Dev — Enviar Resposta"** com:

1. Node **Webhook** (trigger) — recebe a mensagem a enviar
2. Node **Supabase → Insert a Row**:

| Campo | Valor |
|-------|-------|
| **Table** | `resposta_ia` |
| `mensagem` | `={{ $json.mensagem }}` |
| `created_at` | `={{ new Date().toISOString() }}` |

Entao, no workflow principal, substituir cada um dos 12 nodes por um **Execute Workflow** apontando para esse sub-workflow, passando:

```json
{ "mensagem": "texto que seria enviado pelo WhatsApp" }
```

**Opcao B (mais simples): Substituir node a node**

Para cada um dos 12 nodes, deletar o node WhatsApp e colocar no lugar um **Supabase → Insert a Row**:

| Campo | Valor |
|-------|-------|
| **Table** | `resposta_ia` |
| `mensagem` | (copiar o texto que o node original enviava — ver tabela acima) |
| `created_at` | `={{ new Date().toISOString() }}` |

**Exemplo concreto — substituir `Send message4`:**

O original envia:
```
Olá! Tudo bem com você?
Para ativar sua conta, preciso que você digite seu *email*.

_Digite apenas o email de sua conta_
```

O substituto (Supabase Insert):
| Campo | Valor |
|-------|-------|
| Table | `resposta_ia` |
| `mensagem` | `=Olá! Tudo bem com você?\nPara ativar sua conta, preciso que você digite seu *email*.\n\n_Digite apenas o email de sua conta_` |
| `created_at` | `={{ new Date().toISOString() }}` |

**Exemplo concreto — substituir `Send message2` (transcricao de audio):**

O original envia o texto transcrito dinamicamente. O substituto:
| Campo | Valor |
|-------|-------|
| Table | `resposta_ia` |
| `mensagem` | `={{ "_" + $json.text + "_\n— Transcrito por *Total Assistente*, sua IA pessoal" }}` |
| `created_at` | `={{ new Date().toISOString() }}` |

**Exemplo concreto — substituir HTTP Request templates:**

Templates nao tem texto dinamico visivel. Substituir por texto fixo descritivo:
| Campo | Valor |
|-------|-------|
| Table | `resposta_ia` |
| `mensagem` | `=[Template: lembretes_diarios] Lembrete enviado para {{ $('trigger-whatsapp').item.json.contacts[0].profile.name }}` |
| `created_at` | `={{ new Date().toISOString() }}` |

### Fluxo da resposta no dev

```
N8N dev processa mensagem
    │
    ▼
Onde teria "Send message" ou "HTTP Request graph.facebook.com"
    │
    ▼
Supabase INSERT na tabela resposta_ia
    │
    ▼
Supabase dispara evento postgres_changes (Realtime)
    │
    ▼
Simulador (ChatTotal) recebe via subscription no canal 'chat_realtime'
    │
    ▼
Mensagem aparece no chat como resposta da IA
```

> **IMPORTANTE:** O simulador ja escuta a tabela `resposta_ia` via Supabase Realtime (app.js linha 755). Nao precisa mudar NADA no frontend para a resposta funcionar. So precisa garantir que o N8N dev grava nessa tabela.

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
- [ ] Copiar nodes de processamento da producao
- [ ] Conectar `trigger-whatsapp` (Webhook) → `Edit Fields` diretamente
- [ ] Atualizar credenciais Supabase para dev (se aplicavel)
- [ ] Substituir nodes de envio WhatsApp por gravacao em `resposta_ia`
- [ ] Ativar workflow
- [ ] Testar com curl (text, audio, interactive)
- [ ] Confirmar URL do webhook e passar para frontend developer
- [ ] Testar fluxo completo com simulador
