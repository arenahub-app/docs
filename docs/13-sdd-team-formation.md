# SDD — Módulo de Formação de Times (Etapa 5, Feature 3)

**Versão:** 1.0  
**Data:** 2026-06-17  
**Status:** Em revisão  
**Refs:** UC-016, RN-TIM-001–006, BC-05

---

## 1. Objetivo

Implementar o módulo de Formação de Times do ArenaHub, cobrindo:

1. Geração automática de times via algoritmo snake draft balanceado por skill
2. Ajuste manual de times pelo admin (mover jogador entre times)
3. Regeneração de formação (substitui a anterior)
4. Visualização dos times por todos os membros do grupo

Este módulo depende diretamente de BC-03 (lista de presença fechada) e BC-02 (skill e posição dos membros).

---

## 2. Escopo

**Incluído neste módulo:**
- Aggregate `TeamFormation` com entidades `Team` e `TeamPlayer`
- 4 endpoints REST (`/api/v1/groups/{groupId}/matches/{matchId}/team-formations/**`)
- Domain Service `SnakeDraftService` com algoritmo de distribuição e equalização
- Port de saída `MatchPresencePort` para leitura cross-BC de jogadores confirmados
- 2 telas no frontend
- Migrations já existentes (V010)

**Fora do escopo (módulos futuros):**
- Notificação aos jogadores ao confirmar formação (Feature 4.7 — Mensageria Assíncrona)
- Verificação de cobertura de posições por modalidade (requer regras por esporte — deferido)
- `RefereeAssignment` (Feature 5+ — Financial)
- Histórico de versões de formação (apenas a última é mantida por partida)

---

## 3. Arquitetura em Camadas

```
presentation/teamformation/
├── TeamFormationController.java          ← /api/v1/groups/{groupId}/matches/{matchId}/team-formations/**
└── dto/
    ├── GenerateFormationRequest.java
    ├── MovePlayerRequest.java
    ├── TeamFormationResponse.java
    ├── TeamResponse.java
    └── TeamPlayerResponse.java

application/teamformation/
├── port/in/
│   ├── GenerateTeamFormationUseCase.java
│   ├── GetCurrentFormationUseCase.java
│   ├── MovePlayerUseCase.java
│   └── DeleteFormationUseCase.java
├── port/out/
│   ├── TeamFormationRepository.java
│   └── MatchPresencePort.java            ← lê jogadores confirmados do BC-03
└── TeamFormationService.java             ← implementa todos os use cases acima

domain/teamformation/
├── TeamFormation.java                    ← AR
├── Team.java
├── TeamPlayer.java
├── service/
│   └── SnakeDraftService.java            ← algoritmo snake draft + equalização
└── vo/
    └── FormationType.java                (AUTOMATIC | MANUAL_ADJUSTED)

infrastructure/persistence/teamformation/
├── TeamFormationJpaEntity.java
├── TeamJpaEntity.java
├── TeamPlayerJpaEntity.java
├── TeamFormationJpaRepository.java
├── TeamJpaRepository.java
├── TeamPlayerJpaRepository.java
└── TeamFormationRepositoryAdapter.java   ← implementa TeamFormationRepository + MatchPresencePort
```

---

## 4. Modelo de Domínio

### 4.1 TeamFormation (Aggregate Root)

```java
public class TeamFormation {
    private UUID id;
    private UUID matchId;
    private UUID groupId;
    private int numberOfTeams;
    private FormationType formationType;   // AUTOMATIC | MANUAL_ADJUSTED
    private UUID confirmedBy;
    private Instant confirmedAt;
    private List<Team> teams;

    public static TeamFormation create(UUID matchId, UUID groupId,
                                       int numberOfTeams, UUID confirmedBy,
                                       List<Team> teams) {
        // Valida numberOfTeams >= 2
        // formationType = AUTOMATIC
        // confirmedAt = now()
    }

    public void movePlayer(UUID memberId, UUID fromTeamId, UUID toTeamId) {
        // Remove TeamPlayer do fromTeam
        // Adiciona TeamPlayer no toTeam
        // Recalcula averageSkill nos dois times afetados
        // formationType = MANUAL_ADJUSTED
    }

    public double skillStdDev() {
        // Desvio padrão das averageSkill dos times
    }
}
```

