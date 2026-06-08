# Modelagem de Domínio — ArenaHub

## Metodologia

O domínio do ArenaHub é modelado usando **DDD Tático** com as seguintes construções:

- **Aggregate Root (AR):** entidade raiz que controla o ciclo de vida do agregado; única entrada para modificações
- **Entity:** objeto com identidade própria dentro de um agregado
- **Value Object (VO):** objeto imutável definido pelos seus atributos, sem identidade própria
- **Domain Service:** lógica que não pertence a nenhum agregado isolado
- **Domain Event:** fato relevante que ocorreu no domínio

---

## Bounded Contexts

O sistema é organizado em **7 Bounded Contexts** com responsabilidades bem delimitadas:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ArenaHub Domain                             │
│                                                                     │
│  ┌──────────────────┐    ┌──────────────────────────────────────┐  │
│  │ Identity &        │    │         Group Management             │  │
│  │ Access            │    │                                      │  │
│  │                   │    │  Group · GroupMember · GroupInvite   │  │
│  │  User             │    │  RefereeProfile · PlayerSkillProfile │  │
│  │  RefreshToken     │    │                                      │  │
│  └────────┬──────────┘    └──────────────┬───────────────────────┘  │
│           │                              │                           │
│  ┌────────▼──────────┐    ┌─────────────▼───────────────────────┐  │
│  │ Match &           │    │         Skill Voting                 │  │
│  │ Presence          │    │                                      │  │
│  │                   │    │  SkillVoting · Vote · VotingBan     │  │
│  │  Match            │    │                                      │  │
│  │  PresenceEntry    │    └──────────────────────────────────────┘  │
│  │  WaitingEntry     │                                              │
│  │  PresenceBan      │    ┌──────────────────────────────────────┐  │
│  │  RefereeAssign.   │    │         Team Formation               │  │
│  └────────┬──────────┘    │                                      │  │
│           │               │  TeamFormation · Team · TeamPlayer   │  │
│  ┌────────▼──────────┐    │                                      │  │
│  │ Payments          │    └──────────────────────────────────────┘  │
│  │                   │                                              │
│  │  Charge           │    ┌──────────────────────────────────────┐  │
│  │  PaymentAttempt   │    │         Financial                    │  │
│  │  PaymentValidation│    │                                      │  │
│  └───────────────────┘    │  FinancialEntry                      │  │
│                           │                                      │  │
│                           └──────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Descrição dos Bounded Contexts

### BC-01 — Identity & Access
**Responsabilidade:** Gerenciar identidade de usuários, autenticação e tokens de sessão.

**Aggregate Roots:** `User`  
**Entidades:** `RefreshToken`  
**Linguagem Ubíqua:** usuário, credencial, sessão, autenticação, OAuth

**Interações:**
- Publica `UserRegisteredEvent` ao criar conta
- Publica `UserDeletedEvent` ao excluir conta (outros contextos anonimizam dados)

---

### BC-02 — Group Management
**Responsabilidade:** Gerenciar grupos esportivos, membros, papéis, convites, skill e perfil de árbitro dentro do grupo.

**Aggregate Roots:** `Group`  
**Entidades dentro do Group:** `GroupMember`, `GroupInvite`, `RefereeProfile`  
**Value Objects:** `Sport`, `GroupRole`, `Skill`, `PlayerPosition`, `Money`, `InviteToken`, `ChargeType`  
**Linguagem Ubíqua:** grupo, membro, papel, convite, skill, árbitro, mensalista, banimento

**Invariantes do Agregado:**
- Todo grupo tem exatamente um membro com papel `OWNER`
- Skill de membro fica entre 1.0 e 6.0
- Convite expira em 7 dias ou 50 usos

---

### BC-03 — Match & Presence
**Responsabilidade:** Gerenciar partidas, lista de presença, fila de espera, banimentos de presença e associação de árbitro.

