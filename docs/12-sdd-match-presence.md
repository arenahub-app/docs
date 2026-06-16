# SDD — Módulo de Partidas e Presença (Etapa 5, Feature 2)

**Versão:** 1.0  
**Data:** 2026-06-16  
**Status:** Em revisão  
**Refs:** US-010, US-011, US-012, US-013, US-014, RN-PRE-001–008, RN-ARB-002, RN-MBR-004, BC-03

---

## 1. Objetivo

Implementar o módulo de Partidas e Presença do ArenaHub, cobrindo:

1. Criação e gerenciamento de partidas por grupo
2. Confirmação, recusa e cancelamento de presença pelos jogadores
3. Fila de espera automática quando a partida está cheia
4. Fechamento automático da lista 60 minutos antes da partida (scheduler)
5. Fechamento manual da lista por ADMIN/OWNER
6. Sobrescritas de presença por ADMIN após fechamento da lista
7. Banimento e desbanimento de presença por grupo

Este módulo é pré-requisito direto da Feature 3 (Skill Voting pela presença) e da Feature 4 (Team Formation).

---

## 2. Escopo

**Incluído neste módulo:**
- Aggregate `Match` com entidades `PresenceEntry` e `WaitingEntry`
- 13 endpoints REST (`/api/v1/groups/{groupId}/matches/**`)
- Scheduler Spring para fechamento automático de lista (RN-PRE-001)
- Promoção automática da fila quando vaga abre (sem notificação push — adiado para Feature 4.7)
- Banimento/desbanimento de presença (extiende Group Management — modifica `GroupMember`)
- Log imutável de banimentos (`presence_ban_history`)
- 3 telas no frontend
- Migrations já existentes (V006, V007, V008)

**Fora do escopo (módulos futuros):**
- `RefereeAssignment` — depende de BC Financial (Feature 5+)
- Notificações push ao confirmar/cancelar presença (Feature 4.7 — Mensageria Assíncrona)
- Notificação aos 30 min da fila (RN-PRE-003) — infraestrutura de mensageria pendente; fila é mantida e promovida, notificação é deferida
- Fluxo de justificativa para banidos (BANNED_PENDING → APPROVED/REJECTED) — simplificado: banido recebe 403 com motivo ao tentar confirmar
- `RN-PRE-008` (mensalistas confirmam sem pagar diária) — validação cruzada com BC Payments (Feature 5+)
- Formação de times (Feature 3)

---

## 3. Arquitetura em Camadas

```
presentation/match/
├── MatchController.java          ← /api/v1/groups/{groupId}/matches/**
├── PresenceController.java       ← endpoints de presença e ban
└── dto/
    ├── CreateMatchRequest.java
    ├── UpdateMatchRequest.java
    ├── MatchResponse.java
    ├── MatchSummaryResponse.java
    ├── PresenceListResponse.java
    ├── ConfirmPresenceRequest.java
    ├── PresenceEntryResponse.java
    ├── WaitingEntryResponse.java
    ├── AdminPresenceRequest.java
    └── BanMemberRequest.java

application/match/
├── port/in/
│   ├── CreateMatchUseCase.java
│   ├── GetMatchUseCase.java
│   ├── ListMatchesUseCase.java
│   ├── UpdateMatchUseCase.java
│   ├── CancelMatchUseCase.java
│   ├── ClosePresenceListUseCase.java
│   ├── GetPresenceListUseCase.java
│   ├── ConfirmPresenceUseCase.java
│   ├── CancelPresenceUseCase.java
│   ├── AdminForcePresenceUseCase.java
│   ├── AdminRemovePresenceUseCase.java
│   ├── BanFromPresenceUseCase.java
│   └── UnbanFromPresenceUseCase.java
├── port/out/
│   ├── MatchRepository.java
│   └── GroupMemberPort.java      ← lê GroupMember (presenceBanned, role) do BC Group
└── MatchService.java             ← implementa todos os use cases acima

domain/match/
├── Match.java                    ← AR
├── PresenceEntry.java
├── WaitingEntry.java
└── vo/
    ├── MatchStatus.java          (SCHEDULED | COMPLETED | CANCELLED)
    ├── PresenceListStatus.java   (OPEN | CLOSED)
    ├── PresenceStatus.java       (CONFIRMED | DECLINED | BANNED_PENDING)
    ├── JustificationStatus.java  (PENDING | APPROVED | REJECTED)
    └── Location.java             (name: String, address: String?)

infrastructure/persistence/match/
├── MatchJpaEntity.java
├── PresenceEntryJpaEntity.java
├── WaitingEntryJpaEntity.java
├── PresenceBanHistoryJpaEntity.java
├── MatchJpaRepository.java
├── PresenceEntryJpaRepository.java
├── WaitingEntryJpaRepository.java
├── PresenceBanHistoryJpaRepository.java
└── MatchRepositoryAdapter.java   ← implementa MatchRepository + GroupMemberPort

infrastructure/scheduler/
└── PresenceListScheduler.java    ← @Scheduled a cada 60s, fecha listas vencidas
```

