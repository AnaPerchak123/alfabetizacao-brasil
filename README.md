# Alfabetização no Brasil

## Objetivo

Analisar diferenças na alfabetização infantil e avaliar se informações educacionais, territoriais e socioeconômicas permitem prever a classificação de alunos do 2º ano do Ensino Fundamental.

O projeto reúne preparação de dados, análise exploratória e comparação de modelos supervisionados, com foco no apoio à análise de políticas públicas educacionais.

## 1. Entendimento do negócio

### Problema de negócio

Gestores públicos precisam acompanhar os resultados de alfabetização, identificar desigualdades e avaliar o cumprimento das metas educacionais. Além de descrever o cenário, é importante verificar até que ponto os dados disponíveis permitem antecipar dificuldades de aprendizagem.

### Por que analisar a alfabetização?

A alfabetização nos primeiros anos é uma etapa essencial da trajetória escolar. Sua análise pode apoiar o acompanhamento das redes de ensino e orientar investigações sobre diferenças entre territórios.

### Quem pode utilizar a análise?

- **Gestão educacional:** acompanhamento dos resultados e das metas.
- **Secretarias de Educação:** investigação de diferenças entre municípios e redes.
- **Equipes de dados:** avaliação da qualidade das bases e dos limites dos modelos.

> **Insight:** diferenças territoriais ajudam a entender o contexto educacional, mas não explicam, sozinhas, a situação de cada aluno.

## 2. Definição da variável-alvo

### Qual resultado o modelo procura prever?

A coluna `alfabetizado`, com os códigos da fonte:

- **0:** não alfabetizado.
- **1:** alfabetizado.

Entre as avaliações válidas, a classificação foi conferida com a proficiência: valores abaixo de 743 correspondem à classe 0; valores a partir de 743, à classe 1.

### Quais alunos entram na modelagem?

Alunos do 2º ano com presença registrada, prova preenchida, proficiência disponível e classificação válida.

Registros sem avaliação válida permanecem na base bruta, mas não são usados como exemplos de não alfabetização. O desempenho do modelo não demonstra sua capacidade de prever o resultado dos alunos ausentes.

### Como foi evitado o vazamento da resposta?

A proficiência não entra como preditor, pois determina diretamente a classificação. Presença, preenchimento, identificadores, resultados agregados do mesmo ano e metas também ficam fora das variáveis de entrada.

## 3. Base de dados

| Fonte | Informações utilizadas | Referência |
|---|---|---|
| Avaliação da Alfabetização — Inep, via Base dos Dados | Alunos, resultados municipais e estaduais, metas e dicionário | Avaliações de 2023 e 2024; referências das metas preservadas |
| Diretórios Brasileiros — Base dos Dados | Código e nome do município, UF e região | Diretório disponível na extração |
| Atlas do Desenvolvimento Humano — via Base dos Dados | População, IDHM, renda, analfabetismo adulto e saneamento | 2010 |

### Amostra utilizada

A seleção usa um hash de ano, município, escola e aluno para extrair aproximadamente 10% dos registros. O resultado da avaliação não participa da seleção. A reprodução da amostra depende da manutenção do conteúdo da fonte.

| Etapa | Registros |
|---|---:|
| Amostra bruta | 386.367 |
| Sem avaliação válida | 51.012 |
| Base elegível | 335.355 |

A amostra não garante representatividade de todos os municípios. As taxas calculadas sobre alunos são não ponderadas, salvo indicação contrária; o peso disponibilizado pela fonte foi preservado, mas não utilizado nas métricas apresentadas.

## 4. Preparação dos dados

Os dados foram organizados em três camadas:

- **Bronze:** cópias das fontes consultadas, incluindo a amostra de alunos.
- **Silver:** padronização, seleção dos registros elegíveis e integração das bases.
- **Gold:** base analítica, conjuntos de treino, validação e teste e tabelas de contexto.

A preparação foi reconstruída neste projeto. A implementação realiza extrações em lote; não inclui streaming nem orquestração contínua.

### Validações realizadas

- Verificação de chaves incompletas e duplicidades.
- Conferência entre classificação e proficiência.
- Validação de relacionamentos muitos-para-um.
- Conferência da quantidade de alunos após os cruzamentos.
- Identificação de registros sem correspondência e valores ausentes.
- Separação entre metas não aplicáveis e metas não localizadas.

