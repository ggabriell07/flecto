# Flecto

Assistente financeiro comportamental operando via WhatsApp. Sistema em produção na v7.

---

## O que faz

Recebe mensagens de texto, áudio ou foto de nota fiscal pelo WhatsApp e executa extração de dados financeiros, análise comportamental e feedback personalizado com base no histórico do usuário.

---

## Arquitetura

```
WhatsApp (usuário)
    ↓
Evolution API  ←  webhook
    ↓
n8n (orquestração)
    ├── Valida acesso          → Supabase (users)
    ├── Detecta intenção       → Claude API
    └── Roteamento de fluxo
        ├── /gasto             → extração + persistência
        ├── /receita           → extração + persistência
        ├── /analise-compra    → 5 perguntas + parecer LLM
        ├── /relatorio         → agregação mensal + score
        ├── /onboarding        → coleta de perfil step-by-step
        └── /ajuda             → resposta estática
    ↓
Evolution API  →  envia resposta ao usuário
```

---

## Processamento multimodal

**Texto**
Linguagem natural → Claude API extrai: valor, categoria, descrição, estado emocional, forma de pagamento, número de parcelas, data (incluindo referências relativas como "ontem" ou "dia 15").

**Áudio**
1. Evolution API entrega o arquivo criptografado
2. Descriptografia via HKDF customizado (Node.js `crypto` nativo)
3. Transcrição via OpenAI Whisper
4. Texto transcrito segue o mesmo fluxo de texto

**Imagem**
1. Evolution API entrega a imagem criptografada
2. Descriptografia via HKDF customizado
3. Leitura via Claude Vision
4. Dados extraídos da nota fiscal seguem fluxo de gasto

> A descriptografia de mídia é implementada manualmente via HKDF porque o endpoint da Evolution API para esse fim não estava disponível na versão em uso (v2.3.7).

---

## Banco de dados (Supabase / PostgreSQL)

| Tabela | Conteúdo |
|---|---|
| `users` | perfil, renda, limite, objetivo, streak, onboarding_step, profile_tags |
| `expenses` | valor, categoria, descrição, expense_date, estado emocional, parcelas, wasPlanned |
| `installments` | parcelas com vencimento mensal |
| `recent_messages` | JSONB com histórico de conversa para contexto do LLM |

O campo `expense_date` permite registro retroativo o usuário menciona uma data passada e o sistema salva corretamente sem depender da data de envio da mensagem.

---

## Módulos funcionais

**Score financeiro**
Calcula 0–100 com base em proporção de gastos planejados vs impulsivos, cumprimento do limite mensal e comprometimento com parcelas.

**Análise "Posso Comprar?"**
Fluxo de 5 perguntas que coleta: o que quer comprar, necessidade imediata, se estava planejado, sentimento sem a compra, e alinhamento com objetivos. Claude retorna parecer com leitura do padrão emocional.

**Gamificação**
Streak diário de registro persistido em `users.streak`. Barra de progresso visual em ASCII. Microfeedback contextual diferenciado por tipo de gasto.

**Relatório mensal**
Saldo real (receitas − despesas), comparativo com mês anterior, insight comportamental cruzando categoria × dia da semana × estado emocional, previsão de estouro de limite.

---

## Configuração n8n

Variáveis de ambiente necessárias:
```
NODE_FUNCTION_ALLOW_BUILTIN=crypto,https,http
N8N_RUNNERS_DISABLED=true
```

Endpoint interno Evolution API: `http://10.11.0.14:8080`

---

## O que não está aqui

O código-fonte dos workflows n8n e das funções de criptografia não está público neste repositório. A documentação acima cobre arquitetura, decisões técnicas e estrutura de dados.

---

**v7 · em produção**
