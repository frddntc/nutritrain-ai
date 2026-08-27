# Agente Científico de Treino e Dieta 🏋️‍♂️🧠

## 🎯 Qual é a proposta deste agente?
Este projeto é um agente de Inteligência Artificial focado na prescrição baseada em evidências de treinos de musculação e dietas. Desenvolvido com foco educacional e acadêmico, o sistema atua como um planejador de fitness de alta precisão.

Diferente de chatbots genéricos (que frequentemente "alucinam" recomendações nutricionais e erram cálculos calóricos), este agente foi arquitetado sob o princípio da **separação estrita de responsabilidades**:

1. **Motor Cognitivo (Gemini 3.1 Flash Lite):** Inteligência de baixo custo (Free Tier) e alta velocidade. É o orquestrador que interpreta a requisição do usuário, decide quais ferramentas chamar e formula a explicação final.
2. **Base Científica (RAG com Supabase + pgvector):** Consulta uma biblioteca de artigos científicos (PDFs) para garantir que as diretrizes sugeridas venham da literatura real, mitigando viéses não fundamentados.
3. **Motor Matemático Determinístico:** Em vez de pedir ao LLM para fazer contas, o agente aciona algoritmos determinísticos desenvolvidos em Python, garantindo precisão milimétrica no cálculo da Taxa Metabólica Basal, Macronutrientes e Volume de Treino.

## 🛠 Arquitetura e Integrações
*   **n8n Cloud (Orquestração):** Gerencia todo o fluxo de trabalho, os gatilhos e a comunicação com as APIs externas.
*   **Discord (Interface do Usuário):** Canal de comunicação interativo e assíncrono. Conta com gestão avançada de contexto para permitir o planejamento para múltiplas pessoas em um mesmo chat.
*   **Supabase (Banco de Dados):** Armazena o banco vetorial (pgvector) dos PDFs de estudo, os perfis dos usuários, logs de execução e planos gerados.
*   **Google AI Studio (LLM & Embeddings):** Fornece o Gemini 3.1 Flash Lite e o modelo de embeddings (text-embedding-004).

## 👤 Exemplo de Caso de Uso e Teste de Estresse
O sistema está sendo validado para lidar com perfis detalhados e rotinas complexas. 
**Exemplo de Perfil de Validação:**
*   **Usuário:** Masculino, 20 anos, 1,95m de altura, 105kg.
*   **Objetivo:** Recomposição corporal (foco simultâneo em ganho de massa e perda de gordura).
*   **Dieta:** Alvo aproximado de 3200 kcal/dia. Inclusão de variedada de proteínas, com uso de ovos como base rápida em dias sem marmitas preparadas.
*   **Treino:** Rotina estruturada de hipertrofia com divisão de 5 dias na semana.

**Ação do Agente:** Ao receber esse perfil, o agente não chuta a proteína baseando-se no peso total bruto (o que daria um valor inatingível), mas sim cruza o perfil de recomposição com a base científica, aciona a calculadora em Python para distribuir macros viáveis para as 3200 kcal e devolve a resposta estruturada via Discord.
