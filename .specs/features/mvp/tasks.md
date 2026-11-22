# MVP Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, adequacy review, Verifier, discrimination sensor).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/mvp/design.md`
**Status**: Draft

**Donos por domínio (AD-007):** 🟢 Mauricio · 🔵 Lorenzo · 🟣 Mateus. Cada um executa as tasks marcadas para si; revisão cruzada obrigatória antes do MR.

**Branching:** `feat/<fase>-<slug>` a partir de `dev`; uma task = um commit Conventional Commits em inglês; `.specs/` é versionado neste repositório e atualizado no mesmo commit da task.

**Tools (padrão para todas as tasks):** MCP `figcraft` para consultar telas nas tasks de UI (Phases 10–12); demais tasks sem MCP. Skill: `tlc-spec-driven`; tasks de LGPD (Phase 9) consultam as skills `lgpd-*` disponíveis.

---

## Test Coverage Matrix

> Repositório vazio — nenhum guideline encontrado; strong defaults aplicados. **Tipos de teste e comandos são proposta: confirmar na aprovação das tasks.**

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Domain (`packages/domain`) | unit | Todos os ramos; 1:1 com ACs da spec; todos os edge cases listados | `packages/domain/src/**/*.test.ts` | `pnpm turbo run test:unit --filter=@barber/domain` |
| Contracts (`packages/contracts`) | unit | Casos válidos + cada regra de rejeição | `packages/contracts/src/**/*.test.ts` | `pnpm turbo run test:unit --filter=@barber/contracts` |
| API service/processor | integration | Caminho feliz + cada AC + falhas externas (Postgres e Redis reais via Testcontainers; Stripe via stripe-mock) | `apps/api/src/**/*.int.test.ts` | `pnpm turbo run test:int --filter=@barber/api` |
| API controller | integration | Todas as rotas: feliz + edge + erro + isolamento de tenant (404) | `apps/api/src/**/*.int.test.ts` | `pnpm turbo run test:int --filter=@barber/api` |
| API adapter puro (WhatsApp, e-mail, push, link assinado) | unit | Formatação, assinatura, tratamento de erro do provedor | `apps/api/src/**/*.test.ts` | `pnpm turbo run test:unit --filter=@barber/api` |
| DB schema/migrations/RLS | integration | Migration aplica; constraints e policies se comportam como a spec | `packages/db/test/**/*.int.test.ts` | `pnpm turbo run test:int --filter=@barber/db` |
| UI kit (`packages/ui`) | unit | Estados e variantes; contraste dos tokens ≥ 4.5:1 | `packages/ui/src/**/*.test.tsx` | `pnpm turbo run test:unit --filter=@barber/ui` |
| Web page/flow | e2e | Fluxos da spec: feliz + erros visíveis ao usuário | `apps/web/e2e/**/*.spec.ts` | `pnpm turbo run test:e2e --filter=@barber/web` |
| Mobile screen | unit | Renderização + interações + chamadas de API mockadas | `apps/mobile/**/*.test.tsx` | `pnpm turbo run test:unit --filter=@barber/mobile` |
| Load | load (k6) | Cenários NFR-01..03 com thresholds | `load/*.js` | `k6 run load/<cenario>.js` |
| Config/infra/docs | none | build gate only | - | build gate only |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Tasks com apenas unit | `pnpm turbo run lint typecheck test:unit --filter=...[HEAD^]` |
| Full | Tasks com integration/e2e | `pnpm turbo run lint typecheck test:unit test:int test:e2e --filter=...[HEAD^]` |
| Build | Fim de fase, config/infra | `pnpm turbo run lint typecheck build test:unit test:int` |

---

## Execution Plan

Fases em ordem. Dentro de uma fase, as tasks executam na ordem dos diagramas. As Phases 2–9 têm donos diferentes e, após a Phase 1, podem andar em paralelo entre pessoas respeitando as dependências cross-phase declaradas em cada task.

| Phase | Tema | Dono principal | Tasks |
| --- | --- | --- | --- |
| 1 | Fundação | 🟣 Mateus | T1–T9 |
| 2 | Domínio puro | 🟢 Mauricio / 🔵 Lorenzo | T10–T17 |
| 3 | Identidade | 🟣 Mateus | T18–T24 |
| 4 | Catálogo | 🟢 Mauricio | T25–T30 |
| 5 | Agenda | 🟢 Mauricio | T31–T37 |
| 6 | Pagamentos | 🔵 Lorenzo | T38–T46 |
| 7 | Planos | 🔵 Lorenzo | T47–T51 |
| 8 | Notificações | 🟣 Mateus | T52–T58 |
| 9 | Relatórios & LGPD | 🔵 Lorenzo / 🟣 Mateus | T59–T64 |
| 10 | Design system & web pública | 🟢 Mauricio | T65–T71 |
| 11 | Web painel | todos | T72–T79 |
| 12 | App Expo | 🟢 Mauricio | T80–T85 |
| 13 | Release | 🟣 Mateus | T86–T89 |

---

## Task Breakdown

### Phase 1: Fundação

```
T1 -> T4
T1 -> T5 -> T6 -> T7
T1 -> T8
T1 -> T9
T2 (externo)   T3 (externo)
```

#### T1: Bootstrap do monorepo 🟣
**What**: pnpm workspace + Turborepo com `apps/{web,mobile,api}` e `packages/{contracts,db,domain,ui,config}` vazios compilando, scripts `lint typecheck build test:unit test:int test:e2e`.
**Where**: `turbo.json`
**Depends on**: None
**Requirement**: AD-006
**Done when**:
- [ ] `pnpm install && pnpm turbo run build lint typecheck` passa no repo vazio
**Tests**: none
**Gate**: build

#### T2: Provisionar VM Oracle Always Free 🟣
**What**: Conta Oracle (home region São Paulo), VM Ampere A1, firewall 80/443, Docker + Compose, Caddy respondendo `/health`.
**Where**: `infra/docker-compose.prod.yml`
**Depends on**: None
**Requirement**: AD-005
**Done when**:
- [ ] `curl https://<host>/health` → 200 com TLS válido
- [ ] Recursos reais da VM (OCPU/RAM) anotados no STATE.md
**Tests**: none
**Gate**: build

