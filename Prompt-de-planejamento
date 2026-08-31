# CONTEXTO

Quero desenvolver, principalmente para **estudo e aprendizado do n8n Cloud**, um projeto de agente de IA capaz de auxiliar usuários na criação de **planos individualizados de dieta e treinamento físico**.

Este pedido é um **BRAINSTORM DE SOLUÇÃO**.

Não quero que você pule diretamente para uma implementação definitiva. Primeiro quero que você analise os requisitos, identifique lacunas, proponha alternativas de arquitetura, compare as opções e, somente depois, recomende a melhor solução.

O projeto deve utilizar **exclusivamente ferramentas gratuitas**, considerando as limitações e free tiers disponíveis.

Tenho chaves de acesso aos seguintes modelos/recursos Gemini:

* Gemini Embedding 1;
* Gemini Embedding 2;
* Gemini 3 Flash;
* Gemini 3.1 Flash Lite.

Quero aproveitar esses recursos sempre que fizer sentido.

---

# 1. OBJETIVO DO SISTEMA

Quero criar um chatbot que, a partir das informações fornecidas pelo usuário, seja capaz de auxiliar na criação de um **plano específico de dieta e treino**, levando em consideração:

* Características individuais;
* Objetivos;
* Rotina;
* Disponibilidade para treinar;
* Preferências alimentares;
* Restrições;
* Limitações físicas;
* Experiência prévia;
* Outros parâmetros relevantes.

O sistema possuirá uma **base de conhecimento própria**, composta por arquivos científicos e técnicos sobre:

* Nutrição;
* Treinamento;
* Composição corporal;
* Perda de gordura;
* Ganho de massa muscular;
* Hipertrofia;
* Força;
* Recuperação;
* Outros assuntos relevantes.

Além disso, haverá uma **tabela/arquivo de fórmulas** contendo os cálculos que poderão ser necessários para objetivos como:

* Perda de gordura;
* Ganho de massa muscular;
* Estimativa de necessidades energéticas;
* Macronutrientes;
* Outros cálculos relevantes.

O agente deverá utilizar essa base de conhecimento para fundamentar suas decisões.

---

# 2. REGRA FUNDAMENTAL: NÃO INVENTAR INFORMAÇÕES

Esta é uma das regras mais importantes de todo o projeto.

O agente **NÃO pode inventar**:

* Dados do usuário;
* Fórmulas;
* Valores nutricionais;
* Recomendações de treino;
* Recomendações de dieta;
* Resultados de estudos;
* Evidências científicas;
* Informações presentes nos documentos.

Toda recomendação relacionada a nutrição ou treinamento deve ser fundamentada nos arquivos disponibilizados na base de conhecimento.

Caso a base de conhecimento não contenha informação suficiente para responder ou tomar determinada decisão, o agente deve:

1. Identificar a lacuna;
2. Informar que não possui fundamentação suficiente;
3. Solicitar informações adicionais ou indicar que aquela decisão não pode ser tomada com segurança com a base atual.

**Nunca preencher uma lacuna com conhecimento inventado.**

Também quero que você avalie se essa regra é tecnicamente adequada para o funcionamento de um sistema RAG e sugira mecanismos para reduzir alucinações.

---

# 3. COLETA DE INFORMAÇÕES DO USUÁRIO

É de **SUMA IMPORTÂNCIA** que o agente não forneça uma dieta ou planilha de treino sem possuir, no mínimo, informações básicas sobre o usuário.

No primeiro contato, o agente deve agradecer a mensagem e explicar que precisa conhecer melhor o usuário antes de elaborar qualquer plano.

## Informações OBRIGATÓRIAS

Estas informações são obrigatórias para que o agente possa trabalhar:

* **Idade**
* **Gênero/sexo**
* **Altura**
* **Objetivo**
* **Peso**

Se qualquer uma dessas quatro informações estiver ausente, o agente **não deve elaborar o plano final**.

---

# 4. INFORMAÇÕES ADICIONAIS

Além das quatro informações obrigatórias, quero que você proponha quais outras perguntas deveriam ser feitas para melhorar significativamente a personalização.

Considere perguntas relacionadas a:

### Perfil físico

* Percentual de gordura, caso o usuário saiba;
* Histórico de peso;
* Medidas corporais, quando relevantes.

### Objetivos

* Objetivo principal;
* Objetivos secundários;
* Prazo desejado;
* Prioridade entre estética, força, performance etc.

