# SDD — Módulo de Gestão de Grupos (Etapa 5, Feature 1)

**Versão:** 1.0  
**Data:** 2026-06-15  
**Status:** Aprovado  
**Refs:** US-006, US-007, US-008, US-009, RN-GRP-001–004, RN-MBR-001–005, RN-SKL-002, BC-02

---

## 1. Objetivo

Implementar o módulo de Gestão de Grupos do ArenaHub, cobrindo:

1. Criação e configuração de grupos esportivos
2. Listagem e visualização de grupos do usuário
3. Geração e uso de links de convite (join by invite)
4. Gerenciamento de membros: listar, alterar papel/skill/posição, remover
5. Desativação de grupo (soft delete)

Este módulo é a fundação de todos os outros: Match, Skill Voting, Team Formation e Payments
dependem da existência de grupos e membros.

---

## 2. Escopo

**Incluído neste módulo:**
- Aggregate `Group` com entidades `GroupMember` e `GroupInvite`
- 13 endpoints REST (`/api/v1/groups/**` e `/api/v1/invites/**`)
- Migrations já existentes (V002–V004)
- Autorização por papel dentro do grupo
- 5 telas no frontend

**Fora do escopo (módulos futuros):**
- `RefereeProfile` (cadastrado junto com Árbitros no módulo Match)
- Upload de foto do grupo via R2 (módulo de Storage)
- Notificações ao remover membro (módulo de Mensageria Assíncrona)
- Banimento de presença (gerenciado pelo módulo Match & Presence)
- Transferência de OWNER (coberta por `PATCH /members/{id}` com `role: OWNER`)

---

## 3. Arquitetura em Camadas

```
presentation/group/
├── GroupController.java         ← /api/v1/groups/**
├── InviteController.java        ← /api/v1/invites/**
└── dto/
    ├── CreateGroupRequest.java
    ├── UpdateGroupRequest.java
    ├── GroupResponse.java        ← detalhes completos
    ├── GroupSummaryResponse.java ← listagem
    ├── MemberResponse.java
    ├── UpdateMemberRequest.java
    ├── CreateInviteResponse.java
    ├── InviteResponse.java
    ├── InvitePreviewResponse.java
    └── JoinGroupResponse.java

application/group/
├── port/in/
│   ├── CreateGroupUseCase.java
│   ├── GetGroupUseCase.java
│   ├── ListGroupsUseCase.java
│   ├── UpdateGroupUseCase.java
│   ├── DeactivateGroupUseCase.java
│   ├── ListMembersUseCase.java
│   ├── UpdateMemberUseCase.java
│   ├── RemoveMemberUseCase.java
│   ├── GenerateInviteUseCase.java
│   ├── ListInvitesUseCase.java
│   ├── DeactivateInviteUseCase.java
│   ├── GetInvitePreviewUseCase.java
│   └── JoinByInviteUseCase.java
├── port/out/
│   └── GroupRepository.java
└── service/
    └── GroupService.java        ← implementa todos os use cases acima

domain/group/
├── Group.java                   ← AR
├── GroupMember.java
├── GroupInvite.java
└── vo/
    ├── GroupName.java
    ├── Sport.java
    ├── GroupRole.java
    ├── GroupStatus.java
    ├── Skill.java
    ├── SkillSource.java
    ├── PlayerPosition.java
    └── InviteToken.java

infrastructure/persistence/group/
├── GroupJpaEntity.java
├── GroupMemberJpaEntity.java
├── GroupInviteJpaEntity.java
├── GroupJpaRepository.java        ← Spring Data JPA
├── GroupMemberJpaRepository.java
├── GroupInviteJpaRepository.java
└── GroupRepositoryAdapter.java    ← implementa GroupRepository (port/out)
```

---

## 4. Modelo de Domínio

### 4.1 Group (Aggregate Root)