---

## 4. Modelo de Domínio

### 4.1 Match (Aggregate Root)

```java
public class Match {
    private UUID id;
    private UUID groupId;
    private Instant scheduledAt;
    private Instant listClosesAt;    // = scheduledAt − 1h; calculado na factory
    private Location location;
    private int maxPlayers;
    private MatchStatus status;                // SCHEDULED | COMPLETED | CANCELLED
    private PresenceListStatus presenceListStatus; // OPEN | CLOSED
    private UUID createdBy;
    private Instant createdAt;
    private Instant updatedAt;

    public static Match create(UUID groupId, Instant scheduledAt,
                               Location location, int maxPlayers, UUID createdBy) {
        // Valida scheduledAt > now + 30min (partida não pode ser criada no passado)
        // listClosesAt = scheduledAt − 1h
        // status = SCHEDULED, presenceListStatus = OPEN
    }

    public void update(Instant scheduledAt, Location location, int maxPlayers) {
        // Só permitido se status = SCHEDULED e presenceListStatus = OPEN
        // Recalcula listClosesAt ao alterar scheduledAt
    }

    public void cancel() {
        // status = CANCELLED; presenceListStatus = CLOSED
        // Somente se status = SCHEDULED
    }

    public void closePresenceList() {
        // presenceListStatus = OPEN → CLOSED
        // Somente se presenceListStatus = OPEN
    }

    public boolean isListOpen() {
        return presenceListStatus == PresenceListStatus.OPEN;
    }

    public boolean isPast() {
        return scheduledAt.isBefore(Instant.now());
    }
}
```

### 4.2 PresenceEntry (Entity dentro de Match)

```java
public class PresenceEntry {
    private UUID id;
    private UUID matchId;
    private UUID groupId;
    private UUID memberId;
    private PresenceStatus status;          // CONFIRMED | DECLINED | BANNED_PENDING
    private String justification;           // obrigatório se BANNED_PENDING
    private JustificationStatus justificationStatus; // PENDING | APPROVED | REJECTED
    private Instant confirmedAt;            // preenchido se CONFIRMED

    public static PresenceEntry confirm(UUID matchId, UUID groupId, UUID memberId) {
        // status = CONFIRMED, confirmedAt = now
    }

    public static PresenceEntry decline(UUID matchId, UUID groupId, UUID memberId) {
        // status = DECLINED
    }

    public static PresenceEntry pendingBan(UUID matchId, UUID groupId, UUID memberId) {
        // status = BANNED_PENDING, justificationStatus = PENDING
    }
}
```

### 4.3 WaitingEntry (Entity dentro de Match)

```java
public class WaitingEntry {
    private UUID id;
    private UUID matchId;
    private UUID groupId;
    private UUID memberId;
    private int position;               // posição FIFO (começa em 1)
    private Instant notifiedAt;         // null até ser notificado (futuro: mensageria)
    private Instant notificationDeadline; // notifiedAt + 30min (futuro)
    private Instant createdAt;

    public static WaitingEntry create(UUID matchId, UUID groupId, UUID memberId, int nextPosition) {
        // position = nextPosition; notifiedAt = null
    }
}
```

