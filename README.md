# Blueprint de Arquitetura de Dados Corporativa

Documento de arquitetura completo - modelo conceitual, logico,
fisico, fluxo de dados, requisitos nao-funcionais e justificativa de
stack - para uma organizacao fictícia de monitoramento ambiental
multi-cliente ("AeroWatch"). **Este repositorio nao contem codigo**:
e um exercicio deliberado de desenho de sistema, o tipo de
entregavel produzido antes de qualquer linha de codigo existir.

Projeto 09 de uma serie documentando minha transicao de Analista de
Dados para Arquitetura de Dados - veja o [perfil
completo](https://github.com/amanda-martins-data).

## Por que um projeto sem codigo

Os projetos anteriores do portfolio provam execucao (pipelines que
rodam), decisao justificada ([ADRs](https://github.com/amanda-martins-data/adr-arquitetura-dados))
e organizacao de regras ([governanca e catalogo de dados](https://github.com/amanda-martins-data/governanca-catalogo-dados)).
Este projeto prova a competencia que fecha o ciclo: desenhar um
sistema do zero, a partir de requisitos de negocio, antes de
qualquer implementacao comecar - exatamente o tipo de teste tecnico
pedido em processos seletivos para Arquiteto de Dados senior.

## O cenario

**AeroWatch** e uma consultoria ambiental fictícia que opera
infraestrutura de monitoramento de qualidade do ar para multiplas
prefeituras contratantes - hoje 12 clientes, com projecao de
crescer para 50 em 24 meses. O cenario e propositalmente proximo da
minha experiencia profissional real em consultoria ambiental, mas
totalmente fictício.

## Indice

| Documento | Conteudo |
|---|---|
| [01 - Contexto e Requisitos](01-contexto-e-requisitos.md) | O cenario, requisitos de negocio (RN1-RN6), restricoes |
| [02 - Modelo Conceitual](02-modelo-conceitual.md) | Entidades de negocio e relacionamentos, sem tecnologia |
| [03 - Modelo Logico](03-modelo-logico.md) | Star Schema detalhado - fatos, dimensoes, grao, chaves |
| [04 - Modelo Fisico](04-modelo-fisico.md) | Tecnologias, particionamento, e a decisao de isolamento multi-tenant |
| [05 - Arquitetura de Fluxo](05-arquitetura-de-fluxo.md) | Diagrama end-to-end, sensor ate relatorio |
| [06 - Requisitos Nao-Funcionais](06-requisitos-nao-funcionais.md) | Capacidade, disponibilidade, DR (RPO/RTO), LGPD aplicada |
| [07 - Justificativa de Stack](07-justificativa-de-stack.md) | Resumo executivo com trade-offs, para apresentacao a comite tecnico |

## Como este documento se conecta ao resto do portfolio

Este blueprint nao reinventa decisoes ja tomadas nos Projetos 01-08 -
ele as referencia e as aplica a um cenario novo (multi-tenant, escala
de 50 clientes), estendendo-as quando o cenario exige algo que o
portfolio original nao tinha (isolamento entre clientes, LGPD
aplicada a dado pessoal real, requisitos de disponibilidade e
disaster recovery formais). Cada decisao reaproveitada linka de volta
ao [ADR original](https://github.com/amanda-martins-data/adr-arquitetura-dados);
cada decisao nova segue o mesmo formato de raciocinio (opcoes
consideradas, criterio de desempate, consequencias e gatilho de
revisao).
