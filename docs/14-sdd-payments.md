# SDD — Módulo de Pagamentos (Etapa 5, Feature 5)

**Versão:** 1.0  
**Data:** 2026-06-17  
**Status:** Em revisão  
**Refs:** UC-012, UC-013, RN-PAG-001–008, BC-06, BC-07

---

## 1. Objetivo

Implementar o módulo de Pagamentos do ArenaHub, cobrindo:

1. Geração automática de cobrança DAILY quando jogador confirma presença em partida com valor de diária configurado
2. Envio de comprovante de Pix pelo jogador via plataforma web
3. Aprovação e rejeição de comprovantes pelo admin
4. Registro manual de pagamento pelo admin (sem comprovante — ex: dinheiro no dia)
5. Geração automática de entrada financeira (BC-07) ao aprovar cobrança

Este módulo altera o fluxo de confirmação de presença (BC-03) para criar cobranças quando o grupo tem `matchFee` configurado.

---

## 2. Escopo

**Incluído neste módulo:**
- Aggregate `Charge` com entidade `PaymentAttempt` (BC-06)
- Aggregate `FinancialEntry` gerado automaticamente ao aprovar cobrança (BC-07)
- 7 endpoints REST (`/api/v1/groups/{groupId}/charges/**`)
- Alteração no endpoint de confirmação de presença (retorna cobrança criada)
- Upload de comprovante no Cloudflare R2 (JPEG, PNG, PDF — máx. 10 MB)
- 3 telas no frontend
- Migrations V016 (alterações) — V011 e V012 já existem

**Fora do escopo (iterações futuras):**
- Validação de comprovante por IA (Claude Vision)
- Bot do Telegram como canal alternativo de envio de comprovante
- Cobranças SUBSCRIPTION (mensalistas)
- Geração automática de cobranças ao criar a partida
- Notificações push ao jogador (dependente de BC Mensageria — 4.7)
- Relatório financeiro completo (UC-018 — Feature 6)

---

## 3. Dependências

| Módulo | Tipo | Detalhe |
|---|---|---|
| BC-02 Group Management | Leitura | `Group.matchFee` e `Group.pixKey`; validação de role (ADMIN/OWNER vs PLAYER) |
| BC-03 Match & Presence | Modificação | Confirmação de presença cria Charge DAILY se `matchFee > 0` |
| BC-07 Financial | Escrita | `FinancialEntry` criada automaticamente ao aprovar Charge |
| Cloudflare R2 | Infra | Upload de comprovante; `StoragePort` já implementado |

---

## 4. Arquitetura em Camadas

```
presentation/payment/
├── ChargeController.java             ← /api/v1/groups/{groupId}/charges/**
└── dto/
    ├── ChargeResponse.java
    ├── ChargeDetailResponse.java
    ├── PaymentAttemptResponse.java
    ├── SubmitReceiptRequest.java      ← multipart/form-data
    ├── ReviewAttemptRequest.java
    └── ManualApprovalRequest.java

application/payment/
├── usecase/
│   ├── GetGroupChargesUseCase.java
│   ├── GetMatchChargesUseCase.java
│   ├── GetMyChargesUseCase.java
│   ├── GetChargeDetailUseCase.java
│   ├── SubmitReceiptUseCase.java
│   ├── ApproveAttemptUseCase.java
│   ├── RejectAttemptUseCase.java
│   └── ManualApproveChargeUseCase.java
├── port/
│   ├── in/  (interfaces dos use cases acima)
│   └── out/
│       ├── ChargeRepository.java
│       ├── FinancialEntryRepository.java
│       └── PresencePort.java          ← cross-BC: confirma presença após aprovação
└── dto/  (application-layer DTOs)

domain/payment/
├── Charge.java                       ← AR
├── PaymentAttempt.java               ← Entity dentro de Charge
└── vo/
    ├── ChargeType.java               ← DAILY | SUBSCRIPTION
    ├── ChargeStatus.java             ← PENDING | APPROVED | REJECTED
    ├── ValidationResult.java         ← APPROVED | REJECTED | MANUAL_REVIEW
    └── ValidationSource.java         ← AI | MANUAL

domain/financial/
└── FinancialEntry.java               ← AR — criado pelo payment module

infrastructure/payment/
├── ChargeJpaRepository.java
├── FinancialEntryJpaRepository.java
└── PresencePortImpl.java             ← chama PresenceEntryRepository para confirmar
```

---

## 5. Alterações no Banco de Dados

### Migration V016 — alter_groups_match_fee_and_payment_attempts

