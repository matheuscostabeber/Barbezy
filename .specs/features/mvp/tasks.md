# MVP Tasks

## Execution Protocol (MANDATORY -- do not skip)

Implement these tasks with the `tlc-spec-driven` skill: **activate it by name and follow its Execute flow and Critical Rules.** Do not search for skill files by filesystem path. The skill is the source of truth for the full flow (per-task cycle, sub-agent delegation, the walk, check_proofs, the Verifier and the screens sub-agent).

**If the skill cannot be activated, STOP and tell the user - do not proceed without it.**

---

**Design**: `.specs/features/mvp/design.md`
**Status**: Draft
**Approved**: pendente
**Build**: `make build; make typecheck; make lint`
**Run unit**: `cd backend && uv run pytest tests/unit -v`
**Run int**: `cd backend && uv run pytest tests/integration -v`

**Forma de trabalho (AD-012):** cada task diz **quem escreve** e **o que você aprende**.
- **Você (guiado):** Claude explica o conceito e o plano, você escreve o código, Claude revisa e explica as correções.
- **Claude (explicando):** Claude escreve o código e, ao fim da task, explica as partes importantes e responde suas dúvidas.

Depois que você dominar um tipo de task (por exemplo, a terceira rota CRUD), ela pode passar a "Claude (explicando)" se você pedir. A mudança fica registrada em `## Handoff`.

**Branching:** `feat/<fase>-<slug>` a partir de `main`; uma task = um commit Conventional Commits em inglês; `.specs/` é versionado e atualizado no mesmo commit da task.

**Tools:** MCP `figcraft` para consultar telas nas tasks de UI (Phases 8–9). Skill `tlc-spec-driven` em todas; `/security-review` na T65.

---

## Test Coverage Matrix

> Repositório vazio: nenhum guideline encontrado; strong defaults aplicados. Ferramentas da stack decidida em 2026-11-26 (D15). Confirmar na aprovação das tasks.

| Code Layer | Required Test Type | Coverage Expectation | Location Pattern | Run Command |
| --- | --- | --- | --- | --- |
| Aprendizado (`learning/`) | unit | Cada conceito do exercício tem um teste | `learning/**/test_*.py` | `cd learning && uv run pytest -v` |
| Domínio (`backend/app/domain`) | unit | Todos os ramos; 1:1 com ACs; todos os edge cases listados | `backend/tests/unit/domain/test_*.py` | `make test-unit` |
| Schemas Pydantic | unit | Casos válidos + cada regra de rejeição | `backend/tests/unit/schemas/test_*.py` | `make test-unit` |
| Adapter puro (e-mail, templates) | unit | Formatação, erros do provedor | `backend/tests/unit/test_*.py` | `make test-unit` |
| Migrations / RLS / constraints | integration | Migration aplica; constraints e policies se comportam como a spec | `backend/tests/integration/db/test_*.py` | `make test-int` |
| Rotas e serviços da API | integration | Toda rota: feliz + edge + erro + isolamento de tenant (404); Postgres e Redis reais via testcontainers | `backend/tests/integration/**/test_*.py` | `make test-int` |
| Worker / jobs | integration | Cada job: envio, dedup, retry, revalidação | `backend/tests/integration/notificacoes/test_*.py` | `make test-int` |
| Componentes UI | unit | Estados e variantes; contraste dos tokens ≥ 4.5:1 | `frontend/src/**/*.test.tsx` | `make test-unit` |
| Telas / fluxos web | e2e | Fluxos da spec: feliz + erros visíveis | `frontend/e2e/*.spec.ts` | `make test-e2e` |
| Backup | integration | Dump chega ao storage (MinIO no teste) | `backend/tests/integration/infra/test_backup.py` | `make test-int` |
| Restauração | integration | Restaura em container limpo + smoke | `infra/scripts/restore-check.sh` | `bash infra/scripts/restore-check.sh` |
| Carga | load (k6) | Cenário de disputa com threshold | `load/contention.js` | `k6 run load/contention.js` |
| Config / infra / docs / cliente gerado | none | build gate only | - | build gate only |

## Gate Check Commands

| Gate Level | When to Use | Command |
| --- | --- | --- |
| Quick | Tasks com apenas unit | `make lint typecheck test-unit` |
| Full | Tasks com integration/e2e | `make lint typecheck test-unit test-int test-e2e` |
| Build | Fim de fase, config/infra | `make lint typecheck build test-unit test-int` |

---

## Execution Plan

Fases em ordem; dentro de uma fase, na ordem dos diagramas. Dependências entre fases apontam só para fases anteriores.

| Phase | Tema | Tasks |
| --- | --- | --- |
| 0 | Aprendizado | T1–T2 |
| 1 | Fundação | T3–T11 |
| 2 | Regras puras | T12–T17 |
| 3 | Login | T18–T24 |
| 4 | Catálogo | T25–T30 |
| 5 | Agenda | T31–T36 |
| 6 | Notificações | T37–T41 |
| 7 | LGPD e resumo | T42–T46 |
| 8 | Site público | T47–T54 |
| 9 | Painel | T55–T61 |
| 10 | Lançamento | T62–T65 |

---

## Task Breakdown

### Phase 0: Aprendizado

```
T1 -> T2
```

#### T1: Mini-projeto FastAPI + Pydantic
**What**: API de exemplo com `GET /ping`, `POST /eco` (model Pydantic com validação) e um erro 404 personalizado, testada com `TestClient`.
**Where**: `learning/hello_fastapi/main.py`
**Depends on**: None
**Requirement**: AD-012
**Quem escreve**: Você (guiado)
**Você aprende**: o que é uma rota, como o FastAPI lê o corpo da requisição com Pydantic, códigos HTTP (200, 404, 422), `/docs` automático, `uv` e `pytest`.
**Done when**:
- [ ] 4 testes passam: ping, eco válido, eco inválido (422), 404 personalizado
**Tests**: unit
**Gate**: quick

#### T2: Mini-projeto SQLAlchemy assíncrono
**What**: Model `Tarefa`, sessão assíncrona, criar/listar via rota FastAPI com injeção de dependência da sessão, contra SQLite em memória.
**Where**: `learning/hello_db/main.py`
**Depends on**: T1
**Requirement**: AD-012
**Quem escreve**: Você (guiado)
**Você aprende**: model e tabela, sessão e transação, `async`/`await`, `Depends`, por que `expire_on_commit=False` e `selectinload` evitam o erro `MissingGreenlet`.
**Done when**:
- [ ] 3 testes passam: criar, listar, relação carregada sem `MissingGreenlet`
**Tests**: unit
**Gate**: quick

### Phase 1: Fundação

```
T3 -> T4 -> T6 -> T7 -> T8
T3 -> T5 -> T6
T3 -> T9 -> T10 -> T11
T5 -> T10
```

#### T3: Esqueleto do repositório + Makefile
**What**: `backend/` (projeto uv com FastAPI, ruff, mypy, pytest) e `frontend/` (Vite + React + TS, pnpm), com alvos `lint typecheck build test-unit test-int test-e2e gen-api dev` no Makefile.
**Where**: `Makefile`
**Depends on**: None
**Requirement**: AD-006
**Quem escreve**: Claude (explicando)
**Você aprende**: como um repo com duas linguagens se organiza, o que cada alvo do Makefile faz, `pyproject.toml` vs `package.json`.
**Done when**:
- [ ] `make lint typecheck build` passa no repo vazio
**Tests**: none
**Gate**: build

#### T4: Ambiente local com Docker Compose
**What**: Postgres 16 (com `btree_gist`), Redis 7 (AOF) e Mailpit.
**Where**: `infra/docker-compose.yml`
**Depends on**: T3
**Requirement**: AD-005
**Quem escreve**: Claude (explicando)
**Você aprende**: o que é um container, volume e porta; por que rodar o banco em Docker.
**Done when**:
- [ ] `docker compose up` sobe os 3 serviços saudáveis
**Tests**: none
**Gate**: build

