# Diagrama — Modelo de Agregados (UML Classes)

## BC-01: Identity & Access

```mermaid
classDiagram
  class User {
    <<Aggregate Root>>
    +UUID id
    +Name name
    +Email email
    +String passwordHash
    +Phone phone
    +LocalDate birthDate
    +String photoUrl
    +AuthProvider authProvider
    +String googleId
    +Boolean emailVerified
    +Boolean active
    +Instant createdAt
    +Instant updatedAt
    +register(name, email, password, phone, birthDate) User$
    +registerWithGoogle(googleId, email, name) User$
    +updateProfile(name, phone, photoUrl) void
    +changePassword(currentHash, newHash) void
    +verifyEmail() void
    +deactivate() UserDeletedEvent
  }

  class RefreshToken {
    <<Entity>>
    +UUID id
    +UUID userId
    +String tokenHash
    +Instant expiresAt
    +TokenStatus status
    +Instant createdAt
    +revoke() void
    +isValid() Boolean
  }

  class Name {
    <<Value Object>>
    +String value
    +validate() void
  }

  class Email {
    <<Value Object>>
    +String value
    +validate() void
  }

  class Phone {
    <<Value Object>>
    +String value
    +validate() void
  }

  class AuthProvider {
    <<Value Object>>
    LOCAL
    GOOGLE
  }

  class TokenStatus {
    <<Value Object>>
    ACTIVE
    REVOKED
    EXPIRED
  }

  User "1" --> "0..*" RefreshToken : has
  User *-- Name
  User *-- Email
  User *-- Phone
  User *-- AuthProvider
  RefreshToken *-- TokenStatus
```

---

## BC-02: Group Management

```mermaid
classDiagram
  class Group {
    <<Aggregate Root>>
    +UUID id
    +GroupName name
    +Sport sport
    +String description
    +String photoUrl
    +String pixKey
    +GroupStatus status
    +Instant createdAt
    +Instant updatedAt
    +List~GroupMember~ members
    +List~GroupInvite~ invites
    +create(name, sport, creatorUserId) Group$
    +addMember(userId, role) GroupMember
    +removeMember(memberId) void
    +promoteMember(memberId, newRole) void
    +demoteMember(memberId) void
    +generateInvite(createdBy) GroupInvite
    +updateMemberSkill(memberId, skill, source) void
    +banMemberFromPresence(memberId, reason) void
    +unbanMemberFromPresence(memberId) void
    +markAsSubscriber(memberId, amount) void
    +deactivate() void
    +findOwner() GroupMember
    +assertSingleOwner() void
  }

  class GroupMember {
    <<Entity>>
    +UUID id
    +UUID userId
    +GroupRole role
    +Skill skill
    +SkillSource skillSource
    +PlayerPosition position
    +Boolean isSubscriber
    +Money subscriptionAmount
    +Boolean presenceBanned
    +String presenceBanReason
    +Instant joinedAt
    +isOwner() Boolean
    +isAdmin() Boolean
    +isPlayer() Boolean
    +isReferee() Boolean
    +canConfirmPresence() Boolean
  }

  class GroupInvite {
    <<Entity>>
    +UUID id
    +InviteToken token
    +UUID createdBy
    +Instant expiresAt
    +Integer usageCount
    +Integer maxUsages
    +Boolean active
    +isValid() Boolean
    +use() void
    +revoke() void
  }

  class RefereeProfile {
    <<Entity>>
    +UUID id
    +UUID memberId
    +RefereeChargeType chargeType
    +Money amount
    +Boolean active
  }

  class Sport {
    <<Value Object>>
    FOOTBALL
    VOLLEYBALL
    BASKETBALL
    FUTEVOLEI
    BEACH_TENNIS
    OTHER
  }

  class GroupRole {
    <<Value Object>>
    OWNER
    ADMIN
    PLAYER
    REFEREE
  }

  class Skill {
    <<Value Object>>
    +BigDecimal value
    +DEFAULT_SKILL$ 3.0
    +MIN$ 1.0
    +MAX$ 6.0
    +validate() void
  }

  class PlayerPosition {
    <<Value Object>>
    GOALKEEPER
    DEFENDER
    LATERAL
    MIDFIELDER
    FORWARD
    SETTER
    WING_SPIKER
    MIDDLE_BLOCKER
    OPPOSITE
    LIBERO
    POINT_GUARD
    SHOOTING_GUARD
    SMALL_FORWARD
    POWER_FORWARD
    CENTER
    ATTACKER_FV
    DEFENDER_FV
    RIGHT_BT
    LEFT_BT
  }

  class SkillSource {
    <<Value Object>>
    DEFAULT
    VOTING
    MANUAL
  }

  class RefereeChargeType {
    <<Value Object>>
    PER_MATCH
    MONTHLY
  }

  class InviteToken {
    <<Value Object>>
    +String value
    +generate() InviteToken$
  }

  Group "1" *-- "1..*" GroupMember : contains
  Group "1" *-- "0..*" GroupInvite : has
  Group "1" *-- "0..*" RefereeProfile : has
  Group *-- Sport
  GroupMember *-- GroupRole
  GroupMember *-- Skill
  GroupMember *-- PlayerPosition
  GroupMember *-- SkillSource
  RefereeProfile *-- RefereeChargeType
  GroupInvite *-- InviteToken
```

