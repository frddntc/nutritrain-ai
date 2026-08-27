# setup.md — Manual de Implementação: Agente de Nutrição e Treino (Discord + Gemini + n8n + Supabase)

> **Objetivo:** Construir um agente de IA focado em dieta e treino, com custo zero, integrado ao Discord.
>
> **Stack:**
> - **n8n Cloud** — Orquestração de workflows.
> - **Supabase** — PostgreSQL, pgvector (RAG) e tabelas estruturadas.
> - **Google AI Studio (Gemini)** — LLM (`gemini-3.1-flash-lite`) e Embeddings via Free Tier.
> - **n8n Code Node** — Cálculos determinísticos (substitui o microserviço Python separado — veja por quê na ETAPA 4).
> - **Discord API** — Interações via slash commands.

---

## ⚠️ Resumo das mudanças desta revisão

Antes de mais nada, um changelog rápido em relação à v1 do documento, pra você entender o que mudou e por quê:

| # | Mudança | Por quê |
|---|---|---|
| 1 | Microserviço FastAPI → **Code Node nativo do n8n** | n8n Cloud não alcança `localhost`. Um serviço no seu VS Code é invisível para ele. Se você preferir manter o FastAPI, veja a alternativa "avançada" na ETAPA 4. |
| 2 | Adicionada **verificação de assinatura Ed25519** | O Discord recusa salvar o endpoint sem isso — não é opcional. |
| 3 | Interação passa a ser **100% via slash commands** (`/mensagem`, `/novo_plano`) | O webhook de interações do Discord não recebe mensagens digitadas livremente — só slash commands, botões e modais. |
| 4 | Reset de contexto por **usuário**, não por canal | Evita depender do usuário lembrar de rodar `/novo_plano` toda vez que outra pessoa for usar o bot no mesmo canal. |
| 5 | Modelo de embedding atualizado para `gemini-embedding-001` (768d) | `text-embedding-004` ainda funciona, mas o Google já indica o novo modelo como padrão. |
| 6 | Tratamento de falhas do Free Tier (rate limit) | Sem isso, o bot trava em "pensando..." indefinidamente se o Gemini responder 429. |
| 7 | Nota sobre nome/persona do bot | "Nutrólogo" é título de médico regulamentado no Brasil — evite essa palavra pro bot e adicione um disclaimer. Ver ETAPA 9. |

---

## 1. Arquitetura do Sistema

```text
                     ┌───────────────────────────┐
                     │          DISCORD           │
                     │  (slash commands: /mensagem,│
                     │   /novo_plano)              │
                     └─────────────┬───────────────┘
                                   │ POST (Interaction)
                                   ▼
                     ┌────────────────────────────┐
                     │   n8n — Webhook Node        │
                     │   (Raw Body habilitado)     │
                     └─────────────┬───────────────┘
                                   ▼
                     ┌────────────────────────────┐
                     │  Code Node: verificar        │
                     │  assinatura Ed25519          │
                     └─────────────┬───────────────┘
                          válida?  │  inválida → 401
                                   ▼
                     ┌────────────────────────────┐
                     │  IF: type == 1 (PING)?       │──▶ Responde {"type":1}
                     └─────────────┬───────────────┘
                                   │ não (type == 2, comando)
                                   ▼
                     ┌────────────────────────────┐
                     │  Responde {"type":5}         │  (ACK "pensando...",
                     │  (Respond to Webhook)        │   dentro de 3s)
                     └─────────────┬───────────────┘
                                   ▼
                     ┌────────────────────────────┐
                     │  IF: comando == /novo_plano? │
                     └───────┬───────────┬─────────┘
                       sim   │           │ não (/mensagem)
                             ▼           ▼
                  ┌────────────────┐ ┌─────────────────────┐
                  │ Supabase DELETE│ │   AI Agent Node       │
                  │ (memória do    │ │   (gemini-3.1-flash-  │
                  │  autor.id)     │ │    lite)              │
                  └───────┬────────┘ └──────────┬───────────┘
                          │                      │
                          │         ┌────────────┼────────────┐
                          │         ▼            ▼             ▼
                          │  Scientific RAG  Code Node      Postgres
                          │  (Supabase        (cálculo de    Chat Memory
                          │   pgvector 768d)   TDEE/macros)  (session = autor.id)
                          │         │            │
                          │         └─────┬──────┘
                          ▼               ▼
                     ┌────────────────────────────┐
                     │        Plano final           │
                     └─────────────┬───────────────┘
                                   │ PATCH /webhooks/{app_id}/{token}/messages/@original
                                   ▼
                                DISCORD
```

