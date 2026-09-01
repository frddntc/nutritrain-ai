# Plano Consolidado — Agente de IA para Dieta e Treino (n8n Cloud + Gemini)

> Versão 2 — consolida as decisões tomadas em cima do brainstorm inicial. Este documento descreve a arquitetura **escolhida**, não mais um leque de opções. Ainda é planejamento — nenhum workflow foi implementado.

---

## 1. Resumo do projeto (atualizado)

Agente conversacional no n8n Cloud que conduz anamnese, calcula parâmetros de forma determinística e gera planos de dieta e treino individualizados, sempre explicando o raciocínio.

**Mudança-chave em relação ao brainstorm original:** o agente **nunca se recusa a responder**. Toda recomendação segue uma hierarquia de três níveis de fonte, sempre declarada explicitamente ao usuário:

1. **Base de conhecimento interna** (PDFs indexados) — nível preferencial, citado com documento/trecho;
2. **Busca no Google** (ferramenta nativa de grounding da própria API Gemini), quando a base interna não cobrir o ponto — citada como "informação obtida por busca na web, não verificada pela base de conhecimento";
3. **Conhecimento geral do modelo** (pré-treinamento), quando nem a base nem a busca resolverem — citada como "conhecimento geral do modelo, não verificado por fonte específica".

Cálculos numéricos continuam **sempre** determinísticos (nó de código), nunca gerados por LLM, independentemente da fonte usada para o texto ao redor.

---

## 2. Decisões de arquitetura assumidas

| Ponto | Decisão |
|---|---|
| **n8n** | Trial de 14 dias do n8n Cloud, usado de forma intensiva para aprender a experiência gerenciada de verdade. Ao final, exportar o workflow em JSON e migrar para n8n self-hosted gratuito (Docker local ou VPS free tier), reaproveitando 100% da lógica construída. |
| **Regra "não inventar"** | Mantém os mecanismos anti-alucinação (saída estruturada com fonte, limiar de similaridade, verificação de grounding, cálculo sempre em código) — mas o efeito de "informação insuficiente" deixou de ser recusa e passou a ser **degradação controlada e rotulada** pela hierarquia de fontes da seção 1. |
| **Free tiers pausando por inatividade** | Sem tratamento especial. O objetivo é uma execução completa e bem-sucedida do sistema, não operação contínua de longo prazo — se o Supabase pausar entre sessões de trabalho, é só reativar manualmente pelo painel antes de continuar (leva segundos). |
| **Limites da API Gemini** | Nome do modelo isolado em um nó de configuração central no n8n (um `Set`/variável no início do workflow), para trocar de modelo em um único lugar se necessário. |
| **LGPD / dados de saúde** | Medidas leves e proporcionais a uso pessoal: aviso simples de consentimento no primeiro contato explicando que dados de saúde serão processados; não expor chaves/dados em logs; sem necessidade de aparato formal de compliance (DPO, política de privacidade publicada etc.), já que não há usuários terceiros. |
| **Canal de chat** | Telegram, confirmado. |
| **Escopo** | MVP simples primeiro — ver roadmap (seção 17), que prioriza um ciclo completo mínimo antes de qualquer camada adicional. |
| **Modelos e cotas** | Ver tabela completa na seção 9. |

---

## 3. Informações coletadas pelo agente

Mantidas exatamente como definido: as cinco obrigatórias continuam bloqueando a geração do plano; o restante entra como recomendado/opcional, com o mesmo tratamento diferenciado para os campos de segurança (ausência não bloqueia, mas força ressalva explícita e abordagem mais conservadora no plano).

- **Obrigatórias:** idade, sexo/gênero, altura, peso, objetivo.
- **Recomendadas — precisão:** nível de atividade diária, experiência de treino, dias/tempo disponíveis, local/equipamentos, percentual de gordura (se souber), padrão alimentar, número de refeições.
- **Recomendadas — segurança:** lesões, dores, condições médicas, medicamentos, alergias/intolerâncias, gravidez/amamentação.
- **Opcionais:** histórico de peso/medidas, objetivos secundários e prazo, preferências de exercício, sono, rotina, capacidade de cozinhar/orçamento, e as perguntas de adesão (disciplina, dificuldades anteriores, flexibilidade desejada).

---

## 4. Arquitetura RAG (com hierarquia de fontes)

