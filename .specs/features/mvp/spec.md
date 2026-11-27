# MVP Specification — Barbezy (SaaS multi-tenant de agendamento)

**Approved**: pendente
**Escopo:** Complex · **Contexto:** `.specs/features/mvp/context.md` (decisões D1–D16) · **Data:** 2026-11-26 (revisão de stack e escopo; versão original de 2026-11-22)

## Problem Statement

Barbearias pequenas e médias agendam pelo WhatsApp e pelo caderno. Perdem horários por conflito, não medem no-show e não têm agenda que o cliente consulte sozinho. O cliente não vê horários livres nem agenda fora do horário comercial. Queremos um SaaS multi-tenant em que cada barbearia tenha página pública de agendamento, agenda calculada, lembretes por e-mail e um resumo mensal. Esta revisão reduz o MVP ao que uma pessoa entrega sozinha: pagamento online, planos, WhatsApp e app mobile passam para a fase 2.

## Goals

- [ ] Barbearia sai do cadastro para a página pública aceitando agendamentos em menos de 15 minutos.
- [ ] Cliente conclui um agendamento em menos de 60 segundos a partir do link da barbearia.
- [ ] Zero agendamentos duplicados no mesmo horário sob 20 disputas simultâneas.
- [ ] Taxa de no-show mensurável por barbearia desde o primeiro mês.
- [ ] Link da barbearia compartilhado no WhatsApp mostra nome e imagem da barbearia.
- [ ] Produção roda numa única VPS, com backup diário fora dela e restauração testada.

## Out of Scope

| Feature | Reason |
| --- | --- |
| Pagamento online (Stripe Connect, Checkout, reembolso, Pix) | Fase 2 (D2, D10). No MVP todo agendamento é "pagar no local" |
| Planos pagos, freemium, limites e downgrade (Stripe Billing) | Fase 2 (D3, D4). No MVP toda barbearia usa tudo sem limite |
| WhatsApp (OTP, lembretes, "SAIR") | Fase 2 (D7, D8). Verificação Meta e templates custam tempo; e-mail cobre o MVP |
| App mobile Expo, push nativo, Sign in with Apple | Fase 2 (D9). MVP é web responsiva instalável (PWA) |
| Relatórios completos (ocupação, comissão, filtros, CSV) | Fase 2 (D12). MVP tem resumo mensal simples |
| Painel em tempo real (SSE/WebSocket) | Polling de 30 s atende o barbeiro (D8) |
| Encerramento de barbearia com janela de 90 dias para exportação | Fase 2 (D13) |
| Testes de carga de volume (2.000 usuários, 50 agendamentos/s) | Fase 2. MVP prova só a disputa de horário (D14) |
| Ambiente de homologação | Fase 2. MVP tem local + produção (D15) |
| Marketplace / busca de barbearias | Exige busca geográfica, avaliações e moderação (D1) |
| Sinal (depósito) | Depende de pagamento online (D2) |
| Subdomínio / domínio próprio por barbearia | Fase 2 (D1) |
| Produtos, pacotes, cupons, gorjeta | Fora do MVP (D11) |
| NFS-e | Responsabilidade da barbearia (D10) |
| Marketing por e-mail | Exige consentimento (D8) |
| Múltiplos serviços por agendamento | Combos são serviços próprios (D5) |
| SMS | Custo alto e entrega pior (D7) |
| App white-label por barbearia | Plano enterprise futuro |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Fuso horário | Cada barbearia tem fuso IANA (padrão `America/Sao_Paulo`); horários em UTC no banco, exibidos no fuso da barbearia | Brasil tem 4 fusos; UTC evita bugs de horário de verão | n |
| Idioma e moeda | pt-BR e BRL apenas | Mercado inicial é Brasil | n |
| Múltiplas unidades | Cada unidade é uma barbearia (tenant) separada; dono alterna por seletor | Reaproveita o modelo multi-tenant | n |
| Onboarding da barbearia | Dono cria conta → 2FA → nome, slug, endereço, fuso, logo → primeiro serviço → jornada própria | Caminho mais curto até a página pública | n |
| Slug | 3–40 chars `[a-z0-9-]`, único global, lista de reservados (`painel`, `api`, `admin`, `entrar`, `conta`, `assets`, …); alterável 1x a cada 30 dias com redirect 301 do antigo | Links divulgados no Instagram não podem quebrar | n |
| Escopo do admin da plataforma | Listar/buscar barbearias, suspender/reativar por abuso, ver audit log; sem dados pessoais de clientes além de contagens | Menor privilégio (D7) | n |
| Feriados | Bloqueios avulsos criados pelo dono; sem calendário automático | Feriados municipais variam | n |
| Forma de pagamento no MVP | Todo agendamento é "pagar no local"; a página não mostra opção online | Stripe adiado para a fase 2 | y |
| Limites por plano no MVP | Nenhum: barbeiros e agendamentos ilimitados para toda barbearia; sem selo de plano na página | Planos adiados para a fase 2 | y |
| Login do cliente no MVP | Código por e-mail ou Google; sem WhatsApp e sem Sign in with Apple | WhatsApp e app adiados | y |
| Falta recorrente no MVP | `NO_SHOW` só fica registrado; nenhuma restrição ao cliente | A restrição exige pagamento online | y |
| Cancelamento após o início | Recusado (409); depois do início o barbeiro marca concluído ou no-show | Mantém o link assinado coerente com a expiração no início | n |
| Sessão de login | Sessão guardada no banco + cookie httpOnly, `Secure`, `SameSite=Lax`, validade 30 dias | Site e API no mesmo domínio; mais simples que JWT (D7) | y |
| Infra | 1 VPS Hostinger (Brasil) com Docker Compose: Caddy, site, API, worker, Redis, PostgreSQL; DNS na Cloudflare | Decisão do usuário em 2026-11-26 (D15) | y |
| Backup | `pg_dump` diário enviado para Backblaze B2 ou Cloudflare R2; restauração testada todo mês | Banco na VPS exige backup fora dela (D15) | y |
| Provedor de e-mail | Resend, com SPF/DKIM no domínio próprio | Faixa gratuita e API simples | y |
| Painel do barbeiro | Atualiza por polling a cada 30 s | Simplificação aceita (D8) | y |
| Prévia do link | FastAPI devolve `index.html` com meta tags Open Graph preenchidas em `/{slug}` | Prévia no WhatsApp sem servidor Node (D9) | y |