**Aggregate Roots:** `Match`  
**Entidades:** `PresenceEntry`, `WaitingEntry`, `RefereeAssignment`  
**Value Objects:** `MatchStatus`, `PresenceListStatus`, `PresenceStatus`, `Location`  
**Linguagem Ubíqua:** partida, presença, confirmação, recusa, fila de espera, banimento, fechamento automático

**Invariantes do Agregado:**
- Lista só fecha após horário de fechamento (T-1h)
- Após fechamento, nenhuma presença pode ser adicionada sem intervenção de admin
- Um membro tem no máximo uma entrada de presença por partida

---

### BC-04 — Skill Voting
**Responsabilidade:** Gerenciar votações de habilidades, votos individuais e banimentos de votação.

**Aggregate Roots:** `SkillVoting`  
**Entidades:** `Vote`, `VotingBan`  
**Value Objects:** `VotingStatus`, `Stars`  
**Linguagem Ubíqua:** votação, voto, estrelas, banimento de votação, encerramento, prazo

**Invariantes do Agregado:**
- Somente uma votação ativa por grupo
- Jogador não vota em si mesmo
- Votos de membros banidos são excluídos do cálculo

---

### BC-05 — Team Formation
**Responsabilidade:** Registrar e gerenciar formações de times por partida.

**Aggregate Roots:** `TeamFormation`  
**Entidades:** `Team`, `TeamPlayer`  
**Value Objects:** `FormationType`  
**Linguagem Ubíqua:** formação, time, distribuição, snake draft, equilíbrio

**Invariantes do Agregado:**
- Formação só possível após fechamento da lista de presença
- Árbitros não participam de times
- Desvio padrão de skill médio entre times ≤ 0.5

---

### BC-06 — Payments
**Responsabilidade:** Gerenciar cobranças, tentativas de pagamento e validação de comprovantes.

**Aggregate Roots:** `Charge`  
**Entidades:** `PaymentAttempt`  
**Value Objects:** `ChargeType`, `ChargeStatus`, `ValidationResult`, `ValidationSource`, `Money`  
**Linguagem Ubíqua:** cobrança, diária, mensalidade, comprovante, validação, IA, revisão manual

**Invariantes do Agregado:**
- Cobrança aprovada não pode ser revertida sem motivo registrado
- Tamanho máximo de comprovante: 10MB
- Tipos MIME aceitos: image/jpeg, image/png, application/pdf

---

### BC-07 — Financial
**Responsabilidade:** Registrar receitas e despesas do grupo; calcular saldo e gerar relatórios.

**Aggregate Roots:** `FinancialEntry`  
**Value Objects:** `EntryType`, `EntryCategory`, `Money`  
**Linguagem Ubíqua:** receita, despesa, saldo, relatório, categoria, estorno

**Invariantes do Agregado:**
- Entradas não são excluídas, somente estornadas com motivo
- Saldo = Σ(receitas) − Σ(despesas)

---

## Aggregates Detalhados

### User (AR — BC-01)

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | Identificador único |
| name | Name (VO) | Nome completo |
| email | Email (VO) | Email único no sistema |
| passwordHash | String? | Nulo para usuários OAuth |
| phone | Phone (VO) | Telefone |
| birthDate | LocalDate | Data de nascimento |
| photoUrl | String? | URL da foto no R2 |
| authProvider | AuthProvider (VO) | LOCAL \| GOOGLE |
| googleId | String? | ID do Google (para OAuth) |
| emailVerified | Boolean | Email confirmado |
| active | Boolean | Conta ativa |
| createdAt | Instant | — |
| updatedAt | Instant | — |

**Value Objects:**
- `Name`: String não vazio, 2–100 caracteres
- `Email`: String validada por regex RFC 5322
- `Phone`: String com DDD, 10–11 dígitos
- `AuthProvider`: Enum LOCAL | GOOGLE

---

### Group (AR — BC-02)

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| name | GroupName (VO) | 3–80 caracteres |
| sport | Sport (VO) | Modalidade esportiva |
| description | String? | Até 500 caracteres |
| photoUrl | String? | URL da foto no R2 |
| pixKey | String? | Chave Pix para validação de comprovantes |
| status | GroupStatus (VO) | ACTIVE \| INACTIVE |
| members | List\<GroupMember\> | Membros do grupo |
| invites | List\<GroupInvite\> | Convites pendentes |
| createdAt | Instant | — |
| updatedAt | Instant | — |

