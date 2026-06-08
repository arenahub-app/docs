# Requisitos Funcionais — ArenaHub

## Convenção de Rastreabilidade

| Prefixo | Domínio |
|---|---|
| RF-USR | Usuários e Autenticação |
| RF-GRP | Grupos Esportivos |
| RF-MBR | Membros e Papéis |
| RF-PRE | Presença e Partidas |
| RF-SKL | Habilidades (Skill) |
| RF-VOT | Votação |
| RF-PAG | Pagamentos |
| RF-FIN | Financeiro |
| RF-ARB | Árbitros |
| RF-TIM | Formação de Times |
| RF-REL | Relatórios |

---

## RF-USR — Usuários e Autenticação

| ID | Descrição | Prioridade |
|---|---|---|
| RF-USR-001 | O sistema deve permitir cadastro de usuário com nome, email, senha, telefone e data de nascimento | Alta |
| RF-USR-002 | O sistema deve enviar email de confirmação após cadastro | Alta |
| RF-USR-003 | O sistema deve permitir login com email e senha | Alta |
| RF-USR-004 | O sistema deve permitir autenticação via OAuth2 Google | Alta |
| RF-USR-005 | O sistema deve emitir JWT com refresh token após autenticação bem-sucedida | Alta |
| RF-USR-006 | O sistema deve permitir recuperação de senha via email | Alta |
| RF-USR-007 | O sistema deve permitir que o usuário edite seu perfil (nome, telefone, foto) | Média |
| RF-USR-008 | O sistema deve permitir upload de foto de perfil | Média |
| RF-USR-009 | O sistema deve armazenar data de nascimento do usuário | Média |
| RF-USR-010 | O sistema deve permitir exclusão de conta (LGPD) | Alta |

---

## RF-GRP — Grupos Esportivos

| ID | Descrição | Prioridade |
|---|---|---|
| RF-GRP-001 | O sistema deve permitir criação de grupo com nome, modalidade, descrição e foto | Alta |
| RF-GRP-002 | As modalidades suportadas são: Futebol, Vôlei, Basquete, Futevôlei, Beach Tennis, Outros | Alta |
| RF-GRP-003 | O criador do grupo deve ser automaticamente atribuído o papel OWNER | Alta |
| RF-GRP-004 | O sistema deve gerar link de convite único por grupo | Alta |
| RF-GRP-005 | O sistema deve permitir que o OWNER edite as configurações do grupo | Média |
| RF-GRP-006 | O sistema deve permitir desativação de grupo (soft delete) | Média |
| RF-GRP-007 | O sistema deve suportar múltiplos grupos por usuário (um usuário pode ser membro de vários grupos) | Alta |

---

## RF-MBR — Membros e Papéis

| ID | Descrição | Prioridade |
|---|---|---|
| RF-MBR-001 | Os papéis existentes são: OWNER, ADMIN, PLAYER, REFEREE | Alta |
| RF-MBR-002 | O OWNER pode promover membros a ADMIN | Alta |
| RF-MBR-003 | O OWNER pode rebaixar ADMINs a PLAYER | Alta |
| RF-MBR-004 | O OWNER não pode ser rebaixado por outros membros | Alta |
| RF-MBR-005 | ADMINs podem convidar novos membros ao grupo | Alta |
| RF-MBR-006 | Membros convidados por link recebem papel PLAYER por padrão | Alta |
| RF-MBR-007 | O OWNER e ADMINs podem remover membros do grupo | Alta |
| RF-MBR-008 | O sistema deve manter histórico de alterações de papéis | Baixa |

---

## RF-PRE — Presença e Partidas

