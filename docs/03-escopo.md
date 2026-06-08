# Escopo — ArenaHub

## Escopo do MVP (v1.0)

### Dentro do Escopo

#### Autenticação e Usuários
- Cadastro com email e senha
- Login OAuth2 via Google
- Gerenciamento de perfil (nome, telefone, foto, data de nascimento)
- Recuperação de senha

#### Grupos Esportivos
- Criação de grupo com modalidade (Futebol, Vôlei, Basquete, Futevôlei, Beach Tennis, Outros)
- Convite de jogadores por link ou email
- Gerenciamento de papéis: OWNER, ADMIN, PLAYER, REFEREE
- Configurações do grupo (nome, foto, descrição)

#### Presença
- Criação de partidas com data, horário e local
- Abertura de lista de presença
- Confirmação e recusa de presença pelos jogadores
- Fechamento automático da lista 1 hora antes da partida
- Banimento de jogador de lista (com motivo e aprovação de justificativa)

#### Votação de Habilidades (Skill)
- Escala de 1 a 6 estrelas por jogador
- Abertura e encerramento de votação pelo administrador
- Prazo configurável para votação
- Banimento de jogador da votação (com motivo)
- Cálculo automático de skill médio após encerramento

#### Formação de Times
- Distribuição automática por skill e posição
- Suporte às posições por modalidade
- Histórico de formações por partida
- Visualização dos times formados

#### Pagamentos
- Registro de cobrança (diária ou mensalidade)
- Upload de comprovante de pagamento (imagem)
- Validação automática por IA (valor, chave Pix, data)
- Status de pagamento: Aprovado, Reprovado, Revisão Manual
- Aprovação manual pelo administrador

#### Mensalistas
- Marcação de jogador como mensalista
- Pagamento mensal antecipado
- Confirmação de presença sem novo pagamento dentro do mês pago

#### Árbitros
- Cadastro vinculado a um usuário
- Tipos de cobrança: por partida ou mensal
- Valor configurável
- Associação de árbitro a partidas
- Árbitros não participam da formação de times

#### Financeiro
- Registro de receitas (diárias e mensalidades)
- Registro de despesas (árbitros, quadra, outras)
- Relatórios por partida, mês e ano
- Saldo do grupo

#### Infraestrutura Multi-tenant
- Isolamento completo de dados por grupo (tenant)
- Estratégia de tenant via campo `group_id` nas tabelas

### Fora do Escopo (v1.0)

| Funcionalidade | Versão Prevista |
|---|---|
| Processamento direto de pagamentos (gateway) | v2.0 |
| Chat/mensagens integradas | v2.0 |
| Integração com WhatsApp Business API | v2.0 |
| Transmissão ao vivo de partidas | Backlog |
| Estatísticas avançadas (gols, assistências, cartões) | v1.5 |
| Ranking histórico público | v1.5 |
| App mobile offline-first | v2.0 |
| Múltiplos grupos por organizador em painel unificado | v2.0 |
| Marketplace de árbitros | Backlog |
| Integração com plataformas de reserva de quadra | Backlog |
| Suporte a múltiplos idiomas | Backlog |

## Fronteiras do Sistema

```
[Jogador/Administrador]
       │
       ▼
[ArenaHub Web / Mobile]
       │
       ├── Auth: Google OAuth2 / Email+Senha
       │
       ├── Storage: Cloudflare R2 (fotos, comprovantes)
       │
       ├── IA: Validação de comprovantes (integração externa)
       │
       └── BD: PostgreSQL (dados transacionais)
```

## Dependências Externas

| Dependência | Propósito | Risco |
|---|---|---|
| Google OAuth2 | Autenticação social | Baixo — fallback é email/senha |
| Cloudflare R2 | Armazenamento de arquivos | Médio — necessita conta e configuração |
| Provedor de IA | Validação de comprovantes | Alto — fallback obrigatório para revisão manual |
| Railway | Hospedagem inicial | Médio — migração futura para AWS planejada |
