# ADR-009 — Modelagem de Domínio: Decisões de Agregados

**Status:** Aceito  
**Data:** 2026-06-07  
**Decisores:** Time de Arquitetura

---

## Contexto

Ao modelar o domínio do ArenaHub, diversas decisões sobre fronteiras de agregados foram tomadas. Este ADR registra as principais.

---

## Decisão 1 — `Group` contém `GroupMember`

**Decisão:** `GroupMember` é uma entidade dentro do agregado `Group`, não um agregado independente.

**Justificativa:** O `Group` controla invariantes de membership que exigem visibilidade de todos os membros:
- "Exatamente um OWNER por grupo"
- "Unicidade de userId por grupo"
- "Skill entre 1.0 e 6.0"

Se `GroupMember` fosse um AR independente, garantir "somente um OWNER" exigiria coordenação entre agregados (transação distribuída ou eventual consistency), o que é desnecessariamente complexo para este invariante.

**Trade-off:** O agregado `Group` pode ficar grande em grupos com muitos membros. Mitigação: lazy loading via JPA; o agregado carrega membros somente quando necessário para operações que os exigem.

---

## Decisão 2 — `TeamFormation` é separada de `Match`

**Decisão:** `TeamFormation` é um Aggregate Root independente com referência por `matchId`.

**Justificativa:**
- Formação tem ciclo de vida próprio: pode ser recriada, editada manualmente, tem histórico
- `Match` já contém presença, fila de espera e árbitro — adicionar times tornaria o AR excessivamente grande
- Times podem ser visualizados sem carregar todos os dados da partida

**Trade-off:** Acesso cross-aggregate por ID. O caso de uso verifica o estado da partida antes de criar a formação (lista fechada). Consistência eventual entre `Match.presenceListStatus` e criação de `TeamFormation` é aceitável neste domínio.

---

## Decisão 3 — `Charge` separada de `FinancialEntry`

**Decisão:** Pagamentos (BC-06) e Financeiro (BC-07) são contextos separados com entidades distintas.

**Justificativa:**
- `Charge` gerencia ciclo de vida de cobrança com comprovantes e validação: responsabilidade de "cobrar e validar"
- `FinancialEntry` é registro contábil imutável (reversível, não excluível): responsabilidade de "registrar o fato financeiro"
- Um pagamento aprovado → publica evento → Financial cria receita. Separa claramente causa e efeito
- Despesas de árbitro não têm `Charge` associada; entram direto no Financial via evento

**Trade-off:** Dois registros para um pagamento aprovado (Charge + FinancialEntry). Compensado pela clareza das responsabilidades.

---

## Decisão 4 — `SkillVoting` é separado de `Group`

**Decisão:** `SkillVoting` é um Aggregate Root independente com referência por `groupId`.

**Justificativa:**
- Votação tem regras complexas independentes: unicidade por grupo, banimentos, anonimato, cálculo de médias
- Ciclo de vida distinto: OPEN → CLOSED → resultado calculado
- Incorporar no `Group` tornaria o AR com responsabilidades excessivas

**Trade-off:** Ao encerrar a votação, é necessário atualizar `GroupMember.skill` em outro agregado. Resolvido com `VotingClosedEvent` publicado para o contexto de Group Management.

---

## Decisão 5 — Referências entre agregados apenas por UUID

**Decisão:** Agregados se referenciam exclusivamente por UUID, nunca por referência de objeto.

**Justificativa:**
- Evita acoplamento estrutural entre agregados
- Permite que agregados vivam em módulos independentes
- Facilita eventual extração de microserviços no futuro
- Alinhado com DDD: agregados são fronteiras de consistência

**Implementação JPA:** campos `UUID userId`, `UUID groupId` etc. sem `@ManyToOne` cruzando fronteiras de agregado. Joins para leitura são feitos via queries JPQL ou views.

---

## Decisão 6 — Eventos de domínio são síncronos no MVP

**Decisão:** Domain Events são publicados e consumidos de forma síncrona no mesmo processo (Spring ApplicationEvent), sem message broker.

**Justificativa:**
- Simplicidade para MVP: sem Kafka, RabbitMQ ou SQS
- O volume de dados no MVP não justifica complexidade de messaging distribuído
- Consistência transacional garantida: evento publicado dentro da mesma transação do aggregate save

**Trade-off:** Se o handler de evento falhar, a transação inteira é revertida. Aceitável no MVP; migração para mensageria assíncrona é possível sem mudar os Domain Events (apenas trocar o mecanismo de publicação).

---

## Decisão 7 — Multi-tenancy por `group_id` (não por schema)

*Veja ADR-002 e ADR-008 para detalhes completos.*

**Decisão:** Shared schema com coluna `group_id` como discriminador.

**Impacto na modelagem:** Todo AR de negócio (Match, Charge, SkillVoting, TeamFormation, FinancialEntry) carrega `groupId` e toda query de repositório filtra por ele.