```sql
-- Valor da diária do grupo (NULL = sem cobrança)
ALTER TABLE groups
    ADD COLUMN match_fee NUMERIC(10, 2);

ALTER TABLE groups
    ADD CONSTRAINT chk_groups_match_fee
        CHECK (match_fee IS NULL OR match_fee > 0);

COMMENT ON COLUMN groups.match_fee IS
    'Valor cobrado por partida (diária). NULL = grupo sem cobrança obrigatória de pagamento.';

-- Permite file_key nulo para aprovações manuais sem comprovante
ALTER TABLE payment_attempts
    ALTER COLUMN file_key DROP NOT NULL;

ALTER TABLE payment_attempts
    ADD CONSTRAINT chk_payment_attempts_file_required
        CHECK (validation_source = 'MANUAL' OR file_key IS NOT NULL);

COMMENT ON COLUMN payment_attempts.file_key IS
    'Chave do arquivo no R2. NULL apenas quando validation_source = MANUAL (aprovação sem comprovante).';

-- Trigger updated_at em charges (não existia na V011)
CREATE TRIGGER trg_updated_at_charges
    BEFORE UPDATE ON charges
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
```

> As tabelas `charges`, `payment_attempts` e `financial_entries` foram criadas nas migrations V011 e V012 já aplicadas em produção.

---

## 6. Alteração no Fluxo de Confirmação de Presença

O endpoint existente `POST /api/v1/groups/{groupId}/matches/{matchId}/presences` (BC-03) passa a executar a lógica abaixo após confirmar a presença:

```
se group.matchFee != null && group.matchFee > 0:
    se NÃO existe Charge PENDING ou APPROVED para (memberId, matchId):
        criar Charge{
            groupId, memberId, type=DAILY,
            amount=group.matchFee,
            referenceMatchId=matchId,
            status=PENDING
        }
        incluir ChargeResponse no campo "pendingCharge" da resposta
```

A presença é confirmada normalmente (PresenceEntry com status CONFIRMED). O pagamento é exigido, mas a vaga é ocupada imediatamente. A lista de presença exibe o status de pagamento separadamente.

### Resposta ampliada do endpoint de presença

```json
{
  "presenceId": "uuid",
  "matchId": "uuid",
  "memberId": "uuid",
  "status": "CONFIRMED",
  "confirmedAt": "2026-06-17T20:00:00Z",
  "pendingCharge": {
    "chargeId": "uuid",
    "amount": 25.00,
    "pixKey": "chave-pix-do-grupo",
    "status": "PENDING"
  }
}
```

`pendingCharge` é `null` quando o grupo não exige pagamento ou o jogador já tem cobrança existente para a partida.

---

## 7. Contratos de API

### 7.1 Listar cobranças do grupo

```
GET /api/v1/groups/{groupId}/charges
```

**Autorização:** OWNER, ADMIN (lista todas); PLAYER (lista só as suas — equivalente a `/mine`)  
**Query params:**
- `status` — PENDING | APPROVED | REJECTED (opcional)
- `matchId` — UUID (opcional)
- `page`, `size` (default 20)

**Response 200:**
```json
{
  "content": [
    {
      "chargeId": "uuid",
      "memberId": "uuid",
      "memberName": "João Silva",
      "type": "DAILY",
      "amount": 25.00,
      "referenceMatchId": "uuid",
      "matchDate": "2026-06-20T19:00:00Z",
      "status": "PENDING",
      "latestAttemptStatus": "MANUAL_REVIEW",
      "createdAt": "2026-06-17T18:00:00Z"
    }
  ],
  "totalElements": 42,
  "totalPages": 3
}
```

---

### 7.2 Listar cobranças de uma partida

```
GET /api/v1/groups/{groupId}/matches/{matchId}/charges
```

**Autorização:** OWNER, ADMIN  
**Response 200:** lista de `ChargeResponse` para a partida especificada

---

### 7.3 Detalhe de uma cobrança

```
GET /api/v1/groups/{groupId}/charges/{chargeId}
```

**Autorização:** OWNER, ADMIN (qualquer); PLAYER (somente as suas)  
**Response 200:**
```json
{
  "chargeId": "uuid",
  "memberId": "uuid",
  "memberName": "João Silva",
  "type": "DAILY",
  "amount": 25.00,
  "pixKey": "chave-pix-do-grupo",
  "referenceMatchId": "uuid",
  "matchDate": "2026-06-20T19:00:00Z",
  "status": "PENDING",
  "createdAt": "2026-06-17T18:00:00Z",
  "attempts": [
    {
      "attemptId": "uuid",
      "fileUrl": "https://...",
      "contentType": "image/jpeg",
      "submittedAt": "2026-06-17T19:00:00Z",
      "validationResult": "MANUAL_REVIEW",
      "validationSource": "AI",
      "reviewNote": null
    }
  ]
}
```

---

### 7.4 Enviar comprovante

```
POST /api/v1/groups/{groupId}/charges/{chargeId}/payment-attempts
Content-Type: multipart/form-data
```