#### T3: Iniciar verificação Meta + templates WhatsApp 🟣
**What**: Meta Business verificado, número dedicado, templates `auth_code` (authentication), `agendamento_confirmado`, `lembrete_24h`, `lembrete_2h`, `agendamento_cancelado` (utility) submetidos.
**Where**: `docs/whatsapp-templates.md`
**Depends on**: None
**Requirement**: AUTH-01, NOT-01, NOT-02
**Done when**:
- [ ] Templates submetidos; status registrado no doc
**Tests**: none
**Gate**: build

#### T4: Ambiente local com Docker Compose 🟣
**What**: Postgres 16 (com `btree_gist`), Redis 7 (AOF), Mailpit, stripe-mock, stripe-cli listen.
**Where**: `infra/docker-compose.yml`
**Depends on**: T1
**Requirement**: AD-005
**Done when**:
- [ ] `docker compose up` sobe tudo saudável
**Tests**: none
**Gate**: build

#### T5: Schema base de tenancy 🟣
**What**: Drizzle + migrations; tabelas `barbearia` (campos mínimos) e `membro`; role `app_user` sem BYPASSRLS e `app_admin` com BYPASSRLS.
**Where**: `packages/db/src/schema/tenancy.ts`
**Depends on**: T1
**Requirement**: AUTH-10
**Done when**:
- [ ] Migration aplica em banco limpo (Testcontainers)
**Tests**: integration
**Gate**: full

#### T6: Policies RLS + teste-guardião 🟣
**What**: Policies RLS para tabelas de tenant; teste que falha se qualquer tabela com coluna `barbearia_id` não tiver RLS habilitada e policy.
**Where**: `packages/db/sql/rls.sql`
**Depends on**: T5
**Requirement**: AUTH-10
**Done when**:
- [ ] Com `app.barbearia_id = A`, `SELECT` em membro não retorna linhas de B
- [ ] Teste-guardião falha ao criar tabela de teste sem policy
**Tests**: integration
**Gate**: full

#### T7: withTenant + TenantGuard 🟣
**What**: Helper transacional com `SET LOCAL` e guard que resolve tenant por rota/slug e responde 404 sem vínculo.
**Where**: `apps/api/src/tenancy/tenant.guard.ts`
**Depends on**: T6
**Requirement**: AUTH-10, AUTH-11
**Done when**:
- [ ] Usuário de A pedindo recurso de B → 404
**Tests**: integration
**Gate**: full

#### T8: CI 🟣
**What**: GitHub Actions rodando o gate Build em PR para `dev`/`main`, com cache do Turborepo.
**Where**: `.github/workflows/ci.yml`
**Depends on**: T1
**Requirement**: AD-006
**Done when**:
- [ ] PR de teste roda verde
**Tests**: none
**Gate**: build

#### T9: Observabilidade da API 🟣
**What**: OpenTelemetry (traces HTTP, pg, ioredis, BullMQ) + Sentry; propaga `traceparent` recebido.
**Where**: `apps/api/src/observability/otel.ts`
**Depends on**: T1
**Requirement**: NFR-06
**Done when**:
- [ ] Request com `traceparent` gera span filho com o mesmo trace id (exporter em memória)
**Tests**: integration
**Gate**: full

### Phase 2: Domínio puro

```
T11 -> T12
T10 -> T13
```

#### T10: Máquina de estados do agendamento 🟢
**What**: `transicionar(status, evento)` com tabela fechada; lança `TransicaoInvalida`.
**Where**: `packages/domain/src/estado.ts`
**Depends on**: T1
**Requirement**: AGD-13, AGD-14
**Done when**:
- [ ] Cada transição permitida testada; todas as outras 7×N combinações lançam
**Tests**: unit
**Gate**: quick

#### T11: Cálculo de slots 🟢
**What**: Função pura de slots (faixas, bloqueios, ocupados, duração, fuso, antecedências, grade 15 min).
**Where**: `packages/domain/src/slots.ts`
**Depends on**: T1
**Requirement**: AGD-01, AGD-02
**Done when**:
- [ ] Casos: intervalo de almoço, bloqueio parcial, serviço maior que o restante da jornada, troca de dia no fuso, antecedência mín/máx, horário de verão histórico
**Tests**: unit
**Gate**: quick

#### T12: Atribuição "qualquer um" e faixa de preço 🟢
**What**: Escolhe barbeiro elegível com menos ocupantes no dia (desempate por id) e calcula faixa mín–máx com overrides.
**Where**: `packages/domain/src/atribuicao.ts`
**Depends on**: T11
**Requirement**: AGD-03, AGD-04, SRV-02
**Done when**:
- [ ] Empate resolvido por id; faixa considera só elegíveis e overrides
**Tests**: unit
**Gate**: quick

#### T13: Política de cancelamento, remarcação e no-show 🟢
**What**: Decide reembolso/permite remarcar/permite no-show a partir de agora, início, janela e modo de pagamento.
**Where**: `packages/domain/src/politica-cancelamento.ts`
**Depends on**: T10
**Requirement**: CAN-01, CAN-02, CAN-03, CAN-04, CAN-05, CAN-06, CAN-07, CAN-08
**Done when**:
- [ ] Fronteiras exatas: janela −1 min / exata / +1 min; no-show antes do início e em 24 h + 1 min
**Tests**: unit
**Gate**: quick

#### T14: Formas de pagamento permitidas 🟢
**What**: `formasPermitidas(modoBarbearia, noShows90d)` → `so_local` = `['local']`; `cliente_escolhe` = `['agora','local']`, ou só `['agora']` com 2+ faltas; padrão pré-selecionado `agora`.
**Where**: `packages/domain/src/formas-pagamento.ts`
**Depends on**: T1
**Requirement**: CAN-09, AGD-15, AGD-16
**Done when**:
- [ ] 1 falta → as duas formas; 2 faltas → só `agora`; falta de 91 dias atrás não conta; `so_local` com 2 faltas → `['local']`
**Tests**: unit
**Gate**: quick

#### T15: Limites de plano 🔵
**What**: Tabela de limites Free/Pro/Plus; `podeAtivarBarbeiro`, `podeCriarAgendamento`, limiares 80/100%, regra de contagem (exclui EXPIRADO).
**Where**: `packages/domain/src/planos.ts`
**Depends on**: T1
**Requirement**: TEN-07, TEN-08, PLN-02, PLN-03, PLN-04, AGD-12
**Done when**:
- [ ] Free: 2º barbeiro negado; 21º agendamento negado; avisos em 16 e 20
**Tests**: unit
**Gate**: quick

