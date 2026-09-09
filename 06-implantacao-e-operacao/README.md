# Implantação e Operação

Detalha, em código, o pipeline de CI/CD decidido na ADR-0008 e como ele materializa as ADRs anteriores. Este repositório documenta a *arquitetura*, não hospeda as aplicações da Vetria — os arquivos abaixo são o template de referência que cada repositório de aplicação (ex.: `portal-de-pedidos`, `wms`, `tms`) consome.

## Arquivos

- `pipeline/reusable-deploy.yml` — o reusable workflow central (RF-01), chamado por todo repositório de aplicação.
- `pipeline/exemplo-caller-portal-de-pedidos.yml` — exemplo de como um repositório de aplicação invoca o template, promovendo dev → hml → prod.

## Como os estágios do pipeline materializam as ADRs anteriores

1. **build-and-push**: login no Azure via OIDC/federated credentials, sem client secret (ADR-0005) — constrói a imagem e publica no Azure Container Registry.
2. **validar-infraestrutura**: roda `az deployment group what-if` sobre o Bicep da aplicação. Isso não é um gate à parte — é a própria validação do Azure Resource Manager falhando automaticamente se o template violar uma política com efeito `deny` atribuída nos management groups (região, redundância, exposição pública — ADR-0007). É assim que RF-06 se torna realidade, não uma checklist manual.
3. **deploy-infraestrutura**: aplica o Bicep (Application Insights, Key Vault, Container App, banco de dados quando aplicável — ADR-0002, ADR-0006, ADR-0007), sempre em Brazil South (ADR-0004).
4. **smoke-test**: valida que a aplicação responde após o deploy.

Cada estágio roda sob um **GitHub Environment** (`dev`, `hml`, `prod`) — o ambiente `prod` tem uma protection rule configurada no repositório exigindo aprovação manual antes de rodar (RF-03), e cada ambiente usa suas próprias credenciais federadas (`AZURE_CLIENT_ID` por ambiente), nunca compartilhadas entre dev/hml/prod nem entre aplicações (ADR-0005).

## O que fica fora deste repositório
Os templates Bicep de cada aplicação (`infra/main.bicep`) vivem no repositório da própria aplicação, não aqui — este repositório documenta a arquitetura de referência, não a implementação de cada app do portfólio da Vetria.
