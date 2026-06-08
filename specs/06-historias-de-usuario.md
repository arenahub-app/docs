# Histórias de Usuário — ArenaHub

## Convenção

Formato: **Como [persona], quero [ação], para [benefício]**  
Rastreabilidade: cada história referencia RFs, RNs e CAs.

Critérios de estimativa (Story Points — Fibonacci):
- 1: trivial (< 2h)
- 2: simples (2–4h)
- 3: moderado (4–8h)
- 5: complexo (1–2 dias)
- 8: muito complexo (3–5 dias)
- 13: épico (> 5 dias — deve ser quebrado)

---

## Épico 1 — Autenticação e Perfil

### US-001 — Cadastro de usuário
**Como** visitante  
**Quero** criar uma conta com email e senha  
**Para** acessar o ArenaHub e gerenciar meu grupo

**Story Points:** 3  
**Refs:** RF-USR-001, RF-USR-002, RN-USR-001, CA-USR-001, CA-USR-002

**Critérios de Aceite:**
- Formulário com: nome, email, senha (mín. 8 chars), telefone, data de nascimento
- Email de confirmação enviado após cadastro
- Erro claro se email já cadastrado

---

### US-002 — Login com email e senha
**Como** usuário cadastrado  
**Quero** fazer login com meu email e senha  
**Para** acessar minha conta

**Story Points:** 2  
**Refs:** RF-USR-003, RF-USR-005, CA-USR-004

**Critérios de Aceite:**
- JWT (15min) emitido no login bem-sucedido
- Refresh token (7d) emitido e armazenado seguramente
- Mensagem de erro genérica em caso de credenciais inválidas (sem revelar qual campo errou)

---

### US-003 — Login com Google
**Como** usuário  
**Quero** entrar com minha conta Google  
**Para** não precisar criar e lembrar mais uma senha

**Story Points:** 5  
**Refs:** RF-USR-004, CA-USR-003

**Critérios de Aceite:**
- Fluxo OAuth2 Authorization Code funciona
- Conta criada automaticamente no primeiro login
- Se conta Google já existe, faz login sem duplicar

---

### US-004 — Recuperação de senha
**Como** usuário que esqueceu a senha  
**Quero** receber um link de redefinição por email  
**Para** recuperar o acesso à minha conta

**Story Points:** 3  
**Refs:** RF-USR-006

**Critérios de Aceite:**
- Link de redefinição expira em 1 hora
- Após usar o link, a senha é atualizada e o link se torna inválido
- Email de confirmação enviado após redefinição

---

### US-005 — Editar perfil
**Como** usuário autenticado  
**Quero** atualizar meu nome, telefone e foto de perfil  
**Para** manter minhas informações atualizadas

**Story Points:** 2  
**Refs:** RF-USR-007, RF-USR-008

**Critérios de Aceite:**
- Upload de foto aceita JPEG e PNG (máx. 5MB)
- Foto armazenada no R2 e URL atualizada no perfil
- Alterações salvas e refletidas imediatamente

---

## Épico 2 — Grupos Esportivos

### US-006 — Criar grupo
**Como** usuário autenticado  
**Quero** criar um grupo esportivo  
**Para** centralizar a gestão da minha turma

**Story Points:** 3  
**Refs:** RF-GRP-001, RF-GRP-002, RF-GRP-003, CA-GRP-001

**Critérios de Aceite:**
- Modalidades disponíveis: Futebol, Vôlei, Basquete, Futevôlei, Beach Tennis, Outros
- Criador vira OWNER automaticamente
- Link de convite gerado após criação

---

### US-007 — Convidar jogadores por link
**Como** ADMIN/OWNER  
**Quero** compartilhar um link de convite  
**Para** adicionar jogadores ao grupo sem precisar cadastrá-los manualmente

**Story Points:** 3  
**Refs:** RF-GRP-004, RN-GRP-004, CA-GRP-002, CA-GRP-003

**Critérios de Aceite:**
- Link expira após 7 dias ou 50 usos
- Novos membros entram com papel PLAYER
- Link expirado exibe mensagem clara

---

### US-008 — Gerenciar papéis de membros
**Como** OWNER  
**Quero** promover ou rebaixar membros  
**Para** delegar a administração do grupo

**Story Points:** 2  
**Refs:** RF-MBR-002, RF-MBR-003, RN-MBR-001, RN-MBR-002

**Critérios de Aceite:**
- OWNER pode promover PLAYER → ADMIN
- OWNER pode rebaixar ADMIN → PLAYER
- OWNER não pode ser rebaixado

---

### US-009 — Remover membro do grupo
**Como** OWNER/ADMIN  
**Quero** remover um membro do grupo  
**Para** manter a lista de membros atualizada

**Story Points:** 2  
**Refs:** RF-MBR-007, RN-MBR-003

**Critérios de Aceite:**
- Membro removido perde acesso imediato
- Histórico de pagamentos e presenças mantido
- Membro removido é notificado

---

## Épico 3 — Presença e Partidas