Todos os alunos elegíveis receberam identificação territorial. Houve 18 registros sem resultado municipal, 2.234 sem resultado estadual e 126 sem indicadores do Atlas. Os alunos foram mantidos, com as ausências preservadas.

As metas municipais foram vinculadas somente à rede municipal. As metas estaduais e nacionais foram organizadas em tabelas separadas, preservando o ano da meta e a referência da fonte.

## 5. Análise exploratória

A exploração que orientou a modelagem utiliza somente o treino de 2023.

### Distribuição da alfabetização

No treino, 58,5% dos alunos são alfabetizados e 41,5% não alfabetizados. As duas classes possuem volume relevante; a acurácia isolada não é suficiente para avaliar o modelo.

### Diferenças territoriais e educacionais

- A proporção de alfabetizados varia de 51,4% no Norte a 67,9% no Sul, uma diferença aproximada de 16,5 pontos percentuais na amostra de treino.
- A rede estadual apresenta 61,8%, e a municipal, 58,2%. Os grupos têm volumes e contextos distintos.
- A taxa passa de 49,7% no quartil de menor IDHM para 60,4% no de maior IDHM. Os quartis são definidos pelos municípios do treino e não correspondem às categorias oficiais do índice.

### Relações entre os indicadores

IDHM e renda apresentam correlação de Spearman de 0,95. IDHM e analfabetismo adulto apresentam correlação de -0,90. Essa sobreposição precisa ser considerada na interpretação das variáveis.

> **Insight:** o contexto municipal está associado ao resultado observado, mas essas associações não demonstram causalidade nem descrevem as condições individuais das famílias.

## 6. Modelagem supervisionada

### Variáveis utilizadas

**Numéricas:** população, IDHM, renda per capita, analfabetismo adulto, acesso à água encanada e inadequação de água e esgoto, todos referentes a 2010.

**Categóricas:** rede de ensino e UF.

Alunos da mesma rede e município compartilham os atributos utilizados. O modelo representa principalmente diferenças de contexto, com pouca informação individual.

### Pré-processamento

As pipelines do scikit-learn integram:

- Imputação de numéricos pela mediana, com indicador de ausência.
- Tratamento de categorias ausentes.
- Codificação das categorias com OneHotEncoder.
- Padronização numérica para a regressão logística.
- Classificador.

O pré-processamento é ajustado somente no treino. As categorias desconhecidas são tratadas pelo encoder sem interromper a previsão.

### Separação dos dados

| Conjunto | Ano | Alunos | Municípios |
|---|---:|---:|---:|
| Treino | 2023 | 122.978 | 3.843 |
| Validação | 2023 | 27.195 | 961 |
| Teste | 2024 | 185.182 | 5.434 |

Treino e validação não compartilham municípios. O teste avalia a generalização temporal para 2024, mas pode conter municípios presentes no treino. A divisão e os modelos utilizam semente 42, quando aplicável.

### Algoritmos e seleção

Foram comparados DummyClassifier, regressão logística e Random Forest. Configurações adicionais e limiares de risco foram avaliados na validação, pelo F1 da classe não alfabetizado.

A configuração selecionada foi **regressão logística com C=0,1 e limiar de 0,35 para a probabilidade da classe 0**. O modelo permaneceu treinado apenas no treino original. Não houve novo ajuste a partir dos resultados de teste.

A seleção utiliza uma divisão fixa por município, sem validação cruzada. O resultado na validação é usado para escolha, e não como avaliação final independente.

## 7. Resultados no teste de 2024

| Indicador | Modelo selecionado |
|---|---:|
| Precisão dos alertas de não alfabetização | 43,61% |
| Recall dos não alfabetizados | 86,54% |
| F1 dos não alfabetizados | 58,00% |
| Acurácia | 49,45% |
| Acurácia balanceada | 55,47% |
| ROC AUC de risco | 0,6103 |
| Average Precision de risco | 0,5073 |
| Alunos sinalizados | 80,01% |

### O que esses resultados significam?

O modelo identificou 64.620 alunos não alfabetizados e deixou de identificar 10.050. Também gerou 83.553 falsos alertas e classificou corretamente 26.959 alunos alfabetizados.