### 4.4 Value Objects

| VO | Tipo Base | Validação |
|---|---|---|
| `MatchStatus` | Enum | `SCHEDULED`, `COMPLETED`, `CANCELLED` |
| `PresenceListStatus` | Enum | `OPEN`, `CLOSED` |
| `PresenceStatus` | Enum | `CONFIRMED`, `DECLINED`, `BANNED_PENDING` |
| `JustificationStatus` | Enum | `PENDING`, `APPROVED`, `REJECTED` |
| `Location` | Record | `name` obrigatório (max 200 chars); `address` opcional |

### 4.5 GroupMemberPort (leitura cross-BC)

Para verificar banimento e papel do membro sem acoplar ao aggregate `Group`, o `MatchService` usa uma port de saída simples:

```java
public interface GroupMemberPort {
    Optional<GroupMemberView> findMember(UUID groupId, UUID userId);
    void banFromPresence(UUID memberId, String reason, UUID bannedBy, UUID groupId);
    void unbanFromPresence(UUID memberId, UUID groupId, UUID performedBy);
}

public record GroupMemberView(
    UUID id,           // memberId
    UUID userId,
    GroupRole role,
    boolean presenceBanned,
    String presenceBanReason
) {}
```

A implementação (`GroupMemberPortAdapter`) usa o `GroupMemberJpaRepository` existente e `PresenceBanHistoryJpaRepository` para o log.

---

## 5. Endpoints da API

### 5.1 Criar Partida

```
POST /api/v1/groups/{groupId}/matches
Authorization: Bearer <token>
Content-Type: application/json

{
  "scheduledAt": "2026-07-10T20:00:00Z",
  "locationName": "Quadra do Parque",
  "locationAddress": "Av. Central, 100",
  "maxPlayers": 14
}
```

**Autorização:** `OWNER` ou `ADMIN`

**Fluxo:**
1. Valida que o grupo existe e está `ACTIVE`
2. Valida que `scheduledAt > now + 30min`
3. Calcula `listClosesAt = scheduledAt − 1h`
4. Cria `Match` com `status=SCHEDULED`, `presenceListStatus=OPEN`
5. Persiste e retorna `201 Created` com `MatchResponse`

**Respostas:**

| Status | Condição |
|---|---|
| 201 | Sucesso |
| 400 | Campos inválidos ou `scheduledAt` no passado/muito próximo |
| 403 | Papel insuficiente ou grupo inativo |
| 404 | Grupo não encontrado |

---

### 5.2 Listar Partidas do Grupo

```
GET /api/v1/groups/{groupId}/matches?filter=upcoming
Authorization: Bearer <token>
```

**Autorização:** qualquer membro do grupo

**Query params:**
- `filter`: `upcoming` (default) | `past` | `all`
- `page` / `size` (default 20): paginação (para futura escala)

**Response:**
```json
[
  {
    "id": "uuid",
    "scheduledAt": "2026-07-10T20:00:00Z",
    "listClosesAt": "2026-07-10T19:00:00Z",
    "locationName": "Quadra do Parque",
    "locationAddress": "Av. Central, 100",
    "maxPlayers": 14,
    "status": "SCHEDULED",
    "presenceListStatus": "OPEN",
    "confirmedCount": 8,
    "waitingCount": 2
  }
]
```

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso (lista vazia se nenhuma partida) |
| 403 | Não é membro |
| 404 | Grupo não encontrado |

---

### 5.3 Detalhes da Partida

```
GET /api/v1/groups/{groupId}/matches/{matchId}
Authorization: Bearer <token>
```

**Autorização:** qualquer membro do grupo

**Response:**
```json
{
  "id": "uuid",
  "groupId": "uuid",
  "scheduledAt": "2026-07-10T20:00:00Z",
  "listClosesAt": "2026-07-10T19:00:00Z",
  "locationName": "Quadra do Parque",
  "locationAddress": "Av. Central, 100",
  "maxPlayers": 14,
  "status": "SCHEDULED",
  "presenceListStatus": "OPEN",
  "createdBy": "uuid",
  "createdAt": "2026-06-16T00:00:00Z",
  "myPresenceStatus": "CONFIRMED",
  "confirmedCount": 8,
  "waitingCount": 2
}
```

