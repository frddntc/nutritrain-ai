# Brainstorm Técnico — Agente de IA para Dieta e Treino (n8n Cloud + Gemini)

> Documento de discussão de arquitetura. Nenhum workflow ou código foi implementado — o objetivo é permitir que você escolha os componentes antes de partirmos para a construção.

---

## 1. Resumo do projeto

Você quer construir, como projeto de estudo do **n8n Cloud**, um agente conversacional que conduz uma anamnese estruturada, consulta uma base de conhecimento própria (artigos científicos de nutrição/treino + um arquivo de fórmulas) via **RAG**, executa cálculos determinísticos, e gera **planos de dieta e treino individualizados com justificativa e citação de fontes** — nunca inventando dados, fórmulas ou evidências. O sistema deve:

- Coletar no mínimo idade, sexo, altura, peso e objetivo antes de qualquer plano;
- Fundamentar toda recomendação em documentos da base de conhecimento (recusando-se explicitamente quando não houver base suficiente);
- Explicar o raciocínio (dado → cálculo → evidência → decisão);
- Estimar eficácia/prazo de forma honesta, com cenários por nível de adesão, sem fingir precisão que a ciência não sustenta;
- Validar o plano com o usuário e revisá-lo de forma explicável;
- Rodar inteiramente em **ferramentas gratuitas**, usando as chaves Gemini disponíveis (Embedding 001, Embedding 2, Gemini 3 Flash, Gemini 3.1 Flash-Lite) e o **n8n Cloud**.

Entendido — e esse é um projeto tecnicamente sólido para aprender n8n, porque ele força o uso de praticamente todos os blocos avançados da plataforma: webhooks, memória de conversa, agentes com ferramentas, RAG (embeddings + vector store), nós de código para cálculo determinístico, ramificações condicionais de segurança, e persistência em banco de dados. Antes de qualquer recomendação de arquitetura, a seção 2 traz um ponto crítico que muda a forma como você deve planejar o uso do "gratuito".

---

## 2. Pontos críticos e riscos

### 2.1 — O maior risco: **n8n Cloud não tem plano gratuito permanente**

Pesquisei isso especificamente porque era a premissa central do seu pedido. O quadro atual (2026) é:

| Item | Situação |
|---|---|
| n8n Cloud (hospedado pela n8n) | **Sem free tier permanente.** Apenas trial de 14 dias, sem cartão de crédito. Depois disso, o plano mais barato (Starter) é pago (na faixa de US$20–24/mês, com limite de execuções). |
| n8n Community Edition (self-hosted) | **Gratuito para sempre**, open source, sem limite de execuções — mas você precisa hospedá-lo (servidor próprio, VPS, Docker local, etc.). |

Isso é uma contradição direta entre dois requisitos seus: "desenvolver no n8n Cloud" e "usar exclusivamente ferramentas gratuitas". Não dá para ter as duas coisas indefinidamente sem pagar. Três caminhos possíveis, que retomo na recomendação final (seção 16):

1. **Usar os 14 dias de trial do n8n Cloud intensivamente** para aprender a plataforma "de verdade" (interface cloud, deploy, credenciais gerenciadas), migrando depois para self-hosted gratuito com o mesmo workflow (o n8n exporta/importa workflows em JSON — a curva de aprendizado da *lógica* dos nós é praticamente a mesma nos dois modos).
2. **Ir direto para self-hosted** (Docker local, ou uma VPS gratuita como Oracle Cloud Always Free) desde o início, abrindo mão do ambiente "Cloud" oficial, mas mantendo 100% do aprendizado sobre nós, expressões, RAG, etc.
3. Aceitar que, passado o trial, o projeto pode precisar de um pequeno aporte mensal se você quiser continuar exatamente no n8n Cloud gerenciado.

Não existe uma opção que satisfaça literalmente "n8n Cloud" + "grátis para sempre" simultaneamente — é importante que essa decisão seja sua e consciente, não uma surpresa no dia 15.

### 2.2 — A regra "não inventar" é adequada, mas RAG sozinho não a garante

Você pediu para eu avaliar isso tecnicamente: **sim, a regra é o objetivo correto de um sistema RAG bem feito**, mas "colocar documentos no contexto do prompt" não impede, por si só, que o modelo:

- Ignore o contexto recuperado e responda com conhecimento paramétrico (o que "sabe" do pré-treinamento);
- Misture informação real do documento com complementos inventados para "soar completo";
- Faça contas de cabeça (LLMs são notoriamente não confiáveis em aritmética, mesmo com contexto correto).

Ou seja: RAG é **necessário mas não suficiente**. A regra do projeto precisa ser operacionalizada com mecanismos estruturais, não apenas uma instrução de texto no system prompt. Mecanismos concretos (detalhados na seção 4):

- **Saída estruturada obrigatória** com campo de "fonte" por afirmação nutricional/de treino, validado programaticamente contra os IDs de documentos realmente recuperados naquela consulta;
- **Limiar mínimo de similaridade** na busca vetorial — abaixo dele, o agente é instruído a declarar lacuna, não a responder mesmo assim;
- **Segunda chamada de verificação de grounding** (barata, com Flash-Lite): dado o texto gerado + o contexto usado, o modelo responde apenas se cada afirmação está sustentada — funciona como uma checagem de consistência antes de entregar;
- **Cálculos nunca feitos pelo LLM** — sempre em nó de código (elimina uma classe inteira de alucinação numérica);
- **Temperatura baixa** (próxima de 0) nas etapas factuais/de cálculo textual.

### 2.3 — Outros riscos técnicos e operacionais

