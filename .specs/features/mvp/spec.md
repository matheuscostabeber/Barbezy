# MVP Specification — Barber App (SaaS multi-tenant de agendamento)

**Escopo:** Complex · **Contexto:** `.specs/features/mvp/context.md` (decisões D1–D15) · **Data:** 2026-11-22

## Problem Statement

Barbearias pequenas e médias agendam pelo WhatsApp e caderno: perdem horários por conflito, sofrem com no-show sem proteção financeira e calculam comissão de barbeiro à mão. Clientes não conseguem ver horários livres nem agendar fora do horário comercial. Queremos um SaaS multi-tenant em que cada barbearia tenha página pública de agendamento, agenda calculada, pagamento online opcional via Stripe Connect e relatórios, com plano gratuito para validar e assinatura para escalar.

## Goals

- [ ] Barbearia sai do cadastro para a página pública aceitando agendamentos em menos de 15 minutos.
- [ ] Cliente conclui um agendamento (sem pagamento online) em menos de 60 segundos a partir do link da barbearia.
- [ ] Zero agendamentos duplicados no mesmo horário sob 20 disputas simultâneas.
- [ ] Taxa de no-show mensurável por barbearia desde o primeiro mês.
- [ ] Infra de produção com custo mensal R$ 0 (excluindo custos por uso: WhatsApp, Stripe, lojas).

## Out of Scope

| Feature | Reason |
| --- | --- |
| Marketplace / busca de barbearias | Exige busca geográfica, avaliações e moderação; fase 2 (D1) |
| Sinal (depósito) | Reaproveita fluxo integral; fase 2 (D2) |
| Comissão da plataforma por transação | Monetização é assinatura (D3) |
| Subdomínio / domínio próprio | Fase 2, plano Plus (D1) |
| Produtos, pacotes, cupons, gorjeta | Fora do MVP (D11) |
| Split automático para o barbeiro | Exige conta Stripe por barbeiro (D12) |
| NFS-e | Responsabilidade da barbearia; integração é projeto à parte (D10) |
| Marketing por WhatsApp/e-mail | Exige consentimento e templates de marketing (D8) |
| Pagamento online obrigatório para todos os clientes | Cliente final escolhe (D2 revisado); só o cliente com 2 faltas é obrigado |
| Venda de planos dentro do app mobile | Evita IAP da Apple; planos só na web (D9) |
| Telas de dono no app (config, equipe, plano, Stripe, relatórios) | Só na web (D9) |
| Múltiplos serviços por agendamento | Combos são serviços próprios (D5) |
| SMS | Mesmo custo que WhatsApp com entrega pior (D7) |
| App white-label por barbearia | Plano enterprise futuro |

---

## Assumptions & Open Questions

| Assumption / decision | Chosen default | Rationale | Confirmed? |
| --- | --- | --- | --- |
| Fuso horário | Cada barbearia tem fuso IANA (padrão `America/Sao_Paulo`); horários armazenados em UTC e exibidos no fuso da barbearia | Brasil tem 4 fusos; UTC evita bugs de horário de verão | n |
| Idioma e moeda | pt-BR e BRL apenas | Mercado inicial é Brasil | n |
| Múltiplas unidades | Cada unidade é uma barbearia (tenant) separada; dono alterna por seletor | Reaproveita o modelo multi-tenant sem novo conceito | n |
| Onboarding da barbearia | Dono cria conta → 2FA → nome, slug, endereço, fuso, logo → primeiro serviço → jornada própria; começa no Free | Caminho mais curto até a página pública | n |
| Slug | 3–40 chars `[a-z0-9-]`, único global, lista de reservados (`painel`, `api`, `admin`, `login`, …); alterável 1x a cada 30 dias com redirect 301 do antigo | Links já divulgados no Instagram não podem quebrar | n |
| Escopo do admin da plataforma | Listar/buscar barbearias, ver plano e status, suspender/reativar por abuso, ver audit log; sem dados de pagamento nem de clientes além de contagens | Princípio do menor privilégio (D7) | n |
| Pix | Cartão no MVP. Pix fica atrás de feature flag por barbearia e só liga quando o Stripe liberar Pix para a conta conectada (no Brasil Pix é por convite no Stripe) | Docs do Stripe: Pix é invite-only para empresas no Brasil e exige capability na conta conectada | n |
| Provedor WhatsApp | WhatsApp Cloud API (Meta) direto, sem BSP intermediário | Sem mensalidade de BSP; custo só por mensagem (alinha com infra 0800) | n |
| Sign in with Apple | Oferecido no app iOS sempre que Google for oferecido | Guideline 4.8 da App Store | n |
| Fim de semana / feriado | Feriados são bloqueios avulsos criados pelo dono; sem calendário de feriados automático | Feriados municipais variam; evita dependência externa | n |
| Contagem do limite Free | Conta agendamentos criados no mês civil (fuso da barbearia) em qualquer status exceto `EXPIRADO` | Expirado nunca ocupou o serviço | n |
| Moeda mínima | Preço mínimo de serviço R$ 5,00 quando pagamento online estiver habilitado | Valor mínimo de cobrança do Stripe em BRL é baixo, mas taxa fixa inviabiliza abaixo disso | n |
| Infra | Vercel Hobby + VM Oracle Always Free + Neon Free (D15) | Pedido do Mauricio: custo zero | y |
| Quem decide pagar online | Barbearia escolhe entre "Só no local" e "Cliente escolhe"; no segundo, o cliente final escolhe "Pagar agora" ou "Pagar no local" | Decisão do Mauricio em 2026-11-22 | y |
| Opção pré-selecionada | "Pagar agora" vem marcada em barbearia "Cliente escolhe" | Incentiva o pagamento antecipado sem obrigar | n |
| Falta recorrente em barbearia sem Stripe | Em "Só no local" a regra das 2 faltas só registra (não há como exigir pagamento) | Não existe meio online para exigir | n |

