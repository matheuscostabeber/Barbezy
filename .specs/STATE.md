# STATE

## Decisions

### AD-001
- **Decision**: Produto é SaaS multi-tenant: um único app (web + Expo) para todas as barbearias, tenant identificado por slug no caminho.
- **Reason**: Escala de receita e de aprendizado (Stripe Connect, isolamento) sem custo de N apps.
- **Trade-off**: Sem white-label nas lojas; subdomínio/domínio próprio só na fase 2.
- **Scope**: todo o produto
- **Date**: 2026-11-22
- **Status**: active

### AD-002
- **Decision**: Isolamento de tenant por `barbearia_id` em toda tabela de tenant + RLS no Postgres via `SET LOCAL app.barbearia_id` por transação.
- **Reason**: Bug de aplicação não pode vazar dados entre barbearias.
- **Trade-off**: Toda query de tenant precisa rodar dentro de `withTenant`; queries cross-tenant (admin, crons) usam role dedicado com BYPASSRLS auditado.
- **Scope**: packages/db, apps/api
- **Date**: 2026-11-22
- **Status**: active

### AD-003
- **Decision**: Unicidade de horário garantida por exclusion constraint GiST no banco; Redis nunca é fonte da verdade.
- **Reason**: Única garantia correta sob concorrência real (NFR-01).
- **Trade-off**: Requer `btree_gist` e Postgres (sem portabilidade para outro banco).
- **Scope**: agenda
- **Date**: 2026-11-22
- **Status**: active

### AD-004
- **Decision**: Eventos de domínio saem por transactional outbox → BullMQ; efeitos externos (WhatsApp, e-mail, push, Stripe expire) só acontecem em workers.
- **Reason**: Nada é enviado sem commit e nada se perde se o Redis cair.
- **Trade-off**: Latência de ~1 s entre commit e envio.
- **Scope**: apps/api (todos os módulos)
- **Date**: 2026-11-22
- **Status**: active

### AD-005
- **Decision**: Infra custo zero: Vercel Hobby (web), VM Oracle Always Free com Docker Compose (API, worker, Redis, Caddy), Neon Free (Postgres). Sem AWS.
- **Reason**: Pedido do Mauricio ("0800").
- **Trade-off**: Sem SLA, VM única, cold start do Neon, Vercel não-comercial; migração por gatilho documentada em rollout.md.
- **Scope**: infra, deploy
- **Date**: 2026-11-22
- **Status**: active

### AD-006
- **Decision**: Monorepo pnpm + Turborepo, TypeScript ponta a ponta; regras de negócio puras em `packages/domain`, contratos Zod em `packages/contracts`.
- **Reason**: 3 devs mexendo em web, app e API sem duplicar regra nem tipo.
- **Trade-off**: Todo o time preso a TS/Node.
- **Scope**: todo o repositório
- **Date**: 2026-11-22
- **Status**: active

### AD-007
- **Decision**: Divisão por domínio vertical: Mauricio (agenda, página pública, design system, app), Lorenzo (pagamentos, planos, relatórios), Mateus (identidade, notificações, LGPD, infra).
- **Reason**: Cada um entrega back + front da sua área; menos handoff.
- **Trade-off**: Revisão cruzada obrigatória para manter padrões comuns.
- **Scope**: organização do time
- **Date**: 2026-11-22
- **Status**: active

## Handoff

- **Feature**: `.specs/features/mvp`
- **Phase / Task**: Specify + Design + Tasks + Rollout escritos; telas e system design no FigCraft (`design/barber-app.figcraft`, 6 páginas); aguardando aprovação do Mauricio antes do Execute
- **Completed**: none (nenhuma task de código)
- **In-progress** (file:line): none
- **Next step**: Mauricio aprova spec/design/tasks → iniciar T1 (bootstrap do monorepo) e T2/T3 (VM Oracle e verificação Meta) no mesmo dia
- **Blockers**: none
- **Uncommitted files**: none
- **Branch**: `docs/mvp-specs` (repo github.com/matheuscostabeber/CEO-Barbers)
