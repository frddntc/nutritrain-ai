# Assistente Fitness IA com RAG e Auto-Auditoria

Um assistente pessoal de saúde, nutrição e treino rodando via Telegram. O sistema coleta dados corporais de forma conversacional, efetua cálculos metabólicos determinísticos e gera planos de treino e dieta individualizados com busca semântica (RAG) e validação automatizada via IA auditora.

---

## Principais Funcionalidades

* **Anamnese Conversacional Inteligente**: A IA identifica os dados fornecidos pelo usuário no chat e solicita apenas os parâmetros corporais faltantes (`peso`, `altura`, `idade`, `sexo`, `objetivo`).
* **Cálculos Clínicos Determinísticos**: Prescrição nutricional baseada na equação de Mifflin-St Jeor, eliminando erros matemáticos e alucinações de LLM.
* **Busca Semântica na Base de Conhecimento (RAG)**: Resgate de artigos e referências internas em um banco vetorial Postgres (`pgvector`) com 768 dimensões.
* **Ciclo de Auto-Auditoria (QA Auditor)**: Uma segunda instância de IA avalia se a resposta gerada respeitou as metas de macronutrientes e as citações de fonte. Se reprovado, o plano é reescrito automaticamente antes do envio ao cliente.
* **Persistência de Dados**: Salva históricos de perfis e planos gerados no Supabase.

---

## Tecnologia Utilizada

* **Orquestração**: [n8n](https://n8n.io/)
* **Modelos de Linguagem**: Google Gemini (3.5 Flash e 3.5 Flash-Lite)
* **Embeddings**: `gemini-embedding-001` (768 dimensions)
* **Banco de Dados**: Supabase (PostgreSQL + `pgvector`)
* **Interface com Usuário**: Telegram Bot API

---

## Início Rápido

1. Siga o passo a passo de configuração do banco e credenciais no [setup.md](./setup.md).
2. Para detalhes técnicos da arquitetura dos nós e decisões de design, consulte o [docs.md](./docs.md).
3. Ative o workflow no n8n e envie uma mensagem para o seu bot no Telegram!