| Risco | Por quê importa | Mitigação |
|---|---|---|
| **Free tiers "pausam" ou expiram por inatividade** | Qdrant Cloud free suspende cluster após 1 semana sem uso e **deleta após 4 semanas**; Supabase free pausa o projeto após 7 dias sem atividade (mas não deleta). Para um projeto de estudo usado esporadicamente, isso pode "quebrar" a demo sem aviso. | Um workflow n8n com Cron (ex.: 1x a cada 4–5 dias) fazendo um ping/consulta trivial no banco resolve isso sem custo. |
| **Limites da API Gemini gratuita mudam com frequência** | Encontrei relatos de reduções de cota em dez/2025 e números que variam bastante entre fontes (modelos Flash ficam em torno de 10–15 RPM e centenas a ~1.500 requisições/dia; Pro é muito mais restrito). Não trate nenhum número fixo como garantido. | Centralize o nome do modelo em uma variável/nó de configuração no workflow, monitore o painel de rate limits do Google AI Studio, e implemente tratamento de erro 429 com backoff. |
| **LGPD / dados sensíveis de saúde** | O sistema armazenará idade, peso, lesões, condições médicas e medicamentos — dados sensíveis pela LGPD (art. 5º, II). Mesmo em projeto pessoal de estudo, vale desenhar como se fosse produção. | Consentimento explícito no primeiro contato, minimização de dados, e cuidado extra se algum dia isso sair do escopo "uso pessoal". |
| **WhatsApp "grátis" tem complexidade desproporcional** | A API oficial (Cloud API) exige verificação de Business Manager na Meta; pontes não oficiais (ex. Baileys/QR code) têm risco real de banimento do número. Para um projeto de aprendizado, isso é atrito que não ensina nada sobre n8n. | Comece com Telegram; avalie WhatsApp depois, se ainda fizer sentido. |
| **Escopo muito ambicioso para uma primeira implementação** | Os 17 pontos do seu documento formam, juntos, um sistema de nível "produto", não um primeiro workflow de estudo. | Roadmap incremental na seção 17 — comece pequeno, adicione camadas. |
| **Nomenclatura dos modelos Gemini que você informou** | "Gemini Embedding 1/2" e "Gemini 3 Flash / 3.1 Flash-Lite" existem e batem com os nomes reais da API atual (`gemini-embedding-001`, `gemini-embedding-2`, `gemini-3-flash`, `gemini-3.1-flash-lite`) — mas a Google já lançou gerações mais novas (3.5 Flash, 3.5 Flash-Lite, 3.6 Flash) depois desses. Vale checar no seletor de modelos do AI Studio se essas versões mais novas também estão liberadas nas suas chaves — não é obrigatório trocar, mas é bom saber que existem. | Ver seção 9. |

---

## 3. Informações que o chatbot deve coletar

Mantendo a regra que você definiu — **apenas Idade + Sexo/Gênero + Altura + Objetivo + Peso bloqueiam a geração do plano** — organizo o resto por impacto real na qualidade e segurança da recomendação.

### 🔴 Obrigatórias (bloqueiam a geração do plano — definidas por você)

| Campo | Por que é inegociável |
|---|---|
| Idade | Entra em toda fórmula de TMB/GET; também aciona a lógica de segurança para menores. |
| Sexo/gênero biológico relevante ao cálculo | Fórmulas de TMB e distribuição de massa magra diferem por sexo. |
| Altura | Necessária para TMB, IMC de referência e algumas fórmulas de composição corporal. |
| Peso | Base de qualquer cálculo calórico e de macros. |
| Objetivo | Sem objetivo, não há "plano" — é o que direciona déficit/superávit/manutenção. |

### 🟡 Recomendadas (o agente deve perguntar ativamente; se o usuário não responder, o plano pode ser gerado, mas com ressalvas explícitas e abordagem mais conservadora)

Separo estas em duas naturezas — **precisão** e **segurança** — porque o impacto de omiti-las é diferente:

**Precisão do plano:**
- Nível de atividade diária (fora do treino) — afeta diretamente o GET;
- Experiência de treino — define progressão e complexidade dos exercícios;
- Dias disponíveis/semana e tempo por sessão — define volume e divisão de treino;
- Local de treino e equipamentos disponíveis — sem isso, o plano de treino não é executável de fato;
- Percentual de gordura, se o usuário souber — permite fórmulas mais precisas (ex. Katch-McArdle) em vez de estimativas populacionais;
- Padrão alimentar (onívoro, vegetariano, vegano, restrição religiosa) — sem isso o plano alimentar pode ser inutilizável;
- Número de refeições preferido.

**Segurança (recomendo tratar como quase-obrigatórias na prática, mesmo não bloqueando):**
- Lesões, dores, limitações de movimento;
- Condições médicas relevantes (diabetes, hipertensão, cardiopatias, doenças renais);
- Medicamentos em uso;
- Alergias e intolerâncias alimentares;
- Gravidez/amamentação, quando aplicável.

> **Minha recomendação de design:** essas informações de segurança não devem impedir a geração do plano (respeitando sua regra), mas a ausência delas deve **forçar o agente a declarar explicitamente essa lacuna no plano entregue** ("Você não informou lesões ou condições médicas — este plano assume ausência delas; se houver alguma, revise com um profissional antes de iniciar") e a adotar parâmetros mais conservadores por padrão (ex. não sugerir déficits calóricos agressivos sem confirmação de ausência de risco).

### 🟢 Opcionais (melhoram personalização, mas têm baixo risco se ausentes)

- Histórico de peso, medidas corporais detalhadas;
- Objetivos secundários, prazo desejado, prioridade estética vs. performance;
- Exercícios preferidos/que não gosta, histórico de treinamento;
- Sono (horário/qualidade), rotina de trabalho/estudo;
- Capacidade de cozinhar, refeições fora de casa, orçamento;
- Perguntas de adesão: disciplina autopercebida, dificuldades anteriores, flexibilidade desejada, histórico de aderência a planos passados.

Essas últimas (adesão) alimentam diretamente a metodologia da seção 13 — vale coletá-las, mas de forma leve, para não transformar a anamnese em um questionário exaustivo que derruba a taxa de conclusão.