### US-010 — Criar partida
**Como** ADMIN/OWNER  
**Quero** criar uma partida com data, hora e local  
**Para** organizar o próximo jogo do grupo

**Story Points:** 3  
**Refs:** RF-PRE-001, RF-PRE-002, RF-PRE-003

**Critérios de Aceite:**
- Campos obrigatórios: data, hora, local, número máximo de jogadores
- Lista de presença abre na criação
- Todos os PLAYERs do grupo recebem notificação

---

### US-011 — Confirmar presença
**Como** PLAYER  
**Quero** confirmar minha presença em uma partida  
**Para** garantir minha vaga na lista

**Story Points:** 2  
**Refs:** RF-PRE-004, RN-PRE-007, CA-PRE-001, CA-PRE-002

**Critérios de Aceite:**
- Botão "Confirmar Presença" visível enquanto lista aberta e com vaga
- Se lista cheia, entra em fila de espera
- Status atualizado imediatamente

---

### US-012 — Cancelar presença
**Como** PLAYER  
**Quero** cancelar minha presença em uma partida  
**Para** liberar minha vaga para outro jogador

**Story Points:** 1  
**Refs:** RF-PRE-005, RN-PRE-003

**Critérios de Aceite:**
- Cancelamento disponível enquanto lista aberta
- Próximo da fila recebe notificação de vaga disponível (30 min para confirmar)

---

### US-013 — Fechamento automático de lista
**Como** ADMIN  
**Quero** que a lista feche automaticamente 1 hora antes da partida  
**Para** ter tempo de organizar os times

**Story Points:** 3  
**Refs:** RF-PRE-006, RN-PRE-001, CA-PRE-003

**Critérios de Aceite:**
- Scheduler fecha lista exatamente 60 min antes da partida
- Jogadores na fila de espera são notificados
- Status da lista atualiza para FECHADA

---

### US-014 — Banir jogador de presença
**Como** ADMIN/OWNER  
**Quero** banir um jogador da lista de presença  
**Para** aplicar uma sanção por comportamento inadequado

**Story Points:** 3  
**Refs:** RF-PRE-009, RF-PRE-010, RF-PRE-011, RN-PRE-004, RN-PRE-006

**Critérios de Aceite:**
- ADMIN informa motivo do banimento (obrigatório)
- Jogador banido é notificado com o motivo
- Ao tentar confirmar presença, jogador é direcionado para fluxo de justificativa

---

## Épico 4 — Votação de Habilidades

### US-015 — Abrir votação de habilidades
**Como** ADMIN/OWNER  
**Quero** abrir uma votação de habilidades no grupo  
**Para** atualizar o nível de skill dos jogadores de forma coletiva

**Story Points:** 3  
**Refs:** RF-VOT-001, RF-VOT-002, RN-VOT-001, CA-VOT-001, CA-VOT-002

**Critérios de Aceite:**
- Prazo obrigatório
- Erro se já existe votação ativa
- Todos os PLAYERs habilitados notificados

---

### US-016 — Votar em habilidade de colegas
**Como** PLAYER habilitado  
**Quero** votar no nível de habilidade dos meus colegas  
**Para** contribuir para uma formação de times mais justa

**Story Points:** 3  
**Refs:** RF-VOT-006, RF-VOT-007, RF-VOT-010, RN-VOT-002, RN-VOT-003, CA-VOT-003

**Critérios de Aceite:**
- Escala de 1 a 6 estrelas por colega
- Jogador não vota em si mesmo
- Votos podem ser alterados enquanto votação aberta
- Votos de outros jogadores são ocultados ao PLAYER

---

### US-017 — Encerrar votação e atualizar skills
**Como** ADMIN/OWNER  
**Quero** encerrar a votação e ver os skills atualizados  
**Para** usar os novos dados na formação dos próximos times

**Story Points:** 3  
**Refs:** RF-VOT-003, RF-VOT-008, CA-VOT-004

**Critérios de Aceite:**
- Encerramento manual disponível a qualquer momento
- Sistema calcula média por jogador e atualiza skill
- Resultados visíveis para todos os membros

---

## Épico 5 — Pagamentos

### US-018 — Registrar pagamento de diária
**Como** PLAYER  
**Quero** enviar o comprovante do meu pagamento  
**Para** registrar que paguei a diária da partida

**Story Points:** 5  
**Refs:** RF-PAG-001, RF-PAG-003, RF-PAG-004, RN-PAG-007, RN-PAG-008, CA-PAG-001, CA-PAG-002

**Critérios de Aceite:**
- Upload de JPEG, PNG ou PDF (máx. 10MB)
- Tipo inválido gera erro claro
- Comprovante enviado para IA automaticamente

---

### US-019 — Acompanhar resultado da validação
**Como** PLAYER  
**Quero** receber notificação do resultado da validação do meu comprovante  
**Para** saber se meu pagamento foi aprovado

**Story Points:** 2  
**Refs:** RF-PAG-010, CA-PAG-003, CA-PAG-004

**Critérios de Aceite:**
- Notificação de APROVADO, REPROVADO ou EM_REVISAO
- Em caso de reprovação, motivo exibido e opção de reenvio disponível