### Treinamento

* Experiência;
* Quantidade de dias disponíveis;
* Tempo disponível por sessão;
* Local de treinamento;
* Equipamentos disponíveis;
* Exercícios preferidos;
* Exercícios que não gosta;
* Histórico de treinamento.

### Saúde e limitações

* Lesões;
* Dores;
* Limitações de movimento;
* Condições médicas relevantes;
* Medicamentos relevantes;
* Restrições/recomendações médicas.

### Alimentação

* Preferências;
* Alergias;
* Intolerâncias;
* Restrições;
* Padrão alimentar;
* Alimentos que não consome;
* Número de refeições;
* Horários;
* Capacidade de cozinhar;
* Refeições fora de casa;
* Orçamento.

### Rotina

* Trabalho;
* Estudos;
* Horário de sono;
* Qualidade do sono;
* Nível de atividade física diária;
* Horários disponíveis para treinar;
* Horários disponíveis para comer.

### Adesão

Quero também que você proponha perguntas capazes de medir a **probabilidade de adesão** do usuário ao plano.

Por exemplo, aspectos relacionados a:

* Disciplina;
* Preferências;
* Disponibilidade;
* Histórico de aderência;
* Dificuldades anteriores;
* Flexibilidade desejada.

**IMPORTANTE:** as perguntas adicionais devem ser sugestões suas. As únicas perguntas obrigatórias para o agente funcionar são:

**Idade + Gênero/sexo + Altura + Objetivo + Peso.**

Quero que você recomende quais perguntas adicionais deveriam ser obrigatórias, recomendadas ou opcionais, explicando o impacto de cada uma.

---

# 5. FLUXO PRINCIPAL DA INTEGRAÇÃO

O fluxo desejado inicialmente é:

### Etapa 1 — Entrada

O usuário envia uma mensagem pelo chatbot.

### Etapa 2 — Anamnese

O agente agradece o contato e solicita as informações necessárias.

Deve evitar perguntar novamente informações que o usuário já tenha fornecido.

### Etapa 3 — Validação

O sistema verifica se possui, no mínimo:

* Idade;
* Gênero/sexo;
* Altura;
* Objetivo;
* Peso.

Se faltar qualquer uma delas, deve continuar a coleta.

### Etapa 4 — Recuperação de conhecimento

Depois que houver informações suficientes, o agente consulta o banco de conhecimento.

Ele deverá recuperar os documentos relevantes para:

* Objetivo do usuário;
* Cálculos necessários;
* Estratégia nutricional;
* Estratégia de treinamento;
* Restrições e características individuais.

Quero que você proponha uma arquitetura **RAG**, explicando:

* Onde armazenar os documentos;
* Como gerar embeddings;
* Onde armazenar os embeddings;
* Como fazer busca semântica;
* Como filtrar documentos;
* Como entregar o contexto recuperado ao Gemini;
* Como evitar que documentos irrelevantes sejam utilizados.

### Etapa 5 — Cálculos

O agente utiliza as fórmulas existentes no arquivo específico de fórmulas.

Quero que você avalie se esses cálculos devem ser:

* Realizados diretamente pelo LLM;
* Realizados pelo próprio n8n através de Code/expressions;
* Realizados em banco de dados;
* Ou através de alguma combinação.

**Priorize a opção mais confiável para cálculos matemáticos.**

Explique os prós e contras de cada alternativa.

### Etapa 6 — Elaboração do plano

Com base em:

**Dados do usuário + fórmulas + documentos recuperados**

o agente deve elaborar:

* Plano alimentar;
* Plano de treinamento;
* Justificativa das principais decisões.

O resultado deve ser individualizado.

Não quero respostas genéricas do tipo "coma proteínas e treine 3 vezes por semana".

---

# 6. PERSONALIZAÇÃO

A personalização é essencial.

O agente deve considerar simultaneamente:

**EVIDÊNCIA CIENTÍFICA**

* **FÓRMULAS**
* **DADOS DO USUÁRIO**
* **RESTRIÇÕES**
* **ROTINA**
* **OBJETIVOS**
* **PREFERÊNCIAS**

para chegar à recomendação final.

Quero que você proponha uma forma de estruturar esses dados para que o agente consiga utilizá-los de maneira consistente.

Por exemplo, avalie a necessidade de criar um objeto estruturado contendo:

* Perfil;
* Objetivos;
* Saúde;
* Treino;
* Nutrição;
* Rotina;
* Restrições;
* Preferências;
* Histórico;
* Adesão.