---

## 2. ETAPA 1 — Configuração do Google AI Studio (Custo Zero)

1. Acesse `aistudio.google.com` com sua conta Google.
2. Crie um projeto e gere uma **API Key**.
3. No n8n, vá em *Credentials* → *Add Credential* → **Google Gemini API** e insira a chave.
4. Antes de codar qualquer coisa, abra a página de **Rate Limits** do seu projeto no AI Studio e anote os limites do Free Tier (requisições por minuto e por dia). Esses números mudam com frequência — não confie em valores que você viu em tutoriais antigos. Isso importa porque o ETAPA 8 depende de saber o que fazer quando você estourar esse limite.

---

## 3. ETAPA 2 — Banco de Dados Supabase e pgvector

Sem mudanças estruturais aqui — o schema de 768 dimensões continua correto tanto para `text-embedding-004` quanto para o novo `gemini-embedding-001` (que suporta 768, 1536 ou 3072 dimensões configuráveis; usamos 768 para manter compatibilidade e custo baixo).

No SQL Editor do Supabase, rode:

```sql
-- Ativar extensão
create extension if not exists vector with schema extensions;

-- Criar tabela de documentos científicos
create table if not exists public.scientific_documents (
    id bigint generated by default as identity primary key,
    content text not null,
    metadata jsonb not null default '{}'::jsonb,
    embedding extensions.vector(768),
    created_at timestamptz not null default now()
);

-- Criar função de busca semântica
create or replace function public.match_scientific_documents(
    query_embedding extensions.vector(768),
    match_count int default 8,
    filter jsonb default '{}'::jsonb
)
returns table (
    id bigint,
    content text,
    metadata jsonb,
    similarity float
)
language sql stable
as $$
    select
        d.id, d.content, d.metadata,
        1 - (d.embedding <=> query_embedding) as similarity
    from public.scientific_documents d
    where d.metadata @> filter
    order by d.embedding <=> query_embedding
    limit match_count;
$$;

-- Opcional (recomendado quando a biblioteca crescer): índice para buscas rápidas
create index if not exists scientific_documents_embedding_idx
    on public.scientific_documents
    using hnsw (embedding vector_cosine_ops);
```

> 💡 O índice HNSW é opcional no começo (com poucos documentos, a busca sem índice já é rápida), mas vale criar desde já — não tem custo de manutenção relevante nessa escala.

---

## 4. ETAPA 3 — Ingestão de Documentos (RAG) no n8n

1. **Trigger:** Manual Trigger.
2. **Default Data Loader:** Lê os PDFs científicos.
3. **Recursive Character Text Splitter:** Chunk Size 1200, Overlap 200.
4. **Google Gemini Embeddings:** selecione `gemini-embedding-001` no dropdown do nó (se disponível na sua versão do n8n; se só aparecer `text-embedding-004`, pode usar normalmente — ambos geram vetores de 768 dimensões compatíveis com a tabela). Se o nó pedir explicitamente uma dimensão de saída, defina **768**.
5. **Supabase Vector Store:** conecte à tabela `scientific_documents`.

---

## 5. ETAPA 4 — Cálculos Determinísticos (Formula Tools)

### Caminho recomendado: Code Node nativo (zero hospedagem)

Como você está no n8n Cloud, a forma mais simples de isolar a matemática — sem depender de nenhum serviço externo — é usar um **Code Node** dentro do AI Agent como *Custom Tool*, escrito em Python (o n8n roda Python nativamente no Code Node via task runner, sem precisar instalar nada).

1. No AI Agent, adicione uma ferramenta **Code Tool** (ou um Code Node conectado como tool).
2. Configure a linguagem como **Python** e o modo **Run Once for Each Item**.
3. Cole uma lógica como esta (ajuste conforme sua base científica):