> `myPresenceStatus`: `"CONFIRMED"` | `"DECLINED"` | `"BANNED_PENDING"` | `"WAITING"` | `null` (ainda não interagiu)

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso |
| 403 | Não é membro |
| 404 | Partida ou grupo não encontrado |

---

### 5.4 Atualizar Partida

```
PATCH /api/v1/groups/{groupId}/matches/{matchId}
Authorization: Bearer <token>
Content-Type: application/json

{
  "scheduledAt": "2026-07-10T21:00:00Z",
  "locationName": "Quadra Nova",
  "maxPlayers": 16
}
```

**Autorização:** `OWNER` ou `ADMIN`

**Regras:**
- Só permitido se `status = SCHEDULED` e `presenceListStatus = OPEN`
- Campos omitidos não são alterados
- Ao alterar `scheduledAt`, `listClosesAt` é recalculado automaticamente

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso com MatchResponse atualizado |
| 400 | Campos inválidos |
| 403 | Papel insuficiente |
| 404 | Partida não encontrada |
| 422 | `match-not-editable` — lista já fechada ou partida cancelada |

---

### 5.5 Cancelar Partida

```
POST /api/v1/groups/{groupId}/matches/{matchId}/cancel
Authorization: Bearer <token>
```

**Autorização:** `OWNER` ou `ADMIN`

**Fluxo:**
1. Valida `status = SCHEDULED`
2. Define `status = CANCELLED`, `presenceListStatus = CLOSED`
3. Não remove presenças existentes (histórico preservado)

**Respostas:**

| Status | Condição |
|---|---|
| 204 | Sucesso |
| 403 | Papel insuficiente |
| 404 | Partida não encontrada |
| 422 | `match-already-cancelled` |

---

### 5.6 Fechar Lista Manualmente

```
POST /api/v1/groups/{groupId}/matches/{matchId}/close-list
Authorization: Bearer <token>
```

**Autorização:** `OWNER` ou `ADMIN`

**Fluxo:**
1. Valida `presenceListStatus = OPEN`
2. Define `presenceListStatus = CLOSED`
3. Membros na fila de espera permanecem registrados (dados históricos)

**Respostas:**

| Status | Condição |
|---|---|
| 204 | Sucesso |
| 403 | Papel insuficiente |
| 404 | Partida não encontrada |
| 422 | `list-already-closed` |

---

### 5.7 Listar Presenças e Fila

```
GET /api/v1/groups/{groupId}/matches/{matchId}/presence
Authorization: Bearer <token>
```

**Autorização:** qualquer membro do grupo

**Response:**
```json
{
  "confirmed": [
    {
      "id": "uuid",
      "memberId": "uuid",
      "userName": "Carlos Drummond",
      "role": "PLAYER",
      "skill": 4.5,
      "position": "FORWARD",
      "confirmedAt": "2026-07-08T14:30:00Z"
    }
  ],
  "declined": [
    {
      "id": "uuid",
      "memberId": "uuid",
      "userName": "João Silva",
      "role": "PLAYER",
      "skill": 3.0
    }
  ],
  "waiting": [
    {
      "id": "uuid",
      "memberId": "uuid",
      "userName": "Ana Lima",
      "role": "PLAYER",
      "skill": 3.5,
      "position": 1,
      "createdAt": "2026-07-08T15:00:00Z"
    }
  ]
}
```

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso |
| 403 | Não é membro |
| 404 | Partida ou grupo não encontrado |

---

### 5.8 Confirmar ou Recusar Presença

```
POST /api/v1/groups/{groupId}/matches/{matchId}/presence
Authorization: Bearer <token>
Content-Type: application/json

{
  "action": "CONFIRM"
}
```

**Autorização:** qualquer membro com `role ≠ REFEREE` (RN-ARB-002, RN-MBR-004)