#### T16: Métricas e comissão 🔵
**What**: Faturamento online×local, taxa de no-show, ocupação, comissão sobre preço congelado de CONCLUIDO.
**Where**: `packages/domain/src/metricas.ts`
**Depends on**: T1
**Requirement**: REL-01, REL-02, REL-03, REL-04
**Done when**:
- [ ] Fixture da spec (10×R$50 + 2 no-show, 40%) → comissão R$ 200,00 e no-show 16,7%
**Tests**: unit
**Gate**: quick

#### T17: Contratos de catálogo 🟢
**What**: Schemas Zod de barbearia (slug, cor), serviço (duração múltipla de 15 entre 15–240, preço mínimo online) e jornada.
**Where**: `packages/contracts/src/catalogo.ts`
**Depends on**: T1
**Requirement**: TEN-02, SRV-01, SRV-08
**Done when**:
- [ ] Slug reservado/curto/maiúsculo rejeitado; duração 20 rejeitada; R$ 4,99 com online rejeitado
**Tests**: unit
**Gate**: quick

### Phase 3: Identidade

```
T18 -> T19 -> T23
T18 -> T20 -> T21 -> T23
T20 -> T22 -> T23
T23 -> T24
```

#### T18: Schema de identidade 🟣
**What**: Tabelas `usuario`, `identidade`, `sessao`, `otp`.
**Where**: `packages/db/src/schema/auth.ts`
**Depends on**: T5
**Requirement**: AUTH-01, AUTH-07
**Done when**:
- [ ] Migration aplica; unique(provedor, sub)
**Tests**: integration
**Gate**: full

#### T19: Serviço de OTP 🟣
**What**: Gera código de 6 dígitos (hash), expira em 10 min, uso único, rate limit 5/15 min por destino e IP em Redis.
**Where**: `apps/api/src/auth/otp.service.ts`
**Depends on**: T18
**Requirement**: AUTH-01, AUTH-03, AUTH-04
**Done when**:
- [ ] 6ª tentativa → 429; código usado → inválido; 10 min + 1 s → expirado
**Tests**: integration
**Gate**: full

#### T20: Sessões e refresh rotativo 🟣
**What**: Access 15 min (JWT EdDSA), refresh 30 dias com família; reuso revoga família; revogação manual.
**Where**: `apps/api/src/auth/session.service.ts`
**Depends on**: T18
**Requirement**: AUTH-07, AUTH-08, AUTH-09
**Done when**:
- [ ] Refresh antigo reapresentado → família revogada; sessão revogada → 401
**Tests**: integration
**Gate**: full

#### T21: Senha + TOTP obrigatório para dono 🟣
**What**: argon2id; login de dono sem TOTP força setup; com TOTP exige código.
**Where**: `apps/api/src/auth/password-totp.service.ts`
**Depends on**: T20
**Requirement**: AUTH-05, AUTH-06
**Done when**:
- [ ] Dono sem 2FA não recebe tokens de painel; com código errado → 401
**Tests**: integration
**Gate**: full

#### T22: OAuth Google e Apple 🟣
**What**: Verificação de id_token Google/Apple e vínculo por `identidade`.
**Where**: `apps/api/src/auth/oauth.service.ts`
**Depends on**: T20
**Requirement**: AUTH-13
**Done when**:
- [ ] Token com audience errada → 401; primeiro login cria usuário; segundo reutiliza
**Tests**: integration
**Gate**: full

#### T23: Controller de auth 🟣
**What**: Rotas `/auth/*` e `/me/sessoes` conforme design.
**Where**: `apps/api/src/auth/auth.controller.ts`
**Depends on**: T19, T21, T22
**Requirement**: AUTH-01, AUTH-03, AUTH-05, AUTH-07, AUTH-09
**Done when**:
- [ ] Todas as rotas: feliz + 400 + 401 + 429
**Tests**: integration
**Gate**: full

#### T24: Guard de papéis 🟣
**What**: RBAC dono/barbeiro/cliente/admin; barbeiro sem faturamento/config/plano/Stripe; admin sem pagamentos/PII.
**Where**: `apps/api/src/auth/roles.guard.ts`
**Depends on**: T23
**Requirement**: AUTH-12, AUTH-14
**Done when**:
- [ ] Matriz papel × rota sensível testada
**Tests**: integration
**Gate**: full

### Phase 4: Catálogo

```
T25 -> T26 -> T27 -> T30
T25 -> T28 -> T30
T25 -> T29 -> T30
T26 -> T30
```

#### T25: Schema de catálogo 🟢
**What**: Campos completos de `barbearia`, `slug_antigo`, `convite`, `faixa_jornada`, `bloqueio`, `servico`, `servico_membro` + RLS.
**Where**: `packages/db/src/schema/catalogo.ts`
**Depends on**: T6
**Requirement**: TEN-01, SRV-01
**Done when**:
- [ ] Teste-guardião de RLS (T6) verde com as novas tabelas
**Tests**: integration
**Gate**: full

#### T26: Barbearia, slug e identidade visual 🟢
**What**: Onboarding cria barbearia Free; troca de slug com limite de 30 dias e redirect 301 por 12 meses; logo/capa/cor.
**Where**: `apps/api/src/catalogo/barbearia.service.ts`
**Depends on**: T25, T17
**Requirement**: TEN-01, TEN-02, TEN-03, TEN-04, TEN-10, TEN-11
**Done when**:
- [ ] Slug duplicado → "Endereço indisponível"; 2ª troca em 29 dias → recusada; slug antigo → 301
**Tests**: integration
**Gate**: full

#### T27: Equipe e convites 🟢
**What**: Convite com token de 7 dias, aceite cria membro, ativação respeita limite do plano.
**Where**: `apps/api/src/catalogo/membros.service.ts`
**Depends on**: T26, T15
**Requirement**: TEN-05, TEN-06, TEN-07, TEN-08
**Done when**:
- [ ] Convite expirado/usado → "Convite inválido"; Free com 1 ativo nega 2º
**Tests**: integration
**Gate**: full

#### T28: Jornadas e bloqueios 🟢
**What**: CRUD de faixas semanais (sem sobreposição) e bloqueios.
**Where**: `apps/api/src/catalogo/agenda-config.service.ts`
**Depends on**: T25
**Requirement**: TEN-09
**Done when**:
- [ ] Faixas sobrepostas → 400
**Tests**: integration
**Gate**: full