```java
public class Group {
    private UUID id;
    private GroupName name;          // 3–80 chars
    private Sport sport;             // enum
    private String description;      // até 500 chars, opcional
    private String photoUrl;         // URL R2, opcional
    private String pixKey;           // chave Pix do grupo, opcional
    private GroupStatus status;      // ACTIVE | INACTIVE
    private List<GroupMember> members;
    private List<GroupInvite> invites;
    private Instant createdAt;
    private Instant updatedAt;

    // Factory — criador entra automaticamente como OWNER
    public static Group create(GroupName name, Sport sport, String description, UUID creatorId) {
        Group g = new Group(...);
        g.members.add(GroupMember.create(creatorId, GroupRole.OWNER));
        return g;
    }

    // Adiciona membro via convite (role fixo = PLAYER)
    public GroupMember addMember(UUID userId) { ... }

    // Atualiza dados do membro (role, skill, posição)
    public void updateMember(UUID memberId, UpdateMemberCommand cmd, GroupRole callerRole) { ... }

    // Remove membro — invariante: não pode remover se for o único OWNER
    public void removeMember(UUID memberId) { ... }

    // Gera convite — expira em 7 dias, máx 50 usos (RN-GRP-004)
    public GroupInvite generateInvite(UUID createdBy) { ... }

    // Desativa convite
    public void deactivateInvite(UUID inviteId) { ... }

    // Soft delete (RN-GRP-003)
    public void deactivate() { this.status = GroupStatus.INACTIVE; }

    // Invariante: exatamente 1 OWNER a qualquer momento (RN-GRP-001)
    private void ensureSingleOwner() { ... }
}
```

### 4.2 GroupMember (Entity dentro de Group)

```java
public class GroupMember {
    private UUID id;
    private UUID userId;
    private GroupRole role;          // OWNER | ADMIN | PLAYER | REFEREE
    private Skill skill;             // 1.0–6.0; default 3.0
    private SkillSource skillSource; // DEFAULT | VOTING | MANUAL
    private PlayerPosition position; // enum opcional
    private boolean isSubscriber;
    private BigDecimal subscriptionAmount; // obrigatório se isSubscriber=true
    private boolean presenceBanned;
    private String presenceBanReason;
    private Instant joinedAt;
    private Instant updatedAt;

    public static GroupMember create(UUID userId, GroupRole role) {
        // skill=3.0, skillSource=DEFAULT (RN-SKL-002)
    }
}
```

### 4.3 GroupInvite (Entity dentro de Group)

```java
public class GroupInvite {
    private UUID id;
    private UUID groupId;
    private UUID createdBy;
    private InviteToken token;      // UUID v4 aleatório
    private int usageCount;
    private int maxUsages;          // 50
    private boolean active;
    private Instant expiresAt;      // now + 7 dias
    private Instant createdAt;

    public boolean isValid() {
        return active
            && usageCount < maxUsages
            && Instant.now().isBefore(expiresAt);
    }

    public void incrementUsage() { this.usageCount++; }
    public void deactivate() { this.active = false; }
}
```

### 4.4 Value Objects

| VO | Validação |
|---|---|
| `GroupName` | 3–80 caracteres, não vazio |
| `Sport` | Enum: `FOOTBALL`, `VOLLEYBALL`, `BASKETBALL`, `FUTEVOLEI`, `BEACH_TENNIS`, `OTHER` |
| `GroupRole` | Enum: `OWNER`, `ADMIN`, `PLAYER`, `REFEREE` |
| `GroupStatus` | Enum: `ACTIVE`, `INACTIVE` |
| `Skill` | BigDecimal 1.0–6.0, 1 casa decimal |
| `SkillSource` | Enum: `DEFAULT`, `VOTING`, `MANUAL` |
| `PlayerPosition` | Enum com posições por modalidade + `OTHER` |
| `InviteToken` | UUID v4 como String, gerado aleatoriamente |

---

## 5. Endpoints da API

### 5.1 Criar Grupo

```
POST /api/v1/groups
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Pelada do Bairro",
  "sport": "FOOTBALL",
  "description": "Futebol toda quinta às 20h"
}
```

**Fluxo:**
1. Valida `name` (3–80 chars) e `sport` (enum válido)
2. Cria `Group.create(name, sport, description, userId)`
3. Criador já entra como `OWNER` com `skill=3.0`, `skillSource=DEFAULT` (RN-SKL-002)
4. Persiste grupo
5. Gera automaticamente um convite inicial (ativo, 7 dias, 50 usos)
6. Retorna `201 Created` com `GroupResponse`

**Respostas:**

| Status | Condição |
|---|---|
| 201 | Sucesso |
| 400 | Campos inválidos |
| 401 | Não autenticado |

---

### 5.2 Listar Grupos do Usuário

```
GET /api/v1/groups
Authorization: Bearer <token>
```

**Fluxo:**
1. Busca todos os `GroupMember` onde `userId = currentUser`
2. Retorna grupos `ACTIVE` e `INACTIVE` (usuário ainda acessa histórico)
3. Ordena por `joinedAt DESC`

