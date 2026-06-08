# ADR-003 — Autenticação: JWT Stateless + OAuth2 Google

**Status:** Aceito  
**Data:** 2026-06-07  
**Decisores:** Time de Arquitetura

---

## Contexto

O sistema precisa autenticar usuários em:
- Aplicação web (React)
- App mobile (React Native)
- APIs REST (futuros integradores)

Requisitos:
- Suporte a email/senha e OAuth2 Google
- Stateless para facilitar escala horizontal
- Sessão renovável sem re-login frequente
- Segurança contra roubo de tokens

## Decisão

**Estratégia:** JWT stateless com Access Token + Refresh Token rotativo

| Token | Duração | Armazenamento (cliente) |
|---|---|---|
| Access Token | 15 minutos | Memória da aplicação (não localStorage) |
| Refresh Token | 7 dias | HttpOnly Cookie (web) / SecureStorage (mobile) |

**OAuth2 Google:** Authorization Code Flow com PKCE (para mobile)

### Fluxo de Autenticação

```
1. Login (email/senha ou Google)
   └─► Backend emite: access_token (JWT, 15min) + refresh_token (opaco, 7d)
   └─► refresh_token armazenado como HttpOnly Cookie

2. Requisições autenticadas
   └─► Header: Authorization: Bearer <access_token>

3. Access token expirado (401)
   └─► Cliente envia refresh_token (via Cookie automático)
   └─► Backend valida refresh_token, emite novo access_token + NOVO refresh_token (rotação)
   └─► Refresh token anterior é invalidado

4. Logout
   └─► Backend invalida o refresh_token no banco (blacklist)
   └─► Cookie removido
```

### Rotação de Refresh Tokens

Refresh tokens são armazenados no banco de dados (tabela `refresh_tokens`) com:
- Token (hash)
- Usuário
- Expiração
- Status (ACTIVE / REVOKED)

A cada uso, o token atual é revogado e um novo é emitido. Se um token revogado for usado, todos os tokens da sessão são invalidados (detecção de reutilização).

### JWT Payload

```json
{
  "sub": "uuid-do-usuario",
  "email": "user@example.com",
  "iat": 1717776000,
  "exp": 1717776900,
  "jti": "uuid-unico-do-token"
}
```

*Papéis (roles) são resolvidos no banco a cada requisição; não ficam no JWT para evitar tokens desatualizados.*

## Alternativas Consideradas

### Opção A: Sessões no servidor (Spring Session + Redis)
- **Prós:** Invalidação imediata; sem problema de tokens expirados em cache
- **Contras:** Estado no servidor; requer Redis; complica escala horizontal; não funciona nativamente para mobile

### Opção B: JWT de longa duração (sem refresh)
- **Prós:** Simples de implementar
- **Contras:** Token comprometido fica válido por horas/dias; não recomendado para dados sensíveis

### Opção C: JWT stateless + Refresh Token (escolhida)
- **Prós:** Stateless; funciona em web e mobile; refresh token permite invalidação; rotação detecta roubo
- **Contras:** Blacklist de refresh tokens requer persistência; um pouco mais complexo de implementar

## Consequências

**Positivas:**
- Backend stateless → escala horizontal trivial
- Access token curto minimiza janela de ataque
- Rotação de refresh token detecta reutilização maliciosa

**Negativas:**
- Tabela de refresh tokens requer limpeza periódica (job de expiração)
- Cliente precisa implementar lógica de renovação automática

## Bibliotecas

- `spring-boot-starter-oauth2-resource-server` — validação de JWT
- `spring-security-oauth2-client` — fluxo OAuth2 Google
- `jjwt` (io.jsonwebtoken) — emissão e parsing de JWT
