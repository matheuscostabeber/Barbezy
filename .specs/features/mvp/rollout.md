# MVP Rollout

**Spec**: `.specs/features/mvp/spec.md` · **Design**: `.specs/features/mvp/design.md`

## Ambientes

| Ambiente | Web | API/Worker | Banco | Stripe | WhatsApp |
| --- | --- | --- | --- | --- | --- |
| local | `next dev` | Docker Compose | Postgres no Compose | stripe-mock + stripe-cli | número de teste da Meta |
| preview (por PR) | Vercel Preview | Compose local do dev | Neon branch do PR | test mode | número de teste |
| produção | Vercel Hobby | VM Oracle (Compose) | Neon main | live mode | número verificado |

Um único ambiente remoto além dos previews: sem staging dedicado no free tier. O "staging" é o deploy de produção com feature flags desligadas e barbearias de teste.

## Feature flags (por barbearia, coluna/JSON em `barbearia`)

| Flag | Padrão | Liga quando |
| --- | --- | --- |
| `pagamento_online` (global) | on | Chave de emergência: desligada, todas as barbearias "Cliente escolhe" passam a oferecer só "Pagar no local" |
| `pix` | off | Stripe liberar Pix para a conta conectada (PAG-11) |
| `whatsapp_otp` | off global | Template `auth_code` aprovado (T3) |
| `whatsapp_lembretes` | off global | Templates utility aprovados (T3) |
| `app_mobile` | — | Controle pelas lojas (TestFlight / teste interno) |

## Fases de lançamento

### Fase A — Interno (fim das Phases 1–9)
- 3 barbearias fictícias criadas pelo time; ciclo completo em test mode do Stripe.
- Critério de saída: todos os AC da spec com teste verde; Verifier PASS (`validation.md`).

### Fase B — Alfa fechado (Phases 10–12 prontas)
- 3 a 5 barbearias reais conhecidas, plano Free, **pagamento no local** apenas.
- WhatsApp ligado se templates aprovados; senão só e-mail.
- App via TestFlight / teste interno do Google Play.
- Critério de saída: 2 semanas, ≥ 100 agendamentos reais, 0 duplicados, 0 incidentes de vazamento entre tenants, feedback coletado.

### Fase C — Beta com pagamento
- Barbearias do alfa que quiserem conectam o Stripe (live mode) e passam para "Cliente escolhe"; as demais seguem "Só no local".
- Abre assinatura Pro/Plus.
- Critério de saída: ≥ 20 pagamentos live sem divergência entre Stripe e banco (conciliação diária manual); 0 webhooks não processados.

### Fase D — Público
- Cadastro aberto; apps publicados nas lojas.
- Ao primeiro cliente pagante da plataforma: executar o gatilho G1 abaixo.

## Gatilhos de migração do free tier (AD-005)

| ID | Gatilho (métrica) | Ação |
| --- | --- | --- |
| G1 | Primeira assinatura paga ativa | Web sai do Vercel Hobby (não-comercial): Vercel Pro **ou** Next.js self-hosted na VM atrás do Caddy |
| G2 | Neon: uso > 80% de compute ou storage do free por 2 semanas, ou p95 de slots > 300 ms por cold start em produção | Neon Launch (pago) com autosuspend desligado |
| G3 | CPU da VM > 70% sustentado 1 h, ou memória > 80% | Segunda VM/instância paga + Redis gerenciado; API em 2 hosts |
| G4 | Indisponibilidade mensal > 0,5% | Tirar ponto único de falha (G3) antes de qualquer feature nova |
| G5 | > 30 barbearias ativas | Revisar custo de WhatsApp vs. e-mail; renegociar ou ajustar canal padrão |

## Checklist de go-live (Fase C/D)

- [ ] Webhooks Stripe (Connect + Billing) apontando para produção com segredo próprio
- [ ] Chaves live no cofre da VM (arquivo `.env` com permissão 600, fora do repo); nada em Vercel além do necessário ao BFF
- [ ] Backup do Neon verificado por T88 nesta semana
- [ ] Alertas Grafana: 5xx > 1%, fila `notificacoes` com dead-letter > 0, webhook sem processar > 5 min, VM down
- [ ] Termos de Uso, Política de Privacidade e contrato de operador LGPD publicados
- [ ] Página de status simples (fora da VM) para comunicar incidentes

## Rollback

- **API**: `docker compose` com tag de imagem anterior (GHCR mantém as últimas 10). Migrations são sempre aditivas (expand/contract): nenhum deploy remove coluna em uso.
- **Web**: "Promote" do deploy anterior na Vercel.
- **App**: OTA update do Expo para JS; binário só via nova submissão.
- **Pagamentos**: desligar `pagamento_online` globalmente esconde "Pagar agora" (inclusive para clientes com 2 faltas, que voltam a poder pagar no local enquanto a chave estiver desligada); holds existentes expiram normalmente.