#### T5: App FastAPI base
**What**: Configuração por variáveis de ambiente (pydantic-settings), `GET /api/health`, handler de erro 500 genérico e Sentry.
**Where**: `backend/app/main.py`
**Depends on**: T3
**Requirement**: NFR-05
**Quem escreve**: Você (guiado)
**Você aprende**: estrutura de um app FastAPI real, variáveis de ambiente, exception handlers, middleware.
**Done when**:
- [ ] Health responde 200; exceção não tratada responde 500 sem stack trace e chama o Sentry (mock)
**Tests**: integration
**Gate**: full

#### T6: Banco, Alembic e schema base
**What**: Engine e sessão assíncronas, Alembic configurado, migration com `usuario`, `barbearia`, `membro` e os roles `app_owner`, `app_user`, `app_admin`.
**Where**: `backend/alembic/versions/0001_base.py`
**Depends on**: T4, T5
**Requirement**: AUTH-08
**Quem escreve**: Você (guiado) nos models; Claude (explicando) nos roles
**Você aprende**: migrations (por que versionar o banco), `alembic upgrade/downgrade`, roles do Postgres.
**Done when**:
- [ ] Migration aplica e reverte em banco limpo (testcontainers)
**Tests**: integration
**Gate**: full

#### T7: Policies RLS + teste-guardião
**What**: RLS com `FORCE ROW LEVEL SECURITY` em `membro`; teste que falha se qualquer tabela com coluna `barbearia_id` não tiver RLS e policy.
**Where**: `backend/alembic/versions/0002_rls.py`
**Depends on**: T6
**Requirement**: AUTH-08
**Quem escreve**: Claude (explicando)
**Você aprende**: Row Level Security, por que é a segunda barreira contra vazamento entre barbearias.
**Done when**:
- [ ] Com `app.barbearia_id = A`, `SELECT` em `membro` como `app_user` não retorna linhas de B
- [ ] Teste-guardião falha ao criar tabela de teste sem policy
**Tests**: integration
**Gate**: full

#### T8: Sessão por tenant e vínculo do usuário
**What**: Dependências `tenant_session` (`set_config('app.barbearia_id', id, true)`) e `membro_atual` (404 sem vínculo).
**Where**: `backend/app/core/tenancy.py`
**Depends on**: T7
**Requirement**: AUTH-08, AUTH-09
**Quem escreve**: Você (guiado)
**Você aprende**: dependências encadeadas do FastAPI, transação por requisição, por que 404 e não 403.
**Done when**:
- [ ] Usuário de A pedindo recurso de B → 404
**Tests**: integration
**Gate**: full

#### T9: CI
**What**: GitHub Actions rodando o gate Build em PR e na `main`, mais `make gen-api` com falha se o cliente gerado divergir.
**Where**: `.github/workflows/ci.yml`
**Depends on**: T3
**Requirement**: AD-006
**Quem escreve**: Claude (explicando)
**Você aprende**: o que é CI, como ler um job vermelho.
**Done when**:
- [ ] PR de teste roda verde
**Tests**: none
**Gate**: build

#### T10: VPS, domínio e deploy do esqueleto
**What**: VPS Hostinger (Brasil) com Docker; domínio no Cloudflare DNS; SPF/DKIM/DMARC do Resend; Caddy com TLS servindo o site vazio e `/api/health`.
**Where**: `infra/docker-compose.prod.yml`
**Depends on**: T5, T9
**Requirement**: AD-005
**Quem escreve**: Você executa os passos de conta (compra da VPS e do domínio, DNS); Claude (explicando) escreve compose e Caddyfile
**Você aprende**: SSH, DNS (registros A, TXT), HTTPS automático, como o Caddy roteia.
**Done when**:
- [ ] `curl https://<dominio>/api/health` → 200 com TLS válido
- [ ] Recursos reais da VPS (vCPU/RAM/disco) anotados no STATE.md
**Tests**: none
**Gate**: build

#### T11: Deploy automático
**What**: Ao juntar na `main`: build das imagens → GHCR → SSH na VPS → `alembic upgrade head` → `docker compose up -d`.
**Where**: `.github/workflows/deploy.yml`
**Depends on**: T10
**Requirement**: AD-005
**Quem escreve**: Claude (explicando)
**Você aprende**: imagens Docker, registry, segredos no GitHub, como voltar para a versão anterior.
**Done when**:
- [ ] Merge na `main` publica na VPS sem passo manual
**Tests**: none
**Gate**: build

### Phase 2: Regras puras

```
T12 -> T15
T13 -> T14
T16
T17
```

#### T12: Máquina de estados do agendamento
**What**: `transicionar(status, evento)` com tabela fechada; lança `TransicaoInvalida`.
**Where**: `backend/app/domain/estado.py`
**Depends on**: T3
**Requirement**: AGD-09
**Quem escreve**: Você (guiado)
**Você aprende**: enums, exceções próprias, testes parametrizados do pytest.
**Done when**:
- [ ] Cada transição permitida testada; todas as outras combinações lançam
**Tests**: unit
**Gate**: quick

#### T13: Cálculo de slots
**What**: Função pura de slots (faixas, bloqueios, ocupados, duração, fuso, antecedências, grade de 15 min).
**Where**: `backend/app/domain/slots.py`
**Depends on**: T3
**Requirement**: AGD-01, AGD-02
**Quem escreve**: Você (guiado)
**Você aprende**: `datetime` com fuso (`zoneinfo`), intervalos, por que guardar em UTC.
**Done when**:
- [ ] Casos: intervalo de almoço, bloqueio parcial, serviço maior que o restante da jornada, troca de dia no fuso, antecedência mín/máx
**Tests**: unit
**Gate**: quick

#### T14: Atribuição "qualquer um", overrides e faixa de preço
**What**: Escolhe o barbeiro elegível com menos ocupantes no dia (desempate por id); aplica override de preço/duração; calcula a faixa mín–máx.
**Where**: `backend/app/domain/atribuicao.py`
**Depends on**: T13
**Requirement**: AGD-03, AGD-04, SRV-02
**Quem escreve**: Você (guiado)
**Você aprende**: funções puras, `dataclass`, ordenação com chave composta.
**Done when**:
- [ ] Empate resolvido por id; faixa considera só elegíveis e overrides
**Tests**: unit
**Gate**: quick

#### T15: Política de cancelamento, remarcação e no-show
**What**: Decide se pode cancelar, remarcar ou marcar no-show a partir de agora, início e janela; calcula `cancelavelAte` e `remarcavelAte`.
**Where**: `backend/app/domain/politica.py`
**Depends on**: T12
**Requirement**: CAN-04, CAN-05, CAN-06, CAN-07
**Quem escreve**: Você (guiado)
**Você aprende**: testar fronteiras (exatamente no limite, 1 min antes, 1 min depois).
**Done when**:
- [ ] Fronteiras: janela −1 min / exata / +1 min; no-show antes do início, no início e em 24 h + 1 min
**Tests**: unit
**Gate**: quick

#### T16: Métricas do resumo mensal
**What**: Faturamento (soma de `CONCLUIDO`) e taxa de no-show.
**Where**: `backend/app/domain/metricas.py`
**Depends on**: T3
**Requirement**: REL-02, REL-03
**Quem escreve**: Você (guiado)
**Você aprende**: dinheiro em centavos (nunca `float`), divisão por zero.
**Done when**:
- [ ] Fixture da spec (10 × R$ 50 + 2 no-shows) → R$ 500,00 e 16,7%; mês sem atendimentos → taxa 0
**Tests**: unit
**Gate**: quick

