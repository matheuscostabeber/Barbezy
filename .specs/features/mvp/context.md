# MVP Context

**Gathered:** 2026-11-22 (sessão de grill original) · **Revisado:** 2026-11-26 (sessão de grill de stack e escopo: dev solo, FastAPI + React/Vite, VPS Hostinger)
**Spec:** `.specs/features/mvp/spec.md`
**Status:** Ready for tasks

---

## Feature Boundary

SaaS multi-tenant de agendamento para barbearias. Cada barbearia (tenant) tem página pública própria, equipe, serviços, agenda e lembretes por e-mail. Produto único: web responsiva instalável (PWA) para cliente, barbeiro e dono. Desenvolvido e mantido por uma pessoa. Pagamento online, planos, WhatsApp e app mobile ficam para a fase 2.

---

## Implementation Decisions

### D1 · Modelo de produto
- SaaS multi-tenant. Sem marketplace/descoberta no MVP.
- Um único site para todas as barbearias; o tenant é identificado por slug no caminho: `<dominio>/{slug}`.
- Identidade visual por tenant na página pública: logo, capa e cor de destaque. Layout é o mesmo.
- Subdomínio / domínio próprio por barbearia: fase 2.

### D2 · Pagamento do cliente final (revisado em 2026-11-26)
- **MVP:** todo agendamento é "pagar no local". A página não mostra opção de pagamento online.
- **Fase 2:** modo configurável por barbearia ("Só no local" ou "Cliente escolhe"), Stripe Connect opcional, "Pagar agora" pré-selecionado, sinal (depósito). Critérios preservados em `spec.md`, story P3 "Pagamento online".

### D3 · Monetização da plataforma (adiada)
- **MVP:** sem cobrança. Toda barbearia usa tudo sem limite de barbeiros ou agendamentos, sem selo de plano.
- **Fase 2:** Free / Pro / Plus via Stripe Billing, como definido em 2026-11-22:

| | Free | Pro | Plus |
|---|---|---|---|
| Barbeiros ativos | 1 | até 5 | ilimitado |
| Agendamentos/mês | 20 | ilimitado | ilimitado |
| Página pública | com selo | sem selo | sem selo |
| Relatórios | mês atual | completo | completo + CSV |

### D4 · Inadimplência e downgrade (adiada)
- Fase 2, junto com D3: Smart Retries + 7 dias de carência, downgrade automático, dono escolhe barbeiros ativos (48 h), re-upgrade com um clique.

### D5 · Agendamento
- Fluxo: serviço → barbeiro (ou "qualquer um") → data/horário → identificação → confirmação.
- Slots calculados de jornada semanal + bloqueios − agendamentos, grade de **15 min**.
- Antecedência mínima **2 h**, máxima **30 dias** (configuráveis por barbearia).
- "Qualquer um" → barbeiro elegível com menos agendamentos no dia.
- Unicidade garantida no banco (exclusion constraint). Perdedor vê "Esse horário acabou de ser reservado" + sugestões.
- 1 serviço por agendamento; combos são serviços próprios.

### D6 · Cancelamento, remarcação, no-show
- Cliente cancela a qualquer momento antes do início, sem custo. Depois do início, recusado.
- Remarcar: só com antecedência ≥ janela (padrão **4 h**, configurável).
- No-show marcado pelo barbeiro até 24 h após o horário; no MVP só fica registrado.
- Cancelamento via link assinado (expira no início do atendimento) ou pela conta.
- Máquina de estados do MVP: `CONFIRMADO → CONCLUIDO | CANCELADO_CLIENTE | CANCELADO_BARBEARIA | NO_SHOW`. Os estados `PENDENTE_PAGAMENTO` e `EXPIRADO` entram na fase 2 com o pagamento online.

### D7 · Autenticação e papéis (revisado em 2026-11-26)
- Papéis: Cliente (global), Barbeiro (1 barbearia), Dono (N barbearias), Admin da plataforma. Papéis acumuláveis.
- Cliente: OTP de 6 dígitos por **e-mail** ou **Google**. Sem senha. WhatsApp e Sign in with Apple na fase 2.
- Dono/barbeiro: e-mail + senha (argon2id) ou Google. **2FA TOTP obrigatório para o dono.**
- Barbeiro entra por convite (link 7 dias).
- Sessão: registro no banco + cookie httpOnly `Secure` `SameSite=Lax`, 30 dias; revogação pelo painel. Sem JWT no MVP. Tokens entram na fase 2, só para o app.
- Rate limit: 5 tentativas / 15 min por identificador e por IP.
- Isolamento: `barbearia_id` em toda query + RLS no Postgres.
- Login implementado no próprio FastAPI, com bibliotecas conhecidas, testes de segurança desde o início e revisão de segurança antes do lançamento.
- Sem agendamento como convidado.

### D8 · Notificações (revisado em 2026-11-26)
| Evento | Cliente | Barbeiro | Dono |
|---|---|---|---|
| Agendamento confirmado | e-mail (com link de cancelamento) | painel | — |
| Lembrete 24 h e 2 h antes | e-mail | — | — |
| Cancelado/remarcado | e-mail | painel + e-mail | — |
- Opt-out de lembretes por link no e-mail; não desliga confirmações.
- Fila assíncrona (arq), até 3 tentativas; falha de notificação nunca afeta o agendamento.
- Painel atualiza por polling a cada 30 s.
- Sem marketing. WhatsApp e push na fase 2.

### D9 · Plataformas (revisado em 2026-11-26)
- Só web responsiva (React + Vite), instalável como PWA. App Expo na fase 2.
- A página `/{slug}` recebe meta tags Open Graph preenchidas pelo FastAPI para a prévia do link no WhatsApp/Instagram. O resto do site é SPA.