---

## 4. Arquitetura RAG

### 4.1 — Onde armazenar os documentos originais
Uma pasta no **Google Drive** (gratuito, 15GB) é o ponto de entrada mais simples e tem nó nativo no n8n (trigger de "novo arquivo na pasta" + download). Estrutura sugerida:

```
/base-conhecimento
  /nutricao
  /treino
  /composicao-corporal
  /formulas   (o .md de fórmulas vive aqui, versionado)
```

### 4.2 — Pipeline de ingestão (indexação) — roda separado do fluxo de conversa
1. **Trigger**: novo/alterado arquivo na pasta do Drive (ou execução manual ao adicionar PDFs);
2. **Extração de texto**: nó de extração de PDF do n8n (ou, para PDFs mais complexos com tabelas, envio direto do PDF para o Gemini via API, que tem compreensão nativa de documentos);
3. **Chunking**: Text Splitter (recursivo, por caractere/token) — comece com blocos de ~300–500 tokens e sobreposição de ~10–15%, ajustando empiricamente depois;
4. **Metadados por chunk**: título do documento, ano, tipo de estudo, tema (nutrição/treino/cálculo), população, objetivo relacionado, nível de evidência, status (ativo/desatualizado) — ver estrutura completa na seção 17 (documento original, seção "Estrutura dos documentos");
5. **Embeddings**: `gemini-embedding-001` (texto), com `output_dimensionality` truncado para 768 (via Matryoshka Representation Learning — perde muito pouca qualidade e rende 4x mais espaço nos free tiers de banco vetorial, que são o recurso mais escasso);
6. **Armazenamento**: inserção no vector store escolhido (seção 5), com os metadados como *payload* filtrável.

### 4.3 — Pipeline de consulta (recuperação em tempo real)
1. A partir do **objeto estruturado do usuário** (seção 11) — não da mensagem crua do chat — o agente monta uma ou mais queries de busca (ex.: "déficit calórico para perda de gordura em iniciante sedentário", "hipertrofia 3x/semana equipamento limitado");
2. Embedding da query com o mesmo modelo usado na indexação (`gemini-embedding-001` — **nunca misturar modelos de embedding no mesmo índice**, os espaços vetoriais são incompatíveis entre `gemini-embedding-001` e `gemini-embedding-2`);
3. Busca vetorial top-k (ex. k=6 a 10) **combinada com filtro de metadados** (tema compatível com o objetivo, status="ativo");
4. **Corte por limiar de similaridade** — chunks abaixo do limiar são descartados, não usados "de qualquer jeito";
5. Contexto montado com citação de origem (documento + trecho) e entregue ao Gemini 3 Flash junto com o objeto estruturado do usuário e o resultado dos cálculos;
6. Geração com instrução explícita de grounding + resposta estruturada com campo de fontes por recomendação (seção 4 acima).

### 4.4 — Como evitar documentos irrelevantes ou conflitantes
- Filtro de metadados **antes** da busca semântica (não depois) reduz ruído;
- Campo `status` (ativo/desatualizado) no metadado, atualizado manualmente quando um estudo é substituído por evidência mais recente;
- Em caso de conflito entre documentos (ex. dois estudos com conclusões diferentes), o agente deve ser instruído a **declarar o conflito** e priorizar revisões sistemáticas/meta-análises sobre estudos isolados — quando o campo "tipo de estudo" permitir essa distinção;
- Reranqueamento leve opcional (um segundo passe com Flash-Lite pontuando os chunks recuperados por relevância real à pergunta) se k=10 estiver trazendo muito ruído — custo baixo, ganho de precisão.

---

## 5. Opções de banco de conhecimento (vector database)

| Critério | **Qdrant Cloud (Free)** | **Supabase (Postgres + pgvector, Free)** | **Pinecone (Starter, Free)** |
|---|---|---|---|
| Limite gratuito | 0,5 vCPU, 1GB RAM, 4GB disco (~1M vetores de 768 dim) | 500MB de banco (todo o projeto, não só vetores) | 2GB de storage, 2M write units/mês, 1M read units/mês, 5 índices |
| Cartão de crédito | Não exige | Não exige | Não exige |
| Inatividade | Cluster **suspende em 1 semana** e **deleta em 4 semanas** sem uso | Projeto **pausa em 7 dias** sem uso (não deleta, reativa manualmente) | Não há indicação de pausa por inatividade — mais estável para uso esporádico |
| Metadados/filtros | Excelente (payload filtering nativo, é o forte do Qdrant) | Bom (é SQL puro — `WHERE` combinado com `<->` do pgvector) | Bom (filtros de metadata, mas menos expressivo que SQL) |
| Nó nativo no n8n | Sim (Qdrant Vector Store) | Sim (Supabase Vector Store) | Sim (Pinecone Vector Store) |
| Compatível com Gemini Embeddings | Sim (model-agnostic, você traz o vetor pronto) | Sim | Sim |
| Curva de aprendizado | Baixa/média — conceito próprio de "collections/points" | Baixa se você já conhece SQL; ótimo para aprender RAG *e* SQL junto | Baixa — API bem simples, mas é uma "caixa preta" (menos didático para entender o que está acontecendo por baixo) |
| Bônus didático | Ensina um vector DB "de verdade" (dedicado) | Ensina Postgres, pgvector e (se quiser) Auth/Storage — mais transferível para outros projetos | Ensina o modelo "serverless gerenciado" mais popular do mercado |
| Região | Limitada, mas ok para uso pessoal | Múltiplas regiões | Restrito a `us-east-1` (AWS) no free tier |

**Não recomendo** o "Simple Vector Store" nativo do n8n (in-memory) para a base de conhecimento — ele não persiste de forma confiável entre reinicializações da instância n8n, servindo bem só para testes rápidos durante o desenvolvimento, não como base de conhecimento definitiva.