| ID | Descrição | Prioridade |
|---|---|---|
| RF-PRE-001 | ADMINs/OWNER podem criar partidas com data, hora, local e número máximo de jogadores | Alta |
| RF-PRE-002 | O sistema deve abrir a lista de presença na criação da partida | Alta |
| RF-PRE-003 | O sistema deve notificar todos os PLAYERs ao criar uma partida | Alta |
| RF-PRE-004 | PLAYERs podem confirmar presença enquanto a lista estiver aberta | Alta |
| RF-PRE-005 | PLAYERs podem recusar presença a qualquer momento enquanto a lista estiver aberta | Alta |
| RF-PRE-006 | O sistema deve fechar automaticamente a lista 1 hora antes da partida | Alta |
| RF-PRE-007 | O sistema deve gerenciar fila de espera quando a lista atingir o limite | Alta |
| RF-PRE-008 | O sistema deve notificar o próximo da fila quando uma vaga se abrir | Alta |
| RF-PRE-009 | ADMINs podem banir jogadores de confirmar presença, informando motivo | Alta |
| RF-PRE-010 | Jogadores banidos devem informar justificativa ao tentar confirmar presença | Alta |
| RF-PRE-011 | ADMINs devem aprovar ou rejeitar a justificativa do jogador banido | Alta |
| RF-PRE-012 | O sistema deve registrar histórico de banimentos com data e motivo | Média |

---

## RF-SKL — Habilidades (Skill)

| ID | Descrição | Prioridade |
|---|---|---|
| RF-SKL-001 | Cada jogador possui um skill no grupo de 1 a 6 estrelas | Alta |
| RF-SKL-002 | O skill padrão de um jogador ao entrar no grupo é 3 estrelas | Alta |
| RF-SKL-003 | O skill é específico por grupo (mesmo usuário pode ter skills diferentes em grupos diferentes) | Alta |
| RF-SKL-004 | O skill é calculado como média dos votos da última votação encerrada | Alta |
| RF-SKL-005 | O skill pode ser ajustado manualmente por OWNER/ADMIN | Média |
| RF-SKL-006 | Jogadores podem informar sua posição preferencial por modalidade | Média |

---

## RF-VOT — Votação

| ID | Descrição | Prioridade |
|---|---|---|
| RF-VOT-001 | ADMINs/OWNER podem abrir votação de habilidades para o grupo | Alta |
| RF-VOT-002 | ADMINs/OWNER devem definir prazo de encerramento da votação | Alta |
| RF-VOT-003 | ADMINs/OWNER podem encerrar votação manualmente antes do prazo | Alta |
| RF-VOT-004 | ADMINs/OWNER podem banir jogadores específicos da votação, informando motivo | Alta |
| RF-VOT-005 | Jogadores banidos da votação não podem votar | Alta |
| RF-VOT-006 | PLAYERs habilitados podem votar em todos os colegas (exceto si mesmos) | Alta |
| RF-VOT-007 | Votos são anônimos entre PLAYERs; somente ADMINs podem ver votos individuais | Média |
| RF-VOT-008 | Ao encerrar a votação, o sistema calcula a média de votos por jogador e atualiza o skill | Alta |
| RF-VOT-009 | Apenas uma votação pode estar ativa por grupo ao mesmo tempo | Alta |
| RF-VOT-010 | Jogadores podem alterar seus votos enquanto a votação estiver aberta | Média |

---

## RF-PAG — Pagamentos

| ID | Descrição | Prioridade |
|---|---|---|
| RF-PAG-001 | O sistema deve suportar cobranças do tipo diária (por partida) | Alta |
| RF-PAG-002 | O sistema deve suportar cobranças do tipo mensalidade | Alta |
| RF-PAG-003 | PLAYERs podem fazer upload de comprovante de pagamento (imagem ou PDF) | Alta |
| RF-PAG-004 | O sistema deve enviar o comprovante para validação por IA | Alta |
| RF-PAG-005 | A IA deve validar: valor, chave Pix do grupo e data do pagamento | Alta |
| RF-PAG-006 | O resultado da validação por IA pode ser: Aprovado, Reprovado ou Revisão Manual | Alta |
| RF-PAG-007 | Pagamentos aprovados automaticamente pela IA atualizam o status sem intervenção humana | Alta |
| RF-PAG-008 | Pagamentos em Revisão Manual devem ser analisados por ADMIN/OWNER | Alta |
| RF-PAG-009 | ADMINs podem aprovar ou reprovar pagamentos manualmente | Alta |
| RF-PAG-010 | PLAYERs são notificados do resultado da validação | Alta |
| RF-PAG-011 | O sistema deve suportar reenvio de comprovante em caso de reprovação | Média |
| RF-PAG-012 | Mensalistas com mensalidade paga podem confirmar presença sem novo pagamento no mês | Alta |
| RF-PAG-013 | ADMINs podem marcar jogador como mensalista e definir valor da mensalidade | Alta |
| RF-PAG-014 | O sistema deve registrar histórico completo de pagamentos por jogador | Alta |