**Response:**
```json
[
  {
    "id": "uuid",
    "name": "Pelada do Bairro",
    "sport": "FOOTBALL",
    "photoUrl": null,
    "status": "ACTIVE",
    "memberCount": 12,
    "myRole": "OWNER"
  }
]
```

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso (lista vazia se sem grupos) |
| 401 | Não autenticado |

---

### 5.3 Detalhes do Grupo

```
GET /api/v1/groups/{id}
Authorization: Bearer <token>
```

**Fluxo:**
1. Verifica que o grupo existe
2. Verifica que `currentUser` é membro (qualquer papel)
3. Retorna grupo com lista completa de membros

**Response:**
```json
{
  "id": "uuid",
  "name": "Pelada do Bairro",
  "sport": "FOOTBALL",
  "description": "Futebol toda quinta às 20h",
  "photoUrl": null,
  "pixKey": "pelada@pix.com",
  "status": "ACTIVE",
  "myRole": "OWNER",
  "members": [
    {
      "id": "uuid",
      "userId": "uuid",
      "name": "Carlos Drummond",
      "photoUrl": null,
      "role": "OWNER",
      "skill": 5.2,
      "skillSource": "VOTING",
      "position": "FORWARD",
      "isSubscriber": false,
      "presenceBanned": false,
      "joinedAt": "2026-06-01T00:00:00Z"
    }
  ],
  "createdAt": "2026-06-01T00:00:00Z"
}
```

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso |
| 401 | Não autenticado |
| 403 | Usuário não é membro do grupo |
| 404 | Grupo não encontrado |

---

### 5.4 Atualizar Grupo

```
PATCH /api/v1/groups/{id}
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "Pelada do Parque",
  "description": "Mudamos de local!",
  "pixKey": "novo@pix.com"
}
```

**Autorização:** `OWNER` ou `ADMIN`

**Regras:**
- Campos omitidos não são alterados (PATCH semântico)
- `sport` pode ser incluído, mas falha com `422` se já existir TeamFormation para o grupo (RN-GRP-002)
- Grupo `INACTIVE` não pode ser editado

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso com GroupResponse atualizado |
| 400 | Campos inválidos |
| 403 | Papel insuficiente ou grupo inativo |
| 404 | Grupo não encontrado |
| 422 | `sport-change-not-allowed` — já existe formação de times |

---

### 5.5 Desativar Grupo

```
DELETE /api/v1/groups/{id}
Authorization: Bearer <token>
```

**Autorização:** somente `OWNER`

**Fluxo:**
1. Define `status = INACTIVE` (soft delete — RN-GRP-003)
2. Novas partidas ficam bloqueadas; histórico preservado
3. Convites ativos são desativados automaticamente

**Respostas:**

| Status | Condição |
|---|---|
| 204 | Sucesso |
| 403 | Não é OWNER |
| 404 | Grupo não encontrado |

---

### 5.6 Listar Membros

```
GET /api/v1/groups/{id}/members
Authorization: Bearer <token>
```

**Autorização:** qualquer membro do grupo

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Lista de MemberResponse |
| 403 | Não é membro |
| 404 | Grupo não encontrado |

---

### 5.7 Atualizar Membro

```
PATCH /api/v1/groups/{id}/members/{memberId}
Authorization: Bearer <token>
Content-Type: application/json

{
  "role": "ADMIN",
  "skill": 4.5,
  "position": "MIDFIELDER"
}
```

**Autorização e regras por campo:**

| Campo | Quem pode alterar | Regras |
|---|---|---|
| `role` | Somente `OWNER` | Não pode rebaixar o próprio OWNER (RN-MBR-001); para transferir OWNER, role=OWNER no alvo e OWNER atual vira ADMIN (RN-MBR-002) |
| `skill` | `OWNER` ou `ADMIN` | 1.0–6.0 (RN-SKL-001); registra `skillSource=MANUAL` (RN-SKL-006) |
| `position` | `OWNER` ou `ADMIN` | Enum válido ou null |
| `isSubscriber` | `OWNER` ou `ADMIN` | Se true, `subscriptionAmount` obrigatório |
| `subscriptionAmount` | `OWNER` ou `ADMIN` | Decimal positivo |

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso com MemberResponse |
| 400 | Campos inválidos |
| 403 | Papel insuficiente |
| 404 | Grupo ou membro não encontrado |
| 422 | `cannot-demote-owner` — tentativa de rebaixar OWNER sem transferência |