**Autorização:** membro dono da cobrança (PLAYER, ADMIN ou OWNER com a cobrança vinculada ao próprio memberId)  
**Body:** `file` (multipart — JPEG, PNG ou PDF, máx. 10 MB)

**Regras:**
- Charge deve ter status PENDING
- Charge deve pertencer ao memberId do usuário autenticado
- Se já existe uma tentativa com resultado MANUAL_REVIEW pendente de revisão, rejeitar com 409 (aguardar revisão do admin)
- Arquivo é salvo em R2 com chave `payments/{groupId}/{chargeId}/{uuid}.{ext}`

**Response 201:**
```json
{
  "attemptId": "uuid",
  "chargeId": "uuid",
  "contentType": "image/jpeg",
  "submittedAt": "2026-06-17T19:00:00Z",
  "validationResult": null,
  "message": "Comprovante recebido. Aguardando revisão do administrador."
}
```

> Nesta fase, todo comprovante vai para revisão manual (validationResult permanece null até admin revisar).

---

### 7.5 Aprovar comprovante (admin)

```
POST /api/v1/groups/{groupId}/charges/{chargeId}/payment-attempts/{attemptId}/approve
```

**Autorização:** OWNER, ADMIN  
**Body:**
```json
{
  "reviewNote": "Comprovante válido"
}
```
(`reviewNote` opcional)

**Efeitos em transação única:**
1. `PaymentAttempt.validationResult = APPROVED`, `validationSource = MANUAL`, `reviewedBy = adminId`, `reviewedAt = now()`
2. `Charge.status = APPROVED`, `Charge.updatedAt = now()`
3. `FinancialEntry` criada: `type=REVENUE, category=DAILY, amount=charge.amount, matchId, sourceChargeId, registeredBy=adminId`
4. Presença já está CONFIRMED — nenhuma alteração adicional em BC-03

**Response 200:**
```json
{
  "chargeId": "uuid",
  "status": "APPROVED",
  "approvedAt": "2026-06-17T20:00:00Z"
}
```

---

### 7.6 Rejeitar comprovante (admin)

```
POST /api/v1/groups/{groupId}/charges/{chargeId}/payment-attempts/{attemptId}/reject
```

**Autorização:** OWNER, ADMIN  
**Body:**
```json
{
  "reviewNote": "Valor incorreto no comprovante"
}
```
(`reviewNote` obrigatório)

**Efeitos:**
1. `PaymentAttempt.validationResult = REJECTED`, `validationSource = MANUAL`, `reviewedBy = adminId`, `reviewedAt = now()`
2. `Charge` permanece PENDING — jogador pode enviar novo comprovante

**Response 200:**
```json
{
  "chargeId": "uuid",
  "attemptId": "uuid",
  "status": "REJECTED",
  "reviewNote": "Valor incorreto no comprovante"
}
```

---

### 7.7 Aprovação manual sem comprovante (admin)

```
POST /api/v1/groups/{groupId}/charges/{chargeId}/approve-manually
```

**Autorização:** OWNER, ADMIN  
**Body:**
```json
{
  "note": "Pago em dinheiro no dia da partida"
}
```
(`note` opcional)

**Efeitos em transação única:**
1. `PaymentAttempt` criada com `fileKey=null`, `validationResult=APPROVED`, `validationSource=MANUAL`, `reviewedBy=adminId`, `reviewNote=note`, `reviewedAt=now()`
2. `Charge.status = APPROVED`, `Charge.updatedAt = now()`
3. `FinancialEntry` criada: `type=REVENUE, category=DAILY, amount, matchId, sourceChargeId, registeredBy=adminId`

**Response 200:** mesmo formato do 7.5

---

## 8. Regras de Negócio

| ID | Regra |
|---|---|
| RN-PAG-001 | Charge DAILY só é criada se `group.matchFee != null && group.matchFee > 0` |
| RN-PAG-002 | Se o membro já tem Charge PENDING ou APPROVED para a partida, não cria nova Charge ao confirmar presença novamente |
| RN-PAG-003 | Charge APPROVED não pode ser revertida nesta fase (estorno é Feature 6 — Financial) |
| RN-PAG-004 | Só o membro dono da Charge pode fazer upload de comprovante |
| RN-PAG-005 | Enquanto houver PaymentAttempt com resultado pendente de revisão (null), novo upload retorna 409 |
| RN-PAG-006 | Admin pode aprovar ou rejeitar sem exigir comprovante (`approve-manually`) |
| RN-PAG-007 | Tipos MIME aceitos: `image/jpeg`, `image/png`, `application/pdf`; tamanho máximo: 10 MB |
| RN-PAG-008 | Aprovação de Charge gera FinancialEntry REVENUE DAILY automaticamente na mesma transação |
| RN-PAG-009 | Cancelamento de presença NÃO cancela a Charge (admin decide manualmente sobre estorno) |
| RN-PAG-010 | `pixKey` do grupo é informada no `ChargeDetailResponse` para o jogador saber para onde pagar |

