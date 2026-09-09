# ADR-0007: Segurança e governança

## Status
Aceito

## Contexto
As ADR-0001 a ADR-0006 estabeleceram um conjunto extenso de padrões: região única (ADR-0004), ausência de exposição pública (ADR-0003), redundância local/zonal apenas (ADR-0004), identidade sem credenciais de longa duração (ADR-0005), bancos isolados por aplicação (ADR-0006). Até aqui, nada impede que um recurso fora desses padrões seja criado por engano — via portal, CLI ou um template Bicep mal configurado. RNF-14 exige que a conformidade seja **validada automaticamente**, não apenas descrita em documentação, e RNF-15 exige que segredos nunca fiquem em texto plano em pipeline, repositório ou configuração.

## Alternativas consideradas

### 1. Conformidade garantida por revisão manual de Pull Request
Prós: nenhuma ferramenta adicional para configurar.
Contras: não escala, depende de o revisor lembrar de cada regra das sete ADRs anteriores, não impede criação de recurso fora do fluxo de PR (ex.: via portal Azure diretamente); viola RNF-14.

### 2. Ferramenta de terceiros para policy-as-code (ex.: Open Policy Agent) apenas no pipeline
Prós: portável entre provedores de nuvem.
Contras: só bloqueia recursos criados via pipeline — não impede criação direta no portal ou CLI; adiciona uma ferramenta extra para operar (viola RNF-02) quando o provedor já oferece o equivalente nativo.

### 3. Azure Policy nativo nos management groups + Microsoft Defender for Cloud + Azure Key Vault por aplicação
Prós: Azure Policy se aplica a qualquer forma de criação de recurso (portal, CLI, IaC), não só ao pipeline; herda naturalmente a hierarquia de management groups já definida na ADR-0003; Defender for Cloud dá postura de segurança contínua e scanning de vulnerabilidade de imagens no ACR sem operação manual; Key Vault por aplicação mantém o padrão de isolamento já usado em observabilidade (ADR-0002) e dados (ADR-0006).
Contras: exige desenhar as iniciativas de política corretamente desde o início — mitigado por serem definidas uma vez, via IaC, e herdadas por toda a landing zone.

## Decisão

**Azure Policy**, atribuído nos management groups **Platform** e **Landing Zones** (ADR-0003), com as seguintes iniciativas obrigatórias:
- **Allowed locations**: apenas Brazil South — torna a ADR-0004 uma regra bloqueante, não apenas um acordo documentado.
- **Deny public network access**: aplicado a Storage, Key Vault, Azure Container Registry e bancos de dados — reforça RNF-09/ADR-0003.
- **Allowed storage redundancy**: apenas LRS/ZRS — reforça ADR-0004.
- **Require diagnostic settings** enviando para o Log Analytics workspace do ambiente — reforça ADR-0002.
- **Required tags**: `owner`, `aplicacao`, `ambiente` em todo recurso — governança de custo e responsabilidade.

**Microsoft Defender for Cloud** habilitado em todas as subscriptions para postura de segurança contínua (CSPM) e scanning de vulnerabilidade das imagens de container publicadas no Azure Container Registry (ADR-0001).

**Azure Key Vault**: um por aplicação (isolamento, RNF-01), acessado exclusivamente via Managed Identity (ADR-0005) e Private Endpoint (ADR-0003), residente em Brazil South. Um Key Vault de plataforma, na subscription Connectivity, guarda segredos compartilhados (ex.: certificados wildcard, se surgirem). Nenhum segredo, chave ou connection string existe fora de um Key Vault (RNF-15). Expiração de segredos e certificados é monitorada via os alertas já definidos na ADR-0002.

Todo pipeline de infraestrutura roda validação de política **antes** do deploy, com bloqueio automático em caso de não conformidade (RF-06) — como segunda camada, já que o Azure Policy também bloqueia na própria plataforma.

## Consequências
- Um recurso fora de conformidade (região errada, redundância errada, endpoint público) passa a ser **rejeitado na criação**, não descoberto depois em auditoria.
- Squads perdem a possibilidade de criar recursos "no improviso" fora dos padrões — ganho de conformidade com custo de menor flexibilidade individual.
- Provisionar um novo Key Vault por aplicação passa a ser parte padrão do template Bicep de toda nova aplicação, junto com Application Insights (ADR-0002) e o banco de dados (ADR-0006).
- Defender for Cloud adiciona custo (planos pagos além do CSPM gratuito, conforme os recursos habilitados) — avaliado como aceitável frente ao ganho de visibilidade de vulnerabilidades.
- Exceções às políticas (quando genuinamente necessárias) exigem aprovação explícita e registro — não podem ser contornadas silenciosamente.

## Referências
- RNF-02, RNF-09, RNF-14, RNF-15 e RF-06 em `02-requisitos/requisitos-nao-funcionais.md`
- ADR-0001 a ADR-0006 (`03-decisoes-arquiteturais/`)
- [Azure Landing Zone Security Baseline – Protego](https://protego.me/blog/azure-landing-zone-security-baseline-2026)
- [Vulnerability management for containers – Microsoft Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/agentless-vulnerability-assessment-azure)
