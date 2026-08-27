# 📚 Por que cada mudança foi feita

Este documento explica o raciocínio por trás de cada ajuste que fiz no setup gerado pelo Comet. A ideia não é só te entregar um guia que funciona, mas te ajudar a entender **por quê**, para que da próxima vez você consiga avaliar um setup desse tipo sozinho — ou até pegar um erro parecido em outro projeto.

## Resumo rápido

| O que mudou | Por quê, em uma frase |
|---|---|
| RLS sem política pública + `service_role` no lugar da `anon` | A versão original deixava o banco aberto para leitura/escrita de qualquer pessoa com a chave |
| `ivfflat` → `hnsw` | HNSW é o padrão recomendado hoje: mais rápido e não depende de já ter dados na tabela para funcionar bem |
| Função `match_pdf_embeddings` (nova) | Sem ela, o node de busca do n8n não tem como consultar o banco |
| `text-embedding-004` → `gemini-embedding-001` (768 dim) | O modelo antigo está sendo substituído; o novo precisa de configuração explícita de dimensão |
| `gemini-2.5-flash` → modelo atual (a conferir) | A família 2.5 do Gemini será desligada em outubro de 2026 |
| Números fixos de cota do Gemini removidos | Mudaram pelo menos duas vezes nos últimos meses; um número fixo aqui ficaria errado rápido |
| "n8n Cloud grátis" → teste de 14 dias + nota sobre self-hosting | O plano gratuito permanente não existe mais |
| Nota sobre LGPD nos dados do Google Sheets | O bot coleta dado de saúde, que é dado pessoal sensível |

Agora, com mais detalhe.

---

## 1. Supabase e o banco vetorial

### Por que RLS ligado, mas sem política "permitir tudo"?

O Row Level Security (RLS) do Postgres funciona assim: quando você liga ele numa tabela, o banco passa a negar qualquer acesso por padrão, e só libera o que estiver descrito numa política explícita. O setup original ligava o RLS (certo) mas depois criava esta política:

```sql
create policy "Permitir acesso público" on pdf_embeddings
  for all using (true) with check (true);
```

`using (true) with check (true)` significa "permita tudo, para todo mundo, sem checar nada". Na prática, isso **desliga a proteção que o RLS deveria dar** — é como trancar a porta e deixar a chave debaixo do tapete com uma placa escrito "chave aqui".

O problema fica pior porque o guia orientava usar a chave `anon`/`public` nessa configuração. Essa chave é *desenhada* para ser usada em lugares menos protegidos (como o navegador de um usuário), justamente porque o RLS é quem deveria limitar o que ela pode fazer. Com a política acima, qualquer pessoa que conseguisse essa chave — por um print de tela, um repositório público, um workflow exportado por engano — teria acesso total de leitura *e escrita* ao seu banco: poderia ler todo o conteúdo dos PDFs, ou apagar a tabela inteira.

A correção: manter o RLS ligado, **não criar nenhuma política pública**, e usar a `service_role key` só dentro da credencial do n8n. Isso não é uma "regra geral" que você deve replicar em qualquer projeto — é a resposta certa *para este caso específico*, porque quem acessa o banco é um backend que só você controla (o n8n), não o navegador de um usuário final. Se um dia esse mesmo banco precisasse ser acessado direto por um app no celular do usuário, aí sim a `anon` key com políticas de RLS bem desenhadas (por exemplo, "cada usuário só vê os próprios dados") seria o caminho certo.

### Por que HNSW em vez de IVFFlat?

Os dois são formas de acelerar a busca por similaridade num banco de vetores — sem índice, o Postgres precisa comparar o vetor de busca com *cada* linha da tabela, o que fica lento conforme a tabela cresce.

