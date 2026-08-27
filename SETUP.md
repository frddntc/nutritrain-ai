# 🔧 SETUP — Guia de Configuração (revisado)

> **📝 Sobre esta revisão:** parti do setup gerado pelo Comet e fiz três tipos de ajuste: corrigi uma falha de segurança real (a política do banco liberava leitura/escrita para qualquer pessoa com a chave pública), atualizei informações que mudaram nos últimos meses (modelos do Gemini, plano do n8n) e adicionei uma etapa que faltava (sem ela, o bot não consegue buscar nada nos PDFs). Também corrigi a acentuação, que veio corrompida no arquivo original. O porquê de cada mudança está explicado, passo a passo, no arquivo `EXPLICACAO-SETUP.md`.

Este guia cobre **passo a passo** a configuração de todos os serviços necessários para o NutriTrain AI Bot.

## 📋 Pré-requisitos

- Conta no **n8n** (cloud ou self-hosted — veja a nota no passo 4)
- Conta no **Supabase**: https://supabase.com
- Conta no **Google AI Studio** (Gemini API): https://aistudio.google.com
- Conta no **Telegram** (app instalado)

Tempo estimado: **45-60 minutos** (um pouco mais que o original, porque adicionamos uma etapa de banco de dados que faltava)

---

## 1️⃣ Configurar Supabase (Banco Vetorial)

### Passo 1.1: Criar Projeto

1. Acesse https://supabase.com e faça login
2. Clique em **"New Project"**
3. Preencha:
   - **Name**: `nutritrain-db`
   - **Database Password**: salve em um gerenciador de senhas
   - **Region**: a mais próxima de você (ex: `us-east-1` ou `sa-east-1`, se disponível)
4. Clique em **"Create new project"** (aguarde 2-3 min)

### Passo 1.2: Habilitar Extensão pgvector

1. No dashboard do projeto, vá para **SQL Editor** (menu lateral)
2. Clique em **"New Query"**
3. Cole e execute:

```sql
create extension if not exists vector;
```

### Passo 1.3: Criar Tabela de Embeddings

Cole e execute no **SQL Editor**:

```sql
-- Tabela para armazenar os pedaços de texto dos PDFs e seus embeddings
create table pdf_embeddings (
  id bigserial primary key,
  content text not null,
  embedding vector(768), -- 768 = dimensão que vamos pedir ao gemini-embedding-001
  metadata jsonb default '{}'::jsonb,
  created_at timestamptz default now()
);

-- Índice HNSW para busca por similaridade
create index on pdf_embeddings
  using hnsw (embedding vector_cosine_ops);

-- RLS ligado — e SEM política pública.
-- Com RLS ativo e nenhuma política, ninguém acessa a tabela usando a
-- chave pública (anon). O n8n vai acessar por outro caminho (passo 1.5).
alter table pdf_embeddings enable row level security;
```

### Passo 1.4: Criar a Função de Busca Vetorial

Este passo **não existia no setup original** e é essencial: é o que o node "Supabase Vector Store" do n8n chama por baixo dos panos quando você pede para ele buscar (modo "Search"). Sem essa função, o node de busca não encontra nada — mesmo com os embeddings já salvos na tabela.

No **SQL Editor**, cole e execute:

```sql
create or replace function match_pdf_embeddings (
  query_embedding vector(768),
  match_count int default 5
)
returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
begin
  return query
  select
    pdf_embeddings.id,
    pdf_embeddings.content,
    pdf_embeddings.metadata,
    1 - (pdf_embeddings.embedding <=> query_embedding) as similarity
  from pdf_embeddings
  order by pdf_embeddings.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

Anote o nome `match_pdf_embeddings` — você vai usá-lo no passo 5.2.

### Passo 1.5: Obter Credenciais

1. Vá para **Settings** (ícone de engrenagem) → **API**
2. Anote:
   - **Project URL**: `https://xxxxx.supabase.co`
   - **service_role key** (não a `anon`/`public`)

