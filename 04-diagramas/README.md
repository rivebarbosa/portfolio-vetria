# Diagramas

Diagramas C4 (Contexto e Container/Landing Zone), em formato draw.io (`.drawio`), consolidando as decisões das ADR-0001 a ADR-0008. Abra com [diagrams.net](https://app.diagrams.net) (importar arquivo) ou a extensão draw.io do VS Code.

## `c4-contexto.drawio` — C4 Nível 1 (Contexto)
Mostra os dois grupos de atores (colaboradores internos e revendedores externos) e as cinco aplicações do portfólio da Vetria como sistemas independentes entre si (RNF-01), todas autenticando via Microsoft Entra ID / Entra External ID (ADR-0005).

## `c4-container-landing-zone.drawio` — C4 Nível 2 adaptado (Container / Landing Zone)
Consolida a infraestrutura decidida nas ADRs em um único diagrama:
- **Subscription Connectivity (hub)**: Azure Firewall, Private DNS Zones, Key Vault de plataforma (ADR-0003).
- **Subscription Prod**: Container Apps Environment com o Portal de Pedidos detalhado como aplicação representativa (Container App, PostgreSQL Flexible Server via Private Endpoint, Key Vault e Application Insights próprios — ADR-0001, ADR-0002, ADR-0003, ADR-0006), com nota indicando que WMS, TMS, CRM e Financeiro replicam o mesmo padrão. Observabilidade compartilhada do ambiente (Log Analytics, managed Prometheus, Managed Grafana — ADR-0002) e RBAC via grupos + PIM (ADR-0005).
- **Subscription Dev/Hml**: mesma estrutura em escala menor, com nota reforçando RNF-13 (dados mascarados/anonimizados).
- **Externos ao tenant**: Microsoft Entra ID/External ID (ADR-0005), GitHub Actions fazendo deploy via OIDC (ADR-0008), Microsoft Defender for Cloud e Azure Policy bloqueando recursos fora de conformidade (ADR-0007).

Toda a estrutura está contida no boundary "Azure Tenant — Vetria (região Brazil South)", reforçando a ADR-0004.