---

### 5.8 Remover Membro

```
DELETE /api/v1/groups/{id}/members/{memberId}
Authorization: Bearer <token>
```

**Autorização:** `OWNER` ou `ADMIN` (ADMIN não pode remover OWNER)

**Regras:**
- Membro removido perde acesso imediato (RN-MBR-003)
- Histórico de pagamentos e presenças é preservado
- OWNER não pode remover a si mesmo (impediria ter grupo sem OWNER)

**Respostas:**

| Status | Condição |
|---|---|
| 204 | Sucesso |
| 403 | Papel insuficiente (ex.: ADMIN tentando remover OWNER) |
| 404 | Grupo ou membro não encontrado |
| 422 | Tentativa de remover único OWNER |

---

### 5.9 Gerar Convite

```
POST /api/v1/groups/{id}/invites
Authorization: Bearer <token>
```

**Autorização:** `OWNER` ou `ADMIN`

**Fluxo:**
1. Cria `GroupInvite` com token UUID aleatório, `expiresAt = now + 7d`, `maxUsages = 50`
2. Persiste e retorna link completo

**Response:**
```json
{
  "id": "uuid",
  "token": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "link": "https://arenahub.app/invite/f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "expiresAt": "2026-06-22T20:00:00Z",
  "maxUsages": 50,
  "usageCount": 0,
  "active": true
}
```

**Respostas:**

| Status | Condição |
|---|---|
| 201 | Sucesso |
| 403 | Papel insuficiente |
| 404 | Grupo não encontrado |

---

### 5.10 Listar Convites

```
GET /api/v1/groups/{id}/invites
Authorization: Bearer <token>
```

**Autorização:** `OWNER` ou `ADMIN`

Retorna apenas convites `active = true` e ainda não expirados.

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Lista de InviteResponse |
| 403 | Papel insuficiente |
| 404 | Grupo não encontrado |

---

### 5.11 Desativar Convite

```
DELETE /api/v1/groups/{id}/invites/{inviteId}
Authorization: Bearer <token>
```

**Autorização:** `OWNER` ou `ADMIN`

**Respostas:**

| Status | Condição |
|---|---|
| 204 | Sucesso |
| 403 | Papel insuficiente |
| 404 | Convite não encontrado |

---

### 5.12 Preview de Convite (Público)

```
GET /api/v1/invites/{token}
```

**Autorização:** público (sem autenticação)

**Fluxo:**
1. Busca convite pelo token
2. Retorna preview mesmo se expirado/inativo (campo `valid` indica estado)

**Response:**
```json
{
  "groupId": "uuid",
  "groupName": "Pelada do Bairro",
  "sport": "FOOTBALL",
  "groupPhotoUrl": null,
  "memberCount": 12,
  "valid": true,
  "expiresAt": "2026-06-22T20:00:00Z"
}
```

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso (valid=false se expirado/desativado) |
| 404 | Token não existe |

---

### 5.13 Aceitar Convite (Entrar no Grupo)

```
POST /api/v1/invites/{token}/join
Authorization: Bearer <token>
```

**Fluxo:**
1. Busca convite pelo token
2. Valida: `active = true`, `usageCount < maxUsages`, `expiresAt` no futuro
3. Verifica que o usuário ainda não é membro do grupo
4. Cria `GroupMember` com `role = PLAYER`, `skill = 3.0`, `skillSource = DEFAULT` (RN-SKL-002)
5. Incrementa `usageCount` do convite
6. Retorna `GroupSummaryResponse` do grupo recém-entrado

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso com GroupSummaryResponse |
| 401 | Não autenticado |
| 409 | `user-already-member` |
| 410 | `invite-expired` — expirado ou esgotado |
| 404 | Token não existe |

---

## 6. Matriz de Autorização

| Operação | OWNER | ADMIN | PLAYER | REFEREE |
|---|---|---|---|---|
| Ver detalhes do grupo | ✓ | ✓ | ✓ | ✓ |
| Atualizar grupo (nome, desc, pixKey) | ✓ | ✓ | ✗ | ✗ |
| Desativar grupo | ✓ | ✗ | ✗ | ✗ |
| Listar membros | ✓ | ✓ | ✓ | ✓ |
| Alterar `role` de membro | ✓ | ✗ | ✗ | ✗ |
| Alterar `skill`/`position` de membro | ✓ | ✓ | ✗ | ✗ |
| Remover membro | ✓ | ✓* | ✗ | ✗ |
| Gerar convite | ✓ | ✓ | ✗ | ✗ |
| Listar convites | ✓ | ✓ | ✗ | ✗ |
| Desativar convite | ✓ | ✓ | ✗ | ✗ |

