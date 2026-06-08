# Diagrama — Sequências de Casos de Uso

## UC-006 — Confirmar Presença (fluxo completo)

```mermaid
sequenceDiagram
  actor Player
  participant API as REST API
  participant UC as ConfirmPresenceUseCase
  participant GroupSvc as GroupMembershipService
  participant MatchRepo as MatchRepository
  participant Notif as NotificationPort

  Player->>API: POST /groups/{groupId}/matches/{matchId}/presence
  API->>UC: execute(groupId, matchId, currentUserId)

  UC->>GroupSvc: assertMember(groupId, currentUserId)
  alt Não é membro
    GroupSvc-->>UC: ForbiddenException
    UC-->>API: 403 Forbidden
    API-->>Player: 403 Forbidden
  end

  UC->>MatchRepo: findByIdAndGroupId(matchId, groupId)
  alt Partida não encontrada
    MatchRepo-->>UC: Optional.empty()
    UC-->>API: NotFoundException
    API-->>Player: 404 Not Found
  end

  UC->>UC: match.isClosed()?
  alt Lista fechada
    UC-->>API: DomainException("Lista de presença fechada")
    API-->>Player: 422 Unprocessable Entity
  end

  UC->>UC: member.presenceBanned?
  alt Jogador banido
    UC-->>API: DomainException("Informe justificativa")
    API-->>Player: 422 com instrução de justificativa
  end

  UC->>UC: match.hasVacancy()?
  alt Sem vagas
    UC->>MatchRepo: addToWaitingQueue(match, memberId)
    MatchRepo-->>UC: WaitingEntry (posição N)
    UC-->>API: 200 {status: "WAITING_QUEUE", position: N}
    API-->>Player: Na fila, posição N
  else Com vagas
    UC->>MatchRepo: confirmPresence(match, memberId)
    MatchRepo-->>UC: PresenceEntry
    UC-->>API: 200 {status: "CONFIRMED"}
    API-->>Player: Presença confirmada
  end
```

---

## UC-012 — Registrar Pagamento com Validação por IA

```mermaid
sequenceDiagram
  actor Player
  participant API as REST API
  participant StorageUC as GenerateUploadUrlUseCase
  participant R2 as Cloudflare R2
  participant SubmitUC as SubmitPaymentAttemptUseCase
  participant ChargeRepo as ChargeRepository
  participant AIPort as AIValidationPort
  participant FinancialRepo as FinancialRepository
  participant Notif as NotificationPort

  Player->>API: POST /storage/upload-url {chargeId, contentType}
  API->>StorageUC: execute(chargeId, contentType, userId)
  StorageUC->>StorageUC: valida MIME type e tamanho
  StorageUC->>R2: generatePresignedPutUrl(fileKey, 5min)
  R2-->>StorageUC: uploadUrl
  StorageUC-->>API: {uploadUrl, fileKey}
  API-->>Player: {uploadUrl, fileKey}

  Player->>R2: PUT {uploadUrl} com arquivo
  R2-->>Player: 200 OK

  Player->>API: POST /charges/{chargeId}/payment-attempts {fileKey}
  API->>SubmitUC: execute(chargeId, fileKey, userId)
  SubmitUC->>ChargeRepo: findByIdAndGroupId(chargeId, groupId)
  SubmitUC->>ChargeRepo: addAttempt(charge, fileKey)

  SubmitUC->>AIPort: validate(fileKey, expectedAmount, pixKey)
  Note over SubmitUC,AIPort: Timeout: 10 segundos

  alt IA: APPROVED
    AIPort-->>SubmitUC: ValidationResult.APPROVED
    SubmitUC->>ChargeRepo: approveFromAI(charge, attemptId)
    SubmitUC->>FinancialRepository: createRevenue(groupId, amount, chargeId)
    SubmitUC->>Notif: notifyPlayer("Pagamento aprovado")
    SubmitUC-->>API: {status: APPROVED}
  else IA: REJECTED
    AIPort-->>SubmitUC: ValidationResult.REJECTED{reason}
    SubmitUC->>ChargeRepo: rejectFromAI(charge, attemptId, reason)
    SubmitUC->>Notif: notifyPlayer("Pagamento reprovado: " + reason)
    SubmitUC-->>API: {status: REJECTED, reason}
  else Timeout ou MANUAL_REVIEW
    AIPort-->>SubmitUC: ValidationResult.MANUAL_REVIEW
    SubmitUC->>ChargeRepo: flagForManualReview(charge, attemptId)
    SubmitUC->>Notif: notifyAdmins("Comprovante aguarda revisão")
    SubmitUC-->>API: {status: MANUAL_REVIEW}
  end

  API-->>Player: resultado
```