### D10 · Dinheiro do cliente
- MVP: não há dinheiro passando pela plataforma. Ao concluir, o barbeiro informa a forma (dinheiro, Pix, cartão na maquininha) só para o resumo.
- Fase 2: Stripe Connect Express, como definido em 2026-11-22.

### D11 · Serviços e preços
- Serviço: nome, descrição, duração, preço base, foto opcional, categoria (Cabelo, Barba, Combo, Outros).
- Override de preço e duração por barbeiro; dono define quais barbeiros fazem cada serviço.
- "Qualquer um" mostra faixa de preço até atribuição; preço final antes de confirmar.
- Preço congelado no agendamento.
- Serviço desativado some da página; agendamentos futuros mantidos; sem hard delete com histórico.

### D12 · Resumo mensal (revisado em 2026-11-26)
- MVP: contagem por status, faturamento no local (soma de `CONCLUIDO`) e taxa de no-show do mês.
- Barbeiro vê apenas os próprios números.
- Fase 2: ocupação, comissão, filtros, ranking, recorrentes × novos, CSV.

### D13 · LGPD
- Plataforma = controladora dos dados de conta; barbearia = controladora dos dados de atendimento, plataforma = operadora.
- Bases legais: execução de contrato (transacional), legítimo interesse com opt-out (lembretes).
- Exclusão de conta: anonimização em até 15 dias; histórico mantido anonimizado; futuros cancelados.
- Exportação de dados: JSON assíncrono por e-mail.
- Retenção: atendimentos 5 anos; logs de acesso 6 meses; OTP 10 min.
- Minimização: sem CPF/nascimento/endereço do cliente.
- Audit log de ações sensíveis.
- Encerramento de barbearia com janela de 90 dias: fase 2.

### D14 · Metas não-funcionais (revisado em 2026-11-26)
- MVP prova a regra mais importante: 20 disputas pelo mesmo slot → exatamente 1 vence (k6 `contention.js`).
- Backup diário fora da VPS (RPO 24 h); restauração testada todo mês.
- Erros da API reportados no Sentry.
- Metas de volume (2.000 usuários simultâneos, p95 < 300 ms, 50 agendamentos/s, 99,5%) voltam na fase 2, quando houver tráfego que justifique.

### D15 · Stack e infraestrutura (revisado em 2026-11-26)
- **Backend:** Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.0 assíncrono + asyncpg, Alembic, arq (fila sobre Redis). Gerenciado com **uv**; ruff + mypy; pytest.
- **Frontend:** TypeScript, React + Vite, React Router, TanStack Query, Tailwind CSS + shadcn/ui (tokens do FigCraft), cliente HTTP gerado do OpenAPI com `@hey-api/openapi-ts`; Vitest + Playwright.
- **Regras de negócio** só no Python (`backend/app/domain`). O site mostra o que a API calcula.
- **Repositório:** um repo com `backend/` e `frontend/`, `Makefile` na raiz. Sem Turborepo.
- **Infra:** 1 VPS Hostinger com datacenter no Brasil (mín. 2 vCPU / 4 GB), Docker Compose com Caddy (TLS), site estático, API, worker arq, Redis 7 e PostgreSQL 16. DNS na Cloudflare (grátis). Domínio próprio a comprar.
- **Backup:** `pg_dump` diário para Backblaze B2 ou Cloudflare R2; teste de restauração mensal.
- **Ambientes:** local (Docker Compose) + produção. Deploy automático pelo GitHub Actions ao juntar na `main`.
- **Serviços externos:** Resend (e-mail), Google OAuth, Sentry (erros).
- **Custos:** VPS Hostinger (mensal), domínio (anual). O resto usa faixa gratuita.

### D16 · Equipe e forma de trabalho (novo em 2026-11-26)
- Projeto feito por uma pessoa: o usuário. A divisão por domínio entre três pessoas foi removida.
- O usuário está aprendendo FastAPI durante o projeto. Forma de trabalho mista: Claude faz infra e configuração; as partes centrais (rotas, regras de agendamento, banco) o usuário escreve com Claude guiando, ao menos a primeira de cada tipo.
- Fase 0 do plano é um mini-projeto de aprendizado.

### Agent's Discretion
- Grade de 15 min, prazos LGPD, metas de NFR: propostos pelo agente e aceitos.
- Detalhes de UI (layout, componentes) seguem o design no FigCraft e o design system.

### Declined / Undiscussed Gray Areas → Assumptions
Registrados na seção Assumptions & Open Questions da spec: fuso horário, múltiplas unidades, onboarding e slug, escopo do admin, idioma/moeda, cancelamento após o início.

---

## Specific References

- Booksy / Trinks como referência de "um app para N estabelecimentos".
- Estrutura de páginas FigCraft (Design system, Desktop · Light/Dark, Mobile · Light/Dark) em `design/barber-app.figcraft`.

---

## Deferred Ideas

- **Fase 2 (decidida em 2026-11-26):** app Expo + push + Sign in with Apple; Stripe Connect e pagamento online; planos e assinatura (Stripe Billing); WhatsApp (OTP e lembretes); relatórios completos, comissão e CSV; painel em tempo real; Grafana/OpenTelemetry; testes de carga de volume; encerramento de barbearia; ambiente de homologação.
- Marketplace / busca de barbearias por localização.
- Sinal (depósito) como modo de pagamento.
- Comissão da plataforma por transação.
- Subdomínio / domínio próprio por barbearia.
- Produtos, pacotes/assinatura do cliente, cupons, gorjeta.
- Split automático para o barbeiro.
- NFS-e.
- Marketing (WhatsApp/e-mail) com consentimento.
- App white-label por barbearia.
- Múltiplos serviços por agendamento.