#### T29: Serviços e overrides 🟢
**What**: CRUD de serviço, habilitação e override por barbeiro, desativação, exclusão só sem agendamentos.
**Where**: `apps/api/src/catalogo/servicos.service.ts`
**Depends on**: T25, T17
**Requirement**: SRV-01, SRV-02, SRV-03, SRV-06, SRV-07, SRV-08
**Done when**:
- [ ] Serviço desativado some de `/public/:slug`; exclusão com histórico → 409 sugerindo desativar
**Tests**: integration
**Gate**: full

#### T30: Controllers de catálogo 🟢
**What**: Rotas `/barbearias/:id/*` e `GET /public/:slug` com guards.
**Where**: `apps/api/src/catalogo/catalogo.controller.ts`
**Depends on**: T26, T27, T28, T29, T24
**Requirement**: TEN-01, AUTH-11, AUTH-12
**Done when**:
- [ ] Todas as rotas: feliz + validação + 404 cross-tenant + 403 por papel
**Tests**: integration
**Gate**: full

### Phase 5: Agenda

```
T31 -> T32 -> T33 -> T34 -> T35 -> T36
T31 -> T37
```

#### T31: Schema de agenda 🟢
**What**: `agendamento` com `tstzrange`, exclusion constraint filtrada por status ocupante, `UNIQUE(barbearia_id, idempotency_key)`, `uso_mensal`, `outbox`.
**Where**: `packages/db/src/schema/agenda.ts`
**Depends on**: T25
**Requirement**: AGD-05
**Done when**:
- [ ] INSERT sobreposto do mesmo barbeiro em status ocupante → 23P01; em CANCELADO → aceito
**Tests**: integration
**Gate**: full

#### T32: Serviço de slots com cache 🟢
**What**: Carrega faixas/bloqueios/ocupados, chama `domain/slots`, cacheia com chave versionada e invalida por `INCR`.
**Where**: `apps/api/src/agenda/slots.service.ts`
**Depends on**: T31, T11, T12
**Requirement**: AGD-01, AGD-02, AGD-12, NFR-04
**Done when**:
- [ ] Novo agendamento invalida o cache do barbeiro/dia; Free no limite → slots vazios + flag `limiteAtingido`
**Tests**: integration
**Gate**: full

#### T33: Criar agendamento 🟢
**What**: Limpeza de holds vencidos, lock de `uso_mensal`, preço/duração congelados, idempotência, 409 com 3 sugestões; valida `forma` com `formasPermitidas` (422 fora da lista): `local` → CONFIRMADO, `agora` → PENDENTE_PAGAMENTO.
**Where**: `apps/api/src/agenda/criar-agendamento.service.ts`
**Depends on**: T32, T14, T15
**Requirement**: AGD-05, AGD-06, AGD-07, AGD-08, AGD-09, AGD-11, AGD-16, SRV-04, SRV-05, PLN-04, CAN-09
**Done when**:
- [ ] Mesma Idempotency-Key 2× → mesmo agendamento; 21º no Free → 403; conflito → 409 + sugestões
- [ ] `local` com 2 faltas → 422; `agora` em barbearia `so_local` → 422
**Tests**: integration
**Gate**: full

#### T34: Transições do agendamento 🟢
**What**: Cancelar (cliente/barbearia), remarcar, concluir (forma de pagamento local obrigatória), no-show; grava outbox; reembolso via porta `PaymentGateway` (mock nesta fase).
**Where**: `apps/api/src/agenda/transicoes.service.ts`
**Depends on**: T33, T13
**Requirement**: CAN-01, CAN-02, CAN-03, CAN-04, CAN-05, CAN-06, CAN-07, CAN-08, REL-05
**Done when**:
- [ ] Cada AC de CAN com chamada (ou não) ao gateway verificada
**Tests**: integration
**Gate**: full

#### T35: Link assinado de cancelamento 🟢
**What**: HMAC com `agendamentoId` + expiração no início do atendimento.
**Where**: `apps/api/src/agenda/link-cancelamento.ts`
**Depends on**: T34
**Requirement**: CAN-10, CAN-11
**Done when**:
- [ ] Assinatura adulterada ou após início → "Link expirado"
**Tests**: unit
**Gate**: quick

#### T36: Controllers de agenda 🟢
**What**: Rotas de slots, criação, transições e link de cancelamento.
**Where**: `apps/api/src/agenda/agenda.controller.ts`
**Depends on**: T35
**Requirement**: NFR-01, AGD-06, AGD-14
**Done when**:
- [ ] Teste com 20 requisições concorrentes ao mesmo slot → 1×201 e 19×409, 1 linha no banco
**Tests**: integration
**Gate**: full

#### T37: Bloqueio sobre agendamentos existentes 🟢
**What**: Criar bloqueio que cobre agendamentos ocupantes é recusado com a lista dos conflitantes.
**Where**: `apps/api/src/catalogo/agenda-config.service.ts`
**Depends on**: T31, T28
**Requirement**: TEN-09
**Done when**:
- [ ] Bloqueio sobre 2 agendamentos → 409 com os 2 ids
**Tests**: integration
**Gate**: full

### Phase 6: Pagamentos

```
T38 -> T39 -> T42 -> T43
T41 -> T42 -> T44
T39 -> T40
T39 -> T45
T39 -> T46
```

#### T38: Spike — tipo de cobrança Connect Express 🔵
**What**: Ler a doc atual do Stripe e decidir entre direct charge e destination charge com `on_behalf_of` para que a taxa fique com a barbearia (D10) e definir quem responde por saldo negativo; registrar como ADR e AD-008.
**Where**: `docs/adr/0001-stripe-connect-charge-type.md`
**Depends on**: T1
**Requirement**: PAG-03
**Done when**:
- [ ] ADR com fontes e decisão; AD-008 no STATE.md
**Tests**: none
**Gate**: build

#### T39: Porta PaymentGateway + StripeGateway 🔵
**What**: Interface e implementação Stripe (`criarCheckout`, `expirarCheckout`, `reembolsar`, `contaPronta`) conforme ADR.
**Where**: `apps/api/src/pagamentos/stripe.gateway.ts`
**Depends on**: T38
**Requirement**: PAG-03, PAG-08
**Done when**:
- [ ] Contra stripe-mock: parâmetros corretos; nenhum dado de cartão trafega pela API
**Tests**: integration
**Gate**: full