#### T17: Schemas de catálogo
**What**: Schemas Pydantic de barbearia (slug, cor, fuso), serviço (duração múltipla de 15 entre 15–240, preço ≥ 0, categoria) e jornada.
**Where**: `backend/app/modules/catalogo/schemas.py`
**Depends on**: T3
**Requirement**: TEN-02, SRV-01
**Quem escreve**: Você (guiado)
**Você aprende**: validadores do Pydantic (`field_validator`), mensagens de erro, como viram o OpenAPI.
**Done when**:
- [ ] Slug reservado/curto/maiúsculo rejeitado; duração 20 rejeitada; categoria fora da lista rejeitada
**Tests**: unit
**Gate**: quick

### Phase 3: Login

```
T18 -> T20
T19 -> T20
T18 -> T21 -> T22
T21 -> T23
T21 -> T24
```

#### T18: Tabelas de identidade
**What**: Migration de `identidade`, `sessao`, `otp`.
**Where**: `backend/alembic/versions/0003_identidade.py`
**Depends on**: T8
**Requirement**: AUTH-06
**Quem escreve**: Você (guiado)
**Você aprende**: índices únicos, por que guardar hash e não o token.
**Done when**:
- [ ] Migration aplica; `unique(provedor, sub)` e `unique(token_hash)` testados
**Tests**: integration
**Gate**: full

#### T19: Rate limiter
**What**: Janela deslizante no Redis por identificador e por IP; dependência que responde 429 com `Retry-After`.
**Where**: `backend/app/core/rate_limit.py`
**Depends on**: T4
**Requirement**: AUTH-02
**Quem escreve**: Claude (explicando)
**Você aprende**: Redis, sorted sets, ataques de força bruta.
**Done when**:
- [ ] 5 tentativas passam, a 6ª em 15 min responde 429; após a janela volta a aceitar
**Tests**: integration
**Gate**: full

#### T20: OTP por e-mail
**What**: `POST /api/auth/otp/request` e `/verify`: código de 6 dígitos, hash no banco, 10 min, uso único, rate limit.
**Where**: `backend/app/modules/auth/otp.py`
**Depends on**: T18, T19
**Requirement**: AUTH-01, AUTH-03
**Quem escreve**: Você (guiado)
**Você aprende**: geração segura de números (`secrets`), comparação em tempo constante, expiração.
**Done when**:
- [ ] Código expirado, errado e reutilizado recusados; e-mail sai pelo Mailpit
**Tests**: integration
**Gate**: full

#### T21: Sessões
**What**: Criar sessão + cookie (`httpOnly`, `Secure`, `SameSite=Lax`, 30 dias), dependência `usuario_atual`, logout, listar e revogar sessões.
**Where**: `backend/app/modules/auth/sessoes.py`
**Depends on**: T18
**Requirement**: AUTH-06, AUTH-07
**Quem escreve**: Você (guiado)
**Você aprende**: cookies e seus atributos de segurança, fixação de sessão, por que httpOnly.
**Done when**:
- [ ] Cookie com os 3 atributos; sessão revogada ou expirada → 401; login gera token novo
**Tests**: integration
**Gate**: full

#### T22: Login com senha + TOTP do dono
**What**: `POST /api/auth/senha/login` (argon2id), setup e verificação de TOTP obrigatórios para dono.
**Where**: `backend/app/modules/auth/senha_totp.py`
**Depends on**: T21
**Requirement**: AUTH-04, AUTH-05, AUTH-13
**Quem escreve**: Você (guiado)
**Você aprende**: hash de senha, como funciona o app autenticador (TOTP), segredo cifrado no banco.
**Done when**:
- [ ] Dono sem TOTP recebe `precisaConfigurarTotp`; dono com TOTP sem código recebe `precisaTotp`; senha errada → 401
**Tests**: integration
**Gate**: full

#### T23: Login com Google
**What**: Fluxo OIDC com Authlib; vincula por e-mail verificado ou cria conta.
**Where**: `backend/app/modules/auth/google.py`
**Depends on**: T21
**Requirement**: AUTH-12
**Quem escreve**: Claude (explicando)
**Você aprende**: OAuth em linguagem simples (redirect, code, token), por que só e-mail verificado.
**Done when**:
- [ ] Com o Google simulado: conta existente vinculada; conta nova criada; e-mail não verificado recusado
**Tests**: integration
**Gate**: full

#### T24: Autorização por papel
**What**: Dependências `exige_dono`, `exige_membro`, `exige_admin`; barbeiro sem acesso a faturamento da casa, dados de outros barbeiros e configurações; admin sem dados pessoais de clientes.
**Where**: `backend/app/core/permissoes.py`
**Depends on**: T21, T8
**Requirement**: AUTH-10, AUTH-11
**Quem escreve**: Você (guiado)
**Você aprende**: autenticação vs autorização, testes de matriz de permissões.
**Done when**:
- [ ] Matriz papel × rota testada
**Tests**: integration
**Gate**: full

### Phase 4: Catálogo

```
T25 -> T26 -> T27
T26 -> T28
T26 -> T29
T26 -> T30
```

#### T25: Tabelas de catálogo
**What**: Migration de `servico`, `servico_membro`, `faixa_jornada`, `bloqueio`, `convite`, `slug_antigo`, todas com RLS.
**Where**: `backend/alembic/versions/0004_catalogo.py`
**Depends on**: T8
**Requirement**: AUTH-08
**Quem escreve**: Claude (explicando)
**Você aprende**: chaves estrangeiras, tipo `tstzrange`.
**Done when**:
- [ ] Migration aplica; teste-guardião de RLS continua verde
**Tests**: integration
**Gate**: full

#### T26: Onboarding e slug
**What**: `POST /api/barbearias` (dono vira membro dono), `PATCH /slug` com regra de 30 dias e registro em `slug_antigo` por 12 meses.
**Where**: `backend/app/modules/catalogo/barbearia_router.py`
**Depends on**: T25, T17, T24
**Requirement**: TEN-01, TEN-02, TEN-03, TEN-04
**Quem escreve**: Você (guiado)
**Você aprende**: o padrão router → service → banco que se repete em todo o projeto.
**Done when**:
- [ ] Slug indisponível → "Endereço indisponível"; 2ª troca em 29 dias recusada
**Tests**: integration
**Gate**: full

#### T27: Equipe e convites
**What**: Convidar por e-mail (link 7 dias), aceitar convite, listar e inativar membros.
**Where**: `backend/app/modules/catalogo/equipe_router.py`
**Depends on**: T26
**Requirement**: TEN-05, TEN-06
**Quem escreve**: Você (guiado)
**Você aprende**: tokens de uso único, expiração.
**Done when**:
- [ ] Convite expirado ou usado → "Convite inválido" sem vínculo; barbeiro inativado mantém agendamentos futuros
**Tests**: integration
**Gate**: full

#### T28: Jornadas e bloqueios
**What**: CRUD de faixas de jornada (rejeita sobreposição) e bloqueios (recusa sobre agendamentos existentes, listando-os).
**Where**: `backend/app/modules/catalogo/jornadas_router.py`
**Depends on**: T26
**Requirement**: TEN-07
**Quem escreve**: Claude (explicando)
**Você aprende**: validação que depende do banco (não cabe no schema).
**Done when**:
- [ ] Faixas sobrepostas rejeitadas; bloqueio sobre agendamento recusado com a lista
**Tests**: integration
**Gate**: full

#### T29: Serviços e overrides
**What**: CRUD de serviços, habilitar barbeiro por serviço, override de preço/duração, desativar; exclusão recusada quando há agendamentos.
**Where**: `backend/app/modules/catalogo/servicos_router.py`
**Depends on**: T26
**Requirement**: SRV-03, SRV-05, SRV-06, SRV-07
**Quem escreve**: Claude (explicando)
**Você aprende**: soft delete (desativar em vez de apagar).
**Done when**:
- [ ] Serviço desativado some de `/api/public/{slug}`; excluir com agendamento → recusa oferecendo desativar
**Tests**: integration
**Gate**: full

