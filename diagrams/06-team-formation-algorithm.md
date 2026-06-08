# Diagrama — Algoritmo de Formação de Times

## Visão Geral

O algoritmo usa **Snake Draft** como base, com ajuste de equilíbrio de skill médio entre times.

---

## Fluxograma do Algoritmo

```
INPUT:
  players: List<{memberId, skill, position}>  (apenas PLAYER e ADMIN confirmados, sem REFEREE)
  numberOfTeams: Int (N ≥ 2)
  sport: Sport

──────────────────────────────────────────────

STEP 1 — Validação
  ┌─ Partida com lista fechada? ──► NÃO → lança DomainException
  └─ N ≥ 2? ──────────────────── NÃO → lança DomainException

──────────────────────────────────────────────

STEP 2 — Ordenação
  Ordena players por skill DESC
  Exemplo (10 jogadores, N=2):
    [6.0, 5.5, 5.0, 4.5, 4.0, 3.5, 3.0, 2.5, 2.0, 1.5]

──────────────────────────────────────────────

STEP 3 — Snake Draft
  Distribui em zig-zag entre times:

  Round 1 (→):  Time A=6.0  Time B=5.5
  Round 2 (←):  Time B=5.0  Time A=4.5
  Round 3 (→):  Time A=4.0  Time B=3.5
  Round 4 (←):  Time B=3.0  Time A=2.5
  Round 5 (→):  Time A=2.0  Time B=1.5

  Resultado:
    Time A: [6.0, 4.5, 4.0, 2.5, 2.0] → média = 3.8
    Time B: [5.5, 5.0, 3.5, 3.0, 1.5] → média = 3.7

──────────────────────────────────────────────

STEP 4 — Verificação de Equilíbrio
  Calcula desvio padrão dos skill médios entre times

  |3.8 - 3.7| = 0.1 ≤ 0.5 → APROVADO ✓

  SE desvio > 0.5:
    → Aplica SwapOptimizer: troca pares de jogadores
      de times diferentes que reduzam o desvio
    → Máximo de 10 iterações
    → Se após 10 iterações ainda > 0.5: retorna melhor resultado obtido
       (registrado como "melhor possível" no log)

──────────────────────────────────────────────

STEP 5 — Distribuição de Posições (Critério Secundário)
  Para cada modalidade, verifica cobertura mínima de posições críticas:

  Futebol (por time):
    GOL: mínimo 1 (se disponível)
    DEF: mínimo 1 (ZAG ou LAT)

  Vôlei (por time):
    LEV: mínimo 1 (se disponível)
    LIB: mínimo 1 (se disponível)

  Basquete (por time):
    ARM: mínimo 1 (se disponível)

  Algoritmo:
    1. Para cada posição crítica da modalidade:
       a. Verifica se o time tem a posição coberta
       b. SE NÃO: busca jogador com essa posição em outro time
          que possa ser trocado sem aumentar desvio além de 0.5
       c. SE encontrou: realiza a troca
    2. Marca formationType como AUTOMATIC
       (ou MANUAL_ADJUSTED se ADMIN editar depois)

──────────────────────────────────────────────

STEP 6 — Caso jogadores não divisíveis igualmente
  Se len(players) % N != 0:
    Times com mais jogadores recebem os primeiros da lista
    Exemplo: 11 jogadores, N=2 → Time A=6 jogadores, Time B=5
    O snake draft distribui normalmente; o 11º vai para o Time A

──────────────────────────────────────────────

OUTPUT:
  TeamFormationResult {
    teams: List<{name, players: List<{memberId, skill, position}>, averageSkill}>
    skillDeviation: BigDecimal
    positionCoverageScore: Map<Sport, Map<Position, Boolean>>
  }
```

---

## Exemplo Detalhado — Futebol (10 jogadores, 2 times)

