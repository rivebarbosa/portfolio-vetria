# ADR-0003: Topologia de rede e landing zone

## Status
Aceito

## Contexto
A Vetria hoje não tem segmentação formal de rede: aplicações em App Service usam configuração default, com endpoints públicos, sem separação estrutural entre ambientes nem ponto único de auditoria de tráfego de saída. Ao desenhar a infraestrutura do zero (ADR-0001, compute; ADR-0002, observabilidade), é preciso definir a topologia de rede e a estrutura de landing zone que vai hospedar esses recursos, atendendo RNF-08 (isolamento entre ambientes) e RNF-09 (sem exposição pública de serviços de plataforma), sem violar RNF-02 (baixo overhead operacional) — a Vetria não tem, e não pretende montar, um time de rede dedicado.

## Alternativas consideradas

### 1. Sem landing zone formal (subscriptions isoladas, sem hub compartilhado)
Prós: simplicidade inicial, nenhuma dependência entre ambientes.
Contras: cada ambiente duplica recursos de segurança (firewall, DNS), custo mais alto, sem ponto único de auditoria de egress; não escala organizadamente à medida que o portfólio cresce.

### 2. Azure Virtual WAN como topologia de conectividade
Prós: roteamento totalmente gerenciado, escala bem para múltiplas regiões e filiais.
Contras: complexidade e custo desproporcionais para uma operação de região única e porte médio (viola RNF-02); o ganho real só aparece com múltiplos hubs regionais ou dezenas de spokes, cenário que a Vetria não tem hoje.

### 3. Hub-spoke clássico (CAF Landing Zone, "start small") com Private Link ponta a ponta
Prós: padrão maduro do Cloud Adoption Framework, complexidade proporcional ao porte da empresa, ponto único de egress e auditoria (Azure Firewall no hub), caminho claro de evolução para Virtual WAN se a empresa expandir para múltiplas regiões.
Contras: ainda exige management groups e subscriptions bem definidas desde o início (mitigado: escopo reduzido em relação a um enterprise-scale landing zone completo).

## Decisão
Adotar uma **landing zone hub-spoke clássica**, seguindo o padrão "start small" do Cloud Adoption Framework:

**Management groups**: Tenant Root → Vetria → {Platform, Landing Zones}. Platform agrupa Connectivity e Management; Landing Zones agrupa os ambientes de aplicação.

**Subscriptions**: uma subscription **Connectivity** (hub: Azure Firewall, DNS Private Resolver, Private DNS Zones), uma subscription **Dev/Hml** compartilhada (menor criticidade, RGs separados por ambiente) e uma subscription **Prod** isolada (blast radius e billing próprios, RNF-08).

**Rede**: VNet hub peered com as VNets spoke (Dev/Hml, Prod). Cada spoke tem: subnet para o ambiente de Container Apps (workload profile, com VNet integration), subnet para App Service VNet Integration, e subnet dedicada para Private Endpoints (ACR, Key Vault, Log Analytics, bancos de dados) — atendendo RNF-09.

**Egress**: todo tráfego de saída dos spokes passa pelo Azure Firewall no hub via UDR, como ponto único de política e auditoria — serviço gerenciado, sem exigir operação de NVA de terceiros (RNF-02).

**DNS**: Private DNS Zones centralizadas no hub, linkadas a todas as VNets spoke, garantindo resolução consistente dos nomes de Private Link em todos os ambientes.

Conectividade com sistemas on-premises (ex.: futura integração do WMS com equipamentos de armazém) fica fora de escopo desta ADR — o hub já reserva espaço de endereçamento para um gateway VPN/ExpressRoute futuro, sem provisioná-lo agora.

## Consequências
- Nenhum recurso de plataforma (ACR, Key Vault, Log Analytics, bancos de dados) terá endpoint público; todo acesso passa por Private Link.
- Novas aplicações em Container Apps precisam usar workload profiles (não o plano puramente consumption-only) para suportar VNet integration.
- O provisionamento de rede (VNets, subnets, Private DNS Zones, Firewall) passa a ser pré-requisito de infraestrutura antes de qualquer deploy de aplicação, e deve ser feito via IaC (Bicep) para se manter reprodutível entre ambientes.
- A separação Prod / Dev-Hml em subscriptions distintas simplifica RBAC e política de acesso (menos pessoas com permissão em produção).
- Esta topologia pode evoluir para Azure Virtual WAN caso a Vetria expanda para múltiplas regiões ou filiais com conectividade complexa — não é uma decisão definitiva.

## Referências
- RNF-02, RNF-08 e RNF-09 em `02-requisitos/requisitos-nao-funcionais.md`
- ADR-0001 (`03-decisoes-arquiteturais/ADR-0001-plataforma-de-compute.md`) e ADR-0002 (`03-decisoes-arquiteturais/ADR-0002-estrategia-observabilidade.md`)
- [Azure Private Link in a Hub-and-Spoke Network – Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/networking/guide/private-link-hub-spoke-network)
- [Azure Landing Zones Networking: Hub-and-Spoke vs Virtual WAN](https://exodata.io/azure-landing-zone-networking-hub-spoke-vs-virtual-wan/)
