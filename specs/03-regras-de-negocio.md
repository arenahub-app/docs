# Regras de Negócio — ArenaHub

## Convenção

Cada regra é rastreada por ID e referenciada nas histórias de usuário e casos de uso.

| Prefixo | Domínio |
|---|---|
| RN-USR | Usuários |
| RN-GRP | Grupos |
| RN-MBR | Membros e Papéis |
| RN-PRE | Presença |
| RN-SKL | Skill |
| RN-VOT | Votação |
| RN-PAG | Pagamentos |
| RN-ARB | Árbitros |
| RN-TIM | Formação de Times |
| RN-FIN | Financeiro |

---

## RN-USR — Usuários

**RN-USR-001**  
Um usuário só pode ter um cadastro por email no sistema.  
*Violação:* erro 409 Conflict com mensagem "Email já cadastrado".

**RN-USR-002**  
Usuários autenticados via Google que não possuem telefone cadastrado devem completar o perfil antes de acessar funcionalidades de grupo.

**RN-USR-003**  
A exclusão de conta anonimiza os dados pessoais do usuário, mas mantém o histórico de transações financeiras por obrigação legal.

---

## RN-GRP — Grupos

**RN-GRP-001**  
Cada grupo deve ter exatamente um OWNER a qualquer momento.

**RN-GRP-002**  
A modalidade do grupo não pode ser alterada após a criação do primeiro time formado. Até esse momento, pode ser alterada pelo OWNER.

**RN-GRP-003**  
Um grupo desativado (soft delete) impede criação de novas partidas, mas mantém histórico acessível ao OWNER.

**RN-GRP-004**  
Links de convite expiram após 7 dias ou após uso máximo de 50 acessos, o que ocorrer primeiro.

---

## RN-MBR — Membros e Papéis

**RN-MBR-001**  
O OWNER não pode ser rebaixado enquanto não transferir o papel a outro membro.

**RN-MBR-002**  
Somente o OWNER pode promover membros ao papel ADMIN ou transferir o papel OWNER.

**RN-MBR-003**  
Um membro removido do grupo perde acesso imediato a todas as funcionalidades do grupo, mas seus dados históricos (pagamentos, presenças) são mantidos.

**RN-MBR-004**  
REFEREEs são membros do grupo mas com papel restrito: não participam de formação de times, votações ou listas de presença como jogadores.

**RN-MBR-005**  
Um usuário pode ser membro de quantos grupos quiser, com papéis independentes em cada grupo.

---

## RN-PRE — Presença

**RN-PRE-001**  
A lista de presença fecha automaticamente exatamente 60 minutos antes do horário da partida, independentemente de intervenção humana.

**RN-PRE-002**  
Após o fechamento da lista, nenhum jogador pode ser adicionado ou removido sem intervenção manual de ADMIN/OWNER.

**RN-PRE-003**  
A fila de espera respeita ordem de inscrição (FIFO). O próximo da fila é notificado somente quando uma vaga se abre e tem 30 minutos para confirmar antes de perder a vaga para o próximo.

**RN-PRE-004**  
Jogadores banidos de presença não podem confirmar presença sem passar pelo fluxo de justificativa e aprovação.

**RN-PRE-005**  
O banimento de presença é específico por grupo e não afeta outros grupos onde o jogador é membro.

**RN-PRE-006**  
O banimento de presença não tem prazo automático; somente ADMIN/OWNER pode remover o banimento.

**RN-PRE-007**  
Um jogador só pode ter um registro de presença por partida (confirmado, recusado ou em espera).

**RN-PRE-008**  
Mensalistas com mensalidade paga no mês corrente podem confirmar presença sem exigência de pagamento de diária.

---

## RN-SKL — Skill

**RN-SKL-001**  
O skill de um jogador varia de 1 a 6 com uma casa decimal (ex.: 3.5).

**RN-SKL-002**  
O skill padrão ao entrar em um grupo é 3.0.

**RN-SKL-003**  
O skill é calculado como a média aritmética de todos os votos válidos recebidos na última votação encerrada.

**RN-SKL-004**  
Votos de jogadores banidos da votação são desconsiderados no cálculo.

**RN-SKL-005**  
O skill histórico de votações anteriores é mantido mas não influencia o skill atual (somente a última votação conta).

**RN-SKL-006**  
O ADMIN pode ajustar o skill de qualquer jogador manualmente, mas o ajuste é registrado como "manual" no histórico.

**RN-SKL-007**  
O skill é específico por grupo: o mesmo usuário pode ter skills diferentes em grupos distintos.

---

## RN-VOT — Votação

