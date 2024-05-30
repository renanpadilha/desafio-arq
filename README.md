# Desafio - Arquitetura de uma rede social

## Diagrama de arquitetura

O desenho abaixo visa contemplar como serão realizadas as comunicações entre diferentes serviços na rede social.

![desafio-nt (3)](https://github.com/renanpadilha/desafio-arq/assets/5349447/40b602fe-4f7c-410e-8aff-62c13f28e956)

As requisições dos diferentes clients, através de HTTP, passariam por um BFF (Backend For Frontend) que são responsáveis por agregar requisições e dados de acordo com o client. O BFF mobile pode buscar dados diferentes de uma aplicação web. Assim, podemos ter maior performance, já que o próprio design de diferentes telas podem exigir do que outra.

No caso de uma operação de publicação, as informações são enviadas para o serviço `post-service`, o qual grava as informações principais da publicação, como texto, dados do usuário e, caso hajam mídias, salva num bucket de forma assíncrona, retornando somente referências para serem armazenadas na base de dados. Logo, torna a requisição mais transparente para o usuário e, assim que pronta, a publicação ficaria disponível.

No caso de um comentário, a mesma coisa acontece, passando do BFF para o `comment-service` os dados da publicação e o conteúdo do comentário.

Esses dois tipos de operação podem armazenar o conteúdo resultante no Redis para manter o cache dessas informações. A estratégia de cache será definida conforme a necessidade do negócio, podendo ser os mais frequentes acessados, bem como os últimos publicados, por tempo (TTL), etc.

Qualquer um dos dois serviços (`post-service` ou `comment-service`), ao receber a requisição, processaria os dados e publicaria em um tópico a informação de que um novo post/comentário foi criado, o consumidor desse tópico estaria em um outro seviço especialista (`notification-service`) responsável por notificar os usuários de várias formas, de acordo com as configurações daquele usuário, por exemplo, selecionar somente notificação por e-mail ou por push.

A gestão de acesso e identidade ficaria a cargo do Keycloak, o qual conteria as políticas de acesso de cada usuário.

### Escalabilidade e Disponibilidade

A disponibilidade da aplicação se dá através de vários fatores:

- Trabalhar com uma arquitetura de microsserviços: podemos retirar o alto acoplamento entre os domínios da aplicação e o isolamento de possíveis indisponibilidades;
- Trabalhar com cache de dados: não apenas ajuda a garantir a alta disponibilidade do banco de dados, como também ao retornar dados diretamente do cache em vez de consultar repetidamente a base de dados;
- Trabalhar com o Kubernetes: podemos escalar as aplicações de maneira horizontal de acordo com a demanda de cada serviço;
- Trabalhar com banco de dados não relacional: a escalabilidade pode ser realizada de forma horizontal, e em alguns nós e réplicas separadas geograficamente.

Alguns pontos únicos de falhas e possíveis mitigadores são:

- SSO: Dado que o SSO é centralizado, qualquer falha pode resultar na inoperância da aplicação. Integrar o SSO em um ambiente gerenciado pelo Kubernetes pode ajudar a mitigar esse problema. Além disso, armazenar sessões e tokens no cache, apenas durante seu período de validade, também pode contribuir para minimizar tal risco;
- Microsserviços: São serviços independentes uns dos outros. Por exemplo, se os serviços de comentários e notificações falharem, o serviço de publicações pode permanecer operacional de forma independente;
- Sistema de cache: Caso o banco de dados fique indisponível por alguns instantes, a aplicação pode funcionar com o cache existente.

### Armazenamento de Dados

Sobre o armazenamento dos dados, ambos os tipos de bancos de dados, desde que bem otimizados, dariam conta. Entretanto, nesse tipo de aplicação a melhor escolha é um banco de dados não relacional, pois, segundo o teorema CAP, a base teria alta disponibilidade e tolerância à partição. Contudo, ainda de acordo com o teorema, a base pode ter uma baixa consistência, que não é o requisito mais importante para esse tipo de aplicação, mas sim a alta disponibilidade/escalabilidade. 
Sendo assim, tendo uma massiva alteração dos dados, não necessitar de uma consisência alta e não precisar realizar grandes operações com joins, um banco não relacional é a melhor opção.

### Processamento de Conteúdo

O processamento de conteúdo realizaria-se da seguinte forma: 

Os textos seriam armazenados na estrutura da Collection de Post no banco de dados, já as mídias como imagens e vídeos seriam pré-processados com algum tipo de compressor de arquivos e, após, enviados de maneira assíncrona para algum filesystem como o s3 da Amazon, mantendo no banco de dados somente as referências para esses arquivos.

### Notificações em tempo real

Para realizar as notificações em tempo real, nesse caso, a melhor solução seria realizá-las de forma assíncrona utilizando o Kafka ou o próprio Pub/Sub do Redis.

A cada publicação ou comentário produziríamos uma mensagem em um tópico que seria consumida através de um microsserviço de notificação, o qual realizaria a entrega daquela notificação.

Caso de uso:

Um usuário realizou um comentário em uma publicação, ao receber a requisição no serviço de comentários, uma mensagem seria publicada com os dados da publicação/comentário para que fosse lido no serviço de notificações e conectasse com os vendors de push ou e-mail, por exemplo.


### SSO

O SSO deve ser um sistema centralizado, como o Keycloak, que provê um sistema de gestão de identidade. Assim, seriam configurados os usuários, policies, roles e scopes para que os diferentes serviços garantam que os usuários tenham acesso a determinados recursos dentro da aplicação, delegando para a aplicação somente a decodificação do token e realizando a autorização dos recursos conforme configuração do usuário no SSO. 

Exemplo: 

Um usuário autenticado tem acesso a realizar comentários nas publicações, entretanto um não autenticado não tem autorização para ver e comentar, caso isso fosse um requisito de negócio.

### Segurança

Podemos preservar a integridade da segurança por meio da programação e da manutenção regular de frameworks e bibliotecas. No entanto, atualmente, podemos adotar uma abordagem mais abrangente, delegando diversas funcionalidades não essenciais de uma aplicação para ferramentas que realizam uma filtragem preliminar, evitando assim que requisições indesejadas alcancem a borda dos microsserviços.

Ferramentas mais atuais como um Gateway e um WAF podem contribuir com a segurança a ataques DDoS, tentativas de injeção de código e demais tipos de ataques. 

Uma abordagem adicional que poderia ser adotada é a incorporação de ferramentas de análise de código estático, como SonarQube e Checkmarx. Essas ferramentas auxiliariam na identificação de falhas de segurança já revisadas por especialistas a cada alteração de código.

### Monitoramento e diagnóstico

Para monitoramento e diagnóstico poderiam ser adicionadas algumas ferramentas: 

- OpenTelemetry/Jaeger: realiza o tracing completo da aplicação - mostrando a stack completa da requisição -, afim de identificar possíveis gargalos;
- Prometheus/Grafana: coleta as métricas (por exemplo de uma JVM) do status da aplicação como a utilização de memória, CPU, etc. Com o Grafana puglado, podemos ter uma visualização em 360 graus dessas métricas;
- Stack do Elastic: organiza e indexa os logs da aplicação, cria paineis, etc;
- APM: integra uma solução de monitoramento de desempenho de aplicações, como o NewRelic, para acompanhar os servidores em tempo real, identificar oportunidades de otimização e detectar possíveis gargalos na infraestrutura.