#### T30: Página pública e prévia do link
**What**: `GET /api/public/{slug}` e `GET /{path}` devolvendo `index.html` com `og:title`/`og:image`, 301 para slug antigo e 404 "Barbearia não encontrada".
**Where**: `backend/app/modules/public_page/router.py`
**Depends on**: T26
**Requirement**: TEN-09
**Quem escreve**: Você (guiado)
**Você aprende**: como o WhatsApp lê meta tags, HTML a partir do backend, escapar texto contra injeção.
**Done when**:
- [ ] HTML de slug ativo tem OG corretos; nome com `<script>` sai escapado; slug inexistente → 404
**Tests**: integration
**Gate**: full

### Phase 5: Agenda

```
T31 -> T32 -> T33 -> T34 -> T35
T33 -> T36
```

#### T31: Tabela de agendamento + exclusion constraint
**What**: Migration de `agendamento` com `EXCLUDE USING gist` filtrado por status ocupante, `UNIQUE(barbearia_id, idempotency_key)` e RLS.
**Where**: `backend/alembic/versions/0005_agendamento.py`
**Depends on**: T25
**Requirement**: AGD-05
**Quem escreve**: Claude (explicando)
**Você aprende**: exclusion constraint e por que ela resolve concorrência melhor que qualquer `if` no código.
**Done when**:
- [ ] Dois INSERTs sobrepostos para o mesmo barbeiro → 23P01; cancelado não bloqueia
**Tests**: integration
**Gate**: full

#### T32: Endpoint de slots com cache
**What**: `GET /api/public/{slug}/slots` usando `domain/slots.py`, cache Redis versionado e `INCR` da versão em toda mudança de agenda.
**Where**: `backend/app/modules/agenda/slots_router.py`
**Depends on**: T31, T13
**Requirement**: AGD-01, AGD-02, SRV-03, NFR-02
**Quem escreve**: Você (guiado)
**Você aprende**: cache e invalidação, por que versão em vez de apagar chaves.
**Done when**:
- [ ] Barbeiro desabilitado no serviço fora dos slots; novo agendamento invalida o cache do dia
**Tests**: integration
**Gate**: full

#### T33: Criar agendamento
**What**: `POST /api/barbearias/{id}/agendamentos` com `Idempotency-Key`, "qualquer um", preço congelado, 409 com 3 sugestões, outbox.
**Where**: `backend/app/modules/agenda/criar.py`
**Depends on**: T32, T14, T12
**Requirement**: AGD-03, AGD-05, AGD-06, AGD-07, AGD-11, SRV-04
**Quem escreve**: Você (guiado)
**Você aprende**: transações, tratar erro do banco (23P01) e transformar em resposta HTTP.
**Done when**:
- [ ] Cria `CONFIRMADO` "pagar no local"; conflito → 409 + 3 sugestões; mesma chave → mesmo agendamento
**Tests**: integration
**Gate**: full

#### T34: Transições do agendamento
**What**: `POST /api/agendamentos/{id}/{cancelar|remarcar|concluir|no-show}` usando `estado.py` e `politica.py`; concluir exige forma de pagamento; grava outbox.
**Where**: `backend/app/modules/agenda/transicoes.py`
**Depends on**: T33, T15
**Requirement**: CAN-01, CAN-02, CAN-03, AGD-10, REL-04
**Quem escreve**: Você (guiado)
**Você aprende**: ligar regras puras a rotas sem duplicar lógica.
**Done when**:
- [ ] Cancelar depois do início → 409; transição inválida → 409 sem alterar; concluir sem forma → 422
**Tests**: integration
**Gate**: full

#### T35: Link assinado de cancelamento
**What**: Gerar token HMAC que expira no início; `GET` mostra o agendamento, `POST` cancela sem login.
**Where**: `backend/app/modules/agenda/link_cancelamento.py`
**Depends on**: T34
**Requirement**: CAN-08, CAN-09
**Quem escreve**: Claude (explicando)
**Você aprende**: assinatura HMAC, por que o link não precisa de login e ainda é seguro.
**Done when**:
- [ ] Link válido cancela; adulterado ou após o início → "Link expirado"
**Tests**: integration
**Gate**: full

#### T36: Listagens de agenda
**What**: Agenda do painel por período/membro (barbeiro vê só a própria) e "meus agendamentos" do cliente, com `cancelavelAte` e `remarcavelAte`.
**Where**: `backend/app/modules/agenda/listagens.py`
**Depends on**: T33
**Requirement**: AUTH-10
**Quem escreve**: Claude (explicando)
**Você aprende**: paginação, filtros por query string.
**Done when**:
- [ ] Barbeiro não vê agendamentos de outro barbeiro; cliente vê só os próprios
**Tests**: integration
**Gate**: full

### Phase 6: Notificações

```
T37 -> T38 -> T40 -> T41
T39 -> T40
```

#### T37: Outbox
**What**: Tabela `outbox` (migration) e helper `publicar_evento(session, tipo, payload)` usado na transação de domínio.
**Where**: `backend/app/modules/notificacoes/outbox.py`
**Depends on**: T33
**Requirement**: NOT-05
**Quem escreve**: Claude (explicando)
**Você aprende**: o padrão outbox (por que não mandar e-mail direto da rota).
**Done when**:
- [ ] Rollback da transação não deixa evento no outbox
**Tests**: integration
**Gate**: full

#### T38: Worker arq e relay do outbox
**What**: `WorkerSettings` com cron que lê o outbox (`FOR UPDATE SKIP LOCKED`) e enfileira jobs com `_job_id` de dedup.
**Where**: `backend/app/worker.py`
**Depends on**: T37
**Requirement**: NOT-05
**Quem escreve**: Claude (explicando)
**Você aprende**: o que é um worker, filas, `SKIP LOCKED`.
**Done when**:
- [ ] Evento publicado uma única vez mesmo com dois relays rodando
**Tests**: integration
**Gate**: full

#### T39: Adapter de e-mail e templates
**What**: Envio via SDK Resend (Mailpit local) e templates de OTP, convite, confirmação, lembrete, cancelamento/remarcação e exportação.
**Where**: `backend/app/modules/notificacoes/email.py`
**Depends on**: T3
**Requirement**: NOT-07
**Quem escreve**: Você (guiado)
**Você aprende**: templates, integração com API externa, tratar erro do provedor.
**Done when**:
- [ ] Cada template renderiza com dados de exemplo; só existem templates transacionais
**Tests**: unit
**Gate**: quick

#### T40: Jobs de confirmação, cancelamento e lembretes
**What**: Jobs que enviam confirmação (com link de cancelamento), cancelamento/remarcação para cliente e barbeiro, e lembretes 24 h e 2 h com revalidação de status, horário e opt-out; 3 tentativas com backoff.
**Where**: `backend/app/modules/notificacoes/jobs.py`
**Depends on**: T38, T39
**Requirement**: NOT-01, NOT-02, NOT-03, NOT-05
**Quem escreve**: Você (guiado)
**Você aprende**: jobs agendados para o futuro, retry, idempotência.
**Done when**:
- [ ] Lembretes disparam uma vez cada; agendamento cancelado não recebe lembrete; falha do provedor tenta 3 vezes
**Tests**: integration
**Gate**: full

