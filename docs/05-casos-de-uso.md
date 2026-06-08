# Casos de Uso — ArenaHub

## Diagrama de Atores

```
Atores Primários:
  - OWNER (dono do grupo)
  - ADMIN (administrador)
  - PLAYER (jogador)
  - REFEREE (árbitro)

Atores Secundários:
  - Sistema de IA (validação de comprovantes)
  - Google OAuth2 (autenticação)
  - Cloudflare R2 (armazenamento de arquivos)
```

---

## UC-001 — Cadastrar Usuário

**Ator principal:** Visitante  
**Pré-condições:** Nenhuma  
**Fluxo principal:**
1. Usuário acessa a plataforma e clica em "Cadastrar"
2. Informa nome, email, senha, telefone e data de nascimento
3. Confirma email via link enviado
4. Sistema cria conta e redireciona para dashboard

**Fluxo alternativo A — OAuth2 Google:**
1. Usuário clica em "Entrar com Google"
2. Google autentica e retorna dados básicos
3. Sistema cria conta com dados do Google e solicita telefone complementar

**Pós-condições:** Usuário autenticado com sessão ativa

---

## UC-002 — Criar Grupo Esportivo

**Ator principal:** PLAYER (torna-se OWNER)  
**Pré-condições:** Usuário autenticado  
**Fluxo principal:**
1. Usuário acessa "Criar Grupo"
2. Informa nome, modalidade, descrição e foto (opcional)
3. Sistema cria o grupo e atribui papel OWNER ao criador
4. Sistema gera link de convite único

**Pós-condições:** Grupo criado; criador é OWNER

---

## UC-003 — Convidar Jogador

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Grupo criado  
**Fluxo principal:**
1. Admin acessa "Membros" > "Convidar"
2. Informa email ou compartilha link de convite
3. Convidado recebe email ou acessa link
4. Convidado aceita convite; recebe papel PLAYER

**Fluxo alternativo A — Convite por link:**
1. Admin copia link de convite
2. Compartilha via WhatsApp ou outro canal
3. Novo membro acessa link, faz login/cadastro e é adicionado ao grupo

---

## UC-004 — Gerenciar Papéis

**Ator principal:** OWNER  
**Pré-condições:** Membro pertence ao grupo  
**Fluxo principal:**
1. OWNER acessa "Membros"
2. Seleciona membro e escolhe novo papel (ADMIN, PLAYER, REFEREE)
3. Sistema atualiza papel do membro

**Regra:** Somente o OWNER pode promover a ADMIN; OWNER não pode ser rebaixado

---

## UC-005 — Criar Partida

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Grupo ativo  
**Fluxo principal:**
1. Admin acessa "Partidas" > "Nova Partida"
2. Informa data, horário, local e número máximo de jogadores
3. Opcionalmente associa árbitro
4. Sistema cria partida e abre lista de presença
5. Notifica todos os membros PLAYER do grupo

**Pós-condições:** Partida criada; lista de presença aberta

---

## UC-006 — Confirmar Presença

**Ator principal:** PLAYER  
**Pré-condições:** Partida criada; lista aberta; jogador não banido  
**Fluxo principal:**
1. PLAYER recebe notificação de abertura de lista
2. Acessa a partida e clica em "Confirmar Presença"
3. Sistema registra confirmação e atualiza lista

**Fluxo alternativo A — Jogador banido:**
1. Jogador tenta confirmar
2. Sistema exibe aviso de banimento e campo de justificativa
3. Jogador informa justificativa e solicita aprovação
4. Admin aprova ou rejeita a solicitação

**Fluxo alternativo B — Lista cheia:**
1. Jogador acessa lista já com vagas esgotadas
2. Sistema adiciona jogador à fila de espera

**Regra:** Lista fecha automaticamente 1 hora antes da partida

---

## UC-007 — Recusar Presença

**Ator principal:** PLAYER  
**Pré-condições:** Jogador confirmado; lista ainda aberta  
**Fluxo principal:**
1. PLAYER acessa partida
2. Clica em "Cancelar Presença"
3. Sistema remove da lista e notifica próximo na fila de espera

---

## UC-008 — Banir Jogador de Lista de Presença

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Jogador pertence ao grupo  
**Fluxo principal:**
1. Admin acessa perfil do jogador
2. Acessa "Banimento de Presença" > "Banir"
3. Informa motivo do banimento
4. Sistema registra banimento com data e motivo
5. Jogador é notificado

**Pós-condições:** Jogador não pode confirmar presença sem aprovação do admin

