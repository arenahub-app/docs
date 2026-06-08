# Fluxos do Sistema — ArenaHub

## FL-001 — Fluxo de Onboarding de Grupo

```
Usuário novo
    │
    ▼
Cadastro / Login Google
    │
    ▼
Dashboard vazio → "Criar Grupo"
    │
    ▼
Informa: nome, modalidade, foto
    │
    ▼
Grupo criado → usuário vira OWNER
    │
    ▼
Sistema gera link de convite
    │
    ├──► Compartilha link com jogadores
    │
    ▼
Jogadores acessam link → fazem cadastro → viram PLAYER
    │
    ▼
OWNER define papéis (ADMIN, REFEREE)
    │
    ▼
Grupo pronto para operação
```

---

## FL-002 — Fluxo de Partida Completa

```
ADMIN cria partida
  (data, hora, local, nº vagas, árbitro opcional)
    │
    ▼
Lista de presença ABERTA
    │
    ├──► Notificação push/email para todos os PLAYERs
    │
    ▼
PLAYERs confirmam ou recusam presença
    │
    ├──► PLAYER banido? → precisa informar justificativa → ADMIN aprova/rejeita
    ├──► Lista cheia? → entra em fila de espera
    │
    ▼
[T - 1 hora] Sistema fecha lista automaticamente
    │
    ├──► Notifica próximo da fila se houver desistência antes do fechamento
    │
    ▼
ADMIN forma times (automático ou ajuste manual)
    │
    ▼
Times publicados → PLAYERs visualizam no app
    │
    ▼
Partida acontece
    │
    ▼
ADMIN registra despesas da partida
    │
    ▼
Sistema consolida financeiro da partida
```

---

## FL-003 — Fluxo de Pagamento com IA

```
ADMIN cria cobrança para PLAYER
  (tipo: diária ou mensalidade, valor, partida)
    │
    ▼
PLAYER recebe notificação de cobrança pendente
    │
    ▼
PLAYER realiza Pix externamente
    │
    ▼
PLAYER abre app → "Registrar Pagamento"
    │
    ▼
PLAYER faz upload do comprovante (foto/PDF)
    │
    ▼
Sistema envia comprovante para serviço de IA
    │
    ▼
IA analisa:
  - Valor confere com cobrança?
  - Chave Pix do grupo está presente?
  - Data é compatível?
    │
    ├──► APROVADO → pagamento confirmado automaticamente
    │                 → PLAYER notificado
    │                 → financeiro atualizado
    │
    ├──► REPROVADO → PLAYER notificado
    │                 → motivo exibido
    │                 → PLAYER pode reenviar comprovante
    │
    └──► REVISÃO MANUAL → ADMIN notificado
                          → ADMIN visualiza comprovante
                          → ADMIN aprova ou reprova
                          → PLAYER notificado do resultado
```

---

## FL-004 — Fluxo de Votação de Habilidades

```
ADMIN abre votação
  (define prazo; opcionalmente bane jogadores)
    │
    ▼
Sistema notifica PLAYERs habilitados
    │
    ▼
PLAYER acessa votação
    │
    ▼
Para cada colega (exceto si mesmo):
  Atribui de 1 a 6 estrelas
    │
    ▼
PLAYER salva votos
    │
    ▼
[Prazo encerrado OU ADMIN encerra manualmente]
    │
    ▼
Sistema calcula média por jogador
  (excluindo votos de banidos)
    │
    ▼
Skill atualizado para cada jogador
    │
    ▼
Resultados publicados (visíveis a todos os membros)
```

---

## FL-005 — Fluxo de Formação de Times

```
Lista de presença fechada
    │
    ▼
ADMIN acessa "Formar Times"
    │
    ▼
Define: número de times
    │
    ▼
Sistema coleta:
  - Jogadores confirmados (exceto árbitros)
  - Skill de cada jogador
  - Posição preferencial de cada jogador
    │
    ▼
Algoritmo de distribuição:
  1. Ordena jogadores por skill DESC
  2. Distribui em snake draft entre os times
  3. Verifica cobertura de posições por modalidade
  4. Calcula skill médio por time
  5. Se desvio padrão > threshold → realoca para equilibrar
    │
    ▼
Times propostos exibidos ao ADMIN
    │
    ▼
ADMIN pode:
  ├──► Aceitar formação proposta
  ├──► Mover jogador entre times manualmente
  └──► Solicitar nova formação automática
    │
    ▼
Formação confirmada → salva no histórico da partida
    │
    ▼
PLAYERs visualizam seus times no app
```

---

## FL-006 — Fluxo Financeiro Mensal

```
Início do mês
    │
    ▼
Sistema gera cobranças automáticas para mensalistas
    │
    ▼
Ao longo do mês:
  PLAYERs pagam diárias por partida
  Mensalistas confirmam presença sem pagar por partida
    │
    ▼
ADMIN registra despesas:
  - Quadra
  - Árbitros (calculado automaticamente por associação)
  - Outras
    │
    ▼
ADMIN acessa relatório mensal:
  - Total de receitas (diárias + mensalidades)
  - Total de despesas
  - Saldo do período
  - Inadimplentes
    │
    ▼
Exportação de relatório (PDF ou planilha)
```

---

## FL-007 — Fluxo de Banimento de Presença com Justificativa

```
Jogador está banido de confirmar presença
    │
    ▼
Jogador tenta confirmar presença em nova partida
    │
    ▼
Sistema detecta banimento ativo
    │
    ▼
Exibe formulário: "Informe a justificativa"
    │
    ▼
Jogador envia justificativa
    │
    ▼
ADMIN recebe notificação para análise
    │
    ├──► ADMIN aprova → jogador é adicionado à lista da partida
    │                 → banimento mantido (regra de comportamento)
    │
    └──► ADMIN rejeita → jogador não entra na lista
                       → recebe notificação com motivo
```

---

## Mapa de Estados — Presença em Partida

```
                    [ABERTA]
                       │
         ┌─────────────┼─────────────┐
         ▼             ▼             ▼
    [CONFIRMADO]  [RECUSADO]   [FILA_ESPERA]
         │
         ├──► Desistência → [RECUSADO] → próximo da fila notificado
         │
[T-1h] Lista fecha automaticamente
         │
         ▼
      [FECHADA]
         │
         ▼
    [TIMES_FORMADOS]
```

---

## Mapa de Estados — Pagamento

```
[PENDENTE]
    │
    ▼
[AGUARDANDO_VALIDACAO] ── comprovante enviado
    │
    ├──► IA: APROVADO → [APROVADO]
    ├──► IA: REPROVADO → [REPROVADO] → jogador pode reenviar → volta para AGUARDANDO_VALIDACAO
    └──► IA: REVISAO_MANUAL → [EM_REVISAO]
                                    │
                                    ├──► Admin: aprova → [APROVADO]
                                    └──► Admin: reprova → [REPROVADO]
```