---

## 6. Opções de banco operacional

Isso guarda usuários, anamnese, planos gerados, feedback, cálculos, logs — não vetores.

**Minha recomendação de princípio:** se você escolher **Supabase** como banco de conhecimento (seção 5), reaproveite o **mesmo projeto Postgres** para as tabelas operacionais (schemas/tabelas separadas, mesmo banco). Isso reduz peças móveis, mantém tudo dentro do mesmo free tier e do mesmo comportamento de pausa — arquitetura mais simples de manter e mais didática para SQL relacional. Se você escolher **Qdrant**, ele não serve para dados relacionais, então você precisa de um segundo lugar:

| Opção | Prós | Contras |
|---|---|---|
| **Mesmo Postgres do Supabase** (se Supabase for o vector DB) | Um único free tier, transações reais, fácil fazer JOIN entre anamnese/planos/feedback | Amarra tudo ao mesmo limite de 500MB e à mesma janela de pausa |
| **Neon (Postgres serverless free tier)**, separado do vector DB | Free tier generoso, "cold start" rápido, ainda é SQL de verdade — bom se o vector DB for Qdrant/Pinecone | Mais uma conta/credencial para gerenciar |
| **Google Sheets** (nó nativo do n8n) | Zero configuração, visual, ótimo para os primeiros testes e para você "ver" os dados sem abrir um painel SQL | Sem integridade referencial real, não escala bem, pouco profissional para o resultado final |
| **Airtable (free tier)** | Interface amigável para revisar anamneses/planos manualmente enquanto aprende | Limite de linhas por base costuma ser baixo — verifique o limite atual antes de depender dele para histórico de longo prazo |

---

## 7. Opções de chatbot

| Critério | **Telegram** | **n8n Chat Trigger (webchat embutido)** | **Discord** | **WhatsApp Business Cloud API** |
|---|---|---|---|---|
| Custo | Grátis | Grátis (é um nó nativo do n8n) | Grátis | "Grátis" só para conversas iniciadas pelo usuário dentro da janela de 24h; requer verificação de Business Manager na Meta |
| Fricção de setup | Muito baixa (BotFather, 2 minutos) | Zero — já vem no n8n | Baixa/média (criar bot + servidor) | Alta (verificação de negócio, templates aprovados) |
| Webhooks nativos no n8n | Sim (Telegram Trigger) | Sim, é a própria origem | Sim (Discord Trigger) | Sim, mas com mais peças (Meta Developer App) |
| Persistência de `chat_id`/sessão para memória de conversa | Excelente, natural | Precisa gerenciar `sessionId` manualmente | Bom | Bom |
| Experiência do usuário para um app de saúde 1:1 | Boa — conversa privada, familiar | Boa para testes, mas não "sai" do navegador do n8n | Pensado para comunidades/servidores, menos natural para 1:1 | Ótima (é o app mais usado no Brasil) mas com maior custo de implementação |
| Adequação ao seu objetivo de **aprender n8n** | Ótima — é o "hello world" de chatbots no n8n, sem distrações | Ótima para os primeiros testes rápidos do fluxo | Boa, mas ensina menos sobre integrações externas do que Telegram | Ensina bastante sobre integrações de API "do mundo real", mas o retorno de aprendizado por hora investida é menor no início |

**Recomendação:** use o **n8n Chat Trigger** para desenvolvimento/depuração rápida (sem sair da tela do n8n) e o **Telegram** como canal "de verdade" para testar a experiência completa. WhatsApp fica como evolução futura, não como ponto de partida — o custo de aprendizado ali é sobre burocracia da Meta, não sobre n8n.

---

## 8. Opções de arquitetura completa

### Arquitetura A — "Simples e integrada" (Supabase-cêntrica)

```
Telegram (ou n8n Chat) → n8n Cloud
   → Gemini 3.1 Flash-Lite (anamnese/extração de slots)
   → [validação de completude] → Supabase Postgres (tabelas operacionais)
   → Gemini Embedding 001 (query) → Supabase pgvector (RAG, mesmo projeto)
   → Code node (cálculos determinísticos, fórmulas do .md)
   → Gemini 3 Flash (geração do plano + justificativa)
   → Code/LLM node (validação automática do plano — seção 15)
   → Supabase (persistência do plano + trace de fontes/cálculos)
   → resposta ao usuário
```

| | |
|---|---|
| **Prós** | Um único free tier a monitorar; SQL é transferível para outros projetos; filtragem por metadados fácil com `WHERE`; nó nativo robusto no n8n; fácil de depurar (dá para abrir o banco e ver tudo em SQL). |
| **Contras** | Pausa após 7 dias de inatividade (mitigável com Cron de keep-alive); 500MB pode apertar se o corpus de PDFs crescer bastante; RAG e dados operacionais competem pelo mesmo limite de armazenamento. |
| **Custo** | US$0 dentro dos limites. Risco real de custo: só o trial do n8n Cloud expirando (seção 2.1). |
| **Complexidade** | Baixa-média. |
| **Adequação a aprendizado** | Alta — você aprende RAG, SQL e Postgres ao mesmo tempo, com poucas peças. |

### Arquitetura B — "Separação de responsabilidades" (Qdrant + Neon)

```
Telegram → n8n Cloud
   → Gemini 3.1 Flash-Lite (anamnese)
   → Neon Postgres (tabelas operacionais)
   → Gemini Embedding 001 (query) → Qdrant Cloud (RAG, só vetores + payload)
   → Code node (cálculos)
   → Gemini 3 Flash (geração)
   → validação automática
   → Neon (persistência)
   → resposta
```