**Open questions:** none - all resolved or logged above.

---

## User Stories

### P1: Cadastro e configuração da barbearia ⭐ MVP

**User Story**: As a dono, I want cadastrar minha barbearia com equipe, serviços e jornadas so that minha página pública fique pronta para receber agendamentos.

**Why P1**: Sem tenant configurado não existe produto.

**Acceptance Criteria**:

1. WHEN o dono conclui o onboarding THEN o sistema SHALL criar a barbearia no plano Free com página pública ativa em `/{slug}`.
2. IF o slug informado já existe, é reservado ou não casa `^[a-z0-9-]{3,40}$` THEN o sistema SHALL rejeitar com a mensagem "Endereço indisponível" sem criar a barbearia.
3. WHEN o dono altera o slug THEN o sistema SHALL responder 301 do slug antigo para o novo por 12 meses.
4. IF o dono tenta alterar o slug menos de 30 dias após a última alteração THEN o sistema SHALL rejeitar a alteração.
5. WHEN o dono envia convite a um e-mail THEN o sistema SHALL enviar link de convite válido por 7 dias.
6. IF um convite expirado ou já usado é aberto THEN o sistema SHALL exibir "Convite inválido" e não criar vínculo.
7. WHILE a barbearia está no plano Free o sistema SHALL impedir mais de 1 barbeiro ativo.
8. WHILE a barbearia está no plano Pro o sistema SHALL impedir mais de 5 barbeiros ativos.
9. WHEN o dono salva a jornada semanal de um barbeiro com intervalos THEN o sistema SHALL rejeitar intervalos sobrepostos ou fora da jornada.
10. WHEN o dono personaliza logo, capa e cor de destaque THEN a página pública SHALL exibi-los no próximo carregamento.
11. WHILE a barbearia está no plano Free a página pública SHALL exibir o selo da plataforma.

**Independent Test**: Criar barbearia, convidar barbeiro, cadastrar jornada e serviço, abrir `/{slug}` e ver o serviço listado.

---

### P1: Serviços e preços ⭐ MVP

**User Story**: As a dono, I want cadastrar serviços com preço e duração, inclusive por barbeiro, so that a agenda e o preço reflitam a realidade da casa.

**Why P1**: Slots e pagamento dependem de duração e preço.

**Acceptance Criteria**:

1. WHEN o dono cria um serviço THEN o sistema SHALL exigir nome, duração múltipla de 15 min entre 15 e 240 min, preço ≥ R$ 0 e categoria em {Cabelo, Barba, Combo, Outros}.
2. WHEN o dono define override de preço ou duração para um barbeiro THEN o sistema SHALL usar o override para agendamentos com aquele barbeiro.
3. WHEN o dono desabilita um barbeiro para um serviço THEN o sistema SHALL excluir aquele barbeiro dos slots daquele serviço.
4. WHEN um agendamento é criado THEN o sistema SHALL gravar preço e duração vigentes no agendamento (preço congelado).
5. WHEN o dono altera o preço de um serviço THEN o sistema SHALL manter o preço dos agendamentos já criados.
6. WHEN o dono desativa um serviço THEN o sistema SHALL removê-lo da página pública e manter seus agendamentos futuros.
7. IF o dono tenta excluir um serviço com agendamentos THEN o sistema SHALL recusar e oferecer desativação.
8. WHERE pagamento online está habilitado o sistema SHALL rejeitar preço de serviço menor que R$ 5,00.