**Open questions:** none - all resolved or logged above.

---

## User Stories

### P1: Cadastro e configuração da barbearia ⭐ MVP

**User Story**: As a dono, I want cadastrar minha barbearia com equipe, serviços e jornadas so that minha página pública fique pronta para receber agendamentos.

**Why P1**: Sem tenant configurado não existe produto.

**Acceptance Criteria**:

1. WHEN o dono conclui o onboarding THEN o sistema SHALL criar a barbearia com página pública ativa em `/{slug}`.
2. IF o slug informado já existe, é reservado ou não casa `^[a-z0-9-]{3,40}$` THEN o sistema SHALL rejeitar com a mensagem "Endereço indisponível" sem criar a barbearia.
3. WHEN o dono altera o slug THEN o sistema SHALL responder 301 do slug antigo para o novo por 12 meses.
4. IF o dono tenta alterar o slug menos de 30 dias após a última alteração THEN o sistema SHALL rejeitar a alteração.
5. WHEN o dono envia convite a um e-mail THEN o sistema SHALL enviar link de convite válido por 7 dias.
6. IF um convite expirado ou já usado é aberto THEN o sistema SHALL exibir "Convite inválido" e não criar vínculo.
7. WHEN o dono salva a jornada semanal de um barbeiro THEN o sistema SHALL rejeitar faixas sobrepostas.
8. WHEN o dono personaliza logo, capa e cor de destaque THEN a página pública SHALL exibi-los no próximo carregamento.
9. WHEN a URL `/{slug}` é requisitada THEN o sistema SHALL devolver HTML com `og:title` igual ao nome da barbearia e `og:image` igual à capa, ou ao logo quando não há capa.

**Independent Test**: Criar barbearia, convidar barbeiro, cadastrar jornada e serviço, abrir `/{slug}` e ver o serviço listado; colar o link no WhatsApp e ver nome e imagem.

---

### P1: Serviços e preços ⭐ MVP

**User Story**: As a dono, I want cadastrar serviços com preço e duração, inclusive por barbeiro, so that a agenda e o preço reflitam a realidade da casa.

**Why P1**: Slots dependem de duração; o cliente decide pelo preço.

**Acceptance Criteria**:

1. WHEN o dono cria um serviço THEN o sistema SHALL exigir nome, duração múltipla de 15 min entre 15 e 240 min, preço ≥ R$ 0 e categoria em {Cabelo, Barba, Combo, Outros}.
2. WHEN o dono define override de preço ou duração para um barbeiro THEN o sistema SHALL usar o override para agendamentos com aquele barbeiro.
3. WHEN o dono desabilita um barbeiro para um serviço THEN o sistema SHALL excluir aquele barbeiro dos slots daquele serviço.
4. WHEN um agendamento é criado THEN o sistema SHALL gravar preço e duração vigentes no agendamento (preço congelado).
5. WHEN o dono altera o preço de um serviço THEN o sistema SHALL manter o preço dos agendamentos já criados.
6. WHEN o dono desativa um serviço THEN o sistema SHALL removê-lo da página pública e manter seus agendamentos futuros.
7. IF o dono tenta excluir um serviço com agendamentos THEN o sistema SHALL recusar e oferecer desativação.