**Ações:**
- `"CONFIRM"`: confirmar presença
- `"DECLINE"`: recusar presença

**Fluxo para CONFIRM:**
1. Valida que o membro existe no grupo e não é `REFEREE`
2. Valida `presenceListStatus = OPEN` (RN-PRE-002)
3. Verifica que não há `PresenceEntry` com `status = CONFIRMED` para este membro nesta partida (RN-PRE-007)
4. Se membro tem `presenceBanned = true` → cria entry com `status = BANNED_PENDING` e retorna `200` com detalhe do banimento
5. Se vagas disponíveis (`confirmedCount < maxPlayers`) → cria `PresenceEntry(status=CONFIRMED)`
6. Se partida cheia → cria `WaitingEntry` com `position = maxPosition + 1`
7. Retorna `200` com `PresenceEntryResponse` ou `WaitingEntryResponse`

**Fluxo para DECLINE:**
1. Valida que o membro existe no grupo
2. Valida `presenceListStatus = OPEN`
3. Se havia `PresenceEntry(CONFIRMED)` → exclui presença e promove primeiro da fila (se houver)
4. Cria/atualiza entry com `status = DECLINED`

**Promoção da fila (automática ao cancelar presença):**
1. Busca `WaitingEntry` com menor `position` para esta partida
2. Remove da fila de espera
3. Cria `PresenceEntry(CONFIRMED)` para o membro promovido
4. (Notificação push: deferida para Feature 4.7)

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso — retorna entry criada (presença ou fila) |
| 400 | `action` inválido |
| 403 | `REFEREE` tentando confirmar presença |
| 404 | Partida ou grupo não encontrado |
| 409 | `already-confirmed` — já confirmou esta partida |
| 422 | `list-closed` — lista já fechada |

---

### 5.9 Cancelar Própria Presença

```
DELETE /api/v1/groups/{groupId}/matches/{matchId}/presence
Authorization: Bearer <token>
```

**Autorização:** o próprio membro

**Fluxo:**
1. Valida `presenceListStatus = OPEN` (RN-PRE-002)
2. Busca `PresenceEntry` do membro nesta partida
3. Remove a entry de presença
4. Se havia `WaitingEntry` do membro → remove da fila
5. Se membro estava `CONFIRMED` → promove primeiro da fila (se houver)

**Respostas:**

| Status | Condição |
|---|---|
| 204 | Sucesso |
| 404 | Presença não encontrada |
| 422 | `list-closed` — lista já fechada |

---

### 5.10 Admin: Forçar Presença (pós-fechamento)

```
PUT /api/v1/groups/{groupId}/matches/{matchId}/presence/{memberId}
Authorization: Bearer <token>
```

**Autorização:** `OWNER` ou `ADMIN`

**Regra:** Permitido mesmo com `presenceListStatus = CLOSED` (RN-PRE-002 — intervenção manual).

**Fluxo:**
1. Valida papel do caller
2. Cria ou atualiza `PresenceEntry` para o membro com `status = CONFIRMED`
3. Remove da fila de espera se estava lá

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso com PresenceEntryResponse |
| 403 | Papel insuficiente |
| 404 | Membro não encontrado no grupo |

---

### 5.11 Admin: Remover Presença (pós-fechamento)

```
DELETE /api/v1/groups/{groupId}/matches/{matchId}/presence/{memberId}
Authorization: Bearer <token>
```

**Autorização:** `OWNER` ou `ADMIN`

**Regra:** Permitido mesmo com lista fechada.

**Respostas:**

| Status | Condição |
|---|---|
| 204 | Sucesso |
| 403 | Papel insuficiente |
| 404 | Presença não encontrada |

---

### 5.12 Banir Membro de Presença

```
POST /api/v1/groups/{groupId}/members/{memberId}/presence-ban
Authorization: Bearer <token>
Content-Type: application/json

{
  "reason": "Ausências repetidas sem aviso"
}
```

**Autorização:** `OWNER` ou `ADMIN`

