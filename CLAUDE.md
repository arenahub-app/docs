# ArenaHub — Contexto para Claude Code

## O que é este projeto

SaaS multi-tenant para gestão de grupos esportivos amadores (presença, times, pagamentos, skill voting, financeiro).

## Estado atual

| Etapa | Status |
|---|---|
| 1 — Documentação | ✅ Concluída |
| 2 — Modelagem de Domínio | ✅ Concluída |
| 3 — Banco de Dados | ✅ Concluída |
| 4 — Backend, Módulo 1: Autenticação | ✅ Concluída |
| 4 — Frontend, Módulo 1: Autenticação | ✅ Concluída |
| 4.5 — CI/CD & Infraestrutura | ✅ Concluída |
| Feature 1 — Group Management | ✅ Produção |
| Feature 2 — Match & Presence | ✅ Produção |
| Feature 3 — Team Formation | ✅ Produção |
| Feature 4 — Skill Voting | 🔜 Próxima |
| Feature 5 — Payments | ✅ Produção |
| Feature 6 — Financial | ⏳ |
| Fix — Google OAuth | 🔜 Pendente |
| 4.6 — Observabilidade (Grafana Cloud) | ⏳ Pós-MVP |
| 4.7 — Serviço de Mensageria Assíncrona | ⏳ Pós-MVP |
| 5.9 — Domínio Personalizado (pré-produção) | ⏳ |
| Mobile | ⏳ Fase final |

**Próximo passo:** Feature 4 — Skill Voting (SDD → backend → frontend → produção).

> **Fix pendente — Google OAuth:** login/cadastro via Google não está funcionando em produção. O botão está desabilitado no frontend enquanto o problema não é investigado. Investigar configuração do OAuth2 Client no backend (callback URL, client-id/secret no ambiente de produção) e fluxo de redirect.

---

## Etapa 4.5 — CI/CD & Infraestrutura (PRÓXIMA)

Deve ser concluída **antes** de qualquer nova feature. Contém dois blocos:

### Bloco A — Melhorias de CI/CD

| Item | Descrição |
|---|---|
| Separar unit/integration | Surefire (unit tests) + Failsafe (integration tests) para feedback mais rápido |
| OWASP Dependency Check | Scan de vulnerabilidades nas dependências; executado semanalmente (não em todo PR) |
| Coverage comment em PR | JaCoCo report publicado como comentário automático no PR via `madrapps/jacoco-report` |
| Checkstyle | Regras básicas de estilo (permissivas para não quebrar o código atual) |

### Bloco B — Infraestrutura & Ambiente Testável

Serviços gratuitos a conectar:

| Serviço | Plataforma | Uso |
|---|---|---|
| PostgreSQL | Neon (free tier) | Banco de dados (staging + prod) |
| Email | Resend (free tier) | SMTP transacional |
| Backend hosting | Fly.io (free tier) | Container Docker do Spring Boot |
| Frontend hosting | Vercel (free tier) | SPA React |
| Storage | Cloudflare R2 (free tier) | Upload de arquivos |
| Container registry | GitHub Container Registry | Imagens Docker |

Mudanças no pipeline após esta etapa:

```
Push → develop
  └─► validate (testes + qualidade)
  └─► versão semântica + tag
  └─► build Docker → push GHCR
  └─► deploy → staging (Fly.io)   ← NOVO
  └─► PR automática → main (com URL do staging no corpo)

Merge → main
  └─► build Docker → push GHCR
  └─► deploy → production (Fly.io)  ← NOVO
  └─► GitHub Release criada automaticamente
```

