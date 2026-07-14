# Registro de decisões técnicas (ADR)

Este documento registra as decisões técnicas mais relevantes tomadas ao longo do projeto, incluindo o contexto que motivou cada uma, as alternativas consideradas e as consequências práticas. O objetivo é permitir que qualquer pessoa (inclusive eu mesmo, revisitando o projeto no futuro) entenda o porquê de cada escolha, não apenas o quê foi feito.

---

## ADR 001: Ingestão híbrida (download externo + Cloud Storage + BigQuery), em vez de streaming direto

**Contexto**: os dados de origem estão hospedados na plataforma Kaggle, como parte de uma competição, e precisam chegar ao BigQuery para processamento.

**Decisão**: baixar os dados via `kagglehub` dentro do ambiente Workbench, subir os arquivos brutos para o Cloud Storage, e então carregar do Cloud Storage para o BigQuery via `load_table_from_uri`.

**Alternativas consideradas**:
- Ler o CSV diretamente com pandas e inserir linha a linha no BigQuery: descartado por ser muito mais lento e por carregar o arquivo inteiro na memória do notebook antes de qualquer processamento.
- Apontar o BigQuery para ler o CSV diretamente do Kaggle: não é suportado nativamente, o BigQuery só carrega de fontes como Cloud Storage, Drive ou upload direto.

**Consequências**: um passo intermediário a mais (upload para o Cloud Storage), mas o carregamento para o BigQuery se torna rápido e não depende da memória do notebook. Também deixa o dado bruto persistido de forma independente do ambiente de desenvolvimento, disponível para reprocessamento futuro sem precisar baixar do Kaggle de novo.

---

## ADR 002: Datasets separados por camada (fraud_bronze, fraud_silver, fraud_gold), em vez de um único dataset com prefixos de tabela

**Contexto**: a arquitetura Medallion exige isolar dado bruto, dado tratado e dado pronto para consumo.

**Decisão**: criar três datasets distintos no BigQuery, um por camada, em vez de um dataset único com tabelas nomeadas `bronze_x`, `silver_x`, `gold_x`.

**Alternativas consideradas**: dataset único com prefixo de nome de tabela indicando a camada.

**Consequências**: a separação por dataset permite controle de permissão granular via IAM no nível de cada camada (por exemplo, restringir acesso de leitura só à Gold para um time de negócio, sem expor Bronze ou Silver). Isso não seria possível de forma limpa com prefixo de nome dentro de um dataset único.

---

## ADR 003: kagglehub em vez do kaggle CLI clássico

**Contexto**: era necessário escolher uma ferramenta para autenticar e baixar os dados da competição a partir do Kaggle.

**Decisão**: usar a biblioteca `kagglehub`, chamada diretamente dentro do notebook Python.

**Alternativas consideradas**: `kaggle` CLI clássico, executado via terminal, com download de `.zip` e descompactação manual.

**Consequências**: o `kagglehub` gerencia cache e descompactação automaticamente, retornando direto o caminho local dos arquivos prontos. Em contrapartida, ele baixa para uma pasta de cache local do ambiente, não para o Cloud Storage diretamente, exigindo um passo explícito de upload posterior (ver ADR 001).

---

## ADR 004: Autenticação via Legacy API Key do Kaggle

**Contexto**: a conta Kaggle do projeto passou a oferecer dois formatos de token: um novo, exibido diretamente na tela, e um formato legado, baixado como arquivo `kaggle.json`.

**Decisão**: usar o formato Legacy API Key.

**Motivo**: a biblioteca `kagglehub`, assim como o `kaggle` CLI, espera as credenciais no formato clássico (`username` e `key`). O token novo não é compatível com essas bibliotecas no momento da implementação deste projeto.

**Consequências**: nenhuma prática além do passo extra de localizar a opção "Legacy API Credentials" na conta Kaggle, que não é a opção mais visível na interface atual.

---

## ADR 005: Permissões de IAM concedidas de forma granular, em vez de papel amplo (Owner/Editor)

**Contexto**: a conta de serviço padrão da instância Workbench nasce com permissões restritas, e precisou de acesso explícito a Cloud Storage e BigQuery.

**Decisão**: conceder `Storage Object Admin` no nível do bucket específico do projeto, e `BigQuery Job User` mais `BigQuery Data Editor` no nível do projeto.

**Alternativas consideradas**: conceder papel `Editor` ou `Owner` do projeto à conta de serviço, o que resolveria todos os erros de permissão de uma vez.

**Consequências**: mais passos de configuração no início (precisou resolver erros 403 separadamente para Storage e para BigQuery), mas segue o princípio de menor privilégio, o que é considerado boa prática de segurança e é um ponto observável em entrevista técnica.

---

## ADR 006: Adiamento da amplificação sintética de volume de dados

**Contexto**: o volume bruto do dataset (aproximadamente 1,26 GB) é pequeno demais para gerar diferença perceptível de custo de processamento no BigQuery, o que limitava a demonstração prática de otimização de query por bytes escaneados.

**Decisão**: adiar a estratégia de replicação com jitter estatístico para uma fase futura do projeto, priorizando primeiro o ciclo completo de deploy em nuvem e modelagem de Machine Learning.

**Alternativas consideradas**: amplificar o volume imediatamente via replicação controlada (linhas duplicadas com variação de ruído gaussiano e IDs novos), antes de seguir para a Silver.

**Consequências**: a comparação de custo de query por volume de bytes ainda é demonstrável mesmo sem amplificação (a diferença entre `SELECT *` e seleção de colunas específicas já mostrou uma diferença de mais de 100 vezes em bytes processados, mesmo no volume original). A amplificação fica registrada como próximo passo, não como bloqueio do projeto.

---