**GroupMember (Entity dentro de Group):**

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| userId | UUID | Referência ao User (inter-aggregate) |
| role | GroupRole (VO) | OWNER \| ADMIN \| PLAYER \| REFEREE |
| skill | Skill (VO) | 1.0–6.0; default 3.0 |
| skillSource | SkillSource (VO) | DEFAULT \| VOTING \| MANUAL |
| position | PlayerPosition (VO)? | Posição preferencial |
| isSubscriber | Boolean | É mensalista |
| subscriptionAmount | Money (VO)? | Valor da mensalidade |
| presenceBanned | Boolean | Banido de confirmar presença |
| presenceBanReason | String? | Motivo do banimento |
| joinedAt | Instant | — |

**GroupInvite (Entity dentro de Group):**

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| token | InviteToken (VO) | Token único |
| createdBy | UUID | userId do criador |
| expiresAt | Instant | Data de expiração (7 dias) |
| usageCount | Int | Usos até o momento |
| maxUsages | Int | Limite de usos (50) |
| active | Boolean | — |

**RefereeProfile (Entity dentro de Group):**

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| memberId | UUID | Referência ao GroupMember |
| chargeType | RefereeChargeType (VO) | PER_MATCH \| MONTHLY |
| amount | Money (VO) | Valor da cobrança |
| active | Boolean | — |

---

### Match (AR — BC-03)

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| groupId | UUID | Tenant |
| scheduledAt | Instant | Data/hora da partida |
| listClosesAt | Instant | scheduledAt − 1h |
| location | Location (VO) | Local da partida |
| maxPlayers | Int | Número máximo de jogadores |
| status | MatchStatus (VO) | SCHEDULED \| COMPLETED \| CANCELLED |
| presenceListStatus | PresenceListStatus (VO) | OPEN \| CLOSED |
| createdBy | UUID | — |
| presenceEntries | List\<PresenceEntry\> | — |
| waitingQueue | List\<WaitingEntry\> | Ordenada por posição |
| refereeAssignment | RefereeAssignment? | — |
| createdAt | Instant | — |

**PresenceEntry (Entity dentro de Match):**

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| memberId | UUID | — |
| status | PresenceStatus (VO) | CONFIRMED \| DECLINED \| BANNED_PENDING |
| justification | String? | Justificativa de banido |
| justificationStatus | JustificationStatus (VO)? | PENDING \| APPROVED \| REJECTED |
| confirmedAt | Instant? | — |

**WaitingEntry (Entity dentro de Match):**

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| memberId | UUID | — |
| position | Int | Posição na fila |
| notifiedAt | Instant? | Quando foi notificado de vaga |
| notificationDeadline | Instant? | notifiedAt + 30min |

**RefereeAssignment (Entity dentro de Match):**

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| refereeProfileId | UUID | — |
| financialEntryId | UUID | Despesa gerada no BC-07 |
| assignedAt | Instant | — |

---

### SkillVoting (AR — BC-04)

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| groupId | UUID | Tenant |
| status | VotingStatus (VO) | OPEN \| CLOSED |
| openedBy | UUID | userId |
| openedAt | Instant | — |
| deadline | Instant | Prazo de encerramento |
| closedAt | Instant? | — |
| votes | List\<Vote\> | — |
| bans | List\<VotingBan\> | — |

**Vote (Entity dentro de SkillVoting):**

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| voterId | UUID | Quem votou |
| targetMemberId | UUID | Em quem votou |
| stars | Stars (VO) | 1–6 |
| votedAt | Instant | — |

**VotingBan (Entity dentro de SkillVoting):**

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| memberId | UUID | Membro banido |
| reason | String | Motivo |
| bannedBy | UUID | Admin que baniu |
| bannedAt | Instant | — |

---

