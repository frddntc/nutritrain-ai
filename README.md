# Agente (sistema) que seja capaz de, a partir de uma mensagem do usuário, auxiliá-lo na criaão de um plano específico de dieta e treino
## Como isso vai funcionar?
Nosso sistema possui uma base de arquivos para embasar os pensamentos do agente. Esses arquivos, contém informações sobre nutricão e treino, além de uma tabela de fórmulaas que podem ser necessárias para a perda de gordura, o ganho de massa magra e outros mais objetivos. Tendo esta base de dados em mão, o agente deve utilizá-la para, em conjunto com informações extraídas do usuário, moldar um plano de dieta e treino que se encaixe melhor dentro da rotina e dos objetivos do usuário.
É de SUMA IMPORTÂNCIA que o agente não invente dados, não forneça planilhas de treino ou dieta sem ter no mínimo informações básicas sobre o usuário.
Após o contato inicial, o agente deve responder com perguntas sobre o usuário, como: Idade, Gênero, Altura, Rotina, Objetivos e mais o que julgar ser necessário para uma boa parametrização de treino e dieta (me dê ideias de perguntas a serem feitas para o usuário no contato inicial, porém mantenha como OBRIGATÓRIAS para o agente trabalhar, apenas Idade, Gênero, Altura e Objetivos)
Tendo as respostas das perguntas em sua base de dados, o agente deve buscar nos arquivos inseridos, estudos e teorias que embasem suas decisões a partir daí. Mais uma vez, em hipótese alguma, o agente pode criar dados sobre nutrição ou treino, tudo deve ser embasado nos arquivos.
# Qual é o fluxo da integração?
1. Usuário envia pergunta no chat com o agente
2. O agente responde agradecendo o contato e explicando que, para seguir com o atendimento, precisa que o usuário responda algumas perguntas
3. Tendo as respostas das perguntas em mãos, o agente busca no seu banco de dados arquivos que o auxiliem na elaboração do plano de treino e dieta para o cliente (Este banco de dados também deve ser criado por nós para inserção dos arquivos)
4. Tendo convicção do treino e dieta elaborados, o agente retorna estes para o usuário e DEVE dizer pra ele o quão eficaz ela é com base na quantidade de parâmetros que foram inseridos pelo cliente e, além disso, é MUITO IMPORTANTE que o agente faça uma estimativa da eficácia da dieta e do treino com base na adesão do cliente.
    **Exemplo:**
   "Se tiver adesão total deste modelo de dieta e treino, atingirá seus objetivos, em média, em X meses"
   ou
   "Se tiver adesão parcial, isto é, adotar algumas partes, mas optar por não realizar outras, atingirá seus objetivos em X + Y meses"
5. Com isso feito, apenas espere a validação do usuário, se ele estiver satisfeito, vida que segue. Se não, explicite à ele o porque de tais recomendações com base nos estudos dos arquivos e dê a ele a opção de montar um novo plano de treino ou dieta com mais parâmetros para trabalhar.