| | |
|---|---|
| **Prós** | Qdrant é feito especificamente para busca vetorial (filtros de payload muito ricos, performance); separação clara de responsabilidades — arquitetura mais "de livro-texto"; Neon costuma ter cold start rápido. |
| **Contras** | Duas contas/free tiers para gerenciar; o cluster Qdrant grátis **deleta** após 4 semanas de inatividade (pior que a pausa reversível do Supabase) — exige keep-alive mais disciplinado. |
| **Custo** | US$0 dentro dos limites, mesmo risco do trial do n8n Cloud. |
| **Complexidade** | Média — mais integrações para configurar e monitorar. |
| **Adequação a aprendizado** | Alta, especialmente se seu interesse for entender vector DBs dedicados de verdade (útil para currículo/portfólio), não só "Postgres com uma extensão". |

### Arquitetura C (bônus) — "Self-hosted desde o início"

```
n8n self-hosted (Community Edition, grátis para sempre, em VPS gratuita
  ex. Oracle Cloud Always Free / Fly.io / Docker local)
   → mesmo fluxo de A ou B, com Postgres+pgvector rodando no mesmo servidor (ou Neon)
```

| | |
|---|---|
| **Prós** | Resolve na raiz o conflito da seção 2.1 — 100% gratuito **permanentemente**, sem pausas por inatividade, controle total. Ótimo também para aprender Docker/DevOps. |
| **Contras** | Não é literalmente "n8n Cloud"; exige gerenciar servidor, atualizações, SSL; mais fricção inicial que tira foco de aprender os *fluxos* de IA em si. |
| **Custo** | US$0, mas com risco de cobrança inesperada se você sair dos limites do "Always Free" do provedor de nuvem escolhido — leia a letra miúda de cada provedor antes. |
| **Complexidade** | Média-alta (infra) mas baixa (lógica dos workflows, que é idêntica). |
| **Adequação a aprendizado do n8n Cloud especificamente** | Baixa — você aprende n8n, mas não a experiência gerenciada "Cloud". |

---

## 9. Modelos Gemini — verificado na documentação atual

Confirmei na documentação oficial da Google (ai.google.dev, DeepMind model cards e o blog do Gemini) que os quatro recursos que você tem são reais e correspondem a:

| Nome que você usou | ID técnico da API | Situação confirmada |
|---|---|---|
| Gemini Embedding 1 | `gemini-embedding-001` | GA (disponível geralmente), texto, 3072 dimensões por padrão, truncável (MRL) para 768/1536 sem perda relevante de qualidade. |
| Gemini Embedding 2 | `gemini-embedding-2` | Primeiro embedding **multimodal** da Gemini API (texto, imagem, vídeo, áudio, documentos) — espaço vetorial **incompatível** com o `-001` (não misture os dois no mesmo índice). |
| Gemini 3 Flash | `gemini-3-flash` (nome de referência) | Existe, é o modelo intermediário da família Gemini 3. |
| Gemini 3.1 Flash-Lite | `gemini-3.1-flash-lite` | GA, otimizado para custo/latência, com "níveis de pensamento" configuráveis (minimal/low/medium/high) para equilibrar custo x qualidade. |

**Atenção:** desde o lançamento desses modelos, a Google já lançou gerações mais novas (Gemini 3.5 Flash, 3.5 Flash-Lite e 3.6 Flash), e o Gemini 2.5 Flash está com desligamento programado para outubro de 2026. Isso não invalida sua arquitetura — só significa duas coisas práticas: (1) confira no seletor de modelos do Google AI Studio se suas chaves também têm acesso às versões mais novas (pode valer a troca, sem custo adicional de aprendizado), e (2) **centralize o nome do modelo em um nó de configuração/variável no n8n**, para trocar de modelo no futuro sem reescrever o workflow inteiro.

### Distribuição de funções recomendada

| Função | Modelo sugerido | Por quê |
|---|---|---|
| Indexação da base de conhecimento + embedding da query do usuário | `gemini-embedding-001`, truncado para 768 dims | Modelo estável, só texto (seus PDFs são majoritariamente texto), mais barato, mais espaço nos free tiers de vector DB. Reserve o Embedding 2 para o dia em que quiser indexar imagens/tabelas diretamente. |
| Anamnese conversacional (extrair idade/peso/objetivo etc. da fala livre do usuário), classificações leves (ex.: "usuário aprovou o plano? sim/não/quer alterar"), checagem de grounding da resposta final | `gemini-3.1-flash-lite` | É a chamada que mais se repete por conversa — barata, rápida, com RPD mais generoso. Use `thinking_level` baixo aqui, já que é extração/classificação, não raciocínio complexo. |
| Geração do plano final (sintetizar dados + fórmulas + N documentos + justificativa coerente) e a etapa de auto-crítica/validação do plano | `gemini-3-flash` | É a etapa que exige mais capacidade de raciocínio e síntese — vale usar o modelo mais forte disponível aqui, mesmo sendo mais caro por chamada, porque é chamado poucas vezes por conversa (só quando há informação suficiente para gerar/regenerar o plano). |

Sobre limites gratuitos: as fontes que encontrei divergem bastante (mudanças de cota aconteceram em dezembro/2025, e os números variam por relatório), então **não construa a arquitetura em cima de um número fixo específico** — confira o painel de rate limits do seu próprio projeto no Google AI Studio antes de desenhar qualquer lógica de retry/fallback, e trate os limites de RPM/RPD/TPM como algo a monitorar, não a assumir.

---

## 10. Workflow do n8n (conceitual, por nós)