---

## UC-009 — Abrir Votação de Habilidades

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Grupo ativo com ao menos 2 membros PLAYER  
**Fluxo principal:**
1. Admin acessa "Votações" > "Nova Votação"
2. Define prazo de encerramento
3. Optionally bane jogadores específicos da votação
4. Sistema abre votação e notifica jogadores habilitados

---

## UC-010 — Votar em Habilidade

**Ator principal:** PLAYER  
**Pré-condições:** Votação aberta; jogador não banido da votação  
**Fluxo principal:**
1. PLAYER acessa "Votações" > votação ativa
2. Para cada colega elegível, atribui nota de 1 a 6 estrelas
3. Sistema registra votos (anônimos para outros players)

**Regra:** Jogador não vota em si mesmo; voto é anônimo entre players

---

## UC-011 — Encerrar Votação

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Votação aberta  
**Fluxo principal:**
1. Admin encerra manualmente OU sistema encerra automaticamente no prazo
2. Sistema calcula média das notas por jogador
3. Sistema atualiza o skill de cada jogador com a média calculada
4. Resultados ficam visíveis para todos os membros

---

## UC-012 — Registrar Pagamento

**Ator principal:** PLAYER  
**Pré-condições:** Cobrança pendente para o jogador  
**Fluxo principal:**
1. PLAYER acessa "Pagamentos"
2. Visualiza cobrança pendente (diária ou mensalidade)
3. Realiza Pix externamente
4. Faz upload do comprovante no sistema
5. Sistema envia comprovante para validação por IA
6. IA retorna status: Aprovado / Reprovado / Revisão Manual
7. Se Aprovado: pagamento confirmado automaticamente
8. Se Revisão Manual: Admin recebe alerta para verificar

---

## UC-013 — Aprovar Pagamento Manualmente

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Pagamento em status "Revisão Manual" ou "Reprovado"  
**Fluxo principal:**
1. Admin acessa "Pagamentos" > fila de revisão
2. Visualiza comprovante e dados do pagamento
3. Aprova ou reprova manualmente
4. Sistema atualiza status e notifica jogador

---

## UC-014 — Registrar Árbitro

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Usuário existente no sistema  
**Fluxo principal:**
1. Admin acessa "Árbitros" > "Cadastrar"
2. Busca usuário por nome ou email
3. Define tipo de cobrança (por partida ou mensal) e valor
4. Sistema registra árbitro vinculado ao grupo

---

## UC-015 — Associar Árbitro a Partida

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Árbitro cadastrado no grupo; partida criada  
**Fluxo principal:**
1. Admin acessa partida > "Árbitro"
2. Seleciona árbitro disponível
3. Sistema registra associação e notifica árbitro
4. Sistema registra despesa de arbitragem no financeiro da partida

---

## UC-016 — Formar Times Automaticamente

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Lista de presença fechada; jogadores têm skill definido  
**Fluxo principal:**
1. Admin acessa partida > "Formar Times"
2. Define número de times
3. Sistema distribui jogadores por skill e posição
4. Admin visualiza times propostos
5. Admin pode ajustar manualmente e confirmar
6. Sistema salva formação no histórico da partida

**Algoritmo:**
- Ordena jogadores por skill (desc)
- Distribui em snake draft (1-2-3..3-2-1) entre os times
- Verifica cobertura mínima de posições críticas por modalidade
- Recalcula se desvio padrão de skill médio por time exceder threshold

---

## UC-017 — Registrar Despesa

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Grupo ativo  
**Fluxo principal:**
1. Admin acessa "Financeiro" > "Nova Despesa"
2. Informa categoria (árbitro, quadra, outra), valor e descrição
3. Opcionalmente vincula a uma partida
4. Sistema registra e atualiza saldo

---

## UC-018 — Visualizar Relatório Financeiro

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Grupo com movimentações registradas  
**Fluxo principal:**
1. Admin acessa "Financeiro" > "Relatórios"
2. Filtra por partida, mês ou ano
3. Sistema exibe receitas, despesas e saldo do período

---

## UC-019 — Marcar Jogador como Mensalista

**Ator principal:** OWNER, ADMIN  
**Pré-condições:** Jogador pertence ao grupo  
**Fluxo principal:**
1. Admin acessa perfil do jogador
2. Habilita "Mensalista"
3. Define valor da mensalidade
4. Sistema gera cobrança mensal recorrente
5. Quando pago, jogador pode confirmar presença em qualquer partida do mês sem novo pagamento
