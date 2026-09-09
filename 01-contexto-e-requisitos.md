# 01. Contexto e Requisitos

## O cenario fictício

**AeroWatch** e uma consultoria ambiental que opera infraestrutura de
monitoramento de qualidade do ar para prefeituras contratantes. Cada
prefeitura instala uma rede propria de sensores (PM2.5, PM10, O3,
NO2, CO) e contrata a AeroWatch para coletar, validar, armazenar e
reportar esses dados de forma continua, incluindo o relatorio mensal
de conformidade ambiental exigido por orgaos reguladores estaduais.

Este cenario e proximo da minha experiencia profissional real em
consultoria ambiental (CPEA), mas totalmente fictício - nenhum dado,
cliente ou numero aqui corresponde a uma situacao real.

## Estado atual e projecao de crescimento

- **Hoje**: 12 prefeituras contratantes, entre 5 e 40 sensores cada
  (media de 18), 5 poluentes monitorados por sensor, leitura a cada
  5 minutos.
- **Projecao (24 meses)**: crescimento para 50 prefeituras, mesma
  media de sensores por cliente.
- **Volume na escala projetada**: 50 municipios x 18 sensores x 5
  poluentes x 288 leituras/dia = aproximadamente 1,3 milhao de
  leituras brutas por dia.

Este crescimento e o eixo central do blueprint: a arquitetura
precisa suportar 50 clientes sem que o custo operacional cresca na
mesma proporcao (ver Requisito de Negocio RN6).

## Requisitos de negocio

| ID | Requisito | Origem |
|---|---|---|
| RN1 | Cada prefeitura acessa exclusivamente os proprios dados - isolamento total entre clientes (multi-tenant) | Contratual - clausula de confidencialidade padrao em todos os contratos |
| RN2 | Relatorio de conformidade mensal, entregue em ate 5 dias uteis apos o fechamento do mes | Exigencia regulatoria estadual, repassada contratualmente |
| RN3 | Dashboard operacional para a prefeitura, com latencia aceitavel de ate 1 hora entre a medicao e a visualizacao | Requisito operacional, nao regulatorio - definido em reuniao com clientes-piloto |
| RN4 | Dados brutos preservados por no minimo 5 anos, para auditoria ambiental retroativa | Exigencia regulatoria, mesmo periodo de retencao ja adotado para a camada Bronze no [ADR 0002](https://github.com/amanda-martins-data/adr-arquitetura-dados/blob/main/decisions/0002-particionamento-bronze-ingestion-vs-silver-measured.md) |
| RN5 | Equipe tecnica da AeroWatch permanece pequena (2 a 3 pessoas) mesmo com o crescimento de clientes | Restricao de negocio - a AeroWatch nao planeja crescer o time de dados na mesma proporcao dos clientes |
| RN6 | Custo de infraestrutura deve escalar proporcionalmente ao numero de clientes ativos, nao ser um custo fixo alto desde o primeiro cliente | Restricao financeira - modelo de precificacao da AeroWatch e por cliente contratado |

## Requisitos nao-funcionais (resumo - detalhados no documento 06)

- Disponibilidade do dashboard: 99% em horario comercial.
- RPO (perda maxima de dados aceitavel): 24 horas.
- RTO (tempo maximo de recuperacao): 4 horas uteis.
- Isolamento de dados entre tenants: nenhuma consulta de um cliente
  pode, em nenhuma circunstancia, retornar dado de outro cliente.

## Restricoes assumidas

- A AeroWatch nao tem, e nao planeja ter, um Data Engineer dedicado
  por cliente (RN5) - a arquitetura precisa ser operavel por um time
  pequeno cuidando de todos os clientes simultaneamente.
- Cada prefeitura e responsavel pela manutencao fisica dos proprios
  sensores; a AeroWatch e responsavel apenas pelo pipeline de dados
  a partir do momento em que a leitura chega ao sistema.
- O orcamento de infraestrutura por cliente e limitado - a arquitetura
  nao pode assumir que cada novo cliente justifica um novo cluster
  ou banco de dados dedicado (ver RN6, aprofundado no [ADR de
  isolamento multi-tenant](07-justificativa-de-stack.md)).

## Fora de escopo deste blueprint

- Integracao com os equipamentos fisicos dos sensores (protocolo de
  telemetria) - assume-se que os dados ja chegam normalizados a
  borda do sistema AeroWatch via API HTTP.
- Faturamento e gestao contratual - tratados como sistemas externos,
  apenas referenciados onde influenciam a arquitetura de dados
  (ex.: RN1, RN2).