---

## RF-ARB — Árbitros

| ID | Descrição | Prioridade |
|---|---|---|
| RF-ARB-001 | ADMINs podem cadastrar árbitros vinculados a usuários existentes no sistema | Alta |
| RF-ARB-002 | Árbitros podem ter tipo de cobrança: por partida ou mensal | Alta |
| RF-ARB-003 | O valor do árbitro deve ser configurável por grupo | Alta |
| RF-ARB-004 | ADMINs podem associar um árbitro a uma partida | Alta |
| RF-ARB-005 | Árbitros associados a partidas devem ser notificados | Média |
| RF-ARB-006 | Árbitros não participam da formação de times | Alta |
| RF-ARB-007 | A associação de árbitro a partida gera automaticamente uma despesa no financeiro | Alta |
| RF-ARB-008 | O sistema deve registrar histórico de partidas por árbitro | Média |

---

## RF-TIM — Formação de Times

| ID | Descrição | Prioridade |
|---|---|---|
| RF-TIM-001 | O sistema deve suportar formação automática de times por skill e posição | Alta |
| RF-TIM-002 | O número de times deve ser definido pelo ADMIN no momento da formação | Alta |
| RF-TIM-003 | Árbitros não devem ser incluídos na formação de times | Alta |
| RF-TIM-004 | ADMINs podem ajustar manualmente a formação proposta pelo sistema | Alta |
| RF-TIM-005 | O sistema deve salvar o histórico de formações por partida | Alta |
| RF-TIM-006 | O sistema deve suportar posições específicas por modalidade (ver glossário) | Alta |
| RF-TIM-007 | O algoritmo deve garantir equilíbrio de skill médio entre os times | Alta |
| RF-TIM-008 | O algoritmo deve considerar cobertura mínima de posições críticas por modalidade | Média |
| RF-TIM-009 | PLAYERs devem visualizar seus times após a formação | Alta |

---

## RF-FIN — Financeiro

| ID | Descrição | Prioridade |
|---|---|---|
| RF-FIN-001 | O sistema deve registrar receitas provenientes de diárias e mensalidades | Alta |
| RF-FIN-002 | O sistema deve registrar despesas (árbitros, quadra, outras) | Alta |
| RF-FIN-003 | Despesas de árbitros devem ser geradas automaticamente ao associar árbitro a partida | Alta |
| RF-FIN-004 | ADMINs podem registrar despesas avulsas com descrição e categoria | Alta |
| RF-FIN-005 | O sistema deve calcular e exibir saldo do grupo | Alta |
| RF-FIN-006 | O sistema deve gerar relatório financeiro por partida | Alta |
| RF-FIN-007 | O sistema deve gerar relatório financeiro por mês | Alta |
| RF-FIN-008 | O sistema deve gerar relatório financeiro por ano | Alta |
| RF-FIN-009 | O sistema deve identificar e listar jogadores inadimplentes | Alta |

---

## RF-REL — Relatórios

| ID | Descrição | Prioridade |
|---|---|---|
| RF-REL-001 | O sistema deve gerar relatório de presença por partida | Média |
| RF-REL-002 | O sistema deve gerar relatório de presença por jogador (histórico) | Média |
| RF-REL-003 | O sistema deve gerar relatório de pagamentos por período | Alta |
| RF-REL-004 | O sistema deve gerar relatório de inadimplência | Alta |
| RF-REL-005 | O sistema deve gerar relatório financeiro consolidado | Alta |
| RF-REL-006 | Relatórios devem ser exportáveis (PDF e/ou CSV) | Baixa |
