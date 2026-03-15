+++
author = "Hebert Freitas"
title = "DynamoDB local - Um mock funcional para o DynamoDB"
date = "2022-02-13"
description = "DynamoDB local - Um mock funcional para o DynamoDB"
tags = [
    "dynamodb",
]
+++

Recentemente escrevi um [post](https://dev.to/hebertrfreitas/localstack-um-mock-funcional-para-os-servicos-da-aws-1jkd) sobre o localstack, que fornece um mock funcional para os principais serviços da aws.

Porém, caso você esteja trabalhando especificamente com o [DynamoDB](https://aws.amazon.com/pt/dynamodb/) a aws fornece oficialmente uma ferramenta chamada [DynamoDB local](https://docs.aws.amazon.com/pt_br/amazondynamodb/latest/developerguide/DynamoDBLocal.html) que é recomendada para testes e desenvolvimento da aplicação sem se conectar aos serviços reais da aws. 
Neste artigo vamos entender um pouco como funciona esta ferramenta 😀

## Índice
* [Como Usar](#como-usar)
* [Usando via Docker](#usando-via-docker)
* [Aplicação de exemplo] (#aplicacao-exemplo)


### Como usar <a name="como-usar"></a>

O DynamoDB local convenientemente é fornecido de três formas:

- Como um programa java executável (arquivo .jar)
- Através de uma imagem docker
- Através de uma dependência do maven

A página de [documentação](https://docs.aws.amazon.com/pt_br/amazondynamodb/latest/developerguide/DynamoDBLocal.DownloadingAndRunning.html) dá detalhes sobre o uso de cada uma destas opções. Neste post utilizaremos a opção via imagem docker.

### Utilizando o DynamoDB local via docker <a name="usando-via-docker"></a>

Se você apenas executar o comando à seguir no seu terminal(assumindo que você tem o docker instalado corretamente) já terá o dynamodb local rodando na porta 8000

```
docker run -p 8000:8000 amazon/dynamodb-local
```

E sim, você já consegue apontar a sua aplicação para ele 😁, vamos demonstrar isso no no tópico em que construímos uma [aplicação de exemplo] (#aplicacao-exemplo).

Por padrão o entrypoint executado pela imagem docker oficial irá utilizar a propriedade `-inMemory` que significa que o dynamo irá manter todos os dados em memória, logo ao finalizar o container automaticamente os dados são perdidos.

Eventualmente vamos querer manter os arquivos e dados do banco de dados criados via container docker mesmo após a finalização do mesmo, para isso segue o conteúdo de um arquivo docker compose que explora um pouco mais as opções disponíveis.

```yaml
version: '3.8'

services:
  dynamodb-local:
    working_dir: /home/dynamodblocal
    command: "-jar DynamoDBLocal.jar -sharedDb -dbPath ./data"
    image: "amazon/dynamodb-local:latest"
    container_name: dynamodb-local
    ports:
      - "8000:8000"
    volumes:
      - ".dynamodbdata:/home/dynamodblocal/data"
```
> Para utilizar estas instruções, salve o conteúdo em um arquivo chamado `docker-compose.yaml` e em seguida execute o comando `docker-compose up`. 

Algumas observações sobre a configuração realizar no docker compose:

1. sobrescrevemos o comando inicial executado ao subir o container na linha `command:`, observe que não colocamos o comando `java` porque ele já é especificado no entrypoint default da imagem docker fornecida pela aws. 
2. utilizamos o parâmetro `-sharedDb` para informar ao dynamo que desejamos criar um único arquivo para armazenar os bancos de dados.
3. o parâmetros `-dbPath ./data` informa onde queremos que o arquivo de banco de dados seja salvo. Como o workdir foi definido para `/home/dynamodblocal` o banco de dados será salvo no caminho absoluto `/home/dynamodblocal/data` dentro do container.
4. Criamos um volume apontando o caminho `.dynamodbdata` da máquina host para `/home/dynamodblocal/data` dentro do container, logo todos os arquivos de banco de dados gerados pelo dynamo serão mapeados para lá.

Ao executar o comando `docker-compose up` na pasta com o arquivo de conteúdo exemplificado acima podemos observar que será criada a pasta `.dynamodbdata` a princípio vazia.

Caso você tenha o [aws-cli](https://aws.amazon.com/pt/cli/) instalado na sua máquina, é perfeitamente possível executar comandos apontando para o dynamodb local, basta inserir o parâmetro `--endpoint-url=http://localhost:8000`.

Por exemplo, para listar as tabelas existentes:
```sh
aws --endpoint-url http://localhost:8000 dynamodb list-tables
```

### Aplicação de exemplo <a name="aplicacao-exemplo"></a>

Para demonstrar o uso do dynamodb local construi uma pequena api usando spring boot e kotlin, o código está disponível no [github](https://github.com/hebertrfreitas/dynamodb-example).

Nesta aplicação estamos usando a biblioteca [dynamodb-enhanced](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/dynamodb-enhanced-client.html) que faz parte do AWS sdk for java (v2) e fornece uma maneira bem intuitiva de mapear as tabelas dynamo na sua aplicação via anotações. 

Para que a aplicação possa se comunicar com o dynamo é necessário criar um objeto do tipo `software.amazon.awssdk.services.dynamodb.DynamoDbClient` na aplicação fizemos isso provisionando um `bean` do spring da seguinte maneira:

```kotlin
    @Bean
    fun dynamoDBClient():DynamoDbClient{
        return DynamoDbClient.builder()
            .region(awsProperties.regionAsAwsEnum())
            .apply { 
                if(! awsProperties.endpoint.isNullOrEmpty()) 
                    endpointOverride(URI(awsProperties.endpoint)) //awsProperties.endpoint = http://localhost:8000 
            }
            .credentialsProvider(DefaultCredentialsProvider.builder().build())
            .build()
    }
```

Observe que chamamos no builder o método `endpointOverride` apenas se `awsProperties.endpoint` for diferente de nulo.
A ideia aqui é passar nessa propriedade o valor `http://localhost:8000` exatamente como fizemos no aws-cli, apontando para onde está rodando o DynamoDB local na sua máquina.
Se você não passar essa propriedade o builder não executara este ponto e apontará para o serviço real da aws.

Nesta api estamos fazendo um simples crud de um cenário hipotético onde queremos cadastrar pintores famosos e suas pinturas.

A estrutura básica pensada para o modelo de dados pode ser representada da seguinte maneira em json
```json
{
    "id": "123",
    "name": "Leonardo da Vinci",
    "birthdate": "1452-04-15",
    "paintings":[
        {
            "name": "Mona Lisa (La Gioconda)",
            "location": "Louvre, Paris"
        },
        {
            "name": "The Last Supper",
            "location": "Santa Maria delle Grazie, Milan"
        }
    ]
}
```

Esta estrutura pode ser facilmente mapeada para classes através da anotações, segue exemplo de como fizemos isso no código

```kotlin
/**
 * Important note: using `var` because the AWS SDK needs default `setters` in order to detect an attribute
 */
@DynamoDbBean
class Painter(
    @get:DynamoDbPartitionKey var id: String? = null,
    var name: String? = null,
    @field:JsonFormat(pattern="yyyy-MM-dd")
    var birthdate: LocalDate? = null,
    var paintings:List<Paint> = mutableListOf()
){
    constructor():this(null, null, null)
}



@DynamoDbBean
class Paint(var name:String?, var location:String?){
    constructor():this(null, null)
}
```
> OBS: Os contrutores default aceitando nulo e o uso de var são devido a requisitos do AWS SDK que necessita de classes com getters e setters padrão para que seja possível mapear as mesmas para o dynamo

Para executar o projeto baixe o mesmo do github, em seguida na raiz execute 
```shell
docker-compose up --build
```
Isso irá subir um docker-compose
contendo o dynamodb-local e também um container com o aws-cli que fará a criação da tabela inicial.

Em seguida execute 

```shell
./gradlew bootRun
```
Este segundo comando irá subir a aplicação na porta 8080.

> OBS: O java 11 instalado é requisito para que você consiga executar o projeto.

A api está documentada com swagger então é possível acessar o mesmo no endereço `http://localhost:8080/swagger-ui.html`

![Imagem do swagger-ui acessível no navegador](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vkwf2mhhkc93vml7yapx.png)

Para testar, seguem alguns comandos que podem ser executados via `curl`


**Criando um pintor**
```shell
curl --location --request POST 'localhost:8080/painters' \
--header 'Content-Type: application/json' \
--data-raw '{
    "id": "123",
    "name": "Leonardo da Vinci",
    "birthdate": "1452-04-15",
    "paintings":[
        {
            "name": "Mona Lisa (La Gioconda)",
            "location": "Louvre, Paris"
        },
        {
            "name": "The Last Supper",
            "location": "Santa Maria delle Grazie, Milan"
        }
    ]
}'
```

**Bucando um pintor pelo id**
```shell
curl --location --request GET 'localhost:8080/painters/123'
```

**Deletando um pintor**
```shell
curl --location --request DELETE 'localhost:8080/painters/123'
```

