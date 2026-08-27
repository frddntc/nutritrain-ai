# 🤖 NutriTrain AI Bot

Agente de IA para geração de planos personalizados de dieta e treino (musculazione) via Telegram, utilizando n8n Cloud, Supabase Vector e Gemini API.

## 📋 Visã»£o Geral

Este projeto cria um bot no Telegram que:
- **Coleta dados do usuário** (idade, sexo, peso, altura, objetivos, rotina, limitaç»µes)
- **Busca em banco de conhecimento** (PDFs sobre nutriç»£o e treino) usando RAG (Retrieval Augmented Generation)
- **Gera planos personalizados** de dieta e treino com embasamento científico
- **Estima eficá»¡cia e tempo** para atingir objetivos (cená¡¡rios: 100% e 70% de aderê»¢ncia)

## 🏗️ Arquitetura

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────┐
│   Telegram Bot  │────▶│   n8n Cloud  │────▶│  Gemini API │
└─────────────────┘     └──────────────┘     └─────────────┘
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
| **n8n Cloud** | Orquestraç»£o de workflows | Free (5.000 execuç»µes/mê»ªs) |
| **Telegram Bot API** | Interface com usuário | Gratuito |
| **Supabase Vector** | Banco de dados vetorial (PDFs) | Free (500 MB) |
| **Gemini 2.5 Flash** | LLM para geraç»£o de planos | Free (250 req/dia) |
| **Google Sheets** | Armazenar dados dos usuários | Gratuito |

## 📁 Estrutura do Projeto

```
nutritrain-ai-bot/
├── README.md              # Este arquivo
├── SETUP.md               # Guia de configuraç»£o passo a passo
├── workflows/
│   ├── 01-pdf-ingestion.json    # Workflow de ingestã»£o de PDFs
│   ├── 02-telegram-bot.json     # Workflow principal do bot
│   └── 03-plan-generator.json   # Workflow de geraç»£o de planos
├── docs/
│   ├── prompt-templates.md      # Prompts para o Gemini
│   └── database-schema.md       # Schema do Supabase
└── pdfs/
    └── (seus arquivos PDF aqui)
```

## ✨ Funcionalidades

### 1. Coleta de Dados do Usuá¡¡rio
- Idade, sexo, peso, altura
- Objetivo (emagrecer, ganhar massa, manter)
- Ní»¡vel de atividade física
- Rotina diá¡¡ria (horá¡¡rios de trabalho, sono, refeiç»µes)
- Restriç»µes alimentares e preferê»¢ncias
- Histó¡¡¡rico de treinos e limitaç»µes físicas

### 2. Geraç»£o de Plano de Dieta
- Cálculo de TMB e GET
- Distribuiç»£o de macronutrientes (proteí»¡nas, carboidratos, gorduras)
- Exemplos de refeiç»µes (cafá©© da manhã»£, almoç»£o, jantar, lanches)
- Ajustes para restriç»µes (vegetariano, sem glá¡¡ten, etc.)

### 3. Geraç»£o de Plano de Treino
- Divisã»£o de treinos (ABCD, full body, upper/lower)
- Exerccios com séries, repetiç»µes e cargas sugeridas
- Progressã»£o de carga ao longo das semanas
- Adaptaç»µes para limitaç»µes físicas

### 4. Estimativas de Resultados
- **Cená¡¡rio 100%**: Tempo estimado com aderê»¢ncia total
- **Cená¡¡rio 70%**: Tempo estimado com aderê»¢ncia parcial
- Eficá¡¡cia de cada método (ní»¡vel de evidê»¢ncia)

## 🚀 Começ»¡ando

1. **Clone ou baixe este repositó¡¡¡rio**
2. **Siga o guia em `SETUP.md`** para configurar:
   - Supabase (banco vetorial)
   - n8n Cloud (workflows)
   - Telegram Bot (token)
   - Gemini API (chave)
3. **Importe os workflows** do diretó¡¡¡rio `workflows/`
4. **Faç»¡a upload dos PDFs** para o Supabase
5. **Teste o bot** no Telegram com `/start`

## 📚 Documentaç»£o Adicional

- **`SETUP.md`**: Guia completo de configuraç»£o
- **`docs/prompt-templates.md`**: Prompts usados no Gemini
- **`docs/database-schema.md`**: Schema do banco de dados

## ⚠️ Limitaç»µes do Free Tier

| Serviço | Limite | Implicaç»µes |
|---------|--------|--------------|
| n8n Cloud | 5.000 execuç»µes/mê»ªs | ~166 execuç»µes/dia (suficiente para ~5-10 usuários/dia) |
| Supabase Vector | 500 MB | ~50-100 PDFs (depende do tamanho) |
| Gemini API | 250 requests/dia | ~8 usuários completos/dia (30 requests cada) |
| Telegram Bot | Sem limites | Ilimitado |

## 📝 Licenç»¡a

MIT License - Use livremente para projetos pessoais ou comerciais.

## 🤝 Contribuiç»µes

Sugest »Åµes e melhorias sã»£o bem-vindas! Abra uma issue ou pull request.

---

**Desenvolvido com ❤️ usando n8n, Supabase e Gemini AI**
