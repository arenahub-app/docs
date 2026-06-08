# Critérios de Aceitação — ArenaHub

## Convenção

Cada critério segue o formato **Dado / Quando / Então** (Gherkin-style) e está rastreado ao requisito funcional correspondente.

---

## CA-USR — Usuários e Autenticação

### CA-USR-001 — Cadastro com email (RF-USR-001)
```
Dado que um visitante acessa a página de cadastro
Quando preenche nome, email válido, senha (mín. 8 chars), telefone e data de nascimento
  E clica em "Cadastrar"
Então o sistema cria a conta
  E envia email de confirmação
  E redireciona para página de confirmação pendente
```

### CA-USR-002 — Email duplicado (RF-USR-001)
```
Dado que um email já está cadastrado no sistema
Quando um visitante tenta cadastrar com o mesmo email
Então o sistema retorna erro 409
  E exibe mensagem "Email já cadastrado"
```

### CA-USR-003 — Login OAuth2 Google (RF-USR-004)
```
Dado que um usuário clica em "Entrar com Google"
Quando o Google autentica com sucesso e retorna os dados
Então o sistema cria ou recupera a conta vinculada ao email Google
  E emite JWT de acesso (15min) e refresh token (7d)
  E redireciona para o dashboard
```

### CA-USR-004 — Token expirado (RF-USR-005)
```
Dado que o JWT de acesso do usuário expirou
Quando o usuário faz uma requisição autenticada
Então o sistema retorna 401 Unauthorized
  E o cliente usa o refresh token para obter novo JWT
  E a requisição original é refeita com o novo token
```

---

## CA-GRP — Grupos

### CA-GRP-001 — Criação de grupo (RF-GRP-001)
```
Dado que um usuário autenticado acessa "Criar Grupo"
Quando preenche nome, modalidade e clica em "Criar"
Então o sistema cria o grupo
  E atribui papel OWNER ao criador
  E gera link de convite único
  E redireciona para o dashboard do grupo
```

### CA-GRP-002 — Convite por link (RF-GRP-004)
```
Dado que um usuário autenticado acessa um link de convite válido
Quando confirma a entrada no grupo
Então o sistema adiciona o usuário com papel PLAYER
  E redireciona para o dashboard do grupo
```

### CA-GRP-003 — Link expirado (RF-GRP-004 + RN-GRP-004)
```
Dado que um link de convite expirou (> 7 dias)
Quando um usuário tenta acessá-lo
Então o sistema exibe mensagem "Link de convite expirado"
  E não adiciona o usuário ao grupo
```

---

## CA-PRE — Presença

### CA-PRE-001 — Confirmação de presença (RF-PRE-004)
```
Dado que uma partida foi criada com lista aberta e vagas disponíveis
  E o usuário é PLAYER do grupo e não está banido
Quando o PLAYER clica em "Confirmar Presença"
Então o sistema registra a presença como CONFIRMADA
  E decrementa o contador de vagas disponíveis
```

### CA-PRE-002 — Lista cheia (RF-PRE-007)
```
Dado que a lista de uma partida atingiu o número máximo de vagas
Quando um PLAYER tenta confirmar presença
Então o sistema adiciona o jogador à fila de espera
  E informa a posição na fila ao jogador
```

### CA-PRE-003 — Fechamento automático (RF-PRE-006 + RN-PRE-001)
```
Dado que uma partida está marcada para 20:00
  E são 19:00 (exatamente 1 hora antes)
Quando o scheduler do sistema executa
Então a lista de presença é fechada automaticamente
  E jogadores na fila de espera são notificados que não entraram
```

### CA-PRE-004 — Jogador banido tenta confirmar (RF-PRE-010)
```
Dado que um PLAYER está banido de presença no grupo
Quando tenta confirmar presença em uma partida
Então o sistema exibe formulário de justificativa
  E não confirma a presença até aprovação do ADMIN
```

### CA-PRE-005 — ADMIN aprova justificativa (RF-PRE-011)
```
Dado que um jogador banido enviou justificativa para uma partida
Quando o ADMIN aprova a justificativa
Então o sistema adiciona o jogador à lista da partida (se ainda houver vaga)
  E notifica o jogador da aprovação
```

---

## CA-VOT — Votação

### CA-VOT-001 — Abertura de votação (RF-VOT-001)
```
Dado que não há votação ativa no grupo
Quando o ADMIN abre uma votação com prazo definido
Então o sistema cria a votação no status ATIVA
  E notifica todos os PLAYERs habilitados
```

### CA-VOT-002 — Votação duplicada (RN-VOT-001)
```
Dado que já existe uma votação ativa no grupo
Quando o ADMIN tenta abrir outra votação
Então o sistema retorna erro
  E exibe mensagem "Já existe uma votação ativa"
```

