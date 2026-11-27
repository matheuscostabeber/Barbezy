# STATE

## Decisions

### AD-001
- **Decision**: Produto é SaaS multi-tenant: um único site para todas as barbearias, tenant identificado por slug no caminho. App Expo na fase 2 (AD-008).
- **Reason**: Escala de receita e de aprendizado sem custo de N apps.
- **Trade-off**: Sem white-label; subdomínio/domínio próprio por barbearia só na fase 2.
- **Scope**: todo o produto
- **Date**: 2026-11-22 (revisado em 2026-11-26)
- **Status**: active

### AD-002
- **Decision**: Isolamento de tenant por `barbearia_id` em toda tabela de tenant + RLS no Postgres via `set_config('app.barbearia_id', id, true)` por transação, com `FORCE ROW LEVEL SECURITY`.
- **Reason**: Bug de aplicação não pode vazar dados entre barbearias.
- **Trade-off**: Toda query de tenant roda dentro de `tenant_session`; crons cross-tenant usam o role `app_admin` com BYPASSRLS auditado.
- **Scope**: backend/alembic, backend/app/core/tenancy.py
- **Date**: 2026-11-22 (revisado em 2026-11-26: `set_config` porque `SET LOCAL` não aceita parâmetro no asyncpg)
- **Status**: active

### AD-003
- **Decision**: Unicidade de horário garantida por exclusion constraint GiST no banco; Redis nunca é fonte da verdade.
- **Reason**: Única garantia correta sob concorrência real (NFR-01).
- **Trade-off**: Requer `btree_gist` e Postgres.
- **Scope**: agenda
- **Date**: 2026-11-22
- **Status**: active

### AD-004
- **Decision**: Eventos de domínio saem por transactional outbox → arq (Redis); efeitos externos (e-mail) só acontecem no worker.
- **Reason**: Nada é enviado sem commit e nada se perde se o Redis cair.
- **Trade-off**: Latência de ~1 s entre commit e envio.
- **Scope**: backend (todos os módulos)
- **Date**: 2026-11-22 (revisado em 2026-11-26: BullMQ → arq)
- **Status**: active

### AD-005
- **Decision**: Produção numa VPS Hostinger com datacenter no Brasil (mín. 2 vCPU / 4 GB), Docker Compose com Caddy, site, API, worker, Redis e PostgreSQL. DNS na Cloudflare. Backup diário `pg_dump` para Backblaze B2 ou Cloudflare R2, restauração testada todo mês. Ambientes: local + produção, deploy automático pelo GitHub Actions.
- **Reason**: Decisão do usuário em 2026-11-26. Um lugar só para cuidar; site e API no mesmo domínio; banco sempre acordado e sem limite de free tier. Substitui Vercel Hobby + Oracle Always Free + Neon.
- **Trade-off**: VPS única é ponto único de falha; backup e restauração passam a ser responsabilidade do projeto; custo mensal da VPS.
- **Scope**: infra, deploy
- **Date**: 2026-11-22 (reescrito em 2026-11-26)
- **Status**: active

### AD-006
- **Decision**: Um repositório com `backend/` (Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.0 assíncrono + asyncpg, Alembic, arq; gerenciado com uv) e `frontend/` (TypeScript, React + Vite, Tailwind + shadcn/ui; pnpm). `Makefile` na raiz. Sem Turborepo.
- **Reason**: Stack escolhida pelo usuário em 2026-11-26. Um repo mantém a mudança de API e o cliente gerado no mesmo commit.
- **Trade-off**: Duas linguagens; regras de negócio não são compartilhadas entre backend e frontend (ver AD-010).
- **Scope**: todo o repositório
- **Date**: 2026-11-22 (reescrito em 2026-11-26; substitui pnpm + Turborepo + NestJS + Next.js + Drizzle + Zod)
- **Status**: active

### AD-007
- **Decision**: Divisão por domínio vertical entre Mauricio, Lorenzo e Mateus.
- **Reason**: Três devs mexendo em web, app e API.
- **Trade-off**: n/a
- **Scope**: organização do time
- **Date**: 2026-11-22
- **Status**: superseded by AD-012