#### T40: Onboarding Connect 🔵
**What**: Cria conta Express, Account Link, sincroniza `charges_enabled` via webhook `account.updated`; perda de `charges_enabled` volta a barbearia para `so_local` e alerta o dono.
**Where**: `apps/api/src/pagamentos/connect.service.ts`
**Depends on**: T39
**Requirement**: PAG-01, PAG-02
**Done when**:
- [ ] Sem `charges_enabled`, troca para `cliente_escolhe` → 409
- [ ] `account.updated` com `charges_enabled=false` → barbearia em `so_local`
**Tests**: integration
**Gate**: full

#### T41: Schema de pagamentos 🔵
**What**: Tabelas `pagamento` e `webhook_evento` (PK = event.id) + RLS.
**Where**: `packages/db/src/schema/pagamentos.ts`
**Depends on**: T31
**Requirement**: PAG-06, PAG-09
**Done when**:
- [ ] Insert duplicado de evento → ignorado
**Tests**: integration
**Gate**: full

#### T42: Checkout do agendamento 🔵
**What**: Ao criar PENDENTE_PAGAMENTO, cria Checkout com preço congelado; Stripe indisponível → EXPIRADO + 503.
**Where**: `apps/api/src/pagamentos/checkout.service.ts`
**Depends on**: T39, T41, T33
**Requirement**: PAG-03
**Done when**:
- [ ] Falha do stripe-mock → agendamento EXPIRADO e slot liberado
**Tests**: integration
**Gate**: full

#### T43: Webhook Stripe idempotente 🔵
**What**: Verifica assinatura; `checkout.session.completed` confirma ou reembolsa se EXPIRADO; `charge.refunded`; `charge.dispute.created` alerta.
**Where**: `apps/api/src/pagamentos/stripe-webhook.controller.ts`
**Depends on**: T42
**Requirement**: PAG-04, PAG-05, PAG-06, PAG-07, PAG-09, PAG-10
**Done when**:
- [ ] Mesmo evento 2× → 1 efeito; assinatura inválida → 400; pago após expirar → reembolso
**Tests**: integration
**Gate**: full

#### T44: Expiração de hold 🔵
**What**: Job atrasado de 10 min + sweeper a cada 60 s: EXPIRADO + `expirarCheckout` + outbox.
**Where**: `apps/api/src/pagamentos/hold-expirar.processor.ts`
**Depends on**: T42
**Requirement**: AGD-10
**Done when**:
- [ ] Hold vencido → EXPIRADO, Checkout expirado, slot volta a aparecer
**Tests**: integration
**Gate**: full

#### T45: Reembolsos por transição 🔵
**What**: Liga `transicoes.service` (T34) ao gateway real para CAN-01 e CAN-04.
**Where**: `apps/api/src/pagamentos/reembolso.service.ts`
**Depends on**: T39, T34
**Requirement**: CAN-01, CAN-04, PAG-09
**Done when**:
- [ ] Cancelamento ≥ janela e cancelamento pela barbearia emitem refund integral; < janela não emite
**Tests**: integration
**Gate**: full

#### T46: Meios de pagamento e flag Pix 🔵
**What**: Lista de meios = cartão + Pix só se flag ligada e capability ativa.
**Where**: `apps/api/src/pagamentos/metodos-pagamento.ts`
**Depends on**: T39
**Requirement**: PAG-11
**Done when**:
- [ ] 4 combinações flag × capability testadas
**Tests**: unit
**Gate**: quick

### Phase 7: Planos

```
T47 -> T48 -> T49 -> T51
T47 -> T50
```

#### T47: Schema de assinatura 🔵
**What**: Tabela `assinatura` + campos `plano`, `carencia_ate` em barbearia.
**Where**: `packages/db/src/schema/planos.ts`
**Depends on**: T31
**Requirement**: PLN-01
**Done when**:
- [ ] Migration aplica
**Tests**: integration
**Gate**: full

#### T48: Assinatura via Stripe Billing 🔵
**What**: Checkout em modo subscription (Pro/Plus) e Customer Portal; plano aplicado só pelo webhook.
**Where**: `apps/api/src/planos/billing.service.ts`
**Depends on**: T47, T39
**Requirement**: PLN-01
**Done when**:
- [ ] Plano não muda sem webhook `customer.subscription.updated` ativo
**Tests**: integration
**Gate**: full

#### T49: Webhooks de assinatura e carência 🔵
**What**: `invoice.payment_failed` → carência 7 dias + banner; job de fim da carência → Free.
**Where**: `apps/api/src/planos/billing-webhook.handler.ts`
**Depends on**: T48
**Requirement**: PLN-05, PLN-06
**Done when**:
- [ ] Relógio avançado 7 dias sem pagamento → Free
**Tests**: integration
**Gate**: full

#### T50: Avisos de uso mensal 🔵
**What**: Ao incrementar `uso_mensal`, grava outbox de aviso em 16 e 20 (uma vez por mês).
**Where**: `apps/api/src/planos/uso-mensal.service.ts`
**Depends on**: T47, T33
**Requirement**: PLN-02, PLN-03
**Done when**:
- [ ] 16º e 20º agendamentos geram exatamente 1 aviso cada
**Tests**: integration
**Gate**: full

#### T51: Downgrade e reativação 🔵
**What**: Pendência de escolha de barbeiros; job de 48 h escolhe o mais ocupado em 30 dias; inativação preserva futuros; reativação em lote.
**Where**: `apps/api/src/planos/downgrade.service.ts`
**Depends on**: T49
**Requirement**: PLN-07, PLN-08, PLN-09, PLN-10
**Done when**:
- [ ] Pro com 3 → Free; 48 h sem escolha → só o mais ocupado ativo; futuros intactos
**Tests**: integration
**Gate**: full

### Phase 8: Notificações

```
T52 -> T55 -> T56
T53 -> T55 -> T58
T54 -> T55
T52 -> T57
```

#### T52: Outbox relay 🟣
**What**: Worker a cada 1 s lê outbox com `FOR UPDATE SKIP LOCKED` e publica em BullMQ.
**Where**: `apps/api/src/notificacoes/outbox.relay.ts`
**Depends on**: T31
**Requirement**: NOT-05
**Done when**:
- [ ] 2 relays concorrentes não publicam o mesmo evento 2×
**Tests**: integration
**Gate**: full