```
Jogadores (ordenados por skill):
  P1: 6.0 (GOL)   P2: 5.5 (ATA)   P3: 5.0 (ZAG)   P4: 4.5 (MEI)
  P5: 4.0 (LAT)   P6: 3.5 (VOL)   P7: 3.0 (ATA)   P8: 2.5 (ZAG)
  P9: 2.0 (LAT)   P10: 1.5 (MEI)

Snake Draft (N=2):
  Round 1 →:  Time A ← P1(6.0/GOL)    Time B ← P2(5.5/ATA)
  Round 2 ←:  Time B ← P3(5.0/ZAG)    Time A ← P4(4.5/MEI)
  Round 3 →:  Time A ← P5(4.0/LAT)    Time B ← P6(3.5/VOL)
  Round 4 ←:  Time B ← P7(3.0/ATA)    Time A ← P8(2.5/ZAG)
  Round 5 →:  Time A ← P9(2.0/LAT)    Time B ← P10(1.5/MEI)

Time A: P1, P4, P5, P8, P9 → avg = (6.0+4.5+4.0+2.5+2.0)/5 = 3.8
Time B: P2, P3, P6, P7, P10 → avg = (5.5+5.0+3.5+3.0+1.5)/5 = 3.7

Desvio = |3.8 - 3.7| = 0.1 ✓

Verificação de posições (Futebol):
  Time A: GOL=✓ (P1), DEF=✓ (P8/ZAG, P5/LAT)
  Time B: GOL=✗ (sem goleiro!)
    → Busca goleiro em Time A para trocar sem quebrar equilíbrio
    → P1 (GOL, skill=6.0) só pode trocar com jogador de skill similar
    → Não há outro goleiro disponível
    → Registra: "Cobertura de goleiro indisponível no Time B" no log
    → Retorna formação como está (algoritmo faz o melhor possível)
```

---

## Pseudocódigo (Domain Service)

```
class TeamFormationService {

  fun form(players, numberOfTeams, sport): TeamFormationResult {

    // STEP 1 — sort
    val sorted = players.sortedByDescending { it.skill }

    // STEP 2 — snake draft
    val teams = (1..numberOfTeams).map { Team(name="Time ${it.toLabel()}") }
    var direction = 1
    var teamIndex = 0

    for (player in sorted) {
      teams[teamIndex].add(player)
      teamIndex += direction
      if (teamIndex == numberOfTeams) { direction = -1; teamIndex = numberOfTeams - 1 }
      else if (teamIndex == -1)       { direction = +1; teamIndex = 0 }
    }

    // STEP 3 — rebalance if needed
    var deviation = calculateDeviation(teams)
    var iterations = 0
    while (deviation > 0.5 && iterations < 10) {
      swapToMinimizeDeviation(teams)
      deviation = calculateDeviation(teams)
      iterations++
    }

    // STEP 4 — position coverage
    val coverage = enforcePositionCoverage(teams, sport, maxDeviationIncrease = 0.5)

    return TeamFormationResult(teams, deviation, coverage)
  }

  private fun calculateDeviation(teams): BigDecimal {
    val averages = teams.map { it.averageSkill() }
    val mean = averages.sum() / averages.size
    return averages.maxOf { abs(it - mean) }
  }

  private fun swapToMinimizeDeviation(teams): Unit {
    // Para cada par de times, tenta trocar jogadores de skill diferente
    // que reduzam o desvio padrão. Aplica a melhor troca encontrada.
    val bestSwap = findBestSwap(teams)
    bestSwap?.apply()
  }
}
```

---

## Posições Críticas por Modalidade

| Modalidade | Posições Críticas (mínimo 1 por time se disponível) |
|---|---|
| Futebol | Goleiro (GOL) |
| Vôlei | Levantador (LEV), Líbero (LIB) |
| Basquete | Armador (ARM) |
| Futevôlei | Sem críticas definidas |
| Beach Tennis | Sem críticas definidas |
| Outros | Sem críticas definidas |

*"Se disponível" significa: se há ao menos 1 jogador com essa posição entre os confirmados. Não há falha se a posição não existir na partida.*