## ADR 007: Conversão explícita de colunas numéricas para STRING na camada Silver

**Contexto**: diversas colunas do dataset (`card1`, `card2`, `card3`, `card5`, `addr1`, `addr2`, `M1` a `M9`) são armazenadas como número na Bronze, mas representam identificadores ou flags categóricas, não quantidades.

**Decisão**: aplicar `CAST` explícito para STRING nessas colunas na criação da Silver.

**Alternativas consideradas**: manter o tipo original inferido automaticamente pelo BigQuery (`autodetect`), e tratar a questão apenas na fase de modelagem.

**Consequências**: previne operações matemáticas sem sentido sobre identificadores (média ou ordenação de um código de cartão, por exemplo) em qualquer consulta feita a partir da Silver, não só na modelagem. Torna a camada mais segura para uso por qualquer pessoa que venha a consultar essas tabelas depois.

---

## ADR 008: LEFT JOIN entre train_transaction e train_identity, preservando nulos

**Contexto**: apenas 24,4% das transações têm dado de identity associado. Era necessário decidir como tratar essa ausência no momento do join.

**Decisão**: usar `LEFT JOIN`, com `train_transaction` como tabela mestre, preservando todas as 590.540 linhas e mantendo `NULL` explícito nos campos de identity ausentes.

**Alternativas consideradas**: `INNER JOIN`, que descartaria as 446.307 transações sem identity.

**Consequências**: essa decisão se mostrou crítica para o projeto. A investigação posterior revelou que a ausência de dado de identity está fortemente associada a uma taxa de fraude menor (2,09% contra 7,85% com dado presente), um dos achados mais importantes da EDA. Se tivéssemos usado `INNER JOIN`, esse padrão nunca teria sido descoberto, e o modelo perderia acesso a mais de 75% da base de treino.

---

## ADR 009: Exclusão das colunas V1 a V339 da camada Silver

**Contexto**: essas 339 colunas são features numéricas já engenheiradas pela Vesta, sem documentação pública de significado individual.

**Decisão**: não incluir essas colunas na Silver nesta fase do projeto. Serão trazidas diretamente da Bronze quando a fase de modelagem exigir.

**Alternativas consideradas**: incluir todas via `SELECT * EXCEPT(...)`, mantendo a Silver com a tabela completa desde já.

**Consequências**: mantém a Silver enxuta e fácil de auditar visualmente durante a fase de reconhecimento do dataset, evitando gastar esforço de documentação em colunas cujo tratamento ainda não foi decidido. O custo é que a Gold precisará buscar essas colunas da Bronze em vez da Silver, exigindo um join adicional na fase de modelagem.

---

## ADR 010: Uso de mediana e percentis em vez de média para investigar TransactionAmt

**Contexto**: o valor da transação tem distribuição de cauda longa, com poucos valores muito altos distorcendo a média.

**Decisão**: usar `APPROX_QUANTILES` para calcular mediana, percentil 25, 75 e 95, em vez de depender apenas de `AVG`.

**Consequências**: essa escolha foi o que permitiu identificar que a média sozinha mascarava o comportamento real da distribuição. A investigação subsequente, cruzando `TransactionAmt` com `ProductCD`, revelou um efeito de agregação enganosa que não teria sido percebido olhando só a média geral da base.

---

## ADR 011: Criação da feature razao_valor_produto em vez de usar TransactionAmt bruto

**Contexto**: a investigação de `TransactionAmt` cruzada com `ProductCD` revelou que cada produto opera em uma escala de valor muito diferente (mediana de transação legítima variando de 30,78 no produto C a 125,00 no produto R), e que a fraude é sistematicamente mais cara que a transação legítima dentro de cada produto individualmente, mas esse padrão ficava mascarado quando a base era analisada de forma agregada.

**Decisão**: propor, para a camada Gold, uma feature calculada como a razão entre o valor da transação e a mediana histórica daquele `ProductCD` específico, em vez de usar o valor bruto como principal sinal de valor.

**Consequências**: essa feature deve capturar melhor o conceito de "valor atípico para aquele contexto específico", em vez de um valor atípico em termos absolutos, que mistura escalas incomparáveis entre produtos diferentes.

---

## ADR 012: Dois caminhos de deploy planejados (BigQuery ML e endpoint Python), em vez de escolher apenas um

**Contexto**: existem duas abordagens de mercado para colocar um modelo de classificação em produção: treinar e servir via SQL dentro do próprio BigQuery, ou treinar com bibliotecas Python e servir via API/endpoint dedicado.

**Decisão**: implementar as duas abordagens neste projeto, em sequência.

**Alternativas consideradas**: escolher apenas a abordagem mais alinhada ao perfil de vaga que motivou o projeto (endpoint Python, mais próximo do que se espera de um Machine Learning Engineer).

**Consequências**: mais tempo de desenvolvimento, mas cobertura de dois perfis de vaga distintos (Analytics Engineer/Cientista de Dados próximo de BI, que valoriza BigQuery ML, e Machine Learning Engineer/Cientista de Dados Pleno-Sênior, que valoriza deploy via API própria).

---

## ADR 013: SQL desenvolvido e testado no BigQuery Studio, consolidado depois no notebook

**Contexto**: era necessário decidir onde escrever e executar as transformações de Silver e Gold, entre o notebook Python (Workbench) e a interface do BigQuery Studio.

**Decisão**: rascunhar e validar cada query primeiro no BigQuery Studio, e só depois consolidar a versão final validada como código dentro do notebook, chamando a API do BigQuery.

**Consequências**: iteração mais rápida durante o desenvolvimento (testar pedaços de query isoladamente na interface), mantendo o resultado final como pipeline reproduzível e versionável, alinhado à prática de nunca deixar uma transformação de dado depender só de um clique manual não documentado.
