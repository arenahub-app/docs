# Requisitos Não Funcionais — ArenaHub

## RNF-PER — Performance

| ID | Descrição | Métrica |
|---|---|---|
| RNF-PER-001 | Tempo de resposta das APIs para operações de leitura | P95 ≤ 300ms |
| RNF-PER-002 | Tempo de resposta das APIs para operações de escrita | P95 ≤ 500ms |
| RNF-PER-003 | Tempo de upload e início de validação de comprovante | ≤ 5s |
| RNF-PER-004 | Tempo de carregamento inicial da aplicação web (FCP) | ≤ 2s em conexão 4G |
| RNF-PER-005 | O sistema deve suportar ao menos 100 requisições simultâneas sem degradação | Carga: 100 req/s |

---

## RNF-DIS — Disponibilidade e Confiabilidade

| ID | Descrição | Métrica |
|---|---|---|
| RNF-DIS-001 | Disponibilidade mínima do sistema em produção | ≥ 99,5% ao mês |
| RNF-DIS-002 | Janela de manutenção planejada deve ser comunicada com antecedência | ≥ 24h de aviso |
| RNF-DIS-003 | Em caso de falha da IA de validação, o sistema deve colocar o pagamento em Revisão Manual | Fallback obrigatório |
| RNF-DIS-004 | O fechamento automático de lista de presença não pode depender de interação humana | Scheduler confiável |

---

## RNF-SEG — Segurança

| ID | Descrição | Detalhes |
|---|---|---|
| RNF-SEG-001 | Todas as comunicações devem usar TLS 1.2+ | HTTPS obrigatório em produção |
| RNF-SEG-002 | Senhas devem ser armazenadas com hash BCrypt (custo ≥ 12) | Nunca em texto plano |
| RNF-SEG-003 | JWTs devem ter expiração curta e usar refresh tokens | Access: 15min; Refresh: 7d |
| RNF-SEG-004 | Refresh tokens devem ser rotativos (rotation strategy) | Previne reutilização após comprometimento |
| RNF-SEG-005 | Dados de um tenant nunca devem ser acessíveis por outro tenant | Isolamento por group_id validado na camada de aplicação |
| RNF-SEG-006 | APIs devem validar autorização por papel em cada endpoint | Spring Security + método de autorização |
| RNF-SEG-007 | Upload de arquivos deve ter validação de tipo MIME e tamanho máximo | Máx: 10MB; tipos: JPEG, PNG, PDF |
| RNF-SEG-008 | Inputs do usuário devem ser sanitizados para prevenir XSS e SQL Injection | Validação via Bean Validation + Parameterized queries |
| RNF-SEG-009 | Tokens de convite de grupo devem ter prazo de expiração | Expiração: 7 dias |
| RNF-SEG-010 | Logs não devem conter dados sensíveis (CPF, senha, token) | Mascaramento obrigatório |

---

## RNF-ESC — Escalabilidade

| ID | Descrição | Detalhes |
|---|---|---|
| RNF-ESC-001 | A arquitetura deve suportar escalabilidade horizontal da aplicação backend | Stateless por design (JWT) |
| RNF-ESC-002 | O banco de dados deve ser provisionado com estratégia de connection pool | HikariCP configurado |
| RNF-ESC-003 | Armazenamento de arquivos deve ser desacoplado da aplicação | Cloudflare R2 / S3-compatible |
| RNF-ESC-004 | A estrutura de multi-tenancy deve suportar crescimento sem refatoração de schema | Tenant por group_id (shared schema) |

---

## RNF-MAN — Manutenibilidade

| ID | Descrição | Meta |
|---|---|---|
| RNF-MAN-001 | Cobertura mínima de testes unitários e de integração | ≥ 80% |
| RNF-MAN-002 | O código deve ser analisado por SonarQube a cada PR | Quality Gate aprovado |
| RNF-MAN-003 | APIs devem ser documentadas com OpenAPI 3.x | Swagger UI disponível em /docs |
| RNF-MAN-004 | Todas as migrações de banco de dados devem ser gerenciadas pelo Flyway | Sem DDL manual em produção |
| RNF-MAN-005 | Decisões arquiteturais devem ser registradas em ADRs | Mínimo 1 ADR por decisão significativa |
| RNF-MAN-006 | O sistema deve ter logs estruturados (JSON) com nível configurável | INFO em prod; DEBUG em dev |

---

## RNF-OPE — Operabilidade

| ID | Descrição | Detalhes |
|---|---|---|
| RNF-OPE-001 | O sistema deve expor endpoint de health check | GET /actuator/health |
| RNF-OPE-002 | O sistema deve expor métricas no padrão Prometheus | GET /actuator/metrics |
| RNF-OPE-003 | Ambiente de desenvolvimento deve ser inicializável com um único comando | docker compose up |
| RNF-OPE-004 | Variáveis de ambiente devem ser externalizadas (não hardcoded) | .env com .env.example versionado |
| RNF-OPE-005 | Deploy deve ser automatizado via GitHub Actions | Push em main = deploy em staging |

---

## RNF-USE — Usabilidade

| ID | Descrição | Meta |
|---|---|---|
| RNF-USE-001 | O fluxo de confirmação de presença deve ser completável em no máximo 3 toques no mobile | UX mobile-first |
| RNF-USE-002 | A interface deve ser responsiva para dispositivos móveis e desktop | Breakpoints: 360px, 768px, 1280px |
| RNF-USE-003 | Mensagens de erro devem ser claras e orientadas à ação do usuário | Sem stacktraces expostos ao usuário |
| RNF-USE-004 | O sistema deve suportar idioma pt-BR | Internacionalização básica |

---

## RNF-LEG — Conformidade Legal

| ID | Descrição | Referência |
|---|---|---|
| RNF-LEG-001 | O sistema deve estar em conformidade com a LGPD | Lei 13.709/2018 |
| RNF-LEG-002 | O usuário deve poder solicitar exclusão de seus dados (direito ao esquecimento) | LGPD Art. 18 |
| RNF-LEG-003 | O sistema deve registrar o consentimento do usuário para uso de dados | Termos de uso e política de privacidade |
| RNF-LEG-004 | Dados pessoais não devem ser compartilhados entre tenants (grupos) | Isolamento de dados por tenant |
| RNF-LEG-005 | Logs de acesso devem ser retidos por no mínimo 6 meses | Auditoria de segurança |

---

## RNF-INT — Integração

| ID | Descrição | Detalhes |
|---|---|---|
| RNF-INT-001 | Integração com Google OAuth2 deve usar o fluxo Authorization Code | RFC 6749 |
| RNF-INT-002 | Armazenamento de arquivos deve usar API S3-compatible | Cloudflare R2 |
| RNF-INT-003 | A integração com IA de validação deve ter timeout configurável | Default: 10s; fallback: Revisão Manual |
| RNF-INT-004 | Integrações externas devem ser encapsuladas em adapters (Clean Architecture) | Anticorruption Layer |
