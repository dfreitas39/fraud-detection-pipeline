# Fraud Detection Pipeline (IEEE-CIS)

Pipeline de detecção de fraude em transações de e-commerce, construído em arquitetura Medallion (Bronze, Silver, Gold) no Google Cloud Platform, com foco em Machine Learning aplicado a prevenção de fraude.

Status atual: EDA concluída. Gold, modelagem e deploy em andamento.

## Contexto do problema

Fraude em marketplaces é um problema com trade off direto entre risco e conversão: bloquear demais reduz vendas legítimas, bloquear de menos gera prejuízo com chargebacks. Empresas de e commerce e fintech investem continuamente em modelos de classificação de risco que decidem, em tempo real ou próximo disso, se uma transação deve ser aprovada, bloqueada ou enviada para revisão manual.

Este projeto foi motivado pela análise de uma vaga real de Data Science Engineer voltada a prevenção de fraude em um marketplace latino americano, e foi desenhado para cobrir, de ponta a ponta, as competências mais cobradas nesse tipo de posição: análise exploratória investigativa, engenharia de features, modelagem de classificação com dados desbalanceados, deploy e conceitos de monitoramento de modelo em produção.

## Objetivo

Construir um classificador capaz de prever a probabilidade de uma transação ser fraudulenta, com métrica alvo definida antes da modelagem: maximizar recall (captura de fraude real) mantendo precisão suficiente para não gerar excesso de bloqueio de clientes legítimos. A métrica de avaliação principal será AUC-PR, não acurácia, dado o forte desbalanceamento de classes do problema (3,5% de fraude).

Objetivos secundários do projeto:
- Praticar arquitetura de dados em nuvem (GCP: Cloud Storage, BigQuery, Agent Platform Workbench)
- Demonstrar deploy de modelo por dois caminhos distintos: BigQuery ML (SQL) e endpoint em Python
- Documentar decisões técnicas e de negócio de forma auditável, incluindo limitações do dataset

## Fonte dos dados