*ADMIN não pode remover OWNER.

**Implementação:** criar anotação `@RequiresGroupRole(GroupRole.ADMIN)` + `GroupAuthorizationService.checkRole(groupId, userId, minRole)` — encapsula busca do membro e comparação de papéis.

---

## 7. Tratamento de Erros

Todos os erros seguem RFC 7807 (Problem Details):

```json
{
  "type": "/errors/not-a-member",
  "title": "Acesso negado",
  "status": 403,
  "detail": "Você não é membro deste grupo.",
  "instance": "/api/v1/groups/uuid"
}
```

| Código de Negócio | HTTP | Cenário |
|---|---|---|
| `group-not-found` | 404 | Grupo não existe |
| `not-a-member` | 403 | Usuário não é membro |
| `insufficient-role` | 403 | Papel insuficiente para a operação |
| `member-not-found` | 404 | Membro não existe no grupo |
| `cannot-demote-owner` | 422 | Rebaixar OWNER sem transferência |
| `cannot-remove-last-owner` | 422 | Tentar remover único OWNER |
| `user-already-member` | 409 | Usuário já é membro ao tentar entrar |
| `invite-not-found` | 404 | Token de convite inexistente |
| `invite-expired` | 410 | Convite expirado ou com uso máximo atingido |
| `sport-change-not-allowed` | 422 | Alterar sport após 1ª formação de times |
| `group-inactive` | 422 | Operação em grupo desativado |

---

## 8. Telas do Frontend

### 8.1 `/groups` — Lista de Grupos

**Layout:** lista de `Card pressable` com `gap-3`, max-width 480px centralizado.

**Cada card:**
- `Avatar` do grupo (foto ou iniciais do nome, tamanho `lg`)
- Nome do grupo (`font-display text-title`)
- Modalidade como `Badge variant="neutral"` (ex.: "Futebol")
- Badge do papel do usuário: OWNER→success, ADMIN→warning, PLAYER/REFEREE→neutral
- Contagem de membros (`text-caption text-arena-muted`)

**Estado vazio:**
```
Nenhum grupo ainda.
Crie seu grupo ou peça um link de convite a um amigo.
[Criar grupo]
```

**Ação primária:** botão `primary` "Criar grupo" (fixo no bottom ou no topo da lista).

---

### 8.2 `/groups/new` — Criar Grupo

**Form fields:**
- Nome do grupo (`Input`, placeholder "Pelada do Parque", obrigatório, 3–80 chars)
- Modalidade (`Select` com as 6 opções, obrigatório)
- Descrição (`Textarea` opcional, máx. 500 chars)

**Submit:** `Button variant="primary" loading` → "Criar grupo"

**Após sucesso:** redireciona para `/groups/[id]`.

---

### 8.3 `/groups/[id]` — Painel do Grupo

**Header:**
- Nome do grupo (`font-display text-hero uppercase`)
- Modalidade e contagem de membros (`text-caption text-arena-muted`)
- Botões contextuais por papel:
  - OWNER/ADMIN: "Convidar" (abre bottom sheet com link) + "Configurações" (→ `/groups/[id]/settings`)
  - PLAYER/REFEREE: apenas "Convidar" (se ADMIN também permitir — manter restrito OWNER/ADMIN)

**Lista de membros:**
- `PlayerCard` por membro (nome, papel, skill, status de banimento de presença se aplicável)
- Ordenação: OWNER → ADMINs → PLAYERs → REFEREEs, alfabético dentro de cada grupo

---

### 8.4 `/groups/[id]/settings` — Configurações (OWNER/ADMIN)

**Seção "Grupo":**
- Editar nome, descrição, chave Pix
- `Button variant="primary"` "Salvar alterações"

**Seção "Membros":**
- Cada membro com `PlayerCard` + menu de ações (ícone `⋮`):
  - OWNER: Promover para ADMIN / Rebaixar para PLAYER / Remover / Transferir grupo
  - ADMIN: Ajustar skill / Alterar posição / Remover (exceto OWNER)
- Confirmação antes de remover (dialog)