### 4.2 Team (Entity dentro de TeamFormation)

```java
public class Team {
    private UUID id;
    private UUID formationId;
    private UUID groupId;
    private String name;                  // "Time A", "Time B", ...
    private BigDecimal averageSkill;      // Calculado e mantido sincronizado
    private List<TeamPlayer> players;

    public void addPlayer(TeamPlayer player) {
        // Adiciona e recalcula averageSkill
    }

    public void removePlayer(UUID memberId) {
        // Remove e recalcula averageSkill
        // Lança exceção se membro não encontrado
    }

    private void recalculateAverageSkill() {
        // Média simples dos skills dos players
        // Se players vazio: averageSkill = 0.0
    }
}
```

### 4.3 TeamPlayer (Entity dentro de Team)

```java
public class TeamPlayer {
    private UUID id;
    private UUID teamId;
    private UUID groupId;
    private UUID memberId;
    private String position;              // Nullable: posição preferencial do GroupMember
}
```

### 4.4 Value Objects

| VO | Tipo Base | Validação |
|---|---|---|
| `FormationType` | Enum | `AUTOMATIC`, `MANUAL_ADJUSTED` |

### 4.5 MatchPresencePort (leitura cross-BC)

Port de saída para buscar os jogadores confirmados da partida sem acoplar ao BC-03:

```java
public interface MatchPresencePort {
    MatchPresenceSnapshot getPresenceSnapshot(UUID matchId, UUID groupId);
}

public record MatchPresenceSnapshot(
    boolean presenceListClosed,
    List<PlayerSnapshot> confirmedPlayers
) {}

public record PlayerSnapshot(
    UUID memberId,
    String userName,
    BigDecimal skill,       // GroupMember.skill
    String position,        // GroupMember.position (nullable)
    String role             // GroupRole: OWNER | ADMIN | PLAYER | REFEREE
) {}
```

A implementação (`TeamFormationRepositoryAdapter`) faz join entre `presence_entries` (status = `CONFIRMED`) e `group_members` para montar o snapshot. Árbitros (`role = REFEREE`) são incluídos no resultado mas **excluídos** pelo caso de uso antes de passar ao `SnakeDraftService`.

### 4.6 SnakeDraftService (Domain Service)

```java
@Component
public class SnakeDraftService {

    private static final BigDecimal STD_DEV_THRESHOLD = new BigDecimal("0.5");

    public List<Team> distribute(List<PlayerSnapshot> players,
                                 int numberOfTeams, UUID groupId) {
        // 1. Inicializa numberOfTeams times: "Time A", "Time B", ...
        // 2. Ordena players por skill DESC
        // 3. Snake draft:
        //      rodada par:  atribui da esquerda para a direita (time 0, 1, 2, ...)
        //      rodada ímpar: atribui da direita para a esquerda (..., 2, 1, 0)
        // 4. Calcula averageSkill de cada time
        // 5. Se skillStdDev() > 0.5: tenta pares de troca entre times para reduzir stdDev
        //      (best-effort: troca somente se melhora; não falha se threshold não atingido)
        // 6. Retorna lista de Times com players distribuídos
    }

    private double calcStdDev(List<Team> teams) { ... }

    private void trySwapsToEqualize(List<Team> teams) {
        // Itera pares de times; para cada par, tenta todas as trocas 1×1
        // Aceita a troca que mais reduz o stdDev
        // Repete até nenhuma troca melhorar ou limite de iterações (50)
    }
}
```

**Nomenclatura dos times:** alfabética sequencial — "Time A", "Time B", "Time C", etc.

---

## 5. Endpoints da API

### 5.1 Gerar Formação

```
POST /api/v1/groups/{groupId}/matches/{matchId}/team-formations
Authorization: Bearer <token>
Content-Type: application/json

{
  "numberOfTeams": 2
}
```

**Autorização:** `OWNER` ou `ADMIN`