```python
# Tool: calcular_tdee_macros
# Entrada esperada (dados que o AI Agent vai extrair da conversa):
# peso_kg, altura_cm, idade, sexo ('M'/'F'), bf_percent (opcional),
# nivel_atividade ('sedentario'|'leve'|'moderado'|'intenso'|'muito_intenso'),
# objetivo ('emagrecimento'|'manutencao'|'hipertrofia')

dados = _json

peso_kg = dados['peso_kg']
altura_cm = dados['altura_cm']
idade = dados['idade']
sexo = dados['sexo']
bf_percent = dados.get('bf_percent')
nivel_atividade = dados.get('nivel_atividade', 'leve')
objetivo = dados.get('objetivo', 'manutencao')

# 1. Taxa Metabólica Basal (Mifflin-St Jeor)
if sexo == 'M':
    tmb = 10 * peso_kg + 6.25 * altura_cm - 5 * idade + 5
else:
    tmb = 10 * peso_kg + 6.25 * altura_cm - 5 * idade - 161

multiplicadores = {
    'sedentario': 1.2, 'leve': 1.375, 'moderado': 1.55,
    'intenso': 1.725, 'muito_intenso': 1.9,
}
tdee = tmb * multiplicadores.get(nivel_atividade, 1.375)

# 2. Ajuste calórico pelo objetivo
ajustes = {'emagrecimento': -0.20, 'manutencao': 0.0, 'hipertrofia': 0.12}
calorias_alvo = tdee * (1 + ajustes.get(objetivo, 0.0))

# 3. Massa magra estimada — evita superestimar proteína em perfis com % de gordura alto
if bf_percent:
    massa_magra_kg = peso_kg * (1 - bf_percent / 100)
else:
    massa_magra_kg = peso_kg * 0.80  # estimativa conservadora sem %BF

# 4. Proteína por kg de massa magra (não peso total)
proteina_g = massa_magra_kg * (2.0 if objetivo == 'hipertrofia' else 1.8)

# 5. Gordura — piso para saúde hormonal
gordura_g = peso_kg * 0.7

# 6. Carboidrato preenche o restante das calorias
kcal_restante = calorias_alvo - (proteina_g * 4) - (gordura_g * 9)
carboidrato_g = max(kcal_restante / 4, 0)

resultado = {
    'tmb': round(tmb),
    'tdee': round(tdee),
    'calorias_alvo': round(calorias_alvo),
    'massa_magra_kg': round(massa_magra_kg, 1),
    'macros': {
        'proteina_g': round(proteina_g),
        'gordura_g': round(gordura_g),
        'carboidrato_g': round(carboidrato_g),
    },
}

return [{'json': resultado}]
```

Isso resolve exatamente o problema que o documento original queria resolver (matemática confiável, fora do "raciocínio" do LLM) sem precisar expor nada publicamente.

### Alternativa avançada: manter o FastAPI

Se você já tem esse código em Python e quer reaproveitar (ou pretende usar bibliotecas pesadas que o Code Node não suporta), tudo bem manter o FastAPI — mas ele precisa estar **acessível publicamente por HTTPS**, não rodando só no seu VS Code. Opções comuns: Render, Railway, Fly.io (verifique o plano gratuito atual de cada uma antes de decidir, pois os limites mudam com frequência). Depois de hospedado, use uma **HTTP Request Node** como Custom Tool apontando para a URL pública.

---

## 6. ETAPA 5 — Discord: Timeout de 3s e Verificação de Assinatura

Esta é a etapa mais delicada e onde o documento original tinha a lacuna mais crítica. Vamos por partes.

### 6.1 Por que a verificação de assinatura é obrigatória

Quando você cola a URL do seu webhook n8n no campo "Interactions Endpoint URL" do Discord Developer Portal, o Discord envia requisições de teste (com assinaturas válidas e inválidas de propósito) e só aceita salvar a URL se seu endpoint validar corretamente cada uma. Sem essa validação, você nem consegue configurar o endpoint — o fluxo trava antes de começar.

### 6.2 Configurando o Webhook Node

1. **Webhook Node:** método POST, path à sua escolha (ex: `/discord-interactions`).
2. Na aba **Options** do Webhook Node, ative **Raw Body**. Isso é essencial: a assinatura do Discord é calculada sobre os bytes crus do corpo da requisição, e se o n8n já tiver reformatado o JSON, a verificação vai falhar mesmo para requisições legítimas.
3. Configure **Respond**: `Using 'Respond to Webhook' Node` (em vez de "Immediately" — assim você controla exatamente o que responder em cada caso).

### 6.3 Code Node: verificar assinatura

Logo após o Webhook, adicione um Code Node (JavaScript) com esta lógica:

```javascript
// Node: "Verificar Assinatura Discord"
const crypto = require('crypto');

// Guarde isso como credencial/variável, não hardcoded em produção
const PUBLIC_KEY_HEX = 'COLOQUE_AQUI_A_PUBLIC_KEY_DO_SEU_APP_DISCORD';

const headers = $input.item.json.headers || $input.item.headers;
const signature = headers['x-signature-ed25519'];
const timestamp = headers['x-signature-timestamp'];

// Prefira sempre o raw body (ativado no passo anterior)
const rawBody = $getWorkflowStaticData
  ? JSON.stringify($input.item.json.body ?? $input.item.json)
  : JSON.stringify($input.item.json);

const message = Buffer.from(timestamp + rawBody, 'utf-8');

const publicKeyDer = Buffer.concat([
  Buffer.from('302a300506032b6570032100', 'hex'), // prefixo fixo ASN.1/SPKI para Ed25519
  Buffer.from(PUBLIC_KEY_HEX, 'hex'),
]);

const publicKey = crypto.createPublicKey({
  key: publicKeyDer, format: 'der', type: 'spki',
});

const valida = crypto.verify(null, message, publicKey, Buffer.from(signature, 'hex'));

return [{ json: { ...($input.item.json.body ?? $input.item.json), _assinatura_valida: valida } }];
```

> ⚠️ **Teste isso antes de seguir em frente.** O `crypto` é um módulo nativo do Node (diferente de pacotes npm externos, que exigem configuração extra e nem sempre estão liberados no n8n Cloud). Rode esse Code Node isoladamente com um dado de teste para confirmar que `require('crypto')` funciona no seu ambiente antes de conectar o resto do fluxo. Se der erro de módulo bloqueado, me avise que ajustamos a abordagem (existe um caminho alternativo via Web Crypto API em runtimes que bloqueiam `require`).

### 6.4 Roteamento depois da verificação

1. **IF Node** — `{{$json._assinatura_valida}}` é `false`?
   → **Respond to Webhook** com Response Code **401** e encerra.
2. Se válida, **IF Node** — `{{$json.type}}` é `1` (PING, usado só na etapa de configuração no Developer Portal)?
   → **Respond to Webhook** com body `{"type": 1}`.
3. Senão (é `type: 2`, ou seja, um slash command real):
   → **Respond to Webhook** com body `{"type": 5}` (ACK "pensando...").
   → O workflow **continua executando** depois desse node — é assim que o padrão de resposta adiada funciona no n8n.
4. **HTTP Request Node (fim do workflow):** PATCH para `https://discord.com/api/v10/webhooks/{application.id}/{interaction.token}/messages/@original`, substituindo "pensando..." pelo plano final.

### 6.5 Importante: só slash commands chegam aqui

O endpoint de interações do Discord **não recebe mensagens digitadas livremente** no chat — só slash commands, cliques em botões e envios de modal. Isso significa que "conversar" com o agente precisa acontecer via comando de barra, por exemplo `/mensagem texto:"quero um plano para hipertrofia"`.

Para registrar os comandos de barra (isso não tem interface visual no Developer Portal — é feito via API), rode uma vez (pode ser um HTTP Request Node avulso no n8n, ou `curl`):

```bash
curl -X PUT \
  "https://discord.com/api/v10/applications/{application.id}/commands" \
  -H "Authorization: Bot SEU_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[
    {
      "name": "mensagem",
      "description": "Conversar com o agente de dieta e treino",
      "options": [
        {"name": "texto", "description": "Sua mensagem", "type": 3, "required": true}
      ]
    },
    {
      "name": "novo_plano",
      "description": "Reseta seu contexto para começar um novo plano"
    }
  ]'
```

> Se no futuro você quiser chat livre (sem precisar digitar `/mensagem` toda vez), isso exige um bot conectado ao Gateway do Discord — um processo persistente rodando 24/7 em algum host (Railway, Render worker, etc.), o que sai do escopo "só n8n Cloud". Comece com slash commands; é o caminho mais simples e já cobre bem o caso de uso.

---

## 7. ETAPA 6 — Múltiplos Usuários no Mesmo Canal (Memória por Pessoa)

Em vez de resetar manualmente com `/novo_plano` toda vez que outra pessoa for usar o bot no mesmo canal, a forma mais robusta é isolar a memória **por usuário**, automaticamente:

1. No node **Postgres Chat Memory** (conectado ao AI Agent), configure o campo **Session ID** com uma expressão apontando para o ID do autor da interação, não o ID do canal:
   `{{ $json.member.user.id }}` (ou `$json.user.id`, dependendo de onde vier o payload — confira a estrutura exata no seu teste).