#### T53: Adapter WhatsApp 🟣
**What**: Envio de templates via Cloud API e verificação `X-Hub-Signature-256` do webhook inbound.
**Where**: `apps/api/src/notificacoes/whatsapp.adapter.ts`
**Depends on**: T3
**Requirement**: AUTH-01, NOT-02, NOT-04
**Done when**:
- [ ] Payload de template correto; assinatura inválida rejeitada; erro 4xx/5xx mapeado
**Tests**: unit
**Gate**: quick

#### T54: Adapter de e-mail 🟣
**What**: Resend + templates (OTP, confirmação, cancelamento, reembolso, hold expirado, avisos de plano, exportação).
**Where**: `apps/api/src/notificacoes/email.adapter.ts`
**Depends on**: T1
**Requirement**: NOT-01, NOT-03
**Done when**:
- [ ] Cada template renderiza com dados obrigatórios
**Tests**: unit
**Gate**: quick

#### T55: Roteador de notificações 🟣
**What**: Processor que escolhe canal (preferido + e-mail), aplica opt-out, retries 3× com backoff, dedup por jobId, fallback de OTP.
**Where**: `apps/api/src/notificacoes/notificacoes.processor.ts`
**Depends on**: T52, T53, T54
**Requirement**: NOT-01, NOT-03, NOT-04, NOT-05, NOT-08, AUTH-02
**Done when**:
- [ ] Falha 3× → dead-letter sem alterar agendamento; "SAIR" bloqueia lembrete e não confirmação
**Tests**: integration
**Gate**: full

#### T56: Lembretes 24 h e 2 h 🟣
**What**: Agenda jobs atrasados ao confirmar; remove ao cancelar/remarcar; revalida status no disparo.
**Where**: `apps/api/src/notificacoes/lembretes.scheduler.ts`
**Depends on**: T55
**Requirement**: NOT-02
**Done when**:
- [ ] Remarcado → lembretes antigos removidos e novos criados; cancelado → nenhum dispara
**Tests**: integration
**Gate**: full

#### T57: Eventos em tempo real (SSE) 🟣
**What**: `GET /barbearias/:id/eventos` via Redis pub/sub, com guard de tenant e papel.
**Where**: `apps/api/src/notificacoes/eventos.controller.ts`
**Depends on**: T52
**Requirement**: NOT-06
**Done when**:
- [ ] Evento chega ao assinante da barbearia em < 5 s; assinante de outra barbearia não recebe
**Tests**: integration
**Gate**: full

#### T58: Push Expo 🟣
**What**: Envio via Expo Push para barbeiro em novo agendamento/cancelamento; remove token inválido.
**Where**: `apps/api/src/notificacoes/push.adapter.ts`
**Depends on**: T55
**Requirement**: NOT-07
**Done when**:
- [ ] `DeviceNotRegistered` remove o token
**Tests**: unit
**Gate**: quick

### Phase 9: Relatórios & LGPD

```
T59 -> T60
T61 -> T62 -> T63
T62 -> T64
```

#### T59: Relatórios 🔵
**What**: Agregações SQL por período/barbeiro/serviço usando `domain/metricas`; Free limitado ao mês corrente; barbeiro só vê o próprio.
**Where**: `apps/api/src/relatorios/relatorios.service.ts`
**Depends on**: T16, T34
**Requirement**: REL-01, REL-02, REL-03, REL-04, REL-06, REL-08
**Done when**:
- [ ] Fixture da spec bate; barbeiro pedindo outro → 404
**Tests**: integration
**Gate**: full

#### T60: Exportação CSV (Plus) 🔵
**What**: `GET /barbearias/:id/relatorios.csv` só no Plus.
**Where**: `apps/api/src/relatorios/relatorios-csv.controller.ts`
**Depends on**: T59
**Requirement**: REL-07
**Done when**:
- [ ] Pro → 403; Plus → CSV com cabeçalho e linhas esperadas
**Tests**: integration
**Gate**: full

#### T61: Audit log 🟣
**What**: `AuditService.registrar` chamado em preço, plano, papel, reembolso, exportação, exclusão.
**Where**: `apps/api/src/privacidade/audit.service.ts`
**Depends on**: T24
**Requirement**: LGPD-06
**Done when**:
- [ ] Cada ação sensível gera 1 linha com autor e antes/depois
**Tests**: integration
**Gate**: full

#### T62: Exclusão de conta e anonimização 🟣
**What**: Exclusão com OTP; cancela futuros pela política; job anonimiza PII em até 15 dias; passado exibe "Cliente removido".
**Where**: `apps/api/src/privacidade/exclusao.service.ts`
**Depends on**: T61, T34
**Requirement**: LGPD-01, LGPD-02, LGPD-07
**Done when**:
- [ ] Após job: nenhum e-mail/telefone/nome no usuário; agendamento passado preservado
**Tests**: integration
**Gate**: full

#### T63: Exportação de dados 🟣
**What**: Job gera JSON (conta + agendamentos) e envia link por e-mail.
**Where**: `apps/api/src/privacidade/exportacao.processor.ts`
**Depends on**: T62
**Requirement**: LGPD-03
**Done when**:
- [ ] JSON contém só dados do solicitante
**Tests**: integration
**Gate**: full

#### T64: Retenção e encerramento de barbearia 🟣
**What**: Cron de retenção (logs 6 meses, holds 30 dias, OTP) e encerramento de barbearia (página off na hora, anonimização em 90 dias, financeiro 5 anos).
**Where**: `apps/api/src/privacidade/retencao.cron.ts`
**Depends on**: T62
**Requirement**: LGPD-04, LGPD-05
**Done when**:
- [ ] Barbearia encerrada → `/public/:slug` 404 imediato; dia 91 → PII anonimizada, pagamentos intactos
**Tests**: integration
**Gate**: full

### Phase 10: Design system & web pública

```
T65 -> T66 -> T68 -> T69
T67 -> T69
T67 -> T70 -> T71
T66 -> T70
```

#### T65: Tokens do design system 🟢
**What**: Cores (light/dark), tipografia, espaçamento, raios — espelho da página "Design system" do FigCraft; exporta CSS vars e objeto TS.
**Where**: `packages/ui/src/tokens.ts`
**Depends on**: T1
**Requirement**: TEN-10
**Done when**:
- [ ] Teste de contraste ≥ 4.5:1 para pares texto/fundo nos dois temas
**Tests**: unit
**Gate**: quick