**Fluxo:**
1. Valida papel do caller
2. Valida que o membro existe no grupo e não está já banido
3. Atualiza `GroupMember`: `presenceBanned = true`, `presenceBanReason = reason`
4. Insere em `presence_ban_history` com `action = 'BANNED'`
5. Retorna `200` com dados atualizados do membro

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso com MemberResponse atualizado |
| 400 | `reason` ausente |
| 403 | Papel insuficiente |
| 404 | Membro não encontrado |
| 409 | `member-already-banned` |

---

### 5.13 Remover Banimento de Presença

```
DELETE /api/v1/groups/{groupId}/members/{memberId}/presence-ban
Authorization: Bearer <token>
```

**Autorização:** `OWNER` ou `ADMIN`

**Fluxo:**
1. Valida papel
2. Valida que o membro está banido
3. Atualiza `GroupMember`: `presenceBanned = false`, `presenceBanReason = null`
4. Insere em `presence_ban_history` com `action = 'UNBANNED'`

**Respostas:**

| Status | Condição |
|---|---|
| 204 | Sucesso |
| 403 | Papel insuficiente |
| 404 | Membro não encontrado |
| 409 | `member-not-banned` |

---

## 6. Scheduler — Fechamento Automático de Lista

```java
@Component
public class PresenceListScheduler {

    private final MatchRepository matchRepository;

    @Scheduled(fixedDelay = 60_000)  // a cada 60 segundos
    @Transactional
    public void closeExpiredLists() {
        List<Match> toClose = matchRepository
            .findByPresenceListStatusAndListClosesAtBefore(
                PresenceListStatus.OPEN, Instant.now());

        for (Match match : toClose) {
            match.closePresenceList();
            matchRepository.save(match);
            // Notificação aos da fila de espera: deferida para Feature 4.7
        }
    }
}
```

**Index de suporte (já existe na V013):**
```sql
CREATE INDEX idx_matches_list_closes_at ON matches (list_closes_at)
    WHERE presence_list_status = 'OPEN';
```

---

## 7. Matriz de Autorização

| Operação | OWNER | ADMIN | PLAYER | REFEREE |
|---|---|---|---|---|
| Criar partida | ✓ | ✓ | ✗ | ✗ |
| Ver detalhes da partida | ✓ | ✓ | ✓ | ✓ |
| Listar partidas | ✓ | ✓ | ✓ | ✓ |
| Atualizar partida | ✓ | ✓ | ✗ | ✗ |
| Cancelar partida | ✓ | ✓ | ✗ | ✗ |
| Fechar lista manualmente | ✓ | ✓ | ✗ | ✗ |
| Confirmar/recusar presença | ✓ | ✓ | ✓ | ✗ |
| Cancelar própria presença | ✓ | ✓ | ✓ | ✗ |
| Forçar/remover presença (pós-fechamento) | ✓ | ✓ | ✗ | ✗ |
| Ver lista de presença | ✓ | ✓ | ✓ | ✓ |
| Banir / desbanir membro | ✓ | ✓ | ✗ | ✗ |

---

## 8. Tratamento de Erros

```json
{
  "type": "/errors/list-closed",
  "title": "Lista de presença fechada",
  "status": 422,
  "detail": "A lista de presença desta partida já foi encerrada.",
  "instance": "/api/v1/groups/uuid/matches/uuid/presence"
}
```

| Código de Negócio | HTTP | Cenário |
|---|---|---|
| `match-not-found` | 404 | Partida não existe ou não pertence ao grupo |
| `match-not-editable` | 422 | Tentativa de editar partida com lista fechada ou cancelada |
| `match-already-cancelled` | 422 | Cancelar partida já cancelada |
| `list-already-closed` | 422 | Fechar lista já fechada |
| `list-closed` | 422 | Confirmar/cancelar presença com lista fechada |
| `already-confirmed` | 409 | Membro já tem presença confirmada nesta partida |
| `referee-cannot-confirm` | 403 | REFEREE tentando confirmar como jogador (RN-ARB-002) |
| `member-already-banned` | 409 | Banir membro já banido |
| `member-not-banned` | 409 | Desbanir membro que não está banido |
| `match-in-past` | 400 | Criar partida com data no passado |
| `group-inactive` | 422 | Criar partida em grupo desativado |

