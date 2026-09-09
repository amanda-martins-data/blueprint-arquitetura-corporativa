# 03. Modelo Logico

## Por que Star Schema

O [ADR 0008](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0008-modelagem-star-schema-vs-data-vault-vs-obt.md)
do repositorio de arquitetura ja avaliou Star Schema contra Data
Vault e One Big Table para um cenario de multiplas fontes
compartilhando dimensoes comuns - exatamente o cenario da AeroWatch,
agora com o agravante de multi-tenant (50 Municipios). Este
documento aplica formalmente essa decisao, com o modelo completo.

Data Vault permanece descartado pelo mesmo motivo do ADR original:
a AeroWatch tem uma unica fonte de dados por Municipio (os sensores
daquele cliente), nao dezenas de sistemas heterogeneos - o problema
que o Data Vault resolve nao existe aqui na escala que justificaria
sua complexidade adicional.

## Tabelas de dimensao

### dim_municipio (SCD Tipo 2)

| Coluna | Tipo | Observacao |
|---|---|---|
| municipio_sk | inteiro | chave substituta (surrogate key) |
| municipio_id | texto | identificador de negocio, estavel |
| nome | texto | |
| uf | texto | |
| contrato_ativo_id | texto | referencia ao contrato vigente |
| poluentes_contratados | lista de texto | escopo do contrato atual |
| valido_de | data | inicio de vigencia desta versao da linha |
| valido_ate | data | fim de vigencia (null = versao atual) |
| e_atual | booleano | flag de conveniencia para a versao vigente |

SCD Tipo 2 porque o escopo do contrato (quais poluentes, SLA) pode
mudar em uma renovacao - preservar o historico e necessario para o
Relatorio de Conformidade de periodos passados continuar correto
mesmo apos uma renegociacao contratual.

### dim_estacao

| Coluna | Tipo | Observacao |
|---|---|---|
| estacao_sk | inteiro | chave substituta |
| estacao_id | texto | identificador de negocio |
| municipio_sk | inteiro | FK para dim_municipio |
| nome | texto | |
| latitude / longitude | decimal | |

### dim_sensor

| Coluna | Tipo | Observacao |
|---|---|---|
| sensor_sk | inteiro | chave substituta |
| sensor_id | texto | identificador de negocio |
| estacao_sk | inteiro | FK para dim_estacao |
| modelo | texto | fabricante/modelo do equipamento |
| data_instalacao | data | |

### dim_poluente

Dimensao de referencia, compartilhada entre todos os Municipios
(nao e multi-tenant) - PM2.5, PM10, O3, NO2, CO, com limites
regulatorios padrao por unidade da federacao quando aplicavel.

### dim_tempo

Dimensao de calendario padrao (data, mes, trimestre, ano, dia da
semana, feriado) - usada tanto pelo fato horario quanto pelo fato de
conformidade mensal.

## Tabelas de fato

### fact_measurement_hourly

**Grao**: uma linha por sensor, por poluente, por hora.

O grao horario (nao por leitura individual de 5 em 5 minutos) e uma
decisao deliberada: a camada de consumo (Star Schema) serve o
dashboard (RN3, latencia aceitavel de ate 1 hora) e os relatorios,
nenhum dos dois precisa de granularidade de 5 minutos. A leitura
bruta a cada 5 minutos e preservada na camada Bronze/Silver (fora do
Star Schema), seguindo o mesmo raciocinio de camadas do
[ADR 0001](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0001-data-lake-vs-warehouse-vs-lakehouse.md) -
o Star Schema e a camada de consumo agregado, nao o dado bruto.

| Coluna | Tipo | Observacao |
|---|---|---|
| municipio_sk | inteiro | FK - tambem a chave de particionamento fisico (ver documento 04) |
| sensor_sk | inteiro | FK |
| poluente_sk | inteiro | FK |
| tempo_sk | inteiro | FK - granularidade de hora |
| valor_medio | decimal | media das leituras de 5 min dentro da hora |
| valor_maximo | decimal | |
| valor_minimo | decimal | |
| contagem_leituras | inteiro | quantas leituras de 5 min compoem esta hora (para detectar falha de sensor) |

### fact_relatorio_conformidade

**Grao**: uma linha por Municipio, por mes, por poluente.

| Coluna | Tipo | Observacao |
|---|---|---|
| municipio_sk | inteiro | FK |
| poluente_sk | inteiro | FK |
| mes_referencia_sk | inteiro | FK para dim_tempo (granularidade de mes) |
| media_mensal | decimal | |
| dias_em_conformidade | inteiro | dias no mes dentro do limite regulatorio |
| dias_fora_conformidade | inteiro | |
| status_sla_entrega | texto | "no prazo" / "atrasado", medido contra RN2 |

## Por que dois fatos, nao um

`fact_measurement_hourly` responde "o que aconteceu tecnicamente".
`fact_relatorio_conformidade` responde "estamos cumprindo o
contrato". Sao publicos e ritmos de atualizacao diferentes: o
primeiro atualiza a cada hora (RN3), o segundo fecha uma vez por mes
(RN2) e uma vez fechado nao deveria mudar - misturar os dois no
mesmo fato forcaria granularidades incompativeis na mesma tabela.