> ⚠️ **Isso é diferente do setup original, de propósito.** O guia do Comet mandava usar a chave `anon` e liberar a tabela para leitura/escrita pública via política de RLS. Isso funciona, mas deixa o banco aberto para qualquer pessoa que tiver essa chave — e chaves `anon` vazam com facilidade (prints de tela, repositórios públicos etc.). Como quem acessa esse banco é só o n8n (um backend que só você controla, não os usuários do Telegram), o caminho mais seguro é manter o RLS ligado sem política pública e usar a **service_role key** apenas dentro da credencial do n8n. Essa chave ignora RLS por design — trate-a como uma senha de administrador, nunca a use em nada que rode no navegador do usuário. Mais detalhes no arquivo de explicação.

---

## 2️⃣ Configurar Gemini API

### Passo 2.1: Criar Chave de API

1. Acesse https://aistudio.google.com/app/apikey
2. Faça login com sua conta Google
3. Clique em **"Create API Key"** → **"Create API key in new project"**
4. Copie a chave (formato: `AIzaSy...`)

### Passo 2.2: Confirmar Modelos e Limites Atuais

Os nomes de modelo e os limites do plano gratuito do Gemini mudam com frequência — o Google já descontinuou modelos e cortou cotas do free tier mais de uma vez nos últimos meses. Por isso, em vez de anotar aqui um número que já pode estar errado quando você ler:

1. Acesse https://ai.google.dev/gemini-api/docs/models e confirme qual é o modelo "flash" (rápido e barato) recomendado no momento. **Evite os modelos da família Gemini 2.5** — o Google já avisou que serão desligados em outubro de 2026, então começar um projeto novo hoje com `gemini-2.5-flash` significa migrar em poucas semanas. Em agosto de 2026, quando este guia foi escrito, o modelo recomendado era `gemini-3.7-flash`.
2. Acesse https://ai.google.dev/gemini-api/docs/rate-limits para ver os limites atuais do free tier (requisições por minuto e por dia).

---

## 3️⃣ Configurar Telegram Bot

### Passo 3.1: Criar Bot

1. Abra o Telegram e busque por **@BotFather**
2. Envie `/newbot`
3. Escolha um nome: `NutriTrain Bot`
4. Escolha um username terminado em `bot`: `NutriTrainAI_bot`
5. O BotFather vai responder com algo como:
   ```
   Done! Congratulations on your new bot. You can find it at t.me/NutriTrainAI_bot

   You can use this token to access the HTTP API:
   1234567890:ABCdefGHIjklMNOpqrsTUVwxyz
   ```
6. **Copie o token**

> ⚠️ Esse token dá controle total do bot. Não cole em conversas, prints ou repositórios públicos. Se desconfiar que vazou, gere um novo com `/revoke` no BotFather.

### Passo 3.2: Configurar Comandos

1. No @BotFather, envie `/setcommands`
2. Selecione seu bot
3. Envie:
   ```
   start - Iniciar o bot e coletar dados
   plano - Gerar plano de dieta e treino
   dados - Ver/editar meus dados
   ajuda - Ajuda e informações
   ```

### Passo 3.3: Testar o Bot

1. Busque `@NutriTrainAI_bot` no Telegram
2. Envie `/start`
3. Por enquanto o bot não vai responder de verdade — isso só acontece depois de configurar o n8n

---

## 4️⃣ Configurar n8n

> ⚠️ **O plano gratuito "para sempre" do n8n Cloud não existe mais.** Hoje o n8n Cloud oferece um **teste grátis de 14 dias**; depois disso, o plano mais barato (Starter) custa a partir de ~€20/mês para 2.500 execuções. Isso é diferente do que o setup original dizia (5.000 execuções/mês grátis). Duas opções:
>
> - **Opção A — mais simples:** use o teste grátis para montar e testar tudo (é o suficiente para este projeto). Depois de 14 dias, decida se vale pagar.
> - **Opção B — grátis para sempre, mais trabalho:** hospede o n8n você mesmo (self-hosted), de graça, em um servidor/VPS próprio via Docker. Exige um pouco de linha de comando e manter o servidor no ar. Se quiser seguir esse caminho, é só pedir que eu monto um guia à parte — os passos abaixo (workflows, credenciais) são os mesmos nos dois casos.

