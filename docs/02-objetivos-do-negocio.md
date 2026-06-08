# Objetivos do Negócio — ArenaHub

## OKRs de Produto (Horizonte 12 meses)

### Objetivo 1 — Validar o Produto no Mercado

| Key Result | Meta |
|---|---|
| KR1.1 | 50 grupos ativos ao final do 6º mês |
| KR1.2 | NPS médio ≥ 40 entre organizadores |
| KR1.3 | Retenção de grupos após 90 dias ≥ 70% |

### Objetivo 2 — Garantir Receita Recorrente

| Key Result | Meta |
|---|---|
| KR2.1 | 20% dos grupos no plano pago ao final do 12º mês |
| KR2.2 | Churn mensal de grupos pagantes ≤ 3% |
| KR2.3 | MRR de R$ 5.000 ao final do 12º mês |

### Objetivo 3 — Estabelecer Qualidade Técnica

| Key Result | Meta |
|---|---|
| KR3.1 | Cobertura de testes ≥ 80% em todos os módulos |
| KR3.2 | Disponibilidade do sistema ≥ 99,5% |
| KR3.3 | Tempo de resposta das APIs P95 ≤ 500ms |

## Métricas de Sucesso

### Ativação
- Taxa de grupos que completam o cadastro e criam a primeira partida dentro de 7 dias

### Engajamento
- Média de confirmações de presença por partida criada
- Taxa de uso do módulo de pagamentos por grupos ativos
- Taxa de uso da formação automática de times

### Retenção
- Grupos ativos 30/60/90 dias após cadastro
- Frequência média de partidas criadas por grupo ativo

### Receita
- MRR (Monthly Recurring Revenue)
- ARPU (Average Revenue Per User — por grupo)
- LTV estimado por cohort

## Restrições de Negócio

| Restrição | Descrição |
|---|---|
| Orçamento inicial | MVP com time reduzido; deploy em Railway para minimizar custo de infraestrutura |
| Conformidade | LGPD — dados de jogadores são dados pessoais; exige consentimento e política de privacidade |
| Pagamentos | Não processamos pagamentos diretamente; validamos comprovantes de transferência Pix |
| Disponibilidade regional | Brasil; idioma pt-BR obrigatório |

## Premissas

1. Organizadores têm acesso à internet via smartphone
2. Grupos já utilizam Pix como meio de pagamento predominante
3. A maioria dos jogadores não pagará por acesso individual — o valor deve ser percebido pelo organizador
4. Grupos tipicamente têm entre 15 e 40 jogadores cadastrados

## Riscos de Negócio

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Baixa adesão ao app mobile pelos jogadores | Alta | Alto | Manter Web funcionando; notificações por link direto |
| Grupos não migrarem de WhatsApp | Média | Alto | Integração futura com WhatsApp Business API |
| Inadimplência de planos pagos | Média | Médio | Downgrade automático com aviso prévio |
| Validação de IA com alta taxa de erro | Baixa | Alto | Fallback para revisão manual; tuning incremental |