**Seção "Convites":**
- Lista de convites ativos com link, validade e usos restantes
- Botão "Gerar novo convite"
- Botão desativar convite (ícone lixeira)

**Seção "Danger Zone" (apenas OWNER):**
- `Button variant="danger"` "Desativar grupo" com confirmação em dialog

---

### 8.5 `/invite/[token]` — Landing de Convite

**Estados:**

**Convite válido:**
- Foto/ícone do grupo + nome + modalidade + contagem de membros
- `Badge variant="success"` "Convite válido · expira em X dias"
- `Button variant="primary" size="lg"` "Entrar no grupo" (w-full)
  - Se não autenticado: redireciona `/login?redirect=/invite/[token]`
  - Se autenticado: chama `POST /api/v1/invites/{token}/join`

**Convite expirado:**
- Mesma info do grupo, mas `Badge variant="danger"` "Convite expirado"
- Texto: "Este link expirou. Peça um novo link ao organizador do grupo."
- Botão desabilitado ou ausente

**Token inválido (404):**
- "Link inválido. Verifique se o link foi copiado corretamente."

---

## 9. Estratégia de Testes

### 9.1 Testes Unitários (JUnit 5 + Mockito)

| Classe | O que testar |
|---|---|
| `Group` | Factory (criador vira OWNER), `addMember`, `removeMember` (único OWNER), `generateInvite`, `deactivate` |
| `GroupMember` | Factory (skill padrão 3.0), invariante skill 1–6 |
| `GroupInvite` | `isValid` (ativo + uso < max + não expirado), `incrementUsage` |
| `GroupService` | Join com convite expirado, join duplicado, rebaixar OWNER, remover único OWNER, atualizar sport após times |
| `GroupAuthorizationService` | Cada operação × cada papel — matriz de autorização completa |

### 9.2 Testes de Integração (Testcontainers + PostgreSQL)

| Cenário | Descrição |
|---|---|
| Fluxo completo de grupo | POST create → GET detail → PATCH update → DELETE deactivate |
| Convite | POST invite → GET preview (anon) → POST join → verificar membro criado |
| Hierarquia de papéis | ADMIN tenta remover OWNER → 403; OWNER promove ADMIN → 200 |
| Transferência de OWNER | PATCH role=OWNER no alvo → target vira OWNER, antigo OWNER vira ADMIN |
| Convite expirado | Criar convite com expiração passada (via backdoor/clock) → 410 ao tentar join |
| Join duplicado | Tentar entrar duas vezes no mesmo grupo → 409 |

### 9.3 Meta de Cobertura

- **≥ 80%** total (JaCoCo)
- Domain e use cases: **≥ 90%**

---

## 10. Ordem de Implementação

### Backend
1. Value Objects (`GroupName`, `Sport`, `GroupRole`, `Skill`, etc.)
2. Entidades de domínio (`GroupMember`, `GroupInvite`, `Group` com factory e métodos)
3. `GroupRepository` (port/out) — interface
4. JPA Entities e adapters (`GroupJpaEntity`, `GroupRepositoryAdapter`)
5. `GroupAuthorizationService` — verifica papel do usuário no grupo
6. `GroupService` — implementa todos os 13 use cases
7. DTOs e Controllers (`GroupController`, `InviteController`)
8. Security Config — adicionar `GET /api/v1/invites/*` ao `permitAll()`
9. Testes unitários (domain + service)
10. Testes de integração (controllers)

### Frontend
11. TanStack Query hooks (`useGroups`, `useGroup`, `useGroupMembers`, `useGroupInvites`)
12. API client functions (`groupsApi`)
13. Página `/groups` (listagem)
14. Página `/groups/new` (criar)
15. Página `/groups/[id]` (painel)
16. Página `/groups/[id]/settings` (configurações)
17. Página `/invite/[token]` (landing de convite)
18. Validação de fluxo completo no ambiente de staging

---

## 11. Migrações Flyway

As migrations para este módulo **já existem** no repositório:

| Arquivo | Tabela | Observação |
|---|---|---|
| `V002__create_groups.sql` | `groups` | sport CHECK inclui todos os 6 valores; `deleted_at` presente para soft delete |
| `V003__create_group_members.sql` | `group_members` | skill NUMERIC(3,1); constraint uq(group_id, user_id); todos os ENUMs com CHECK |
| `V004__create_group_invites.sql` | `group_invites` | unique index no token; índice composto (group_id, active) |

Nenhuma migration nova é necessária para este módulo.