---

## 9. Telas do Frontend

### 9.1 `/groups/[id]/matches` — Lista de Partidas

**Layout:** lista vertical com dois tabs: "Próximas" e "Passadas".

**Card de partida (upcoming):**
- Data e hora formatadas (`"Qui, 10 Jul · 20h00"`)
- Local da partida
- Contador: `8/14 confirmados` com barra de progresso
- Badge de status da lista: `"Lista aberta"` (success) ou `"Lista fechada"` (neutral)
- Badge do status do usuário: `"Confirmado"` / `"Recusado"` / `"Na fila (#2)"` / (sem badge se ainda não interagiu)
- Tap no card → `/groups/[id]/matches/[matchId]`

**Estado vazio (upcoming):**
```
Nenhuma partida agendada.
[Criar partida]   ← visível apenas para ADMIN/OWNER
```

**Ação primária (ADMIN/OWNER):** botão "Nova partida" no canto superior direito.

**Integração no painel do grupo:** no `/groups/[id]`, exibir card "Próxima partida" com dados da partida mais próxima e link para `/groups/[id]/matches`.

---

### 9.2 `/groups/[id]/matches/new` — Criar Partida

**Form fields:**
- Data e hora (`<input type="datetime-local">`, obrigatório, mínimo: agora + 1h)
- Local (`Input`, placeholder "Quadra do Parque", obrigatório, max 200 chars)
- Endereço (`Input`, placeholder "Av. Central, 100", opcional)
- Número máximo de jogadores (`Input type="number"`, mínimo 2, obrigatório)

**Submit:** `Button variant="primary" loading` → "Criar partida"

**Após sucesso:** redireciona para `/groups/[id]/matches/[matchId]`.

---

### 9.3 `/groups/[id]/matches/[matchId]` — Detalhe da Partida

**Header:**
- Data, hora e local da partida
- Badge: `"Agendada"` / `"Cancelada"` / `"Encerrada"`
- Badge lista: `"Lista aberta"` / `"Lista fechada"` com ícone de cadeado

**Área de ação do usuário (acima da lista):**

*Lista aberta + usuário não confirmado + não banido:*
```
[Confirmar presença]  [Recusar]
```

*Lista aberta + usuário confirmado:*
```
Você está confirmado ✓    [Cancelar presença]
```

