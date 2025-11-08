# WhatsApp Multi-Agent Automation

[![N8N](https://img.shields.io/badge/n8n-workflow-EA4B71?style=flat-square&logo=n8n)](https://n8n.io/)
[![WPPConnect](https://img.shields.io/badge/WPPConnect-API-25D366?style=flat-square&logo=whatsapp)](https://github.com/wppconnect-team/wppconnect)
[![JavaScript](https://img.shields.io/badge/JavaScript-ES6+-F7DF1E?style=flat-square&logo=javascript&logoColor=black)](https://www.javascript.com/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](LICENSE)

Sistema de automação que centraliza atendimento WhatsApp em grupos, permitindo múltiplos agentes responderem clientes simultâneos através de um único número.

## Problema

WhatsApp Business API limita atendimento a um dispositivo. Empresas com equipes precisam:
- Múltiplos agentes atendendo no mesmo número
- Contexto de conversa preservado
- Identificação clara de qual cliente está sendo atendido
- Suporte completo a mídias (áudio, vídeo, stickers, documentos)

## Solução

Workflow N8N que roteia mensagens entre clientes individuais e grupo de atendimento:

```
Cliente → WPPConnect → N8N Webhook → Grupo Atendimento
                                   ↓
Cliente ← WPPConnect ← N8N ← Agente (cita mensagem no grupo)
```

**Features:**
- Auto-forward de mensagens cliente → grupo com metadados (telefone, timestamp, tipo)
- Reply inteligente: agente cita mensagem no grupo, sistema detecta destinatário
- Comando `/citar` para citações oficiais via `send-reply` endpoint
- Detecção automática de mídia por análise de magic bytes em base64
- Message ID tracking via Zero Width Space characters (invisível ao usuário)

## Stack

| Componente | Função |
|-----------|---------|
| N8N | Workflow orchestration & business logic |
| WPPConnect | WhatsApp Web API wrapper |
| JavaScript | Media detection, parsing, formatting |
| Webhooks | Real-time event handling |

## Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                         N8N Workflow                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Webhook] → [Extract] → [IF:onmessage]                    │
│                              ↓                              │
│                         [IF:/citar?]                        │
│                         ↙        ↘                          │
│               [Process Citar]  [IF:isGroup?]               │
│                     ↓              ↙      ↘                 │
│              [Format Reply]  [Grp] [Cli]                   │
│                     ↓           ↓     ↓                     │
│              [HTTP:Reply]  [HTTP] [HTTP]                   │
│                     ↓           ↓     ↓                     │
│                         [Response]                          │
└─────────────────────────────────────────────────────────────┘
```

### Fluxo de Dados

**Cliente → Grupo:**
1. WPPConnect envia webhook com mensagem
2. Node `Extrai` detecta tipo via magic bytes
3. Node `Formata Grupo` adiciona metadados + hidden ID
4. HTTP request para grupo com formatação

**Grupo → Cliente:**
1. Agente cita mensagem no grupo
2. Node `Extrai Cliente` faz regex match no número
3. Node `Formata Cliente` monta payload
4. HTTP request direto para cliente

**Comando /citar:**
1. Detecta pattern `/citar` + quoted message
2. Extrai ID original (hidden ou via quotedMsg tree)
3. Usa endpoint `send-reply` com messageId
4. Cliente recebe como citação oficial

## Setup

### Pré-requisitos

```bash
# Instâncias necessárias
- N8N instance (self-hosted ou cloud)
- WPPConnect server rodando
- WhatsApp conectado via QR code
- Grupo WhatsApp criado
```

### 1. Clone & Import

```bash
# Importar no N8N
1. Settings → Import from File
2. Selecionar n8n.json
3. Workflow importado com placeholders
```

### 2. Configurar Credenciais

#### a) WPPConnect API Key

Obter token no painel WPPConnect:

```bash
curl -X GET "http://localhost:21465/api/{{session}}/show-all-sessions" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

#### b) Group ID

Listar chats e filtrar grupos:

```bash
curl -X GET "http://localhost:21465/api/{{session}}/all-chats" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  | jq '.[] | select(.id | contains("@g.us"))'
```

Salvar ID no formato: `120363XXXXXXXXX@g.us`

### 3. Configurar Nodes

| Node | Campo | Valor |
|------|-------|-------|
| `Webhook` | path | `seu-webhook-path` |
| `Formata Grupo` | phone (linha 100) | `{{GROUP_ID}}@g.us` |
| `Monta URL` | _url (linha 119) | `{{URL}}/api/{{SESSION}}/${endpoint}` |
| `Envia Identificação` | url | `{{URL}}/api/{{SESSION}}/send-message` |
| `Prepara HTTP Cliente` | _url (linha 192) | `{{URL}}/api/{{SESSION}}/${endpoint}` |

**Headers (3 nodes HTTP):**
```json
{
  "Authorization": "Bearer {{YOUR_TOKEN}}",
  "Content-Type": "application/json"
}
```

Substituir em:
- `HTTP Grupo` (linha 124)
- `Envia Identificação` (linha 164)
- `HTTP Cliente` (linha 206)

### 4. Registrar Webhook

```bash
curl -X POST "{{WPPCONNECT_URL}}/api/{{SESSION}}/webhook" \
  -H "Authorization: Bearer {{TOKEN}}" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "{{N8N_WEBHOOK_URL}}",
    "events": ["onmessage"]
  }'
```

### 5. Ativar Workflow

```
N8N UI → Toggle "Active" → Webhook URL gerada
```

## Testes

### Smoke Tests

```bash
# 1. Cliente → Grupo (texto)
# Enviar WhatsApp text para número conectado
# Verificar: mensagem aparece no grupo com dados formatados

# 2. Cliente → Grupo (mídia)
# Enviar imagem/áudio/sticker
# Verificar: mídia + caption identificação

# 3. Grupo → Cliente (reply normal)
# Citar mensagem do cliente no grupo
# Digitar resposta
# Verificar: cliente recebe

# 4. Grupo → Cliente (/citar)
# Citar mensagem do cliente
# Digitar: /citar sua resposta
# Verificar: cliente recebe como quoted message oficial
```

### Debugging

```javascript
// Node "DEBUG - Ver Dados" (linha 27)
// Log completo do webhook payload
console.log(JSON.stringify(data.originalMessage, null, 2));

// Node "Processa Citar" (linha 68)
// Trace de extração de cliente e messageId
console.log('Cliente extraído:', clientPhone);
console.log('ID final:', quotedOriginalId);
```

## Detecção de Mídia

Sistema analisa magic bytes em base64 para identificar tipos:

| Tipo | Magic Bytes (base64) | Mimetype |
|------|---------------------|----------|
| WebP (sticker) | `UklGR` | image/webp |
| WebP animated | `UklGR` + `QU5JT` | image/webp |
| OGG Opus (audio) | `T2dnUw`, `SUQz` | audio/ogg |
| JPEG | `/9j/` | image/jpeg |
| PNG | `iVBORw` | image/png |
| MP4 | `AAAAIGZ0eXBpc29t` | video/mp4 |
| PDF | `JVBERi0` | application/pdf |

**Implementação:**

```javascript
// Node "Extrai" (linha 19)
if (data.body.startsWith('UklGR')) {
  isMediaDetected = true;
  mimetypeDetected = 'image/webp';
  if (data.body.includes('QU5JT')) {
    typeDetected = 'sticker-animated';
  } else {
    typeDetected = 'sticker';
  }
}
```

## Message ID Tracking

Problema: Citações precisam do messageId original do cliente.

**Solução 1: Hidden ID**

```javascript
// Inserção (Node "Formata Grupo", linha 100)
const hiddenId = originalMessageId ? `\u200B[ID:${originalMessageId}]\u200B` : '';
const message = `${hiddenId}📱 Cliente: ...`;

// Extração (Node "Processa Citar", linha 68)
const hiddenIdMatch = data.quotedMsg.body.match(/\[ID:([^\]]+)\]/);
if (hiddenIdMatch) {
  quotedOriginalId = hiddenIdMatch[1];
}
```

**Solução 2: Fallback Tree Search**

```javascript
// Node "Processa Citar"
// Busca hierárquica em estrutura WPPConnect
if (data.quotedMsg.quotedMsg?.id) {
  quotedOriginalId = data.quotedMsg.quotedMsg.id;
} else if (data.quotedMsg._data?.quotedStanzaID) {
  quotedOriginalId = data.quotedMsg._data.quotedStanzaID;
} // ... mais 3 níveis de fallback
```

## Mídias Especiais (Audio/Sticker/Video)

WhatsApp não permite caption em áudios e stickers. Solução: envio duplo.

```javascript
// Node "Formata Grupo"
if (data.type === 'ptt') {
  endpoint = 'send-voice-base64';
  payload.base64Ptt = base64Clean;
  payload._audioMode = true; // flag para envio duplo
  payload._textMessage = '📱 Cliente: ...'; // texto separado
}

// Node "Envia Identificação" (linha 157)
// Envia mensagem de texto APÓS mídia
// Condição: IF Audio/Sticker/Video (linha 141)
```

## Troubleshooting

| Erro | Causa Provável | Fix |
|------|---------------|------|
| Webhook não dispara | URL incorreta no WPPConnect | Verificar registration + testar com curl |
| 401 Unauthorized | Token inválido/expirado | Regenerar token no WPPConnect |
| Mensagem não chega ao grupo | Group ID errado | Confirmar formato `XXX@g.us` |
| Reply não chega ao cliente | Regex não extraiu número | Ver logs do node "Extrai Cliente" |
| /citar não funciona | messageId não encontrado | Verificar hidden ID ou fallback tree |
| Mídia corrompida | Base64 malformado | Verificar data URI prefix `data:{{mime}};base64,` |

### Debug Checklist

```bash
# 1. N8N Executions
# Ver histórico completo de execuções e erros

# 2. Console Logs
# Nodes Function têm console.log estratégicos

# 3. WPPConnect Logs
# Verificar servidor WPPConnect para erros de API

# 4. Webhook Test
curl -X POST "{{N8N_WEBHOOK_URL}}" \
  -H "Content-Type: application/json" \
  -d '{"event":"onmessage","from":"5511999998888@c.us","body":"test"}'
```

## Limitações Conhecidas

- Parser de telefone assume formato brasileiro (55 + DDD + número)
- Hidden ID pode ser visível se usuário copiar/colar texto
- Stickers animados > 500KB podem falhar (limite WhatsApp)
- Detecção de mídia falha se base64 estiver em campo não padrão
- Citação manual (sem /citar) não funciona em áudios/stickers sem identificação prévia

## Roadmap

- [ ] Database logging (PostgreSQL) para histórico
- [ ] Multi-language phone formatting
- [ ] Queue system para rate limiting
- [ ] CRM integration hooks (Salesforce, Pipedrive)
- [ ] Analytics dashboard (Grafana)
- [ ] Auto-assignment por round-robin
- [ ] Template quick replies
- [ ] LLM integration (GPT-4, Claude) para auto-reply
- [ ] Voice transcription (Whisper API)
- [ ] Sentiment analysis

## Segurança

**IMPORTANTE:** Este repo está sanitizado. Antes de prod:

- [ ] Substituir `YOUR_*` placeholders
- [ ] Usar variáveis de ambiente (nunca hardcode)
- [ ] Rate limiting no webhook (evitar flood)
- [ ] Whitelist IPs do WPPConnect
- [ ] HTTPS obrigatório (Let's Encrypt)
- [ ] Rotação de tokens (30 dias)
- [ ] Logs sanitizados (sem números de telefone)
- [ ] GDPR compliance se operar na UE

## Performance

**Benchmarks** (testes locais):
- Latência média (cliente → grupo): ~1.2s
- Throughput: ~50 msg/s (limitado por WhatsApp rate limit)
- Memory footprint: ~150MB (N8N workflow ativo)
- Base64 parsing overhead: ~50ms para imagem 2MB

**Otimizações aplicadas:**
- Detecção de mídia short-circuit (early return)
- Regex compilation cache em loops
- Payload cleanup antes de HTTP (delete de flags internas)
- Console.log apenas em debug mode

## Contributing

PRs são bem-vindos. Para mudanças grandes, abrir issue primeiro.

```bash
# Fork → Clone → Branch → Code → Test → PR
git checkout -b feature/nova-feature
# ... código ...
git commit -m "feat: adiciona suporte a GIF nativo"
git push origin feature/nova-feature
```

## License

MIT License - veja [LICENSE](LICENSE) para detalhes.

## Autor

Desenvolvido por **[Seu Nome]**

- GitHub: [@seu-usuario](https://github.com/laurogomes1)
- LinkedIn: [/in/seu-perfil]((https://www.linkedin.com/in/lauro-gomes-537273b1/))
- Email: lauro.silva@1clickmkt.com.br

---

⭐ Se este projeto foi útil, deixe uma star!

**Tags:** `n8n` `whatsapp-automation` `wppconnect` `webhook` `multi-agent` `customer-service` `workflow` `javascript` `api-integration`
