# ArenaHub — Contexto para Claude Code

## O que é este projeto

SaaS multi-tenant para gestão de grupos esportivos amadores (presença, times, pagamentos, skill voting, financeiro).

## Estado atual

| Etapa | Status |
|---|---|
| 1 — Documentação | ✅ Concluída |
| 2 — Modelagem de Domínio | ✅ Concluída |
| 3 — Banco de Dados | ✅ Concluída |
| 4 — Backend | 🔜 Próxima |
| 5 — Frontend | ⏳ |
| 6 — Mobile | ⏳ |
| 7 — Infraestrutura | ⏳ |

**Próximo passo:** Etapa 4 — Backend, módulo 1: Autenticação (registro, login, OAuth2 Google, JWT).

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

## Stack

- **Backend:** Java 21, Spring Boot 3.3.5, PostgreSQL 16, Flyway, JPA, MapStruct, jjwt 0.12.6
- **Auth:** Spring Security + OAuth2 Client (Google) + JWT stateless
- **Storage:** Cloudflare R2 via AWS SDK v2
- **Testes:** JUnit 5, Mockito, Testcontainers
- **Frontend:** React 18, TypeScript, Vite, MUI, React Query, Zod (não iniciado)
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
