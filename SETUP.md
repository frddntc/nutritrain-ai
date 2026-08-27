# 🔧 SETUP - Guia de Configuraç»£o

Este guia cobre **passo a passo** a configuraç»£o de todos os serviços necessários para o NutriTrain AI Bot.

## 📋 Pré-requisitos

- Conta no **n8n Cloud** (free tier): https://n8n.io
- Conta no **Supabase**: https://supabase.com
- Conta no **Google AI Studio** (Gemini API): https://aistudio.google.com
- Conta no **Telegram** (app instalado)

Tempo estimado: **30-45 minutos**

---

## 1️⃣ Configurar Supabase (Banco Vetorial)

### Passo 1.1: Criar Projeto

1. Acesse https://supabase.com e faça login
2. Clique em **"New Project"**
3. Preencha:
   - **Name**: `nutritrain-db`
   - **Database Password**: (salve em local seguro)
   - **Region**: Escolha a mais próxima (ex: `us-east-1`)
4. Clique em **"Create new project"** (aguarde 2-3 min)

### Passo 1.2: Habilitar Extensã»£o pgvector

1. No dashboard do projeto, vá para **SQL Editor** (menu lateral)
2. Clique em **"New Query"**
3. Cole e execute:

```sql
-- Habilitar extensão pgvector
create extension if not exists vector;
```

### Passo 1.3: Criar Tabela de Embeddings

1. No **SQL Editor**, cole e execute:

```sql
-- Tabela para armazenar embeddings dos PDFs
create table pdf_embeddings (
  id bigserial primary key,
  content text not null,
  embedding vector(768),  -- Gemini usa 768 dimensõ»µµes
  metadata jsonb default '{}'::jsonb,
  created_at timestamp with time zone default timezone('utc'::text, now())
);

-- Índices para busca vetorial
create index on pdf_embeddings using ivfflat (embedding vector_cosine_ops) with (lists = 100);

-- Habilitar Row Level Security (RLS)
alter table pdf_embeddings enable row level security;

-- Política para permitir leitura/escrita pública (para teste)
create policy "Permitir acesso público" on pdf_embeddings
  for all using (true) with check (true);
```

### Passo 1.4: Obter Credenciais

1. Vá»¡ para **Settings** (í»¡cone de engrenagem no menu lateral)
2. Clique em **API**
3. Anote:
   - **Project URL**: `https://xxxxx.supabase.co`
   - **API Key (public/anon)**: `eyJhbG...` (chave longa)

> ⚠️ **Importante**: Use a chave **public/anon**, nã»£o a `service_role` (esta é administrativa).

---

## 2️⃣ Configurar Gemini API

### Passo 2.1: Criar Chave de API

1. Acesse https://aistudio.google.com/app/apikey
2. Faça login com sua conta Google
3. Clique em **"Create API Key"**
4. Clique em **"Create API key in new project"**
5. Copie a chave (formato: `AIzaSy...`)

### Passo 2.2: Verificar Limites

1. Acesse https://aistudio.google.com/app/quota
2. Confirme que está no **Free Tier**
3. Limites atuais (agosto 2026):
   - **Gemini 2.5 Flash**: 250 requests/dia
   - **Gemini 2.5 Flash-Lite**: 1.000 requests/dia

---

## 3️⃣ Configurar Telegram Bot

### Passo 3.1: Criar Bot

1. Abra o Telegram e busque por **@BotFather**
2. Envie o comando `/newbot`
3. BotFather pedir á um nome: digite `NutriTrain Bot`
4. BotFather pedir á um username: digite `NutriTrainAI_bot` (deve terminar em `bot`)
5. BotFather responder á com uma mensagem como:
   ```
   Done! Congratulations on your new bot. You can find it at t.me/NutriTrainAI_bot

   You can use this token to access the HTTP API:
   1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
   ```
6. **Copie o token** (a linha após "token:")

### Passo 3.2: Configurar Comandos

1. No @BotFather, envie `/setcommands`
2. Selecione seu bot (`@NutriTrainAI_bot`)
3. Envie a lista de comandos:
   ```
   start - Iniciar o bot e coletar dados
   plano - Gerar plano de dieta e treino
   dados - Ver/editar meus dados
   ajuda - Ajuda e informações
   ```

### Passo 3.3: Testar o Bot

1. No Telegram, busque por `@NutriTrainAI_bot`
2. Clique em **Iniciar** ou digite `/start`
3. O bot deve responder (poré©©m ainda nã»£o far á nada at é configurar o n8n)

---

## 4️⃣ Configurar n8n Cloud

### Passo 4.1: Criar Conta

1. Acesse https://n8n.io
2. Clique em **"Get started for free"**
3. Escolha **"n8n Cloud - Free"** (5.000 execuç»µes/mê»ªs)
4. Preencha email e senha
5. Confirme o email

### Passo 4.2: Criar Credenciais

No n8n, vá para **Credentials** (menu lateral) e crie:

#### 4.2.1: Supabase
- **Name**: `Supabase nutritrain`
- **Connection Type**: `Connect with API Key`
- **URL**: Cole a **Project URL** do Supabase
- **API Key**: Cole a **API Key (public/anon)** do Supabase

