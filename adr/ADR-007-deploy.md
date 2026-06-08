# ADR-007 — Deploy e Infraestrutura: Railway (MVP) → AWS (Escala)

**Status:** Aceito  
**Data:** 2026-06-07  
**Decisores:** Time de Arquitetura

---

## Contexto

Para o MVP precisamos de deploy rápido com custo mínimo. A longo prazo, à medida que o produto escala, precisaremos de mais controle sobre infraestrutura, multi-região e SLAs mais rígidos.

## Decisão

**MVP (H1 — meses 1-4):** Railway  
**Crescimento (H2/H3 — meses 5+):** AWS (ECS Fargate + RDS PostgreSQL + S3)

### Arquitetura no Railway (MVP)

```
Railway Project: arenahub
├── Service: backend          # Spring Boot JAR via Dockerfile
├── Service: frontend         # Vite build estático via Nginx Dockerfile
├── Plugin: PostgreSQL         # Railway-managed PostgreSQL
└── Plugin: Redis (futuro)    # Cache / job queue
```

**Por que Railway:**
- Deploy de Dockerfile com zero configuração de Kubernetes
- PostgreSQL gerenciado incluído no plano
- Deploy automático via push no GitHub (branch main → staging, tag → prod)
- Custo: ~US$ 20-40/mês para MVP

### Ambientes

| Ambiente | Branch Git | Propósito |
|---|---|---|
| development | feature/* | Local com docker compose |
| staging | main | Validação antes de produção |
| production | tags (v*) | Usuários reais |

### Docker Compose (Desenvolvimento Local)

```yaml
# docker-compose.yml — ambiente local completo
services:
  postgres:   # PostgreSQL 16
  backend:    # Spring Boot com hot reload
  frontend:   # Vite dev server
  mailhog:    # SMTP fake para emails
```

### CI/CD — GitHub Actions

```
Push em feature/* branch:
  1. Lint + testes unitários
  2. Build (backend + frontend)
  3. SonarQube análise

Push em main:
  1. Lint + testes unitários + integração (Testcontainers)
  2. Build e push de imagem Docker
  3. Deploy automático em staging
  4. Smoke tests em staging

Tag v*:
  1. Todos os passos acima
  2. Deploy em produção (aprovação manual obrigatória)
```

### Caminho de Migração para AWS

Quando o Railway se tornar limitante (> 500 grupos ativos, nécessidade de SLA 99.9%):

```
AWS Target Architecture:
├── ECS Fargate          # Backend (containers gerenciados)
├── RDS PostgreSQL       # Banco gerenciado com Multi-AZ
├── S3 → Cloudflare R2   # Já compatível (API S3)
├── CloudFront           # CDN para frontend estático
├── Route 53             # DNS
└── ALB                  # Load Balancer
```

A migração é viabilizada pelas decisões anteriores:
- **ADR-002:** PostgreSQL padrão → compatível com RDS
- **ADR-006:** R2 API S3-compatible → migração sem mudança de código
- **ADR-003:** JWT stateless → sem estado em servidor → escala horizontal trivial

## Alternativas Consideradas

### Opção A: AWS desde o início
- **Prós:** Sem migração futura; mais controle
- **Contras:** Custo inicial alto; complexidade de configuração atrasa o MVP; requer DevOps experiente

### Opção B: Heroku
- **Prós:** Familiar; simples
- **Contras:** Mais caro que Railway; sem PostgreSQL gratuito desde 2022; planos limitados

### Opção C: Render
- **Prós:** Semelhante ao Railway; boa alternativa
- **Contras:** Railway tem melhor integração com GitHub e menor latência de deploy

### Opção D: Railway + migração futura para AWS (escolhida)
- **Prós:** Deploy imediato no MVP; custo baixo; migração planejada e viável pelas decisões de stack
- **Contras:** Migração eventual requer trabalho de DevOps (estimado: 2 sprints quando necessário)

## Consequências

**Positivas:**
- MVP em produção em < 1 semana após código pronto
- Custo previsível e baixo no MVP
- Migração para AWS viabilizada pela stack escolhida (PostgreSQL padrão, S3-compatible, JWT stateless)

**Negativas:**
- Railway tem menos recursos de observabilidade que AWS
- Migração futura requer planejamento de downtime mínimo

## Variáveis de Ambiente por Ambiente

Todas as configurações sensíveis são externalizadas via variáveis de ambiente:

```bash
# Exemplo .env.example (commitado)
DATABASE_URL=
JWT_SECRET=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
R2_ACCESS_KEY=
R2_SECRET_KEY=
R2_BUCKET=
R2_ENDPOINT=
AI_VALIDATION_URL=
AI_VALIDATION_API_KEY=
```

Valores reais nunca são commitados; gerenciados como secrets no Railway e no GitHub Actions.
