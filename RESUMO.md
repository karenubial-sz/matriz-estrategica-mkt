# Disparo de Marketing — Resumo do Projeto

**Ultima atualizacao:** 2026-06-19
**Responsavel:** Karen Ubial (Kakau)
**Stakeholders:** Mari (SZI), Leo (SZS), Roberto (Marketplace), Bruno Benetti (diretor)

---

## O que e o projeto

Motor de Disparos de Marketing da Seazone — um webapp que segmenta a base de leads com regras flexiveis, orquestra cadencias inteligentes de comunicacao (WhatsApp + email) baseadas no comportamento do lead, e analisa resultados com visao de funil x cobertura por vertical.

## Estado atual (19/06)

### Webapp funcional — E2E testado com envio real
- **Backend FastAPI** rodando em `localhost:8000` — 208k leads, CRUD completo, disparo funcional
- **Frontend Next.js** rodando em `localhost:3000` — todas as paginas + fluxo de disparo E2E
- **Postgres local** com schema completo (18 tabelas, 2 schemas: staging + main)
- **Disparo testado e funcionando** — envio real via Morada Gateway confirmado
- **MCP Server** criado (17 tools) pra operadores usarem via Claude

### Fluxo E2E validado
1. Criar campanha pelo wizard (nome, tipo, vertical, canal, audiencia, template)
2. Audiencia filtrada por subtipo — modal mostra leads que vao receber
3. Clicar "Disparar" na lista de campanhas
4. Backend prepara disparos + envia via Morada Gateway `/send-notification`
5. Status muda pra "finalizada" automaticamente
6. Trava de seguranca: whitelist de telefones no executor (so numeros autorizados)

### Integracoes — estado

| Sistema | Status |
|---------|--------|
| Morada Gateway (WA envio) | **FUNCIONAL** — `/send-notification` com X-Morada-Api-Key |
| Meta Graph API (templates) | **FUNCIONAL** — submit + check status de templates |
| Nekt (dados leads) | Funcional via script manual (sync_full.py) |
| Spotometro | Conectado com cache 1h |
| Google OAuth | Funcional |
| SIA/CIA (execucao) | MISSING — zero codigo |
| RD Station | MISSING |
| Email (Resend) | MISSING |
| Smartsheet (marcos) | MISSING — API key obtida |

### Concluido nesta sessao (19/06)
- **UI melhorada:** navbar, tabelas, dropdowns, pagina Sobre reescrita
- **Wizard de campanha reescrito:** step indicator com icones, dropdown com busca, audiencia simplificada
- **Disparo E2E pela UI:** botao Disparar na lista, modal de confirmacao com leads, envio real via Morada
- **Integracao Meta:** submissao de templates pra aprovacao + check de status
- **MCP Server:** 17 tools mapeadas pro backend FastAPI, instructions com fluxos guiados
- **Botoes de acao:** deletar campanha/template, ativar campanha, submeter pra Meta
- **Cores/tema:** cinza neutro frio light, dark mode mais suave, contraste WCAG AA

### Subtipos de envio ativos (teste)

| Subtipo | Leads | Vertical |
|---------|-------|----------|
| ENVIO GRUPO SEMANA LIDERANCA | Fernando, Kremer, Kakau, Monica, Mateus (5) | SZS morno |
| ENVIO BILL | Bruno Benetti (1) | SZS morno |
| ENVIO MARIO | Mario Lopes (1) | SZS morno |
| ENVIO CRIS | Cristiane Bianchin (1) | SZS morno |

### Pendencias

**Bloqueios externos:**
- Credenciais Integration API Morada (MIA_TENANT_ID + MIA_API_KEY) — pra envio sem gateway
- Verificacao PIN numero +55 48 93618-2856 (expirado, precisa acesso fisico ao chip)
- Validacao schema por Mari e Roberto

**Desenvolvimento:**
- Smartsheet sync
- Cron de scoring automatico
- Integracao SIA/CIA
- Dashboard de funil (Pilar 4)
- Deploy (Coolify ou Vercel)

## Arquitetura

- **Backend:** FastAPI + SQLAlchemy async + asyncpg
- **Frontend:** Next.js 16 + React 19 + TanStack Query + Tailwind
- **DB:** PostgreSQL local
- **Envio WA:** Morada AI Gateway (`/send-notification`)
- **Templates Meta:** Graph API v20.0 (submit + status)
- **MCP:** FastMCP Python (17 tools, transport stdio)
- **Dados:** Nekt/Pipedrive + Sapron + Smartsheet

## Artefatos

| Arquivo | Descricao |
|---------|-----------|
| `sz-disparos-mkt/` | Monorepo frontend + backend |
| `sz-disparos-mkt/mcp/` | MCP Server (17 tools) |
| `db/schema.sql` | Schema 18 tabelas, 2 schemas |
| `index.html` | Spec visual do motor |
| `embasamento-cientifico-disparos.html` | Pesquisa 60+ fontes |
| `DEFINICOES_PARAMETRIZAVEIS.md` | Regras por vertical |
