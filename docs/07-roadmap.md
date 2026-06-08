# Roadmap — ArenaHub

## Visão de Horizonte

| Horizonte | Período | Foco |
|---|---|---|
| H1 — MVP | Meses 1–4 | Funcionalidades core para validar o produto com grupos reais |
| H2 — Crescimento | Meses 5–8 | Expansão de funcionalidades e melhorias com base em feedback |
| H3 — Escala | Meses 9–12 | Relatórios avançados, integrações e preparação para escala |

---

## H1 — MVP (Meses 1–4)

### Etapa 1 — Infraestrutura e Documentação (Semanas 1–2)
- [x] Estrutura de diretórios
- [x] Documentação de produto
- [x] Especificações funcionais e não-funcionais
- [x] ADRs iniciais
- [ ] Diagrama de domínio e ER
- [ ] Setup de CI/CD básico

### Etapa 2 — Modelagem de Domínio (Semanas 3–4)
- [ ] Entidades e agregados DDD
- [ ] Diagrama ER completo
- [ ] Value Objects
- [ ] Migrations Flyway iniciais

### Etapa 3 — Backend — Auth + Usuários (Semanas 5–6)
- [ ] Cadastro e login (email/senha)
- [ ] OAuth2 Google
- [ ] JWT com refresh token
- [ ] Gestão de perfil

### Etapa 4 — Backend — Grupos e Membros (Semana 7)
- [ ] CRUD de grupos
- [ ] Convites e membros
- [ ] Papéis (OWNER, ADMIN, PLAYER, REFEREE)

### Etapa 5 — Backend — Presença (Semana 8)
- [ ] CRUD de partidas
- [ ] Confirmação/recusa de presença
- [ ] Fechamento automático de lista
- [ ] Fila de espera
- [ ] Banimento de presença com justificativa

### Etapa 6 — Backend — Skill e Votação (Semanas 9–10)
- [ ] Skill por jogador
- [ ] Abertura/encerramento de votação
- [ ] Votação anônima
- [ ] Banimento de votação
- [ ] Cálculo automático de média

### Etapa 7 — Backend — Pagamentos (Semanas 11–12)
- [ ] Cobranças (diária e mensalidade)
- [ ] Upload de comprovante (R2)
- [ ] Integração com IA de validação
- [ ] Fluxo de revisão manual
- [ ] Mensalistas

### Etapa 8 — Backend — Árbitros e Financeiro (Semanas 13–14)
- [ ] Cadastro de árbitros
- [ ] Associação a partidas
- [ ] Registro de despesas
- [ ] Relatórios financeiros por período

### Etapa 9 — Backend — Formação de Times (Semana 15)
- [ ] Algoritmo snake draft por skill
- [ ] Cobertura de posições por modalidade
- [ ] Histórico de formações

### Etapa 10 — Frontend Web (Semanas 12–18, paralelo ao backend)
- [ ] Design System + componentes base
- [ ] Login e cadastro
- [ ] Dashboard
- [ ] Gestão de grupos e membros
- [ ] Presença (admin e player)
- [ ] Votação de skill
- [ ] Pagamentos
- [ ] Financeiro e relatórios
- [ ] Formação de times

### Etapa 11 — Lançamento Beta (Semana 18)
- [ ] Deploy em Railway
- [ ] Onboarding de 3–5 grupos beta
- [ ] Coleta de feedback estruturado

---

## H2 — Crescimento (Meses 5–8)

### Mobile (React Native / Expo)
- [ ] Login e cadastro
- [ ] Confirmação de presença (fluxo principal)
- [ ] Upload de comprovante
- [ ] Visualização de times
- [ ] Perfil do jogador
- [ ] Notificações push

### Funcionalidades de Maturidade
- [ ] Exportação de relatórios (PDF/CSV)
- [ ] Histórico de partidas com estatísticas básicas
- [ ] Painel de inadimplência
- [ ] Configurações avançadas de grupo

### Infraestrutura
- [ ] SonarQube integrado ao CI
- [ ] Ambientes homolog e prod separados
- [ ] Monitoramento básico (uptime, alertas)

---

## H3 — Escala (Meses 9–12)

### Estatísticas Avançadas
- [ ] Gols, assistências, cartões por partida
- [ ] MVP da partida (votação)
- [ ] Ranking histórico de jogadores
- [ ] Defesas por goleiro

### Integrações
- [ ] Notificações via WhatsApp Business API
- [ ] Integração com plataformas de reserva de quadra
- [ ] Webhook para pagamentos confirmados

### Planos e Monetização
- [ ] Implementação de planos (Freemium / Pro / Club)
- [ ] Billing e controle de limites por plano
- [ ] Painel administrativo interno (backoffice)

### Infraestrutura de Escala
- [ ] Migração de Railway para AWS (ECS / RDS / S3)
- [ ] CDN para assets estáticos
- [ ] Estratégia de cache (Redis)
- [ ] Observabilidade completa (traces, métricas, logs)

---

## Marcos de Entrega (Milestones)

| Marco | Data Alvo | Critério de Conclusão |
|---|---|---|
| M1 — Documentação Completa | Semana 2 | Todos os docs, specs e ADRs revisados e aprovados |
| M2 — Modelagem Aprovada | Semana 4 | Diagrama ER, entidades e migrations revisados |
| M3 — Auth + Grupos Funcionando | Semana 8 | APIs testadas; fluxo de cadastro + grupo E2E funcionando |
| M4 — Core do MVP Backend | Semana 15 | Todos os módulos backend com cobertura ≥ 80% |
| M5 — Frontend Web MVP | Semana 18 | Todas as telas do MVP funcionando em staging |
| M6 — Beta Launch | Semana 18 | Primeiro grupo externo usando o sistema em produção |
| M7 — Mobile v1 | Mês 7 | App publicado (TestFlight/Play Store interno) |
| M8 — GA v1.0 | Mês 9 | Produto estável com 50+ grupos ativos |
