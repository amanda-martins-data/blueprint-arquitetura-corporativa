# 04. Modelo Fisico

## Formato de armazenamento e camadas

Segue o medallion (Bronze/Silver/Gold) e o formato Parquet ja
justificados no [ADR 0001](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0001-data-lake-vs-warehouse-vs-lakehouse.md)
e no particionamento por camada do [ADR 0002](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0002-particionamento-bronze-ingestion-vs-silver-measured.md).
O que muda neste blueprint, em relacao ao portfolio anterior, e a
dimensao multi-tenant - uma decisao nova, especifica deste cenario.

## Decisao nova: isolamento fisico de dados entre Municipios

### Contexto

RN1 exige isolamento total entre clientes. RN6 exige que o custo de
infraestrutura escale com o numero de clientes, nao seja um custo
fixo alto por cliente. Essas duas restricoes empurram para direcoes
opostas: isolamento maximo tende a significar infraestrutura
dedicada por cliente (cara), enquanto custo proporcional tende a
significar infraestrutura compartilhada (risco de vazamento entre
tenants se malfeita).

### Opcoes consideradas

**Opcao A - Banco de dados fisico separado por Municipio.** Isolamento
mais forte possivel - impossivel vazar dado entre clientes porque
nem estao no mesmo banco. Descartada: com 50 clientes projetados,
significa 50 bancos para uma equipe de 2-3 pessoas (RN5) manter,
e custo fixo por cliente que viola RN6 diretamente.

**Opcao B - Particionamento fisico por municipio_id, sem controle de
acesso adicional na camada de consumo.** Mais barata, mas depende de
toda query em toda ferramenta downstream lembrar de filtrar por
municipio_id - um unico erro humano ou um novo relatorio esquecido
do filtro vaza dado entre tenants. Descartada por ser fragil demais
para um requisito contratual (RN1).

**Opcao C - Particionamento fisico por municipio_id (poda de
particao para performance) combinado com controle de acesso
obrigatorio na camada de consumo (views ou politicas de acesso que
tornam o filtro por tenant nao-opcional, nao uma convencao que
depende de disciplina).**

### Decisao

Opcao C. O particionamento fisico por `municipio_id` como primeira
chave de particao (antes da data) da a poda de particao necessaria
para desempenho - uma consulta de um Municipio nunca varre dados de
outro fisicamente. O controle de acesso na camada de consumo (nao
detalhado em ferramenta especifica aqui, mas como requisito
arquitetural: toda credencial de Operador Municipal so pode
consultar a particao do proprio municipio_id, imposto pela camada
de servico, nao pela query) fecha a lacuna que a Opcao B deixava
aberta.

### Consequencias

Cada novo cliente adiciona uma nova particao, nao uma nova
infraestrutura - o custo cresce proporcionalmente ao volume de dados
do cliente, nao a um custo fixo por cliente (RN6 atendido). O
gatilho para revisitar esta decisao seria um cliente exigindo
contratualmente isolamento fisico total (banco dedicado) - algo que
nao existe hoje nos contratos padrao da AeroWatch, mas que a
arquitetura deveria conseguir acomodar como excecao pontual, nao
como redesenho completo.

## Estrutura de particionamento fisico

```
data/
├── bronze/municipio_id=X/ingestion_date=YYYY-MM-DD/
├── silver/municipio_id=X/measured_date=YYYY-MM-DD/
└── gold/
    ├── fact_measurement_hourly/municipio_id=X/mes=YYYY-MM/
    └── fact_relatorio_conformidade/municipio_id=X/
```

`municipio_id` como primeiro nivel de particao em todas as camadas -
nao so na Gold - porque o isolamento precisa valer desde a ingestao,
nao ser adicionado so na camada de consumo.

## Motor de consulta

DuckDB continua adequado para o volume atual (12 clientes). Na
escala projetada de 50 clientes (documento 06), o gatilho de
migracao para um motor com melhor suporte a concorrencia multi-usuario
(ex.: Athena sobre o mesmo Parquet particionado, sem reescrever o
formato de armazenamento) e o numero de consultas concorrentes de
dashboards de diferentes Municipios simultaneamente excedendo o que
uma unica instancia DuckDB atende com latencia aceitavel (RN3).

## Orquestracao

Airflow, seguindo o [ADR 0003](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0003-orquestracao-self-hosted-vs-gerenciada.md) -
mas com uma diferenca de design importante em relacao ao portfolio
anterior: **um unico DAG parametrizado processa todos os
Municipios em lote**, iterando sobre a lista de clientes ativos, em
vez de um DAG por cliente. Um DAG por cliente escalaria linearmente
o esforco operacional com o numero de clientes - exatamente o que
RN5 (equipe pequena) probe.

## Indices

Sem indice tradicional de banco relacional - a estrategia de
performance e inteiramente baseada em particionamento fisico
(poda de particao por `municipio_id` e por data/mes) e nas
estatisticas de zona (min/max por coluna) que o proprio formato
Parquet mantem internamente por row group.