**Independent Test**: Criar serviço com override para um barbeiro, agendar com ele, mudar o preço base e conferir que o agendamento manteve o valor.

---

### P1: Agendamento pelo cliente ⭐ MVP

**User Story**: As a cliente, I want escolher serviço, barbeiro e horário livre so that eu agende sem trocar mensagens.

**Why P1**: É o núcleo do produto.

**Acceptance Criteria**:

1. The system SHALL calcular slots a partir da jornada e dos bloqueios do barbeiro menos agendamentos ocupantes, em grade de 15 minutos, no fuso da barbearia.
2. The system SHALL oferecer apenas slots com início ≥ agora + antecedência mínima (padrão 2 h) e ≤ hoje + antecedência máxima (padrão 30 dias).
3. WHEN o cliente escolhe "qualquer um disponível" THEN o sistema SHALL atribuir o barbeiro elegível com menos agendamentos ocupantes naquele dia, desempatando pelo id do barbeiro.
4. WHEN o cliente escolhe "qualquer um disponível" THEN o sistema SHALL exibir a faixa de preço (mín–máx entre barbeiros elegíveis) até a atribuição e o preço final antes da confirmação.
5. The system SHALL garantir no banco que dois agendamentos ocupantes do mesmo barbeiro nunca se sobreponham no tempo.
6. IF a criação falha por conflito de horário THEN o sistema SHALL responder 409 com a mensagem "Esse horário acabou de ser reservado" e até 3 slots livres mais próximos.
7. WHEN o cliente confirma o agendamento THEN o sistema SHALL criá-lo em `CONFIRMADO` com forma "Pagar no local".
8. IF o cliente não está autenticado ao confirmar THEN o sistema SHALL exigir login antes de criar o agendamento, preservando a seleção.
9. The system SHALL permitir apenas as transições `CONFIRMADO→CONCLUIDO|CANCELADO_CLIENTE|CANCELADO_BARBEARIA|NO_SHOW`.
10. IF uma transição fora da lista é solicitada THEN o sistema SHALL responder 409 e não alterar o agendamento.
11. IF a mesma requisição de criação chega duas vezes com o mesmo `Idempotency-Key` THEN o sistema SHALL devolver o mesmo agendamento sem criar outro.

**Independent Test**: Duas abas agendando o mesmo slot ao mesmo tempo → uma confirma, a outra recebe 409 com sugestões.

---

### P1: Cancelamento, remarcação e no-show ⭐ MVP

**User Story**: As a cliente ou barbearia, I want cancelar e remarcar com regras claras so that ninguém seja surpreendido.

**Why P1**: Sem política, a agenda fica cheia de horários fantasmas.

**Acceptance Criteria**:

1. WHEN o cliente cancela um agendamento antes do início THEN o sistema SHALL movê-lo para `CANCELADO_CLIENTE` sem cobrança.
2. IF o cliente tenta cancelar depois do início THEN o sistema SHALL responder 409 e não alterar o agendamento.
3. WHEN a barbearia cancela um agendamento THEN o sistema SHALL movê-lo para `CANCELADO_BARBEARIA`.
4. WHEN o cliente remarca com antecedência ≥ janela (padrão 4 h) THEN o sistema SHALL mover o agendamento para o novo slot.
5. IF o cliente tenta remarcar com antecedência < janela THEN o sistema SHALL recusar a remarcação.
6. WHEN o barbeiro marca no-show até 24 h após o início THEN o sistema SHALL mover o agendamento para `NO_SHOW`.
7. IF o barbeiro tenta marcar no-show antes do início ou mais de 24 h após o início THEN o sistema SHALL recusar.
8. WHEN o cliente abre o link de cancelamento do e-mail antes do início do atendimento THEN o sistema SHALL permitir cancelar sem login.
9. IF o link de cancelamento é aberto após o início do atendimento ou tem assinatura inválida THEN o sistema SHALL exibir "Link expirado".

**Independent Test**: Agendar para daqui a 5 h, remarcar (aceito); agendar para daqui a 3 h, remarcar (recusado); cancelar pelo link do e-mail sem login.

---

### P1: Autenticação e papéis ⭐ MVP

**User Story**: As a usuário, I want entrar de forma rápida e segura conforme meu papel so that eu acesse só o que me cabe.

**Why P1**: Multi-tenant sem isolamento é vazamento de dados.

**Acceptance Criteria**:

1. WHEN o cliente solicita código por e-mail THEN o sistema SHALL enviar OTP de 6 dígitos válido por 10 minutos.
2. IF um identificador ou IP excede 5 tentativas de login/OTP em 15 minutos THEN o sistema SHALL responder 429 até o fim da janela.
3. WHEN o OTP é usado com sucesso THEN o sistema SHALL invalidá-lo para qualquer uso posterior.
4. WHEN um dono faz login sem 2FA configurado THEN o sistema SHALL exigir configuração de TOTP antes de acessar o painel.
5. WHILE o usuário tem papel de dono o sistema SHALL exigir TOTP válido em todo login.
6. WHEN o login conclui THEN o sistema SHALL criar uma sessão no banco e devolver cookie httpOnly, `Secure`, `SameSite=Lax`, com validade de 30 dias.
7. WHEN o usuário revoga uma sessão no painel THEN o sistema SHALL responder 401 a requisições com o cookie daquela sessão.
8. The system SHALL filtrar toda leitura e escrita de dados de tenant pela barbearia do contexto, com RLS no Postgres como segunda barreira.
9. IF um usuário acessa recurso de outra barbearia THEN o sistema SHALL responder 404.
10. WHILE o usuário é barbeiro o sistema SHALL negar acesso ao faturamento da casa, a dados de outros barbeiros e às configurações.
11. The admin da plataforma SHALL ter acesso negado a dados pessoais de clientes.
12. WHEN o usuário entra com Google THEN o sistema SHALL vincular a identidade Google à conta com o mesmo e-mail verificado, ou criar uma conta nova quando não existir.
13. WHEN o dono ou barbeiro entra com e-mail e senha corretos THEN o sistema SHALL autenticá-lo comparando com o hash argon2id armazenado.

**Independent Test**: Dono da barbearia A tenta GET de agendamento da barbearia B → 404; consulta direta no banco com o role da aplicação e `barbearia_id` de A não retorna linhas de B.

---

### P1: Notificações por e-mail ⭐ MVP

**User Story**: As a cliente, I want ser avisado e lembrado do meu horário so that eu não esqueça nem falte.

**Why P1**: Lembrete é a maior alavanca contra no-show.

**Acceptance Criteria**:

1. WHEN um agendamento é criado em `CONFIRMADO` THEN o sistema SHALL enviar e-mail de confirmação ao cliente com link de cancelamento.
2. WHEN faltam 24 horas e 2 horas para um agendamento `CONFIRMADO` THEN o sistema SHALL enviar lembrete por e-mail.
3. WHEN um agendamento é cancelado ou remarcado THEN o sistema SHALL enviar e-mail ao cliente e ao barbeiro.
4. IF o cliente desativou lembretes pelo link do e-mail THEN o sistema SHALL suspender lembretes e manter confirmações e cancelamentos.
5. IF o envio de uma notificação falha THEN o sistema SHALL tentar novamente até 3 vezes com backoff e registrar a falha sem alterar o agendamento.
6. WHEN um agendamento é criado, cancelado ou alterado THEN o painel do barbeiro e do dono SHALL refletir a mudança sem recarregar a página em até 30 segundos.
7. The system SHALL não enviar mensagens de marketing.

**Independent Test**: Criar agendamento para daqui a 25 h e avançar o relógio da fila → lembretes de 24 h e 2 h disparam uma vez cada.

---

### P1: Resumo mensal ⭐ MVP

**User Story**: As a dono, I want ver um resumo do mês so that eu saiba quanto a casa atendeu e quanto faltaram.

**Why P1**: Dor real do dono e base para os relatórios completos da fase 2.

**Acceptance Criteria**:

1. WHEN o dono abre o resumo de um mês THEN o sistema SHALL exibir a contagem de agendamentos por status no mês, no fuso da barbearia.
2. The system SHALL calcular o faturamento do mês como a soma do preço congelado dos agendamentos `CONCLUIDO`.
3. The system SHALL calcular a taxa de no-show como `NO_SHOW / (CONCLUIDO + NO_SHOW)` no mês.
4. WHEN o barbeiro marca `CONCLUIDO` um agendamento THEN o sistema SHALL exigir a forma de pagamento em {dinheiro, pix, cartão}.
5. WHILE o usuário é barbeiro o sistema SHALL exibir apenas os próprios atendimentos no resumo.

**Independent Test**: Seed com 10 concluídos (R$ 50) e 2 no-shows → faturamento R$ 500,00 e taxa de no-show 16,7%.

---

### P1: Privacidade e LGPD ⭐ MVP

**User Story**: As a cliente, I want controlar meus dados so that eu confie na plataforma.

**Why P1**: Exigência legal.

**Acceptance Criteria**:

1. WHEN o cliente solicita exclusão de conta e confirma por OTP THEN o sistema SHALL cancelar agendamentos futuros e anonimizar os dados pessoais em até 15 dias.
2. The system SHALL manter agendamentos passados de contas excluídas com cliente exibido como "Cliente removido".
3. WHEN o cliente solicita exportação THEN o sistema SHALL enviar por e-mail, em até 24 horas, um JSON com conta e agendamentos.
4. The system SHALL reter registros de atendimento por 5 anos e logs de acesso por 6 meses, apagando o que passar do prazo.
5. The system SHALL registrar em audit log mudanças de preço, papel, exportações e exclusões, com autor e data.
6. The system SHALL não coletar CPF, data de nascimento ou endereço do cliente.

**Independent Test**: Cliente com 1 agendamento passado e 1 futuro exclui a conta → futuro cancelado, passado aparece como "Cliente removido", e-mail sem PII após o job de anonimização.

---

### P1: Operação e confiabilidade ⭐ MVP

**User Story**: As a plataforma, I want aguentar disputas sem duplicar agendamentos e recuperar dados após falha so that crescer não quebre a confiança.

**Why P1**: Uma VPS única exige backup e prova de concorrência.

**Acceptance Criteria**:

1. WHEN 20 requisições simultâneas tentam o mesmo slot THEN o sistema SHALL confirmar exatamente 1 e responder 409 às demais.
2. WHEN um agendamento ou bloqueio altera a agenda de um barbeiro em um dia THEN o sistema SHALL invalidar o cache de slots daquele barbeiro e dia.
3. WHEN o job diário de backup roda THEN o sistema SHALL enviar o dump do banco para um storage fora da VPS.
4. WHEN o script de restauração roda sobre o último backup THEN o sistema SHALL restaurar o banco num container limpo e passar o smoke test.
5. IF a API lança uma exceção não tratada THEN o sistema SHALL responder 500 sem detalhes internos e registrar o erro no Sentry.

**Independent Test**: Rodar o cenário k6 `contention.js` → 1 sucesso, 19 × 409, zero duplicados no banco.

---

### P3: Pagamento online do cliente (Stripe Connect) — fase 2

**User Story**: As a dono, I want receber online pelos agendamentos so that eu reduza no-show e receba direto na minha conta.

**Why P3**: Adiado para a fase 2 (dev solo). Critérios preservados da versão de 2026-11-22.

**Acceptance Criteria**:

1. WHEN o dono clica "Conectar Stripe" THEN o sistema SHALL criar conta Connect Express e redirecionar ao onboarding hospedado pelo Stripe.
2. WHILE a conta conectada não tem `charges_enabled` o sistema SHALL manter a barbearia no modo "Só no local".
3. WHERE a barbearia está no modo "Cliente escolhe" the system SHALL exibir as formas "Pagar agora" (pré-selecionada) e "Pagar no local" antes da confirmação.
4. WHEN o cliente confirma com a forma "Pagar agora" THEN o sistema SHALL criar o agendamento em `PENDENTE_PAGAMENTO` com expiração em 10 minutos.
5. WHILE o agendamento está em `PENDENTE_PAGAMENTO` o sistema SHALL tratá-lo como ocupante do horário.
6. WHEN a expiração de um `PENDENTE_PAGAMENTO` é atingida sem pagamento THEN o sistema SHALL movê-lo para `EXPIRADO` e liberar o horário.
7. WHEN o webhook de pagamento bem-sucedido chega para um agendamento `PENDENTE_PAGAMENTO` THEN o sistema SHALL movê-lo para `CONFIRMADO`.
8. IF o webhook de pagamento bem-sucedido chega para um agendamento `EXPIRADO` THEN o sistema SHALL emitir reembolso integral e notificar o cliente.
9. The system SHALL processar cada evento de webhook no máximo uma vez, identificado pelo id do evento.
10. IF a assinatura do webhook é inválida THEN o sistema SHALL responder 400 e não processar o evento.
11. The system SHALL nunca receber, armazenar ou registrar em log dados de cartão.
12. WHEN o cliente cancela um agendamento pago online com antecedência ≥ janela THEN o sistema SHALL emitir reembolso integral.
13. WHEN o cliente cancela um agendamento pago online com antecedência < janela THEN o sistema SHALL cancelar sem reembolso.
14. WHEN a barbearia cancela um agendamento pago online THEN o sistema SHALL emitir reembolso integral.
15. WHEN um cliente com 2 `NO_SHOW` nos últimos 90 dias agenda numa barbearia "Cliente escolhe" THEN o sistema SHALL oferecer apenas "Pagar agora".
16. WHERE pagamento online está habilitado o sistema SHALL rejeitar preço de serviço menor que R$ 5,00.
17. WHERE a flag Pix está ligada e a conta conectada tem a capability Pix ativa o sistema SHALL oferecer Pix.

