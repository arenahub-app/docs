# Diagrama — Máquinas de Estado

## Match.presenceListStatus

```mermaid
stateDiagram-v2
  [*] --> OPEN : Partida criada

  OPEN --> OPEN : Jogador confirma presença
  OPEN --> OPEN : Jogador recusa presença
  OPEN --> OPEN : Jogador entra na fila
  OPEN --> OPEN : Vaga liberada → próximo da fila notificado

  OPEN --> CLOSED : Scheduler (T-1h antes da partida)
  OPEN --> CLOSED : ADMIN fecha manualmente

  CLOSED --> CLOSED : ADMIN adiciona/remove manualmente
  CLOSED --> CLOSED : Times formados
```

## Match.status

```mermaid
stateDiagram-v2
  [*] --> SCHEDULED : Partida criada

  SCHEDULED --> COMPLETED : ADMIN registra conclusão
  SCHEDULED --> CANCELLED : ADMIN cancela

  COMPLETED --> [*]
  CANCELLED --> [*]
```

## Charge.status

```mermaid
stateDiagram-v2
  [*] --> PENDING : Cobrança criada

  PENDING --> PENDING : Tentativa de pagamento enviada\n(aguarda validação)

  PENDING --> APPROVED : Aprovação automática (IA)\nou manual (ADMIN)
  PENDING --> REJECTED : Reprovação definitiva (ADMIN)

  APPROVED --> [*]
  REJECTED --> PENDING : PLAYER reenvia comprovante
```

## PaymentAttempt — Validação

```mermaid
stateDiagram-v2
  [*] --> SUBMITTED : Comprovante enviado

  SUBMITTED --> AI_APPROVED : IA aprova
  SUBMITTED --> AI_REJECTED : IA reprova
  SUBMITTED --> MANUAL_REVIEW : IA timeout ou incerto

  AI_APPROVED --> [*] : Charge → APPROVED
  AI_REJECTED --> [*] : PLAYER notificado
  MANUAL_REVIEW --> ADMIN_APPROVED : ADMIN aprova
  MANUAL_REVIEW --> ADMIN_REJECTED : ADMIN reprova

  ADMIN_APPROVED --> [*] : Charge → APPROVED
  ADMIN_REJECTED --> [*] : PLAYER notificado
```

## SkillVoting.status

```mermaid
stateDiagram-v2
  [*] --> OPEN : ADMIN abre votação

  OPEN --> OPEN : Jogador vota
  OPEN --> OPEN : Jogador altera voto
  OPEN --> OPEN : ADMIN bane jogador da votação

  OPEN --> CLOSED : Prazo atingido (scheduler)
  OPEN --> CLOSED : ADMIN encerra manualmente

  CLOSED --> [*] : Skills atualizados via VotingClosedEvent
```

## PresenceEntry.status

```mermaid
stateDiagram-v2
  [*] --> CONFIRMED : Jogador confirma\n(lista aberta, não banido)
  [*] --> DECLINED : Jogador recusa
  [*] --> BANNED_PENDING : Jogador banido tenta confirmar\n(submete justificativa)

  CONFIRMED --> DECLINED : Jogador cancela\n(lista ainda aberta)

  BANNED_PENDING --> CONFIRMED : ADMIN aprova justificativa\n(se há vaga)
  BANNED_PENDING --> DECLINED : ADMIN rejeita justificativa
```

## WaitingEntry — Ciclo de Vida

```mermaid
stateDiagram-v2
  [*] --> IN_QUEUE : Lista cheia ao confirmar

  IN_QUEUE --> NOTIFIED : Vaga aberta → notificado
  IN_QUEUE --> REMOVED : Lista fechada sem vaga

  NOTIFIED --> CONFIRMED : Jogador confirma em 30 min
  NOTIFIED --> EXPIRED : 30 min sem confirmação → próximo notificado

  CONFIRMED --> [*]
  REMOVED --> [*]
  EXPIRED --> [*]
```

## GroupMember — Banimento de Presença

```mermaid
stateDiagram-v2
  [*] --> NOT_BANNED : Membro entra no grupo

  NOT_BANNED --> BANNED : ADMIN aplica banimento\n(com motivo)

  BANNED --> NOT_BANNED : ADMIN remove banimento

  state BANNED {
    [*] --> CANNOT_CONFIRM
    CANNOT_CONFIRM --> JUSTIFICATION_SUBMITTED : Membro tenta confirmar\ne submete justificativa
    JUSTIFICATION_SUBMITTED --> CANNOT_CONFIRM : ADMIN rejeita\n(por partida específica)
    JUSTIFICATION_SUBMITTED --> TEMPORARILY_ALLOWED : ADMIN aprova\n(por partida específica)
    TEMPORARILY_ALLOWED --> CANNOT_CONFIRM : Próxima partida
  }
```

## FinancialEntry.status

```mermaid
stateDiagram-v2
  [*] --> ACTIVE : Entrada criada

  ACTIVE --> REVERSED : ADMIN estorna\n(com motivo obrigatório)

  REVERSED --> [*]

  note right of ACTIVE : Não é excluída,\napenas estornada
```