---

# 7. EXPLICAÇÃO DAS RECOMENDAÇÕES

O agente não deve apenas entregar o plano.

Ele deve conseguir explicar **por que recomendou determinada estratégia**.

Sempre que possível, deverá relacionar:

**Dado do usuário → cálculo → evidência → decisão.**

Quero que você proponha uma estrutura para isso.

Por exemplo, quero que seja possível identificar posteriormente:

* Qual documento fundamentou determinada recomendação;
* Qual fórmula foi utilizada;
* Quais dados do usuário entraram no cálculo;
* Qual foi o raciocínio que levou à decisão.

Isso é importante tanto para aprendizado quanto para auditoria do sistema.

---

# 8. ESTIMATIVA DE EFICÁCIA E TEMPO

Depois de elaborar o plano, o agente **DEVE informar uma estimativa de eficácia**, levando em consideração a quantidade e qualidade dos parâmetros fornecidos pelo usuário.

Além disso, é **MUITO IMPORTANTE** considerar a adesão do usuário.

A lógica desejada é semelhante a:

> "Com adesão total a este modelo de dieta e treino, a estimativa é atingir o objetivo em aproximadamente X meses."

ou:

> "Com adesão parcial, adotando algumas partes do plano e não outras, a estimativa pode se estender para aproximadamente X + Y meses."

Porém, quero que você analise criticamente esse requisito.

**Não quero que o sistema invente prazos científicos.**

Quero que você proponha uma metodologia tecnicamente defensável para:

1. Estimar eficácia;
2. Estimar tempo;
3. Relacionar resultado esperado à adesão;
4. Expressar incerteza;
5. Evitar falsa precisão.

Se a base científica não permitir estimar um prazo confiável, o agente deverá dizer isso.

Considere a possibilidade de apresentar:

* Cenário de alta adesão;
* Cenário de adesão moderada;
* Cenário de baixa adesão;

em vez de fornecer uma data artificialmente precisa.

Explique como isso poderia ser implementado no n8n.

---

# 9. VALIDAÇÃO DO PLANO

Depois de apresentar o plano, o agente deve aguardar a validação do usuário.

Se o usuário estiver satisfeito:

* Encerrar o fluxo ou passar para o próximo estágio definido pelo sistema.

Se o usuário não estiver satisfeito:

1. Perguntar o que deseja alterar;
2. Explicar por que a recomendação original foi feita;
3. Apresentar a fundamentação nos documentos utilizados;
4. Identificar quais parâmetros estão causando a recomendação;
5. Oferecer a possibilidade de coletar **mais informações**;
6. Gerar um novo plano quando houver informações suficientes.

O agente não deve simplesmente modificar arbitrariamente o plano porque o usuário pediu algo incompatível com as evidências.

Ele deve explicar o conflito e buscar uma alternativa viável.

---

# 10. BANCO DE CONHECIMENTO

Preciso criar um banco de dados/base de conhecimento própria para armazenar os arquivos que fundamentarão o agente.

Quero que você proponha **pelo menos 2 arquiteturas gratuitas** para isso.

Considere, por exemplo:

* PostgreSQL + pgvector;
* Supabase;
* Qdrant;
* Chroma;
* Google Drive + banco vetorial;
* Outras alternativas gratuitas que façam sentido.

Não assuma que essas opções são necessariamente as melhores.

Pesquise/avalie as alternativas atuais e compare:

* Gratuidade;
* Limites do free tier;
* Facilidade de integração com n8n Cloud;
* Suporte a vetores;
* Metadados;
* Busca semântica;
* Facilidade de manutenção;
* Curva de aprendizado;
* Escalabilidade para um projeto pessoal;
* Compatibilidade com Gemini Embeddings.

---

# 11. BANCO DE DADOS OPERACIONAL

Além do banco de conhecimento, preciso de uma estrutura para armazenar informações do sistema e facilitar a integração com o n8n.

Proponha uma arquitetura para armazenar, quando necessário:

* Usuários;
* Perfil;
* Respostas da anamnese;
* Objetivos;
* Histórico de conversas;
* Planos gerados;
* Versões dos planos;
* Feedback;
* Adesão;
* Documentos utilizados;
* Cálculos;
* Status do atendimento.

Explique se é melhor utilizar:

* O mesmo banco do RAG;
* Um banco separado;
* Tabelas separadas dentro da mesma solução.