**Independent Test**: Em modo teste do Stripe, agendar com cartão de teste → webhook confirma → `CONFIRMADO`; reenviar o mesmo evento → nenhum efeito.

---

### P3: Planos, freemium e downgrade — fase 2

**User Story**: As a plataforma, I want cobrar assinatura com plano gratuito limitado so that barbearias validem de graça e paguem para crescer.

**Why P3**: Adiado para a fase 2. Critérios preservados da versão de 2026-11-22.

**Acceptance Criteria**:

1. WHEN o dono assina Pro ou Plus na web THEN o sistema SHALL criar assinatura no Stripe Billing e aplicar o plano ao receber o webhook de assinatura ativa.
2. WHILE a barbearia está no plano Free o sistema SHALL impedir mais de 1 barbeiro ativo.
3. WHILE a barbearia está no plano Pro o sistema SHALL impedir mais de 5 barbeiros ativos.
4. WHEN a contagem mensal de uma barbearia Free atinge 16 THEN o sistema SHALL notificar o dono (80%).
5. WHEN a contagem mensal de uma barbearia Free atinge 20 THEN o sistema SHALL notificar o dono e ocultar os slots do mês com "Agenda indisponível online".
6. IF a barbearia Free já tem 20 agendamentos no mês THEN o sistema SHALL recusar novos agendamentos com 403 e manter os existentes.
7. WHEN uma cobrança de assinatura falha THEN o sistema SHALL marcar a barbearia em carência por 7 dias e exibir banner ao dono.
8. WHEN a carência termina sem pagamento THEN o sistema SHALL rebaixar a barbearia para Free.
9. WHEN uma barbearia é rebaixada com mais barbeiros ativos que o limite THEN o sistema SHALL pedir ao dono que escolha quais permanecem ativos.
10. IF o dono não escolhe em 48 horas THEN o sistema SHALL manter ativo o barbeiro com mais agendamentos nos últimos 30 dias.
11. WHILE a barbearia está no plano Free a página pública SHALL exibir o selo da plataforma.

**Independent Test**: Barbearia Pro com 3 barbeiros; simular falha de cobrança e avançar 7 dias → Free; após 48 h sem escolha, só o barbeiro mais ocupado fica ativo.

---

### P3: WhatsApp — fase 2

**User Story**: As a cliente, I want receber código e lembretes no WhatsApp so that eu use o canal que já leio.

**Why P3**: Adiado para a fase 2 (verificação Meta e templates).

**Acceptance Criteria**:

1. WHEN o cliente solicita código por WhatsApp THEN o sistema SHALL enviar OTP de 6 dígitos pelo template `auth_code`.
2. IF o envio por WhatsApp falha THEN o sistema SHALL oferecer "Receber por e-mail" na mesma tela.
3. WHEN faltam 24 horas e 2 horas para um agendamento `CONFIRMADO` de cliente com canal WhatsApp THEN o sistema SHALL enviar lembrete por WhatsApp.
4. IF o cliente respondeu "SAIR" no WhatsApp THEN o sistema SHALL suspender lembretes e manter confirmações.

**Independent Test**: Com número de teste da Meta, pedir OTP por WhatsApp e entrar.

---

### P3: App mobile — fase 2

**User Story**: As a cliente ou barbeiro, I want um app nas lojas so that eu agende e acompanhe a agenda com push.

**Why P3**: Adiado para a fase 2. A web responsiva (PWA) cobre o MVP.

**Acceptance Criteria**:

1. WHERE o app iOS oferece login com Google o app SHALL oferecer Sign in with Apple.
2. WHERE o app está instalado com permissão de push o sistema SHALL enviar push ao barbeiro para novo agendamento e cancelamento.
3. The app mobile SHALL oferecer exclusão de conta sem sair do app.
4. The app mobile SHALL não oferecer compra ou troca de plano.

**Independent Test**: Build de preview instalada; barbeiro recebe push ao criar agendamento pela web.

---

### P3: Relatórios completos e encerramento — fase 2

**User Story**: As a dono, I want relatórios detalhados e comissão so that eu pague a equipe sem planilha.

**Why P3**: Adiado para a fase 2.

**Acceptance Criteria**:

1. The system SHALL calcular ocupação por barbeiro como minutos ocupados por `CONFIRMADO`/`CONCLUIDO` sobre minutos da jornada no período.
2. The system SHALL calcular comissão do barbeiro como percentual configurado × soma do preço congelado dos seus `CONCLUIDO` no período.
3. WHERE a barbearia está no plano Plus o sistema SHALL permitir exportar o relatório em CSV.
4. WHEN uma barbearia encerra a conta THEN o sistema SHALL desativar a página pública imediatamente e anonimizar dados não financeiros após 90 dias.
5. WHILE o sistema recebe 2.000 usuários simultâneos no cenário k6 de referência a consulta de slots SHALL manter p95 < 300 ms.

**Independent Test**: Seed com 10 concluídos (R$ 50) para barbeiro com 40% → comissão R$ 200,00.

---

## Edge Cases

- IF o relógio do cliente está errado THEN o sistema SHALL usar sempre o horário do servidor para antecedência e janelas.
- IF um bloqueio é criado sobre agendamentos existentes THEN o sistema SHALL recusar o bloqueio e listar os agendamentos conflitantes.
- IF o barbeiro é inativado com agendamentos futuros THEN o sistema SHALL mantê-los e exibir aviso ao dono.
- WHEN um slot cruza a troca de dia no fuso da barbearia THEN o sistema SHALL contar o agendamento no dia de início.
- IF o serviço tem duração maior que o restante da jornada THEN o sistema SHALL não oferecer aquele slot.
- IF o cliente agenda no mesmo horário em barbearias diferentes THEN o sistema SHALL permitir (não há restrição cross-tenant).
- IF o slug requisitado em `/{slug}` não existe THEN o sistema SHALL responder 404 com a página "Barbearia não encontrada".

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| --- | --- | --- | --- |
| TEN-01 | P1: Cadastro | Tasks | Pending |
| TEN-02 | P1: Cadastro | Tasks | Pending |
| TEN-03 | P1: Cadastro | Tasks | Pending |
| TEN-04 | P1: Cadastro | Tasks | Pending |
| TEN-05 | P1: Cadastro | Tasks | Pending |
| TEN-06 | P1: Cadastro | Tasks | Pending |
| TEN-07 | P1: Cadastro | Tasks | Pending |
| TEN-08 | P1: Cadastro | Tasks | Pending |
| TEN-09 | P1: Cadastro | Tasks | Pending |
| SRV-01 | P1: Serviços | Tasks | Pending |
| SRV-02 | P1: Serviços | Tasks | Pending |
| SRV-03 | P1: Serviços | Tasks | Pending |
| SRV-04 | P1: Serviços | Tasks | Pending |
| SRV-05 | P1: Serviços | Tasks | Pending |
| SRV-06 | P1: Serviços | Tasks | Pending |
| SRV-07 | P1: Serviços | Tasks | Pending |
| AGD-01 | P1: Agendamento | Tasks | Pending |
| AGD-02 | P1: Agendamento | Tasks | Pending |
| AGD-03 | P1: Agendamento | Tasks | Pending |
| AGD-04 | P1: Agendamento | Tasks | Pending |
| AGD-05 | P1: Agendamento | Tasks | Pending |
| AGD-06 | P1: Agendamento | Tasks | Pending |
| AGD-07 | P1: Agendamento | Tasks | Pending |
| AGD-08 | P1: Agendamento | Tasks | Pending |
| AGD-09 | P1: Agendamento | Tasks | Pending |
| AGD-10 | P1: Agendamento | Tasks | Pending |
| AGD-11 | P1: Agendamento | Tasks | Pending |
| CAN-01 | P1: Cancelamento | Tasks | Pending |
| CAN-02 | P1: Cancelamento | Tasks | Pending |
| CAN-03 | P1: Cancelamento | Tasks | Pending |
| CAN-04 | P1: Cancelamento | Tasks | Pending |
| CAN-05 | P1: Cancelamento | Tasks | Pending |
| CAN-06 | P1: Cancelamento | Tasks | Pending |
| CAN-07 | P1: Cancelamento | Tasks | Pending |
| CAN-08 | P1: Cancelamento | Tasks | Pending |
| CAN-09 | P1: Cancelamento | Tasks | Pending |
| AUTH-01 | P1: Autenticação | Tasks | Pending |
| AUTH-02 | P1: Autenticação | Tasks | Pending |
| AUTH-03 | P1: Autenticação | Tasks | Pending |
| AUTH-04 | P1: Autenticação | Tasks | Pending |
| AUTH-05 | P1: Autenticação | Tasks | Pending |
| AUTH-06 | P1: Autenticação | Tasks | Pending |
| AUTH-07 | P1: Autenticação | Tasks | Pending |
| AUTH-08 | P1: Autenticação | Tasks | Pending |
| AUTH-09 | P1: Autenticação | Tasks | Pending |
| AUTH-10 | P1: Autenticação | Tasks | Pending |
| AUTH-11 | P1: Autenticação | Tasks | Pending |
| AUTH-12 | P1: Autenticação | Tasks | Pending |
| AUTH-13 | P1: Autenticação | Tasks | Pending |
| NOT-01 | P1: Notificações | Tasks | Pending |
| NOT-02 | P1: Notificações | Tasks | Pending |
| NOT-03 | P1: Notificações | Tasks | Pending |
| NOT-04 | P1: Notificações | Tasks | Pending |
| NOT-05 | P1: Notificações | Tasks | Pending |
| NOT-06 | P1: Notificações | Tasks | Pending |
| NOT-07 | P1: Notificações | Tasks | Pending |
| REL-01 | P1: Resumo mensal | Tasks | Pending |
| REL-02 | P1: Resumo mensal | Tasks | Pending |
| REL-03 | P1: Resumo mensal | Tasks | Pending |
| REL-04 | P1: Resumo mensal | Tasks | Pending |
| REL-05 | P1: Resumo mensal | Tasks | Pending |
| LGPD-01 | P1: Privacidade | Tasks | Pending |
| LGPD-02 | P1: Privacidade | Tasks | Pending |
| LGPD-03 | P1: Privacidade | Tasks | Pending |
| LGPD-04 | P1: Privacidade | Tasks | Pending |
| LGPD-05 | P1: Privacidade | Tasks | Pending |
| LGPD-06 | P1: Privacidade | Tasks | Pending |
| NFR-01 | P1: Operação | Tasks | Pending |
| NFR-02 | P1: Operação | Tasks | Pending |
| NFR-03 | P1: Operação | Tasks | Pending |
| NFR-04 | P1: Operação | Tasks | Pending |
| NFR-05 | P1: Operação | Tasks | Pending |
| PAG-01 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-02 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-03 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-04 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-05 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-06 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-07 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-08 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-09 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-10 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-11 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-12 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-13 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-14 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-15 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-16 | P3: Pagamento online | - | Deferred (fase 2) |
| PAG-17 | P3: Pagamento online | - | Deferred (fase 2) |
| PLN-01 | P3: Planos | - | Deferred (fase 2) |
| PLN-02 | P3: Planos | - | Deferred (fase 2) |
| PLN-03 | P3: Planos | - | Deferred (fase 2) |
| PLN-04 | P3: Planos | - | Deferred (fase 2) |
| PLN-05 | P3: Planos | - | Deferred (fase 2) |
| PLN-06 | P3: Planos | - | Deferred (fase 2) |
| PLN-07 | P3: Planos | - | Deferred (fase 2) |
| PLN-08 | P3: Planos | - | Deferred (fase 2) |
| PLN-09 | P3: Planos | - | Deferred (fase 2) |
| PLN-10 | P3: Planos | - | Deferred (fase 2) |
| PLN-11 | P3: Planos | - | Deferred (fase 2) |
| WPP-01 | P3: WhatsApp | - | Deferred (fase 2) |
| WPP-02 | P3: WhatsApp | - | Deferred (fase 2) |
| WPP-03 | P3: WhatsApp | - | Deferred (fase 2) |
| WPP-04 | P3: WhatsApp | - | Deferred (fase 2) |
| APP-01 | P3: App mobile | - | Deferred (fase 2) |
| APP-02 | P3: App mobile | - | Deferred (fase 2) |
| APP-03 | P3: App mobile | - | Deferred (fase 2) |
| APP-04 | P3: App mobile | - | Deferred (fase 2) |
| RPT-01 | P3: Relatórios completos | - | Deferred (fase 2) |
| RPT-02 | P3: Relatórios completos | - | Deferred (fase 2) |
| RPT-03 | P3: Relatórios completos | - | Deferred (fase 2) |
| RPT-04 | P3: Relatórios completos | - | Deferred (fase 2) |
| RPT-05 | P3: Relatórios completos | - | Deferred (fase 2) |

**ID format:** `[CATEGORIA]-[NN]`, onde NN é o número do critério dentro da story (ex.: `AGD-05` = critério 5 de Agendamento). Os IDs foram renumerados na revisão de 2026-11-26; nenhuma task tinha sido executada.

**Coverage:** 72 P1 total, 72 mapped to tasks (ver `## Checks` em tasks.md), 0 unmapped. 41 critérios P3 adiados.

---

## Success Criteria

- [ ] Fluxo ponta a ponta (cadastro → agendamento → e-mail de confirmação → lembrete → conclusão → resumo do mês) demonstrável na web, no celular e no desktop.
- [ ] Cenário k6 de contenção: 0 duplicados.
- [ ] Teste de isolamento entre tenants: 0 vazamentos.
- [ ] Restauração do backup testada com sucesso antes do lançamento.
