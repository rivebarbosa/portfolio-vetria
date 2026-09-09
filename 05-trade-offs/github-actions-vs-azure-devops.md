# Trade-off: GitHub Actions vs. Azure DevOps Pipelines

Detalhamento da comparação que embasa a escolha de ferramenta de CI/CD na ADR-0008.

## Critérios avaliados

| Critério | Azure DevOps Pipelines | GitHub Actions |
|---|---|---|
| Onde o pipeline vive em relação ao código | Separado (Azure Repos ou repo externo + Azure Pipelines) | Junto do repositório (GitHub) |
| Autenticação sem credenciais de longa duração (OIDC) | Suportado — federated service connections desde 2023 | Suportado — `azure/login` + federated credentials |
| Diferença de segurança de autenticação entre as duas | **Nenhuma — paridade completa** | **Nenhuma — paridade completa** |
| Secret scanning / code scanning (SAST) | GitHub Advanced Security for Azure DevOps — add-on licenciado à parte | GitHub Advanced Security — nativo, sem licença adicional |
| Push protection (bloqueia secret antes do commit) | Disponível via o add-on acima | Nativo |
| ALM completo (Boards, Test Plans) | Sim, integrado nativamente | Não — GitHub Projects é mais simples, sem Test Plans equivalente |
| Aprovação manual por ambiente | Environments/approvals nativos | GitHub Environments com protection rules |
| Reutilização de pipeline entre aplicações | Templates YAML reutilizáveis | Reusable workflows |
| Ferramentas a operar, dado que o código já está no GitHub | Duas (GitHub para código + Azure DevOps para pipeline) | Uma |

## Por que GitHub Actions foi escolhido

1. **RNF-16 (unicidade de ferramenta) pesa contra o Azure DevOps aqui especificamente** — não porque o Azure DevOps seja tecnicamente inferior, mas porque o código já está hospedado no GitHub. Usar Azure DevOps só para pipelines recriaria a fragmentação que o discovery já identificou como dor, só que entre ferramentas em vez de entre squads.
2. **OIDC é paridade, não diferencial.** Vale registrar isso explicitamente: as duas ferramentas resolvem a autenticação sem secret de longa duração (RNF-11) igualmente bem hoje. Quem decidiu essa ADR não foi esse critério.
3. **GitHub Advanced Security nativo vs. add-on.** A funcionalidade existe nas duas, mas no GitHub ela é parte do produto; no Azure DevOps é uma licença adicional ("GitHub Advanced Security for Azure DevOps"). Isso simplifica RNF-15 na prática.
4. **O ganho que o Azure DevOps teria (ALM completo) não é um requisito hoje.** A Vetria não pediu Boards ou Test Plans formais — se pedir no futuro, é uma decisão independente da ferramenta de CI/CD.

## O que essa decisão não é
Não é um veredito de que Azure DevOps é pior. Em uma empresa que já hospeda código em Azure Repos, ou que depende de Test Plans/Boards integrados, a mesma análise levaria à conclusão oposta — o critério decisivo aqui foi consolidação de ferramentas em torno de onde o código já vive, não uma vantagem técnica isolada do GitHub Actions.

## Referências
- ADR-0008 (`03-decisoes-arquiteturais/ADR-0008-ferramenta-de-cicd.md`)
- [Authenticate to Azure from GitHub Actions by OpenID Connect – Microsoft Learn](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)
- [Configure GitHub Advanced Security for Azure DevOps features – Microsoft Learn](https://learn.microsoft.com/en-us/azure/devops/repos/security/configure-github-advanced-security-features?view=azure-devops)
