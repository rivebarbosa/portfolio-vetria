# Discovery — Vetria Distribuição

## Sobre a empresa

Vetria Distribuição é uma empresa fictícia de distribuição/logística B2B de porte médio, atuando na venda e distribuição de produtos para revendedores em todo o Brasil. Está em processo de expansão digital, ampliando o autoatendimento de clientes e a automação de operações internas.

## Situação atual

- Portfólio de aplicações internas cresceu de forma orgânica ao longo dos anos: portal de pedidos, WMS (gestão de armazém), TMS (gestão de transporte), CRM interno e módulo financeiro.
- Todas as aplicações hoje rodam em Azure App Service (Web Apps), cada uma com seu próprio pipeline de deploy.
- As aplicações não têm relação direta entre si — não compartilham banco de dados, não se comunicam de forma síncrona/assíncrona entre elas, e são mantidas por squads diferentes.
- Não existe um padrão único de CI/CD entre as aplicações; cada squad monta o pipeline à sua maneira.

## Dores

- Dificuldade de padronizar deploys e observabilidade entre aplicações.
- Dúvida recorrente sobre se a evolução da infraestrutura deveria migrar para Kubernetes (AKS), motivada por benchmarks de mercado, sem uma avaliação formal do real ganho para o cenário da empresa.
- Ausência de registro formal das decisões de arquitetura tomadas até aqui.

## Objetivo deste caso de estudo

Desenhar, do zero, a arquitetura de soluções da Vetria — começando pela infraestrutura de compute na Azure e pela estratégia de CI/CD — com decisões registradas via ADR e diagramas C4, servindo tanto como direcionamento técnico quanto como peça de portfólio.

## Stakeholders (fictícios)

- Head de Tecnologia — patrocinador da iniciativa
- Squads de Portal de Pedidos, WMS, TMS, CRM e Financeiro — donos das aplicações
- Arquiteto de Soluções — condução deste case