**Independent Test**: Criar serviço com override para um barbeiro, agendar com ele, mudar o preço base e conferir que o agendamento manteve o valor.

---

### P1: Agendamento pelo cliente ⭐ MVP

**User Story**: As a cliente, I want escolher serviço, barbeiro e horário livre so that eu agende sem trocar mensagens.

**Why P1**: É o núcleo do produto.

**Acceptance Criteria**:

1. The system SHALL calcular slots a partir da jornada, intervalos e bloqueios do barbeiro menos agendamentos ocupantes, em grade de 15 minutos, no fuso da barbearia.
2. The system SHALL oferecer apenas slots com início ≥ agora + antecedência mínima (padrão 2 h) e ≤ hoje + antecedência máxima (padrão 30 dias).
3. WHEN o cliente escolhe "qualquer um disponível" THEN o sistema SHALL atribuir o barbeiro elegível com menos agendamentos ocupantes naquele dia, desempatando pelo id do barbeiro.
4. WHEN o cliente escolhe "qualquer um disponível" THEN o sistema SHALL exibir a faixa de preço (mín–máx entre barbeiros elegíveis) até a atribuição e o preço final antes da confirmação.
5. The system SHALL garantir no banco que dois agendamentos ocupantes do mesmo barbeiro nunca se sobreponham no tempo.
6. IF a criação falha por conflito de horário THEN o sistema SHALL responder 409 com a mensagem "Esse horário acabou de ser reservado" e até 3 slots livres mais próximos.
7. WHEN o cliente confirma com a forma "Pagar no local" THEN o sistema SHALL criar o agendamento em `CONFIRMADO`.
8. WHEN o cliente confirma com a forma "Pagar agora" THEN o sistema SHALL criar o agendamento em `PENDENTE_PAGAMENTO` com expiração em 10 minutos.
9. WHILE o agendamento está em `PENDENTE_PAGAMENTO` o sistema SHALL tratá-lo como ocupante do horário.
10. WHEN a expiração de um `PENDENTE_PAGAMENTO` é atingida sem pagamento confirmado THEN o sistema SHALL movê-lo para `EXPIRADO` e liberar o horário.
11. IF o cliente não está autenticado ao confirmar THEN o sistema SHALL exigir login antes de criar o agendamento, preservando a seleção.
12. WHILE a barbearia Free atingiu 20 agendamentos no mês o sistema SHALL ocultar os slots do mês e exibir "Agenda indisponível online".
13. The system SHALL permitir apenas as transições `PENDENTE_PAGAMENTO→CONFIRMADO|EXPIRADO`, `CONFIRMADO→CONCLUIDO|CANCELADO_CLIENTE|CANCELADO_BARBEARIA|NO_SHOW`.
14. IF uma transição fora da lista é solicitada THEN o sistema SHALL responder 409 e não alterar o agendamento.
15. WHERE a barbearia está no modo "Cliente escolhe" the system SHALL exibir as formas "Pagar agora" (pré-selecionada) e "Pagar no local" antes da confirmação.
16. WHILE a barbearia está no modo "Só no local" the system SHALL criar agendamentos apenas com a forma "Pagar no local" e não exibir opção de pagamento online.

**Independent Test**: Duas abas agendando o mesmo slot ao mesmo tempo → uma confirma, a outra recebe 409 com sugestões.

---

### P1: Cancelamento, remarcação e no-show ⭐ MVP

**User Story**: As a cliente ou barbearia, I want cancelar e remarcar com regras claras so that ninguém seja surpreendido e a barbearia se proteja de no-show.

**Why P1**: Sem política, o pagamento online não protege ninguém.

**Acceptance Criteria**:

1. WHEN o cliente cancela um agendamento pago online com antecedência ≥ janela (padrão 4 h) THEN o sistema SHALL mover para `CANCELADO_CLIENTE` e emitir reembolso integral.
2. WHEN o cliente cancela um agendamento pago online com antecedência < janela THEN o sistema SHALL mover para `CANCELADO_CLIENTE` sem reembolso.
3. WHEN o cliente cancela um agendamento "pagar no local" THEN o sistema SHALL mover para `CANCELADO_CLIENTE` sem cobrança.
4. WHEN a barbearia cancela um agendamento THEN o sistema SHALL mover para `CANCELADO_BARBEARIA` e emitir reembolso integral se houver pagamento online.
5. WHEN o cliente remarca com antecedência ≥ janela THEN o sistema SHALL mover o agendamento para o novo slot mantendo o pagamento existente.
6. IF o cliente tenta remarcar com antecedência < janela THEN o sistema SHALL recusar a remarcação.
7. WHEN o barbeiro marca no-show até 24 h após o início THEN o sistema SHALL mover para `NO_SHOW` e manter o valor pago com a barbearia.
8. IF o barbeiro tenta marcar no-show antes do início ou mais de 24 h após o início THEN o sistema SHALL recusar.
9. WHEN um cliente com 2 `NO_SHOW` nos últimos 90 dias agenda numa barbearia no modo "Cliente escolhe" THEN o sistema SHALL oferecer apenas a forma "Pagar agora".
10. WHEN o cliente abre o link de cancelamento do e-mail/WhatsApp antes do início do atendimento THEN o sistema SHALL permitir cancelar sem login.
11. IF o link de cancelamento é aberto após o início do atendimento ou tem assinatura inválida THEN o sistema SHALL exibir "Link expirado".

**Independent Test**: Agendar com pagamento online, cancelar 5 h antes → reembolso; agendar de novo, cancelar 1 h antes → sem reembolso.

---

### P1: Autenticação e papéis ⭐ MVP

**User Story**: As a usuário, I want entrar de forma rápida e segura conforme meu papel so that eu acesse só o que me cabe.

**Why P1**: Multi-tenant sem isolamento é vazamento de dados.

**Acceptance Criteria**:

1. WHEN o cliente solicita código por WhatsApp ou e-mail THEN o sistema SHALL enviar OTP de 6 dígitos válido por 10 minutos.
2. IF o envio por WhatsApp falha THEN o sistema SHALL oferecer "Receber por e-mail" na mesma tela.
3. IF um identificador ou IP excede 5 tentativas de login/OTP em 15 minutos THEN o sistema SHALL responder 429 até o fim da janela.
4. WHEN o OTP é usado com sucesso THEN o sistema SHALL invalidá-lo para qualquer uso posterior.
5. WHEN um dono faz login sem 2FA configurado THEN o sistema SHALL exigir configuração de TOTP antes de acessar o painel.
6. WHILE o usuário tem papel de dono o sistema SHALL exigir TOTP válido em todo login.
7. WHEN o login conclui THEN o sistema SHALL emitir access token de 15 minutos e refresh token rotativo de 30 dias.
8. IF um refresh token já rotacionado é reapresentado THEN o sistema SHALL revogar toda a família de tokens daquela sessão.
9. WHEN o usuário revoga uma sessão no painel THEN o sistema SHALL rejeitar o refresh token dela com 401.
10. The system SHALL filtrar toda leitura e escrita de dados de tenant pela barbearia do contexto, com RLS no Postgres como segunda barreira.
11. IF um usuário acessa recurso de outra barbearia THEN o sistema SHALL responder 404.
12. WHILE o usuário é barbeiro o sistema SHALL negar acesso a faturamento da casa, dados de outros barbeiros, configurações, plano e Stripe.
13. WHERE o app iOS oferece login com Google o app SHALL oferecer Sign in with Apple.
14. The admin da plataforma SHALL ter acesso negado a dados de pagamento e a dados pessoais de clientes.

**Independent Test**: Dono da barbearia A tenta GET de agendamento da barbearia B → 404; consulta direta no banco com o role da aplicação e `barbearia_id` de A não retorna linhas de B.

---

### P1: Pagamento online do cliente (Stripe Connect) ⭐ MVP

**User Story**: As a dono, I want receber online pelos agendamentos so that eu reduza no-show e receba direto na minha conta.

**Why P1**: Diferencial do produto e parte central do fluxo com gateway.

**Acceptance Criteria**:

1. WHEN o dono clica "Conectar Stripe" THEN o sistema SHALL criar conta Connect Express e redirecionar ao onboarding hospedado pelo Stripe.
2. WHILE a conta conectada não tem `charges_enabled` o sistema SHALL manter a barbearia no modo "Só no local" e o modo "Cliente escolhe" desabilitado.
3. WHEN um agendamento entra em `PENDENTE_PAGAMENTO` THEN o sistema SHALL criar sessão de pagamento no valor do preço congelado, com recebimento pela conta conectada e expiração alinhada ao hold.
4. WHEN o webhook de pagamento bem-sucedido chega para um agendamento `PENDENTE_PAGAMENTO` THEN o sistema SHALL movê-lo para `CONFIRMADO`.
5. IF o webhook de pagamento bem-sucedido chega para um agendamento `EXPIRADO` THEN o sistema SHALL emitir reembolso integral e notificar o cliente.
6. The system SHALL processar cada evento de webhook no máximo uma vez, identificado pelo id do evento.
7. IF a assinatura do webhook é inválida THEN o sistema SHALL responder 400 e não processar o evento.
8. The system SHALL nunca receber, armazenar ou registrar em log dados de cartão.
9. WHEN um reembolso é emitido THEN o sistema SHALL registrar o pagamento como `REEMBOLSADO` ao receber a confirmação do Stripe.
10. WHEN o Stripe notifica uma contestação THEN o sistema SHALL exibir alerta no painel do dono com link para o Stripe.
11. WHERE a flag Pix está ligada para a barbearia e a conta conectada tem a capability Pix ativa o sistema SHALL oferecer Pix como meio de pagamento.

**Independent Test**: Em modo teste do Stripe, agendar com cartão de teste → webhook confirma → agendamento `CONFIRMADO`; reenviar o mesmo evento → nenhum efeito.

---

### P1: Planos, freemium e downgrade ⭐ MVP

**User Story**: As a plataforma, I want cobrar assinatura com plano gratuito limitado so that barbearias validem de graça e paguem para crescer.

**Why P1**: É a receita da plataforma.

**Acceptance Criteria**:

1. WHEN o dono assina Pro ou Plus na web THEN o sistema SHALL criar assinatura no Stripe Billing e aplicar o plano ao receber o webhook de assinatura ativa.
2. WHEN a contagem mensal de uma barbearia Free atinge 16 THEN o sistema SHALL notificar o dono (80%).
3. WHEN a contagem mensal de uma barbearia Free atinge 20 THEN o sistema SHALL notificar o dono (100%) e bloquear novos agendamentos online no mês.
4. IF a barbearia Free já tem 20 agendamentos no mês THEN o sistema SHALL recusar a criação de novos agendamentos com 403 e manter os existentes.
5. WHEN uma cobrança de assinatura falha THEN o sistema SHALL marcar a barbearia em carência por 7 dias e exibir banner ao dono.
6. WHEN a carência termina sem pagamento THEN o sistema SHALL rebaixar a barbearia para Free.
7. WHEN uma barbearia é rebaixada com mais barbeiros ativos que o limite THEN o sistema SHALL pedir ao dono que escolha quais permanecem ativos.
8. IF o dono não escolhe em 48 horas THEN o sistema SHALL manter ativo o barbeiro com mais agendamentos nos últimos 30 dias e inativar os demais.
9. WHEN um barbeiro é inativado THEN o sistema SHALL mantê-lo sem novos slots e preservar seus agendamentos futuros já confirmados.
10. WHEN a barbearia volta a um plano pago THEN o sistema SHALL permitir reativar barbeiros inativos com uma ação.
11. The app mobile SHALL não oferecer compra ou troca de plano.

**Independent Test**: Barbearia Pro com 3 barbeiros; simular falha de cobrança e avançar 7 dias → Free; após 48 h sem escolha, só o barbeiro mais ocupado fica ativo.

---

### P1: Notificações ⭐ MVP

**User Story**: As a cliente, I want ser avisado e lembrado do meu horário so that eu não esqueça nem falte.

**Why P1**: Lembrete é a maior alavanca contra no-show.

**Acceptance Criteria**:

1. WHEN um agendamento vai para `CONFIRMADO` THEN o sistema SHALL enviar confirmação pelo canal preferido do cliente e por e-mail.
2. WHEN faltam 24 horas e 2 horas para um agendamento `CONFIRMADO` THEN o sistema SHALL enviar lembrete por WhatsApp.
3. WHEN um agendamento é cancelado ou remarcado THEN o sistema SHALL notificar cliente (canal preferido + e-mail) e barbeiro (painel + e-mail).
4. IF o cliente respondeu "SAIR" no WhatsApp THEN o sistema SHALL suspender lembretes e manter confirmações e cancelamentos.
5. IF o envio de uma notificação falha THEN o sistema SHALL tentar novamente até 3 vezes com backoff e registrar a falha sem alterar o agendamento.
6. WHEN um agendamento é criado, cancelado ou alterado THEN o painel do barbeiro e do dono SHALL refletir a mudança sem recarregar a página em até 5 segundos.
7. WHERE o app está instalado com permissão de push o sistema SHALL enviar push ao barbeiro para novo agendamento e cancelamento.
8. The system SHALL não enviar mensagens de marketing.

