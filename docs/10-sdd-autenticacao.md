# SDD — Módulo de Autenticação (Etapa 4, Módulo 1)

**Versão:** 1.0  
**Data:** 2026-06-08  
**Status:** Aprovado  
**Refs:** ADR-003, RF-USR-001–006, US-001–004, RN-USR-001–002, BC-01

---

## 1. Objetivo

Implementar o módulo de Autenticação do backend ArenaHub cobrindo:

1. Registro de conta com email/senha
2. Login com email/senha
3. Login via OAuth2 Google (Authorization Code Flow)
4. Renovação de sessão com refresh token rotativo
5. Logout (revogação de refresh token)
6. Recuperação de senha via email

---

## 2. Escopo

**Incluído neste módulo:**
- Entidade `User` (AR do BC-01)
- Entidade `RefreshToken`
- Todos os endpoints de `/auth/**`
- Spring Security config (JWT filter, CORS, OAuth2)
- Email transacional (confirmação de cadastro, redefinição de senha)
- Migrations Flyway já existentes para `users` e `refresh_tokens`

**Fora do escopo (módulos futuros):**
- Upload de foto de perfil (RF-USR-008)
- Exclusão de conta LGPD (RF-USR-010)
- Endpoints de edição de perfil (RF-USR-007)

---

## 3. Arquitetura em Camadas

```
presentation/auth/
├── AuthController.java              ← POST /auth/register, /login, /refresh, /logout
├── OAuthController.java             ← GET /auth/oauth2/callback
├── PasswordController.java          ← POST /auth/password/forgot, /reset
└── dto/
    ├── RegisterRequest.java
    ├── LoginRequest.java
    ├── AuthResponse.java            ← { accessToken, expiresIn }
    ├── RefreshRequest.java          ← (cookie automático; body vazio)
    ├── ForgotPasswordRequest.java
    └── ResetPasswordRequest.java

application/auth/
├── port/in/
│   ├── RegisterUserUseCase.java
│   ├── LoginUseCase.java
│   ├── OAuthLoginUseCase.java
│   ├── RefreshTokenUseCase.java
│   ├── LogoutUseCase.java
│   ├── ForgotPasswordUseCase.java
│   └── ResetPasswordUseCase.java
├── port/out/
│   ├── UserRepository.java
│   ├── RefreshTokenRepository.java
│   ├── EmailSenderPort.java
│   └── OAuthUserInfoPort.java
└── service/
    ├── AuthService.java             ← implementa Register, Login, Logout, Refresh
    ├── OAuthService.java            ← implementa OAuthLogin
    └── PasswordService.java         ← implementa Forgot/Reset

domain/user/
├── User.java                        ← AR
├── RefreshToken.java                ← Entity
└── vo/
    ├── Email.java
    ├── Name.java
    ├── Phone.java
    └── AuthProvider.java

infrastructure/
├── persistence/
│   ├── UserJpaRepository.java
│   ├── UserJpaEntity.java
│   ├── RefreshTokenJpaRepository.java
│   └── RefreshTokenJpaEntity.java
├── security/
│   ├── SecurityConfig.java
│   ├── JwtAuthFilter.java
│   ├── JwtService.java
│   └── OAuth2SuccessHandler.java
├── email/
│   └── JavaMailEmailAdapter.java
└── oauth/
    └── GoogleOAuthAdapter.java
```

---

## 4. Modelo de Domínio

### 4.1 User (Aggregate Root)

```java
// domain/user/User.java
public class User {
    private UUID id;
    private Name name;
    private Email email;
    private String passwordHash;      // null para GOOGLE
    private Phone phone;
    private LocalDate birthDate;
    private String photoUrl;
    private AuthProvider authProvider; // LOCAL | GOOGLE
    private String googleId;           // null para LOCAL
    private boolean emailVerified;
    private boolean active;
    private Instant createdAt;
    private Instant updatedAt;

    // Factory para registro LOCAL
    public static User registerLocal(Name name, Email email, String rawPassword,
                                     Phone phone, LocalDate birthDate,
                                     PasswordEncoder encoder) { ... }

    // Factory para OAuth Google
    public static User fromGoogle(String googleId, Name name, Email email) { ... }

    public void verifyEmail() { this.emailVerified = true; }
    public void changePassword(String rawPassword, PasswordEncoder encoder) { ... }
}
```