#### T41: Opt-out de lembretes
**What**: `GET/POST /api/lembretes/optout?token=` (link assinado no rodapé do lembrete) liga `opt_out_lembretes`.
**Where**: `backend/app/modules/notificacoes/optout_router.py`
**Depends on**: T40
**Requirement**: NOT-04
**Quem escreve**: Claude (explicando)
**Você aprende**: base legal de legítimo interesse com opt-out (LGPD).
**Done when**:
- [ ] Após opt-out, lembrete não sai e confirmação continua saindo
**Tests**: integration
**Gate**: full

### Phase 7: LGPD e resumo

```
T42 -> T43
T42 -> T44
T42 -> T45
T46
```

#### T42: Audit log
**What**: Tabela `audit_log` e `registrar_auditoria(...)` chamado em mudança de preço, papel, exportação e exclusão.
**Where**: `backend/app/modules/privacidade/audit.py`
**Depends on**: T8
**Requirement**: LGPD-05
**Quem escreve**: Claude (explicando)
**Você aprende**: trilha de auditoria, quando registrar antes/depois.
**Done when**:
- [ ] As 4 ações geram registro com autor e data
**Tests**: integration
**Gate**: full

#### T43: Exclusão de conta
**What**: `POST /api/me/exclusao` com OTP; cancela futuros; job `lgpd_anonimizar` limpa dados pessoais em até 15 dias; passados exibem "Cliente removido". Inclui teste que varre schemas e models e falha se existir campo de CPF, nascimento ou endereço do cliente.
**Where**: `backend/app/modules/privacidade/exclusao.py`
**Depends on**: T42, T34
**Requirement**: LGPD-01, LGPD-02, LGPD-06
**Quem escreve**: Você (guiado)
**Você aprende**: anonimizar vs apagar, por que o histórico continua.
**Done when**:
- [ ] Futuro cancelado; passado com "Cliente removido"; e-mail sem PII após o job
**Tests**: integration
**Gate**: full

#### T44: Exportação de dados
**What**: `POST /api/me/exportacao` enfileira job que envia JSON com conta e agendamentos por e-mail.
**Where**: `backend/app/modules/privacidade/exportacao.py`
**Depends on**: T42
**Requirement**: LGPD-03
**Quem escreve**: Claude (explicando)
**Você aprende**: tarefas longas fora da requisição.
**Done when**:
- [ ] JSON chega ao Mailpit com conta e agendamentos; nenhum dado de outro cliente
**Tests**: integration
**Gate**: full

#### T45: Retenção
**What**: Cron `lgpd_retencao` apaga logs de acesso > 6 meses, OTPs vencidos e atendimentos > 5 anos.
**Where**: `backend/app/modules/privacidade/retencao.py`
**Depends on**: T42
**Requirement**: LGPD-04
**Quem escreve**: Claude (explicando)
**Você aprende**: crons no arq, apagar em lote com segurança.
**Done when**:
- [ ] Registros fora do prazo apagados; dentro do prazo preservados
**Tests**: integration
**Gate**: full

#### T46: Resumo mensal
**What**: `GET /api/barbearias/{id}/resumo?mes=` com contagem por status no fuso da barbearia, faturamento e taxa de no-show; barbeiro vê só os próprios.
**Where**: `backend/app/modules/relatorios/router.py`
**Depends on**: T16, T36
**Requirement**: REL-01, REL-05
**Quem escreve**: Você (guiado)
**Você aprende**: agregações SQL (`GROUP BY`) com SQLAlchemy, mês no fuso certo.
**Done when**:
- [ ] Agendamento às 23h30 do último dia (fuso SP) conta no mês certo; barbeiro só vê os próprios
**Tests**: integration
**Gate**: full

### Phase 8: Site público

```
T47 -> T49
T48 -> T49
T49 -> T50 -> T51
T49 -> T52 -> T53
T49 -> T54
```

#### T47: Design system
**What**: Tokens do FigCraft (cores light/dark, tipografia, espaçamento, raios) no `@theme` do Tailwind + componentes base do shadcn/ui.
**Where**: `frontend/src/styles/tokens.css`
**Depends on**: T3
**Requirement**: AD-006
**Quem escreve**: Claude (explicando)
**Você aprende**: Tailwind, como o shadcn coloca componentes no seu código, dark mode.
**Done when**:
- [ ] Teste de contraste dos pares de tokens ≥ 4.5:1
**Tests**: unit
**Gate**: quick

#### T48: Cliente da API gerado
**What**: `openapi-ts.config.ts` gerando tipos e hooks TanStack Query a partir de `/api/openapi.json`; `make gen-api`.
**Where**: `frontend/openapi-ts.config.ts`
**Depends on**: T5
**Requirement**: AD-006
**Quem escreve**: Claude (explicando)
**Você aprende**: como o OpenAPI liga Python e TypeScript, o que é TanStack Query.
**Done when**:
- [ ] `make gen-api` gera o cliente e `make typecheck` passa
**Tests**: none
**Gate**: build

#### T49: Shell do app, rotas e PWA
**What**: React Router com as rotas do design, layout, tratamento de 401 (redireciona para `/entrar` preservando a página), manifest e service worker só de arquivos estáticos.
**Where**: `frontend/src/main.tsx`
**Depends on**: T47, T48
**Requirement**: AD-006
**Quem escreve**: Você (guiado)
**Você aprende**: componentes, rotas, estado de carregamento/erro, o que torna um site "instalável".
**Done when**:
- [ ] E2E: rota desconhecida mostra 404; rota protegida sem sessão vai para `/entrar`
**Tests**: e2e
**Gate**: full

#### T50: Página pública da barbearia
**What**: `/{slug}` com logo, capa, cor de destaque, serviços ativos e preços.
**Where**: `frontend/src/features/publico/PaginaBarbearia.tsx`
**Depends on**: T49, T30
**Requirement**: TEN-08, SRV-06
**Quem escreve**: Você (guiado)
**Você aprende**: buscar dados com hooks gerados, CSS variável para a cor da barbearia.
**Done when**:
- [ ] E2E: logo, capa e cor exibidos; serviço desativado não aparece
**Tests**: e2e
**Gate**: full

#### T51: Fluxo de agendamento
**What**: Serviço → barbeiro/"qualquer um" (faixa de preço) → data/horário → login preservando a seleção → confirmação; 409 mostra sugestões.
**Where**: `frontend/src/features/agendar/FluxoAgendamento.tsx`
**Depends on**: T50, T33
**Requirement**: AGD-04, AGD-06, AGD-08
**Quem escreve**: Você (guiado)
**Você aprende**: formulário em etapas, guardar seleção entre páginas, mutações e erros da API.
**Done when**:
- [ ] E2E: fluxo completo; seleção preservada após login; conflito mostra "Esse horário acabou de ser reservado" e chips
**Tests**: e2e
**Gate**: full

#### T52: Telas de login
**What**: `/entrar` com código por e-mail, Google, e-mail + senha, TOTP e configuração de TOTP (QR code).
**Where**: `frontend/src/features/auth/Entrar.tsx`
**Depends on**: T49, T22, T23
**Requirement**: AUTH-01, AUTH-04
**Quem escreve**: Claude (explicando)
**Você aprende**: estados de formulário, mensagens de erro (429 com tempo de espera).
**Done when**:
- [ ] E2E: cliente entra por código; dono sem TOTP é levado à configuração
**Tests**: e2e
**Gate**: full

#### T53: Minha conta
**What**: `/conta` com agendamentos futuros/passados, cancelar, remarcar, sessões, desligar lembretes, exportar e excluir conta.
**Where**: `frontend/src/features/conta/MinhaConta.tsx`
**Depends on**: T52, T34, T41, T43, T44
**Requirement**: CAN-01, CAN-05, LGPD-01, LGPD-03
**Quem escreve**: Claude (explicando)
**Você aprende**: confirmação de ação destrutiva, lista com estados vazio/carregando/erro.
**Done when**:
- [ ] E2E: cancelar com confirmação; remarcar dentro da janela mostra o limite; excluir conta exige código
**Tests**: e2e
**Gate**: full

