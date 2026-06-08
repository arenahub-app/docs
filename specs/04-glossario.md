# Glossário — ArenaHub

## Termos de Domínio

### Grupo Esportivo
Entidade principal do sistema. Representa uma turma organizada que pratica uma modalidade esportiva regularmente. Cada grupo tem configurações independentes, membros, partidas e financeiro próprio. É a unidade de tenant no sistema multi-tenant.

### Tenant
No contexto do ArenaHub, cada **Grupo Esportivo** é um tenant. Os dados de um grupo são isolados dos dados de outros grupos.

### Modalidade
Esporte praticado pelo grupo. Modalidades suportadas:
- **Futebol** — campo, society ou salão
- **Vôlei** — quadra ou praia
- **Basquete**
- **Futevôlei**
- **Beach Tennis**
- **Outros** — categoria genérica para esportes não listados

### Membro
Usuário que pertence a um grupo esportivo. Um usuário pode ser membro de múltiplos grupos com papéis independentes.

### Papel (Role)
Permissão atribuída a um membro dentro de um grupo específico:

| Papel | Descrição |
|---|---|
| **OWNER** | Dono do grupo. Possui todas as permissões. Criado automaticamente ao criar o grupo. Único por grupo. |
| **ADMIN** | Administrador. Possui permissões de gestão (partidas, presenças, pagamentos, times). Promovido pelo OWNER. |
| **PLAYER** | Jogador. Pode confirmar presença, votar e registrar pagamentos. |
| **REFEREE** | Árbitro. Vinculado ao grupo; não participa de times, presenças ou votações como jogador. |

### Partida
Evento esportivo criado pelo ADMIN dentro de um grupo. Possui data, horário, local e capacidade máxima de jogadores.

### Lista de Presença
Lista associada a uma partida que controla quem irá participar. Abre na criação da partida e fecha automaticamente 1 hora antes do horário da partida.

### Fila de Espera
Fila formada quando a lista de presença atinge o limite máximo de jogadores. Jogadores que confirmam quando a lista está cheia entram na fila por ordem de inscrição (FIFO).

### Skill (Habilidade)
Nota de 1 a 6 estrelas que representa o nível de habilidade de um jogador dentro de um grupo específico. Calculada como média dos votos na última votação encerrada. Padrão inicial: 3.0.

### Votação de Habilidades
Processo periódico aberto pelo ADMIN onde os jogadores do grupo votam na habilidade uns dos outros. Resultado atualiza o skill de cada jogador.

### Banimento de Presença
Restrição aplicada pelo ADMIN que impede um jogador de confirmar presença sem passar pelo fluxo de justificativa e aprovação. Sem prazo automático.

### Banimento de Votação
Restrição aplicada pelo ADMIN que impede um jogador de votar em uma votação específica. Votos anteriores do banido são invalidados.

### Mensalista
Jogador que paga uma mensalidade fixa em vez de diárias por partida. Com mensalidade paga no mês, pode confirmar presença em qualquer partida sem novo pagamento.

### Diária
Cobrança por partida. Paga individualmente para cada partida que o jogador vai participar.

### Mensalidade
Cobrança mensal recorrente que cobre todas as presenças do mês.

### Comprovante de Pagamento
Arquivo (imagem ou PDF) enviado pelo jogador como prova de pagamento via Pix.

### Validação por IA
Processo automático de verificação do comprovante de pagamento. Analisa: valor, chave Pix do grupo e data.

### Árbitro
Membro com papel REFEREE. Associado a partidas para apitar. Tem cobrança própria (por partida ou mensal). Não participa de times, presenças ou votações como jogador.

### Formação de Times
Processo de divisão dos jogadores presentes em uma partida em times equilibrados, considerando skill e posição.

### Snake Draft
Algoritmo de distribuição em que jogadores são ordenados por skill e distribuídos entre times em ordem zig-zag: time 1, 2, 3, ..., N, N, ..., 3, 2, 1, repetindo.

### Financeiro do Grupo
Módulo que registra todas as receitas (pagamentos aprovados) e despesas (árbitros, quadra, outras) do grupo. Permite visualização de saldo e relatórios por período.

---

## Posições por Modalidade

### Futebol
| Posição | Abreviação |
|---|---|
| Goleiro | GOL |
| Zagueiro | ZAG |
| Lateral | LAT |
| Volante | VOL |
| Meia | MEI |
| Atacante | ATA |

### Vôlei
| Posição | Abreviação |
|---|---|
| Levantador | LEV |
| Ponteiro | PON |
| Central | CEN |
| Oposto | OPO |
| Líbero | LIB |

### Basquete
| Posição | Abreviação |
|---|---|
| Armador | ARM |
| Ala-Armador | ALA-ARM |
| Ala | ALA |
| Ala-Pivô | ALA-PIV |
| Pivô | PIV |

### Futevôlei
| Posição | Abreviação |
|---|---|
| Atacante | ATA |
| Defensor | DEF |

### Beach Tennis
| Posição | Abreviação |
|---|---|
| Lado Direito | DIR |
| Lado Esquerdo | ESQ |

### Outros
Sem posições predefinidas. ADMIN configura livremente.

---

## Acrônimos e Termos Técnicos

| Termo | Significado |
|---|---|
| SaaS | Software as a Service |
| JWT | JSON Web Token |
| OAuth2 | Open Authorization 2.0 |
| DDD | Domain-Driven Design |
| SDD | Spec Driven Development |
| ADR | Architecture Decision Record |
| R2 | Cloudflare R2 (object storage S3-compatible) |
| ER | Entity-Relationship (diagrama de entidades e relacionamentos) |
| CI/CD | Continuous Integration / Continuous Delivery |
| LGPD | Lei Geral de Proteção de Dados (Lei 13.709/2018) |
| FIFO | First In, First Out (fila: primeiro a entrar, primeiro a sair) |
| P95 | Percentil 95 (tempo que 95% das requisições ficam abaixo) |
| MRR | Monthly Recurring Revenue |
| NPS | Net Promoter Score |
| ARPU | Average Revenue Per User |