**Invariantes aplicadas na factory:**
- `email` único no sistema (verificado no use case via `UserRepository`)
- `passwordHash` nunca armazenado em texto puro (BCrypt delegado ao factory)
- `authProvider = GOOGLE` implica `passwordHash = null`

### 4.2 RefreshToken (Entity)

```java
// domain/user/RefreshToken.java
public class RefreshToken {
    private UUID id;
    private UUID userId;
    private String tokenHash;          // SHA-256 do token opaco
    private Instant expiresAt;
    private boolean revoked;
    private Instant createdAt;

    public static RefreshToken issue(UUID userId, String rawToken, Duration ttl) { ... }

    public boolean isValid() {
        return !revoked && Instant.now().isBefore(expiresAt);
    }

    public void revoke() { this.revoked = true; }
}
```

**Política de rotação:** ao usar um refresh token, o token atual é revogado e um novo é emitido. Se um token já revogado for apresentado, todos os tokens ativos do usuário são revogados (detecção de reutilização).

### 4.3 Value Objects

| VO | Validação |
|---|---|
| `Email` | Regex RFC 5322, lowercase, máx. 254 chars |
| `Name` | 2–100 chars, não vazio |
| `Phone` | 10–11 dígitos numéricos (DDD + número) |
| `AuthProvider` | Enum `LOCAL \| GOOGLE` |

---

## 5. Endpoints da API

### 5.1 Registro

```
POST /auth/register
Content-Type: application/json

{
  "name": "João Silva",
  "email": "joao@example.com",
  "password": "MinhaSenh@123",
  "phone": "11987654321",
  "birthDate": "1990-05-20"
}
```

**Fluxo:**
1. Valida campos obrigatórios e formatos
2. Verifica unicidade do email (`RN-USR-001`)
3. Cria `User` via `User.registerLocal(...)` (BCrypt aplicado na factory)
4. Persiste o usuário com `emailVerified = false`
5. Envia email de confirmação com link `GET /auth/verify-email?token=<jwt-curto>`
6. Retorna `201 Created` com `AuthResponse` (access token + cookie com refresh token)

**Respostas:**

| Status | Condição |
|---|---|
| 201 | Sucesso |
| 400 | Campos inválidos (bean validation) |
| 409 | Email já cadastrado (`RN-USR-001`) |

---

### 5.2 Login (email/senha)

```
POST /auth/login
Content-Type: application/json

{
  "email": "joao@example.com",
  "password": "MinhaSenh@123"
}
```

**Fluxo:**
1. Busca usuário pelo email
2. Verifica `BCrypt.matches(password, user.passwordHash)`
3. Emite access token (JWT, 15min) + refresh token (opaco, 7d)
4. Persiste `RefreshToken` com hash SHA-256
5. Retorna `AuthResponse` + `Set-Cookie: refresh_token=<token>; HttpOnly; Secure; SameSite=Strict; Path=/auth/refresh; Max-Age=604800`

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso |
| 401 | Credenciais inválidas (mensagem genérica — não revela qual campo) |
| 403 | Conta inativa ou email não verificado |

---

### 5.3 Login via Google (OAuth2)

```
GET /auth/oauth2/authorize/google
```
→ redireciona o browser para o consent screen do Google.

```
GET /auth/oauth2/callback?code=<code>&state=<state>
```

**Fluxo:**
1. Spring Security OAuth2 Client troca `code` por tokens Google
2. `GoogleOAuthAdapter` busca `userinfo` (id, name, email, picture)
3. `OAuthService` faz upsert do `User`:
   - Se `googleId` existe → login normal
   - Se `email` existe com `authProvider = LOCAL` → vincula Google ao usuário existente
   - Caso contrário → cria `User.fromGoogle(...)` com `emailVerified = true`
4. Emite access token + refresh token
5. Redireciona para frontend com `?token=<access_token>` (SPA captura e armazena em memória)

**Regra RN-USR-002:** se o usuário Google não tem `phone`, flag `profileIncomplete = true` no payload do JWT.

---

### 5.4 Renovação de Sessão

```
POST /auth/refresh
Cookie: refresh_token=<token>
```

**Fluxo:**
1. Lê token do cookie HttpOnly (não do body)
2. Busca `RefreshToken` pelo hash SHA-256
3. Se `revoked = true` → detecta reutilização → revoga todos os tokens do usuário → retorna `401`
4. Se expirado → retorna `401`
5. Revoga token atual + emite novo par (access + refresh)
6. Retorna novo `AuthResponse` + novo `Set-Cookie`

**Respostas:**

