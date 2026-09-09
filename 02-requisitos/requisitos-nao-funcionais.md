# Requisitos — Plataforma de Compute e CI/CD

## Requisitos não funcionais

- **RNF-01 — Baixo acoplamento entre aplicações**: a plataforma não deve impor comunicação ou dependência entre aplicações que não têm relação de negócio entre si.
- **RNF-02 — Baixo overhead operacional**: a solução não deve exigir um time de plataforma dedicado full-time para operação (cluster, patching, upgrades).
- **RNF-03 — Escalabilidade sob demanda**: aplicações com uso sazonal (ex.: portal de pedidos em datas de pico) devem escalar automaticamente, inclusive a zero em ambientes de baixo uso (homologação).
- **RNF-04 — Portabilidade**: aplicações devem rodar de forma consistente em todos os ambientes (dev/hml/prod), preferencialmente via containers.
- **RNF-05 — Deploy independente por aplicação**: cada squad deve poder implantar sua aplicação sem depender do ciclo de deploy de outra.
- **RNF-06 — Observabilidade centralizada**: mesmo com aplicações isoladas, deve existir visão consolidada de logs, métricas e falhas.
- **RNF-07 — Evolutividade**: a arquitetura deve comportar, no futuro, aplicações que eventualmente precisem se comunicar entre si (ex.: pub/sub entre WMS e TMS), sem exigir uma re-arquitetura completa.

- **RNF-08 — Isolamento de rede entre ambientes**: produção deve ser isolada de dev/homologação, com blast radius e billing separados.
- **RNF-09 — Sem exposição pública de serviços de plataforma**: Azure Container Registry, Key Vault, Log Analytics e bancos de dados não devem ter endpoint público, acessíveis somente via Private Link.

- **RNF-10 — Residência de dados no Brasil**: todos os dados da Vetria (aplicacionais, telemetria e backups) devem permanecer fisicamente em território brasileiro; nenhum recurso pode replicar dados para outra geografia, mesmo para fins de disaster recovery.

- **RNF-11 — Sem credenciais de longa duração**: pipelines e aplicações devem se autenticar via identidade federada (OIDC) ou managed identity, nunca via client secret, connection string ou chave de acesso fixa.
- **RNF-12 — Acesso privilegiado just-in-time**: acesso a subscriptions de produção deve ser temporário, elevado sob demanda e auditável (PIM), nunca atribuição permanente de Owner/Contributor.

- **RNF-13 — Proteção de dados pessoais em ambientes não produtivos**: dados pessoais de clientes, revendedores ou colaboradores não podem existir em sua forma real em Dev/Hml — devem ser mascarados, anonimizados ou substituídos por dados sintéticos antes de qualquer cópia a partir de produção.

- **RNF-14 — Conformidade validada automaticamente**: qualquer recurso fora dos padrões definidos nas ADRs (região, redundância, exposição pública) deve ser bloqueado na criação, não apenas identificado depois.
- **RNF-15 — Segredos centralizados e nunca em texto plano**: toda credencial, chave ou certificado deve residir em um cofre de segredos, nunca em pipeline, repositório ou configuração de aplicação.

- **RNF-16 — Unicidade de ferramenta de CI/CD**: todas as aplicações do portfólio devem usar o mesmo ecossistema de CI/CD, evitando a fragmentação hoje existente (cada squad monta o pipeline à sua maneira, identificada no discovery).

## Requisitos funcionais (relevantes para este bloco)

- **RF-01**: cada aplicação deve ter pipeline de CI/CD próprio, a partir de um template reutilizável.
- **RF-02**: o processo de build deve gerar artefatos containerizados armazenados em um registry privado (Azure Container Registry).
- **RF-03**: a promoção entre ambientes (dev → hml → prod) deve ocorrer via pipeline, com aprovação manual antes de produção.

- **RF-04**: toda aplicação (Container Apps ou App Service) deve emitir logs estruturados (JSON), instrumentados via OpenTelemetry, com trace/span IDs para correlação entre requisições.

- **RF-05**: toda aplicação que acessa outros serviços Azure (Key Vault, Storage, Azure Container Registry, bancos de dados) deve usar Managed Identity, nunca connection string ou API key em configuração.

- **RF-06**: todo pipeline de infraestrutura deve validar conformidade de política antes do deploy, com bloqueio automático em caso de não conformidade.

## Fora de escopo (por ora)

- Comunicação síncrona/assíncrona entre aplicações (não há esse requisito hoje).
- Multi-cloud (a decisão está restrita ao ecossistema Azure).
