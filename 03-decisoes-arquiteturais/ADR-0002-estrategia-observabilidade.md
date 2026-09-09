# ADR-0002: Estratégia de observabilidade

## Status
Aceito

## Contexto
Com múltiplas aplicações independentes (RNF-01) rodando em plataformas distintas (Azure Container Apps como padrão, conforme ADR-0001; Azure App Service para o que ainda não migrou), a Vetria precisa de uma estratégia de observabilidade que atenda RNF-06 (visão consolidada de logs, métricas e falhas) sem comprometer o isolamento entre aplicações (RNF-01) nem exigir operação manual por squad.

Hoje, sem um pipeline de CI/CD e monitoramento padronizados, cada squad decide individualmente como (e se) instrumenta sua aplicação — uma das dores já identificadas no discovery.

## Alternativas consideradas

### 1. Um Log Analytics workspace + Application Insights isolado por aplicação
Prós: isolamento total de custo e dados por squad.
Contras: impossibilita correlação entre aplicações e visão consolidada (viola RNF-06); squads reinventam configuração de alertas e dashboards individualmente.

### 2. Um único Application Insights compartilhado por todas as aplicações
Prós: simplicidade máxima de configuração inicial.
Contras: telemetria de todas as aplicações misturada no mesmo recurso, dificultando atribuição de custo e gerando ruído entre squads; não escala bem à medida que o portfólio cresce.

### 3. Log Analytics workspace centralizado por ambiente + Application Insights individual por aplicação (workspace-based), com OpenTelemetry como padrão de instrumentação
Prós: cada aplicação mantém seu próprio recurso de Application Insights (isolamento de custo e ruído, RNF-01), mas todos apontam para o mesmo workspace por ambiente, permitindo queries e dashboards consolidados (RNF-06); OpenTelemetry evita lock-in ao SDK proprietário e prepara terreno para correlação entre serviços caso surja comunicação entre eles (RNF-07); Managed Prometheus + Managed Grafana cobrem métricas de infraestrutura (KEDA, revisions, utilização) sem exigir operação de um Prometheus próprio.
Contras: mais recursos Azure para provisionar via IaC do que a opção 2 (mitigado: tudo via template Bicep reutilizável).

## Decisão
Adotar, por ambiente (dev/hml/prod), um **Log Analytics workspace centralizado**, com um recurso de **Application Insights (workspace-based) por aplicação** apontando para esse workspace. Toda aplicação nova — Container Apps ou App Service — deve instrumentar-se com o **Azure Monitor OpenTelemetry Distro** (.NET/Node/Python/Java conforme a stack), emitindo logs estruturados em JSON com trace/span IDs (RF-04).

Métricas de infraestrutura (ambiente Container Apps, KEDA, revisions) são coletadas via **Azure Monitor managed Prometheus**, visualizadas em **Azure Managed Grafana** com dashboards cross-app. Alertas usam **Azure Monitor Alert rules + Action Groups**, com severidade definida pela criticidade de cada aplicação.

## Consequências
- Todo novo pipeline de CI/CD (RF-01) passa a provisionar, via Bicep, o Application Insights da aplicação já apontando para o workspace do ambiente correspondente.
- Squads mantêm autonomia sobre o dashboard da própria aplicação, mas a visão consolidada (Grafana/Workbooks) fica centralizada e não depende de ação individual.
- Aplicações que migrarem do App Service para Container Apps (conforme ADR-0001) não perdem histórico de telemetria, desde que reaproveitem o mesmo recurso de Application Insights.
- Define-se retenção padrão de 30 dias em modo interativo no workspace, com Basic Logs para volumes maiores de baixo valor de consulta (custo).
- Esta decisão depende da adoção de RF-04 (logs estruturados via OpenTelemetry) por todas as aplicações.

## Referências
- RNF-01, RNF-06, RNF-07 e RF-04 em `02-requisitos/requisitos-nao-funcionais.md`
- ADR-0001 (`03-decisoes-arquiteturais/ADR-0001-plataforma-de-compute.md`)
- [Observability in Azure Container Apps – Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/observability)
- [Application Insights OpenTelemetry observability overview – Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [Overview of Azure Monitor Managed Service for Prometheus – Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/prometheus-metrics-overview)
