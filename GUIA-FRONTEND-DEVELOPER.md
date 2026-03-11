# Guia Frontend Developer — Simulador WhatsApp no ChatTotal

**Data:** 2026-03-11
**Arquivo:** `app.js` (componente `ChatTotal`, linha 681)
**Objetivo:** Evoluir o ChatTotal para simular webhooks identicos aos da WhatsApp Cloud API (Meta), apontando para o N8N dev.

---

## Passo 1: Adicionar constantes de configuracao (topo do ChatTotal)

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

---

## Passo 2: Criar funcao `buildPayload`

Criar ANTES do `handleSend`. Gera payloads **identicos** ao que a Meta envia ao webhook do WhatsApp Business:

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
            messageBase.audio = {
                mime_type: "audio/ogg; codecs=opus",
                sha256: randomId(44) + "=",
                id: randomMediaId(),
                voice: true
            };
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

> **IMPORTANTE:** Os valores `display_phone_number` e `phone_number_id` sao os mesmos da producao para que o workflow funcione identico.

---

## Passo 3: Atualizar o `handleSend` (linha 794)

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

---

## Passo 4: Adicionar UI de controles

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

---

## Passo 5: Atualizar placeholder do input

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

---

## Passo 6: Adicionar CSS minimo

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

---

## Resumo de mudancas

| Arquivo | O que muda |
|---------|-----------|
| `app.js` linhas 692-693 | Substituir constantes por PROFILES + state |
| `app.js` linhas 812-857 | Substituir payload inline por `buildPayload()` |
| `app.js` linha 851 | URL hardcoded → `webhookUrl` do state |
| `app.js` linhas 895-905 | Adicionar controles de simulador antes do form |
| `style.css` | 6 linhas de CSS |

**Nenhum arquivo novo. Nenhuma dependencia nova.**

---

## Checklist

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
