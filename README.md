# Agente Científico de Treino e Nutrição 🏋️‍♂️🧠

## 🎯 Proposta do Projeto
Este projeto é um agente de Inteligência Artificial focado na prescrição baseada em evidências de treinos de musculação e dietas. Desenvolvido com foco educacional e acadêmico, o sistema atua como um assistente de planejamento fitness de alta precisão, operando **100% com custo zero** (bootstrap) através de ferramentas em cloud.

Diferente de chatbots genéricos, este agente foi arquitetado sob o princípio da **separação estrita de responsabilidades**:
1. **Motor Cognitivo (Gemini 3.1 Flash Lite):** Atua como orquestrador lógico. Interpreta as requisições via Discord, decide quais ferramentas acionar e elabora as respostas baseadas em dados recuperados.
2. **Base Científica (RAG com Supabase + pgvector):** Consulta uma biblioteca de artigos científicos (em vetores de 768 dimensões) para fundamentar as diretrizes em literatura atual.
3. **Motor Matemático Determinístico (Code Node nativo n8n em Python):** Cálculos metabólicos e distribuição de macronutrientes são executados via código Python isolado do LLM, garantindo precisão matemática (ex: equações de Mifflin-St Jeor) sem risco de alucinações lógicas.

## 🛠 Arquitetura e Stack Tecnológica
*   **n8n Cloud:** Orquestração de todo o fluxo de trabalho (Workflows, Webhooks, Roteamento, Validação Ed25519).
*   **Discord API:** Interface de interação com o usuário operando exclusivamente via *Slash Commands* (`/mensagem`, `/novo_plano`) para total compatibilidade com os endpoints de *Interaction* do Discord.
*   **Supabase:** Banco de dados PostgreSQL contendo:
    *   `pgvector`: Armazenamento de embeddings científicos.
    *   `Chat Memory`: Armazenamento de contexto isolado por ID de usuário (permitindo múltiplos usuários no mesmo canal sem cruzar dados).
*   **Google AI Studio:** LLM (`gemini-3.1-flash-lite`) e modelo de embeddings (`gemini-embedding-001`) via API Free Tier.

## 👤 Exemplo de Caso de Uso e Teste de Estresse
O sistema foi desenhado para lidar com fisiologias e rotinas complexas, não se limitando a regras genéricas engessadas.

**Exemplo Prático de Validação:**
*   **Perfil:** Masculino, 20 anos, 1,95m de altura, 105kg.
*   **Objetivo:** Recomposição corporal (ganho de massa muscular com perda simultânea de gordura).
*   **Demanda:** Rotina estruturada de musculação para 5 dias na semana, meta aproximada de 3200 kcal/dia, aproveitando fontes proteicas rápidas (como ovos) para dias sem refeições pré-preparadas.

**Resolução do Agente:** Em vez de usar a diretriz comum e falha de "2g de proteína por kg de peso total" (o que resultaria em inviáveis 210g+ de proteína diária para alguém com 105kg), o *Code Node* em Python estima a massa magra do indivíduo e calibra os macronutrientes de forma executável dentro da meta de 3200 kcal. O LLM então recupera diretrizes científicas sobre volume de treino adequado para recomposição e monta a resposta personalizada.

## ⚖️ Disclaimer Legal e Ético
Este assistente atua estritamente como um **Copiloto de Dieta e Treino** orientado à educação e tecnologia. **Não se trata de um Nutrólogo, Nutricionista ou Médico.** Toda sugestão gerada pelo agente deve ser validada por um profissional de saúde habilitado, e o sistema é instruído a reforçar essa ressalva em suas interações.