| Status | Condição |
|---|---|
| 200 | Sucesso (novos tokens) |
| 401 | Token inválido, expirado ou reutilizado |

---

### 5.5 Logout

```
POST /auth/logout
Authorization: Bearer <access_token>
Cookie: refresh_token=<token>
```

**Fluxo:**
1. Extrai refresh token do cookie
2. Revoga o `RefreshToken` no banco
3. Responde com `Set-Cookie: refresh_token=; Max-Age=0` (limpa o cookie)
4. Retorna `204 No Content`

---

### 5.6 Confirmação de Email

```
GET /auth/verify-email?token=<jwt-verificacao>
```

**Fluxo:**
1. Valida o JWT de verificação (claim `purpose = email-verification`, expiração 24h)
2. Busca usuário pelo `sub`
3. Define `emailVerified = true`
4. Redireciona para `/login?emailVerified=true`

---

### 5.7 Recuperação de Senha

```
POST /auth/password/forgot
{ "email": "joao@example.com" }
```

**Fluxo:**
1. Busca usuário pelo email
2. Se não existir: retorna `200` mesmo assim (evita enumeração de emails)
3. Emite JWT de reset (claim `purpose = password-reset`, expiração 1h)
4. Envia email com link `GET /reset-password?token=<jwt>` (rota do frontend)
5. Retorna `200 OK`

```
POST /auth/password/reset
{ "token": "<jwt-reset>", "newPassword": "NovaSenha@456" }
```

**Fluxo:**
1. Valida JWT de reset (`purpose = password-reset`)
2. Verifica que o token não foi usado antes (claim `jti` na blacklist temporária — Redis ou tabela)
3. Atualiza `passwordHash` via `user.changePassword(...)`
4. Invalida todos os refresh tokens do usuário
5. Envia email de confirmação
6. Retorna `200 OK`

---

## 6. JWT

### 6.1 Access Token

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "email": "joao@example.com",
  "profileIncomplete": false,
  "iat": 1749168000,
  "exp": 1749168900,
  "jti": "uuid-unico"
}
```

- Assinado com HMAC-SHA256 (`HS256`)
- Chave de 256 bits lida de `JWT_SECRET` (variável de ambiente)
- Papéis (roles) **não ficam no JWT** — são resolvidos no banco a cada requisição via `UserDetailsService`

### 6.2 Tokens de Propósito Especial

| Tipo | Claim `purpose` | TTL |
|---|---|---|
| Verificação de email | `email-verification` | 24h |
| Redefinição de senha | `password-reset` | 1h |

Esses tokens são JWTs assinados com a mesma chave mas validados com checagem adicional do claim `purpose` para evitar que um tipo seja usado no lugar do outro.

---

## 7. Spring Security

### 7.1 SecurityConfig (esboço)

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(
                "/auth/register", "/auth/login", "/auth/refresh",
                "/auth/verify-email", "/auth/password/**",
                "/auth/oauth2/**", "/actuator/health"
            ).permitAll()
            .anyRequest().authenticated()
        )
        .oauth2Login(oauth2 -> oauth2
            .successHandler(oAuth2SuccessHandler)
        )
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
        .build();
}
```

### 7.2 JwtAuthFilter

- Extrai token do header `Authorization: Bearer <token>`
- Valida assinatura e expiração via `JwtService`
- Carrega `UserDetails` do banco
- Injeta `UsernamePasswordAuthenticationToken` no `SecurityContextHolder`
- Não lança exceção em token ausente (rotas públicas passam direto)

### 7.3 CORS

Origens permitidas lidas de `CORS_ALLOWED_ORIGINS` (variável de ambiente):
- Dev: `http://localhost:5173` (Vite), `http://localhost:8081` (Expo)
- Prod: domínio do frontend

---

## 8. Tratamento de Erros

Todos os erros seguem o formato RFC 7807 (Problem Details):

```json
{
  "type": "/errors/email-already-registered",
  "title": "Email já cadastrado",
  "status": 409,
  "detail": "O email joao@example.com já possui uma conta.",
  "instance": "/auth/register"
}
```

| Código de Negócio | HTTP | Cenário |
|---|---|---|
| `email-already-registered` | 409 | RN-USR-001 |
| `invalid-credentials` | 401 | Login com senha errada |
| `email-not-verified` | 403 | Login antes de verificar email |
| `account-inactive` | 403 | Conta desativada |
| `refresh-token-invalid` | 401 | Token inválido/expirado |
| `refresh-token-reuse-detected` | 401 | Reutilização de token revogado |
| `reset-token-invalid` | 400 | JWT de reset inválido ou já usado |
| `user-not-found` | 404 | (apenas internamente; nunca exposto no forgot-password) |

