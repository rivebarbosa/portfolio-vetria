# Trade-off: Azure Database for PostgreSQL Flexible Server vs. Azure SQL Database

Detalhamento da comparação que embasa a escolha de motor de banco de dados padrão na ADR-0006.

## Critérios avaliados

| Critério | Azure SQL Database | PostgreSQL Flexible Server |
|---|---|---|
| Custo (sem Azure Hybrid Benefit) | Referência (licença SQL Server embutida no vCore) | ~40–45% mais barato (open-source, só paga infraestrutura) |
| Extensibilidade | Limitada — spatial types, CLR assemblies, linked servers | Rica — PostGIS, pgvector, Citus, TimescaleDB, pg_cron |
| Portabilidade | Ecossistema Microsoft | Padrão aberto, portável entre clouds/on-prem |
| Alta disponibilidade / zona | Zone redundancy + active geo-replication | Zone redundancy + read replicas |
| Autenticação Microsoft Entra ID / Managed Identity | Suportado | Suportado |
| Geo-replicação entre regiões | Disponível, mas **fora de cogitação** — viola ADR-0004 (residência de dados no Brasil) | Idem — não se aplica de qualquer forma |
| Melhor cenário de uso | Stack T-SQL já existente, ou licenças SQL Server disponíveis para Hybrid Benefit | Começando do zero, custo relevante, necessidade de extensões (geoespacial, vetorial) |

## Por que PostgreSQL Flexible Server foi escolhido como padrão

1. **Sem legado para aproveitar Hybrid Benefit.** A Vetria está desenhando a arquitetura do zero, sem licenças SQL Server on-premises — a única vantagem de custo do Azure SQL Database (desconto de 30–40% via Hybrid Benefit) simplesmente não existe aqui. Sem ela, PostgreSQL Flexible Server fica de 40 a 45% mais barato em vCores equivalentes.
2. **O TMS precisa de dados geoespaciais nativos.** PostGIS é o padrão de mercado para esse tipo de carga, mais maduro que os spatial types do SQL Server.
3. **Alta disponibilidade é equivalente**, e a vantagem do Azure SQL em geo-replicação entre regiões não se aplica — a ADR-0004 já proíbe replicação de dados para fora do Brasil.
4. **Autenticação via Microsoft Entra ID é suportada nos dois**, então não há diferença nesse quesito (ADR-0005).

Com custo menor, extensibilidade maior e nenhuma desvantagem real de HA ou segurança para o cenário da Vetria, manter Azure SQL Database como padrão significaria pagar mais por paridade de recursos. Por isso ele fica como **exceção justificada** (ADR-0006) — usado apenas se uma aplicação específica já depender de recursos nativos do T-SQL.

## Referências
- ADR-0006 (`03-decisoes-arquiteturais/ADR-0006-estrategia-de-dados.md`)
- ADR-0004 (`03-decisoes-arquiteturais/ADR-0004-residencia-de-dados-regiao.md`)
- [Pricing – Azure Database for PostgreSQL Flexible Server](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/)
- [Azure SQL vs PostgreSQL in Azure – CloudWebSchool](https://cloudwebschool.com/docs/azure/service-comparisons/azure-sql-vs-postgresql-in-azure/)