2. Assim, cada pessoa que interage no canal automaticamente tem seu próprio histórico, sem interferir no de outra.
3. Mantenha `/novo_plano` como um reset **opcional e pessoal**: ao ser chamado, o Supabase deve fazer `DELETE` filtrando pelo mesmo `session_id` (autor.id) de quem chamou o comando — não pelo canal inteiro. Confirme o nome real da tabela que o node de memória cria no seu banco (pode variar conforme a versão do n8n) antes de escrever a query de `DELETE`.
4. Uma mensagem de confirmação como *"Contexto resetado! Pode enviar peso, altura e idade do novo aluno."* continua fazendo sentido — só que agora ela reseta o histórico de quem pediu, não de todo mundo no canal.

---

## 8. ETAPA 7 — Tratamento de Exceções (Validation Loop)

Duas camadas de robustez aqui:

**8.1 Discrepância nos cálculos (como no documento original).** Instrua no *System Prompt* do AI Agent:

> "Se a ferramenta de cálculo retornar uma distribuição de macros que não bate com a meta calórica, chame a ferramenta novamente com valores ajustados antes de entregar a resposta final. Não exponha esse processo de tentativa ao usuário."

**8.2 Falhas de infraestrutura (rate limit do Free Tier, timeout do Gemini, etc.) — isso não estava no documento original e é importante.** Sem tratamento, se o Gemini responder erro 429 (limite excedido), o workflow falha silenciosamente e a mensagem do Discord fica travada em "pensando..." para sempre, sem nenhum feedback ao usuário.

Configure:
1. No AI Agent Node e nos HTTP Request Nodes que chamam APIs externas, ative **Continue on Fail** (ou **Retry on Fail** com 1–2 tentativas e um pequeno delay).
2. Depois desses nodes, adicione um **IF** checando se houve erro. Se sim, ainda assim faça o PATCH final no Discord — mas com uma mensagem amigável, por exemplo: *"Estou recebendo muitas requisições agora (limite do plano gratuito). Tente novamente em alguns minutos."*

Isso garante que o usuário nunca fique olhando para um "pensando..." eterno.

---

## 9. Segurança, Custos e Avisos

- **Chaves e tokens** (API Key do Gemini, Bot Token do Discord, credenciais do Supabase) devem ficar só nas Credentials do n8n — nunca hardcoded em Code Nodes que você venha a compartilhar ou versionar em algum repositório.
- **Sobre o nome/persona do bot:** "Nutrólogo" é título de médico com especialização reconhecida pelo CFM no Brasil. Chamar o bot por esse nome (ou deixá-lo se apresentar como tal) pode passar a impressão de que ele é um profissional de saúde de verdade, o que é enganoso e pode te trazer problema se você disponibilizar o bot para outras pessoas usarem. Sugestões práticas:
  - Nomeie o bot como algo como "Copiloto de Dieta e Treino" ou "Assistente de Nutrição Esportiva", evitando termos regulamentados (nutrólogo, nutricionista, médico, endocrinologista).
  - Inclua no *System Prompt* uma instrução para o modelo sempre se identificar como assistente de IA, recomendar avaliação profissional presencial (nutricionista/médico) antes de iniciar qualquer plano — especialmente se o usuário mencionar alguma condição de saúde, gravidez, ser menor de idade, ou usar medicação que afete metabolismo/apetite.
  - Considere adicionar uma linha fixa de disclaimer no final de cada plano gerado.
- **Rate limits do Free Tier** mudam com frequência — revise periodicamente a página de limites no AI Studio, principalmente se for convidar mais pessoas para testar o bot.

---

## 10. Conclusão

Com esses ajustes, o sistema roda inteiramente dentro do que o n8n Cloud consegue fazer sozinho — sem precisar hospedar nada além do próprio n8n e do Supabase:
- Os cálculos determinísticos rodam num Code Node dentro do próprio n8n (sem servidor externo).
- O Discord se comunica corretamente porque o endpoint valida assinatura e trata PING/ACK como o Discord exige.
- Múltiplos usuários no mesmo canal ficam isolados automaticamente por ID de usuário, sem depender de reset manual.
- Falhas de rate limit do Free Tier não deixam o bot travado, com feedback claro para o usuário.
- O Gemini (LLM + embeddings) cuida de toda a cognição e busca científica por custo zero.