Pipeline de ingestão e recuperação mantidos como especificado (Google Drive → extração → chunking com metadados → embeddings `gemini-embedding-001` truncados para 768 dims → Supabase pgvector com filtro por metadados → corte por limiar de similaridade). O que muda é o que acontece **quando a recuperação não é suficiente**:

1. O agente monta a query a partir do objeto estruturado do usuário e busca na base interna;
2. Se os chunks recuperados passam no limiar de similaridade → geração fundamentada na base, com citação de documento/trecho (nível 1);
3. Se os chunks ficam abaixo do limiar (ou a busca não retorna nada relevante para aquele ponto específico) → o próprio nó de geração (Gemini 3.7 Flash) tem a ferramenta de **Grounding with Google Search** habilitada e é instruído a usá-la nesse caso, citando a origem como busca na web (nível 2);
4. Se mesmo a busca não resolver (ou o modelo decidir que não há necessidade de buscar, por já ser conhecimento amplamente estabelecido) → resposta com conhecimento geral do modelo, explicitamente rotulada como não verificada por fonte específica (nível 3).

Essa decisão de qual nível usar acontece **dentro do próprio prompt de geração**, via instrução de sistema clara sobre a ordem de prioridade e a obrigatoriedade de rotular a fonte de cada recomendação — não é mais um branch condicional separado no n8n, o que simplifica o workflow.

> Nota técnica: a Grounding with Google Search é gratuita até 5.000 prompts/mês na família Gemini 3.x (depois disso, passa a ser paga) — folga enorme para o volume de um projeto pessoal. Se o nó nativo "Google Gemini Chat Model" do n8n não expuser essa ferramenta diretamente na interface, o fallback é uma chamada HTTP customizada à API Gemini com o parâmetro `tools: [{ google_search: {} }]` — isso é um detalhe a confirmar no momento da implementação.

---

## 5. Banco de conhecimento

**Supabase (Postgres + pgvector), free tier.** Um projeto único, schema dedicado para os chunks vetorizados e seus metadados (título, ano, tema, população, objetivo, nível de evidência, status ativo/desatualizado).

## 6. Banco operacional

**Mesmo projeto Postgres do Supabase**, schemas/tabelas separadas das de RAG: usuários, perfil estruturado, histórico de conversas, planos gerados (com versionamento), feedback, registros de cálculo e trilha de auditoria (fonte→cálculo→decisão).

## 7. Chatbot

**Telegram**, via Telegram Trigger + Telegram node nativos do n8n. Desenvolvimento/depuração usando o Chat Trigger embutido do n8n antes de validar no Telegram real.

---

## 8. Arquitetura completa final

```
Telegram → n8n Cloud (trial → depois self-hosted)
   → Gemini 3.5 Flash-Lite (anamnese, extração de slots, classificação de risco)
   → [validação de completude das 5 obrigatórias]
   → Supabase Postgres (perfil estruturado)
   → Code node (cálculos determinísticos — TMB/GET/macros, a partir do .md de fórmulas)
   → Gemini Embedding 001 (embedding da query) → Supabase pgvector (busca semântica + filtro de metadados)
   → Gemini 3.7 Flash, com Grounding with Google Search habilitado
        (geração do plano com hierarquia de fontes: base interna → busca web → conhecimento geral,
         sempre com rótulo de fonte por recomendação)
   → Gemini 3.5 Flash-Lite (validação automática: fontes rotuladas, números batem com o Code node,
        restrições respeitadas, dieta/treino consistentes, sem contradição com flags de segurança)
   → Supabase (persistência do plano + trilha de auditoria)
   → Telegram (entrega ao usuário)
   → Gemini 3.5 Flash-Lite (coleta validação do usuário) → encerra ou realimenta o ciclo
```

---

## 9. Modelos Gemini — alocação final e cotas confirmadas

Com base no que você tem acesso hoje:

| Modelo | Cota diária informada | Função no sistema |
|---|---|---|
| `gemini-embedding-001` | ~1.500 req/dia, 10M TPM (free tier padrão) | Indexação da base + embedding de cada query de busca. Truncado para 768 dimensões. |
| `gemini-3.5-flash-lite` | **500/dia** | Anamnese conversacional, extração de slots, classificação de risco (segurança), validação automática/checagem de grounding do plano final. É o modelo que carrega o volume de chamadas por conversa. |
| `gemini-3.7-flash` | **20/dia** | Geração do plano final (dieta + treino + justificativa), com a ferramenta de Grounding with Google Search habilitada. Reservado deliberadamente para esta única etapa por conversa. |
| `gemini-3.6-flash` | 20/dia (cota própria) | Fallback de contingência caso a cota do 3.7 Flash se esgote no dia de testes — mesma função, qualidade ligeiramente inferior mas ainda robusta. |
| `gemini-3.5-flash` | 20/dia (cota própria) | Segundo fallback, mesma lógica do 3.6 Flash. |
| `gemini-3-flash` / `gemini-3.1-flash-lite` | não utilizados na alocação principal | Mantidos disponíveis nas suas chaves, mas superados pelas versões mais novas acima — ficam como opção de emergência se todas as outras cotas se esgotarem no mesmo dia. |