### AD-008
- **Decision**: MVP reduzido para uma pessoa: página pública, agendamento "pagar no local", painéis, login (e-mail OTP, Google, senha + TOTP do dono), lembretes por e-mail, resumo mensal, LGPD básica. Fase 2: app Expo, Stripe Connect, planos/Billing, WhatsApp, relatórios completos, painel em tempo real, Grafana, testes de volume, encerramento de barbearia, homologação.
- **Reason**: Pergunta da sessão de grill de 2026-11-26: "Q1 - Tamanho do MVP para uma pessoa", opção (b) "MVP do MVP" escolhida pelo usuário. Barbearias reais usam antes e o plano de rollout já começava com pagamento no local.
- **Trade-off**: Sem receita no MVP; cliente com faltas recorrentes não é obrigado a pagar online.
- **Scope**: spec.md (stories P3 preservam os critérios adiados)
- **Date**: 2026-11-26
- **Status**: active

### AD-009
- **Decision**: Sessão guardada no banco (hash SHA-256 de token aleatório) + cookie httpOnly, `Secure`, `SameSite=Lax`, 30 dias. Sem JWT no MVP.
- **Reason**: Site e API no mesmo domínio (AD-005); revogação trivial; mais simples de entender e proteger. JWT fazia sentido com o app mobile.
- **Trade-off**: Uma consulta ao banco por requisição autenticada. Tokens para o app entram na fase 2.
- **Scope**: backend/app/modules/auth
- **Date**: 2026-11-26
- **Status**: active

### AD-010
- **Decision**: Regras de negócio só no Python (`backend/app/domain`). O frontend mostra valores calculados pela API. Contratos: Pydantic → OpenAPI → cliente TS gerado com `@hey-api/openapi-ts`; CI falha se o cliente gerado divergir. A página `/{slug}` recebe meta tags Open Graph do FastAPI.
- **Reason**: Uma implementação e um conjunto de testes por regra; a prévia do link no WhatsApp funciona sem servidor Node.
- **Trade-off**: O site depende da API para exibir prazos e faixas; SEO da SPA é mais fraco que SSR.
- **Scope**: backend, frontend
- **Date**: 2026-11-26
- **Status**: active

### AD-011
- **Decision**: Login implementado no próprio FastAPI com bibliotecas conhecidas (argon2-cffi, pyotp, Authlib), testes de cada ataque conhecido desde a primeira task de auth e `/security-review` antes do lançamento (T65).
- **Reason**: Custo zero, sem dependência de fornecedor, e o usuário quer aprender. Supabase Auth/Clerk descartados.
- **Trade-off**: Responsabilidade de segurança fica no projeto.
- **Scope**: backend/app/modules/auth
- **Date**: 2026-11-26
- **Status**: active

### AD-012
- **Decision**: Projeto feito por uma pessoa, que está aprendendo FastAPI durante o projeto. Forma de trabalho mista: Claude escreve infra e configuração explicando; as partes centrais o usuário escreve com Claude guiando, ao menos a primeira de cada tipo. Fase 0 é um mini-projeto de aprendizado. Cada task diz quem escreve e o que o usuário aprende.
- **Reason**: Decisão do usuário em 2026-11-26 ("o projeto será feito somente por mim"; "quero nesse mesmo processo aprender").
- **Trade-off**: Execução mais lenta que Claude escrevendo tudo.
- **Scope**: tasks.md, execução
- **Date**: 2026-11-26
- **Status**: active

## Handoff

- **Feature**: `.specs/features/mvp`
- **Phase / Task**: Specs reescritos em 2026-11-26 para FastAPI + React/Vite, dev solo, VPS Hostinger e MVP reduzido. Aguardando aprovação do usuário para spec.md e tasks.md antes do Execute.
- **Completed**: none (nenhuma task de código)
- **In-progress** (file:line): none
- **Next step**: usuário aprova spec e tasks → T1 (mini-projeto FastAPI). Em paralelo, o usuário compra a VPS Hostinger e o domínio (necessários na T10).
- **Blockers**: none
- **Uncommitted files**: none
- **Branch**: `docs/stack-fastapi` (repo github.com/matheuscostabeber/Barbezy)
- **Desatualizado**: página "System design" do FigCraft ainda mostra a stack de 2026-11-22.