---

### US-020 — Revisar pagamento manualmente
**Como** ADMIN/OWNER  
**Quero** revisar comprovantes marcados como "Revisão Manual"  
**Para** garantir que nenhum pagamento legítimo seja rejeitado automaticamente

**Story Points:** 3  
**Refs:** RF-PAG-008, RF-PAG-009, CA-PAG-004

**Critérios de Aceite:**
- Fila de revisão exibe comprovante, valor e dados da cobrança
- ADMIN pode aprovar ou reprovar com anotação opcional
- PLAYER notificado do resultado

---

### US-021 — Mensalidade e presença sem pagamento adicional
**Como** PLAYER mensalista  
**Quero** confirmar presença sem pagar diária quando minha mensalidade está em dia  
**Para** ter o benefício de pagar uma vez ao mês

**Story Points:** 2  
**Refs:** RF-PAG-012, RF-PAG-013, RN-PRE-008, CA-PAG-005

**Critérios de Aceite:**
- Ao confirmar presença, sistema verifica se mensalidade do mês foi paga
- Se paga, confirmação liberada sem exigir novo pagamento

---

## Épico 6 — Árbitros

### US-022 — Cadastrar árbitro no grupo
**Como** ADMIN/OWNER  
**Quero** cadastrar um árbitro com valor e tipo de cobrança  
**Para** ter os dados formalizados no sistema

**Story Points:** 2  
**Refs:** RF-ARB-001, RF-ARB-002, RF-ARB-003, RN-ARB-001

**Critérios de Aceite:**
- Árbitro vinculado a usuário existente
- Tipos disponíveis: por partida ou mensal
- Valor configurável

---

### US-023 — Associar árbitro a partida
**Como** ADMIN/OWNER  
**Quero** associar um árbitro a uma partida  
**Para** registrar quem vai apitar e gerar a despesa automaticamente

**Story Points:** 2  
**Refs:** RF-ARB-004, RF-ARB-007, RN-ARB-004, CA-ARB-001

**Critérios de Aceite:**
- Despesa gerada automaticamente com o valor configurado para o árbitro
- Árbitro notificado da associação
- Despesa aparece no financeiro da partida

---

## Épico 7 — Formação de Times

### US-024 — Formar times automaticamente
**Como** ADMIN/OWNER  
**Quero** que o sistema forme times equilibrados por skill  
**Para** ter uma distribuição justa sem precisar fazer manualmente

**Story Points:** 8  
**Refs:** RF-TIM-001, RF-TIM-002, RF-TIM-003, RF-TIM-005, RN-TIM-001 a RN-TIM-009, CA-TIM-001, CA-TIM-002, CA-TIM-003

**Critérios de Aceite:**
- Disponível somente após fechamento da lista
- Árbitros excluídos dos times
- Algoritmo snake draft com skill como critério primário
- Desvio padrão ≤ 0.5 entre times
- Times visíveis para todos os membros após confirmação

---

### US-025 — Ajustar formação manualmente
**Como** ADMIN/OWNER  
**Quero** mover jogadores entre times após a formação automática  
**Para** fazer ajustes que o algoritmo não considerou

**Story Points:** 3  
**Refs:** RF-TIM-004, RF-TIM-008, RN-TIM-007, RN-TIM-008

**Critérios de Aceite:**
- Interface de drag-and-drop para mover jogadores
- Histórico da formação salvo com timestamp e responsável
- Confirmação necessária antes de publicar para os jogadores

---

## Épico 8 — Financeiro e Relatórios

### US-026 — Visualizar saldo do grupo
**Como** ADMIN/OWNER  
**Quero** ver o saldo atual do grupo  
**Para** saber a situação financeira do grupo a qualquer momento

**Story Points:** 2  
**Refs:** RF-FIN-005, RN-FIN-004, CA-FIN-001

**Critérios de Aceite:**
- Saldo = Σ(receitas) − Σ(despesas)
- Atualizado em tempo real após aprovação de pagamentos e registro de despesas

---

### US-027 — Registrar despesa avulsa
**Como** ADMIN/OWNER  
**Quero** registrar despesas como aluguel de quadra  
**Para** ter o controle financeiro completo do grupo

**Story Points:** 2  
**Refs:** RF-FIN-002, RF-FIN-004, RN-FIN-001

**Critérios de Aceite:**
- Campos: valor, categoria (árbitro/quadra/outra), descrição, data
- Optionally vinculada a uma partida
- Saldo do grupo atualizado após registro

---

### US-028 — Relatório financeiro mensal
**Como** ADMIN/OWNER  
**Quero** ver um relatório financeiro do mês  
**Para** entender quanto o grupo arrecadou e gastou

**Story Points:** 3  
**Refs:** RF-FIN-007, RF-REL-003, RF-REL-004, CA-FIN-002

**Critérios de Aceite:**
- Filtro por mês/ano
- Exibe: receitas, despesas, saldo, inadimplentes
- Dados corretos e atualizados