| Etapa | Nó (tipo) | Função | Entrada | Saída |
|---|---|---|---|---|
| 1 | **Telegram Trigger** / **Chat Trigger** | Recebe mensagem do usuário | Mensagem + `chat_id`/`sessionId` | Item de entrada do workflow |
| 2 | **Postgres/Supabase (Get)** | Recupera perfil parcial já coletado nesta sessão | `sessionId` | Objeto estruturado do usuário (parcial) |
| 3 | **AI Agent + Simple/Postgres Memory + Gemini 3.1 Flash-Lite (Chat Model)** | Conduz a anamnese: extrai novos campos da mensagem, atualiza o objeto estruturado | Mensagem + perfil parcial | Perfil atualizado |
| 4 | **IF / Switch** | Valida se as 5 obrigatórias estão completas | Perfil atualizado | Ramo "continuar coletando" ou "prosseguir" |
| 4a | (se incompleto) **Set + resposta ao usuário** | Pergunta o que falta, sem repetir o que já foi dado | Campos faltantes | Mensagem de volta ao chat |
| 5 | **Switch (camada de segurança)** | Detecta sinalizadores de risco (menor de idade, gravidez, condição médica grave, sinais de transtorno alimentar) — seção 14 | Perfil completo | Ramo normal ou ramo de encaminhamento |
| 5a | (se risco) **Set + resposta de encaminhamento** | Explica, sem diagnosticar, e orienta buscar profissional | Sinalizador | Mensagem + encerramento do fluxo de geração |
| 6 | **Code node** | Calcula TMB/GET/macros a partir do arquivo de fórmulas (determinístico) | Perfil completo | Objeto de cálculos com `formula_id`, inputs e resultado |
| 7 | **Embeddings Google Gemini** | Gera embedding da(s) query(ies) de recuperação | Objetivo + contexto do perfil | Vetor(es) |
| 8 | **Vector Store (Qdrant/Supabase) — modo Query** | Busca semântica + filtro de metadados | Vetor + filtros | Top-k chunks com metadados |
| 9 | **Filter node** | Corte por limiar de similaridade | Chunks + score | Chunks aprovados |
| 10 | **Gemini 3 Flash (Chat Model / Basic LLM Chain)** | Gera o plano com justificativa, exigindo saída estruturada com fontes | Perfil + cálculos + chunks aprovados | Plano estruturado (JSON) |
| 11 | **Code node + Gemini 3.1 Flash-Lite** | Validação automática (seção 15): confere fontes citadas, bate números com a etapa 6, checa restrições e conflitos dieta/treino | Plano estruturado | Aprovado / reprovado com motivo |
| 11a | (se reprovado) volta para o passo 10 com feedback, ou escala para "faltam informações" | | | |
| 12 | **Postgres/Supabase (Insert)** | Persiste plano, versão, cálculos usados e fontes citadas | Plano aprovado | Registro salvo |
| 13 | **Telegram / respond to webhook** | Envia o plano ao usuário | Plano formatado | Mensagem final |
| 14 | **AI Agent (aguarda resposta)** | Coleta validação do usuário (satisfeito / quer mudar) | Resposta do usuário | Ramo "encerrar" ou "ajustar" |
| 14a | (se ajustar) **retorna à etapa 3** com o pedido de mudança como novo input, reaproveitando o perfil já coletado | | | |

---

## 11. Personalização — estrutura de dados

Proponho um objeto único, persistido e reutilizado em cada etapa (evita que informações se percam entre chamadas):

```
PerfilUsuario {
  perfil: { idade, sexo, altura_cm, peso_kg, percentual_gordura?, medidas? }
  objetivos: { principal, secundarios[], prazo_desejado?, prioridade }
  saude: { lesoes[], dores[], limitacoes[], condicoes_medicas[], medicamentos[], flags_seguranca[] }
  treino: { experiencia, dias_semana, tempo_sessao_min, local, equipamentos[],
            exercicios_preferidos[], exercicios_evitar[], historico }
  nutricao: { preferencias[], alergias[], intolerancias[], restricoes[], padrao_alimentar,
              refeicoes_dia, horarios[], capacidade_cozinhar, refeicoes_fora, orcamento }
  rotina: { trabalho, sono_horario, sono_qualidade, nivel_atividade_diaria,
            janelas_treino[], janelas_refeicao[] }
  adesao: { disciplina_autopercebida, historico_aderencia, dificuldades_anteriores[],
            flexibilidade_desejada }
  historico: { conversas_anteriores[], planos_anteriores[], feedback_anterior[] }
  meta: { completude_pct, obrigatorias_ok: bool, campos_seguranca_pendentes[] }
}
```

Esse objeto é o que alimenta a query de recuperação (seção 4), os cálculos (seção 12) e o prompt de geração — garantindo que **a mesma fonte de verdade** seja usada em todas as etapas, em vez de cada nó reconstruir o contexto do zero a partir do histórico de chat bruto (o que aumenta risco de inconsistência e de "esquecer" uma restrição já informada).

### Rastreabilidade (dado → cálculo → evidência → decisão)

Cada recomendação no plano final deve carregar sua própria "ficha de auditoria":

```
Recomendacao {
  decisao: "..."
  dados_usuario_usados: [...]
  calculo_usado: { formula_id, inputs, resultado }
  fontes: [ { doc_id, titulo, ano, nivel_evidencia } ]
  raciocinio: "..."
}
```

Isso serve tanto para você auditar o comportamento do agente durante o aprendizado quanto para alimentar a etapa de validação automática (seção 15) e a explicação ao usuário em caso de desacordo (seção 9 do seu documento original).

---

## 12. Cálculos — onde devem acontecer

**Recomendação direta: nó de código (Code node, JavaScript) no n8n, nunca o LLM.**

| Opção | Confiabilidade | Custo | Auditabilidade |
|---|---|---|---|
| LLM calcula "de cabeça" | Baixa — LLMs erram aritmética mesmo com a fórmula correta no contexto, e o erro não é determinístico (pode variar entre chamadas idênticas) | Consome tokens | Difícil — não dá para testar unitariamente |
| **Code node no n8n** | **Alta — determinístico, testável, mesmo resultado sempre para a mesma entrada** | **Zero custo de token** | **Alta — você pode escrever testes e revisar o código versionado** |
| Função no banco (ex. função SQL no Postgres) | Alta também, mas menos didático para quem quer focar em n8n, e mistura lógica de negócio dentro do banco | Zero | Boa, mas menos visível no fluxo do n8n |