#### T54: Cancelar pelo link
**What**: `/cancelar?token=` mostra o agendamento e cancela sem login; link inválido mostra "Link expirado".
**Where**: `frontend/src/features/agendar/CancelarPorLink.tsx`
**Depends on**: T49, T35
**Requirement**: CAN-08, CAN-09
**Quem escreve**: Você (guiado)
**Você aprende**: ler parâmetros da URL, página pública sem sessão.
**Done when**:
- [ ] E2E: link válido cancela; link adulterado mostra "Link expirado"
**Tests**: e2e
**Gate**: full

### Phase 9: Painel

```
T55 -> T56
T55 -> T57
T55 -> T58
T55 -> T59
T55 -> T60
T55 -> T61
```

#### T55: Onboarding do dono
**What**: Assistente: conta → 2FA → nome, slug, endereço, fuso, logo → primeiro serviço → jornada.
**Where**: `frontend/src/features/painel/Onboarding.tsx`
**Depends on**: T52, T26
**Requirement**: TEN-01, TEN-02
**Quem escreve**: Você (guiado)
**Você aprende**: fluxos de várias etapas com validação vinda da API.
**Done when**:
- [ ] E2E: dono novo chega à página pública pronta; slug indisponível mostra "Endereço indisponível"
**Tests**: e2e
**Gate**: full

#### T56: Agenda do barbeiro
**What**: Visão dia/semana, concluir (com forma de pagamento), no-show, cancelar; atualização a cada 30 s.
**Where**: `frontend/src/features/painel/AgendaBarbeiro.tsx`
**Depends on**: T55, T36
**Requirement**: NOT-06, REL-04, CAN-06
**Quem escreve**: Você (guiado)
**Você aprende**: `refetchInterval`, calendário, ações que mudam o estado no servidor.
**Done when**:
- [ ] E2E: agendamento criado por outra sessão aparece em até 30 s; concluir pede a forma
**Tests**: e2e
**Gate**: full

#### T57: Serviços
**What**: Listar, criar, editar, desativar serviços; barbeiros habilitados e overrides.
**Where**: `frontend/src/features/painel/Servicos.tsx`
**Depends on**: T55, T29
**Requirement**: SRV-01, SRV-02, SRV-07
**Quem escreve**: Claude (explicando)
**Você aprende**: formulários com validação espelhando a API.
**Done when**:
- [ ] E2E: criar serviço com override; excluir com agendamento oferece desativar
**Tests**: e2e
**Gate**: full

#### T58: Equipe e convites
**What**: Convidar por e-mail, ver convites pendentes, inativar barbeiro (com aviso de agendamentos futuros).
**Where**: `frontend/src/features/painel/Equipe.tsx`
**Depends on**: T55, T27
**Requirement**: TEN-05, TEN-06
**Quem escreve**: Claude (explicando)
**Você aprende**: reaproveitar componentes de lista e formulário.
**Done when**:
- [ ] E2E: convite enviado aparece pendente; aceitar o link adiciona o barbeiro
**Tests**: e2e
**Gate**: full

#### T59: Jornadas e bloqueios
**What**: Editor da jornada semanal por barbeiro e criação de bloqueios (mostra conflitos).
**Where**: `frontend/src/features/painel/Jornadas.tsx`
**Depends on**: T55, T28
**Requirement**: TEN-07
**Quem escreve**: Claude (explicando)
**Você aprende**: entrada de horários, mostrar erro de validação do servidor no campo certo.
**Done when**:
- [ ] E2E: faixa sobreposta mostra erro; bloqueio sobre agendamento lista o conflito
**Tests**: e2e
**Gate**: full

#### T60: Configurações da barbearia
**What**: Identidade visual (logo, capa, cor), slug (com regra de 30 dias), antecedências e janela de remarcação.
**Where**: `frontend/src/features/painel/Configuracoes.tsx`
**Depends on**: T55
**Requirement**: TEN-03, TEN-04, TEN-08
**Quem escreve**: Claude (explicando)
**Você aprende**: upload de imagem, pré-visualização.
**Done when**:
- [ ] E2E: nova cor aparece na página pública; troca de slug antes de 30 dias recusada
**Tests**: e2e
**Gate**: full

#### T61: Resumo do mês
**What**: Cartões com contagem por status, faturamento e taxa de no-show; seletor de mês; barbeiro vê só os próprios números.
**Where**: `frontend/src/features/painel/Resumo.tsx`
**Depends on**: T55, T46
**Requirement**: REL-01, REL-05
**Quem escreve**: Você (guiado)
**Você aprende**: formatar dinheiro e porcentagem em pt-BR.
**Done when**:
- [ ] E2E: seed da spec mostra R$ 500,00 e 16,7%
**Tests**: e2e
**Gate**: full

### Phase 10: Lançamento

```
T62 -> T63
T64
T65
```

#### T62: Backup diário
**What**: Script `pg_dump` comprimido enviado para Backblaze B2 ou Cloudflare R2 (API S3), agendado por cron no compose de produção; teste com MinIO.
**Where**: `infra/scripts/backup.sh`
**Depends on**: T11
**Requirement**: NFR-03
**Quem escreve**: Claude (explicando)
**Você aprende**: backup lógico, storage compatível com S3, retenção de backups.
**Done when**:
- [ ] Teste de integração encontra o dump no bucket MinIO
**Tests**: integration
**Gate**: full

#### T63: Restauração testada
**What**: Script que baixa o último backup, restaura num container Postgres limpo e roda smoke test (contagens e uma consulta de slots).
**Where**: `infra/scripts/restore-check.sh`
**Depends on**: T62
**Requirement**: NFR-04
**Quem escreve**: Claude (explicando)
**Você aprende**: por que backup sem restauração testada não é backup.
**Done when**:
- [ ] Script termina com código 0; procedimento mensal anotado no rollout
**Tests**: integration
**Gate**: full

#### T64: Teste de disputa de horário
**What**: Cenário k6 com 20 requisições simultâneas no mesmo slot contra o compose local.
**Where**: `load/contention.js`
**Depends on**: T33
**Requirement**: NFR-01
**Quem escreve**: Claude (explicando)
**Você aprende**: teste de carga, thresholds do k6.
**Done when**:
- [ ] Threshold: exatamente 1 × 201 e 19 × 409; zero duplicados no banco
**Tests**: load (k6)
**Gate**: full

#### T65: Revisão de segurança
**What**: Rodar `/security-review` no branch completo e registrar achados e correções.
**Where**: `docs/security-review.md`
**Depends on**: T61
**Requirement**: AD-011
**Quem escreve**: Claude (explicando)
**Você aprende**: o que se procura numa revisão de segurança, como priorizar achados.
**Done when**:
- [ ] Nenhum achado de severidade alta em aberto
**Tests**: none
**Gate**: build

---

## Checks

**C1** - Onboarding concluído cria a barbearia com página pública ativa em `/{slug}` (TEN-01, AC 1)
Proof: run int | test_onboarding.py::test_onboarding_cria_barbearia_com_pagina_ativa PASSED

**C2** - Slug existente, reservado ou fora de `^[a-z0-9-]{3,40}$` é rejeitado com "Endereço indisponível" (TEN-02, AC 2)
Proof: run int | test_onboarding.py::test_slug_indisponivel_rejeitado PASSED

**C3** - Slug antigo responde 301 para o novo por 12 meses (TEN-03, AC 3)
Proof: run int | test_slug.py::test_slug_antigo_redireciona_301 PASSED

**C4** - Troca de slug com menos de 30 dias da anterior é rejeitada (TEN-04, AC 4)
Proof: run int | test_slug.py::test_alterar_slug_antes_de_30_dias_rejeitado PASSED

