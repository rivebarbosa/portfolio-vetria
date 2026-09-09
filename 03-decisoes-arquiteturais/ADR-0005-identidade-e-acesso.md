# ADR-0005: Estratégia de identidade e acesso (Microsoft Entra ID)

## Status
Aceito

## Contexto
As ADRs anteriores definiram onde as aplicações rodam (ADR-0001), como são observadas (ADR-0002), a topologia de rede (ADR-0003) e a região dos dados (ADR-0004) — mas nenhuma tratou de quem e o quê pode acessar esses recursos. A Vetria precisa de uma estratégia de identidade cobrindo três frentes distintas: acesso humano (squads e administradores às subscriptions da landing zone), acesso de workload (aplicações a Key Vault, Storage, ACR, bancos de dados) e acesso de pipeline (CI/CD provisionando e implantando recursos).

O incidente de exposição acidental de um personal access token durante a configuração deste próprio repositório, no início deste projeto, é um lembrete concreto e recente do risco de credenciais de longa duração — a decisão de identidade da Vetria precisa evitar esse padrão de risco na infraestrutura que está sendo desenhada, não só reconhecê-lo em retrospecto.

A Vetria também opera um Portal de Pedidos usado por revendedores externos, o que introduz uma necessidade de identidade de cliente/parceiro distinta da identidade de colaboradores internos.

## Alternativas consideradas

### 1. Service principals com client secret para pipelines e aplicações
Prós: simples de configurar, amplamente documentado.
Contras: secrets de longa duração precisam ser armazenados e rotacionados manualmente, e representam superfície de ataque permanente — o mesmo padrão de risco que se materializou ao autenticar a ferramenta de linha de comando usada neste projeto.

### 2. Acesso humano via atribuição direta de roles permanentes (Owner/Contributor) por subscription
Prós: simplicidade operacional inicial.
Contras: viola least privilege, não deixa trilha de auditoria de por que alguém tinha acesso a produção em um dado momento, e não escala à medida que squads mudam (viola RNF-02 a médio prazo).

### 3. Microsoft Entra ID como IdP único, Managed Identity para workloads, Workload Identity Federation (OIDC) para pipelines, RBAC via grupos + PIM para acesso humano, Entra External ID para o Portal de Pedidos
Prós: elimina credenciais de longa duração (RNF-11) — workloads e pipelines nunca guardam secret; acesso humano a produção é elevado sob demanda e por tempo limitado (RNF-12), auditável; separa claramente identidade de colaborador da identidade de cliente/parceiro externo.
Contras: mais peças para configurar inicialmente (app registrations, federated credentials, políticas de PIM) — mitigado por ser configuração única via IaC, não recorrente.

## Decisão

**Acesso humano**: Microsoft Entra ID como identity provider único da Vetria. Nenhuma atribuição de role diretamente a usuário — sempre via grupos do Entra ID mapeados a roles Azure RBAC por subscription (Connectivity, Dev/Hml, Prod — ADR-0003). Acesso a **Prod** nunca é permanente: usa **Microsoft Entra PIM**, com elevação just-in-time, tempo limitado e aprovação. MFA obrigatório via Conditional Access para qualquer acesso administrativo.

**Acesso de workload**: toda aplicação em Container Apps ou App Service usa **Managed Identity** (preferencialmente user-assigned, para permitir reuso e rotação controlada) para acessar Key Vault, Storage, Azure Container Registry e bancos de dados — nunca connection string ou chave de acesso em configuração de aplicação (RF-05).

**Acesso de pipeline**: pipelines de CI/CD (RF-01) autenticam via **Workload Identity Federation (OIDC)** — um app registration por ambiente/pipeline, com federated credential atrelado ao repositório e branch/ambiente específico, sem client secret armazenado em lugar nenhum.

**Identidade de cliente/parceiro**: o Portal de Pedidos usa **Microsoft Entra External ID** para autenticação de revendedores externos, mantendo essa população de identidade separada do tenant de colaboradores internos.

## Consequências
- Nenhum secret de longa duração (client secret, connection string, chave de storage) deve existir em código, pipeline ou configuração de aplicação a partir de agora — isso se torna critério de revisão em qualquer PR de infraestrutura.
- Acesso a produção exige elevação PIM, com custo de fricção operacional em troca de auditoria e redução de superfície de ataque.
- Todo app registration e federated credential deve ser provisionado via Bicep, mantendo o padrão de reprodutibilidade já estabelecido nas ADRs anteriores.
- A separação entre tenant de colaboradores e Entra External ID evita que uma política de Conditional Access pensada para funcionários vaze para o público externo, ou vice-versa.
- Grupos do Entra ID usados para RBAC precisam de dono claro e revisão periódica de membros — governança operacional fora do escopo técnico desta ADR.

## Referências
- RNF-01, RNF-02, RNF-09, RNF-11, RNF-12 e RF-05 em `02-requisitos/requisitos-nao-funcionais.md`
- ADR-0001 e ADR-0003 (`03-decisoes-arquiteturais/`)
- [Landing zone identity and access management – Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/identity-access-landing-zones)
- [Workload Identity Federation – Microsoft Entra Workload ID](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- [Best practices for Microsoft Entra roles](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/best-practices)
