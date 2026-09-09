# Requisitos — Plataforma de Compute e CI/CD

## Requisitos não funcionais

- **RNF-01 — Baixo acoplamento entre aplicações**: a plataforma não deve impor comunicação ou dependência entre aplicações que não têm relação de negócio entre si.
- **RNF-02 — Baixo overhead operacional**: a solução não deve exigir um time de plataforma dedicado full-time para operação (cluster, patching, upgrades).
- **RNF-03 — Escalabilidade sob demanda**: aplicações com uso sazonal (ex.: portal de pedidos em datas de pico) devem escalar automaticamente, inclusive a zero em ambientes de baixo uso (homologação).
- **RNF-04 — Portabilidade**: aplicações devem rodar de forma consistente em todos os ambientes (dev/hml/prod), preferencialmente via containers.
- **RNF-05 — Deploy independente por aplicação**: cada squad deve poder implantar sua aplicação sem depender do ciclo de deploy de outra.
- **RNF-06 — Observabilidade centralizada**: mesmo com aplicações isoladas, deve existir visão consolidada de logs, métricas e falhas.
- **RNF-07 — Evolutividade**: a arquitetura deve comportar, no futuro, aplicações que eventualmente precisem se comunicar entre si (ex.: pub/sub entre WMS e TMS), sem exigir uma re-arquitetura completa.

## Requisitos funcionais (relevantes para este bloco)

- **RF-01**: cada aplicação deve ter pipeline de CI/CD próprio, a partir de um template reutilizável.
- **RF-02**: o processo de build deve gerar artefatos containerizados armazenados em um registry privado (Azure Container Registry).
- **RF-03**: a promoção entre ambientes (dev → hml → prod) deve ocorrer via pipeline, com aprovação manual antes de produção.

## Fora de escopo (por ora)

- Comunicação síncrona/assíncrona entre aplicações (não há esse requisito hoje).
- Multi-cloud (a decisão está restrita ao ecossistema Azure).