A diferença prática: o IVFFlat divide os vetores em "grupos" (clusters) calculados a partir dos dados existentes na tabela *no momento em que o índice é criado*. Se você cria o índice com a tabela vazia (como no setup original, que criava o índice antes de qualquer PDF ser processado), os clusters ficam mal calculados, e a busca perde qualidade até você recriar o índice depois de ter dados de verdade. O HNSW não tem essa pegadinha: ele constrói um grafo de vizinhança que vai se ajustando conforme os dados entram, então funciona bem desde o primeiro documento inserido. Hoje é considerado o índice padrão para a maioria dos casos de uso — só costuma perder para o IVFFlat em cenários bem específicos de bancos gigantescos com filtros muito seletivos, o que não é o seu caso aqui.

### Por que a função `match_pdf_embeddings`?

Quando você configura o node "Supabase Vector Store" do n8n no modo de busca, ele não monta uma consulta SQL na hora — ele chama uma função do Postgres que **você precisa ter criado antes**, passando o vetor de busca como parâmetro. Essa função devolve os textos mais parecidos, ordenados por similaridade. O guia original pulava essa etapa inteira; o resultado seria o node de busca simplesmente não encontrar nada (ou dar erro de "função não existe"), mesmo com os PDFs corretamente processados e salvos.

A distância usada (`<=>`, distância de cosseno) mede o ângulo entre dois vetores — quanto menor o ângulo, mais parecido o significado dos textos. `1 - distância` transforma isso numa nota de "similaridade" mais intuitiva, onde 1 é idêntico e 0 é sem relação nenhuma.

### Por que 768 dimensões, e por que isso importa?

"Dimensão" aqui é só o tamanho da lista de números que representa cada pedaço de texto. A coluna da tabela (`vector(768)`) e o modelo de embedding usado precisam concordar nesse número — se o modelo devolver um vetor de tamanho diferente, o insert simplesmente falha.

O `gemini-embedding-001` (o modelo atual do Google) devolve, por padrão, vetores de 3072 dimensões — bem maior que os 768 do `text-embedding-004` antigo. Isso trouxe dois problemas: primeiro, não bateria com a tabela original; segundo, o pgvector só consegue *indexar* vetores de até 2.000 dimensões (HNSW e IVFFlat têm esse teto), então um vetor de 3072 dimensões nem poderia ter um índice HNSW criado em cima dele. A solução foi pedir explicitamente ao modelo para devolver vetores truncados para 768 dimensões (o parâmetro `output_dimensionality`) — o modelo foi treinado de um jeito que permite isso sem perder muita qualidade semântica, e assim mantemos a mesma estrutura de tabela do setup original.

---

## 2. Gemini API

### Por que não seguir com `gemini-2.5-flash`?

Fui conferir a documentação oficial do Google (Firebase AI Logic, que compartilha o mesmo ciclo de vida de modelos da Gemini API) e a página, atualizada em 24 de agosto de 2026, avisa que **toda a família Gemini 2.5 será desligada em outubro de 2026** — cerca de um mês e meio a partir de quando você provavelmente vai rodar este setup. Configurar um projeto novo hoje com um modelo que some em semanas é um convite a um workflow quebrado do nada. Por isso troquei a recomendação para o modelo "flash" mais atual (no momento, `gemini-3.7-flash`) e deixei uma instrução para você sempre conferir a lista de modelos atual antes de finalizar — porque essa lista também vai mudar de novo, e um nome fixo aqui ficaria desatualizado eventualmente.

### Por que tirei os números de cota (250 requisições/dia etc.)?

Pesquisando, encontrei relatos consistentes de que o Google cortou as cotas gratuitas da Gemini API de forma bem agressiva em dezembro de 2025 (em alguns casos, uma redução de mais de 80% para certos modelos), e diferentes fontes — inclusive posts recentes de blogs especializados — davam números diferentes entre si para os limites atuais. Isso é um sinal claro de que qualquer número que eu escrevesse aqui teria uma chance real de já estar errado. Preferi apontar para a página oficial de rate limits, que é a única fonte que se mantém correta por definição.

