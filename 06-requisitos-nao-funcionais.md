# 06. Requisitos Nao-Funcionais

Este e o documento que normalmente falta em portfolios de dados -
a maioria mostra que o pipeline funciona, poucos mostram que
pensaram em escala, disponibilidade e recuperacao de desastre antes
de precisar deles na pratica.

## Capacidade e escala

| Cenario | Volume diario de leituras brutas | Observacao |
|---|---|---|
| Atual (12 municipios) | ~310 mil | media de 18 sensores x 5 poluentes x 288 leituras/dia x 12 |
| Projetado, 24 meses (50 municipios) | ~1,3 milhao | mesma media por cliente |
| Projetado, 10x (500 municipios, cenario de estresse) | ~13 milhoes | fora do horizonte de 24 meses, mas usado para testar os limites do desenho |

**Gatilho de reprojeto identificado**: no cenario de 10x, o motor de
consulta unico (DuckDB) deixa de ser adequado para concorrencia de
dashboard entre centenas de tenants simultaneos - o mesmo gatilho ja
identificado no [documento 04](04-modelo-fisico.md). O
particionamento fisico por `municipio_id`, por outro lado,
continua valido sem alteracao ate esse ponto - a decisao do
documento 04 foi feita para nao precisar de reprojeto nessa
dimensao especifica antes do motor de consulta.

## Disponibilidade

- **Dashboard operacional**: meta de 99% de disponibilidade em
  horario comercial (8h-18h, dias uteis) - nao 99,9%, porque RN3 ja
  aceita ate 1 hora de latencia, entao uma falha curta e absorvida
  pela proxima execucao horaria sem quebrar o requisito de negocio.
- **Pipeline batch**: pode ter janelas de manutencao planejadas fora
  do horario comercial, sem violar nenhum requisito - a unica
  restricao dura e o fechamento mensal do relatorio de conformidade
  (RN2).

## Disaster Recovery

| Metrica | Meta | Justificativa |
|---|---|---|
| RPO (Recovery Point Objective) | 24 horas | equivale a, no maximo, uma execucao horaria de pipeline perdida somada a folga - aceitavel porque a Bronze e reconstruivel a partir da API de origem para o periodo perdido, se necessario |
| RTO (Recovery Time Objective) | 4 horas uteis | tempo maximo para restaurar o pipeline batch a partir de backup do ultimo estado consistente da Silver/Gold, sem impactar o prazo do relatorio mensal (RN2) |

A camada Bronze, por ser imutavel e retida por 5 anos (RN4), e a
base da estrategia de recuperacao: nenhum backup adicional de
Silver/Gold precisa ser mais confiavel que a capacidade de
reprocessar a Bronze do zero - a mesma logica de "Bronze como fonte
de verdade reprocessavel" que motivou a retencao longa no
[ADR 0002](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0002-particionamento-bronze-ingestion-vs-silver-measured.md).

## Isolamento multi-tenant (RN1) - resumo de seguranca

Detalhado no [documento 04](04-modelo-fisico.md#decisao-nova-isolamento-fisico-de-dados-entre-municipios).
Resumo do requisito nao-funcional: **zero tolerancia** para vazamento
de dado entre Municipios - isso nao e um requisito de melhor
esforco, e uma condicao contratual (RN1). A arquitetura trata isso
como camada dupla (particionamento fisico + controle de acesso
obrigatorio), nao como uma unica linha de defesa.

## LGPD aplicada (nao apenas teorica)

Diferente do catalogo de dados do [Projeto 08](https://github.com/amanda-martins-data/governanca-catalogo-dados),
onde nenhum dataset continha dado pessoal de verdade, este cenario
tem dado pessoal real: nome, e-mail e cargo de cada Operador
Municipal com acesso ao sistema. Isso significa que, aplicando o
framework do Projeto 08 na pratica:

- `dim_usuario` (nao detalhada no modelo logico por estar fora do
  escopo analitico, mas presente no sistema operacional) seria
  classificada como `sensitivity: personal`, com `pii_fields:
  ["nome", "email", "cargo"]`.
- A base legal aplicavel e execucao de contrato (Art. 7, V da LGPD) -
  o dado do Operador Municipal e necessario para a prestacao do
  servico contratado pelo proprio Municipio, nao exige consentimento
  adicional separado do contrato.
- Retencao do dado do Usuario: enquanto o contrato estiver vigente,
  mais um periodo de retencao pos-encerramento definido por
  obrigacao legal (nao coberto em detalhe aqui - dependeria de
  orientacao juridica especifica, fora do escopo tecnico deste
  blueprint).
