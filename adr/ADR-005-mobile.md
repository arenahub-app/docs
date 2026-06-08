# ADR-005 — Mobile: React Native com Expo

**Status:** Aceito  
**Data:** 2026-06-07  
**Decisores:** Time de Arquitetura

---

## Contexto

A maioria dos jogadores acessa o sistema pelo celular. O app mobile precisa cobrir:
- Confirmação de presença (fluxo principal — mínimo 3 toques)
- Upload de comprovante de pagamento
- Visualização de times formados
- Notificações push

O time de desenvolvimento tem experiência em React e TypeScript, sem experiência nativa em iOS/Android.

## Decisão

**Framework:** React Native com Expo (managed workflow)  
**Linguagem:** TypeScript  
**Navegação:** React Navigation v6  
**Gerenciamento de estado:** TanStack Query (mesmo do frontend web)  
**Notificações push:** Expo Notifications + FCM/APNs

### Por que Expo (Managed Workflow)

- Push to device sem configurar Xcode/Android Studio para build local
- Expo Go para testes rápidos em dispositivos reais
- EAS Build para builds de produção em CI/CD
- Módulos prontos: câmera, filesystem, SecureStorage, notificações

### Reutilização de Código com o Frontend Web

- Tipos TypeScript compartilhados: `shared/types/`
- Funções de validação Zod: `shared/validation/`
- Contratos de API (interfaces): reutilizados diretamente

```
workspace/arenahub/
├── shared/          # Código compartilhado entre web e mobile
│   ├── types/       # Tipos de domínio TypeScript
│   └── validation/  # Schemas Zod
├── frontend/        # React Web
└── mobile/          # React Native / Expo
```

### Escopo do App Mobile (MVP)

| Funcionalidade | Status |
|---|---|
| Login (email/senha + Google) | MVP |
| Dashboard básico (partidas do grupo) | MVP |
| Confirmar/cancelar presença | MVP |
| Upload de comprovante | MVP |
| Visualizar times formados | MVP |
| Perfil do usuário | MVP |
| Notificações push | MVP |
| Votação de habilidades | v1.5 |
| Relatórios financeiros | v2.0 |

## Alternativas Consideradas

### Opção A: Flutter
- **Prós:** Performance nativa; UI consistente em Android e iOS; Dart é tipado
- **Contras:** Time não tem experiência em Dart/Flutter; reaprendizado completo; não reutiliza código com o frontend React

### Opção B: React Native sem Expo (bare workflow)
- **Prós:** Mais controle sobre módulos nativos
- **Contras:** Requer Xcode e Android Studio configurados; CI/CD mais complexo; para o MVP, o managed workflow do Expo cobre todos os casos de uso

### Opção C: PWA (Progressive Web App)
- **Prós:** Sem app store; reutiliza 100% do código frontend
- **Contras:** Notificações push limitadas no iOS (< Safari 16.4); câmera e armazenamento com APIs menos confiáveis; experiência inferior ao nativo

### Opção D: React Native + Expo (escolhida)
- **Prós:** Reutiliza TypeScript, React Query e tipos; Expo simplifica build e deploy; EAS Build para CI/CD
- **Contras:** Expo managed limita módulos nativos customizados (mitigado pela biblioteca de módulos da Expo)

## Consequências

**Positivas:**
- Time pode usar conhecimento React existente
- Expo EAS Build integra com GitHub Actions sem macOS local
- SecureStorage para refresh tokens no mobile (equivalente ao HttpOnly Cookie do web)

**Negativas:**
- Expo managed workflow adiciona ~20MB ao bundle final
- Eventual ejection para bare workflow se precisar de módulos nativos não suportados pelo Expo