### Passo 4.1: Criar Conta

1. Acesse https://n8n.io
2. Clique em **"Get started for free"**
3. Preencha email e senha, confirme o email
4. Você começa automaticamente no teste grátis de 14 dias

### Passo 4.2: Criar Credenciais

No n8n, vá para **Credentials** e crie:

#### 4.2.1: Supabase
- **Name**: `Supabase nutritrain`
- **Connection Type**: `Connect with API Key`
- **URL**: a **Project URL** do Supabase
- **API Key**: a **service_role key** (não a `anon`) — veja o aviso no passo 1.5

#### 4.2.2: Google Gemini
- **Name**: `Gemini API`
- **Credential Type**: `Google Gemini API`
- **API Key**: a chave criada no passo 2.1

#### 4.2.3: Telegram Bot
- **Name**: `Telegram NutriTrain`
- **Access Token**: o token do BotFather

#### 4.2.4: Google Sheets (Opcional)
- **Name**: `Google Sheets`
- **Credential Type**: `Google Service Account`
- Siga o fluxo de OAuth do Google

> 💡 O n8n criptografa as credenciais salvas e não as inclui quando você exporta um workflow como JSON — mesmo assim, nunca cole chaves de API diretamente dentro de um node; use sempre a seção de Credentials.

---

## 5️⃣ Importar Workflows

### Passo 5.1: Workflow 01 - Ingestão de PDFs

1. **Workflows** → **Add workflow** → menu **⋮** → **Import from File**
2. Selecione `workflows/01-pdf-ingestion.json`
3. Configure os nós:

#### Nó: `Supabase Vector Store`
- **Credential**: `Supabase nutritrain`
- **Table**: `pdf_embeddings`
- **Operation**: `Insert`
- **Embedding**: `Gemini Embedding` (sub-nó)
  - **Model**: `gemini-embedding-001` (confirme o nome atual em ai.google.dev/gemini-api/docs/models)
  - **Output dimensionality**: `768` — precisa bater com a coluna `vector(768)` da tabela. Sem isso, o `gemini-embedding-001` devolve 3072 dimensões por padrão, o que não bate com a tabela e também não pode ser indexado pelo pgvector (o limite de índice é 2.000 dimensões).

#### Nó: `Extract from File`
- Deixe como está (extrai texto de PDF)

4. **Salve** (Ctrl+S) e **Ative** o workflow

### Passo 5.2: Workflow 02 - Bot Telegram

1. **Add workflow** → **Import from File** → `workflows/02-telegram-bot.json`
2. Configure os nós:

#### Nó: `Telegram Trigger`
- **Credential**: `Telegram NutriTrain`
- **Updates**: `message`
- **Chat ID**: deixe em branco (escuta todas as conversas do bot)

#### Nó: `Supabase Vector Store` (RAG Search)
- **Credential**: `Supabase nutritrain`
- **Table**: `pdf_embeddings`
- **Operation**: `Search`
- **Query Name**: `match_pdf_embeddings` ⚠️ **campo que faltava no setup original** — é o nome da função SQL criada no passo 1.4. Sem preencher isso, a busca falha ou dá erro de função inexistente.
- **Query**: `{{ $json.message.text }}`
- **Top K**: `5`

#### Nó: `Google Gemini` (LLM)
- **Credential**: `Gemini API`
- **Model**: o modelo "flash" atual (veja passo 2.2)
- **Prompt**: use o template em `docs/prompt-templates.md`. Vale incluir uma frase deixando claro que as sugestões não substituem acompanhamento de nutricionista/educador físico — além de mais responsável, isso te protege caso alguém siga uma recomendação sem checar com um profissional.

3. **Salve** e **Ative**

### Passo 5.3: Workflow 03 - Gerador de Planos

1. **Add workflow** → **Import from File** → `workflows/03-plan-generator.json`
2. Configure os nós:

#### Nó: `Google Sheets` (ler dados do usuário)
- **Credential**: `Google Sheets`
- **Operation**: `Read`
- **Spreadsheet ID**: crie uma planilha com os dados dos usuários
- **Range**: `Sheet1!A:Z`

> 💡 Essa planilha vai guardar dados de saúde (peso, altura, eventualmente condições médicas) — no Brasil isso conta como **dado pessoal sensível pela LGPD**. Vale, no mínimo: restringir quem tem acesso à planilha (só a service account, não "qualquer pessoa com o link"), avisar os usuários do bot para que servem os dados coletados, e ter uma forma de apagar os dados de quem pedir. Isto não é um parecer jurídico — é só uma checklist prática.

#### Nó: `Google Gemini` (gerar plano)
- **Credential**: `Gemini API`
- **Model**: o modelo "flash" atual (veja passo 2.2)
- **Prompt**: template específico para planos (`docs/prompt-templates.md`)

3. **Salve** e **Ative**

---

## 6️⃣ Fazer Upload dos PDFs

### Opção A: Via Workflow 01

1. No workflow `01-pdf-ingestion`, adicione um nó **Manual Trigger**
2. Adicione **Read Binary File** (PDFs no seu computador) ou **Google Drive Trigger** (PDFs no Drive)
3. Execute o workflow manualmente

### Opção B: Via SQL Direto (Avançado)

1. Extraia o texto dos PDFs manualmente
2. No **SQL Editor** do Supabase:

```sql
insert into pdf_embeddings (content, metadata)
values (
  'Texto extraído do PDF...',
  '{"fonte": "nutricao_esportiva.pdf", "pagina": 1}'::jsonb
);
```

> Nota: inserir assim não gera o `embedding` — a coluna fica `null` e o item nunca aparece numa busca por similaridade. Essa opção serve só para guardar texto de referência que você vai processar depois; para entrar de fato na busca, o texto precisa passar pelo node de embedding do Workflow 01.

---

## 7️⃣ Testar o Bot

1. Abra `@NutriTrainAI_bot` no Telegram
2. Envie `/start`
3. Responda as perguntas que o bot pedir
4. Envie `/plano`
5. O bot deve gerar um plano baseado nos seus dados + PDFs

---

## 🐛 Troubleshooting

### Erro: "Failed to connect to Supabase"
- Confira **Project URL** e a **service_role key**
- Confirme que a tabela existe: `select * from pdf_embeddings limit 1;`

### Erro: "permission denied for table pdf_embeddings"
- Você está usando a chave `anon` em vez da `service_role`. Como o RLS está ligado sem política pública (de propósito, veja passo 1.5), a `anon` não tem acesso mesmo.

### Erro: "function match_pdf_embeddings does not exist"
- O passo 1.4 não foi executado, ou o campo **Query Name** no node de busca (passo 5.2) não está com o nome certo.

### Erro: "Gemini API rate limit exceeded" / 429
- Você excedeu a cota do free tier do dia/minuto. Confira os limites atuais em https://ai.google.dev/gemini-api/docs/rate-limits — eles mudam com frequência, então não dá para confiar num número fixo.

### Erro: "Telegram Bot: Unauthorized"
- Token errado. Gere um novo com `/revoke` no @BotFather e atualize no n8n.

### Workflow não executa
- Confira se está **Ativo** (toggle verde)
- Veja os **logs de execução** (workflow → Execution List)

### Dimensão do embedding não bate com a tabela
- Confirme que o node de embedding está configurado para devolver **768** dimensões (passo 5.1). Se trocar de modelo de embedding no futuro, ajuste também a coluna `vector(768)` da tabela — e reprocesse os PDFs, porque embeddings de modelos diferentes não são comparáveis entre si.

---

## 📞 Suporte

- **n8n Community**: https://community.n8n.io
- **Supabase Discord**: https://discord.supabase.com
- **Gemini API Docs**: https://ai.google.dev/gemini-api/docs
- **pgvector (índices, sintaxe)**: https://github.com/pgvector/pgvector

---

**Pronto! Agora você tem um agente de IA funcional para dieta e treino — e um pouco mais seguro do que a versão original.** 🎉