### TeamFormation (AR — BC-05)

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| matchId | UUID | — |
| groupId | UUID | Tenant |
| numberOfTeams | Int | Mínimo 2 |
| formationType | FormationType (VO) | AUTOMATIC \| MANUAL_ADJUSTED |
| confirmedBy | UUID | Admin |
| confirmedAt | Instant | — |
| teams | List\<Team\> | — |

**Team (Entity dentro de TeamFormation):**

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| name | String | "Time A", "Time B", etc. |
| averageSkill | Skill (VO) | Calculado |
| players | List\<TeamPlayer\> | — |

**TeamPlayer (Entity dentro de Team):**

| Atributo | Tipo | Descrição |
|---|---|---|
| memberId | UUID | — |
| position | PlayerPosition (VO)? | Posição no time |

---

### Charge (AR — BC-06)

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| groupId | UUID | Tenant |
| memberId | UUID | Quem deve |
| type | ChargeType (VO) | DAILY \| SUBSCRIPTION |
| amount | Money (VO) | Valor cobrado |
| referenceMatchId | UUID? | Para DAILY |
| referenceMonth | YearMonth? | Para SUBSCRIPTION |
| status | ChargeStatus (VO) | PENDING \| APPROVED \| REJECTED |
| attempts | List\<PaymentAttempt\> | — |
| createdAt | Instant | — |

**PaymentAttempt (Entity dentro de Charge):**

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| fileKey | String | Chave do arquivo no R2 |
| contentType | String | MIME type |
| submittedAt | Instant | — |
| validationResult | ValidationResult (VO)? | APPROVED \| REJECTED \| MANUAL_REVIEW |
| validationSource | ValidationSource (VO)? | AI \| MANUAL |
| validatedAt | Instant? | — |
| reviewedBy | UUID? | Admin (para MANUAL) |
| reviewNote | String? | — |

---

### FinancialEntry (AR — BC-07)

| Atributo | Tipo | Descrição |
|---|---|---|
| id | UUID | — |
| groupId | UUID | Tenant |
| type | EntryType (VO) | REVENUE \| EXPENSE |
| category | EntryCategory (VO) | DAILY \| SUBSCRIPTION \| REFEREE \| COURT \| OTHER |
| amount | Money (VO) | Valor positivo |
| description | String | — |
| matchId | UUID? | Partida relacionada |
| referenceMonth | YearMonth? | Para mensalidades |
| sourceChargeId | UUID? | Charge que originou a receita |
| registeredBy | UUID | — |
| registeredAt | Instant | — |
| status | EntryStatus (VO) | ACTIVE \| REVERSED |
| reversalReason | String? | — |
| reversedAt | Instant? | — |

---

## Value Objects Globais

| Value Object | Tipo Base | Regras de Validação |
|---|---|---|
| `Email` | String | Regex RFC 5322; lowercase; único no sistema |
| `Phone` | String | DDD + número; 10–11 dígitos; somente números |
| `Name` | String | 2–100 caracteres; não vazio |
| `GroupName` | String | 3–80 caracteres; não vazio |
| `Skill` | BigDecimal | 1.0–6.0; precisão de 1 casa decimal |
| `Stars` | Int | 1–6 |
| `Money` | BigDecimal | ≥ 0; 2 casas decimais (centavos) |
| `InviteToken` | String | UUID v4 sem hífens; único |
| `Location` | Record | name (String, obrigatório); address (String, opcional) |

---

## Domain Services

### `TeamFormationService`
Aplica o algoritmo snake draft para distribuir jogadores entre times.

**Responsabilidade:** lógica de formação que cruza dados de múltiplos membros e não pertence a um único agregado.

**Assinatura:**
```
TeamFormationResult form(List<MemberSkillDto> players, int numberOfTeams, Sport sport)
```

---

### `GroupMembershipService`
Verifica se um usuário é membro de um grupo e qual é seu papel.

**Responsabilidade:** validação de autorização baseada em membership que é usada transversalmente por múltiplos casos de uso.

---