### CA-VOT-003 — Votar em si mesmo (RN-VOT-002)
```
Dado que uma votação está ativa
Quando o PLAYER tenta votar em si mesmo
Então o sistema não permite e exibe erro de validação
```

### CA-VOT-004 — Encerramento e cálculo (RF-VOT-008)
```
Dado que uma votação está ativa com votos registrados
Quando o ADMIN encerra a votação
Então o sistema calcula a média de votos por jogador (excluindo banidos)
  E atualiza o skill de cada jogador com a média calculada
  E registra o encerramento no histórico
```

---

## CA-PAG — Pagamentos

### CA-PAG-001 — Upload de comprovante (RF-PAG-003)
```
Dado que um PLAYER tem uma cobrança pendente
Quando faz upload de comprovante JPEG, PNG ou PDF (≤ 10MB)
Então o sistema armazena o arquivo no R2
  E envia para validação por IA
  E atualiza status para AGUARDANDO_VALIDACAO
```

### CA-PAG-002 — Arquivo inválido (RN-PAG-007)
```
Dado que um PLAYER tenta enviar um comprovante
Quando o arquivo é de tipo não permitido (ex: .exe, .docx)
Então o sistema rejeita o upload
  E exibe mensagem "Tipo de arquivo não suportado. Use JPEG, PNG ou PDF."
```

### CA-PAG-003 — IA aprova pagamento (RF-PAG-006)
```
Dado que um comprovante foi enviado para validação
Quando a IA retorna resultado APROVADO
Então o sistema marca o pagamento como APROVADO
  E notifica o PLAYER
  E registra receita no financeiro do grupo
```

### CA-PAG-004 — Timeout da IA (RN-PAG-006)
```
Dado que um comprovante foi enviado para validação
Quando a IA não responde em 10 segundos
Então o sistema marca o pagamento como EM_REVISAO
  E notifica o ADMIN para análise manual
```

### CA-PAG-005 — Mensalista confirma presença (RF-PAG-012)
```
Dado que um PLAYER é mensalista com mensalidade paga no mês corrente
Quando tenta confirmar presença em uma partida do mesmo mês
Então o sistema permite a confirmação sem exigir pagamento de diária
```

---

## CA-ARB — Árbitros

### CA-ARB-001 — Associação a partida gera despesa (RN-ARB-004)
```
Dado que um árbitro "por partida" está cadastrado no grupo com valor R$ 80
Quando o ADMIN associa o árbitro à partida X
Então o sistema registra uma despesa de R$ 80 na partida X
  E a despesa aparece no financeiro da partida
```

---

## CA-TIM — Formação de Times

### CA-TIM-001 — Formação automática (RF-TIM-001)
```
Dado que a lista de presença está fechada com 10 jogadores confirmados
Quando o ADMIN solicita formação de 2 times
Então o sistema distribui os jogadores por snake draft (skill DESC)
  E o desvio padrão do skill médio entre os times é ≤ 0.5
  E os times são exibidos ao ADMIN para confirmação
```

### CA-TIM-002 — Árbitro excluído dos times (RN-TIM-002)
```
Dado que há um REFEREE confirmado no grupo
Quando o ADMIN forma os times da partida
Então o REFEREE não aparece em nenhum time
```

### CA-TIM-003 — Lista não fechada (RN-TIM-001)
```
Dado que a lista de presença ainda está aberta
Quando o ADMIN tenta formar times
Então o sistema retorna erro
  E exibe mensagem "A lista de presença ainda está aberta"
```

---

## CA-FIN — Financeiro

### CA-FIN-001 — Saldo do grupo (RN-FIN-004)
```
Dado que o grupo tem R$ 500 em receitas e R$ 200 em despesas
Quando o ADMIN acessa o painel financeiro
Então o sistema exibe saldo de R$ 300
```

### CA-FIN-002 — Relatório mensal (RF-FIN-007)
```
Dado que existem movimentações no mês de Junho/2026
Quando o ADMIN acessa "Relatórios" e filtra por Junho/2026
Então o sistema exibe: total de receitas, total de despesas, saldo e lista de inadimplentes do período
```

---

## CA-SEG — Segurança

### CA-SEG-001 — Acesso entre tenants (RNF-SEG-005)
```
Dado que um usuário é PLAYER do grupo A
Quando tenta acessar uma API com o ID do grupo B via manipulação de parâmetro
Então o sistema retorna 403 Forbidden
  E não expõe dados do grupo B
```

### CA-SEG-002 — Operação sem papel suficiente (RNF-SEG-006)
```
Dado que um PLAYER tenta criar uma partida (ação restrita a ADMIN/OWNER)
Quando faz a requisição à API
Então o sistema retorna 403 Forbidden
  E nenhuma partida é criada
```