*Lista aberta + usuário na fila (#N):*
```
Você está na fila (posição #N)    [Sair da fila]
```

*Usuário banido:*
```
⚠ Você está banido de presença neste grupo.
Motivo: "Ausências repetidas sem aviso"
Entre em contato com o organizador.
```

*Lista fechada:*
```
Lista encerrada — não é mais possível confirmar presença.
```

**Seção "Confirmados" (N/maxPlayers):**
- Lista de membros confirmados: nome, papel (badge), skill, posição
- ADMIN/OWNER vê botão "Remover" por membro (visível mesmo com lista fechada)

**Seção "Fila de espera" (se houver):**
- Membros em ordem de posição: `#1 Ana Lima`, `#2 Pedro Costa`
- ADMIN/OWNER vê botão "Confirmar" para promover da fila manualmente

**Seção "Recusados":**
- Lista de quem recusou (nome)

**Ações do ADMIN/OWNER (na parte inferior):**
- Se lista aberta: `[Fechar lista agora]` com confirmação
- Se lista aberta: `[Cancelar partida]` (abre dialog de confirmação)
- Se lista fechada + status SCHEDULED: `[Adicionar jogador]` (seleciona da lista de membros)

**Banir membro:** acessível via botão "⋮" em cada membro confirmado (ADMIN/OWNER) → abre drawer com opção "Banir de presença" (pede motivo).

---

## 10. Estratégia de Testes

### 10.1 Testes Unitários (JUnit 5 + Mockito)

| Classe | O que testar |
|---|---|
| `Match` | Factory (valida scheduledAt, calcula listClosesAt), `cancel()` (idempotência), `closePresenceList()`, `update()` bloqueado se fechado |
| `PresenceEntry` | Factory para CONFIRMED, DECLINED, BANNED_PENDING |
| `WaitingEntry` | Factory com posição sequencial |
| `MatchService` | Confirmar presença quando partida cheia → entra na fila; cancelar presença com membros na fila → promoção automática; REFEREE não confirma presença; ban verifica papel e registra histórico; fechar lista com scheduler; tentar confirmar com lista fechada → 422 |
| `PresenceListScheduler` | Chama `closePresenceList()` somente para partidas com `listClosesAt ≤ now` e `presenceListStatus = OPEN` |

### 10.2 Testes de Integração (Testcontainers + PostgreSQL)

| Cenário | Descrição |
|---|---|
| Fluxo completo | Criar partida → confirmar presenças até lotar → entrar na fila → cancelar confirmação → promoção da fila |
| Fechamento automático | Criar partida com `listClosesAt = now − 1min` → scheduler fecha lista |
| Tentativa pós-fechamento | Confirmar presença com lista fechada → 422 |
| Admin sobrescreve | Fechar lista → admin adiciona membro → 200 |
| REFEREE bloqueado | REFEREE tenta confirmar presença → 403 |
| Banimento | Banir membro → membro tenta confirmar → BANNED_PENDING; desbanir → consegue confirmar |
| Histórico de ban | Banir → desbanir → histórico tem 2 registros (BANNED + UNBANNED) |

### 10.3 Meta de Cobertura

- **≥ 80%** total (JaCoCo)
- Domain e use cases: **≥ 90%**

---

## 11. Migrations Flyway

As migrations para este módulo **já existem** no repositório:

| Arquivo | Tabelas | Observação |
|---|---|---|
| `V006__create_matches.sql` | `matches` | `list_closes_at` indexado com partial index (`WHERE presence_list_status = 'OPEN'`) para o scheduler |
| `V007__create_presence.sql` | `presence_entries`, `waiting_entries`, `presence_ban_history` | `uq_presence_entries_match_member` garante RN-PRE-007; `presence_ban_history` é imutável (sem UPDATE/DELETE) |
| `V008__create_referee_assignments.sql` | `referee_assignments` | Tabela criada mas **não usada nesta feature** — `RefereeAssignment` fica para Feature 5+ |

Nenhuma migration nova é necessária.

---

## 12. Ordem de Implementação

### Backend
1. Value Objects (`MatchStatus`, `PresenceListStatus`, `PresenceStatus`, `JustificationStatus`, `Location`)
2. Entidades de domínio (`PresenceEntry`, `WaitingEntry`, `Match` com factory e métodos)
3. `MatchRepository` (port/out) e `GroupMemberPort` (port/out) — interfaces
4. JPA Entities e adapters (`MatchJpaEntity`, `PresenceEntryJpaEntity`, `WaitingEntryJpaEntity`, `PresenceBanHistoryJpaEntity`, `MatchRepositoryAdapter`, `GroupMemberPortAdapter`)
5. `MatchService` — use cases de CRUD de partidas
6. `MatchService` — use cases de presença (confirmar, cancelar, fila, admin override)
7. `MatchService` — use cases de banimento
8. `PresenceListScheduler` com `@Scheduled`
9. DTOs e Controllers (`MatchController`, `PresenceController`)
10. Testes unitários (domain + service + scheduler)
11. Testes de integração (controllers)

### Frontend
12. Tipos e cliente de API (`Match`, `PresenceList`, `PresenceEntry`, `WaitingEntry` em `src/lib/api/matches.ts`)
13. TanStack Query hooks (`useMatches`, `useMatch`, `usePresenceList`, `useConfirmPresence`, etc.)
14. Tela `/groups/[id]/matches` (lista de partidas com tabs)
15. Tela `/groups/[id]/matches/new` (criar partida)
16. Tela `/groups/[id]/matches/[matchId]` (detalhe + presença + ações)
17. Integração do card "Próxima partida" no `/groups/[id]`
18. Validação do fluxo completo no staging
