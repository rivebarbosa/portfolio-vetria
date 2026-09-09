# ADR-0008: Ferramenta de CI/CD — GitHub Actions vs. Azure DevOps Pipelines

## Status
Aceito

## Contexto
O discovery (`01-discovery/contexto-vetria.md`) identificou que hoje "não existe um padrão único de CI/CD entre as aplicações; cada squad monta o pipeline à sua maneira" — formalizado em RNF-16. É preciso escolher uma ferramenta única de CI/CD para todo o portfólio, que suporte pipeline reutilizável por aplicação (RF-01), autenticação sem credenciais de longa duração (RNF-11/ADR-0005), gate de política antes do deploy (RF-06/ADR-0007) e aprovação manual antes de produção (RF-03).

Este próprio repositório de portfólio já está hospedado no GitHub — assim como o restante do portfólio de Roberto (PagFácil, ArqFlow) — o que é um dado relevante de contexto, não a decisão em si.

## Alternativas consideradas

### 1. Azure DevOps Pipelines
Prós: ALM completo integrado (Boards, Repos, Artifacts, Test Plans) em uma única plataforma; service connections nativos com Azure; federated service connections suportam Workload Identity Federation (OIDC) desde 2023, então **não há diferença de segurança de autenticação** em relação ao GitHub Actions nesse quesito; maduro em ambientes corporativos que já usam o ecossistema Microsoft para gestão ágil.
Contras: exigiria hospedar o código-fonte das aplicações em Azure Repos (ou manter uma ferramenta de CI/CD separada do repositório de código), fragmentando o fluxo de trabalho — viola RNF-16 na prática, já que passaríamos a operar duas plataformas (GitHub para o código deste portfólio e organização geral, Azure DevOps só para pipelines); GitHub Advanced Security (secret scanning, CodeQL) existe para Azure DevOps, mas como add-on licenciado à parte ("GitHub Advanced Security for Azure DevOps"), não nativo.

### 2. GitHub Actions
Prós: pipeline vive junto do repositório de código, sem sincronizar duas ferramentas (atende RNF-16 diretamente); autenticação via OIDC com Azure (`azure/login` + federated credentials) é o mesmo padrão já adotado neste projeto na ADR-0005; GitHub Advanced Security (secret scanning com push protection, CodeQL, dependency review) é nativo do GitHub, sem licença adicional — reforça RNF-15 diretamente, inclusive como mitigação exata para o tipo de incidente que ocorreu durante a configuração deste próprio repositório (exposição acidental de um token); GitHub Environments com protection rules cobre a aprovação manual antes de produção (RF-03); reusable workflows cobrem o pipeline-template por aplicação (RF-01).
Contras: não tem um módulo de Test Plans/Boards formal equivalente ao Azure DevOps — se a Vetria precisar de gestão ágil estruturada, precisaria avaliar GitHub Projects (mais simples) ou uma ferramenta dedicada à parte, fora do escopo desta ADR.

## Decisão
Adotar **GitHub Actions** como ferramenta única de CI/CD para todas as aplicações do portfólio da Vetria, com:

- **Reusable workflows**: um template central de pipeline, referenciado por cada repositório de aplicação (RF-01) — build da imagem, push para Azure Container Registry, deploy via Bicep.
- **Autenticação via OIDC**: `azure/login` com federated credentials (Workload Identity Federation, ADR-0005) — nenhum secret de longa duração armazenado no GitHub.
- **GitHub Advanced Security habilitado em todo repositório**: secret scanning com push protection (bloqueia o commit antes mesmo de chegar ao histórico), CodeQL para análise estática, dependency review.
- **GitHub Environments** para `dev`, `hml` e `prod`, com protection rules exigindo aprovação manual antes de produção (RF-03) e secrets escopados por ambiente.
- **Branch protection na `main`**: exige Pull Request revisado e checks (incluindo o gate de política da ADR-0007, RF-06) antes do merge — o mesmo padrão usado neste próprio repositório desde o início do projeto.

## Consequências
- Todo novo repositório de aplicação da Vetria passa a ser hospedado no GitHub, com o reusable workflow central como dependência de bootstrap.
- O incidente de exposição de token ocorrido durante a configuração deste projeto não se repetiria nesse desenho: push protection do GitHub Advanced Security bloqueia o commit de um secret reconhecível antes mesmo de ele ser aceito.
- Se a Vetria precisar futuramente de gestão ágil formal (Boards/Test Plans), isso é uma decisão separada — não implica trocar a ferramenta de CI/CD.
- Federated credentials por ambiente/aplicação (ADR-0005) precisam ser provisionados via Bicep como parte do bootstrap de cada novo repositório, mantendo o padrão de reprodutibilidade já estabelecido.
- Esta decisão pode ser revista se a Vetria decidir migrar o código-fonte para Azure Repos por alguma razão organizacional — não é o cenário atual.

## Referências
- RNF-11, RNF-15, RNF-16, RF-01, RF-03 e RF-06 em `02-requisitos/requisitos-nao-funcionais.md`
- ADR-0005 e ADR-0007 (`03-decisoes-arquiteturais/`)
- [Authenticate to Azure from GitHub Actions by OpenID Connect – Microsoft Learn](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)
- [Configure GitHub Advanced Security for Azure DevOps features – Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/repos/security/configure-github-advanced-security-features?view=azure-devops)