**C5** - Convite envia link válido por 7 dias (TEN-05, AC 5)
Proof: run int | test_convites.py::test_convite_envia_link_valido_por_7_dias PASSED

**C6** - Convite expirado ou usado mostra "Convite inválido" e não cria vínculo (TEN-06, AC 6)
Proof: run int | test_convites.py::test_convite_expirado_ou_usado_invalido PASSED

**C7** - Faixas de jornada sobrepostas são rejeitadas (TEN-07, AC 7)
Proof: run int | test_jornadas.py::test_faixas_sobrepostas_rejeitadas PASSED

**C8** - Logo, capa e cor de destaque aparecem na página pública no próximo carregamento (TEN-08, AC 8)
Proof: `cd frontend && pnpm playwright test e2e/publico.spec.ts -g "exibe logo capa e cor"`

**C9** - `/{slug}` devolve HTML com `og:title` = nome e `og:image` = capa, ou logo sem capa (TEN-09, AC 9)
Proof: run int | test_public_page.py::test_html_tem_og_title_e_og_image PASSED

**C10** - Serviço exige nome, duração múltipla de 15 entre 15 e 240, preço ≥ 0 e categoria válida (SRV-01, AC 1)
Proof: run unit | test_catalogo_schemas.py::test_servico_valida_duracao_preco_categoria PASSED

**C11** - Override de preço e duração do barbeiro é usado nos agendamentos com ele (SRV-02, AC 2)
Proof: run unit | test_atribuicao.py::test_override_de_preco_e_duracao_por_barbeiro PASSED

**C12** - Barbeiro desabilitado para um serviço não aparece nos slots desse serviço (SRV-03, AC 3)
Proof: run int | test_slots_api.py::test_barbeiro_desabilitado_fora_dos_slots PASSED

**C13** - Agendamento grava preço e duração vigentes (SRV-04, AC 4)
Proof: run int | test_agendamento_criar.py::test_preco_e_duracao_congelados PASSED

**C14** - Alterar o preço do serviço mantém o preço dos agendamentos existentes (SRV-05, AC 5)
Proof: run int | test_servicos.py::test_alterar_preco_mantem_agendamentos_existentes PASSED

**C15** - Serviço desativado some da página pública e mantém agendamentos futuros (SRV-06, AC 6)
Proof: run int | test_servicos.py::test_servico_desativado_some_da_pagina_publica PASSED

**C16** - Excluir serviço com agendamentos é recusado com oferta de desativação (SRV-07, AC 7)
Proof: run int | test_servicos.py::test_excluir_servico_com_agendamentos_recusado PASSED

**C17** - Slots = jornada − bloqueios − ocupados, grade de 15 min, no fuso da barbearia (AGD-01, AC 1)
Proof: run unit | test_slots.py::test_slots_jornada_bloqueios_ocupados_grade_15 PASSED

**C18** - Slots respeitam antecedência mínima de 2 h e máxima de 30 dias (AGD-02, AC 2)
Proof: run unit | test_slots.py::test_slots_respeitam_antecedencia_min_e_max PASSED

**C19** - "Qualquer um" atribui o barbeiro com menos ocupantes no dia, desempate por id (AGD-03, AC 3)
Proof: run unit | test_atribuicao.py::test_qualquer_um_menos_ocupado_desempate_por_id PASSED

**C20** - "Qualquer um" mostra faixa mín–máx entre elegíveis (AGD-04, AC 4)
Proof: run unit | test_atribuicao.py::test_faixa_de_preco_entre_elegiveis PASSED

**C21** - O banco impede dois agendamentos ocupantes sobrepostos do mesmo barbeiro (AGD-05, AC 5)
Proof: run int | test_agendamento_schema.py::test_exclusion_impede_sobreposicao PASSED

**C22** - Conflito responde 409 com "Esse horário acabou de ser reservado" e até 3 sugestões (AGD-06, AC 6)
Proof: run int | test_agendamento_criar.py::test_conflito_409_com_3_sugestoes PASSED

**C23** - Confirmação cria o agendamento em `CONFIRMADO` com forma "Pagar no local" (AGD-07, AC 7)
Proof: run int | test_agendamento_criar.py::test_cria_confirmado_pagar_no_local PASSED

**C24** - Cliente sem login é levado ao login e volta com a seleção preservada (AGD-08, AC 8)
Proof: `cd frontend && pnpm playwright test e2e/agendar.spec.ts -g "exige login preservando selecao"`

**C25** - Só as transições de `CONFIRMADO` para os 4 estados finais são permitidas (AGD-09, AC 9)
Proof: run unit | test_estado.py::test_transicoes_permitidas PASSED

**C26** - Transição fora da lista responde 409 e não altera o agendamento (AGD-10, AC 10)
Proof: run int | test_transicoes.py::test_transicao_invalida_409_sem_alterar PASSED

**C27** - Mesma `Idempotency-Key` devolve o mesmo agendamento sem criar outro (AGD-11, AC 11)
Proof: run int | test_agendamento_criar.py::test_idempotency_key_devolve_mesmo_agendamento PASSED

**C28** - Cliente cancela antes do início → `CANCELADO_CLIENTE` (CAN-01, AC 1)
Proof: run int | test_transicoes.py::test_cliente_cancela_antes_do_inicio PASSED

**C29** - Cliente cancela depois do início → 409 sem alterar (CAN-02, AC 2)
Proof: run int | test_transicoes.py::test_cliente_cancela_depois_do_inicio_409 PASSED

**C30** - Barbearia cancela → `CANCELADO_BARBEARIA` (CAN-03, AC 3)
Proof: run int | test_transicoes.py::test_barbearia_cancela PASSED

**C31** - Remarcar com antecedência ≥ janela é permitido (CAN-04, AC 4)
Proof: run unit | test_politica.py::test_remarcar_com_antecedencia_maior_ou_igual_janela PASSED

**C32** - Remarcar com antecedência < janela é recusado (CAN-05, AC 5)
Proof: run unit | test_politica.py::test_remarcar_dentro_da_janela_recusado PASSED

**C33** - No-show até 24 h após o início é permitido (CAN-06, AC 6)
Proof: run unit | test_politica.py::test_no_show_ate_24h PASSED

**C34** - No-show antes do início ou após 24 h é recusado (CAN-07, AC 7)
Proof: run unit | test_politica.py::test_no_show_antes_do_inicio_ou_apos_24h_recusado PASSED

**C35** - Link de cancelamento antes do início cancela sem login (CAN-08, AC 8)
Proof: run int | test_link_cancelamento.py::test_link_valido_cancela_sem_login PASSED

**C36** - Link após o início ou com assinatura inválida mostra "Link expirado" (CAN-09, AC 9)
Proof: run int | test_link_cancelamento.py::test_link_expirado_ou_assinatura_invalida PASSED

**C37** - OTP por e-mail tem 6 dígitos e vale 10 minutos (AUTH-01, AC 1)
Proof: run int | test_otp.py::test_otp_6_digitos_valido_10_min PASSED

**C38** - 6ª tentativa em 15 min por identificador ou IP responde 429 (AUTH-02, AC 2)
Proof: run int | test_rate_limit.py::test_sexta_tentativa_em_15_min_429 PASSED

**C39** - OTP usado não vale de novo (AUTH-03, AC 3)
Proof: run int | test_otp.py::test_otp_usado_nao_vale_de_novo PASSED

**C40** - Dono sem 2FA precisa configurar TOTP antes do painel (AUTH-04, AC 4)
Proof: run int | test_senha_totp.py::test_dono_sem_totp_precisa_configurar PASSED

**C41** - Dono precisa de TOTP válido em todo login (AUTH-05, AC 5)
Proof: run int | test_senha_totp.py::test_dono_exige_totp_em_todo_login PASSED