**Independent Test**: Criar agendamento para daqui a 25 h e avançar o relógio da fila → lembretes de 24 h e 2 h disparam uma vez cada.

---

### P1: Relatórios e comissão ⭐ MVP

**User Story**: As a dono, I want ver faturamento, no-show, ocupação e comissão so that eu gerencie a casa e pague a equipe sem planilha.

**Why P1**: Dor real do dono e argumento de venda do plano pago.

**Acceptance Criteria**:

1. WHEN o dono abre relatórios para um período THEN o sistema SHALL exibir faturamento separado em online e no local, contando apenas agendamentos `CONCLUIDO` e `NO_SHOW` com pagamento retido.
2. The system SHALL calcular taxa de no-show como `NO_SHOW / (CONCLUIDO + NO_SHOW)` no período.
3. The system SHALL calcular ocupação por barbeiro como minutos ocupados por agendamentos `CONFIRMADO`/`CONCLUIDO` sobre minutos disponíveis da jornada no período.
4. The system SHALL calcular comissão do barbeiro como percentual configurado × soma do preço congelado dos seus agendamentos `CONCLUIDO` no período.
5. WHEN o barbeiro marca `CONCLUIDO` um agendamento "pagar no local" THEN o sistema SHALL exigir a forma de pagamento em {dinheiro, pix, cartão}.
6. WHILE a barbearia está no plano Free o sistema SHALL restringir relatórios ao mês corrente.
7. WHERE a barbearia está no plano Plus o sistema SHALL permitir exportar o relatório em CSV.
8. WHILE o usuário é barbeiro o sistema SHALL exibir apenas seus próprios atendimentos e comissão.

**Independent Test**: Seed com 10 concluídos (R$ 50) e 2 no-shows para um barbeiro com 40% → faturamento e comissão R$ 200,00 batem; taxa de no-show 16,7%.

---

### P1: Privacidade e LGPD ⭐ MVP

**User Story**: As a cliente, I want controlar meus dados so that eu confie na plataforma.

**Why P1**: Exigência legal e da Apple.

**Acceptance Criteria**:

1. WHEN o cliente solicita exclusão de conta e confirma por OTP THEN o sistema SHALL cancelar agendamentos futuros conforme a política e anonimizar os dados pessoais em até 15 dias.
2. The system SHALL manter agendamentos passados de contas excluídas com cliente exibido como "Cliente removido".
3. WHEN o cliente solicita exportação THEN o sistema SHALL enviar por e-mail, em até 24 horas, um JSON com conta e agendamentos.
4. WHEN uma barbearia encerra a conta THEN o sistema SHALL desativar a página pública imediatamente e anonimizar dados não financeiros após 90 dias.
5. The system SHALL reter registros financeiros por 5 anos e logs de acesso por 6 meses.
6. The system SHALL registrar em audit log mudanças de preço, plano, papel, reembolsos, exportações e exclusões, com autor e data.
7. The system SHALL não coletar CPF, data de nascimento ou endereço do cliente.
8. The app mobile SHALL oferecer exclusão de conta sem sair do app.

**Independent Test**: Cliente com 1 agendamento passado e 1 futuro exclui a conta → futuro cancelado, passado aparece como "Cliente removido", e-mail/telefone sem PII após o job de anonimização.

---

### P1: Escalabilidade e operação ⭐ MVP

**User Story**: As a plataforma, I want aguentar picos sem duplicar agendamentos so that crescer não quebre a confiança.

**Why P1**: Pedido explícito do projeto.

**Acceptance Criteria**:

1. WHEN 20 requisições simultâneas tentam o mesmo slot THEN o sistema SHALL confirmar exatamente 1 e responder 409 às demais.
2. WHILE o sistema recebe 2.000 usuários simultâneos no cenário k6 de referência a página pública e a consulta de slots SHALL manter p95 < 300 ms.
3. WHILE o sistema recebe 50 criações de agendamento por segundo no cenário k6 de referência a criação SHALL manter p95 < 500 ms excluindo chamadas ao Stripe.
4. WHEN um agendamento ou bloqueio altera a agenda de um barbeiro em um dia THEN o sistema SHALL invalidar o cache de slots daquele barbeiro e dia.
5. The API SHALL ser stateless, permitindo múltiplas instâncias atrás de um proxy.
6. The system SHALL propagar trace id do cliente (web/app) até a API, fila e webhook.
7. The system SHALL executar backup diário do banco com restauração testada a cada release.

**Independent Test**: Rodar o cenário k6 `contention.js` → 1 sucesso, 19 × 409, zero duplicados no banco.

---