O arquivo `.md` de fórmulas funciona como **especificação versionada e legível** (fonte de verdade documental, revisável por você), e o Code node implementa exatamente essas fórmulas como funções puras. O LLM entra depois, apenas para **narrar e contextualizar** o resultado numérico já calculado — nunca para produzi-lo. Essa separação também facilita a validação automática (seção 15): comparar "número que o LLM escreveu no plano" com "número que o Code node calculou" é uma checagem trivial e 100% confiável.

---

## 13. Estimativa de eficácia e adesão — metodologia defensável

Você pediu uma avaliação crítica, então aqui vai: **um prazo pontual e preciso ("6,3 meses") não é defensável cientificamente** para a maioria dos objetivos de composição corporal — a variação individual é grande demais. A abordagem tecnicamente honesta é:

1. **Separar o que é matemática determinística do que é literatura.** A relação entre déficit/superávit calórico e mudança de peso ao longo do tempo é aritmética (Code node, seção 12) — isso pode ser calculado com segurança a partir do próprio déficit calculado para o usuário. Já a *faixa saudável* de variação de peso por semana, ou a taxa de ganho de massa magra esperada por nível de treino, **só deve ser usada se estiver explicitamente presente nos documentos da base de conhecimento** — nunca como número "conhecido" do LLM.
2. **Apresentar cenários, não uma data.** Alta adesão / adesão moderada / baixa adesão, cada um como uma **faixa** (ex. "entre X e Y semanas"), não um valor único — exatamente como você sugeriu.
3. **Expressar incerteza explicitamente**, no próprio texto entregue ao usuário (ex.: "esta é uma estimativa baseada no seu déficit calórico e em [N] documentos da base; resultados individuais variam e não é uma promessa de resultado").
4. **Se a base de conhecimento não tiver literatura suficiente sobre a taxa esperada para aquele objetivo específico** (ex. um objetivo muito nichado), **o agente deve dizer isso** e apresentar só a projeção matemática pura (deficit → peso), sem uma camada de "tempo para o objetivo" que dependeria de dados que não existem na base.

**Implementação no n8n:** o Code node da etapa de cálculo já produz a projeção determinística (semanas por cenário de aderência, com base no déficit/superávit). Uma chamada de LLM (Flash) então busca, na base recuperada, faixas de literatura sobre variáveis não puramente calóricas (ex. limites de segurança de perda de peso por semana, taxa de síntese proteica) — se encontrar, ajusta a narrativa; se não encontrar, o agente declara a lacuna em vez de inventar. Isso mantém a separação "matemática confiável" vs. "afirmação que precisa de fonte" que sustenta toda a regra de não-alucinação do projeto.

---

## 14. Segurança

Como você pediu, isso precisa ser **parte da arquitetura** (um branch estrutural no workflow), não só um aviso de rodapé.

**Onde entra no fluxo:** logo após a anamnese estar completa e **antes** de qualquer chamada de RAG/cálculo/geração (etapa 5 da seção 10) — um nó de classificação (Flash-Lite, barato, roda em toda conversa) verifica sinalizadores:

- Sinais de transtorno alimentar (padrões de fala sobre restrição extrema, purgação, medo intenso de ganhar peso) → **não gerar plano de dieta**; resposta empática, sem diagnosticar, orientando buscar um profissional (nutricionista/psicólogo/médico);
- Idade informada menor de 18 anos → escopo reduzido: sem déficits agressivos, foco em orientações gerais, recomendação explícita de acompanhamento de responsável + profissional. Para um projeto pessoal de estudo, considere seriamente **não atender menores** — não há uma rede de segurança clínica real por trás do sistema;
- Gravidez/amamentação → não gerar plano de déficit calórico ou treino de alta intensidade sem qualificação; encaminhar;
- Condições médicas relevantes (diabetes, cardiopatias, doença renal) mencionadas → o plano pode ser gerado, mas com aviso destacado e recomendação de validação médica antes de iniciar;
- Medicamentos que interagem com exercício/dieta (ex. insulina, anticoagulantes) → mesma lógica de aviso destacado;
- Metas potencialmente perigosas (déficit calórico extremo, volume de treino excessivo pedido explicitamente pelo usuário) → o agente não deve simplesmente atender; deve explicar o risco com base na literatura disponível e propor uma alternativa dentro de faixas seguras.

**Quando interromper completamente a geração:** transtorno alimentar ativo, gravidez de risco, ou recusa do usuário em fornecer as informações de segurança mínimas quando há indício prévio de risco levantado por ele mesmo na conversa.

**Como não tornar o chatbot inutilizável:** os avisos devem ser **contextuais e proporcionais** ao risco identificado — não um bloco de disclaimer genérico repetido em toda mensagem. A maioria dos usuários (sem sinalizadores) nunca vê esses avisos além de um lembrete padrão de "isto não substitui acompanhamento profissional" ao final do plano.

---

## 15. Validação — controle de qualidade automático antes de entregar

Uma segunda etapa (nó separado, depois da geração do plano, antes de persistir/enviar) checando, de forma combinada (regras determinísticas + LLM como juiz):

| Checagem | Como |
|---|---|
| Toda afirmação nutricional/de treino tem fonte válida | Determinístico: comparar os `doc_id` citados no plano com os `doc_id` realmente presentes no contexto recuperado naquela chamada |
| Números do plano batem com os cálculos | Determinístico: comparar o valor no texto gerado com a saída do Code node (mesma fonte de verdade) |
| Restrições do usuário foram respeitadas (alergias, lesões) | Híbrido: checagem de palavra-chave determinística + checagem semântica por LLM (Flash-Lite) para casos não literais |
| Consistência entre dieta e treino | LLM (Flash-Lite): ex. sinalizar déficit muito agressivo combinado com treino de volume muito alto sem justificativa |
| Nenhuma recomendação contradiz os sinalizadores de segurança da etapa 14 | Determinístico + LLM |