---

## BC-03: Match & Presence

```mermaid
classDiagram
  class Match {
    <<Aggregate Root>>
    +UUID id
    +UUID groupId
    +Instant scheduledAt
    +Instant listClosesAt
    +Location location
    +Integer maxPlayers
    +MatchStatus status
    +PresenceListStatus presenceListStatus
    +UUID createdBy
    +List~PresenceEntry~ presenceEntries
    +List~WaitingEntry~ waitingQueue
    +RefereeAssignment refereeAssignment
    +Instant createdAt
    +create(groupId, scheduledAt, location, maxPlayers, createdBy) Match$
    +confirmPresence(memberId) PresenceEntry
    +declinePresence(memberId) void
    +addToWaitingQueue(memberId) WaitingEntry
    +closePresenceList() PresenceListClosedEvent
    +assignReferee(refereeProfileId) RefereeAssignedEvent
    +submitBannedJustification(memberId, text) void
    +approveJustification(memberId) void
    +rejectJustification(memberId) void
    +notifyNextInQueue() void
    +countConfirmed() Integer
    +hasVacancy() Boolean
    +isClosed() Boolean
  }

  class PresenceEntry {
    <<Entity>>
    +UUID id
    +UUID memberId
    +PresenceStatus status
    +String justification
    +JustificationStatus justificationStatus
    +Instant confirmedAt
    +decline() void
    +submitJustification(text) void
    +approveJustification() void
    +rejectJustification() void
  }

  class WaitingEntry {
    <<Entity>>
    +UUID id
    +UUID memberId
    +Integer position
    +Instant notifiedAt
    +Instant notificationDeadline
    +notify() void
    +isDeadlineExpired() Boolean
  }

  class RefereeAssignment {
    <<Entity>>
    +UUID id
    +UUID refereeProfileId
    +UUID financialEntryId
    +Instant assignedAt
  }

  class Location {
    <<Value Object>>
    +String name
    +String address
  }

  class MatchStatus {
    <<Value Object>>
    SCHEDULED
    COMPLETED
    CANCELLED
  }

  class PresenceListStatus {
    <<Value Object>>
    OPEN
    CLOSED
  }

  class PresenceStatus {
    <<Value Object>>
    CONFIRMED
    DECLINED
    BANNED_PENDING
  }

  class JustificationStatus {
    <<Value Object>>
    PENDING
    APPROVED
    REJECTED
  }

  Match "1" *-- "0..*" PresenceEntry
  Match "1" *-- "0..*" WaitingEntry
  Match "1" *-- "0..1" RefereeAssignment
  Match *-- Location
  Match *-- MatchStatus
  Match *-- PresenceListStatus
  PresenceEntry *-- PresenceStatus
  PresenceEntry *-- JustificationStatus
```

---

## BC-04: Skill Voting

```mermaid
classDiagram
  class SkillVoting {
    <<Aggregate Root>>
    +UUID id
    +UUID groupId
    +VotingStatus status
    +UUID openedBy
    +Instant openedAt
    +Instant deadline
    +Instant closedAt
    +List~Vote~ votes
    +List~VotingBan~ bans
    +open(groupId, openedBy, deadline) SkillVoting$
    +castVote(voterId, targetMemberId, stars) void
    +updateVote(voterId, targetMemberId, stars) void
    +banMember(memberId, reason, bannedBy) void
    +close(closedBy) VotingClosedEvent
    +isOpen() Boolean
    +isMemberBanned(memberId) Boolean
    +calculateResults() Map~UUID, BigDecimal~
    +assertVoterNotBanned(voterId) void
    +assertNotSelfVote(voterId, targetId) void
  }

  class Vote {
    <<Entity>>
    +UUID id
    +UUID voterId
    +UUID targetMemberId
    +Stars stars
    +Instant votedAt
    +update(stars) void
  }

  class VotingBan {
    <<Entity>>
    +UUID id
    +UUID memberId
    +String reason
    +UUID bannedBy
    +Instant bannedAt
  }

  class Stars {
    <<Value Object>>
    +Integer value
    +MIN$ 1
    +MAX$ 6
    +validate() void
  }

  class VotingStatus {
    <<Value Object>>
    OPEN
    CLOSED
  }

  SkillVoting "1" *-- "0..*" Vote
  SkillVoting "1" *-- "0..*" VotingBan
  SkillVoting *-- VotingStatus
  Vote *-- Stars
```