#### 4.2.2: Google Gemini
- **Name**: `Gemini API`
- **Credential Type**: `Google Gemini API`
- **API Key**: Cole a chave do Gemini API

#### 4.2.3: Telegram Bot
- **Name**: `Telegram NutriTrain`
- **Access Token**: Cole o token do BotFather

#### 4.2.4: Google Sheets (Opcional)
- **Name**: `Google Sheets`
- **Credential Type**: `Google Service Account`
- Siga o fluxo de OAuth do Google

---

## 5️⃣ Importar Workflows

### Passo 5.1: Workflow 01 - Ingestã»£o de PDFs

1. No n8n, vá para **Workflows** → **Add workflow**
2. Clique nos **trê»ªs pontos** no canto superior direito → **Import from File**
3. Selecione o arquivo `workflows/01-pdf-ingestion.json`
4. **Configure os nós**:

#### Nó: `Supabase Vector Store`
- **Credential**: Selecione `Supabase nutritrain`
- **Table**: `pdf_embeddings`
- **Embedding**: `Gemini Embedding` (sub-nóººº)
  - **Model**: `text-embedding-004`

#### Nó: `Extract from File`
- Deixe como está (extrai texto de PDF)

5. **Salve** o workflow (Ctrl+S)
6. **Ative** o workflow (toggle no topo)

### Passo 5.2: Workflow 02 - Bot Telegram

1. **Add workflow** → **Import from File**
2. Selecione `workflows/02-telegram-bot.json`
3. **Configure os nós**:

#### Nó: `Telegram Trigger`
- **Credential**: Selecione `Telegram NutriTrain`
- **Updates**: `message`
- **Chat ID**: Deixe em branco (para escutar todos)

#### Nó: `Supabase Vector Store` (RAG Search)
- **Credential**: `Supabase nutritrain`
- **Table**: `pdf_embeddings`
- **Operation**: `Search`
- **Query**: `{{ $json.message }}` (ou variá¡¡vel apropriada)
- **Top K**: `5`

#### Nó: `Google Gemini` (LLM)
- **Credential**: `Gemini API`
- **Model**: `gemini-2.5-flash`
- **Prompt**: Use o template em `docs/prompt-templates.md`

5. **Salve** e **Ative**

### Passo 5.3: Workflow 03 - Gerador de Planos

1. **Add workflow** → **Import from File**
2. Selecione `workflows/03-plan-generator.json`
3. **Configure os nós**:

#### Nó: `Google Sheets` (Ler dados do usuário)
- **Credential**: `Google Sheets`
- **Operation**: `Read`
- **Spreadsheet ID**: (crie uma planilha com os dados)
- **Range**: `Sheet1!A:Z`

#### Nó: `Google Gemini` (Gerar plano)
- **Credential**: `Gemini API`
- **Model**: `gemini-2.5-flash`
- **Prompt**: Template especí»¡fico para planos (ver `docs/prompt-templates.md`)

5. **Salve** e **Ative**

---

## 6️⃣ Fazer Upload dos PDFs

### Opç»£o A: Via Workflow 01

1. No workflow `01-pdf-ingestion`, adicione um nó **Manual Trigger**
2. Adicione um nó **Read Binary File** (se os PDFs estiverem no seu computador)
3. Ou use **Google Drive Trigger** (se os PDFs estiverem no Drive)
4. Execute o workflow manualmente

### Opç»£o B: Via SQL Direto (Avanç»¡ado)

1. Extraia o texto dos PDFs manualmente (use https://smallpdf.com/pt/pdf-para-texto)
2. No **SQL Editor** do Supabase, insira:

```sql
insert into pdf_embeddings (content, metadata)
values (
  'Texto extraí»¡do do PDF...',
  '{"fonte": "nutricao_esportiva.pdf", "pagina": 1}'::jsonb
);
```

---

## 7️⃣ Testar o Bot

1. No Telegram, abra `@NutriTrainAI_bot`
2. Envie `/start`
3. O bot deve responder pedindo seus dados
4. Responda as perguntas
5. Envie `/plano`
6. O bot deve gerar um plano baseado nos seus dados + PDFs

---

## 🐛 Troubleshooting

### Erro: "Failed to connect to Supabase"
- Verifique se a **Project URL** e **API Key** estão corretas
- Confirme se a tabela `pdf_embeddings` existe (execute `select * from pdf_embeddings limit 1;`)

### Erro: "Gemini API rate limit exceeded"
- Você excedeu 250 requests/dia
- Aguarde 24h ou use `gemini-2.5-flash-lite` (1.000 requests/dia)

### Erro: "Telegram Bot: Unauthorized"
- O token do bot está errado
- Regere o token no @BotFather com `/revoke` e atualize no n8n

### Workflow nã»£o executa
- Verifique se o workflow está **Ativo** (toggle verde no topo)
- Confira os **logs de execução** (clique no workflow → Execution List)

---

## 📞 Suporte

- **n8n Community**: https://community.n8n.io
- **Supabase Discord**: https://discord.supabase.com
- **Gemini API Docs**: https://ai.google.dev/gemini-api/docs

---

**Pronto! Agora você»ª tem um agente de IA funcional para dieta e treino.** 🎉
