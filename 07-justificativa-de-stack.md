# 07. Justificativa de Stack

Resumo executivo - a versao "5 minutos para entender por que
escolhemos isso" deste blueprint, para apresentacao a um comite
tecnico ou a lideranca da AeroWatch.

## Tabela de decisoes

| Componente | Escolha | Alternativa descartada | Motivo resumido | Referencia |
|---|---|---|---|---|
| Formato de armazenamento | Parquet, camadas Bronze/Silver/Gold | Data Warehouse relacional unico | Portabilidade entre ferramentas, custo de armazenamento desacoplado de computacao | [ADR 0001](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0001-data-lake-vs-warehouse-vs-lakehouse.md) |
| Particionamento por camada | Bronze por ingestao, Silver/Gold por evento | Mesmo criterio em todas as camadas | Bronze precisa ser trilha de auditoria fiel; Silver/Gold precisam ser consultaveis por periodo de negocio | [ADR 0002](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0002-particionamento-bronze-ingestion-vs-silver-measured.md) |
| Modelagem da camada de consumo | Star Schema | Data Vault, One Big Table | Uma unica familia de fontes (sensores) compartilhando dimensoes comuns - Data Vault resolveria um problema de escala de fontes heterogeneas que nao existe aqui | [ADR 0008](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0008-modelagem-star-schema-vs-data-vault-vs-obt.md), aplicado em detalhe no [documento 03](03-modelo-logico.md) |
| Isolamento multi-tenant | Particionamento fisico por municipio_id + controle de acesso obrigatorio na camada de consumo | Banco de dados dedicado por Municipio | Isolamento forte sem custo fixo por cliente, compativel com equipe pequena (RN5) e orcamento proporcional (RN6) | Decisao nova, detalhada no [documento 04](04-modelo-fisico.md#decisao-nova-isolamento-fisico-de-dados-entre-municipios) |
| Orquestracao | Airflow, DAG unico multi-tenant parametrizado | Airflow com um DAG por cliente | Esforco operacional constante independente do numero de clientes (RN5) | [ADR 0003](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0003-orquestracao-self-hosted-vs-gerenciada.md), adaptado no [documento 04](04-modelo-fisico.md) |
| Motor de consulta (estado atual) | DuckDB | Data warehouse gerenciado desde o primeiro cliente | Custo proporcional ao uso real (RN6) - adequado ate o volume atual e o projetado em 24 meses | [documento 06](06-requisitos-nao-funcionais.md) |
| Motor de consulta (gatilho de migracao) | Reavaliar quando concorrencia de dashboards multi-tenant exceder a capacidade de uma unica instancia | - | Decisao adiada deliberadamente ate o gatilho concreto aparecer, nao antecipada sem necessidade | [documento 04](04-modelo-fisico.md) |

## O que este blueprint deliberadamente nao resolve agora

Coerente com o principio de nao fazer over-engineering presente em
todo o portfolio (ver, por exemplo, o raciocinio de proporcionalidade
do [ADR 0001](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0001-data-lake-vs-warehouse-vs-lakehouse.md)):

- Streaming em tempo real - RN3 aceita ate 1 hora de latencia,
  streaming resolveria um problema que a AeroWatch nao tem hoje (ver
  raciocinio equivalente no [ADR 0010](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0010-batch-vs-streaming-proxima-evolucao.md)
  do portfolio original).
- Motor de consulta distribuido desde o primeiro cliente - o gatilho
  de migracao esta definido, mas nao antecipado sem necessidade
  (RN6).
- Banco de dados dedicado por Municipio - resolveria isolamento de
  forma mais direta, mas nao proporcional ao numero de clientes
  (RN6), e a Opcao C do documento 04 ja atende RN1 sem esse custo.

## Consideracao final

Este blueprint prioriza duas coisas que a maioria dos exercicios de
arquitetura ignora: **quando** cada decisao deveria ser revisitada
(gatilhos explicitos, nao "no futuro") e **por que** a opcao mais
sofisticada tecnicamente nem sempre e a certa para o estagio atual
do negocio. E o mesmo principio aplicado em todo o portfolio -
aqui, aplicado a um sistema inteiro antes de qualquer linha de
codigo existir.
