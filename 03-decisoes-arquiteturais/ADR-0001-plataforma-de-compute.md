# ADR-0001: Plataforma de compute para aplicações independentes

## Status
Aceito

## Contexto
A Vetria opera hoje um portfólio de aplicações internas (portal de pedidos, WMS, TMS, CRM, financeiro) hospedadas em Azure App Service, mantidas por squads distintas e sem relação direta entre si (RNF-01). Existe pressão de mercado para migrar para Kubernetes (AKS), mas essa decisão nunca foi formalmente avaliada frente aos requisitos reais da empresa (RNF-01 a RNF-07).

Precisamos escolher uma plataforma de compute que sirva de padrão para novas aplicações e para a evolução das existentes, equilibrando portabilidade e capacidade de escala com o overhead operacional que a empresa está disposta a assumir.

## Alternativas consideradas

### 1. Manter Azure App Service
Prós: menor curva de aprendizado, já em uso, baixo overhead.
Contras: menor portabilidade (não é container-first por padrão), menos controle sobre blue/green e autoscaling orientado a evento, sem caminho nativo para comunicação entre serviços caso ela surja no futuro (RNF-07).

### 2. Migrar para Azure Kubernetes Service (AKS)
Prós: máximo controle e flexibilidade, ecossistema Kubernetes maduro, atende bem cenários de microsserviços fortemente acoplados.
Contras: exige time de plataforma dedicado (viola RNF-02), introduz complexidade operacional (patching de nós, upgrades de cluster, RBAC, políticas de rede) desproporcional ao cenário de aplicações independentes (RNF-01), maior custo de operação para o ganho obtido.

### 3. Azure Container Apps
Prós: containers sem operar cluster (atende RNF-02 e RNF-04), escala a zero e autoscaling via KEDA orientado a evento/fila (RNF-03), revisions nativas para blue/green e canary, Dapr integrado para comunicação entre serviços caso surja necessidade futura (RNF-07), isolamento total entre Container Apps distintas (RNF-01).
Contras: menos controle de baixo nível que AKS (não é um problema para o cenário atual); plataforma mais nova que App Service (mitigado: já é GA e amplamente documentada).

## Decisão
Adotar **Azure Container Apps** como plataforma padrão de compute para novas aplicações e como destino de migração incremental das aplicações existentes, mantendo o Azure App Service para cargas que não justificam migração no curto prazo.

Azure Kubernetes Service (AKS) fica reservado como alternativa futura, a ser reavaliada somente se surgir ao menos um dos seguintes gatilhos:

- Necessidade real de comunicação intensa entre serviços, exigindo service mesh;
- Requisitos de rede/scheduling que o Container Apps não suporte;
- Ferramental específico do ecossistema Kubernetes (operators, CRDs) já dominado pelo time;
- Volume que justifique a formação de um time de plataforma dedicado.

## Consequências
- Todas as novas aplicações passam a ser containerizadas desde o início, mesmo as que inicialmente rodariam em App Service.
- É necessário padronizar um pipeline de CI/CD reutilizável (RF-01 a RF-03), com build de imagem, push para Azure Container Registry e deploy via IaC (Bicep).
- Observabilidade centralizada (RNF-06) precisa ser desenhada desde já em um Log Analytics workspace único, independente da plataforma de cada aplicação.
- A migração de aplicações existentes do App Service para Container Apps pode ocorrer de forma incremental, aplicação por aplicação, sem necessidade de big bang.
- Esta decisão não é definitiva: pode ser revisada caso os gatilhos para AKS listados acima se concretizem.

## Referências
- RNF-01 a RNF-07 e RF-01 a RF-03 em `02-requisitos/requisitos-nao-funcionais.md`
- [Azure Container Apps overview – Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/overview)
- [Scale Dapr Applications with KEDA Scalers – Microsoft Learn](https://learn.microsoft.com/en-us/azure/container-apps/dapr-keda-scaling)