#### T66: Componentes primitivos 🟢
**What**: Button, Input, OtpInput, Chip de horário, Card de serviço, Toast — variantes e estados das telas FigCraft.
**Where**: `packages/ui/src/components/primitives.tsx`
**Depends on**: T65
**Requirement**: AGD-06
**Done when**:
- [ ] Estados (default, hover, disabled, loading, erro) testados
**Tests**: unit
**Gate**: quick

#### T67: BFF e sessão web 🟣
**What**: Route handler proxy para a API; cookies httpOnly `SameSite=Lax`; refresh transparente; repassa `traceparent`.
**Where**: `apps/web/app/api/[...path]/route.ts`
**Depends on**: T23
**Requirement**: AUTH-07, NFR-06
**Done when**:
- [ ] Access expirado → refresh automático; cookie nunca acessível via JS
**Tests**: e2e
**Gate**: full

#### T68: Página pública da barbearia 🟢
**What**: `/{slug}` SSR com identidade visual, serviços, selo Free e estado "Agenda indisponível online".
**Where**: `apps/web/app/[slug]/page.tsx`
**Depends on**: T66, T30
**Requirement**: TEN-10, TEN-11, AGD-12
**Done when**:
- [ ] Free mostra selo; Pro não; Free no limite mostra aviso
**Tests**: e2e
**Gate**: full

#### T69: Fluxo de agendamento web 🟢
**What**: Serviço → barbeiro/qualquer → dia/slot → login preservando seleção → escolha "Pagar agora"/"Pagar no local" (quando permitido) → Checkout ou confirmação; tela de retorno com polling.
**Where**: `apps/web/app/[slug]/agendar/page.tsx`
**Depends on**: T68, T67, T36, T43
**Requirement**: AGD-04, AGD-06, AGD-07, AGD-08, AGD-11, AGD-15, AGD-16, CAN-09
**Done when**:
- [ ] E2E "Pagar no local" e "Pagar agora" (cartão de teste) até CONFIRMADO; conflito mostra sugestões
- [ ] Barbearia `so_local` não mostra a escolha; cliente com 2 faltas só vê "Pagar agora"
**Tests**: e2e
**Gate**: full

#### T70: Login do cliente web 🟣
**What**: OTP WhatsApp/e-mail com fallback e Google.
**Where**: `apps/web/app/entrar/page.tsx`
**Depends on**: T67, T66
**Requirement**: AUTH-01, AUTH-02, AUTH-03
**Done when**:
- [ ] Falha WhatsApp exibe "Receber por e-mail"; 429 exibe tempo de espera
**Tests**: e2e
**Gate**: full

#### T71: Minha conta 🟢
**What**: Agendamentos futuros/passados, cancelar, remarcar, canal preferido, exportar e excluir conta.
**Where**: `apps/web/app/conta/page.tsx`
**Depends on**: T70, T62
**Requirement**: CAN-01, CAN-05, LGPD-01, LGPD-03
**Done when**:
- [ ] Cancelar dentro/fora da janela mostra o resultado correto de reembolso
**Tests**: e2e
**Gate**: full

### Phase 11: Web painel

```
T72 -> T73
T72 -> T74
T72 -> T75
T72 -> T76
T72 -> T77
T72 -> T78
T72 -> T79
```

#### T72: Onboarding da barbearia 🟢
**What**: Wizard (2FA → dados/slug → identidade → 1º serviço → jornada) que termina com link público.
**Where**: `apps/web/app/painel/onboarding/page.tsx`
**Depends on**: T30, T21, T66
**Requirement**: TEN-01, AUTH-05
**Done when**:
- [ ] Do cadastro ao link público em uma sessão de e2e
**Tests**: e2e
**Gate**: full

#### T73: Agenda do painel 🟢
**What**: Dia/semana por barbeiro com SSE, concluir (forma de pagamento), no-show, cancelar, bloquear.
**Where**: `apps/web/app/painel/agenda/page.tsx`
**Depends on**: T72, T57
**Requirement**: NOT-06, REL-05, CAN-07
**Done when**:
- [ ] Agendamento criado em outra aba aparece sem reload
**Tests**: e2e
**Gate**: full

#### T74: Serviços no painel 🟢
**What**: CRUD de serviços, overrides por barbeiro, desativar.
**Where**: `apps/web/app/painel/servicos/page.tsx`
**Depends on**: T72
**Requirement**: SRV-01, SRV-02, SRV-03, SRV-06
**Done when**:
- [ ] Override aparece na página pública para aquele barbeiro
**Tests**: e2e
**Gate**: full

#### T75: Equipe e jornadas no painel 🟢
**What**: Convites, ativar/inativar, comissão %, jornadas semanais.
**Where**: `apps/web/app/painel/equipe/page.tsx`
**Depends on**: T72
**Requirement**: TEN-05, TEN-07, TEN-09, REL-04
**Done when**:
- [ ] Free bloqueia 2º barbeiro com CTA de upgrade
**Tests**: e2e
**Gate**: full

#### T76: Configurações e Stripe 🔵
**What**: Modo de pagamento ("Só no local" / "Cliente escolhe"), políticas (antecedência, janela), conectar Stripe (opcional), alertas de contestação.
**Where**: `apps/web/app/painel/configuracoes/page.tsx`
**Depends on**: T72, T40
**Requirement**: PAG-01, PAG-02, PAG-10
**Done when**:
- [ ] "Cliente escolhe" desabilitado até `charges_enabled`, com CTA "Conectar Stripe"
**Tests**: e2e
**Gate**: full

#### T77: Plano e assinatura 🔵
**What**: Comparativo de planos, upgrade, portal, banner de carência, escolha de barbeiros no downgrade.
**Where**: `apps/web/app/painel/plano/page.tsx`
**Depends on**: T72, T51
**Requirement**: PLN-01, PLN-05, PLN-07, PLN-10
**Done when**:
- [ ] Fluxo de downgrade com escolha testado
**Tests**: e2e
**Gate**: full

#### T78: Relatórios no painel 🔵
**What**: Cards de KPI, gráfico de faturamento, ocupação, ranking, comissão; CSV no Plus.
**Where**: `apps/web/app/painel/relatorios/page.tsx`
**Depends on**: T72, T60
**Requirement**: REL-01, REL-06, REL-07, REL-08
**Done when**:
- [ ] Barbeiro logado vê só os próprios números
**Tests**: e2e
**Gate**: full

