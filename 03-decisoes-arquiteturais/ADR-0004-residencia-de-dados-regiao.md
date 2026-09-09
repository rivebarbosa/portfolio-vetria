# ADR-0004: Residência de dados e seleção de região

## Status
Aceito

## Contexto
A Vetria estabeleceu uma restrição transversal: todos os dados da empresa — aplicacionais, telemetria e backups — devem permanecer em território brasileiro (RNF-10). Essa restrição não foi considerada explicitamente nas ADR-0001 a ADR-0003 e precisa ser formalizada como critério obrigatório para qualquer recurso provisionado, retroativo às decisões já tomadas.

O Azure tem duas regiões no Brasil: **Brazil South** (São Paulo) — GA, com 3 availability zones, madura, com todos os serviços usados neste projeto até agora — e **Brazil Southeast** (Rio de Janeiro) — região restrita, que exige solicitação de acesso específica à Microsoft e tem disponibilidade de serviços mais limitada.

Um ponto que não é óbvio e precisa ser tratado explicitamente: a **região par (paired region) de Brazil South é South Central US**, nos Estados Unidos. Isso significa que qualquer recurso configurado com replicação geográfica padrão (Storage GRS/RA-GRS, backup com Cross-Region Restore, SQL auto-failover groups, Site Recovery) replicaria dados para fora do Brasil por padrão — mesmo que a região primária esteja correta. Esse é exatamente o tipo de detalhe que viola RNF-10 silenciosamente se não for tratado no desenho.

## Alternativas consideradas

### 1. Multi-região dentro do Brasil (Brazil South + Brazil Southeast)
Prós: alta disponibilidade regional sem sair do país.
Contras: Brazil Southeast é uma região restrita (acesso mediante solicitação e aprovação da Microsoft) e não há confirmação de que todos os serviços já decididos (Container Apps, Managed Grafana) estejam disponíveis lá; adiciona complexidade e overhead operacional desproporcional ao porte da empresa (RNF-02).

### 2. Brazil South com redundância geográfica padrão (GRS) para o par South Central US
Prós: maior resiliência a desastre regional, menor RTO/RPO em caso de perda total da região.
Contras: viola diretamente RNF-10 — os dados replicados passam a residir fisicamente nos EUA, mesmo que apenas para fins de disaster recovery. Inaceitável dado o requisito.

### 3. Brazil South como região única, com redundância local/zonal (LRS/ZRS)
Prós: dados nunca saem do país, usa as 3 availability zones de Brazil South para resiliência intra-região (ZRS) sem depender de outra geografia; atende RNF-10 sem exigir uma segunda região restrita.
Contras: RTO/RPO maiores em caso de desastre que afete a região inteira (cenário raro, mas não impossível) — não há failover automático para outra região.

## Decisão
Adotar **Brazil South** como região única para todos os recursos da Vetria. Toda configuração de redundância deve usar **LRS (Locally Redundant Storage) ou ZRS (Zone-Redundant Storage)** — nunca GRS, RA-GRS ou GZRS, que replicariam dados para South Central US. Recuperação de desastre de região inteira é tratada via backup dentro da própria Brazil South combinado com reprovisionamento rápido via Infraestrutura como Código (Bicep), aceitando RTO/RPO maiores como trade-off consciente pela soberania de dados.

Isso é retroativo às três ADRs anteriores:
- **ADR-0001** (compute): Azure Container Apps e Azure App Service confirmados disponíveis em Brazil South.
- **ADR-0002** (observabilidade): Log Analytics, Application Insights e Azure Monitor managed Prometheus disponíveis em Brazil South. **Atenção**: a disponibilidade do Azure Managed Grafana em Brazil South deve ser confirmada no momento do provisionamento; caso não esteja disponível na região, usar Azure Workbooks como alternativa para os dashboards cross-app — os dados de telemetria em si permanecem em Brazil South independentemente da ferramenta de visualização.
- **ADR-0003** (rede/landing zone): hub-spoke, subscriptions e Private Link não têm dependência de região — todas as VNets, subnets e recursos de conectividade devem ser criados em Brazil South.

## Consequências
- Todo template Bicep passa a fixar `brazilsouth` como região e a definir explicitamente LRS/ZRS em qualquer recurso de storage/dados — o padrão de alguns serviços é GRS, então essa configuração nunca pode ser deixada implícita.
- Não há failover automático entre regiões; um incidente que derrube Brazil South inteira exige reprovisionamento manual/via pipeline, com RTO maior do que uma topologia multi-região teria.
- A disponibilidade do Azure Managed Grafana em Brazil South precisa ser validada antes do provisionamento (ADR-0002); Azure Workbooks é o plano de contingência.
- Contratos com fornecedores terceiros que eventualmente processem dados da Vetria (ex.: gateways de pagamento, se surgirem no futuro) precisam ser avaliados quanto a residência de dados — fora do escopo técnico desta ADR, mas registrado como ponto de atenção para jurídico/compliance.
- Esta decisão pode ser revista se Brazil Southeast deixar de ser uma região restrita e alcançar paridade de serviços com Brazil South.

## Referências
- RNF-02 e RNF-10 em `02-requisitos/requisitos-nao-funcionais.md`
- ADR-0001, ADR-0002 e ADR-0003 (`03-decisoes-arquiteturais/`)
- [Data Residency in Azure – Microsoft Azure](https://azure.microsoft.com/en-us/explore/global-infrastructure/data-residency)
- [List of Azure regions – Microsoft Learn](https://learn.microsoft.com/en-us/azure/reliability/regions-list)