**Fluxo:**
1. Valida que o grupo existe e o caller é OWNER/ADMIN
2. Carrega `MatchPresenceSnapshot` via `MatchPresencePort`
3. Valida `presenceListClosed = true` (RN-TIM-001)
4. Filtra árbitros da lista de jogadores (RN-TIM-002)
5. Valida `confirmedNonRefereeCount >= numberOfTeams` (RN-TIM-003)
6. Valida `numberOfTeams >= 2` (RN-TIM-004)
7. Se já existe formação para esta partida → deleta a anterior (times + players em cascata)
8. Chama `SnakeDraftService.distribute(players, numberOfTeams, groupId)`
9. Persiste `TeamFormation` + `Team`s + `TeamPlayer`s
10. Retorna `201 Created` com `TeamFormationResponse`

**Respostas:**

| Status | Condição |
|---|---|
| 201 | Sucesso |
| 400 | `numberOfTeams` inválido |
| 403 | Papel insuficiente |
| 404 | Partida ou grupo não encontrado |
| 422 | `presence-list-not-closed` — lista ainda aberta |
| 422 | `not-enough-players` — menos jogadores do que times |

---

### 5.2 Obter Formação Atual

```
GET /api/v1/groups/{groupId}/matches/{matchId}/team-formations/current
Authorization: Bearer <token>
```

**Autorização:** qualquer membro do grupo

**Response:**
```json
{
  "id": "uuid",
  "matchId": "uuid",
  "numberOfTeams": 2,
  "formationType": "AUTOMATIC",
  "confirmedBy": "uuid",
  "confirmedAt": "2026-07-10T20:30:00Z",
  "teams": [
    {
      "id": "uuid",
      "name": "Time A",
      "averageSkill": 4.2,
      "playerCount": 5,
      "players": [
        {
          "memberId": "uuid",
          "userName": "Carlos Drummond",
          "skill": 5.0,
          "position": "FORWARD"
        }
      ]
    },
    {
      "id": "uuid",
      "name": "Time B",
      "averageSkill": 4.1,
      "playerCount": 5,
      "players": [...]
    }
  ]
}
```

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso |
| 403 | Não é membro |
| 404 | Formação não encontrada (ainda não foi gerada) |
| 404 | Partida ou grupo não encontrado |

---

### 5.3 Mover Jogador entre Times

```
PUT /api/v1/groups/{groupId}/matches/{matchId}/team-formations/{formationId}/move
Authorization: Bearer <token>
Content-Type: application/json

{
  "memberId": "uuid",
  "toTeamId": "uuid"
}
```

**Autorização:** `OWNER` ou `ADMIN`

**Fluxo:**
1. Valida que a formação existe e pertence à partida/grupo
2. Localiza o time de origem do membro (busca em todos os times da formação)
3. Valida que `fromTeamId ≠ toTeamId`
4. Chama `teamFormation.movePlayer(memberId, fromTeamId, toTeamId)`
5. Persiste alterações (`team_players` + `average_skill` nos dois times)
6. Retorna `200` com `TeamFormationResponse` atualizado

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso com TeamFormationResponse atualizado |
| 400 | `toTeamId` igual ao time atual do jogador |
| 403 | Papel insuficiente |
| 404 | Formação, membro ou time destino não encontrado |

---

### 5.4 Deletar Formação

```
DELETE /api/v1/groups/{groupId}/matches/{matchId}/team-formations/{formationId}
Authorization: Bearer <token>
```

**Autorização:** `OWNER` ou `ADMIN`

**Fluxo:**
1. Valida que a formação existe e pertence à partida/grupo
2. Deleta `team_players` → `teams` → `team_formations` (cascata lógica na aplicação)
3. Retorna `204 No Content`

> Admin pode gerar nova formação após deletar chamando `POST .../team-formations` novamente.

**Respostas:**

| Status | Condição |
|---|---|
| 204 | Sucesso |
| 403 | Papel insuficiente |
| 404 | Formação não encontrada |

---

## 6. Regras de Negócio

