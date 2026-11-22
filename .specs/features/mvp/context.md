# MVP Context

**Gathered:** 2026-11-22 (sessão de grill com Mauricio)
**Spec:** `.specs/features/mvp/spec.md`
**Status:** Ready for design

---

## Feature Boundary

SaaS multi-tenant de agendamento para barbearias: cada barbearia (tenant) tem página pública própria, equipe, serviços, agenda e recebimento online via Stripe Connect; a plataforma cobra assinatura (freemium). Um único produto: web responsiva completa (Next.js) + app Expo (fluxos de cliente e barbeiro).

---

## Implementation Decisions

### D1 · Modelo de produto
- SaaS multi-tenant. Sem marketplace/descoberta no MVP.
- Um único aplicativo para todas as barbearias; o tenant é identificado por slug no caminho: `app.com/{slug}`.
- Identidade visual por tenant na página pública: logo, capa e cor de destaque. Layout é o mesmo.
- Subdomínio / domínio próprio: fase 2 (plano Plus).

### D2 · Pagamento do cliente final (revisado em 2026-11-22)
- Modo configurável por barbearia, com 2 opções:
  - **Só no local** — sem Stripe; todo agendamento é "pagar no local".
  - **Cliente escolhe** — barbearia conectou o Stripe; no agendamento o **cliente final escolhe** "Pagar agora" (integral online) ou "Pagar no local". "Pagar agora" vem pré-selecionado.
- Não existe modo "online obrigatório" para todos. A única obrigação de pagar online é a do cliente com 2 faltas em 90 dias naquela barbearia (D6), e só vale em barbearia "Cliente escolhe".
- Stripe é opcional para a barbearia: o onboarding não exige conexão.
- **Sinal (depósito)**: fase 2 (reaproveita o fluxo integral com outro valor).

### D3 · Monetização da plataforma
- Assinatura mensal via Stripe Billing + plano **Free** permanente (sem trial).

| | Free | Pro | Plus |
|---|---|---|---|
| Barbeiros ativos | 1 | até 5 | ilimitado |
| Agendamentos/mês | **20** | ilimitado | ilimitado |
| Pagamento online | sim | sim | sim |
| Página pública | com selo da plataforma | sem selo | sem selo |
| Lembretes | sim | sim | sim |
| Relatórios | mês atual | completo | completo + export CSV |

- Ao atingir 20 agendamentos no mês, a página pública deixa de aceitar novos agendamentos no mês ("agenda indisponível online"); agendamentos existentes nunca são cancelados. Aviso ao dono em 16 (80%) e 20 (100%).
- Comissão da plataforma (application_fee) fica desligada; pode ser ativada depois sem retrabalho.

### D4 · Inadimplência e downgrade
- Falha de cobrança → Smart Retries do Stripe + **7 dias de carência** com banner ao dono.
- Fim da carência → downgrade automático para Free (sem bloqueio).
- Excesso de barbeiros no downgrade → dono escolhe quem fica ativo; demais ficam **inativos** (não excluídos). Sem escolha em 48 h → fica o barbeiro com mais agendamentos nos últimos 30 dias.
- Agendamentos futuros já confirmados de barbeiros inativados são mantidos.
- Re-upgrade reativa barbeiros com um clique.

### D5 · Agendamento
- Fluxo: serviço → barbeiro (ou "qualquer um") → data/horário → identificação → escolha da forma de pagamento (se a barbearia usa Stripe) → pagamento (se escolheu "Pagar agora") → confirmação.
- Slots calculados de jornada semanal + intervalos + bloqueios − agendamentos, grade de **15 min**.
- Antecedência mínima **2 h**, máxima **30 dias** (configuráveis por barbearia).
- "Qualquer um" → barbeiro elegível com menos agendamentos no dia.
- Unicidade garantida no banco (exclusion constraint). Perdedor vê "esse horário acabou de ser reservado" + sugestões.
- Pagamento online: hold `PENDENTE_PAGAMENTO` de **10 min**; webhook confirma; expirado libera o slot.
- 1 serviço por agendamento; combos são serviços próprios.

### D6 · Cancelamento, reembolso, no-show
- Janela de cancelamento padrão **4 h** (configurável).
- Cliente cancela ≥ janela → online: reembolso integral automático; local: sem custo.
- Cliente cancela < janela → online: sem reembolso; local: registrado.
- Barbearia cancela → reembolso integral sempre.
- No-show marcado pelo barbeiro até 24 h após o horário; online: valor fica com a barbearia.
- Remarcar: só dentro da janela; pagamento é transferido.
- 2 no-shows em 90 dias na mesma barbearia → em barbearia "Cliente escolhe", a opção "Pagar no local" some para esse cliente. Em barbearia "Só no local" a falta só fica registrada.
- Cancelamento via link assinado (expira no horário do atendimento) ou pela conta.
- Máquina de estados: `PENDENTE_PAGAMENTO → CONFIRMADO → CONCLUIDO | CANCELADO_CLIENTE | CANCELADO_BARBEARIA | NO_SHOW`; `PENDENTE_PAGAMENTO → EXPIRADO`. Nenhuma outra transição.