## Edge Cases

- IF o relógio do cliente está errado THEN o sistema SHALL usar sempre o horário do servidor para antecedência e expiração.
- IF um bloqueio é criado sobre agendamentos existentes THEN o sistema SHALL recusar o bloqueio e listar os agendamentos conflitantes.
- IF o barbeiro é inativado com agendamentos futuros THEN o sistema SHALL mantê-los e exibir aviso ao dono.
- IF o Stripe está indisponível ao criar a sessão de pagamento THEN o sistema SHALL mover o agendamento para `EXPIRADO` imediatamente e exibir "Pagamento indisponível. Tente de novo ou escolha pagar no local" (a segunda opção só quando permitida ao cliente).
- IF o webhook chega antes da resposta do redirect THEN o sistema SHALL exibir o estado do banco na tela de retorno (polling até 30 s).
- WHEN um slot cruza a troca de dia no fuso da barbearia THEN o sistema SHALL contar o agendamento no dia de início.
- IF o serviço tem duração maior que o restante da jornada THEN o sistema SHALL não oferecer aquele slot.
- IF o cliente tenta agendar duas vezes no mesmo horário em barbearias diferentes THEN o sistema SHALL permitir (não há restrição cross-tenant).

---

## Requirement Traceability

| Requirement ID | Story | Phase | Status |
| --- | --- | --- | --- |
| TEN-01 | P1: Cadastro | Design | Pending |
| TEN-02 | P1: Cadastro | Design | Pending |
| TEN-03 | P1: Cadastro | Design | Pending |
| TEN-04 | P1: Cadastro | Design | Pending |
| TEN-05 | P1: Cadastro | Design | Pending |
| TEN-06 | P1: Cadastro | Design | Pending |
| TEN-07 | P1: Cadastro | Design | Pending |
| TEN-08 | P1: Cadastro | Design | Pending |
| TEN-09 | P1: Cadastro | Design | Pending |
| TEN-10 | P1: Cadastro | Design | Pending |
| TEN-11 | P1: Cadastro | Design | Pending |
| SRV-01 | P1: Serviços | Design | Pending |
| SRV-02 | P1: Serviços | Design | Pending |
| SRV-03 | P1: Serviços | Design | Pending |
| SRV-04 | P1: Serviços | Design | Pending |
| SRV-05 | P1: Serviços | Design | Pending |
| SRV-06 | P1: Serviços | Design | Pending |
| SRV-07 | P1: Serviços | Design | Pending |
| SRV-08 | P1: Serviços | Design | Pending |
| AGD-01 | P1: Agendamento | Design | Pending |
| AGD-02 | P1: Agendamento | Design | Pending |
| AGD-03 | P1: Agendamento | Design | Pending |
| AGD-04 | P1: Agendamento | Design | Pending |
| AGD-05 | P1: Agendamento | Design | Pending |
| AGD-06 | P1: Agendamento | Design | Pending |
| AGD-07 | P1: Agendamento | Design | Pending |
| AGD-08 | P1: Agendamento | Design | Pending |
| AGD-09 | P1: Agendamento | Design | Pending |
| AGD-10 | P1: Agendamento | Design | Pending |
| AGD-11 | P1: Agendamento | Design | Pending |
| AGD-12 | P1: Agendamento | Design | Pending |
| AGD-13 | P1: Agendamento | Design | Pending |
| AGD-14 | P1: Agendamento | Design | Pending |
| AGD-15 | P1: Agendamento | Design | Pending |
| AGD-16 | P1: Agendamento | Design | Pending |
| CAN-01 | P1: Cancelamento | Design | Pending |
| CAN-02 | P1: Cancelamento | Design | Pending |
| CAN-03 | P1: Cancelamento | Design | Pending |
| CAN-04 | P1: Cancelamento | Design | Pending |
| CAN-05 | P1: Cancelamento | Design | Pending |
| CAN-06 | P1: Cancelamento | Design | Pending |
| CAN-07 | P1: Cancelamento | Design | Pending |
| CAN-08 | P1: Cancelamento | Design | Pending |
| CAN-09 | P1: Cancelamento | Design | Pending |
| CAN-10 | P1: Cancelamento | Design | Pending |
| CAN-11 | P1: Cancelamento | Design | Pending |
| AUTH-01 | P1: Autenticação | Design | Pending |
| AUTH-02 | P1: Autenticação | Design | Pending |
| AUTH-03 | P1: Autenticação | Design | Pending |
| AUTH-04 | P1: Autenticação | Design | Pending |
| AUTH-05 | P1: Autenticação | Design | Pending |
| AUTH-06 | P1: Autenticação | Design | Pending |
| AUTH-07 | P1: Autenticação | Design | Pending |
| AUTH-08 | P1: Autenticação | Design | Pending |
| AUTH-09 | P1: Autenticação | Design | Pending |
| AUTH-10 | P1: Autenticação | Design | Pending |
| AUTH-11 | P1: Autenticação | Design | Pending |
| AUTH-12 | P1: Autenticação | Design | Pending |
| AUTH-13 | P1: Autenticação | Design | Pending |
| AUTH-14 | P1: Autenticação | Design | Pending |
| PAG-01 | P1: Pagamento | Design | Pending |
| PAG-02 | P1: Pagamento | Design | Pending |
| PAG-03 | P1: Pagamento | Design | Pending |
| PAG-04 | P1: Pagamento | Design | Pending |
| PAG-05 | P1: Pagamento | Design | Pending |
| PAG-06 | P1: Pagamento | Design | Pending |
| PAG-07 | P1: Pagamento | Design | Pending |
| PAG-08 | P1: Pagamento | Design | Pending |
| PAG-09 | P1: Pagamento | Design | Pending |
| PAG-10 | P1: Pagamento | Design | Pending |
| PAG-11 | P1: Pagamento | Design | Pending |
| PLN-01 | P1: Planos | Design | Pending |
| PLN-02 | P1: Planos | Design | Pending |
| PLN-03 | P1: Planos | Design | Pending |
| PLN-04 | P1: Planos | Design | Pending |
| PLN-05 | P1: Planos | Design | Pending |
| PLN-06 | P1: Planos | Design | Pending |
| PLN-07 | P1: Planos | Design | Pending |
| PLN-08 | P1: Planos | Design | Pending |
| PLN-09 | P1: Planos | Design | Pending |
| PLN-10 | P1: Planos | Design | Pending |
| PLN-11 | P1: Planos | Design | Pending |
| NOT-01 | P1: Notificações | Design | Pending |
| NOT-02 | P1: Notificações | Design | Pending |
| NOT-03 | P1: Notificações | Design | Pending |
| NOT-04 | P1: Notificações | Design | Pending |
| NOT-05 | P1: Notificações | Design | Pending |
| NOT-06 | P1: Notificações | Design | Pending |
| NOT-07 | P1: Notificações | Design | Pending |
| NOT-08 | P1: Notificações | Design | Pending |
| REL-01 | P1: Relatórios | Design | Pending |
| REL-02 | P1: Relatórios | Design | Pending |
| REL-03 | P1: Relatórios | Design | Pending |
| REL-04 | P1: Relatórios | Design | Pending |
| REL-05 | P1: Relatórios | Design | Pending |
| REL-06 | P1: Relatórios | Design | Pending |
| REL-07 | P1: Relatórios | Design | Pending |
| REL-08 | P1: Relatórios | Design | Pending |
| LGPD-01 | P1: Privacidade | Design | Pending |
| LGPD-02 | P1: Privacidade | Design | Pending |
| LGPD-03 | P1: Privacidade | Design | Pending |
| LGPD-04 | P1: Privacidade | Design | Pending |
| LGPD-05 | P1: Privacidade | Design | Pending |
| LGPD-06 | P1: Privacidade | Design | Pending |
| LGPD-07 | P1: Privacidade | Design | Pending |
| LGPD-08 | P1: Privacidade | Design | Pending |
| NFR-01 | P1: Escalabilidade | Design | Pending |
| NFR-02 | P1: Escalabilidade | Design | Pending |
| NFR-03 | P1: Escalabilidade | Design | Pending |
| NFR-04 | P1: Escalabilidade | Design | Pending |
| NFR-05 | P1: Escalabilidade | Design | Pending |
| NFR-06 | P1: Escalabilidade | Design | Pending |
| NFR-07 | P1: Escalabilidade | Design | Pending |

**ID format:** `[CATEGORIA]-[NN]`, onde NN é o número do critério dentro da story (ex.: `AGD-05` = critério 5 de Agendamento).

**Coverage:** 113 total, 0 mapped to tasks, 113 unmapped ⚠️ (mapeamento em tasks.md)

---

## Success Criteria

- [ ] Fluxo ponta a ponta (cadastro → agendamento com pagamento de teste → lembrete → conclusão → relatório) demonstrável em web e app.
- [ ] Cenário k6 de contenção: 0 duplicados.
- [ ] Teste de isolamento entre tenants: 0 vazamentos.
- [ ] Custo de infra de produção no primeiro mês: R$ 0.