| Código | Regra |
|---|---|
| RN-TIM-001 | Formação só é possível após `presenceListStatus = CLOSED` |
| RN-TIM-002 | Membros com `role = REFEREE` não participam de nenhum time |
| RN-TIM-003 | `numberOfTeams` deve ser ≥ 2 |
| RN-TIM-004 | Número de jogadores confirmados (sem árbitros) deve ser ≥ `numberOfTeams` |
| RN-TIM-005 | O algoritmo tenta manter desvio padrão de `averageSkill` entre times ≤ 0.5; não é garantido quando o grupo de jogadores não permite balanceamento perfeito |
| RN-TIM-006 | Existe no máximo uma formação ativa por partida; gerar nova substitui a existente |

---

## 7. Matriz de Autorização

| Operação | OWNER | ADMIN | PLAYER | REFEREE |
|---|---|---|---|---|
| Gerar formação | ✓ | ✓ | ✗ | ✗ |
| Ver formação atual | ✓ | ✓ | ✓ | ✓ |
| Mover jogador entre times | ✓ | ✓ | ✗ | ✗ |
| Deletar formação | ✓ | ✓ | ✗ | ✗ |

---

## 8. Tratamento de Erros

```json
{
  "type": "/errors/presence-list-not-closed",
  "title": "Lista de presença ainda aberta",
  "status": 422,
  "detail": "Não é possível gerar times enquanto a lista de presença estiver aberta.",
  "instance": "/api/v1/groups/uuid/matches/uuid/team-formations"
}
```

| Código de Negócio | HTTP | Cenário |
|---|---|---|
| `presence-list-not-closed` | 422 | Tentar gerar formação com lista ainda aberta |
| `not-enough-players` | 422 | Menos jogadores confirmados do que times solicitados |
| `formation-not-found` | 404 | Formação não existe para esta partida |
| `player-not-in-formation` | 404 | `memberId` não encontrado em nenhum time da formação |
| `team-not-in-formation` | 404 | `toTeamId` não pertence à formação |
| `player-already-in-team` | 400 | Tentativa de mover jogador para o time em que já está |

---

## 9. Telas do Frontend

### 9.1 `/groups/[id]/matches/[matchId]/teams` — Formação de Times

**Acesso:** botão "Ver Times" / "Formar Times" na tela de detalhes da partida (`/groups/[id]/matches/[matchId]`), visível somente se `presenceListStatus = CLOSED`.

**Estado: sem formação (ADMIN/OWNER):**
```
Nenhuma formação gerada ainda.

Número de times: [2] [−] [+]

[Gerar Times Automaticamente]
```

**Estado: sem formação (PLAYER/REFEREE):**
```
Os times ainda não foram formados.
Aguarde o organizador gerar a formação.
```

**Estado: com formação gerada:**

Header:
- Badge `"Geração automática"` (se `AUTOMATIC`) ou `"Ajustado manualmente"` (se `MANUAL_ADJUSTED`)
- Confirmado por: nome do admin · data/hora

Cards lado a lado (responsivo: coluna em mobile, linha em desktop):

```
┌─────────────────────┐   ┌─────────────────────┐
│ Time A              │   │ Time B              │
│ Skill médio: 4.2 ★  │   │ Skill médio: 4.1 ★  │
├─────────────────────┤   ├─────────────────────┤
│ Carlos D.     5.0 ★ │   │ Ana Lima      4.5 ★ │
│ Pedro C.      4.5 ★ │   │ João Silva    4.0 ★ │
│ Maria S.      3.5 ★ │   │ Lucas F.      3.8 ★ │
│  ...                │   │  ...                │
└─────────────────────┘   └─────────────────────┘
```

- Cada jogador exibe: nome, skill (estrelas), posição (se definida)
- Badge de desequilíbrio: se `|averageSkillA − averageSkillB| > 0.5`, exibe aviso `"⚠ Times desequilibrados"`

**Ações do ADMIN/OWNER:**

- Botão "Mover" em cada jogador → abre sheet com seletor de time destino:
  ```
  Mover Carlos D. para:
  ○ Time B
  [Confirmar]
  ```
- Botão "Regenerar Times" → confirma (modal) → chama `POST .../team-formations` novamente
- Botão "Desfazer Formação" → confirma (modal) → chama `DELETE .../team-formations/{id}`

---

### 9.2 Card "Times" na tela de detalhes da partida

Na tela `/groups/[id]/matches/[matchId]`, após a seção de presença:

**Se lista fechada + formação existe:**
```
Times formados ✓
[Ver Times →]
```

**Se lista fechada + sem formação (ADMIN/OWNER):**
```
Lista encerrada. Pronto para formar times.
[Formar Times →]
```

**Se lista aberta ou em andamento:**
*(Seção não exibida)*

---

## 10. Estratégia de Testes

### 10.1 Testes Unitários (JUnit 5 + Mockito)

| Classe | O que testar |
|---|---|
| `SnakeDraftService` | Distribuição snake com 10 jogadores em 2 times (verificar ordem); 9 jogadores em 3 times; equalização quando stdDev > 0.5; caso onde equalização perfeita é impossível (não falha, retorna melhor esforço) |
| `TeamFormation` | Factory valida `numberOfTeams >= 2`; `movePlayer()` recalcula averageSkill nos dois times; `movePlayer()` lança exceção se membro não encontrado; `movePlayer()` atualiza `formationType = MANUAL_ADJUSTED` |
| `Team` | `recalculateAverageSkill()` com lista vazia retorna 0; média correta com múltiplos players |
| `TeamFormationService` | Lista não fechada → 422; árbitros excluídos da distribuição; jogadores insuficientes → 422; deleta formação anterior antes de criar nova |

### 10.2 Testes de Integração (Testcontainers + PostgreSQL)

| Cenário | Descrição |
|---|---|
| Fluxo completo | Criar partida → confirmar presenças (misto de PLAYERs e REFEREEs) → fechar lista → gerar formação → verificar que árbitros não foram incluídos |
| Mover jogador | Gerar formação → mover jogador → verificar averageSkill recalculado e formationType = MANUAL_ADJUSTED |
| Regenerar | Gerar formação → regenerar → verificar que formação anterior foi removida |
| Listar fechada abre | Tentar gerar com lista aberta → 422 |
| Jogadores insuficientes | Fechar lista com 1 jogador → tentar gerar 2 times → 422 |
| PLAYER tenta gerar | PLAYER tenta POST → 403 |
| Ver sem formação | GET current sem formação gerada → 404 |

### 10.3 Meta de Cobertura

- **≥ 80%** total (JaCoCo)
- `SnakeDraftService` e domínio: **≥ 90%**

---

## 11. Migrations Flyway

As migrations para este módulo **já existem** no repositório:

| Arquivo | Tabelas | Observação |
|---|---|---|
| `V010__create_team_formations.sql` | `team_formations`, `teams`, `team_players` | Nenhuma alteração necessária |

Nenhuma migration nova é necessária.

---

## 12. Ordem de Implementação

### Backend
1. Value Object `FormationType` (enum `AUTOMATIC`, `MANUAL_ADJUSTED`)
2. Entidades de domínio (`TeamPlayer`, `Team` com `addPlayer`/`removePlayer`/`recalculateAverageSkill`, `TeamFormation` com factory e `movePlayer`)
3. Domain Service `SnakeDraftService` (snake draft + equalização por troca)
4. Port/out `TeamFormationRepository` e `MatchPresencePort` — interfaces
5. JPA Entities (`TeamFormationJpaEntity`, `TeamJpaEntity`, `TeamPlayerJpaEntity`)
6. `TeamFormationRepositoryAdapter` (implementa `TeamFormationRepository` + `MatchPresencePort` com join em `presence_entries` + `group_members`)
7. Ports de entrada (`GenerateTeamFormationUseCase`, `GetCurrentFormationUseCase`, `MovePlayerUseCase`, `DeleteFormationUseCase`)
8. `TeamFormationService` — implementação dos use cases
9. DTOs e `TeamFormationController`
10. Testes unitários (domain + SnakeDraftService + service)
11. Testes de integração (controller)

### Frontend
12. Tipos e cliente de API (`TeamFormation`, `Team`, `TeamPlayer` em `src/lib/api/team-formations.ts`)
13. TanStack Query hooks (`useCurrentFormation`, `useGenerateFormation`, `useMovePlayer`, `useDeleteFormation`)
14. Tela `/groups/[id]/matches/[matchId]/teams`
15. Integração do card "Times" na tela de detalhes da partida
16. Validação do fluxo completo no staging