### `SkillCalculationService`
Calcula o skill médio de cada membro após o encerramento de uma votação.

**Responsabilidade:** lógica de cálculo que agrega múltiplos votos; não pertence ao SkillVoting AR isoladamente pois deve atualizar o GroupMember.

---

### `SubscriptionPresenceService`
Verifica se um mensalista tem mensalidade paga no mês da partida.

**Responsabilidade:** cruza dados do BC-06 (Payments) e BC-02 (Group) para liberar presença.

---

## Domain Events

| Evento | Publicado por | Consumido por | Efeito |
|---|---|---|---|
| `UserRegisteredEvent` | User AR | — | — |
| `UserDeletedEvent` | User AR | Todos os BCs | Anonimização de dados pessoais |
| `MemberJoinedGroupEvent` | Group AR | — | Notificação ao grupo |
| `MemberRemovedFromGroupEvent` | Group AR | Match BC | Remove de listas abertas |
| `MatchCreatedEvent` | Match AR | — | Notificação a todos os PLAYERs |
| `PresenceListClosedEvent` | Match AR (scheduler) | — | Notifica fila de espera |
| `RefereeAssignedEvent` | Match AR | Financial BC | Gera despesa automática |
| `VotingClosedEvent` | SkillVoting AR | Group BC | Atualiza skill dos membros |
| `PaymentApprovedEvent` | Charge AR | Financial BC | Gera receita automática |
| `PaymentRequiresReviewEvent` | Charge AR | — | Notifica admins |

---

## Relacionamentos entre Agregados

```
User ──────────────────────────────── (referenciado por userId em)
  ├── GroupMember (BC-02)
  ├── PresenceEntry (BC-03)
  ├── Vote.voterId / Vote.targetMemberId (BC-04)
  ├── TeamPlayer (BC-05)
  ├── Charge.memberId (BC-06)
  └── FinancialEntry.registeredBy (BC-07)

Group ──────────────────────────────── (referenciado por groupId em)
  ├── Match (BC-03)
  ├── SkillVoting (BC-04)
  ├── TeamFormation (BC-05)
  ├── Charge (BC-06)
  └── FinancialEntry (BC-07)

Match ──────────────────────────────── (referenciado por matchId em)
  ├── TeamFormation (BC-05)
  ├── Charge.referenceMatchId (BC-06)
  └── FinancialEntry.matchId (BC-07)

Charge ─────────────────────────────── referenciado em
  └── FinancialEntry.sourceChargeId (BC-07)

RefereeAssignment ──────────────────── referenciado em
  └── FinancialEntry.id via refereeAssignment.financialEntryId
```

*Referências entre agregados são feitas **apenas por ID** (UUID), nunca por referência de objeto direto.*

---

## Decisões de Modelagem

### Por que `Group` contém `GroupMember` e não o contrário?
O agregado `Group` controla as invariantes de membership (somente um OWNER, unicidade de membros). A raiz do agregado precisa garantir essas invariantes ao adicionar/remover membros. Se membros fossem um agregado independente, seria impossível garantir "apenas um OWNER" sem transação distribuída.

### Por que `Match` não contém `TeamFormation`?
`TeamFormation` tem seu próprio ciclo de vida (pode ser recriada, tem histórico) e pode crescer com funcionalidades independentes. Mantê-la separada evita que o agregado `Match` fique sobrecarregado. A referência é feita por `matchId`.

### Por que `Charge` (cobrança) é separada de `FinancialEntry`?
`Charge` gerencia o ciclo de vida de uma cobrança (pendente → aprovada) e seus comprovantes. `FinancialEntry` é o registro contábil imutável que surge quando a cobrança é aprovada. São responsabilidades distintas com diferentes invariantes.

### Por que `SkillVoting` é um agregado separado?
A votação tem regras complexas independentes (unicidade por grupo, banimentos, anonimato, cálculo de média). Incorporá-la no `Group` tornaria esse agregado excessivamente complexo. A comunicação com `Group` (para atualizar skill) ocorre via `VotingClosedEvent`.
