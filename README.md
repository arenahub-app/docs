# ArenaHub

SaaS multi-tenant para gestão de grupos esportivos amadores.

## Status do Projeto

| Etapa | Status |
|---|---|
| Etapa 1 — Descoberta e Documentação | ✅ Concluída |
| Etapa 2 — Modelagem de Domínio | ✅ Concluída |
| Etapa 3 — Banco de Dados | ⏳ Pendente |
| Etapa 4 — Backend | ⏳ Pendente |
| Etapa 5 — Frontend | ⏳ Pendente |
| Etapa 6 — Mobile | ⏳ Pendente |
| Etapa 7 — Infraestrutura | ⏳ Pendente |

## Estrutura do Projeto

```
arenahub/
├── backend/        # Spring Boot 3 + Java 21
├── frontend/       # React + TypeScript + Vite
├── mobile/         # React Native + Expo
├── specs/          # Especificações SDD (requisitos, regras, histórias)
├── docs/           # Documentação funcional e técnica
├── adr/            # Architecture Decision Records
├── diagrams/       # Diagramas de arquitetura e domínio
├── scripts/        # Scripts utilitários
├── infra/          # Docker, Terraform, Kubernetes
└── .github/        # Pipelines CI/CD
```

## Documentação

### docs/
| Documento | Descrição |
|---|---|
| [01 — Visão do Produto](docs/01-visao-do-produto.md) | Visão geral, problema, solução e proposta de valor |
| [02 — Objetivos do Negócio](docs/02-objetivos-do-negocio.md) | OKRs, métricas e restrições |
| [03 — Escopo](docs/03-escopo.md) | Dentro/fora do escopo do MVP |
| [04 — Personas](docs/04-personas.md) | Perfis dos usuários principais |
| [05 — Casos de Uso](docs/05-casos-de-uso.md) | UC detalhados com fluxos |
| [06 — Fluxos do Sistema](docs/06-fluxos-do-sistema.md) | Diagramas de fluxo por funcionalidade |
| [07 — Roadmap](docs/07-roadmap.md) | Horizonte de entregas e milestones |
| [08 — Modelagem de Domínio](docs/08-modelagem-de-dominio.md) | Bounded Contexts, Aggregates, Entities, VOs, Domain Services |

### diagrams/
| Diagrama | Descrição |
|---|---|
| [01 — Bounded Contexts](diagrams/01-bounded-contexts.md) | Context Map e relacionamentos entre contextos |
| [02 — Aggregate Model](diagrams/02-aggregate-model.md) | UML de classes por BC (Mermaid) |
| [03 — ER Diagram](diagrams/03-er-diagram.md) | Diagrama Entidade-Relacionamento com tabelas e constraints |
| [04 — Domain Events](diagrams/04-domain-events.md) | Event Storming e sequências de eventos |
| [05 — State Machines](diagrams/05-state-machines.md) | Máquinas de estado por entidade |
| [06 — Team Formation Algorithm](diagrams/06-team-formation-algorithm.md) | Snake Draft e pseudocódigo |
| [07 — Use Case Sequences](diagrams/07-use-case-sequence.md) | Diagramas de sequência dos principais casos de uso |

### specs/
| Documento | Descrição |
|---|---|
| [01 — Requisitos Funcionais](specs/01-requisitos-funcionais.md) | RFs por módulo |
| [02 — Requisitos Não Funcionais](specs/02-requisitos-nao-funcionais.md) | Performance, segurança, disponibilidade |
| [03 — Regras de Negócio](specs/03-regras-de-negocio.md) | RNs por domínio |
| [04 — Glossário](specs/04-glossario.md) | Termos do domínio e posições por modalidade |
| [05 — Critérios de Aceitação](specs/05-criterios-de-aceitacao.md) | CAs no formato Gherkin |
| [06 — Histórias de Usuário](specs/06-historias-de-usuario.md) | USs com story points e rastreabilidade |

### adr/
| ADR | Decisão |
|---|---|
| [ADR-001](adr/ADR-001-arquitetura-geral.md) | Clean Architecture + DDD Tático |
| [ADR-002](adr/ADR-002-banco-de-dados.md) | PostgreSQL + Flyway + Shared Schema |
| [ADR-003](adr/ADR-003-autenticacao.md) | JWT Stateless + OAuth2 Google |
| [ADR-004](adr/ADR-004-frontend.md) | React + Vite + MUI |
| [ADR-005](adr/ADR-005-mobile.md) | React Native + Expo |
| [ADR-006](adr/ADR-006-storage.md) | Cloudflare R2 |
| [ADR-007](adr/ADR-007-deploy.md) | Railway (MVP) → AWS (Escala) |
| [ADR-008](adr/ADR-008-multitenancy.md) | Multi-tenancy por group_id |
| [ADR-009](adr/ADR-009-domain-modeling.md) | Decisões de Agregados e Fronteiras |

## Stack Tecnológica

| Camada | Tecnologia |
|---|---|
| Backend | Java 21, Spring Boot 3, PostgreSQL, Flyway, JPA/Hibernate |
| Auth | Spring Security, OAuth2 Google, JWT |
| Frontend | React 18, TypeScript, Vite, MUI, React Query, React Hook Form, Zod |
| Mobile | React Native, Expo |
| Storage | Cloudflare R2 |
| Testes | JUnit 5, Mockito, Testcontainers |
| Qualidade | SonarQube, OpenAPI |
| Deploy | Docker, GitHub Actions, Railway → AWS |

## Metodologia

- **Spec Driven Development (SDD):** toda funcionalidade documentada antes de implementada
- **Clean Architecture:** separação de domínio, aplicação e infraestrutura
- **DDD Tático:** Aggregates, Entities, Value Objects, Domain Services
- **TDD:** testes escritos junto ou antes da implementação
- **Cobertura mínima:** 80%