Dataset da competição [IEEE-CIS Fraud Detection](https://www.kaggle.com/competitions/ieee-fraud-detection), disponibilizado pela Vesta Corporation via Kaggle.

Composição original:

| Arquivo | Linhas | Colunas | Descrição |
|---|---|---|---|
| train_transaction.csv | 590.540 | 394 | Dados da transação: valor, produto, cartão, endereço |
| train_identity.csv | 144.233 | 41 | Dados de dispositivo e sinais de identidade, presentes em apenas 24,4% das transações |
| test_transaction.csv | 506.691 | 393 | Base de teste da competição (sem rótulo) |
| test_identity.csv | 141.907 | 41 | Base de teste da competição (sem rótulo) |

Volume bruto total: aproximadamente 1,26 GB.

Taxa de fraude real na base de treino, confirmada via query: 3,5% (20.663 casos em 590.540 transações).

## Arquitetura

```
Kaggle (kagglehub)
   |
   v
Cloud Storage (bucket raw/)
   |
   v
BigQuery
   |-- fraud_bronze   dados brutos, tipagem automatica, sem regra de negocio
   |-- fraud_silver   tipagem corrigida, join, regras de negocio aplicadas
   |-- fraud_gold     features prontas para o modelo (em construcao)
   |
   v
Modelagem (BigQuery ML e/ou notebook Python com scikit-learn/XGBoost)
   |
   v
Deploy (ML.PREDICT via SQL e/ou endpoint Python)
```

Ambiente de desenvolvimento: instância do Agent Platform Workbench (antigo Vertex AI Workbench), tipo e2-standard-4, região southamerica-east1.

## Tratamento dos dados

### Camada Bronze

Carga direta dos CSVs do Cloud Storage para o BigQuery via `load_table_from_uri`, sem nenhuma transformação de negócio. Único tratamento aplicado é a tipagem automática do BigQuery (`autodetect=True`). Esta camada existe para preservar o dado exatamente como recebido, garantindo auditabilidade.

### Camada Silver

Regras de negócio formalizadas e aplicadas:

| Regra | Decisão tomada | Motivo |
|---|---|---|
| Identidade de cliente | Não existe ID de cliente explícito no dataset. Não foi reconstruído artificialmente. | Evita tratar uma aproximação como fato garantido |
| Rótulo isFraud | Aceito como veio, sem correção | Segundo a documentação da competição, o rótulo pode ser propagado retroativamente a partir de um chargeback confirmado, o que significa que o modelo aprende "comportamento associado a cliente fraudulento", não necessariamente "esta transação específica é fraude". Limitação documentada explicitamente |
| TransactionDT | Mantido como contador relativo de segundos, sem conversão para data de calendário | O ponto de referência temporal não é divulgado oficialmente pela Vesta; converter para data geraria conclusão sazonal fictícia |
| Colunas numéricas categóricas | card1, card2, card3, card5, addr1, addr2 e M1 a M9 convertidas para STRING via CAST | Essas colunas são identificadores ou flags, não quantidades. Mantê las como número permitiria operações matemáticas sem sentido (média de código de cartão, por exemplo) |
| card4 | Confirmado como bandeira do cartão (Visa, Mastercard, Discover, American Express), já armazenado como STRING | Validado com consulta direta na coluna |
| Nulos em identity | Mantidos como nulo, sem imputação | A ausência de dado de identity provou ser, ela mesma, um sinal relevante (ver seção de análises) |
| Colunas V1 a V339 | Não incluídas nesta camada | Features anônimas geradas pela Vesta, sem documentação de significado individual. Serão tratadas de forma agregada na fase de modelagem, direto a partir da Bronze |
| Join transaction + identity | LEFT JOIN, com train_transaction como tabela mestre | Garante que nenhuma transação seja perdida por falta de dado de identity associado |

Validações aplicadas após a criação da Silver:
- Contagem de linhas preservada (590.540, idêntica à Bronze)
- Distribuição de tem_dado_identity conferida (144.233 com dado, 446.307 sem)
- Tipagem das colunas convertidas confirmada via INFORMATION_SCHEMA.COLUMNS

## Análises realizadas

EDA conduzida de forma guiada por hipótese de negócio, não exploratória genérica. Cada investigação partiu de uma pergunta específica sobre o que poderia diferenciar transação fraudulenta de legítima.

| Hipótese investigada | Resultado | Interpretação |
|---|---|---|
| Presença de dado de identity está associada a mais fraude? | 2,09% de fraude sem identity contra 7,85% com identity | Sinal forte. Confirmado robusto mesmo controlando por ProductCD, ou seja, não é um efeito indireto do tipo de produto |
| Taxa de fraude varia por tipo de produto (ProductCD)? | De 2,04% (produto W) a 12,28% (produto C, no subgrupo com identity) | Feature categórica isoladamente mais discriminativa encontrada até agora |
| Taxa de fraude varia por tipo de dispositivo (DeviceType)? | 10,17% em mobile contra 6,52% em desktop, dentro do subgrupo com identity | Sinal forte, mas com cobertura limitada a 24,4% da base |
| Taxa de fraude varia por bandeira do cartão (card4)? | De 2,60% (nulo) a 7,73% (Discover), passando por 3,43% a 3,48% em Visa e Mastercard | Sinal moderado. Discover tem amostra pequena (6.651 transações), leitura tratada com ressalva estatística |
| Valor da transação (TransactionAmt) difere entre fraude e legítima? | Na base agregada, o padrão pareceu inconsistente (mediana da fraude mais alta, mas valor máximo mais baixo) | Investigação aprofundada revelou a causa: efeito de agregação enganosa |
| TransactionAmt cruzado com ProductCD | Dentro de cada ProductCD individualmente, a fraude é sistematicamente mais cara que a legítima (mediana e percentil 95 maiores em todos os 5 produtos) | A mistura de escalas de valor entre produtos diferentes mascarava o padrão real na análise agregada. Levou à proposta de uma feature nova: razão entre o valor da transação e a mediana histórica daquele produto, em vez do valor bruto |

### Features candidatas para a camada Gold

| Feature | Cobertura da base | Tratamento planejado |
|---|---|---|
| tem_dado_identity | 100% | Binária |
| ProductCD | 100% | Categórica |
| DeviceType | 24,4% | Categórica, nulo tratado como categoria própria |
| card4 | 100% | Categórica |
| TransactionAmt (bruto) | 100% | Mantido, mas não como sinal principal isolado |
| razao_valor_produto (nova) | 100% | Numérica, calculada como TransactionAmt dividido pela mediana histórica do ProductCD correspondente |

## Modelagem

Ainda não iniciada. Planejamento:
- Baseline com regressão logística
- Modelo principal com XGBoost, tratando o desbalanceamento via class weights
- Validação temporal (não aleatória), respeitando a ordem cronológica de TransactionDT
- Explicabilidade via SHAP
- Métrica principal: AUC-PR

## Principais resultados

Seção a ser preenchida após a fase de modelagem.

## Limitações

- Ausência de ID de cliente explícito no dataset, exigindo cautela em qualquer feature que tente aproximar esse conceito
- Rótulo isFraud pode conter propagação retroativa de casos confirmados de chargeback, o que muda a interpretação do que o modelo está de fato aprendendo a prever
- TransactionDT não permite reconstrução de data calendário real, limitando análises de sazonalidade
- Colunas V1 a V339 não têm significado de negócio documentado publicamente, o que limita a interpretabilidade dessas variáveis específicas mesmo que estatisticamente relevantes
- Amostra pequena para a bandeira Discover (6.651 transações), o que reduz a confiabilidade estatística da taxa de fraude observada para esse grupo especificamente

## Próximos passos

1. Desenhar e criar a tabela Gold, incluindo a lógica de razao_valor_produto
2. Treinar baseline (regressão logística) e modelo principal (XGBoost)
3. Avaliar com validação temporal e métricas apropriadas para desbalanceamento
4. Aplicar SHAP para explicabilidade
5. Deploy via BigQuery ML (ML.PREDICT)
6. Deploy via endpoint Python (FastAPI ou Agent Platform endpoint)
7. Notebook de simulação de drift para demonstrar conceito de monitoramento em produção

## Como executar

Pré requisitos: conta GCP com billing ativo, projeto criado, APIs de BigQuery, Cloud Storage e Agent Platform habilitadas, credenciais Kaggle (Legacy API Key).

```bash
# dentro do ambiente Workbench, no notebook 01_eda.ipynb

pip install kagglehub --quiet

# autenticacao interativa
import kagglehub
kagglehub.login()

# download dos dados
path = kagglehub.competition_download('ieee-fraud-detection')

# upload para o Cloud Storage e carga da camada Bronze
# ver notebooks/01_eda.ipynb para o passo a passo completo
```

As queries de criação da Silver e da Gold estão documentadas em `src/` e podem ser executadas diretamente no BigQuery Studio ou via notebook.

## Tecnologias

- Python (pandas, google-cloud-storage, google-cloud-bigquery)
- Google Cloud Storage
- BigQuery (SQL, INFORMATION_SCHEMA, APPROX_QUANTILES)
- Google Cloud Agent Platform Workbench (Jupyter Lab)
- kagglehub
- scikit-learn e XGBoost (fase de modelagem, planejado)
- SHAP (explicabilidade, planejado)
