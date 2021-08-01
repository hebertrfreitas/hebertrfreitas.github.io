---
title: "LocalStack"
date: 2021-08-01T00:30:00+00:00
# weight: 1
# aliases: ["/first"]
tags: ["localstack"]
author: "Hebert"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Um mock funcional para os serviços da aws"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---




Atualmente é bem comum trabalhar em projetos que se comuniquem com alguns serviços da aws, seja uma fila SQS, um bucket S3 para armazenar arquivos ou até mesmo um banco de dados DynamoDB.

Durante o desenvolvimento deste software você irá precisar fazer a comunicação com estes serviços em nuvem da aws para testar o seu produto, simular cenários ou até mesmo executar alguns testes de integração.

No entanto, nem sempre você terá disponível um ambiente na aws onde você possa alterar livremente de maneira que atenda os requisitos do desenvolvimento, ou mesmo que tenha, pode ser complexo fazer a comunicação em um primeiro momento. 

Para resolver estes problemas, existe o projeto [localstack](https://github.com/localstack/localstack), que fornece um mock dos principais servicos da AWS como SQS, SNS, S3 entre outros.

  
### Como usar o localstack

A forma mais conveniente de executar o projeto é através de uma imagem docker oficial disponível neste link:  [dockerhub localstack](https://hub.docker.com/r/localstack/localstack), segue um exemplo de um arquivo do docker-compose:

```yaml
version: "3.8"

services:
  localstack:
    container_name: "localstack"
    image: localstack/localstack:0.12.11
    ports:
      - "4566:4566"
      - "4571:4571"
    environment:
      - SERVICES=sqs,s3

```

Para executar este arquivo crie um arquivo yaml chamado docker-compose.yaml com o conteúdo acima e execute o comando `docker-compose up` na mesma pasta do arquivo criado. 

Alguns pontos importantes a se destacar:

 - neste exemplo estamos usando a versão 0.12.11 mas você pode e deve checar qual versão mais atualizada está disponível no docker hub do projeto.
 - a variável de ambiente `SERVICES` aceita uma lista de nomes dos serviços que desejamos provisionar, neste exemplo estamos pedindo ao localstack para disponibilizar uma api do SQS e uma do S3.


### Interagindo com os serviços da aws no localstack

Uma das formas oficiais de se comunicar com os serviços da aws é o [AWS CLI](https://hub.docker.com/r/localstack/localstack)

Você pode ter o awscli instalado na sua máquina e apontar para os serviços da aws mockados no localstack.
Para fazer isso basta inserir em todos os comando o seguinte parâmetro: `--endpoint-url=http://localstack:4566`

Exemplo de criação de uma queue no SQS: 

```sh
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name teste
```

No entanto, caso você não queria ou não precise ter o `awscli` instalado no seu computador, o localstack já disponibiliza um wrapper do `awscli` chamado `awslocal` que já vem pré-configurado para interagir localmente com os serviços mockados da aws.

Considerando que você esteja executando o projeto via docker é possível fazer o mesmo exemplo citado acima da seguinte maneira:

```sh
#para entrar no container (considerando que o nome do seu container é localstack)
docker exec -it localstack sh
#comando equivalente ao apresentado acima para criar uma fila 
awslocal sqs create-queue --queue-name teste 
```

Observe que não precisamos adicionar o parâmetro `--endpoint-url` pois o `awslocal` já está configurado assumindo que desejo me comunicar com os serviços localmente.


> **⚠ Porque localhost:4566 ?**  
> O projeto localstack disponibiliza o mock dos serviços da aws na porta 4566.
>
> Considerando que você está executando o projeto via docker é importante que você mapeie a porta 4566 do container para a 4566(ou outra a seu critério) da sua máquina local, foi isso que fizemos no arquivo de exemplo do docker-compose.


### Como executar comandos no startup do container

Caso você queira executar comandos após o startup do container como criar um bucket no s3 ou inicializar uma fila sqs basta criar um arquivo .sh com a lista de comandos e montá-lo como volume na pasta `/docker-entrypoint-initaws.d` dentro do container.

Ao iniciar, o localstack irá executar todos os scripts existentes neste caminho.
Sendo assim nosso docker-compose ficaria com a seguinte configuração:

```yaml
services:
    localstack:
      container_name: "localstack"
      image: localstack/localstack:0.12.11
      ports:
        - "4566:4566"
        - "4571:4571"
      environment:
        - SERVICES=sqs,s3
      volumes: 
        - ./init-scripts:/docker-entrypoint-initaws.d #mapeamento do volume
```

Agora podemos criar a pasta `init-scripts` na raiz do projeto e um arquivo shell script `startup.sh` dentro dela com alguns comandos por exemplo:

```sh
echo 'creating test bucket'
awslocal s3api create-bucket --bucket test-bucket

echo 'listing buckets'
awslocal s3api list-buckets
```
A estrutura do projeto deverá ficar da seguinte maneira:

```txt
 .
├── docker-compose.yaml
└── init-scripts
    └── startup.sh
```
Se executar o comando `docker-compose up` após o startup inicial você verá as mensagens do script sendo executado.

![Screen Shot 2021-07-31 at 17.53.36.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1627764858619/9HfzKFprK.png)


O conteúdo deste post está disponível no github no endereço: https://github.com/hebertrfreitas/localstack-docker