**Sobre sua pergunta de teste:** como só a etapa de geração (passo destacado acima) consome a cota de 20/dia do 3.7 Flash, um ciclo completo de teste — anamnese → cálculo → RAG → **1 geração** → validação → entrega — custa **1 chamada** dessa cota (2, se você quiser testar também o caminho de regeneração após reprovação na validação). Isso dá margem para rodar o fluxo completo repetidas vezes no mesmo dia. Se durante o ajuste fino do prompt de geração você precisar de mais tentativas do que isso em um único dia, use a cota do 3.6 Flash e depois a do 3.5 Flash como extensão — na prática, até ~60 execuções de geração por dia somando os três, se necessário.

---

## 10. Workflow do n8n (por nós, atualizado)

| Etapa | Nó | Função | Mudança em relação ao brainstorm |
|---|---|---|---|
| 1 | Telegram Trigger / Chat Trigger | Recebe mensagem | — |
| 2 | Postgres (Get) | Recupera perfil parcial da sessão | — |
| 3 | AI Agent + Memory + `gemini-3.5-flash-lite` | Anamnese: extrai/atualiza campos | — |
| 4 | IF/Switch | Valida as 5 obrigatórias | — |
| 4a | Set + resposta | Pergunta o que falta | — |
| 5 | `gemini-3.5-flash-lite` (classificador) | Detecta sinalizadores de segurança | Continua igual — a camada de segurança **não** foi flexibilizada, só a regra de grounding |
| 5a | Set + resposta de encaminhamento | Caso de risco | — |
| 6 | Code node | Cálculos determinísticos (fórmulas do `.md`) | — |
| 7 | Embeddings Google Gemini (`gemini-embedding-001`) | Embedding da query | — |
| 8 | Supabase Vector Store (Query) | Busca semântica + filtro de metadados | — |
| 9 | Filter node | Corte por limiar de similaridade | Agora alimenta o prompt como "contexto disponível", não como condição de recusa |
| 10 | `gemini-3.7-flash` (+ Grounding with Google Search) | Gera o plano com hierarquia de fontes (KB → web → geral), rotulando cada recomendação | **Novo:** ferramenta de busca habilitada; prompt de sistema define a ordem de prioridade das fontes |
| 11 | `gemini-3.5-flash-lite` + Code node | Validação: toda recomendação tem rótulo de fonte (não precisa mais ser exclusivamente da KB); números batem com a etapa 6; restrições respeitadas; sem contradição com a etapa 5 | Critério de reprovação mudou de "sem fonte na KB" para "sem rótulo de fonte nenhum" |
| 11a | (se reprovado) volta ao passo 10 com o motivo específico | — | — |
| 12 | Postgres (Insert) | Persiste plano + trilha de auditoria | — |
| 13 | Telegram | Envia o plano | — |
| 14 | AI Agent (`gemini-3.5-flash-lite`) | Coleta validação do usuário | — |
| 14a | (se ajuste pedido) retorna ao passo 3 com o novo input | — | — |

---

## 11. Personalização — estrutura de dados

Mantida conforme especificado — o objeto único `PerfilUsuario` (perfil, objetivos, saúde, treino, nutrição, rotina, adesão, histórico, meta) segue sendo a fonte de verdade compartilhada entre anamnese, cálculo, recuperação e geração. A ficha de auditoria por recomendação (`decisao`, `dados_usuario_usados`, `calculo_usado`, `fontes`, `raciocinio`) agora inclui explicitamente o **nível da fonte** (`kb_interna` | `busca_web` | `conhecimento_geral`) em cada item de `fontes`, para que a trilha de auditoria reflita a nova hierarquia.

## 12. Cálculos

Confirmado: **Code node no n8n**, nunca o LLM. O `.md` de fórmulas continua como especificação versionada e legível; o Code node implementa essas fórmulas como funções puras e testáveis.

