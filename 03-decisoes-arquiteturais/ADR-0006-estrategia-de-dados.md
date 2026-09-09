# ADR-0006: Estratégia de dados

## Status
Aceito

## Contexto
As aplicações da Vetria (portal de pedidos, WMS, TMS, CRM, financeiro) são independentes entre si (RNF-01) e hoje não têm um padrão definido de banco de dados — cada squad decidiu individualmente no passado. Ao desenhar a arquitetura de dados do zero, é preciso definir um padrão que respeite o isolamento entre aplicações, a residência de dados em Brazil South com redundância local/zonal (ADR-0004), o acesso via Microsoft Entra ID/Managed Identity em vez de senha (ADR-0005, RF-05) e a ausência de exposição pública (RNF-09, ADR-0003). O portal de pedidos e o CRM também manipulam dados pessoais de clientes/revendedores e colaboradores, o que exige tratamento específico em ambientes não produtivos (RNF-13).

## Alternativas consideradas

### 1. Banco de dados compartilhado entre aplicações
Prós: menor número de recursos para operar.
Contras: viola RNF-01 diretamente — cria acoplamento de dados entre aplicações sem relação de negócio; mudança de schema de uma aplicação passa a poder quebrar outra.

### 2. Um único motor de banco de dados obrigatório para todas as aplicações
Prós: padronização máxima, uma única expertise operacional.
Contras: não considera que cargas diferentes (transacional financeiro vs. dados geoespaciais de rastreamento do TMS) se beneficiam de modelos de dados diferentes; força soluções alternativas onde um motor mais adequado resolveria de forma nativa.

### 3. Banco de dados dedicado por aplicação (polyglot), com Azure Database for PostgreSQL Flexible Server como padrão e Azure SQL Database como alternativa justificada
Prós: mantém isolamento entre aplicações (RNF-01); PostgreSQL Flexible Server cobre a maioria dos casos transacionais com custo de licenciamento menor que SQL Server, além de suportar a extensão PostGIS para os dados geoespaciais do TMS; Azure SQL Database fica disponível quando uma aplicação já depende de recursos específicos do T-SQL/ecossistema Microsoft.
Contras: dois motores para operar em vez de um — mitigado por ser exceção justificada, não a regra.

## Decisão
Adotar **um banco de dados dedicado por aplicação** — nenhuma aplicação compartilha schema ou instância com outra. O motor padrão é **Azure Database for PostgreSQL Flexible Server**; **Azure SQL Database** é aceito como alternativa quando a aplicação já depende de recursos específicos do ecossistema Microsoft/T-SQL. O TMS usa PostgreSQL com a extensão **PostGIS** para dados geoespaciais de rastreamento; se o volume de eventos de rastreamento em tempo real justificar futuramente, Azure Cosmos DB pode ser avaliado como complemento — não como substituto do banco transacional.

Todos os bancos: ficam em **Brazil South**, sem endpoint público, acessíveis apenas via **Private Endpoint** na subnet dedicada de cada spoke (ADR-0003); usam **Microsoft Entra ID para autenticação**, acessados pela aplicação via **Managed Identity** em vez de usuário/senha (RF-05); têm backup automatizado com redundância **LRS/ZRS** apenas — nunca geo-redundante (ADR-0004).

Dados pessoais de clientes, revendedores ou colaboradores **não existem em sua forma real em Dev/Hml** — são mascarados, anonimizados ou substituídos por dados sintéticos antes de qualquer cópia a partir de produção (RNF-13).

## Consequências
- Cada squad passa a ser responsável pelo ciclo de vida do próprio banco de dados (schema, migrations), sem depender de uma equipe central de DBA para mudanças de rotina.
- O pipeline de CI/CD de cada aplicação (RF-01) passa a incluir um passo de migração de schema (ex.: Flyway ou EF Core Migrations, conforme a stack) antes do deploy da aplicação.
- É necessário um processo de mascaramento/geração de dados sintéticos para popular Dev/Hml a partir de Prod — uma etapa que não existia antes.
- Manter dois motores de banco de dados (PostgreSQL como padrão, SQL Database como exceção) exige que a equipe mantenha conhecimento operacional em ambos, ainda que um seja predominante.
- Esta decisão não cobre um data warehouse/data lake para analytics entre aplicações — fora de escopo, a ser tratado se essa necessidade surgir.

## Referências
- RNF-01, RNF-09, RNF-13 e RF-05 em `02-requisitos/requisitos-nao-funcionais.md`
- ADR-0003, ADR-0004 e ADR-0005 (`03-decisoes-arquiteturais/`)
- [Microsoft Entra Authentication in Azure Database for PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/security/security-entra-concepts)
- [Network with Private Access in Azure Database for PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/network/concepts-networking-private)
