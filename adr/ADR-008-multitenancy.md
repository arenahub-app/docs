# ADR-008 — Multi-Tenancy: Isolamento por Group ID na Camada de Aplicação

**Status:** Aceito  
**Data:** 2026-06-07  
**Decisores:** Time de Arquitetura

---

## Contexto

O ArenaHub é multi-tenant onde cada grupo esportivo é um tenant. Precisamos garantir que dados de um grupo não sejam acessíveis por membros de outro grupo, mesmo em cenários de manipulação de parâmetros de API.

## Decisão

**Estratégia:** Tenant por `group_id` (shared schema) com validação em múltiplas camadas:

### Camada 1 — Spring Security (Autorização)
Cada endpoint que opera em contexto de grupo verifica se o usuário autenticado é membro do grupo informado via `@PreAuthorize` ou `SecurityContextHolder`.

### Camada 2 — Camada de Aplicação (Casos de Uso)
Todo caso de uso que acessa dados de um grupo recebe o `groupId` do usuário autenticado (não do parâmetro de URL). O GroupMembershipService valida membership antes de qualquer operação.

```java
// Padrão em todos os casos de uso de grupo
public MatchResponse createMatch(UUID groupId, CreateMatchRequest request, UUID currentUserId) {
    groupMembershipService.assertAdmin(groupId, currentUserId); // lança 403 se não for admin
    // ... lógica do caso de uso
}
```

### Camada 3 — Repositórios (Queries)
Todas as queries de entidades de negócio incluem `AND group_id = :groupId` obrigatoriamente. O `group_id` vem do contexto autenticado, não do input do usuário.

```java
// Exemplo de repository
Optional<Match> findByIdAndGroupId(UUID matchId, UUID groupId);
// Nunca: findById(matchId) — sem filtragem por grupo
```

### Camada 4 — PostgreSQL Row-Level Security (Defesa em Profundidade)
RLS habilitado nas tabelas principais como última linha de defesa.

## Modelo de Contexto de Tenant

O `groupId` é resolvido a partir do JWT (sub) + parâmetro de rota e validado contra `group_members` a cada requisição. Não é armazenado no JWT para evitar tokens obsoletos quando um usuário é removido do grupo.

```
JWT token: { sub: userId }
Request path: /api/v1/groups/{groupId}/matches
               ─────────────────────────────
               → GroupMembershipService.assertMember(groupId, userId)
               → Se não membro: throw ForbiddenException
```

## Consequências

**Positivas:**
- Segurança em múltiplas camadas (defense in depth)
- Simples de auditar: toda query tem group_id explícito
- Sem overhead de schema routing

**Negativas:**
- Requer disciplina dos desenvolvedores (sempre filtrar por group_id)
- Testes de segurança de cross-tenant são obrigatórios (CA-SEG-001)
- RLS adiciona overhead de configuração de sessão do PostgreSQL
