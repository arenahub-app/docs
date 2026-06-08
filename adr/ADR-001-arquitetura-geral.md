# ADR-001 — Arquitetura Geral: Clean Architecture com DDD Tático

**Status:** Aceito  
**Data:** 2026-06-07  
**Decisores:** Time de Arquitetura  

---

## Contexto

O ArenaHub é um SaaS multi-tenant com domínio rico: votação com regras complexas, formação automática de times, validação de pagamentos por IA, gestão financeira e multi-tenancy. Precisamos de uma arquitetura que:

1. Permita isolar regras de negócio do framework e da infraestrutura
2. Facilite testes unitários de casos de uso sem subir o Spring ou banco de dados
3. Suporte crescimento do domínio sem acoplamento tecnológico
4. Seja compreensível por desenvolvedores com experiência em Spring Boot

## Decisão

Adotar **Clean Architecture** (por Robert C. Martin) combinada com **DDD Tático** (Aggregates, Entities, Value Objects, Domain Services, Repositories) como estrutura organizacional do backend.

### Camadas

```
arenahub-backend/
└── src/main/java/com/arenahub/
    ├── domain/              # Regras de negócio puras — sem dependências externas
    │   ├── model/           # Entidades, Aggregates, Value Objects
    │   ├── repository/      # Interfaces de repositórios (contratos)
    │   ├── service/         # Domain Services
    │   └── event/           # Domain Events
    │
    ├── application/         # Casos de uso (orquestração)
    │   ├── usecase/         # Um caso de uso por classe
    │   ├── port/            # Ports (input/output — interfaces)
    │   └── dto/             # DTOs de entrada e saída dos casos de uso
    │
    ├── infrastructure/      # Implementações concretas
    │   ├── persistence/     # JPA Repositories, Entities JPA, Mappers
    │   ├── storage/         # Cloudflare R2 adapter
    │   ├── ai/              # Adapter do serviço de IA
    │   ├── messaging/       # Notificações, emails
    │   └── security/        # Spring Security config, JWT
    │
    └── presentation/        # Entrada do sistema
        ├── controller/      # REST Controllers
        ├── dto/             # DTOs de request/response HTTP
        └── mapper/          # MapStruct: DTO HTTP ↔ DTO de caso de uso
```

### Regras de Dependência

- `domain` não depende de nada além do Java
- `application` depende de `domain`
- `infrastructure` e `presentation` dependem de `application` e `domain`
- Dependências externas (Spring, JPA, etc.) ficam exclusivamente em `infrastructure` e `presentation`

## Alternativas Consideradas

### Opção A: Arquitetura em Camadas Tradicional (Controller → Service → Repository)
- **Prós:** Familiar para a maioria dos devs Java; menos overhead inicial
- **Contras:** Regras de negócio ficam misturadas com Spring/JPA; testes unitários exigem mocking pesado do framework; difícil de evoluir sem acoplamento crescente

### Opção B: Hexagonal Architecture pura
- **Prós:** Rigor teórico maior na separação de ports/adapters
- **Contras:** Terminologia menos familiar; overhead de mapeamento entre ports aumenta em domínios simples; Clean Architecture já captura a essência sem adicionar complexidade de nomenclatura

## Consequências

**Positivas:**
- Regras de negócio testáveis com JUnit puro (sem Spring Test)
- Substituição de infra (banco, storage, IA) sem alterar domain/application
- Onboarding facilitado: camadas têm responsabilidades claras

**Negativas:**
- Mais boilerplate inicial (DTOs em múltiplas camadas, mappers)
- Curva de aprendizado para devs acostumados com arquitetura anêmica
- MapStruct adiciona complexidade de build (geração de código)

## Referências

- Clean Architecture — Robert C. Martin (2017)
- Domain-Driven Design — Eric Evans (2003)
- Implementing Domain-Driven Design — Vaughn Vernon (2013)