Essa é uma lição que vale generalizar: **documentação técnica com números específicos de terceiros (preço, cota, limite) envelhece rápido**. Sempre que possível, é mais seguro linkar para a fonte oficial do que copiar o número.

### Por que `gemini-embedding-001` no lugar de `text-embedding-004`?

O Google já sinaliza o `gemini-embedding-001` como o modelo de embedding estável e recomendado atualmente, unificando o que antes eram modelos separados. Manter uma configuração baseada em `text-embedding-004` significaria começar já em cima de um modelo em processo de substituição.

---

## 3. Telegram

Aqui só reforcei um cuidado de segurança simples, sem mudar a lógica: o token do bot funciona, na prática, como uma senha. Qualquer pessoa com ele pode controlar seu bot — ler mensagens, responder no seu lugar, mudar comandos. Vale o mesmo cuidado que você teria com a senha de um e-mail.

---

## 4. n8n

### Por que o "grátis" mudou?

O setup original dizia "n8n Cloud - Free (5.000 execuções/mês)". Pesquisando o plano atual, encontrei consistentemente que isso não existe mais: hoje o n8n Cloud oferece um teste gratuito de 14 dias, e depois disso os planos são pagos (a partir de ~€20/mês para 2.500 execuções no plano Starter). Isso é uma mudança de modelo de negócio da empresa, não uma flutuação de preço — importante você saber antes de investir tempo configurando tudo esperando um plano gratuito permanente que não vai aparecer.

### Cloud vs. self-hosted, rapidamente

Deixei as duas opções no guia porque a escolha depende do que você valoriza mais agora: **Cloud** é mais rápido para começar e não exige manter servidor no ar, mas custa depois do teste. **Self-hosted** (rodar o n8n você mesmo, geralmente via Docker, num VPS barato ou até numa máquina em casa) é genuinamente gratuito indefinidamente, mas exige um pouco mais de trabalho de manutenção — e algum conhecimento básico de linha de comando. Para um projeto de aprendizado como este, self-hosted também tem a vantagem de te ensinar mais sobre como as peças se conectam.

---

## 5. Os workflows

### O campo "Query Name" que faltava

Esse é o mesmo problema do item da função `match_pdf_embeddings`, só que do lado da configuração do node: mesmo com a função criada no banco, o node de busca do n8n precisa saber o *nome* dela para chamá-la. Esse campo (`Query Name`) não aparecia mencionado na configuração original do node de busca — eu o adicionei explicitamente porque é um dos erros mais comuns (e mais confusos de diagnosticar) em integrações n8n + Supabase Vector Store: tudo parece configurado certo, mas a busca simplesmente não retorna nada.

### A frase de responsabilidade no prompt

Isso não é uma mudança técnica, é uma de bom senso: o bot vai dar sugestões de dieta e treino a partir de dados pessoais de saúde. Uma frase simples no prompt lembrando que não substitui acompanhamento profissional custa pouco e reduz o risco de alguém seguir uma recomendação de forma inadequada para o próprio caso.

---

## 6. Google Sheets e a nota de LGPD

Não mudei nada tecnicamente aqui — só adicionei um alerta. Peso, altura e (potencialmente) condições de saúde são classificados como **dado pessoal sensível** pela LGPD (Lei Geral de Proteção de Dados), o que implica um cuidado maior do que dado comum: mais restrição de acesso, mais transparência com quem fornece o dado, e um jeito de apagar quando pedido. Não é um parecer jurídico — é só o tipo de coisa que vale ter em mente desde o início de um projeto assim, porque é bem mais fácil construir isso desde o começo do que adicionar depois.

---

## Um hábito que vale manter

Sempre que um guia de setup pedir para você colar uma chave ou segredo em algum lugar, vale parar um segundo e perguntar: "quem mais consegue ver isso, e o que essa chave permite fazer?". Foi exatamente essa pergunta que revelou o problema de RLS neste setup — a chave em si (`anon`) era a recomendada pela documentação do Supabase para esse tipo de uso; o problema estava na política que foi combinada com ela.