**C42** - Login cria sessão no banco e cookie httpOnly, Secure, SameSite=Lax, 30 dias (AUTH-06, AC 6)
Proof: run int | test_sessoes.py::test_login_cria_sessao_e_cookie_seguro PASSED

**C43** - Sessão revogada responde 401 (AUTH-07, AC 7)
Proof: run int | test_sessoes.py::test_sessao_revogada_401 PASSED

**C44** - RLS impede ler dados de outra barbearia mesmo sem filtro na query (AUTH-08, AC 8)
Proof: run int | test_rls.py::test_rls_isola_tenants PASSED

**C45** - Recurso de outra barbearia responde 404 (AUTH-09, AC 9)
Proof: run int | test_tenancy.py::test_recurso_de_outra_barbearia_404 PASSED

**C46** - Barbeiro não acessa faturamento da casa, dados de outros barbeiros nem configurações (AUTH-10, AC 10)
Proof: run int | test_permissoes.py::test_barbeiro_negado_faturamento_e_configuracoes PASSED

**C47** - Admin não acessa dados pessoais de clientes (AUTH-11, AC 11)
Proof: run int | test_permissoes.py::test_admin_sem_dados_pessoais_de_clientes PASSED

**C48** - Google vincula à conta com mesmo e-mail verificado ou cria conta nova (AUTH-12, AC 12)
Proof: run int | test_google.py::test_google_vincula_ou_cria_conta PASSED

**C49** - Login com senha compara com hash argon2id (AUTH-13, AC 13)
Proof: run int | test_senha_totp.py::test_login_senha_argon2id PASSED

**C50** - Agendamento confirmado gera e-mail com link de cancelamento (NOT-01, AC 1)
Proof: run int | test_jobs_notificacao.py::test_confirmacao_enviada_com_link_de_cancelamento PASSED

**C51** - Lembretes de 24 h e 2 h saem uma vez cada (NOT-02, AC 2)
Proof: run int | test_jobs_notificacao.py::test_lembretes_24h_e_2h_uma_vez_cada PASSED

**C52** - Cancelamento e remarcação notificam cliente e barbeiro por e-mail (NOT-03, AC 3)
Proof: run int | test_jobs_notificacao.py::test_cancelamento_e_remarcacao_notificam_cliente_e_barbeiro PASSED

**C53** - Opt-out suspende lembretes e mantém confirmações e cancelamentos (NOT-04, AC 4)
Proof: run int | test_optout.py::test_optout_suspende_lembretes_mantem_confirmacoes PASSED

**C54** - Falha de envio tenta 3 vezes com backoff e não altera o agendamento (NOT-05, AC 5)
Proof: run int | test_jobs_notificacao.py::test_falha_tenta_3_vezes_sem_alterar_agendamento PASSED

**C55** - Painel reflete mudança em até 30 s sem recarregar (NOT-06, AC 6)
Proof: `cd frontend && pnpm playwright test e2e/painel.spec.ts -g "agenda atualiza em ate 30 segundos"`

**C56** - Só existem templates de e-mail transacionais (NOT-07, AC 7)
Proof: run unit | test_email.py::test_apenas_templates_transacionais PASSED

**C57** - Resumo mostra contagem por status do mês no fuso da barbearia (REL-01, AC 1)
Proof: run int | test_resumo.py::test_contagem_por_status_no_fuso_da_barbearia PASSED

**C58** - Faturamento = soma do preço congelado dos `CONCLUIDO` (REL-02, AC 2)
Proof: run unit | test_metricas.py::test_faturamento_soma_concluidos PASSED

**C59** - Taxa de no-show = NO_SHOW / (CONCLUIDO + NO_SHOW) (REL-03, AC 3)
Proof: run unit | test_metricas.py::test_taxa_de_no_show PASSED

**C60** - Concluir exige forma de pagamento em {dinheiro, pix, cartão} (REL-04, AC 4)
Proof: run int | test_transicoes.py::test_concluir_exige_forma_de_pagamento PASSED

**C61** - Barbeiro vê só os próprios atendimentos no resumo (REL-05, AC 5)
Proof: run int | test_resumo.py::test_barbeiro_ve_apenas_proprios_numeros PASSED

**C62** - Exclusão confirmada por OTP cancela futuros e anonimiza em até 15 dias (LGPD-01, AC 1)
Proof: run int | test_exclusao.py::test_exclusao_cancela_futuros_e_anonimiza PASSED

**C63** - Agendamentos passados de conta excluída mostram "Cliente removido" (LGPD-02, AC 2)
Proof: run int | test_exclusao.py::test_passados_exibem_cliente_removido PASSED

**C64** - Exportação envia JSON com conta e agendamentos por e-mail (LGPD-03, AC 3)
Proof: run int | test_exportacao.py::test_exportacao_envia_json_por_email PASSED

**C65** - Retenção apaga atendimentos > 5 anos e logs > 6 meses, preserva o resto (LGPD-04, AC 4)
Proof: run int | test_retencao.py::test_retencao_apaga_fora_do_prazo PASSED

**C66** - Audit log registra preço, papel, exportação e exclusão com autor e data (LGPD-05, AC 5)
Proof: run int | test_audit.py::test_audit_registra_acoes_sensiveis PASSED

**C67** - Nenhum schema ou model coleta CPF, nascimento ou endereço do cliente (LGPD-06, AC 6)
Proof: run int | test_exclusao.py::test_nenhum_campo_de_cpf_nascimento_endereco PASSED

**C68** - 20 requisições simultâneas no mesmo slot: exatamente 1 sucesso e 19 × 409 (NFR-01, AC 1)
Proof: `k6 run load/contention.js`

**C69** - Mudança de agenda invalida o cache de slots daquele barbeiro e dia (NFR-02, AC 2)
Proof: run int | test_slots_api.py::test_mudanca_de_agenda_invalida_cache PASSED

**C70** - Job de backup envia o dump para storage fora da VPS (NFR-03, AC 3)
Proof: run int | test_backup.py::test_backup_envia_dump_para_storage PASSED

**C71** - Restauração do último backup em container limpo passa no smoke test (NFR-04, AC 4)
Proof: `bash infra/scripts/restore-check.sh`

**C72** - Exceção não tratada responde 500 sem detalhes e é registrada no Sentry (NFR-05, AC 5)
Proof: run int | test_errors.py::test_excecao_nao_tratada_500_e_sentry PASSED

---

## Handoff

**Settled mid-build:** none
**Dispatched:** none
**Walked:** none

---

## Task Granularity Check

Toda task tem um único arquivo em `Where`. Tasks de serviço agrupam regras de um mesmo módulo coeso (⚠️ OK). Nenhuma task cobre múltiplos componentes em arquivos diferentes.

## Diagram-Definition Cross-Check

Verificado por `validate_tasks.py` (paridade intra-fase; dependências entre fases apontam só para fases anteriores).

## Test Co-location Validation

| Task | Layer | Matrix | Task | Status |
| --- | --- | --- | --- | --- |
| T1, T2 | aprendizado | unit | unit | ✅ |
| T3, T4, T9, T10, T11, T48, T65 | config/infra/docs/cliente gerado | none | none | ✅ |
| T6, T7, T18, T25, T31 | migrations/RLS | integration | integration | ✅ |
| T5, T8, T19–T24, T26–T30, T32–T38, T40–T46 | rotas/serviços/worker | integration | integration | ✅ |
| T12–T17 | domínio/schemas | unit | unit | ✅ |
| T39 | adapter puro | unit | unit | ✅ |
| T47 | componentes UI | unit | unit | ✅ |
| T49–T61 | telas web | e2e | e2e | ✅ |
| T62 | backup | integration | integration | ✅ |
| T63 | restauração | integration | integration | ✅ |
| T64 | carga | load (k6) | load (k6) | ✅ |