Secrets a configurar no GitHub (após criar as contas):
- `FLY_API_TOKEN` — token da CLI do Fly.io
- `DATABASE_URL` — Neon staging (jdbc:postgresql://...)
- `DATABASE_URL_PROD` — Neon production
- `JWT_SECRET` / `JWT_SECRET_PROD`
- `RESEND_API_KEY`
- `FRONTEND_BASE_URL` / `FRONTEND_BASE_URL_PROD`
- `R2_ENDPOINT`, `R2_ACCESS_KEY`, `R2_SECRET_KEY`, `R2_BUCKET`, `R2_PUBLIC_URL`

---

## Etapa 4.7 — Serviço de Mensageria Assíncrona (FUTURA)

Operações que não devem bloquear o request HTTP nem comprometer a transação principal:

| Operação | Situação atual | Problema |
|---|---|---|
| Envio de email (verificação, reset, notificações) | `afterCommit()` síncrono | Lentidão e falha silenciosa sem retry |
| Pagamentos (webhooks, confirmações) | Não implementado | Requer retry e idempotência |
| Notificações push (mobile) | Não implementado | Alta volumetria, fire-and-forget |
| Geração de relatórios | Não implementado | Processamento pesado off-request |

### Opções gratuitas de message broker a avaliar

| Serviço | Free tier | Pontos positivos | Limitações |
|---|---|---|---|
| **Upstash (Redis/QStash)** | 10.000 msg/dia, 500 req/dia (QStash) | Serverless, HTTP-based, retry nativo, delay, FIFO | Volume baixo |
| **CloudAMQP (RabbitMQ)** | 1M msgs/mês, 20 conexões | RabbitMQ gerenciado, Spring AMQP nativo | 1 vhost, sem persistência longa |
| **Render (Redis)** | 512 MB RAM, expira em 90 dias | Fácil integração com Railway | Redis puro — precisa implementar retry manual |
| **Neon + cron job** | Já em uso no projeto | Zero infra nova — tabela de `outbox` + job que consome | Latência e complexidade do polling |

### Padrão recomendado: Outbox Pattern

Independente do broker escolhido, implementar o **Transactional Outbox Pattern**:

```
[Request HTTP]
  └─► @Transactional: salva entidade + grava evento na tabela outbox (mesma tx)
  └─► Job periódico (ex: @Scheduled 10s): lê outbox → publica no broker → marca como enviado
```

Vantagens: garante "at-least-once delivery" sem acoplamento direto ao broker na hora do request.

### Decisão a tomar antes de implementar

- Qual broker usar (avaliar volume esperado de emails/notificações por mês)
- Se usar Outbox: definir schema da tabela e política de retry/dead-letter
- Integração com o módulo de Pagamentos (Etapa 5+) para não duplicar esforço

---

## Etapa 5.9 — Domínio Personalizado (pré-produção)

Executar **antes do primeiro deploy em produção real**, após todas as features do MVP estarem prontas.

| Item | Detalhe |
|---|---|
| Registrar domínio | Cloudflare Registrar (preço de custo, sem markup) — sugestões: `arenahub.app` (~$14/ano) ou `arenahub.com` (~$10/ano) |
| DNS frontend | CNAME `arenahub.app` → Vercel (**DNS only**, sem proxy Cloudflare) |
| DNS backend | CNAME `api.arenahub.app` → Railway (**DNS only**) |
| Variáveis de ambiente | `FRONTEND_BASE_URL` no Railway → `https://arenahub.app`; `NEXT_PUBLIC_API_URL` na Vercel → `https://api.arenahub.app` |
| SSL | Gerado automaticamente por Vercel e Railway após propagação DNS |

> Até lá, usar as URLs geradas automaticamente para staging: Vercel preview URL + `backend-staging-production-2fe6.up.railway.app`.

---

## Estratégia de entrega (atualizada)

Cada feature é entregue de forma completa e **sequencial**: backend implementado e deployado em staging primeiro, depois frontend integrado. Só então a feature vai para produção. Mobile fica para a fase final.

Fluxo por feature:
1. SDD (documentação da feature — contratos de API, telas, regras de negócio)
2. Backend implementado + testes + deploy staging
3. Frontend implementado + integrado ao staging
4. Validação manual da experiência de usuário no ambiente testável
5. PR → main → deploy produção (backend e frontend juntos)

**Regra:** nenhuma feature vai para produção com apenas backend ou apenas frontend. Os dois lados sobem juntos.

## Repositórios GitHub

Organização: `arenahub-app`

| Repo | URL | Branch atual |
|---|---|---|
| docs | https://github.com/arenahub-app/docs | develop |
| backend | https://github.com/arenahub-app/backend | develop |
| frontend | https://github.com/arenahub-app/frontend | develop |
| mobile | https://github.com/arenahub-app/mobile | develop |

## Diretórios locais

```
/home/roberto/workspace/arenahub/          → docs repo
/home/roberto/workspace/arenahub/backend/  → backend repo (Spring Boot)
/home/roberto/workspace/arenahub/frontend/ → frontend repo (React)
/home/roberto/workspace/arenahub/mobile/   → mobile repo (Expo)
```

## Regras do projeto

- **SDD obrigatório:** documentação antes de código
- **Commits semânticos:** `feat:`, `fix:`, `docs:`, `chore:`, `test:`, `refactor:`, `ci:`
- **Branches:** `develop` (padrão) → PR → `main` (protegida)
- **Cobertura mínima:** 80% (JaCoCo no backend)
- **Sem comentários desnecessários no código**
- **Testes obrigatórios em toda alteração de código:** qualquer mudança em código de produção deve ser acompanhada da atualização dos testes unitários e/ou de integração correspondentes. Não deixar coverage cair abaixo do mínimo configurado no JaCoCo.

## Stack

- **Backend:** Java 21, Spring Boot 3.3.5, PostgreSQL 16, Flyway, JPA, MapStruct, jjwt 0.12.6
- **Auth:** Spring Security + OAuth2 Client (Google) + JWT stateless
- **Storage:** Cloudflare R2 via AWS SDK v2
- **Testes:** JUnit 5, Mockito, Testcontainers
- **Frontend:** Next.js 16, React 19, TypeScript, Tailwind CSS, shadcn/ui (@base-ui/react), TanStack Query v5, Zod v4, react-hook-form, Vercel
- **Mobile:** React Native + Expo (não iniciado)

## Arquitetura do backend

Clean Architecture + DDD Tático. Pacotes:

```
com.arenahub/
├── domain/        ← entidades, VOs, repositórios (interfaces), domain services
├── application/   ← casos de uso, ports (interfaces de entrada/saída), DTOs
├── infrastructure/← JPA, R2, IA adapter, security config, email
└── presentation/  ← REST controllers, DTOs HTTP, mappers MapStruct
```

## 7 Bounded Contexts do domínio

1. Identity & Access — User, RefreshToken
2. Group Management — Group, GroupMember, GroupInvite, RefereeProfile
3. Match & Presence — Match, PresenceEntry, WaitingEntry
4. Skill Voting — SkillVoting, Vote, VotingBan
5. Team Formation — TeamFormation, Team, TeamPlayer
6. Payments — Charge, PaymentAttempt
7. Financial — FinancialEntry