---

# 12. CHATBOT

O agente deverá ser utilizado como chatbot.

Quero que você apresente **pelo menos 2 opções gratuitas** de plataforma para hospedar/intermediar o chatbot.

Considere, entre outras:

* Telegram;
* Discord;
* Webchat;
* WhatsApp, caso exista alguma alternativa realmente gratuita e viável;
* Outras opções.

Compare:

* Facilidade de integração com n8n Cloud;
* Custo;
* Facilidade de configuração;
* Experiência do usuário;
* Persistência de conversa;
* Autenticação;
* Webhooks;
* Limitações;
* Adequação ao meu projeto de estudo.

**Não recomende uma ferramenta apenas porque possui plano gratuito. Verifique se ela realmente é adequada para o fluxo proposto.**

---

# 13. MODELOS GEMINI

Tenho acesso aos seguintes modelos/recursos:

* Gemini Embedding 1;
* Gemini Embedding 2;
* Gemini 3 Flash;
* Gemini 3.1 Flash Lite.

Quero que você avalie como distribuir essas funções entre os modelos.

Por exemplo:

* Embeddings → modelo de embedding;
* Conversação/anamnese → modelo adequado;
* Raciocínio e elaboração do plano → modelo adequado;
* Tarefas simples → modelo mais econômico/rápido.

Não assuma que os nomes ou capacidades informadas estão atualizados.

**Verifique a documentação atual do Google/Gemini antes de recomendar a arquitetura**, especialmente nomes de modelos, disponibilidade, limites gratuitos e compatibilidade com embeddings.

Se algum modelo citado não existir mais, tiver sido substituído ou possuir nomenclatura diferente, informe isso e proponha a alternativa atual.

---

# 14. N8N CLOUD

O projeto será desenvolvido no **n8n Cloud**.

Quero que você proponha o workflow completo, pensando em nós do n8n.

Explique uma possível arquitetura contendo, quando necessário:

* Trigger/Webhook;
* Entrada do chatbot;
* Memória/conversação;
* Validação dos dados;
* Banco de usuários;
* RAG;
* Embeddings;
* Busca vetorial;
* Recuperação dos documentos;
* Cálculos;
* LLM;
* Geração do plano;
* Persistência;
* Feedback;
* Replanejamento.

Para cada etapa, indique:

**Nó → Função → Entrada → Processamento → Saída → Próxima etapa**

Não precisa implementar tudo ainda.

Neste momento quero entender a **arquitetura ideal**.

---

# 15. SEGURANÇA E LIMITAÇÕES

Como o sistema envolve saúde, alimentação e exercício, quero que você inclua uma camada de segurança.

Analise:

* Quais informações exigem encaminhamento para profissional de saúde;
* Como lidar com doenças, lesões e transtornos alimentares;
* Como evitar diagnósticos;
* Como lidar com usuários menores de idade;
* Como lidar com medicamentos;
* Como lidar com metas potencialmente perigosas;
* Quando interromper a geração do plano;
* Como incluir avisos sem tornar o chatbot inutilizável.

Quero que essa camada seja parte da arquitetura, e não apenas um texto colocado no final da resposta.

---

# 16. CONTROLE DE QUALIDADE

Proponha mecanismos para verificar se o agente está:

* Utilizando os documentos corretos;
* Utilizando as fórmulas corretas;
* Respeitando os dados do usuário;
* Não inventando informações;
* Não ignorando restrições;
* Não fornecendo cálculos inconsistentes;
* Não fornecendo recomendações sem fonte;
* Mantendo consistência entre dieta e treino.

Avalie a possibilidade de criar uma segunda etapa de **validação automática do plano** antes de enviá-lo ao usuário.

---

# 17. ESTRUTURA DOS DOCUMENTOS

Também quero recomendações sobre como devemos preparar os arquivos que serão inseridos na base de conhecimento.

Explique:

* Quais tipos de PDF devemos priorizar;
* Como organizar os documentos;
* Como nomeá-los;
* Quais metadados devemos armazenar;
* Como separar documentos de nutrição e treinamento;
* Como armazenar o arquivo `.md` de fórmulas;
* Como versionar as fórmulas;
* Como evitar documentos conflitantes ou desatualizados.

Proponha uma estrutura de metadados, por exemplo:

* Título;
* Autor;
* Ano;
* Tipo de estudo;
* Tema;
* População;
* Objetivo;
* Fonte;
* DOI;
* Data de atualização;
* Nível de evidência.

