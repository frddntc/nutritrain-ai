# 🤖 NutriTrain AI Bot

Agente de IA para geração de planos personalizados de dieta e treino via Telegram, utilizando n8n Cloud, Supabase Vector e Gemini API.

## 📋 Visão Geral

Este projeto cria um bot no Telegram que:
- **Coleta dados do usuário** (idade, sexo, peso, altura, objetivos, rotina, limitações)
- **Busca em banco de conhecimento** (PDFs sobre nutriç»£o e treino) usando RAG (Retrieval Augmented Generation)
- **Gera planos personalizados** de dieta e treino com embasamento científico
- **Estima eficácia e tempo** para atingir objetivos (cenários: 100% e 70% de aderência)

## 🏗️ Arquitetura

```
┌─────────────────┐      ┌──────────────┐      ┌─────────────┐
│   Telegram Bot  │────▶│   n8n Cloud   │────▶│  Gemini API │
└─────────────────┘      └──────────────┘      └─────────────┘
                               │
                               ▼
                        ┌─────────────┐
                        │   Supabase  │
                        │   Vector    │
                        └─────────────┘
                               │
                               ▼
                        ┌─────────────┐
                        │  Google     │
                        │  Sheets     │
                        └─────────────┘
```

## 🛠️ Tecnologias Utilizadas

| Ferramenta | Finalidade | Custo |
|------------|------------|-------|
| **n8n Cloud** | Orquestração de workflows | Free (5.000 execuções/mês) |
| **Telegram Bot API** | Interface com usuário | Gratuito |
| **Supabase Vector** | Banco de dados vetorial (PDFs) | Free (500 MB) |
| **Gemini 2.5 Flash** | LLM para geração de planos | Free (250 req/dia) |
| **Google Sheets** | Armazenar dados dos usuários | Gratuito |

## 📁 Estrutura do Projeto

```
nutritrain-ai-bot/
├── README.md              # Este arquivo
├── SETUP.md               # Guia de configuraç»£o passo a passo
├── workflows/
│   ├── 01-pdf-ingestion.json    # Workflow de ingestão de PDFs
│   ├── 02-telegram-bot.json     # Workflow principal do bot
│   └── 03-plan-generator.json   # Workflow de geração de planos
├── docs/
│   ├── prompt-templates.md      # Prompts para o Gemini
│   └── database-schema.md       # Schema do Supabase
└── pdfs/
    └── (seus arquivos PDF aqui)
```

## ✨ Funcionalidades

### 1. Coleta de Dados do Usuário
- Idade, sexo, peso, altura
- Objetivo (emagrecer, ganhar massa, manter)
- Nível de atividade física
- Rotina diária (horários de trabalho, sono, refeições)
- Restrições alimentares e preferências
- Histórico de treinos e limitações físicas

### 2. Geração de Plano de Dieta
- Cálculo de TMB e GET
- Distribuição de macronutrientes (proteínas, carboidratos, gorduras)
- Exemplos de refeições (café da manhã, almoço, jantar, lanches)
- Ajustes para restrições (vegetariano, sem glúten, etc.)

### 3. Geração de Plano de Treino
- Divisão de treinos (ABCD, full body, upper/lower)
- Exercícios com séries, repetições e cargas sugeridas
- Progressão de carga ao longo das semanas
- Adaptações para limitações físicas

### 4. Estimativas de Resultados
- **Cenário 100%**: Tempo estimado com aderência total
- **Cenário 70%**: Tempo estimado com aderência parcial
- Eficácia de cada método (nível de evidência)

## 🚀 Começando

1. **Clone ou baixe este repositório**
2. **Siga o guia em `SETUP.md`** para configurar:
   - Supabase (banco vetorial)
   - n8n Cloud (workflows)
   - Telegram Bot (token)
   - Gemini API (chave)
3. **Importe os workflows** do diretório `workflows/`
4. **Faça upload dos PDFs** para o Supabase
5. **Teste o bot** no Telegram com `/start`

## 📚 Documentaç»£o Adicional

- **`SETUP.md`**: Guia completo de configuraç»£o
- **`docs/prompt-templates.md`**: Prompts usados no Gemini
- **`docs/database-schema.md`**: Schema do banco de dados

## ⚠️ Limitações do Free Tier

| Serviço | Limite | Implicações |
|---------|--------|--------------|
| n8n Cloud | 5.000 execuções/mês | ~166 execuções/dia (suficiente para ~5-10 usuários/dia) |
| Supabase Vector | 500 MB | ~50-100 PDFs (depende do tamanho) |
| Gemini API | 250 requests/dia | ~8 usuários completos/dia (30 requests cada) |
| Telegram Bot | Sem limites | Ilimitado |

## 📝 Licença

MIT License - Use livremente para projetos pessoais ou comerciais.

## 🤝 Contribuições

Sugestões e melhorias são bem-vindas! Abra uma issue ou pull request.
