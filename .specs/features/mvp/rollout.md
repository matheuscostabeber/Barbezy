# MVP Rollout

**Spec**: `.specs/features/mvp/spec.md` · **Design**: `.specs/features/mvp/design.md`
**Revisado**: 2026-11-26 (VPS Hostinger, local + produção, MVP reduzido)

## Ambientes

| Ambiente | Site | API/Worker | Banco | E-mail | Login Google |
| --- | --- | --- | --- | --- | --- |
| local | `pnpm dev` (Vite) | `make dev` + Docker Compose | Postgres no Compose | Mailpit | app OAuth de teste |
| produção | Caddy na VPS (`frontend/dist`) | VPS Hostinger (Compose) | Postgres na VPS | Resend (domínio próprio) | app OAuth de produção |

Sem homologação no MVP. Os testes automáticos no CI são o "ensaio" antes de publicar. Barbearias de teste vivem em produção, marcadas como teste.

## Feature flags (por barbearia, coluna JSON em `barbearia`)

| Flag | Padrão | Liga quando |
| --- | --- | --- |
| `pagina_publica` (global) | on | Chave de emergência: desligada, todas as páginas públicas mostram "Agendamento online temporariamente indisponível" |

As flags de pagamento, Pix, WhatsApp e app voltam com a fase 2.

## Fases de lançamento

### Fase A — Interno
- 3 barbearias fictícias criadas pelo usuário; ciclo completo (cadastro → agendamento → lembrete → conclusão → resumo).
- Critério de saída: todos os checks de `tasks.md` com prova verde; Verifier PASS; restauração do backup testada.

### Fase B — Alfa fechado
- 3 a 5 barbearias reais conhecidas, pagamento no local.
- Site instalável (PWA) divulgado pelo link de cada barbearia.
- Critério de saída: 2 semanas, ≥ 100 agendamentos reais, 0 duplicados, 0 vazamentos entre tenants, feedback coletado.

### Fase C — Público
- Cadastro aberto.
- Planejar a fase 2 (pagamento online, planos, app) a partir do feedback do alfa.

## Gatilhos de revisão da infra (AD-005)

| ID | Gatilho (métrica) | Ação |
| --- | --- | --- |
| G1 | CPU da VPS > 70% por 1 h, ou memória > 80% | Plano maior na Hostinger |
| G2 | Disco > 70% | Plano com mais disco ou limpeza de backups locais |
| G3 | Indisponibilidade mensal > 0,5% | Avaliar banco gerenciado e segunda VPS antes de qualquer feature nova |
| G4 | Primeira barbearia pagante (fase 2) | Criar ambiente de homologação |
| G5 | > 30 barbearias ativas | Revisar volume de e-mail no Resend e avaliar WhatsApp (fase 2) |

## Checklist de go-live (Fase B)

- [ ] Domínio com SPF, DKIM e DMARC validados no Resend
- [ ] Segredos de produção em `.env` na VPS com permissão 600, fora do repo; segredos de deploy no GitHub
- [ ] Backup diário chegando ao B2/R2 e restauração testada na semana (T63)
- [ ] Alerta de uptime externo (ex.: monitor gratuito) apontando para `/api/health`
- [ ] Sentry recebendo erros da API e do worker
- [ ] Termos de Uso e Política de Privacidade publicados
- [ ] `/security-review` sem achados altos em aberto (T65)

## Backup e restauração

- **Diário:** `infra/scripts/backup.sh` (cron no compose) envia `pg_dump` comprimido para o bucket; retenção de 30 diários + 12 mensais.
- **Mensal:** rodar `infra/scripts/restore-check.sh` e anotar a data e o resultado aqui.

| Data | Backup usado | Resultado |
| --- | --- | --- |
| — | — | — |

## Rollback

- **API e worker:** `docker compose` com a tag de imagem anterior (GHCR mantém as últimas 10). Migrations sempre aditivas (expand/contract): nenhum deploy remove coluna em uso.
- **Site:** a imagem do Caddy carrega o `frontend/dist` da versão; voltar a tag volta o site junto.
- **Banco:** restauração do último backup (perda máxima de 24 h, RPO do D14) só em desastre.
- **Emergência:** desligar `pagina_publica` interrompe novos agendamentos sem derrubar o painel.
