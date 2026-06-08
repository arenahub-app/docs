# ADR-004 — Frontend Web: React + Vite + Material UI

**Status:** Aceito  
**Data:** 2026-06-07  
**Decisores:** Time de Arquitetura

---

## Contexto

Precisamos de uma aplicação web que:
- Seja rápida para desenvolver no contexto de MVP
- Tenha boa experiência de uso em desktop e mobile (responsiva)
- Consuma APIs REST do backend
- Suporte formulários complexos com validação
- Seja tipada para reduzir bugs

## Decisão

**Framework:** React 18 com TypeScript  
**Build tool:** Vite  
**Gerenciamento de estado de servidor:** TanStack Query (React Query v5)  
**Formulários:** React Hook Form + Zod  
**UI:** Material UI v6 (MUI)  
**Roteamento:** React Router v6

### Estrutura de Diretórios

```
frontend/src/
├── api/          # Funções de chamada à API (axios instances, hooks de query)
├── components/   # Componentes reutilizáveis (design system)
├── features/     # Funcionalidades organizadas por domínio
│   ├── auth/
│   ├── groups/
│   ├── matches/
│   ├── payments/
│   └── ...
├── hooks/        # Custom hooks genéricos
├── pages/        # Páginas (composição de features)
├── router/       # Configuração do React Router
├── theme/        # Configuração do MUI theme
└── types/        # Tipos TypeScript globais
```

### Padrão de Data Fetching

```typescript
// Toda chamada de servidor via TanStack Query
const { data: group } = useQuery({
  queryKey: ['groups', groupId],
  queryFn: () => groupsApi.getById(groupId),
});

// Mutations com invalidação de cache
const { mutate: confirmPresence } = useMutation({
  mutationFn: matchesApi.confirmPresence,
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['matches'] }),
});
```

## Alternativas Consideradas

### Opção A: Next.js (SSR/SSG)
- **Prós:** SEO; performance de carregamento inicial; fullstack opcional
- **Contras:** Overkill para painel admin/app que requer autenticação; complexidade de deploy; ArenaHub é uma SPA protegida por login — SEO não é prioridade

### Opção B: React + Vite (escolhida)
- **Prós:** Setup simples; dev server rápido; sem complexidade de SSR para MVP; time familiar com React SPA
- **Contras:** Sem SSR; carregamento inicial ligeiramente mais lento que Next.js

### Opção C: Tailwind CSS (em vez de MUI)
- **Prós:** Mais controle visual; bundle menor
- **Contras:** MUI entrega componentes ricos prontos (Table, DatePicker, etc.) que acelerariam o MVP; Tailwind requer mais tempo de design

## Consequências

**Positivas:**
- Vite: hot reload em < 100ms no desenvolvimento
- MUI: componentes de formulário, tabela e modal prontos para uso
- React Query: cache de server state, loading/error states automáticos, refetch on focus

**Negativas:**
- MUI adiciona ~200KB ao bundle (mitigado por tree-shaking)
- TypeScript adiciona overhead de tipagem mas reduz bugs em produção
