# Simulador WhatsApp para N8N Dev

Guias separados para implementar o simulador de WhatsApp Cloud API no Analise Total, apontando para o N8N dev.

## Guias

| Arquivo | Para quem | O que faz |
|---------|-----------|-----------|
| [GUIA-FRONTEND-DEVELOPER.md](GUIA-FRONTEND-DEVELOPER.md) | Frontend Developer | Evoluir o ChatTotal no `app.js` — seletor de perfil, tipo de mensagem, webhook configuravel |
| [GUIA-N8N-ENGINEER.md](GUIA-N8N-ENGINEER.md) | N8N Engineer | Criar workflow dev com Webhook generico, normalizar input, substituir envio WhatsApp por Supabase |

## Arquitetura

```
Simulador (Analise Total / Wireguard)
    │
    │  POST payload IDENTICO ao Meta WhatsApp Cloud API
    ▼
Webhook generico (N8N dev: 76.13.172.17:5678)
    │
    ▼
Workflow identico a producao
    │
    ▼
Resposta gravada em Supabase (tabela resposta_ia)
    │
    ▼
Simulador captura via Supabase Realtime → Mostra no chat
```

## Ponto-chave

O payload e **identico** ao que a Meta envia. A unica diferenca e:
- **URL diferente** — vem do simulador, nao da Meta
- **Node diferente** — Webhook generico no lugar do WhatsAppTrigger nativo
- **Resposta diferente** — grava no Supabase em vez de enviar via WhatsApp API

Na hora de ir para producao, so troca o Webhook generico pelo WhatsAppTrigger nativo e reconecta os nodes de envio WhatsApp.