## 13. Estimativa de eficácia e adesão

Metodologia mantida (matemática determinística para a projeção calórica + cenários de alta/moderada/baixa adesão como faixas, nunca uma data pontual). Ajuste pela nova regra de fontes: quando a base de conhecimento não tiver uma faixa de literatura para a variável em questão (ex. taxa esperada de ganho de massa magra para o perfil específico), o agente pode agora **complementar com busca no Google ou conhecimento geral**, desde que rotule claramente que aquele trecho da estimativa não vem da base de conhecimento interna — em vez de simplesmente declarar a lacuna sem oferecer nada.

## 14. Segurança

Mantida integralmente, sem flexibilização — a mudança na regra de fontes é sobre *transparência de origem da informação nutricional/de treino*, não sobre os critérios de risco à saúde. A camada de classificação (etapa 5 do workflow) continua bloqueando geração de plano em casos de sinais de transtorno alimentar, gravidez de risco, ou recusa em fornecer informações de segurança após sinal de risco já levantado pelo próprio usuário.

## 15. Validação — controle de qualidade automático

Atualizada conforme a nova regra: o critério de reprovação deixa de ser "toda afirmação precisa ter fonte na base de conhecimento" e passa a ser **"toda afirmação precisa ter um rótulo de fonte explícito"** (base interna, busca web, ou conhecimento geral) — o plano só é reprovado se alguma recomendação aparecer sem rótulo algum, se os números não baterem com o Code node, se restrições do usuário forem ignoradas, ou se houver contradição com os sinalizadores de segurança da etapa 14.

---

## 16. Recomendação final

Arquitetura fechada: **n8n Cloud (trial → self-hosted) + Telegram + Supabase (pgvector + operacional no mesmo projeto) + `gemini-embedding-001` para embeddings + `gemini-3.5-flash-lite` para todo o volume conversacional/validação + `gemini-3.7-flash` com Grounding with Google Search para a geração do plano**, com hierarquia de três níveis de fonte sempre rotulada explicitamente ao usuário, e cálculos sempre determinísticos em Code node. Essa combinação usa a cota escassa (20/dia dos modelos Flash "cheios") exatamente onde ela importa — a única etapa que exige raciocínio forte — e deixa tudo o que se repete várias vezes por conversa na cota generosa do Flash-Lite (500/dia).

## 17. Roadmap incremental (atualizado)

| Fase | Objetivo | Observação sobre cota do 3.7 Flash |
|---|---|---|
| **0 — Setup** | Contas, chaves, bot Telegram, projeto Supabase, PDFs + `.md` de teste | Nenhuma chamada ao 3.7 Flash ainda |
| **1 — Eco** | Telegram → `gemini-3.5-flash-lite` → resposta simples | Não usa 3.7 Flash |
| **2 — Anamnese** | Coleta das 5 obrigatórias + memória de conversa | Não usa 3.7 Flash |
| **3 — Cálculos** | Code node com fórmulas testado isoladamente | Não usa 3.7 Flash |
| **4 — RAG básico** | Ingestão + recuperação semântica funcionando | Não usa 3.7 Flash — teste a recuperação isoladamente antes de plugar na geração |
| **5 — Geração do plano** | Primeira versão do prompt de geração com `gemini-3.7-flash` + hierarquia de fontes | **Primeiro uso real da cota de 20/dia** — vale desenhar o prompt "no papel" antes de gastar chamadas ajustando |
| **6 — Personalização completa** | Perguntas recomendadas/opcionais + objeto de perfil rico | Usa a cota normalmente (1 geração por teste) |
| **7 — Explicabilidade** | Trilha dado→cálculo→evidência→decisão, com nível de fonte | — |
| **8 — Eficácia e adesão** | Cenários de aderência com a nova regra de fontes | — |
| **9 — Validação automática** | Camada de QA com o critério atualizado (rótulo de fonte) | — |
| **10 — Segurança** | Branch de classificação e encaminhamento | — |
| **11 — Ciclo completo** | Feedback → ajuste → nova versão do plano | — |
| **12 — Corte para self-hosted** | Ao fim do trial do n8n Cloud, exportar o workflow em JSON e migrar | — |

Cada fase continua sendo testável isoladamente — a única mudança prática de disciplina é: **evite gastar chamadas do 3.7 Flash antes da Fase 5**, já que tudo antes disso roda inteiramente no Flash-Lite ou em nós determinísticos.