### D7 · Autenticação e papéis
- Papéis: Cliente (global), Barbeiro (1 barbearia), Dono (N barbearias), Admin da plataforma (sem acesso a dados de pagamento). Papéis acumuláveis.
- Cliente: OTP de 6 dígitos por **WhatsApp** ou **e-mail**, ou **Google** (e **Sign in with Apple** no app iOS, exigência da guideline 4.8). Sem senha. Fallback "receber por e-mail" quando WhatsApp falhar.
- Dono/barbeiro: e-mail + senha ou Google. **2FA TOTP obrigatório para o dono.**
- Barbeiro entra por convite (link 7 dias).
- Sessão: access token 15 min + refresh rotativo 30 dias em cookie httpOnly (web) / secure storage (app); revogação pelo painel.
- Rate limit: 5 tentativas / 15 min por identificador e por IP.
- Isolamento: `barbearia_id` em toda query + RLS no Postgres.
- Sem agendamento como convidado.

### D8 · Notificações
| Evento | Cliente | Barbeiro | Dono |
|---|---|---|---|
| Agendamento confirmado | WhatsApp + e-mail | painel | — |
| Lembrete 24 h e 2 h antes | WhatsApp | — | — |
| Cancelado/remarcado | WhatsApp + e-mail | painel + e-mail | — |
| Reembolso processado | e-mail | — | — |
| Hold expirou | e-mail | — | — |
| Limite 80/100%, cobrança falhou, downgrade | — | — | e-mail + banner |
- Cliente escolhe canal preferido; e-mail sempre como fallback/registro.
- Opt-out "SAIR" desliga lembretes, não confirmações.
- Fila assíncrona, até 3 tentativas; falha de notificação nunca afeta o agendamento.
- Painel em tempo real (SSE/WebSocket). Push nativo no app Expo (complemento).
- Sem marketing no MVP.

### D9 · Plataformas
- Web responsiva completa (cliente, barbeiro, dono) + **app Expo** com fluxos de cliente e barbeiro (um app, telas por papel).
- Configuração, equipe, plano, Stripe e relatórios **só na web** (evita IAP da Apple; planos não são vendidos no app).
- Exclusão de conta disponível no app (Apple 5.1.1(v) + LGPD).

### D10 · Dinheiro do cliente (Stripe Connect)
- Connect **Express**, onboarding hospedado pelo Stripe; nunca tocamos dados bancários.
- Modo "Cliente escolhe" só habilitado após onboarding aprovado (`charges_enabled`). Se a conta perder `charges_enabled`, a barbearia volta a "Só no local" para novos agendamentos.
- Taxa Stripe absorvida pela barbearia; cliente paga o preço exato.
- Repasse automático no prazo padrão do Stripe; plataforma não retém saldo.
- Chargeback é da barbearia (alerta no painel).
- Recibo Stripe em nome da barbearia. NFS-e fora do MVP.
- Cartão garantido; Pix condicionado (ver Assumptions na spec).

### D11 · Serviços e preços
- Serviço: nome, descrição, duração, preço base, foto opcional, categoria (Cabelo, Barba, Combo, Outros).
- Override de preço e duração por barbeiro; dono define quais barbeiros fazem cada serviço.
- "Qualquer um" mostra faixa de preço até atribuição; preço final antes de confirmar/pagar.
- Preço congelado no agendamento.
- Serviço desativado some da página; agendamentos futuros mantidos; sem hard delete com histórico.

### D12 · Relatórios e comissão
- Dono (web): faturamento (online × local), agendamentos por status, taxa de no-show, ocupação por barbeiro, ranking de serviços, recorrentes × novos; filtros por período/barbeiro/serviço; CSV no Plus; Free só mês atual.
- Comissão: % por barbeiro (padrão 0%), só cálculo, sobre `CONCLUIDO`, preço congelado, antes da taxa Stripe; no-show retido sem comissão.
- Barbeiro vê apenas os próprios números e comissão estimada.
- Ao concluir atendimento pago no local, barbeiro informa forma (dinheiro, Pix, cartão na maquininha) — só para relatório.