---

## BC-05: Team Formation

```mermaid
classDiagram
  class TeamFormation {
    <<Aggregate Root>>
    +UUID id
    +UUID matchId
    +UUID groupId
    +Integer numberOfTeams
    +FormationType formationType
    +UUID confirmedBy
    +Instant confirmedAt
    +List~Team~ teams
    +create(matchId, groupId, numberOfTeams, teams, confirmedBy) TeamFormation$
    +movePlayer(memberId, fromTeamId, toTeamId) void
    +markAsManuallyAdjusted() void
    +getAverageSkillDeviation() BigDecimal
  }

  class Team {
    <<Entity>>
    +UUID id
    +String name
    +Skill averageSkill
    +List~TeamPlayer~ players
    +addPlayer(memberId, position) void
    +removePlayer(memberId) void
    +recalculateAverageSkill() void
  }

  class TeamPlayer {
    <<Entity>>
    +UUID memberId
    +PlayerPosition position
  }

  class FormationType {
    <<Value Object>>
    AUTOMATIC
    MANUAL_ADJUSTED
  }

  TeamFormation "1" *-- "2..*" Team
  Team "1" *-- "1..*" TeamPlayer
  TeamFormation *-- FormationType
```

---

## BC-06: Payments

```mermaid
classDiagram
  class Charge {
    <<Aggregate Root>>
    +UUID id
    +UUID groupId
    +UUID memberId
    +ChargeType type
    +Money amount
    +UUID referenceMatchId
    +YearMonth referenceMonth
    +ChargeStatus status
    +List~PaymentAttempt~ attempts
    +Instant createdAt
    +createDaily(groupId, memberId, matchId, amount) Charge$
    +createSubscription(groupId, memberId, month, amount) Charge$
    +submitPaymentAttempt(fileKey, contentType) PaymentAttempt
    +approveFromAI(attemptId) PaymentApprovedEvent
    +rejectFromAI(attemptId, reason) void
    +flagForManualReview(attemptId) PaymentRequiresReviewEvent
    +approveManually(attemptId, reviewedBy, note) PaymentApprovedEvent
    +rejectManually(attemptId, reviewedBy, note) void
    +isPending() Boolean
    +isApproved() Boolean
  }

  class PaymentAttempt {
    <<Entity>>
    +UUID id
    +String fileKey
    +String contentType
    +Instant submittedAt
    +ValidationResult validationResult
    +ValidationSource validationSource
    +Instant validatedAt
    +UUID reviewedBy
    +String reviewNote
    +applyAIResult(result, reason) void
    +applyManualReview(result, reviewedBy, note) void
  }

  class ChargeType {
    <<Value Object>>
    DAILY
    SUBSCRIPTION
  }

  class ChargeStatus {
    <<Value Object>>
    PENDING
    APPROVED
    REJECTED
  }

  class ValidationResult {
    <<Value Object>>
    APPROVED
    REJECTED
    MANUAL_REVIEW
  }

  class ValidationSource {
    <<Value Object>>
    AI
    MANUAL
  }

  class Money {
    <<Value Object>>
    +BigDecimal amount
    +validate() void
    +add(other) Money
    +subtract(other) Money
  }

  Charge "1" *-- "0..*" PaymentAttempt
  Charge *-- ChargeType
  Charge *-- ChargeStatus
  Charge *-- Money
  PaymentAttempt *-- ValidationResult
  PaymentAttempt *-- ValidationSource
```

---

## BC-07: Financial

```mermaid
classDiagram
  class FinancialEntry {
    <<Aggregate Root>>
    +UUID id
    +UUID groupId
    +EntryType type
    +EntryCategory category
    +Money amount
    +String description
    +UUID matchId
    +YearMonth referenceMonth
    +UUID sourceChargeId
    +UUID registeredBy
    +Instant registeredAt
    +EntryStatus status
    +String reversalReason
    +Instant reversedAt
    +createRevenue(groupId, category, amount, desc, chargeId) FinancialEntry$
    +createExpense(groupId, category, amount, desc, matchId) FinancialEntry$
    +reverse(reason, reversedBy) void
    +isRevenue() Boolean
    +isExpense() Boolean
    +isActive() Boolean
  }

  class EntryType {
    <<Value Object>>
    REVENUE
    EXPENSE
  }

  class EntryCategory {
    <<Value Object>>
    DAILY
    SUBSCRIPTION
    REFEREE
    COURT
    OTHER
  }

  class EntryStatus {
    <<Value Object>>
    ACTIVE
    REVERSED
  }

  FinancialEntry *-- EntryType
  FinancialEntry *-- EntryCategory
  FinancialEntry *-- EntryStatus
  FinancialEntry *-- Money
```
