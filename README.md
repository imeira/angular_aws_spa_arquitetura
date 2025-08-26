# Documentação da Arquitetura AWS: Angular SPA + Microserviços ECS Fargate

**Autor:** Manus AI
**Data:** 26/08/2025

## 1. Visão Geral da Solução

Esta documentação descreve uma arquitetura de referência 100% nativa da AWS, projetada para uma aplicação web moderna, escalável e resiliente. A solução é composta por um frontend Single Page Application (SPA) desenvolvido em Angular e um backend baseado em microserviços containerizados, orquestrados pelo Amazon ECS com Fargate.

A arquitetura foi desenhada para ser altamente disponível, segura e com custos otimizados, aproveitando os serviços gerenciados da AWS para reduzir a carga operacional. Os principais componentes incluem Amazon S3 e CloudFront para a entrega do frontend, API Gateway para o gerenciamento das APIs, ECS Fargate para a execução dos microserviços, Amazon RDS e DynamoDB para persistência de dados, e Amazon SQS para comunicação assíncrona, garantindo o desacoplamento entre os serviços.

## 2. Diagrama da Arquitetura

O diagrama a seguir ilustra a arquitetura completa da solução, mostrando a interação entre os diferentes serviços da AWS.

*(O diagrama gerado anteriormente, `arquitetura_aws_diagrama.png`, seria inserido aqui na documentação final.)*

## 3. Detalhamento dos Componentes

A seguir, cada componente da arquitetura é detalhado, explicando sua função e configuração principal.




### 3.1. Frontend (Angular SPA)

O frontend da aplicação é uma Single Page Application (SPA) desenvolvida com o framework Angular. A entrega e hospedagem dos arquivos estáticos (HTML, CSS, JavaScript, imagens) são otimizadas para performance e baixa latência global.

- **Amazon S3 (Simple Storage Service):** Um bucket S3 é utilizado para armazenar todos os arquivos estáticos da aplicação Angular. O bucket é configurado para *static website hosting*, mas o acesso direto é restrito. Apenas a distribuição do CloudFront terá permissão para ler os objetos do bucket, garantindo que todo o tráfego passe pela CDN.

- **Amazon CloudFront:** Atua como a Content Delivery Network (CDN) global. Ele distribui o conteúdo do frontend a partir de *edge locations* próximas aos usuários finais, o que reduz significativamente a latência. O CloudFront é configurado para fazer cache dos arquivos estáticos do S3. Além disso, ele é o ponto de entrada para todo o tráfego da aplicação, roteando tanto as requisições para o conteúdo estático quanto as chamadas de API para o backend. A configuração inclui um certificado SSL/TLS (via AWS Certificate Manager) para garantir a comunicação HTTPS.

- **Amazon Route 53:** É o serviço de DNS da AWS. Um registro de domínio (ex: `www.sua-aplicacao.com`) é configurado no Route 53 para apontar para a distribuição do CloudFront. Isso permite que os usuários acessem a aplicação através de um nome de domínio amigável.

- **AWS WAF (Web Application Firewall):** Integrado ao CloudFront, o WAF protege a aplicação contra ataques web comuns, como SQL Injection e Cross-Site Scripting (XSS). Ele inspeciona as requisições HTTP/S antes que elas cheguem à aplicação, bloqueando tráfego malicioso com base em regras de segurança predefinidas e customizadas.




### 3.2. Backend (Microserviços com ECS Fargate)

O backend é construído seguindo uma arquitetura de microserviços, onde cada serviço é responsável por uma funcionalidade de negócio específica. Os serviços são empacotados como containers Docker e executados de forma *serverless* com o AWS Fargate.

- **Amazon ECS (Elastic Container Service) com AWS Fargate:** O ECS é o orquestrador de containers da AWS. A utilização do Fargate como *launch type* elimina a necessidade de gerenciar a infraestrutura de servidores subjacente (instâncias EC2). O Fargate provisiona e gerencia os recursos computacionais necessários para os containers de forma automática. Cada microserviço é definido como um *ECS Service*, que garante que um número desejado de instâncias (tasks) do container esteja sempre em execução. O auto-scaling pode ser configurado para ajustar o número de tasks com base em métricas como uso de CPU e memória.

- **Amazon ECR (Elastic Container Registry):** É o registro de imagens Docker gerenciado da AWS. As imagens Docker de cada microserviço, construídas no pipeline de CI/CD, são armazenadas no ECR. O ECS Fargate puxa as imagens do ECR para iniciar as tasks dos serviços.

- **Application Load Balancer (ALB):** O ALB é posicionado na frente dos microserviços para distribuir o tráfego de entrada das requisições da API. Ele opera na camada 7 (aplicação) e pode rotear requisições para diferentes microserviços com base em regras de path (ex: `/api/users` para o serviço de usuários, `/api/orders` para o serviço de pedidos). O ALB está localizado em sub-redes públicas da VPC, enquanto os serviços do ECS rodam em sub-redes privadas por segurança.

- **Amazon API Gateway:** Atua como um ponto de entrada único, seguro e gerenciado para todas as APIs do backend. Ele se integra com o Application Load Balancer para rotear as chamadas para os microserviços correspondentes. O API Gateway oferece funcionalidades como controle de acesso (autenticação e autorização), throttling (limitação de requisições), caching de respostas e monitoramento de chamadas de API. Ele também pode ser usado para transformar requisições e respostas, se necessário.

