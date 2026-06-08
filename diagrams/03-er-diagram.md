# Diagrama ER — ArenaHub

## Convenções

- Tabelas seguem `snake_case`
- PKs: `id UUID DEFAULT gen_random_uuid()`
- Todas as tabelas de negócio têm `group_id UUID NOT NULL` (multi-tenancy)
- Soft delete: campo `deleted_at TIMESTAMP WITH TIME ZONE`
- Auditoria: `created_at`, `updated_at` em todas as tabelas

---

## Diagrama ER (Mermaid)

```mermaid
erDiagram

  %% ─── BC-01: Identity & Access ───────────────────────────────────────

  users {
    uuid id PK
    varchar name
    varchar email UK
    varchar password_hash
    varchar phone
    date birth_date
    varchar photo_url
    varchar auth_provider
    varchar google_id
    bool email_verified
    bool active
    timestamp created_at
    timestamp updated_at
    timestamp deleted_at
  }

  refresh_tokens {
    uuid id PK
    uuid user_id FK
    varchar token_hash
    varchar status
    timestamp expires_at
    timestamp created_at
  }

  %% ─── BC-02: Group Management ─────────────────────────────────────────

  groups {
    uuid id PK
    varchar name
    varchar sport
    text description
    varchar photo_url
    varchar pix_key
    varchar status
    timestamp created_at
    timestamp updated_at
    timestamp deleted_at
  }

  group_members {
    uuid id PK
    uuid group_id FK
    uuid user_id FK
    varchar role
    decimal skill
    varchar skill_source
    varchar position
    bool is_subscriber
    decimal subscription_amount
    bool presence_banned
    text presence_ban_reason
    timestamp joined_at
    timestamp updated_at
  }

  group_invites {
    uuid id PK
    uuid group_id FK
    uuid created_by FK
    varchar token UK
    int usage_count
    int max_usages
    bool active
    timestamp expires_at
    timestamp created_at
  }

  referee_profiles {
    uuid id PK
    uuid group_id FK
    uuid member_id FK
    varchar charge_type
    decimal amount
    bool active
    timestamp created_at
    timestamp updated_at
  }

  %% ─── BC-03: Match & Presence ─────────────────────────────────────────

  matches {
    uuid id PK
    uuid group_id FK
    timestamp scheduled_at
    timestamp list_closes_at
    varchar location_name
    text location_address
    int max_players
    varchar status
    varchar presence_list_status
    uuid created_by FK
    timestamp created_at
    timestamp updated_at
  }

  presence_entries {
    uuid id PK
    uuid match_id FK
    uuid group_id FK
    uuid member_id FK
    varchar status
    text justification
    varchar justification_status
    timestamp confirmed_at
    timestamp updated_at
  }

  waiting_entries {
    uuid id PK
    uuid match_id FK
    uuid group_id FK
    uuid member_id FK
    int position
    timestamp notified_at
    timestamp notification_deadline
    timestamp created_at
  }

  referee_assignments {
    uuid id PK
    uuid match_id FK
    uuid group_id FK
    uuid referee_profile_id FK
    uuid financial_entry_id FK
    timestamp assigned_at
  }

  presence_ban_history {
    uuid id PK
    uuid group_id FK
    uuid member_id FK
    varchar action
    text reason
    uuid performed_by FK
    timestamp performed_at
  }

  %% ─── BC-04: Skill Voting ─────────────────────────────────────────────

  skill_votings {
    uuid id PK
    uuid group_id FK
    varchar status
    uuid opened_by FK
    timestamp opened_at
    timestamp deadline
    timestamp closed_at
  }

  votes {
    uuid id PK
    uuid voting_id FK
    uuid group_id FK
    uuid voter_id FK
    uuid target_member_id FK
    int stars
    timestamp voted_at
    timestamp updated_at
  }

  voting_bans {
    uuid id PK
    uuid voting_id FK
    uuid group_id FK
    uuid member_id FK
    text reason
    uuid banned_by FK
    timestamp banned_at
  }

  %% ─── BC-05: Team Formation ───────────────────────────────────────────

  team_formations {
    uuid id PK
    uuid match_id FK
    uuid group_id FK
    int number_of_teams
    varchar formation_type
    uuid confirmed_by FK
    timestamp confirmed_at
  }

  teams {
    uuid id PK
    uuid formation_id FK
    uuid group_id FK
    varchar name
    decimal average_skill
    int player_count
  }

  team_players {
    uuid id PK
    uuid team_id FK
    uuid group_id FK
    uuid member_id FK
    varchar position
  }

  %% ─── BC-06: Payments ─────────────────────────────────────────────────

  charges {
    uuid id PK
    uuid group_id FK
    uuid member_id FK
    varchar type
    decimal amount
    uuid reference_match_id FK
    varchar reference_month
    varchar status
    timestamp created_at
    timestamp updated_at
  }

  payment_attempts {
    uuid id PK
    uuid charge_id FK
    uuid group_id FK
    varchar file_key
    varchar content_type
    varchar validation_result
    varchar validation_source
    text validation_reason
    uuid reviewed_by FK
    text review_note
    timestamp submitted_at
    timestamp validated_at
    timestamp reviewed_at
  }

  %% ─── BC-07: Financial ────────────────────────────────────────────────

  financial_entries {
    uuid id PK
    uuid group_id FK
    varchar type
    varchar category
    decimal amount
    text description
    uuid match_id FK
    varchar reference_month
    uuid source_charge_id FK
    uuid registered_by FK
    timestamp registered_at
    varchar status
    text reversal_reason
    uuid reversed_by FK
    timestamp reversed_at
  }

  %% ─── RELACIONAMENTOS ──────────────────────────────────────────────────

  users ||--o{ refresh_tokens : "has"
  users ||--o{ group_members : "belongs to"

  groups ||--o{ group_members : "has"
  groups ||--o{ group_invites : "generates"
  groups ||--o{ referee_profiles : "has"
  groups ||--o{ matches : "hosts"
  groups ||--o{ skill_votings : "runs"
  groups ||--o{ team_formations : "has"
  groups ||--o{ charges : "bills"
  groups ||--o{ financial_entries : "records"

  group_members ||--o{ referee_profiles : "may be"
  group_members ||--o{ presence_entries : "has"
  group_members ||--o{ waiting_entries : "may be in"
  group_members ||--o{ votes : "casts"
  group_members ||--o{ votes : "receives"
  group_members ||--o{ voting_bans : "may receive"
  group_members ||--o{ team_players : "is placed in"
  group_members ||--o{ charges : "has"
  group_members ||--o{ presence_ban_history : "has"

  matches ||--o{ presence_entries : "has"
  matches ||--o{ waiting_entries : "has"
  matches ||--o| referee_assignments : "may have"
  matches ||--o{ team_formations : "has"
  matches ||--o{ charges : "generates"
  matches ||--o{ financial_entries : "has"

  skill_votings ||--o{ votes : "contains"
  skill_votings ||--o{ voting_bans : "has"

  team_formations ||--o{ teams : "contains"
  teams ||--o{ team_players : "has"

  charges ||--o{ payment_attempts : "has"
  charges ||--o{ financial_entries : "generates"

  referee_profiles ||--o{ referee_assignments : "assigned via"
  referee_assignments ||--|| financial_entries : "generates"
```