O recall elevado foi acompanhado por muitos alertas: cerca de 80% dos alunos foram sinalizados. A referência que sinaliza todos alcança F1 de 57,47%, apenas 0,53 ponto percentual abaixo do modelo. A referência majoritária alcança acurácia de 59,68%, superior à do modelo, mas não identifica nenhum aluno da classe 0.

> **Insight:** o modelo apresenta discriminação limitada. O recall de 86,54% não equivale à acurácia e não deve ser apresentado isoladamente como sucesso da solução.

## 8. Interpretação e aplicação estratégica

### Quais variáveis contribuem para as previsões?

A importância por permutação foi calculada na validação, com cinco repetições e ROC AUC como métrica.

| Variável | Queda média na ROC AUC |
|---|---:|
| UF | 0,13465 |
| IDHM de 2010 | 0,03344 |
| Analfabetismo adulto de 2010 | 0,01736 |
| Renda per capita de 2010 | 0,00351 |

Esses valores não representam percentuais de explicação ou efeitos causais. A permutação pode romper relações entre atributos correlacionados; a importância deve ser interpretada em conjunto com a análise exploratória.

### Quais municípios apresentam maior risco?

Foram agregados os escores do teste por município. A visualização considerou 311 municípios com pelo menos 100 alunos na amostra. O critério reduz a exposição de grupos pequenos, mas não garante representatividade.

O ranking descreve escores do modelo nesse recorte. Não constitui uma lista oficial de prioridade, e a média dos escores não deve ser tratada como taxa calibrada de não alfabetização.

### Quais regiões apresentam resultados semelhantes?

Na descrição do treino, Sudeste e Centro-Oeste apresentam proporções próximas, de 59,2% e 58,5%. Essa proximidade em um indicador não comprova semelhança de todo o perfil educacional. Não foi realizado agrupamento multivariado de regiões.

### Como os resultados se comparam às metas?

A análise retrospectiva utiliza a taxa e a meta publicadas para a rede municipal em 2024, na referência 2024 da fonte, sem substituir os resultados pela taxa da amostra.

- **2.788 municípios** atingiram ou superaram a meta: 53,3% dos comparáveis.
- **2.444 municípios** ficaram abaixo: 46,7% dos comparáveis.
- **120 municípios** não tinham dados suficientes para comparação.

O denominador é de 5.232 municípios comparáveis. A tabela completa contém 5.352 registros municipais nessa referência e não deve ser confundida com a cobertura da amostra de alunos.

### É possível prever o cumprimento de metas futuras?

Esta versão não valida uma previsão de metas futuras. Essa aplicação exigiria histórico adicional, informações disponíveis antes de cada avaliação e validação própria no nível municipal. As metas apoiam o acompanhamento descritivo nesta entrega.

## 9. Limitações e evoluções

- Indicadores socioeconômicos de 2010 têm defasagem de 13 a 14 anos em relação às avaliações.
- Poucas características individuais limitam a diferenciação entre alunos do mesmo contexto.
- A amostra não assegura representatividade municipal, e os resultados apresentados não utilizam pesos amostrais.
- A cobertura de municípios varia entre anos.
- A seleção de parâmetros e limiar utiliza uma única validação; não foram estimados intervalos de confiança.
- Os escores não passaram por avaliação específica de calibração.
- A elevada quantidade de falsos alertas limita o uso operacional.

Como evolução, recomenda-se integrar dados mais recentes de infraestrutura escolar, frequência e trajetória de aprendizagem, respeitando a disponibilidade anterior ao resultado. Também são necessários avaliação ponderada, validações temporais adicionais e critérios de decisão alinhados à capacidade de intervenção.

## 10. Organização do repositório

O notebook é a implementação principal e reúne o fluxo completo. A organização prevista para os demais arquivos é:

| Caminho | Conteúdo |
|---|---|
| `notebooks/01_analise_modelagem_alfabetizacao.ipynb` | Notebook principal, com código e resultados |
| `data/` | Orientações sobre Bronze, Silver e Gold e acesso às bases |
| `src/` | Funções organizadas por preparação, modelagem, avaliação e visualização — publicação pendente |
| `images/` | Gráficos exportados — publicação pendente |
| `reports/` | Tabelas de resultados, relatório e apresentação — publicação pendente |
| `requirements.txt` | Versões das bibliotecas — publicação pendente |