### D13 · LGPD
- Plataforma = controladora dos dados de conta; barbearia = controladora dos dados de atendimento, plataforma = operadora.
- Bases legais: execução de contrato (transacional), legítimo interesse com opt-out (lembretes).
- Exclusão de conta: anonimização em até 15 dias; histórico mantido anonimizado; futuros cancelados pela política.
- Exportação de dados: JSON assíncrono por e-mail.
- Retenção: financeiro 5 anos; logs de acesso 6 meses; OTP 10 min; holds expirados 30 dias.
- Barbearia que encerra: página desativada na hora; export por 90 dias; depois anonimização (exceto financeiro).
- Minimização: sem CPF/nascimento/endereço do cliente; cartão nunca passa pelo servidor.
- Audit log de ações sensíveis.

### D14 · Metas não-funcionais
- 1.000 barbearias, ~4.000 barbeiros, ~300 mil clientes, ~40 mil agendamentos/dia.
- Pico: 2.000 usuários simultâneos; 50 agendamentos/s; 20 disputas pelo mesmo slot → exatamente 1 vence.
- p95 < 300 ms (página pública e slots); p95 < 500 ms (criar agendamento, sem Stripe).
- 99,5% disponibilidade mensal; RPO 24 h; RTO 4 h.
- Webhooks idempotentes e ordenados por agendamento.
- Teste de carga k6 antes de declarar MVP pronto.
- **Com infra 0800:** as metas acima são metas de **arquitetura**, provadas por k6 contra o stack em Docker Compose dimensionado como a VM de produção. O free tier em produção atende o volume real inicial; o ponto de migração para plano pago é disparado por métrica (ver rollout.md), não por data.

### D15 · Stack e divisão
- Monorepo pnpm + Turborepo, TypeScript ponta a ponta.
- Next.js (web + BFF), Expo (app), `packages/ui` (design system com tokens compartilhados), NestJS (API), PostgreSQL + Drizzle, Redis + BullMQ, OpenTelemetry + Sentry; Docker Compose local.
- **Infra custo zero ("0800")** — decisão do Mauricio em 2026-11-22, substitui a proposta inicial de AWS:
  - Web Next.js → **Vercel Hobby** (uso não-comercial; migrar para plano pago ou para a VM quando houver receita).
  - API NestJS + workers BullMQ + Redis → **1 VM Oracle Cloud Always Free (Ampere A1)** com Docker Compose + Caddy (TLS automático). Redis self-hosted na VM porque BullMQ estoura a cota mensal de comandos de Redis serverless gratuito.
  - PostgreSQL → **Neon Free** (branching por PR, backups gerenciados; suporta RLS e `btree_gist`).
  - E-mail → **Resend Free**; erros → **Sentry Developer**; traces/métricas → **Grafana Cloud Free** via OpenTelemetry; CI → **GitHub Actions**; builds mobile → **EAS Free**.
  - Containers e IaC portáveis: nada acoplado a um provedor; trocar de VM/DB é troca de variável de ambiente.
- **Custos que não são evitáveis** (não são infra): Apple Developer (anual), Google Play (taxa única), mensagens WhatsApp (por template de autenticação/utilidade), taxa Stripe por transação (absorvida pela barbearia), domínio (opcional).
- Divisão por domínio vertical:
  - **Mauricio** — Agenda & agendamento, página pública, fluxo do cliente, design system, app Expo.
  - **Lorenzo** — Pagamentos (Connect, Checkout, webhooks, reembolso), planos/Billing/freemium/downgrade, relatórios e comissão.
  - **Mateus** — Identidade & plataforma (auth, papéis, RLS), notificações/filas, LGPD, infra, CI/CD, observabilidade.

### Agent's Discretion
- Grade de 15 min, prazos LGPD (15/90 dias), metas de NFR: propostos pelo agente e aceitos.
- Detalhes de UI (layout, componentes) ficam a critério do design no FigCraft, respeitando as decisões acima.

### Declined / Undiscussed Gray Areas → Assumptions
Registrados na seção Assumptions & Open Questions da spec: fuso horário, múltiplas unidades, onboarding da barbearia e slug, escopo do admin, Pix, idioma/moeda, WhatsApp provider.

---

## Specific References

- Booksy / Trinks como referência de "um app nas lojas para N estabelecimentos".
- Estrutura de pastas e páginas FigCraft no mesmo padrão do projeto cutscore (Design system, Desktop · Light/Dark, Mobile · Light/Dark).

---

## Deferred Ideas

- Marketplace / busca de barbearias por localização.
- Sinal (depósito) como modo de pagamento.
- Comissão da plataforma por transação.
- Subdomínio / domínio próprio (Plus).
- Produtos, pacotes/assinatura do cliente, cupons, gorjeta.
- Split automático para o barbeiro (conta Stripe por barbeiro).
- NFS-e.
- Marketing (WhatsApp/e-mail) com consentimento.
- App white-label por barbearia.
- Múltiplos serviços por agendamento.