---

# 18. REQUISITO DE CUSTO

Este projeto é principalmente para **estudo pessoal do n8n Cloud**.

Portanto:

**Priorize ferramentas gratuitas.**

Quero saber claramente:

* O que é realmente gratuito;
* O que possui free tier;
* Quais são os limites;
* O que poderá gerar custo posteriormente;
* Onde existe risco de cobrança inesperada.

Se uma solução aparentemente gratuita exigir cartão de crédito ou possuir limite muito baixo, destaque isso.

---

# 19. COMPARAÇÃO DE ARQUITETURAS

Antes de escolher uma solução final, quero que você apresente **pelo menos 2 arquiteturas completas**.

Por exemplo:

### Arquitetura A

Chatbot + n8n + Banco X + Vector DB X + Gemini.

### Arquitetura B

Chatbot + n8n + Banco Y + Vector DB Y + Gemini.

Mas não se limite a essas possibilidades.

Para cada arquitetura, apresente:

* Diagrama textual;
* Ferramentas;
* Fluxo dos dados;
* Pontos positivos;
* Pontos negativos;
* Custo;
* Complexidade;
* Facilidade de manutenção;
* Adequação para aprendizado;
* Adequação para uso pessoal.

Depois, faça uma **recomendação objetiva de qual arquitetura eu deveria escolher e por quê**.

---

# 20. NÃO IMPLEMENTE AINDA

Neste primeiro momento, **não quero que você escreva o workflow completo do n8n nem código definitivo**.

Quero primeiro um brainstorm técnico e uma proposta de solução.

A resposta deve me permitir escolher:

1. Arquitetura;
2. Banco de dados;
3. Banco vetorial;
4. Plataforma do chatbot;
5. Organização dos documentos;
6. Estratégia de RAG;
7. Estratégia de uso dos modelos Gemini;
8. Estrutura do workflow no n8n.

**Depois que eu escolher uma arquitetura, poderemos partir para a implementação passo a passo.**

---

# FORMATO DA SUA RESPOSTA

Organize o brainstorm nesta ordem:

## 1. Resumo do projeto

Demonstre que você entendeu o que estou tentando construir.

## 2. Pontos críticos e riscos

Identifique problemas técnicos ou conceituais no projeto original.

## 3. Informações que o chatbot deve coletar

Separe em:

* Obrigatórias;
* Recomendadas;
* Opcionais.

Explique o impacto de cada grupo.

## 4. Arquitetura RAG

Explique como os PDFs e fórmulas entrarão no sistema e serão utilizados pelo agente.

## 5. Opções de banco de conhecimento

Apresente pelo menos 2 alternativas.

## 6. Opções de banco operacional

Apresente pelo menos 2 alternativas.

## 7. Opções de chatbot

Apresente pelo menos 2 alternativas.

## 8. Opções de arquitetura completa

Apresente pelo menos 2 arquiteturas end-to-end.

## 9. Gemini

Explique qual modelo utilizar em cada parte e verifique a documentação atual.

## 10. Workflow do n8n

Apresente o fluxo conceitual dos nós, sem implementar ainda.

## 11. Personalização

Explique como transformar os dados do usuário + evidências + fórmulas em um plano realmente individualizado.

## 12. Cálculos

Explique onde os cálculos devem ocorrer e como evitar erros do LLM.

## 13. Estimativa de eficácia e adesão

Proponha uma metodologia tecnicamente defensável para essa parte.

## 14. Segurança

Proponha mecanismos de segurança para um sistema que lida com saúde, dieta e exercício.

## 15. Validação

Explique como verificar automaticamente a qualidade do plano antes de entregá-lo.

## 16. Recomendação final

Escolha a arquitetura que considera mais adequada para **meu objetivo de aprender n8n, utilizando ferramentas gratuitas**, e justifique.

## 17. Roadmap

Divida o desenvolvimento em etapas progressivas, começando pelo MVP mais simples e evoluindo até o sistema completo.

**Não implemente ainda.**

Quero primeiro discutir e escolher a arquitetura.

Lembre-se: isto é um **BRAINSTORM**. Você pode adicionar componentes, melhorias, fluxos ou mecanismos que não foram mencionados acima caso considere que eles sejam importantes para tornar o sistema mais confiável, seguro, barato e tecnicamente consistente.

Porém, **não remova nenhum dos requisitos que defini neste documento**.