- **AWS Cognito:** É o serviço de identidade para a aplicação. Ele gerencia a autenticação e autorização de usuários. O API Gateway se integra com o Cognito para validar os tokens JWT (JSON Web Tokens) enviados nas requisições, garantindo que apenas usuários autenticados e autorizados possam acessar determinados endpoints da API.




### 3.3. Persistência de Dados e Cache

A arquitetura utiliza uma abordagem de *polyglot persistence*, selecionando o banco de dados mais adequado para cada tipo de dado e caso de uso.

- **Amazon RDS (Relational Database Service):** Para dados estruturados e relacionais, como informações de usuários, pedidos e produtos, utiliza-se o Amazon RDS com o engine PostgreSQL (ou MySQL). A configuração Multi-AZ (Multiple Availability Zones) é ativada para garantir alta disponibilidade e failover automático em caso de falha em uma zona de disponibilidade.

- **Amazon DynamoDB:** Para dados não-relacionais ou que exigem altíssima performance e escalabilidade, como metadados de pedidos, logs de atividades ou carrinhos de compra, o DynamoDB é a escolha ideal. É um banco de dados NoSQL totalmente gerenciado que oferece latência de milissegundos de um dígito em qualquer escala.

- **Amazon ElastiCache:** Para caching de dados em memória, o ElastiCache com Redis é utilizado. Ele armazena dados frequentemente acessados, como sessões de usuário ou resultados de queries complexas, para reduzir a carga sobre os bancos de dados principais e acelerar o tempo de resposta das APIs.

### 3.4. Comunicação Assíncrona e Desacoplamento

Para garantir o desacoplamento entre os microserviços e permitir a comunicação assíncrona, a arquitetura faz uso intensivo de serviços de mensageria.

- **Amazon SQS (Simple Queue Service):** É um serviço de filas de mensagens totalmente gerenciado. As filas SQS são usadas para desacoplar a comunicação entre os microserviços. Por exemplo, quando um pedido é criado, o serviço de pedidos pode enviar uma mensagem para uma fila SQS. O serviço de pagamentos e o serviço de notificações podem então consumir mensagens dessa fila para processar o pagamento e enviar uma confirmação ao usuário, respectivamente. Isso torna o sistema mais resiliente, pois uma falha em um serviço consumidor não impacta o serviço produtor.

- **Amazon SNS (Simple Notification Service):** É um serviço de pub/sub (publish/subscribe). O SNS é usado para distribuir mensagens para múltiplos consumidores. Por exemplo, um evento de "pedido criado" pode ser publicado em um tópico SNS. Diferentes microserviços (notificação, faturamento, análise de dados) podem se inscrever nesse tópico para receber a notificação e agir de acordo.

- **Amazon EventBridge:** É um barramento de eventos (event bus) *serverless* que facilita a construção de arquiteturas orientadas a eventos. Ele pode ser usado para rotear eventos de forma mais complexa e integrada, não apenas de microserviços, mas também de outros serviços AWS e aplicações SaaS de terceiros. O EventBridge permite a criação de regras para filtrar e transformar eventos antes de enviá-los aos alvos.




### 3.5. Segurança

A segurança é um pilar fundamental da arquitetura, implementada em múltiplas camadas.

- **AWS IAM (Identity and Access Management):** Define permissões granulares para todos os recursos da AWS, seguindo o princípio do menor privilégio. Cada serviço e usuário tem apenas as permissões estritamente necessárias para realizar suas funções.
- **AWS Secrets Manager:** Armazena e gerencia de forma segura as credenciais de banco de dados, chaves de API e outros segredos. Os microserviços recuperam os segredos do Secrets Manager em tempo de execução, evitando que sejam expostos no código-fonte.
- **VPC (Virtual Private Cloud):** Isola os recursos da aplicação em uma rede virtual privada. O uso de sub-redes públicas e privadas, juntamente com Security Groups e Network ACLs, controla o tráfego de entrada e saída, garantindo que apenas as comunicações autorizadas sejam permitidas.

### 3.6. Monitoramento e Logs

- **Amazon CloudWatch:** É o serviço central para monitoramento e observabilidade. Ele coleta métricas, logs e traces de todos os serviços da AWS utilizados. Alarmes podem ser configurados para notificar a equipe de operações sobre comportamentos anômalos ou falhas. Dashboards personalizados fornecem uma visão consolidada da saúde da aplicação.
- **AWS X-Ray:** Ajuda a analisar e depurar aplicações distribuídas, como as baseadas em microserviços. Ele fornece uma visão do fluxo de requisições através dos diferentes serviços, facilitando a identificação de gargalos de performance e erros.

### 3.7. CI/CD (Continuous Integration/Continuous Deployment)

Um pipeline de CI/CD automatizado é essencial para entregar novas funcionalidades de forma rápida e segura.

- **AWS CodePipeline:** Orquestra o pipeline de build, teste e deploy.
- **AWS CodeBuild:** Compila o código-fonte, executa testes e produz os artefatos, como as imagens Docker.
- **AWS CodeDeploy:** Automatiza o deploy das novas versões dos microserviços no ECS Fargate, utilizando estratégias como blue/green para minimizar o tempo de inatividade.

## 4. Conclusão

A arquitetura apresentada oferece uma base robusta, escalável e segura para o desenvolvimento e operação de uma aplicação web moderna na AWS. Ao alavancar os serviços gerenciados, a equipe de desenvolvimento pode focar na entrega de valor para o negócio, enquanto a AWS cuida da complexidade da infraestrutura subjacente. A combinação de ECS Fargate, microserviços, bancos de dados especializados e comunicação assíncrona resulta em um sistema resiliente, desacoplado e preparado para o crescimento futuro.


