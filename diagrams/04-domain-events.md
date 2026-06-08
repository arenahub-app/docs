# Diagrama — Domain Events e Fluxos de Dados

## Event Storming — Fluxo Principal

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Event Storming — ArenaHub                           │
│          (Laranja = Evento | Azul = Comando | Verde = Read Model)       │
└─────────────────────────────────────────────────────────────────────────┘

AUTENTICAÇÃO
────────────
  [Registrar Usuário]
       │
       ▼
  ◉ UserRegisteredEvent
       │
       ├──► Envia email de confirmação
       └──► Cria conta com emailVerified=false

  [Confirmar Email]
       │
       ▼
  ◉ EmailVerifiedEvent
       └──► emailVerified=true


GRUPOS
──────
  [Criar Grupo]
       │
       ▼
  ◉ GroupCreatedEvent
       │
       ├──► Atribui papel OWNER ao criador
       └──► Gera link de convite


  [Aceitar Convite]
       │
       ▼
  ◉ MemberJoinedGroupEvent
       └──► Membro com papel PLAYER


  [Promover Membro]
       │
       ▼
  ◉ MemberRoleChangedEvent


  [Banir de Presença]
       │
       ▼
  ◉ MemberPresenceBannedEvent
       └──► Notifica membro


PRESENÇA
────────
  [Criar Partida]
       │
       ▼
  ◉ MatchCreatedEvent
       │
       └──► Notifica todos os PLAYERs do grupo

  [Confirmar Presença]
       │
       ▼
  ◉ PresenceConfirmedEvent

  [Lista Atinge Limite]
       │
       ▼
  ◉ PresenceListFullEvent

  [T-1h] Scheduler
       │
       ▼
  ◉ PresenceListClosedEvent
       │
       ├──► Notifica jogadores da fila que não entraram
       └──► Habilita formação de times

  [Associar Árbitro à Partida]
       │
       ▼
  ◉ RefereeAssignedEvent
       │
       └──► Financial BC: cria FinancialEntry (EXPENSE, REFEREE)


VOTAÇÃO
───────
  [Abrir Votação]
       │
       ▼
  ◉ VotingOpenedEvent
       └──► Notifica PLAYERs habilitados

  [Encerrar Votação] (manual ou prazo)
       │
       ▼
  ◉ VotingClosedEvent
       │
       └──► Group BC: calcula médias → atualiza skill de cada GroupMember


PAGAMENTOS
──────────
  [Upload de Comprovante]
       │
       ▼
  ◉ PaymentAttemptSubmittedEvent
       │
       └──► IA valida comprovante (assíncrono)

  [IA: Aprovado]
       │
       ▼
  ◉ PaymentApprovedEvent
       │
       └──► Financial BC: cria FinancialEntry (REVENUE, DAILY|SUBSCRIPTION)

  [IA: Reprovado]
       │
       ▼
  ◉ PaymentRejectedEvent
       └──► Notifica PLAYER

  [IA: Timeout ou Incerteza]
       │
       ▼
  ◉ PaymentRequiresReviewEvent
       └──► Notifica ADMIN para revisão manual

  [ADMIN Aprova Manualmente]
       │
       ▼
  ◉ PaymentApprovedEvent (source=MANUAL)
       │
       └──► Financial BC: cria FinancialEntry (REVENUE)


FORMAÇÃO DE TIMES
─────────────────
  [Solicitar Formação]
       │
       ▼
  [TeamFormationService.form()]
  (snake draft por skill + posição)
       │
       ▼
  ◉ TeamFormationCreatedEvent
       │
       └──► PLAYERs visualizam seus times
```

---

## Mapa de Eventos por Contexto

```mermaid
sequenceDiagram
  participant Admin
  participant Match_BC as Match BC
  participant Financial_BC as Financial BC
  participant Scheduler

  Admin->>Match_BC: assignReferee(matchId, refereeProfileId)
  Match_BC->>Match_BC: valida árbitro e partida
  Match_BC-->>Financial_BC: RefereeAssignedEvent{matchId, refereeProfileId, amount}
  Financial_BC->>Financial_BC: createExpense(REFEREE, amount)
  Financial_BC-->>Admin: despesa registrada
```

```mermaid
sequenceDiagram
  participant Player
  participant Payments_BC as Payments BC
  participant AI as Serviço IA
  participant Financial_BC as Financial BC

  Player->>Payments_BC: submitPaymentAttempt(fileKey)
  Payments_BC->>AI: validateReceipt(fileKey, expectedAmount, pixKey)
  alt Aprovado
    AI-->>Payments_BC: APPROVED
    Payments_BC-->>Financial_BC: PaymentApprovedEvent{chargeId, amount}
    Financial_BC->>Financial_BC: createRevenue(DAILY|SUBSCRIPTION, amount)
  else Reprovado
    AI-->>Payments_BC: REJECTED{reason}
    Payments_BC-->>Player: PaymentRejectedEvent
  else Timeout (> 10s)
    Payments_BC-->>Payments_BC: flagForManualReview()
    Payments_BC-->>Admin: PaymentRequiresReviewEvent
  end
```

```mermaid
sequenceDiagram
  participant Admin
  participant Voting_BC as Voting BC
  participant Group_BC as Group BC

  Admin->>Voting_BC: closeVoting(votingId)
  Voting_BC->>Voting_BC: calculateResults() → Map<memberId, avgSkill>
  Voting_BC-->>Group_BC: VotingClosedEvent{groupId, results}
  Group_BC->>Group_BC: updateMemberSkill(memberId, avgSkill, VOTING)
  Group_BC-->>Admin: skills atualizados
```

```mermaid
sequenceDiagram
  participant Scheduler
  participant Match_BC as Match BC

  Scheduler->>Match_BC: checkMatchesForListClosure()
  Note over Scheduler,Match_BC: Executa a cada minuto
  Match_BC->>Match_BC: find matches where listClosesAt <= now AND status=OPEN
  Match_BC->>Match_BC: closePresenceList()
  Match_BC-->>Match_BC: PresenceListClosedEvent
  Match_BC->>Match_BC: notifyWaitingQueue() → remove da fila
```

---

## Invariantes Cross-Aggregate Críticos

| Invariante | Como Garantir |
|---|---|
| Somente uma votação ativa por grupo | Verificação no caso de uso antes de criar `SkillVoting`; índice parcial no banco |
| Formação só após lista fechada | Verificação no caso de uso: `match.isClosed()` |
| Árbitro não participa de times | Filtro no `TeamFormationService`: exclui `GroupRole.REFEREE` |
| Mensalista com mensalidade paga não paga diária | `SubscriptionPresenceService` verifica `Charge` aprovada no mês antes de exigir pagamento |
| Receita gerada somente para pagamento aprovado | `PaymentApprovedEvent` publicado somente em `approveFromAI/approveManually` |
| Despesa de árbitro gerada uma vez por partida | Constraint `UNIQUE (match_id)` em `referee_assignments` |