---

## Índices Planejados

```sql
-- Acesso por tenant (group_id) — todos os principais endpoints
CREATE INDEX idx_group_members_group_id ON group_members(group_id);
CREATE INDEX idx_group_members_user_id ON group_members(user_id);
CREATE INDEX idx_group_members_group_user ON group_members(group_id, user_id);

CREATE INDEX idx_matches_group_id ON matches(group_id);
CREATE INDEX idx_matches_scheduled_at ON matches(group_id, scheduled_at DESC);

CREATE INDEX idx_presence_entries_match ON presence_entries(match_id, group_id);
CREATE INDEX idx_presence_entries_member ON presence_entries(member_id, group_id);

CREATE INDEX idx_waiting_entries_match ON waiting_entries(match_id, position);

CREATE INDEX idx_skill_votings_group ON skill_votings(group_id, status);

CREATE INDEX idx_votes_voting ON votes(voting_id);
CREATE INDEX idx_votes_target ON votes(voting_id, target_member_id);

CREATE INDEX idx_charges_group_member ON charges(group_id, member_id);
CREATE INDEX idx_charges_status ON charges(group_id, status);
CREATE INDEX idx_charges_match ON charges(reference_match_id) WHERE reference_match_id IS NOT NULL;

CREATE INDEX idx_financial_entries_group ON financial_entries(group_id, registered_at DESC);
CREATE INDEX idx_financial_entries_month ON financial_entries(group_id, reference_month);
CREATE INDEX idx_financial_entries_match ON financial_entries(match_id) WHERE match_id IS NOT NULL;

-- Token lookup (autenticação — hot path)
CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens(token_hash);
CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id, status);

-- Convite por token
CREATE UNIQUE INDEX idx_group_invites_token ON group_invites(token);

-- Email único
CREATE UNIQUE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
```

---

## Constraints de Integridade

```sql
-- Skill entre 1.0 e 6.0
ALTER TABLE group_members ADD CONSTRAINT chk_skill_range
  CHECK (skill >= 1.0 AND skill <= 6.0);

-- Stars entre 1 e 6
ALTER TABLE votes ADD CONSTRAINT chk_stars_range
  CHECK (stars >= 1 AND stars <= 6);

-- Money não negativo
ALTER TABLE charges ADD CONSTRAINT chk_charge_amount_positive
  CHECK (amount > 0);

ALTER TABLE financial_entries ADD CONSTRAINT chk_financial_amount_positive
  CHECK (amount > 0);

ALTER TABLE referee_profiles ADD CONSTRAINT chk_referee_amount_positive
  CHECK (amount > 0);

-- Max players positivo
ALTER TABLE matches ADD CONSTRAINT chk_max_players_positive
  CHECK (max_players > 0);

-- Unicidade de membro por grupo
ALTER TABLE group_members ADD CONSTRAINT uq_group_member
  UNIQUE (group_id, user_id);

-- Unicidade de presença por partida/membro
ALTER TABLE presence_entries ADD CONSTRAINT uq_presence_entry
  UNIQUE (match_id, member_id);

-- Uma votação ativa por grupo (enforced na aplicação; RLS complementar)

-- Votação: voter != target
ALTER TABLE votes ADD CONSTRAINT chk_no_self_vote
  CHECK (voter_id != target_member_id);

-- Um árbitro por partida
ALTER TABLE referee_assignments ADD CONSTRAINT uq_referee_assignment
  UNIQUE (match_id);
```
