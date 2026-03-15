+++
author = "Hebert Freitas"
title = "Murmurhash - criando um rollout progressivo via backend "
date = "2023-04-21"
description = "Murmurhash - criando um rollout progressivo via backend "
tags = [
    "murmurhash",
]
+++

Ao ler o título deste post o desenvolvedor mais experiente provavelmente vai se perguntar: "Mas já não existem várias formas de se liberar uma nova funcionalidade para usuários progressivamente ?"

E a resposta é sim, de fato existem algumas maneiras.
Por exemplo, se você estiver em um ambiente com k8s e istio pode por exemplo usar um [Virtual Service para isso](https://istio.io/latest/docs/reference/config/networking/virtual-service/), você ainda pode fazer isso fora mesmo do k8s com um simples proxy reverso na frente de instâncias diferentes da sua aplicação (uma com a nova feature, e outra sem) além de várias outras.

Porem aqui neste artigo, vou lhe apresentar uma forma de atingir o mesmo objetivo usando um algoritmo de hash bem conhecido, o [Murmurhash](https://en.wikipedia.org/wiki/MurmurHash) que pode ser bem útil dependendo do seu cenário.

###Murmurhash

O murmurhash é um algoritmo de hash não criptográfico.

Ser um algoritmo de hash significa que dado um input ele converte isso para um output encriptado. 
Quer um exemplo simples no nosso dia a dia ?
Se você estiver em um linux ou mac pode usar o sha256 da seguinte maneira:
```sh
echo -n teste de mensagem sha256 | shasum -a 256
8785c54a5c506d0c4f031a76b7170c35b2bde862a2bbd7ab2d0485570b75bc06
```
Já a parte do não criptográfico se refere ao fato de muitas funções hash terem como principal característica tornar impossível a reversão da saída de volta no texto original, o que é especialmente útil se você estiver trabalhando com segurança onde isto é uma premissa. Este não é o caso do murmurhash que não se preocupa tanto com esta característica em detrimento de outras.

Ao contrário da família de algoritmos criptográficos, o murmurhash foi criado para ser rápido devido a sua otimização, com uma boa resistência a colisões e outras características interessantes.
O algoritmo Murmurhash foi criado por Austin Appleby e a implementação de referência em C++ pode ser encontrada no [github](https://github.com/aappleby/smhasher/blob/master/src/MurmurHash3.cpp).

Para demonstrar um pouco do que o algoritmo pode fazer, vamos olhar um simples código em java:

```java
    public static void main(String[] args) {
        String input = "hebert freitas";
        byte[] inputBytes = input.getBytes(Charset.forName("UTF8"));
        HashCode hashCode = Hashing.murmur3_128().hashBytes(inputBytes);
        byte[] outputBytes = hashCode.asBytes();
    }
```
Neste simples código estamos aplicando o murmurhash e fazendo o hash de um array de bytes que foi gerado a partir de uma simples string com o meu nome.
Optei por usar uma implementação do [murmurhash disponível dentro da lib guava](https://github.com/google/guava/blob/v31.1/guava/src/com/google/common/hash/Murmur3_128HashFunction.java) mas nada lhe impede de usar outra implementação.

O algoritmo do murmurhash é também determinístico, de maneira que para uma mesma entrada sempre haverá uma mesma saída.

Neste exemplo, estamos optando por pegar o resultado do hash também como um array de bytes, mas seria possível também pegar o resultado como um inteiro ou long.

Todas estas características fazem com que o murmurhash seja usado em cenários interessantes, vamos ver a seguir um possível cenário.

### Aplicabilidade

Você sabia que quando usa a lib para interagir com o [kafka](https://pt.wikipedia.org/wiki/Apache_Kafka) na sua aplicação o murmurhash é usado por debaixo dos panos para definir em qual partição de um tópico a mensagem será postada com base na sua key ? 
A referencia pode ser vista [aqui](https://github.com/apache/kafka/blob/6cf1010a6bf3bd6a9e0495c4787eb5b3aa01d5e8/clients/src/main/java/org/apache/kafka/clients/producer/internals/BuiltInPartitioner.java#L328) e basicamente o que é feito é usar o murmurhash aplicado a key da mensagem e busca o resto da divisão sobre o total de partições.
Lembram-se também do fato de que o kafka consegue garantir a ordenão de mensagem apenas dentro de uma partição de um tópico mas não dentro de todas as partições ?
Eis outro benefício do murmurhash neste cenário, como sempre teremos o mesmo valor de saída para a mesma entrada mensagens com a mesma key sempre serão direcionadas para a mesma partição.
O murmurhash também é muito eficaz no chamado [efeito avalanche](https://en.wikipedia.org/wiki/Avalanche_effect) que nada mais é do que uma característica onde caso modifiquemos minimamente o input teremos um resultado totalmente diferente,  e devido a isso, também é eficaz em distribuir uniformemente entre as partições de um tópico mensagens com keys distintas.

A seguir vamos ver como algumas destas características podem ser aplicadas em um cenário real.

### Criando um rollout progressivo com o murmurhash

Imagine que você está diante do seguinte cenário: 
"Uma determinada aplicação possui uma base de usuários relativamente grande e você deseja liberar uma nova funcionalidade que foi implementada de forma progressiva para estes usuários, primeiro para 10% da base, depois 20%, e assim sucessivamente até atingir todos os usuários"

Uma das melhores formas de fazer isso é implementar uma feature toogle que permite ligar ou desligar a nova feature na sua aplicação, mas neste caso, não pode ser uma toogle estática que representa o mesmo valor para todos os usuário da aplicação, ela teria que identificar se um determinado cliente entraria ou não nela naquele determinado momento baseado em alguma lógica.
Uma outra alternativa que você pode pensar é marcar os seus usuários na base de dados dizendo quem entra e quem não, atualizando isso aos poucos conforme você vai expandindo o fluxo, mas esta solução não parece ser muito eficiente e de difícil manutenção.

Para atender essa demanda podemos aplicar o murmurhash, usando como input alguma coisa que sempre seja única e nunca mude para um usuário ( id, cpf, etc)
Vamos ver um simples exemplo:
```java
private static double calculatePercentage(String userId){
        HashCode hashCode = Hashing.murmur3_32_fixed().hashBytes(userId.getBytes(Charset.forName("UTF8")));
        long hashedValueInLong = Integer.toUnsignedLong(hashCode.asInt());
        var percentual = hashedValueInLong  / Math.pow(2,32);
        System.out.println("Percentual: " + percentual);
        return percentual;
    }
```
 
**Detalhando cada um dos passos:**
1. Aplicamos o murmurhash a uma string de input da mesma maneira que fizemos ateriormente.
2. Pegamos o resultado do murmurhash como um inteiro e convertermos para um unsigned long (vamos aprofundar este detalhe a seguir)
3. Dividimos o numero resultado do murmurhash por 2 elevado a 32 porque este é o valor máximo que pode ser atingido.
4. O resultado é um percentual, um valor em 0.1 e 1.0 aplicado a um usuário.

Neste exemplo, estamos usando a versão de 32 bits do murmurhash, que pode gerar um hash de até 32 bits.
Na linha 3 do código convertermos um int para um long usando porque valores inteiros podem ser negativos e esta é uma forma simples de se obter o valor sem sinal, para mais detalhes veja a [documentação do método `Integer.toUnsignedLong`](https://docs.oracle.com/javase/8/docs/api/java/lang/Integer.html#toUnsignedLong-int-)
Se você executar este método com o input `836d473d-56be-4ef9-9b13-1fc8f36ef98e` terá como resultado o valor `0.913...`. 
Com este valor basta comparar com o percentual máximo que você deseja disponibilizar a nova feature. Neste exemplo, se estivermos liberando para 20% da base com certeza este usuário não entraria, mas se já estivermos liberando para 95% da base este usuário entraria porque seu percentual é 91%.

### É possível ter exatidão máxima com este método ?

Considerando as características do algoritmo murmurhash ele funciona melhor conforme o publico alvo vai aumentando, ou neste caso, conforme maior é sua base de usuários.

### Projeto de exemplo

Para exemplificar tudo o que discutimos criei um projeto de exemplo, onde a partir de uma lista fictícia de 10.000 usuários e um valor predeterminado de 10% classificamos esta lista para demonstrar que o algoritmo consegue distribuir uniformemente os hashs e filtrar um valor muito próximo de 10% de usuários mesmo sem conhecer toda a base.

O projeto está disponível neste link = https://github.com/hebertrfreitas/murmurhash-example