#### T79: Segurança da conta 🟣
**What**: Setup/reset de TOTP, lista e revogação de sessões.
**Where**: `apps/web/app/painel/seguranca/page.tsx`
**Depends on**: T72
**Requirement**: AUTH-06, AUTH-09
**Done when**:
- [ ] Revogar sessão desloga o outro navegador
**Tests**: e2e
**Gate**: full

### Phase 12: App Expo

```
T80 -> T81 -> T82
T80 -> T83 -> T84
T80 -> T85
```

#### T80: App base e login 🟢
**What**: expo-router com grupos `(cliente)`/`(barbeiro)`, secure-store, OTP, Google e Sign in with Apple.
**Where**: `apps/mobile/app/(auth)/entrar.tsx`
**Depends on**: T23, T65
**Requirement**: AUTH-01, AUTH-02, AUTH-13
**Done when**:
- [ ] iOS com Google exibe Apple; fallback de e-mail funciona
**Tests**: unit
**Gate**: quick

#### T81: Agendar no app 🟢
**What**: Fluxo de agendamento com escolha "Pagar agora"/"Pagar no local" (via `formasPermitidas`) + Checkout via `expo-web-browser` + deep link de retorno com polling.
**Where**: `apps/mobile/app/(cliente)/agendar.tsx`
**Depends on**: T80
**Requirement**: AGD-04, AGD-06, AGD-07, AGD-08, AGD-15
**Done when**:
- [ ] Retorno do Checkout leva à tela de confirmação com status do servidor
**Tests**: unit
**Gate**: quick

#### T82: Meus agendamentos no app 🟢
**What**: Lista, cancelar, remarcar, excluir conta dentro do app.
**Where**: `apps/mobile/app/(cliente)/agendamentos.tsx`
**Depends on**: T81
**Requirement**: CAN-01, CAN-05, LGPD-08
**Done when**:
- [ ] Exclusão de conta acessível sem sair do app
**Tests**: unit
**Gate**: quick

#### T83: Agenda do barbeiro no app 🟢
**What**: Agenda do dia/semana, concluir com forma de pagamento, no-show, bloquear horário, comissão estimada.
**Where**: `apps/mobile/app/(barbeiro)/agenda.tsx`
**Depends on**: T80
**Requirement**: REL-05, REL-08, CAN-07
**Done when**:
- [ ] No-show indisponível antes do início
**Tests**: unit
**Gate**: quick

#### T84: Registro de push 🟢
**What**: Permissão, token Expo enviado ao membro, abertura da notificação navega ao agendamento.
**Where**: `apps/mobile/src/push.ts`
**Depends on**: T83
**Requirement**: NOT-07
**Done when**:
- [ ] Permissão negada não quebra o fluxo
**Tests**: unit
**Gate**: quick

#### T85: App sem compra de plano 🟢
**What**: Teste que garante que nenhuma rota do app expõe compra/troca de plano.
**Where**: `apps/mobile/src/__tests__/sem-planos.test.ts`
**Depends on**: T80
**Requirement**: PLN-11
**Done when**:
- [ ] Varredura das rotas não encontra telas de plano
**Tests**: unit
**Gate**: quick

### Phase 13: Release

```
T87 -> T86
T87 -> T88
```

#### T86: Testes de carga k6 🟣
**What**: `browse.js` (2.000 VUs), `booking.js` (50 req/s), `contention.js` (20 no mesmo slot) contra compose dimensionado como a VM.
**Where**: `load/contention.js`
**Depends on**: T87
**Requirement**: NFR-01, NFR-02, NFR-03
**Done when**:
- [ ] Thresholds p95 e 0 duplicados passam
**Tests**: load (k6)
**Gate**: full

#### T87: Deploy da API 🟣
**What**: Build arm64 → GHCR → SSH `docker compose pull && up -d` com migrations antes; API ×2 réplicas atrás do Caddy.
**Where**: `.github/workflows/deploy-api.yml`
**Depends on**: T2, T8
**Requirement**: NFR-05
**Done when**:
- [ ] Deploy sem downtime perceptível (health check entre réplicas)
**Tests**: none
**Gate**: build

#### T88: Backup e restauração testada 🟣
**What**: Script que restaura o último backup do Neon em branch e roda smoke test.
**Where**: `infra/scripts/restore-check.sh`
**Depends on**: T87
**Requirement**: NFR-07
**Done when**:
- [ ] Restore + smoke verde no CI de release
**Tests**: integration
**Gate**: full

#### T89: Build e submissão nas lojas 🟢
**What**: `eas.json` com perfis dev/preview/production; metadados e privacidade nas lojas.
**Where**: `apps/mobile/eas.json`
**Depends on**: T80
**Requirement**: AUTH-13, LGPD-08
**Done when**:
- [ ] Build de preview instalável em iOS e Android
**Tests**: none
**Gate**: build

---

## Task Granularity Check

Toda task tem um único arquivo em `Where` (ou um doc/config). Tasks de serviço agrupam regras de um mesmo módulo coeso (⚠️ OK). Nenhuma task cobre múltiplos componentes em arquivos diferentes.

## Diagram-Definition Cross-Check

Verificado por `validate_tasks.py` (paridade intra-fase; dependências cross-fase apontam só para fases anteriores).

## Test Co-location Validation

| Task | Layer | Matrix | Task | Status |
| --- | --- | --- | --- | --- |
| T1–T4, T8, T38, T87, T89 | config/infra/docs | none | none | ✅ |
| T5, T6, T18, T25, T31, T41, T47 | DB schema/RLS | integration | integration | ✅ |
| T7, T9, T19–T24, T26–T30, T32–T34, T36, T37, T39, T40, T42–T45, T48–T52, T55–T57, T59–T64, T88 | API service/controller | integration | integration | ✅ |
| T35, T53, T54, T58 | API adapter puro | unit | unit | ✅ |
| T46 | API util puro | unit | unit | ✅ |
| T10–T17 | domain/contracts | unit | unit | ✅ |
| T65, T66 | UI kit | unit | unit | ✅ |
| T67–T79 | web | e2e | e2e | ✅ |
| T80–T85 | mobile | unit | unit | ✅ |
| T86 | load | load (k6) | load (k6) | ✅ |