**RN-VOT-001**  
Somente uma votação pode estar ativa por grupo em qualquer momento.

**RN-VOT-002**  
Um jogador não pode votar em si mesmo.

**RN-VOT-003**  
Votos são anônimos entre PLAYERs. ADMINs/OWNER podem visualizar os votos individuais.

**RN-VOT-004**  
Jogadores banidos da votação não podem votar e seus votos anteriores nessa votação são invalidados.

**RN-VOT-005**  
O prazo de encerramento é obrigatório ao abrir uma votação.

**RN-VOT-006**  
Uma votação encerrada não pode ser reaberta.

**RN-VOT-007**  
PLAYERs podem alterar seus votos enquanto a votação estiver aberta.

---

## RN-PAG — Pagamentos

**RN-PAG-001**  
Um comprovante de pagamento só pode ser associado a uma cobrança específica.

**RN-PAG-002**  
Cobranças reprovadas permitem reenvio de comprovante. Não há limite de tentativas, mas o histórico é mantido.

**RN-PAG-003**  
Pagamentos aprovados (IA ou manual) não podem ser estornados pelo sistema; exigem intervenção direta do ADMIN com registro de motivo.

**RN-PAG-004**  
Uma cobrança do tipo mensalidade cobre todas as presenças do mês de referência.

**RN-PAG-005**  
A validação por IA deve verificar: (a) valor coincide com a cobrança; (b) chave Pix do grupo consta no comprovante; (c) data do pagamento está dentro do período esperado.

**RN-PAG-006**  
Se a IA não responder em até 10 segundos, o pagamento é automaticamente marcado como Revisão Manual.

**RN-PAG-007**  
Apenas arquivos com tipo MIME image/jpeg, image/png ou application/pdf são aceitos como comprovante.

**RN-PAG-008**  
Tamanho máximo de upload de comprovante: 10 MB.

---

## RN-ARB — Árbitros

**RN-ARB-001**  
Um árbitro deve ser vinculado a um usuário existente no sistema antes de ser cadastrado como árbitro num grupo.

**RN-ARB-002**  
Árbitros não são incluídos na lista de presença como jogadores, portanto não confirmam presença como PLAYER.

**RN-ARB-003**  
Árbitros não participam da formação de times.

**RN-ARB-004**  
A associação de um árbitro a uma partida gera automaticamente uma despesa no financeiro da partida com o valor configurado para o árbitro.

**RN-ARB-005**  
Se o árbitro tiver cobrança mensal, a despesa é registrada uma vez por mês, não por partida.

**RN-ARB-006**  
Um árbitro pode ser desassociado de uma partida pelo ADMIN; a despesa gerada deve ser cancelada manualmente se necessário.

---

## RN-TIM — Formação de Times

**RN-TIM-001**  
A formação de times só pode ser realizada após o fechamento da lista de presença.

**RN-TIM-002**  
Árbitros não participam da formação de times.

**RN-TIM-003**  
O número mínimo de times é 2.

**RN-TIM-004**  
Se o número de jogadores não for divisível pelo número de times, os times com mais jogadores devem ser os primeiros da distribuição.

**RN-TIM-005**  
O algoritmo de formação usa snake draft: distribui jogadores ordenados por skill DESC entre os times em zig-zag (1, 2, 3, ..., N, N, ..., 3, 2, 1, 1, ...).

**RN-TIM-006**  
O desvio padrão do skill médio entre times não deve exceder 0.5 estrelas. Se exceder, o sistema realoca jogadores para equalizar.

**RN-TIM-007**  
ADMINs podem sobrescrever a formação automática movendo jogadores manualmente.

**RN-TIM-008**  
Cada formação de times é registrada no histórico da partida com timestamp e identificação de quem confirmou.

**RN-TIM-009**  
A posição preferencial do jogador é considerada como critério secundário (skill é o critério primário).

---

## RN-FIN — Financeiro

**RN-FIN-001**  
Toda movimentação financeira (receita ou despesa) deve ter: valor, categoria, data, descrição e usuário responsável pelo registro.

**RN-FIN-002**  
Receitas provenientes de pagamentos aprovados são registradas automaticamente.

**RN-FIN-003**  
Despesas de arbitragem são registradas automaticamente ao associar árbitro a partida.

**RN-FIN-004**  
O saldo do grupo é calculado como: Σ(receitas) − Σ(despesas) de todo o histórico.

**RN-FIN-005**  
Movimentações financeiras não podem ser excluídas; somente estornadas com registro de motivo.

**RN-FIN-006**  
Relatórios financeiros devem incluir: total de receitas, total de despesas, saldo e lista de inadimplentes do período.
