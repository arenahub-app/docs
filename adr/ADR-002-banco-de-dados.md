# ADR-002 — Banco de Dados: PostgreSQL com Shared Schema Multi-Tenancy

**Status:** Aceito  
**Data:** 2026-06-07  
**Decisores:** Time de Arquitetura

---

## Contexto

O sistema é multi-tenant: cada grupo esportivo é um tenant isolado. Precisamos definir:
1. Qual banco de dados usar
2. Qual estratégia de multi-tenancy adotar (schema separado vs. banco separado vs. coluna discriminadora)

## Decisão

**Banco:** PostgreSQL 16  
**Estratégia de multi-tenancy:** Shared Schema com coluna `group_id` como discriminador em todas as tabelas de negócio

**Gestão de migrações:** Flyway  
**ORM:** Spring Data JPA + Hibernate

### Estratégia Shared Schema

Todas as tabelas de negócio contêm a coluna `group_id UUID NOT NULL` com foreign key para `groups.id`. O isolamento é garantido por:

1. **Camada de aplicação:** toda query de negócio filtra por `group_id` vindo do contexto autenticado do usuário
2. **Testes de segurança:** CA-SEG-001 verifica que PLAYER de um grupo não acessa dados de outro
3. **Row-Level Security (RLS):** habilitado no PostgreSQL como camada adicional de defesa

```sql
-- Exemplo de RLS policy
ALTER TABLE matches ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON matches
  USING (group_id = current_setting('app.current_group_id')::uuid);
```

*Nota: RLS é uma camada de defesa em profundidade, não substitui a validação na aplicação.*

## Alternativas Consideradas

### Opção A: Schema por tenant (schema-per-tenant)
- **Prós:** Isolamento mais forte no BD; migrations por tenant mais simples de reverter
- **Contras:** Inviável com muitos tenants (100+ grupos = 100+ schemas); custo de conexão por schema; complexidade de connection pooling

### Opção B: Banco por tenant (database-per-tenant)
- **Prós:** Isolamento máximo; backup por tenant simples
- **Contras:** Inviável em SaaS small; custo de infraestrutura proibitivo; connection pooling impossível de compartilhar

### Opção C: Shared Schema com group_id (escolhida)
- **Prós:** Simples de escalar; connection pool compartilhado; migrations únicas para todos os tenants; custo baixo
- **Contras:** Isolamento depende da aplicação; risco de vazamento de dados por bug

### Opção D: MySQL / MariaDB
- **Prós:** Familiar; amplamente suportado no Railway
- **Contras:** Menos recursos avançados (CTE recursivas, JSONB, RLS, GIN indexes); PostgreSQL é superior para queries analíticas dos relatórios

## Consequências

**Positivas:**
- Uma única migration atualiza todos os tenants simultaneamente
- Connection pooling simples com HikariCP
- Custo de infra mínimo no Railway para MVP

**Negativas:**
- Bug na camada de aplicação pode vazar dados entre tenants
- RLS adiciona overhead de configuração e testes

## Padrão de Migração Flyway

```
src/main/resources/db/migration/
├── V001__create_users.sql
├── V002__create_groups.sql
├── V003__create_group_members.sql
└── ...
```

- Prefixo `V` com número sequencial de 3 dígitos
- Descrição em snake_case após `__`
- Migrations são imutáveis após merge em main
