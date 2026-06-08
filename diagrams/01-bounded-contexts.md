# Diagrama — Bounded Contexts

## Context Map

```mermaid
C4Context
  title ArenaHub — Bounded Contexts

  Person(player, "PLAYER", "Jogador do grupo")
  Person(admin, "ADMIN/OWNER", "Administrador do grupo")
  Person(referee, "REFEREE", "Árbitro")

  System_Boundary(arenahub, "ArenaHub") {

    Boundary(identity, "Identity & Access") {
      Component(user, "User", "Cadastro, autenticação, perfil")
    }

    Boundary(group_mgmt, "Group Management") {
      Component(group, "Group", "Grupos, membros, papéis, skill, convites, árbitros")
    }

    Boundary(match_presence, "Match & Presence") {
      Component(match, "Match", "Partidas, lista de presença, fila de espera, banimentos")
    }

    Boundary(skill_voting, "Skill Voting") {
      Component(voting, "SkillVoting", "Votações, votos, banimentos de votação")
    }

    Boundary(team_formation, "Team Formation") {
      Component(teams, "TeamFormation", "Formação e histórico de times")
    }

    Boundary(payments_bc, "Payments") {
      Component(charge, "Charge", "Cobranças, comprovantes, validação por IA")
    }

    Boundary(financial_bc, "Financial") {
      Component(financial, "FinancialEntry", "Receitas, despesas, saldo, relatórios")
    }
  }

  System_Ext(google, "Google OAuth2", "Autenticação social")
  System_Ext(r2, "Cloudflare R2", "Storage de arquivos")
  System_Ext(ai, "Serviço de IA", "Validação de comprovantes")

  Rel(player, user, "Cadastra e autentica")
  Rel(admin, group, "Gerencia grupo e membros")
  Rel(player, match, "Confirma presença")
  Rel(player, voting, "Vota em colegas")
  Rel(admin, teams, "Forma times")
  Rel(player, charge, "Envia comprovante")
  Rel(admin, financial, "Visualiza relatórios")

  Rel(user, google, "OAuth2 login")
  Rel(charge, r2, "Upload de comprovante")
  Rel(charge, ai, "Validação de comprovante")

  Rel(voting, group, "VotingClosedEvent → atualiza skill")
  Rel(charge, financial, "PaymentApprovedEvent → gera receita")
  Rel(match, financial, "RefereeAssignedEvent → gera despesa")
```

---

## Relacionamentos entre Contextos

```
┌─────────────────────────────────────────────────────────┐
│              Fluxo de Dados entre Contextos             │
└─────────────────────────────────────────────────────────┘

Identity & Access
      │
      │  userId (referência)
      ▼
Group Management ◄──────────────────────────────────────────┐
      │                                                      │
      │  groupId + memberId (referência)                     │ VotingClosedEvent
      ▼                                                      │ (atualiza skill)
Match & Presence ─────────────────────► Skill Voting ────────┘
      │
      │  matchId (referência)
      ├──────────────────────────────► Team Formation
      │
      │  RefereeAssignedEvent
      ▼
Financial ◄─────────────────────────── Payments
                                          (PaymentApprovedEvent)
```

---

## Linguagem Ubíqua por Contexto

| Contexto | Termos Chave |
|---|---|
| Identity & Access | usuário, credencial, token, sessão, login, OAuth |
| Group Management | grupo, membro, papel, skill, convite, mensalista, árbitro, banimento |
| Match & Presence | partida, presença, confirmação, recusa, lista, vaga, fila, fechamento |
| Skill Voting | votação, voto, estrelas, prazo, banimento de votação |
| Team Formation | formação, time, snake draft, equilíbrio, posição |
| Payments | cobrança, comprovante, diária, mensalidade, validação, revisão |
| Financial | receita, despesa, saldo, relatório, estorno, categoria |