---

## UC-016 — Encerrar Votação e Atualizar Skills

```mermaid
sequenceDiagram
  actor Admin
  participant API as REST API
  participant UC as CloseVotingUseCase
  participant VotingRepo as VotingRepository
  participant SkillSvc as SkillCalculationService
  participant GroupRepo as GroupRepository
  participant EventBus as DomainEventBus

  Admin->>API: POST /groups/{groupId}/votings/{votingId}/close
  API->>UC: execute(groupId, votingId, adminId)

  UC->>VotingRepo: findByIdAndGroupId(votingId, groupId)
  UC->>UC: voting.isOpen()?
  alt Votação já fechada
    UC-->>API: DomainException
    API-->>Admin: 422 "Votação já encerrada"
  end

  UC->>UC: voting.close(adminId)
  UC->>VotingRepo: save(voting)

  UC->>SkillSvc: calculateResults(voting)
  Note over SkillSvc: Para cada target membro:
  Note over SkillSvc: média = Σ(stars de voters não banidos) / count
  SkillSvc-->>UC: Map<memberId, avgSkill>

  loop Para cada membro com resultado
    UC->>GroupRepo: updateMemberSkill(groupId, memberId, avgSkill, VOTING)
  end

  UC->>EventBus: publish(VotingClosedEvent{groupId, results})
  UC-->>API: {closedAt, results: [{memberId, previousSkill, newSkill}]}
  API-->>Admin: 200 OK com resultados
```

---

## UC-024 — Formar Times

```mermaid
sequenceDiagram
  actor Admin
  participant API as REST API
  participant UC as FormTeamsUseCase
  participant MatchRepo as MatchRepository
  participant GroupRepo as GroupRepository
  participant TeamSvc as TeamFormationService
  participant FormRepo as TeamFormationRepository

  Admin->>API: POST /groups/{groupId}/matches/{matchId}/team-formations
  Note right of Admin: body: {numberOfTeams: 2}
  API->>UC: execute(groupId, matchId, numberOfTeams, adminId)

  UC->>MatchRepo: findByIdAndGroupId(matchId, groupId)
  UC->>UC: match.isClosed()?
  alt Lista ainda aberta
    UC-->>API: DomainException
    API-->>Admin: 422 "Lista de presença ainda aberta"
  end

  UC->>MatchRepo: getConfirmedPlayerIds(matchId)
  UC->>GroupRepo: getMembersWithSkill(groupId, playerIds)
  Note over GroupRepo: Exclui membros com role=REFEREE

  UC->>TeamSvc: form(players, numberOfTeams, group.sport)
  TeamSvc-->>UC: TeamFormationResult{teams, deviation}

  UC->>FormRepo: save(TeamFormation{matchId, groupId, teams, confirmedBy=adminId})
  FormRepo-->>UC: TeamFormation salvo

  UC-->>API: TeamFormationResponse{teams, skillDeviation}
  API-->>Admin: 200 com times formados

  Note over Admin: Admin pode ajustar manualmente via PATCH
```

---

## Scheduler — Fechamento Automático de Lista

```mermaid
sequenceDiagram
  participant Scheduler
  participant UC as ClosePresenceListsUseCase
  participant MatchRepo as MatchRepository
  participant Notif as NotificationPort

  Note over Scheduler: Executa a cada 1 minuto (cron)

  Scheduler->>UC: execute(now)
  UC->>MatchRepo: findMatchesToClose(now)
  Note over MatchRepo: WHERE list_closes_at <= now
  Note over MatchRepo: AND presence_list_status = 'OPEN'

  loop Para cada partida a fechar
    UC->>UC: match.closePresenceList()
    UC->>MatchRepo: save(match)
    UC->>MatchRepo: getWaitingQueue(match.id)
    loop Para cada waiting entry
      UC->>Notif: notifyPlayer(memberId, "Você não entrou na lista")
    end
  end
```