Se qualquer checagem falhar: **o plano não é enviado**. Ele volta para regeneração com o motivo específico da falha como feedback (ex. "a recomendação X não tem fonte no contexto recuperado — remova ou busque mais contexto"), ou é escalado como "faltam informações para decidir com segurança" quando a falha for de fundo (ex. contexto insuficiente), não de formatação.

---

## 16. Recomendação final

Para o seu objetivo declarado — **aprender n8n Cloud a fundo, com ferramentas gratuitas** — recomendo:

1. **Comece pela Arquitetura A (Supabase-cêntrica, seção 8)** durante os 14 dias de trial do n8n Cloud. Menos peças móveis = você aprende os conceitos de RAG, agentes e cálculo determinístico mais rápido, sem se perder gerenciando múltiplas contas de infraestrutura.
2. **Banco de conhecimento e operacional: Supabase (Postgres + pgvector), mesmo projeto.** SQL é uma habilidade mais transferível do que a API específica de um vector DB, e você reduz o número de free tiers para monitorar a um só.
3. **Chatbot: n8n Chat Trigger para desenvolvimento + Telegram para a experiência real.** WhatsApp fica para depois, se ainda fizer sentido.
4. **Modelos:** `gemini-embedding-001` (768 dims) para indexação/busca; `gemini-3.1-flash-lite` para anamnese e checagens leves; `gemini-3-flash` para geração do plano e validação de grounding.
5. **Decisão consciente sobre o n8n Cloud (seção 2.1):** use o trial para aprender a experiência gerenciada oficial; ao final dele, migre o mesmo workflow (exportável em JSON) para **n8n self-hosted gratuito** (Docker local ou uma VPS com free tier permanente) se quiser continuar sem custo indefinidamente. Isso maximiza o aprendizado real do n8n Cloud dentro do orçamento que você definiu, em vez de descobrir a cobrança no dia 15.
6. **Configure um Cron de keep-alive** (uma consulta trivial a cada 4–5 dias) para evitar que o Supabase pause por inatividade entre suas sessões de estudo.

Por que não a Arquitetura B (Qdrant) como ponto de partida: ela é tecnicamente mais "correta" para um sistema de produção real, mas o comportamento de **deleção** (não só pausa) do free tier do Qdrant após 4 semanas de inatividade é um risco desnecessário para um projeto de estudo com ritmo irregular. Vale revisitar Qdrant depois, como uma segunda iteração, quando quiser comparar vector DBs dedicados de verdade — inclusive é uma boa forma de aprofundar o aprendizado uma vez que a base (Arquitetura A) já estiver funcionando.

---

## 17. Roadmap incremental

| Fase | Objetivo | Entrega |
|---|---|---|
| **0 — Setup** | Contas e credenciais | Chaves Gemini testadas no AI Studio; projeto Supabase criado; bot Telegram criado (BotFather); 3–5 PDFs de teste + 1 `.md` de fórmulas pequeno |
| **1 — "Eco"** | Validar o cano de mensagens | Telegram/Chat Trigger → Gemini Flash-Lite → resposta simples, sem RAG, sem cálculo |
| **2 — Anamnese** | Coleta estruturada | Extração das 5 obrigatórias + memória de conversa (Simple/Postgres Memory) + validação de completude (etapa 4 da seção 10) |
| **3 — Cálculos** | Motor determinístico | Code node com TMB/GET/macros a partir do arquivo de fórmulas, testado isoladamente |
| **4 — RAG básico** | Indexação e recuperação | Pipeline de ingestão dos PDFs + busca semântica com filtro por metadados + geração fundamentada simples |
| **5 — Personalização completa** | Objeto de perfil rico | Perguntas recomendadas/opcionais da seção 3 + estrutura completa da seção 11 |
| **6 — Explicabilidade** | Rastreabilidade | Estrutura dado→cálculo→evidência→decisão (seção 11) anexada a cada recomendação |
| **7 — Eficácia e adesão** | Cenários múltiplos | Metodologia da seção 13, com projeção determinística + narrativa fundamentada |
| **8 — Validação automática** | QA antes de entregar | Camada da seção 15, com reprovação e regeneração |
| **9 — Segurança** | Camada estrutural | Classificação de risco + branch de encaminhamento (seção 14) |
| **10 — Ciclo completo** | Persistência e replanejamento | Feedback do usuário → ajuste → nova versão do plano, com histórico versionado |
| **11 — Polimento** | Robustez de longo prazo | Cron de keep-alive, tratamento de erro 429, monitoramento de custo/limites, decisão final sobre migrar para self-hosted |

Cada fase é um workflow funcional e testável por si só — você não precisa (nem deveria) tentar construir os 17 pontos do documento original de uma vez.

---

## Referências consultadas (verifique periodicamente — estes números mudam com frequência)

- Modelos e embeddings Gemini: ai.google.dev/gemini-api/docs/embeddings · ai.google.dev/gemini-api/docs/models/gemini-3.1-flash-lite
- Qdrant Cloud (free tier): qdrant.tech/pricing/ · qdrant.tech/cloud/
- Nós n8n: docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreqdrant · .../vectorstoresupabase · docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.embeddingsgooglegemini · .../lmchatgooglegemini
- WhatsApp Business Cloud + n8n: n8n.io/integrations/whatsapp-business-cloud/
- Preços/limites de n8n Cloud, Supabase e Pinecone: consulte diretamente n8n.io/pricing, supabase.com/pricing e pinecone.io/pricing antes de decidir, pois os valores exatos mudam ao longo do ano.
