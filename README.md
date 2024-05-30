# Desafio - Arquitetura de uma rede social de conteúdo

## Diagrama de arquitetura

O desenho abaixo visa contemplar como serão realizadas as comunicações entre diferentes serviços na rede social.

![desafio-nt (3)](https://github.com/renanpadilha/desafio-arq/assets/5349447/40b602fe-4f7c-410e-8aff-62c13f28e956)

As requisições dos diferentes clients, através de HTTP, passariam por um BFF (Backend For Frontend), por exemplo, o mobile teria um bff próprio busca dados diferentes do que de uma aplicação web, assim podemos ter maior performance, já que o próprio design de tela mobile exige menos dados e carregamentos do que uma aplicação web. Esses BFFs seriam responsáveis por agregar e requisitar diferentes pontos da aplicação.

No caso de uma publicação, as informações são enviadas para o serviço `post-service` que grava as informações principais da publicação, como texto, dados do usuário e caso hajam mídias, salva num bucket de forma assíncrona, retornando somente referências para serem armazenadas na base de dados, tornando a requisição mais transparente para o usuário, assim que pronta, a publicação ficaria disponível.

No caso de um comentário, a mesma coisa acontece, passando do bff para o `comment-service` os dados da publicação + o conteúdo do comentário.

Todas essas duas transações podem ser armazenadas no Redis para manter cache desses conteúdos, a estratégia de cache será definida conforme o negócio definir, pode ser os mais frequentes acessados, podem ser os últimos publicados, pode ser por tempo (TTL).

Qualquer um dos dois serviços (`post-service` ou `comment-service`), ao receber a requisição, publicaria num tópico a informação de que um novo conteúdo/comentário foi criado, o consumidor desse tópico estaria num outro seviço que poderia notificar de várias formas de acordo com as configurações daquele usuário, por exemplo, selecionar somente notificação por e-mail ou só por push.

A gestão de acesso e identidade ficaria a cargo do Keycloak, que conteria as políticas de acesso de cada usuário.

### Escalabilidade e Disponibilidade

A disponibilidade da aplicação se dá através de vários fatores:
- Trabalhar com uma arquitetura de microsserviços podemos retirar o alto acoplamento entre os domínios da aplicação e o isolamento de possíveis indisponibilidades;
- Trabalhar com cache dos dados também ajuda a garantir a alta disponibilidade do banco do dados, devolvendo dados direto do cache ao invés de ir a base de dados o tempo todo consultar o mesmo dado;
- Trabalhar com o Kubernetes podemos escalar os serviços de maneira horizontal de acordo com a demanda de cada serviço;
- Trabalhar com banco de dados não relacional também, já que a escalabilidade pode ser realizada de forma horizontal e em alguns nós e réplicas separadas geograficamente;

Alguns pontos únicos de falhas e possíveis mitigadores são:

- SSO: pelo fato do SSO ser centralizado, qualquer falha pode fazer com que a aplicação não funcione corretamente, adicionando o SSO também gerenciado por um Kubernetes pode mitigar esse problema, armazenar sessões e tokens no cache (somente durante o período de validade) também pode mitigar esse problema;
- Microsserviços: Serviços que não são cores, por exemplo, se os serviços de comentários e notificações caírem, o de publicações pode se manter independente dos outros;
- Sistema de cache: Caso o banco de dados fique indisponível por alguns instantes, a aplicação pode funcionar com o cache existente.

### Armazenamento de Dados

Sobre o armazenamento dos dados, ambos os tipos de bancos de dados, desde que bem otimizados dariam conta, porém, nessa questão eu escolheria um banco de dados não relacional, segundo o teorema CAP teríamos alta disponibilidade, tolerância a partição, porém baixa consistência, que não é o requisito mais importante para esse tipo de aplicação e sim a alta disponibilidade/escalabilidade. Sendo assim, tendo uma massiva alteração dos dados, não necessitar de uma consisência alta e não precisar realizar grandes operações com joins um banco não relacional é a melhor opção.

### Processamento de Conteúdo

O processamento de conteúdo será realizado da seguinte forma: os textos seriam armazenados na estrutura do próprio banco de dados, mídias como imagens e vídeos seriam pré-processados com algum tipo de compressão dos arquivos e após, enviados de maneira assíncrona para algum sistema de armazenamento de arquivos como o s3 da Amazon, mantendo no banco de dados somente o apontamento para esses arquivos.


### Notificações em tempo real

Para realizar as notificações em real time nesse caso a melhor solução seria realizá-las de forma assíncrona utilizando o Kafka ou o próprio Pub/Sub do Redis.

A cada publicação ou comentário produziríamos uma mensagem num tópico que seria consumida num microsserviço de notificação e realizariam a entrega daquela notificação.

Caso de uso:
Um usuário realizou um comentário numa publicação, ao receber a requisição no serviço de comentários, uma mensagem seria publicada com os dados da publicação/comentário para que fosse lido no serviço de notificações e conectasse com os vendors de push ou e-mail, por exemplo.


### SSO

O SSO deve ser um sistema centralizado, como o Keycloak, que provê um sistema de autenticação e autorização, configuraríamos os usuários, roles e scopes para que os diferentes serviços garantam que os usuários tenham acesso a determinados recursos dentro da aplicação, delegando pra aplicação somente a decodificação do token e realizando a autorização dos recursos conforme configuração do usuário no SSO. 

Exemplo: um usuário logado tem acesso a realizar comentários nas publicações, mas um não autenticado não tem autorização pra ver e comentar, caso isso fosse um requisito de negócio.

### Segurança

Podemos evitar que a segurança seja afetada com programação, manter os frameworks e libs sempre atualizados, porém, atualmente podemos ir além disso, delegar várias funcionalidades que não são core de uma aplicação para ferrementas que realizam uma pré-filtragem para que requisições indesejadas nem cheguem a borda dos microsserviços.

Ferramentas mais atuais como Gateway e um WAF podem contribuir com a segurança a ataques DDoS, tentativas de injeção de código, adicionando rate limit, IP filtering, configurando a rede e limitando o acesso dentro dos serviços.

Outra técnica que poderia ser empregada é a utilização de ferramentas de análise de código estático como o SonarQube e Checkmarx e a cada alteração de código essas ferrementas ajudariam com uma lista de restrições já revisadas por especialistas.


### Monitoramento e diagnóstico

Para monitoramento e diagnóstico serão adicionados algumas ferramentas: 

- OpenTelemetry / Jaeger: realiza o tracing completo da aplicação afins de identificar possíveis gargalos;
- Prometheus / Grafana: coleta as métricas (por exemplo de uma JVM) do status da aplicação como utilização de memória, CPU, etc. Com o Grafana puglado, podemos ter uma visualização em 360 graus dessas métricas;
- Stack do Elastic: organiza e indexa dos logs da aplicação, cria paineis e etc.;
- APM: Podemos adicionar um APM como o NewRelic para realizar o monitoramento dos servidores em tempo real e descobrir otimizações e gargalos na infraestrtura.
