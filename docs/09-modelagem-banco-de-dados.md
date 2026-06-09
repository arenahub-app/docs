# Modelagem de Banco de Dados — ArenaHub

## Decisões de Modelagem

| Decisão | Escolha | Justificativa |
|---|---|---|
| SGBD | PostgreSQL 16 | Suporte a UUID nativo, JSONB, RLS, GIN indexes, geração de UUID com `gen_random_uuid()` |
| Migrations | Flyway | Versionamento imutável, integração Spring Boot, rollback controlado |
| Multi-tenancy | Shared schema + `group_id` | Ver ADR-002 e ADR-008 |
| UUIDs | `gen_random_uuid()` (pgcrypto) | Sem sequenciais previsíveis; compatível com geração no cliente |
| Timestamps | `TIMESTAMP WITH TIME ZONE` | Armazena timezone; evita bugs de DST |
| Soft delete | `deleted_at TIMESTAMPTZ` | Apenas em `users` e `groups`; demais entidades são desativadas por status |
| Auditoria de `updated_at` | Trigger PostgreSQL | Garante atualização mesmo via SQL direto, não apenas via ORM |
| Valores monetários | `NUMERIC(10,2)` | Precisão exata; sem arredondamento de ponto flutuante |
| Skill | `NUMERIC(3,1)` | Escala 1.0–6.0 com uma casa decimal |

---

## Convenções de Nomenclatura

| Elemento | Padrão | Exemplo |
|---|---|---|
| Tabelas | `snake_case` plural | `group_members` |
| Colunas | `snake_case` | `presence_banned` |
| PKs | `id UUID` | `id` |
| FKs | `{tabela_singular}_id` | `group_id`, `user_id` |
| Índices | `idx_{tabela}_{coluna(s)}` | `idx_matches_group_id` |
| Constraints check | `chk_{tabela}_{regra}` | `chk_skill_range` |
| Constraints unique | `uq_{tabela}_{coluna(s)}` | `uq_group_members_group_user` |
| Triggers | `trg_{evento}_{tabela}` | `trg_updated_at_users` |
| Migrations | `V{NNN}__{descricao}.sql` | `V001__create_users.sql` |

---

## Estratégia de Auditoria

### Auditoria de Timestamps

Todas as tabelas possuem `created_at`. Tabelas mutáveis possuem `updated_at`, atualizado por trigger:

```sql
CREATE OR REPLACE FUNCTION fn_set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

O trigger é aplicado como `BEFORE UPDATE` em todas as tabelas com `updated_at`.

### Soft Delete

Aplicado somente em `users` e `groups`:
- `deleted_at TIMESTAMPTZ NULL` — `NULL` = ativo; preenchido = excluído
- Índices únicos usam `WHERE deleted_at IS NULL` para manter unicidade apenas nos registros ativos

### Auditoria de Banimentos

A tabela `presence_ban_history` registra cada ação de ban/unban como um log imutável (sem UPDATE/DELETE), garantindo rastreabilidade completa.

### Auditoria Financeira

`financial_entries` usa campo `status` com valor `REVERSED` em vez de DELETE. O estorno cria um novo registro de tipo contrário.

---

## Estratégia de Índices

### Índices por Padrão de Query

| Padrão | Índice | Tabela |
|---|---|---|
| Tenant isolation (hot path) | `(group_id)` | Todas as tabelas de negócio |
| Auth lookup | `(token_hash)` | `refresh_tokens` |
| Email único (ativo) | `(email) WHERE deleted_at IS NULL` | `users` |
| Lista de presença por partida | `(match_id, group_id)` | `presence_entries` |
| Fila de espera ordenada | `(match_id, position)` | `waiting_entries` |
| Partidas por data | `(group_id, scheduled_at DESC)` | `matches` |
| Cobranças pendentes | `(group_id, status)` | `charges` |
| Votação ativa | `(group_id, status)` | `skill_votings` |
| Financeiro por mês | `(group_id, reference_month)` | `financial_entries` |
| Financeiro por data | `(group_id, registered_at DESC)` | `financial_entries` |

---

## Inventário de Tabelas

| Tabela | BC | Aggregate Root | Registros estimados (1 ano) |
|---|---|---|---|
| `users` | Identity | User | 2.000 |
| `refresh_tokens` | Identity | — | 10.000 |
| `groups` | Group Mgmt | Group | 100 |
| `group_members` | Group Mgmt | — | 5.000 |
| `group_invites` | Group Mgmt | — | 500 |
| `referee_profiles` | Group Mgmt | — | 300 |
| `matches` | Match | Match | 2.000 |
| `presence_entries` | Match | — | 40.000 |
| `waiting_entries` | Match | — | 5.000 |
| `presence_ban_history` | Match | — | 500 |
| `referee_assignments` | Match | — | 2.000 |
| `skill_votings` | Voting | SkillVoting | 200 |
| `votes` | Voting | — | 20.000 |
| `voting_bans` | Voting | — | 100 |
| `team_formations` | Teams | TeamFormation | 2.000 |
| `teams` | Teams | — | 5.000 |
| `team_players` | Teams | — | 50.000 |
| `charges` | Payments | Charge | 30.000 |
| `payment_attempts` | Payments | — | 35.000 |
| `financial_entries` | Financial | FinancialEntry | 40.000 |

---

## Inventário de Migrations

| Versão | Arquivo | Conteúdo |
|---|---|---|
| V001 | `create_users` | `users`, `refresh_tokens` |
| V002 | `create_groups` | `groups` |
| V003 | `create_group_members` | `group_members` |
| V004 | `create_group_invites` | `group_invites` |
| V005 | `create_referee_profiles` | `referee_profiles` |
| V006 | `create_matches` | `matches` |
| V007 | `create_presence` | `presence_entries`, `waiting_entries`, `presence_ban_history` |
| V008 | `create_referee_assignments` | `referee_assignments` |
| V009 | `create_skill_votings` | `skill_votings`, `votes`, `voting_bans` |
| V010 | `create_team_formations` | `team_formations`, `teams`, `team_players` |
| V011 | `create_charges` | `charges`, `payment_attempts` |
| V012 | `create_financial_entries` | `financial_entries` |
| V013 | `create_indexes` | Todos os índices de performance |
| V014 | `create_audit` | Função e triggers de `updated_at` |

---

## Ambientes

| Ambiente | Banco | Config |
|---|---|---|
| development | `arenahub_dev` (Docker local) | `application-dev.yml` |
| test | `arenahub_test` (Testcontainers) | `application-test.yml` |
| staging | Railway PostgreSQL | Variáveis de ambiente |
| production | Railway PostgreSQL | Variáveis de ambiente |