As etapas abaixo descrevem a execução do notebook principal. Não é necessário que `src/` esteja preenchida para executar essa implementação.

## 11. Como executar

### Ambiente recomendado

**Google Colab**, ambiente utilizado no desenvolvimento. O arquivo usa `google.colab` para autenticação e downloads e caminhos em `/content`. Esta versão não executa diretamente no VS Code local apenas com a instalação das bibliotecas.

### Requisitos

- Conta Google com acesso ao Colab.
- Projeto próprio no Google Cloud com permissão para executar consultas no BigQuery e acesso às fontes públicas utilizadas.
- Bibliotecas compatíveis com as versões registradas no projeto.

O BigQuery Sandbox pode ser utilizado dentro de suas limitações e cotas. Em projetos com faturamento ativo, aplicam-se as condições de cobrança do projeto. As consultas do notebook possuem limite de 1 GiB de processamento faturável por consulta.

### Passo a passo

1. Abra o notebook principal no Google Colab.
2. Se necessário, instale as versões indicadas no `requirements.txt` e reinicie a sessão antes da execução.
3. Na célula de configuração, substitua `PROJECT_ID` pelo ID do seu próprio projeto. O projeto da autora não concede acesso a outros usuários.
4. Execute a célula de autenticação e conclua a autorização com a conta que possui acesso ao projeto.
5. Execute as demais células na ordem, do início ao fim.
6. Confira os registros e as métricas de referência apresentados neste README.
7. Salve o notebook executado e exporte os arquivos antes de encerrar a sessão.

### Arquivos gerados

O diretório `/content/alfabetizacao-brasil` recebe as bases em Parquet, gráficos em PNG/SVG, tabelas em CSV e o modelo em Joblib. A pasta `/content` é temporária. A execução também disponibiliza backups em ZIP em etapas do fluxo.

Para aplicar o modelo salvo, use `predict_proba`, selecione a probabilidade da classe 0 e aplique o limiar armazenado de 0,35. Usar somente `predict()` não reproduz essa regra de decisão.

### Reprodução

A execução completa no Colab foi confirmada pela autora. O notebook entregue contém 45 células de código executadas, sem erros registrados, e resultados finais compatíveis com os valores documentados. O histórico contém reexecuções e não comprova uma execução única em sessão limpa.

O ambiente original registrou Python 3.13, pandas 2.2.3 e scikit-learn 1.6.1. As versões completas devem acompanhar o repositório. Alterações nas fontes ou no ambiente podem modificar os resultados.

## 12. Materiais complementares

- **Relatório técnico:** publicação pendente.
- **Apresentação de apoio:** publicação pendente.
- **Vídeo executivo:** link pendente.
- **Bases e backups no Google Drive:** link pendente.

## Conclusão

A análise identificou diferenças territoriais e associações entre alfabetização e contexto socioeconômico. A comparação com metas mostrou que 46,7% dos municípios com dados comparáveis ficaram abaixo da meta de 2024.

O modelo individual apresentou desempenho limitado, com muitos falsos alertas. A principal contribuição do projeto é uma base analítica reproduzível, acompanhada de uma avaliação transparente do que os dados permitem concluir e do que ainda precisa evoluir antes de uma aplicação operacional.

## Autoria e contexto

**Ana Carolina Perchak dos Santos**

Projeto desenvolvido no contexto da pós-graduação, no Tech Challenge da Fase 3, e organizado para apresentação em portfólio.

## Referências

- [Avaliação da Alfabetização — Base dos Dados](https://basedosdados.org/dataset/073a39d4-89cf-4068-b1e8-34ed0d9c0b72)
- [Atlas do Desenvolvimento Humano — Base dos Dados](https://basedosdados.org/dataset/cbfc7253-089b-44e2-8825-755e1419efc8)
- Diretório territorial: `basedosdados.br_bd_diretorios_brasil.municipio`.
- [Google Colab](https://research.google.com/colaboratory/faq.html)
- [BigQuery Sandbox](https://docs.cloud.google.com/bigquery/docs/sandbox)
- [Prevenção de erros e vazamento de dados — scikit-learn](https://scikit-learn.org/stable/common_pitfalls.html)

