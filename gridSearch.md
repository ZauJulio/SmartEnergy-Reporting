# SOM - Grid Search

Especificações do experimento grid_search.

## Sumário

- [1. Conjunto de dados](#1-conjunto-de-dados)

- [2. Self-Organizing Maps](#2-self-organizing-maps)

- [3. Grid Search](#3-grid-search)

  - [3.1. Métricas](#31-métricas)

    - [3.1.1 Notas](#311-notas)

- [4. Definição do problema](#4-definição-do-problema)
  
  - [4.1. Considerações](#41-considerações)

- [**Referências**](#referências)

## 1. Conjunto de dados

O conjunto de dados utilizado no projeto atualmente corresponde a séries temporais de valores de potência energética, organizadas da seguinte forma: um conjunto de dias com valores em intervalos fixos de 1 minuto, logo (1440 minutos, n dias). O estudo principal foi realizado utilizando o corte das 20:00 às 23:59, (240 minutos, n dias). O conjunto atual tem cerca de 2 anos de amostras. Para este experimento, até o presente momento, foi considerado somente o conjunto das segundas-feiras, (240 minutos, 75 amostras(dias)).

|   time   | 2018-03-19 | 2018-03-26 | ... | 2019-09-09 | 2019-09-16 |
| :------: | :--------: | :--------: | :-: | :--------: | :--------: |
| 20:00:00 |   309.71   |   188.87   | ... |   199.35   |   143.96   |
| 20:01:00 |   311.4    |   188.74   | ... |   199.61   |   143.96   |
|   ...    |    ...     |    ...     | ... |    ...     |    ...     |
| 23:58:00 |   179.95   |   328.47   | ... |   379.54   |   413.32   |
| 23:59:00 |   179.46   |   328.5    | ... |   376.56   |   413.13   |

## 2. Self-Organizing Maps

O uso do SOM foi vinculado principalmente a clusterização não supervisionada dos dados, com o propósito de gerar labels para classificação supervisionada das amostras. Até o presente momento, os hiperparâmetros da rede são:

    Nº de linhas: 3;
    Nº de colunas: 3;
    Comprimento da entrada: 240 "minutos";
    Inicialização: aleatória;
    Nº de iterações: 100,000;
    Sigma: 1.0;
    Taxa de aprendizado inicial: 0.5;
    Função de vizinhança: gaussiana;
    Topologia: retangular;
    Distância de ativação: Euclidiana;

Implementação utilizada: [Minisom](https://github.com/JustGlowing/minisom)

Comumente a rede é treinada com amostras de 2018 e é utilizada para clusterizar amostras de 2019. Com a rede treinada, as amostras utilziadas na fase de treino são mapeadas e separadas em clusters e a [regressão](https://github.com/labepi/smartEnergy/blob/master/src/util/regression.py) é utilizada para estimar um modelo para este novo conjunto.

Os pesos sinápticos da rede também foram utilizados esporadicamente como uma "regressão" das amostras de treino.

Em seguida coletamos [métricas](https://github.com/labepi/smartEnergy/blob/master/src/util/metrics.py) como [MAE](https://github.com/labepi/smartEnergy/blob/master/src/util/metrics.py#L31) e [RMSE](https://github.com/labepi/smartEnergy/blob/master/src/util/metrics.py#L91) do modelo e as adicionamos ao modelos, gerando uma série de mesmo comprimento das amostras, com a finalidade de separar os outliers encontrados no [K-Means](https://github.com/labepi/ecic_gerince/blob/master/chapters/3_results.tex).

## 3. Grid Search

O experimento consiste em buscar uma rede SOM que consiga agrupar as amostras da melhor maneira possível, com a finalidade de gerar labels para uma classificação supervisionada. Para isso explorei o espaço hiperparamétrico buscando no produto entre os seguintes parâmetros:

    Nº de linhas: [3, 5, 6, 9, 10];
    Nº de colunas: [3, 5, 6, 9, 10];
    Nº de iterações: [2000, 5000, 10000, 20000];
    Inicialização: [Aleatória, PCA];
    Sigma: [0.5, 1.0, 1.5, 2.0];
    Taxa de aprendizado inicial: [0.5, 1.0, 1.5, 2.0];
    Função de vizinhança: [Gaussiana, Mexican hat, Bubble, Triangle];
    Topologia: [Retangular, Hexagonal];
    Distância de ativação: [Euclideana, Cosine, Manhattan, Chebyshev];

Um total de 102,400 combinações. Como entrada, esses modelos receberam o conjunto das segundas-feiras no corte das 20:00 às 23:59, 240 minutos, com um total de 75 amostras(Dias). Para realizar a validação cruzada dos modelos, o conjunto de dados foi embaralhado de 5 maneiras diferentes com as seguintes sementes: 42, 58, 16, 75, 14. Totalizando 512,000 modelos.

Em seguida foi realizada a divisão do conjunto em 50% para treino e 50% para teste(validação). Por fim os [modelos](https://github.com/labepi/smartEnergy/blob/master/src/ModelManager/Core.py) foram treinados paralelamente e identificados em um [banco de dados](https://github.com/labepi/smartEnergy/blob/master/src/Adapter/Adapter.py).

### 3.1 Métricas

A coleta de métricas foi relizada utilizando o conjunto de teste e os modelos treinados como parâmetros para as seguintes [métricas](https://github.com/labepi/smartEnergy/blob/master/src/ModelManager/Metrics.py#L51):

- Métricas por [cluster](https://github.com/labepi/smartEnergy/blob/master/src/util/som.py):
  - Nº de amostras;
  - Média do cluster;
  - Média da variância;
  - Média do desvio padrão;
  - Métricas do [produto interno](https://github.com/labepi/smartEnergy/blob/master/src/util/numlib.py#L193):
    - Média do produto interno do cluster;
    - Variância;
    - Desvio padrão;
  - Métricas clássicas:
    - Média do [QE](https://github.com/JustGlowing/minisom/blob/master/minisom.py#L488);
    - Média do [RMSE](https://github.com/labepi/smartEnergy/blob/master/src/util/metrics.py#L91);
    - Média do [MAE](https://github.com/labepi/smartEnergy/blob/master/src/util/metrics.py#L31);
- Métricas para clusterização não supervisionada(Valor único):
  - [Silhouette Score](https://scikit-learn.org/stable/modules/clustering.html#silhouette-coefficient);
  - [Davies-Bouldin Score](https://scikit-learn.org/stable/modules/clustering.html#davies-bouldin-index);
  - [Calinski-Harabasz Score](https://scikit-learn.org/stable/modules/clustering.html#calinski-harabasz-index);

#### 3.1.1 Notas

- As métricas por cluster contém **n** valores, tal que **n** corresponde ao tamanho do map/número de clusters;
- As métricas de clusterização não supervisionada recebem um único valor pois avaliam o desempenho geral do modelo de clusterização;
  - Silhouette Score:
    - _"The best value is 1 and the worst value is -1. Values near 0 indicate overlapping clusters. Negative values generally indicate that a sample has been assigned to the wrong cluster, as a different cluster is more similar."_ - [sklearn.metrics.silhouette_score - Docs](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.silhouette_score.html#sklearn.metrics.silhouette_score);
    - [Formulação Matemática](https://scikit-learn.org/stable/modules/clustering.html#silhouette-coefficient);
  - Davies-Bouldin Score:
    - _"The minimum score is zero, with lower values indicating better clustering."_ - [sklearn.metrics.davies_bouldin_score - Docs](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.davies_bouldin_score.html#sklearn.metrics.davies_bouldin_score);
    - [Formulação Matemática](https://scikit-learn.org/stable/modules/clustering.html#id33);
  - Calinski-Harabasz Score:
    - _"The score is defined as ratio between the within-cluster dispersion and the between-cluster dispersion."_ - [sklearn.metrics.calinski_harabasz_score](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.calinski_harabasz_score.html#sklearn.metrics.calinski_harabasz_score);
    - [Formulação Matemática](https://scikit-learn.org/stable/modules/clustering.html#id29);

## 4. Definição do problema

O problema atual do projeto é que a partir destas métricas, é possível avaliar de forma quantitativa a performance e a qualidade dos modelos, a fim de encontrar o modelo mais eficiente possível para o conjunto de dados atual. Caso não seja possível, seguindo essas métricas, como avaliar qualitativamente os modelos. Por fim, existem outras métricas que possam avaliar esse modelo de forma quantitativa ou qualitativa, com certa precisão, ou esse tipo de análise é inválida para o algoritmo.

### 4.1. Considerações

- Utilizando as métricas do produto interno e do cluster, é possível ponderar sobre elas e convergi-las em uma única métrica?
- Considerando o algoritmo utilizado, é válido utilizar somente as métricas definidas especificamente para ele(Erro de quantização, Silhouette Score, Davies-Bouldin Score, Calinski-Harabasz Score)?

## **Referências**

Repositórios:

- [GridSearch](https://github.com/labepi/smartEnergy/blob/master/experiments/grid_search/main.py)
- [Minisom](https://github.com/JustGlowing/minisom)
- [Erro de quantização](https://github.com/JustGlowing/minisom/blob/master/minisom.py#L488)

Métricas para clusterização não supervisionada:

- [Silhouette Score](https://scikit-learn.org/stable/modules/clustering.html#silhouette-coefficient);
- [Davies-Bouldin Score](https://scikit-learn.org/stable/modules/clustering.html#davies-bouldin-index);
- [Calinski-Harabasz Score](https://scikit-learn.org/stable/modules/clustering.html#calinski-harabasz-index);

Métricas clássicas:

- [QE](https://github.com/JustGlowing/minisom/blob/master/minisom.py#L488)
- [RMSE](https://github.com/labepi/smartEnergy/blob/master/src/util/metrics.py#L91)
- [MAE](https://github.com/labepi/smartEnergy/blob/master/src/util/metrics.py#L31)

Resultados das últimas iterações:

- [SmartEnergy - Grid Search Results](https://github.com/labepi/smartEnergy/tree/master/experiments/grid_search/results)