---

## 9. Email Transacional

| Evento | Template | Conteúdo mínimo |
|---|---|---|
| Registro | `email-verification` | Link de verificação (expira 24h) |
| Forgot password | `password-reset` | Link de redefinição (expira 1h) |
| Senha alterada | `password-changed` | Aviso de segurança com data/hora |

Implementação: `JavaMailSender` (Spring Mail) com templates Thymeleaf.  
Configuração: variáveis `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_FROM`.

---

## 10. Variáveis de Ambiente

| Variável | Descrição | Exemplo |
|---|---|---|
| `JWT_SECRET` | Chave HMAC-SHA256 (mín. 32 chars) | `change-me-in-production-xxxxx` |
| `JWT_ACCESS_TOKEN_TTL` | TTL do access token | `900` (segundos) |
| `JWT_REFRESH_TOKEN_TTL` | TTL do refresh token | `604800` (segundos) |
| `GOOGLE_CLIENT_ID` | OAuth2 Google Client ID | — |
| `GOOGLE_CLIENT_SECRET` | OAuth2 Google Client Secret | — |
| `CORS_ALLOWED_ORIGINS` | Origens permitidas (CSV) | `http://localhost:5173` |
| `MAIL_HOST` | Servidor SMTP | `smtp.gmail.com` |
| `MAIL_PORT` | Porta SMTP | `587` |
| `MAIL_USERNAME` | Usuário SMTP | — |
| `MAIL_PASSWORD` | Senha/app password SMTP | — |
| `MAIL_FROM` | Remetente | `noreply@arenahub.app` |
| `FRONTEND_BASE_URL` | URL do frontend (links nos emails) | `http://localhost:5173` |

---

## 11. Estratégia de Testes

### 11.1 Testes Unitários (JUnit 5 + Mockito)

| Classe | O que testar |
|---|---|
| `User` | Factories, `verifyEmail`, `changePassword`, invariantes de VO |
| `RefreshToken` | `isValid`, `revoke`, lógica de expiração |
| `AuthService` | Login correto, senha errada, email não verificado, rotação de tokens |
| `JwtService` | Emissão, parsing, expiração, `purpose` incorreto |
| `PasswordService` | Forgot (email inexistente retorna 200), token de reset expirado |

### 11.2 Testes de Integração (Testcontainers + PostgreSQL)

| Cenário | Descrição |
|---|---|
| Registro completo | POST /register → email enviado → GET /verify-email → login |
| Login email/senha | Happy path + credencial inválida |
| Refresh token | Rotação → uso do token antigo → detecção de reutilização |
| Logout | Cookie limpo + token revogado |
| Forgot/reset password | Fluxo completo + token já usado |

### 11.3 Meta de Cobertura

- **≥ 80%** (JaCoCo) — conforme regra do projeto
- Classes de domínio e use cases devem atingir ≥ 90%

---

## 12. Ordem de Implementação

1. **Migrations** — verificar que `V001__create_users.sql` e `V002__create_refresh_tokens.sql` existem e estão corretos
2. **Domain** — `User`, `RefreshToken`, Value Objects
3. **Ports** — interfaces de entrada e saída
4. **JwtService** — emissão, validação, extração de claims
5. **AuthService** — register, login, refresh, logout
6. **Security** — `SecurityConfig`, `JwtAuthFilter`, `UserDetailsServiceImpl`
7. **Persistence** — JPA entities, repositories
8. **Email** — `JavaMailEmailAdapter`, templates Thymeleaf
9. **OAuth2** — `OAuth2SuccessHandler`, `GoogleOAuthAdapter`, `OAuthService`
10. **PasswordService** — forgot/reset
11. **Controllers e DTOs**
12. **Testes unitários e de integração**

---

## 13. Dependências Maven (já no pom.xml ou a adicionar)

```xml
<!-- Segurança -->
<dependency>spring-boot-starter-security</dependency>
<dependency>spring-boot-starter-oauth2-client</dependency>

<!-- JWT -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>

<!-- Email -->
<dependency>spring-boot-starter-mail</dependency>
<dependency>spring-boot-starter-thymeleaf</dependency>

<!-- Testes -->
<dependency>spring-security-test</dependency>
<dependency>testcontainers (postgresql)</dependency>
```