---

## 9. Telas do Frontend

### 9.1 Detalhe da Partida — lista de presença com status de pagamento

**Rota:** `/groups/[groupId]/matches/[matchId]` (tela existente)

**Alterações:**
- Coluna "Pagamento" na lista de presença: badge `Pago` (verde) | `Pendente` (amarelo) | `-` (quando grupo não cobra)
- Se o usuário logado tem cobrança PENDING na partida: card de alerta com botão **"Enviar comprovante"**
- Card exibe: valor da diária, chave Pix do grupo, instruções curtas

### 9.2 Modal: Enviar Comprovante

**Trigger:** botão "Enviar comprovante" no detalhe da partida  
**Layout:**
- Chave Pix + valor destacados (para o jogador copiar antes de enviar)
- Upload de arquivo (drag-and-drop + botão)
- Tipos aceitos: PNG, JPG, PDF; tamanho máximo: 10 MB
- Botão "Enviar comprovante" → chama POST /payment-attempts
- Estado de sucesso: "Comprovante enviado! Aguardando confirmação do administrador."

### 9.3 Admin: Fila de Pagamentos

**Rota:** `/groups/[groupId]/admin/payments`

**Layout:**
- Tabs: **"Aguardando revisão"** | **"Aprovados"** | **"Rejeitados"**
- Cada item da fila exibe: foto do membro, nome, partida, valor, miniatura do comprovante
- Ações por item: **"Aprovar"** | **"Rejeitar"** (ambos abrem modal de confirmação com campo de nota)
- Botão **"Registrar pagamento manual"** (abre modal: busca membro, seleciona partida com cobrança PENDING, campo de nota opcional)
- Link para esta tela no menu admin do grupo

### 9.4 Configurações do Grupo — campo Valor da Diária

**Rota:** `/groups/[groupId]/settings` (tela existente)

**Alteração:** adicionar campo **"Valor da diária (R$)"** com validação `> 0` ou vazio (desabilita cobrança). Campo **"Chave Pix"** já existe.

---

## 10. Fluxos de Sequência

### Fluxo 1 — Jogador confirma presença com cobrança

```
PLAYER → POST /presences
  Backend:
    1. Valida presença (regras BC-03)
    2. Cria PresenceEntry (CONFIRMED)
    3. Se group.matchFee > 0 e sem Charge existente:
         Cria Charge (DAILY, PENDING, amount=matchFee)
    4. Retorna PresenceResponse com pendingCharge

PLAYER → vê card "Pagar R$ 25,00" na tela da partida
```

### Fluxo 2 — Jogador envia comprovante, admin aprova

```
PLAYER → POST /payment-attempts (multipart)
  Backend:
    1. Valida Charge (PENDING, pertence ao memberId)
    2. Faz upload do arquivo para R2
    3. Cria PaymentAttempt (validationResult=null, pendente revisão)
    4. Retorna 201

ADMIN → GET /charges?status=PENDING (vê lista de pendências)
ADMIN → GET /charges/{chargeId} (vê comprovante)
ADMIN → POST /payment-attempts/{attemptId}/approve
  Backend:
    1. Aprova PaymentAttempt
    2. Charge.status = APPROVED
    3. Cria FinancialEntry (REVENUE, DAILY)
    4. Retorna 200
```

### Fluxo 3 — Admin registra pagamento manual (sem comprovante)

```
ADMIN → POST /charges/{chargeId}/approve-manually { note: "Pago em dinheiro" }
  Backend:
    1. Cria PaymentAttempt (fileKey=null, APPROVED, MANUAL)
    2. Charge.status = APPROVED
    3. Cria FinancialEntry (REVENUE, DAILY)
    4. Retorna 200
```

---

## 11. Migrations

| Migration | Arquivo | Ação |
|---|---|---|
| V011 | `create_charges.sql` | Já aplicada — `charges` + `payment_attempts` |
| V012 | `create_financial_entries.sql` | Já aplicada — `financial_entries` |
| V016 | `alter_groups_match_fee_and_payment_attempts.sql` | Adiciona `match_fee` em `groups`; torna `file_key` nullable em `payment_attempts` com check constraint |

---

## 12. Considerações de Implementação

- `SubmitReceiptUseCase` usa `StoragePort` (já implementado para R2) — padrão idêntico ao upload de foto de grupo
- A criação de `Charge` dentro do use case de confirmação de presença deve usar injeção de `ChargeRepository` via port, sem quebrar a Clean Architecture do BC-03
- `ApproveAttemptUseCase` e `ManualApproveChargeUseCase` compartilham a lógica de criação de `FinancialEntry` — extrair para método privado ou domain service
- Nenhuma notificação push nesta fase (pendente BC 4.7); o admin acompanha pela fila `/admin/payments`
