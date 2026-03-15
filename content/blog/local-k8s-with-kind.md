+++
author = "Hebert Freitas"
title = "Simulando um cluster k8s localmente com Kind"
date = "2025-04-22"
description = "Simulando um cluster k8s localmente com Kind"
tags = [
    "k8s",
]
+++

Kubernetes é uma realidade bem difundida atualmente. Os principais cloud providers oferecem soluções built-in como o EKS na AWS ou o AKS na Azure.
Do ponto de vista do desenvolvedor tradicional, suas aplicações devem ser construídas e preparadas para executar em containers, e em algum momento (geralmente em uma pipeline de CI/CD) o container desta aplicação será implantado em um cluster k8s.

No entanto, seja por necessidade de conhecer melhor a infraestrutura do k8s (o que recomendo que todo desenvolvedor conheça profundamente se trabalha com aplicações dentro dele), ou seja porque você precisa simular ou testar algum recurso, pode ser necessário efetivamente fazer o deploy do container no ambiente com pods, deployments, services, e toda a estrutura necessária para rodar em um cluster kubernetes.
Além disso, nem sempre existe a possibilidade de você fazer o deploy de fato em um ambiente em cloud para simular uma situação específica, ou as vezes você só quer fazer um teste de algo rápido.

Já a alguns anos existem projetos que simulam um cluster k8s em sua máquina local, hoje vou apresentar um que atende muito bem os meus requisitos que é o [Kind](https://kind.sigs.k8s.io/)

## Kind

O kind é uma ferramenta que executa um cluster kubernetes na sua máquina com o uso de docker.
Basicamente ela cria um cluster k8s com os nodes e control plane sendo executados em containers docker.
Para executar o kind você precisa das seguintes ferramentas instaladas e funcionando:

 - docker
 - kubectl (para iteragir com o cluster)

### Instalação

Recomendo seguir as [instruções oficiais para instalação do kind](https://kind.sigs.k8s.io/#installation-and-usage).
Ele pode ser instalado como um pacote do golang.
Em um mac pode ser instalado via brew executando: 

```bash
brew install kind
```

### Criando clusters

Você pode criar um cluster k8s simplesmente executando:

```bash
kind create cluster
```
Por padrão, este comando criará um cluster com um único nós, mas se você quiser criar um cluster com múltiplos nós você pode usar um arquivo yaml como este:

```yaml
#kind-multinode-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```
e então executar o comando:
```bash
kind create cluster --config kind-multinode-cluster.yaml
```
Quando você cria um cluster no kind ele também já adiciona um contexto no seu arquivo de configuração do kubectl (geralmente em `~/.kube/config`) apontando para as configurações do seu cluster local, portanto logo depois da criação do cluster basta executar comandos do kubectl tradicionais, por exemplo:

```bash
kubectl cluster-info #exibe as informações do cluster
kubectl get nodes #exibe as informações dos nodes do cluster
```

### Load Balancers no Kind

Quando você precisa expor um determinado serviço no kubernetes é comum criar um Service do tipo Load Balancer.
Quando você está usando uma solução de kubernetes em um cloud provider como AWS ou Azure, automaticamente ao criar este Load Balancer um IP é fornecido e você consegue acessar o serviço pelo IP e porta.
Ocorre que no kind, você está rodando localmente em sua máquina, portanto é preciso que alguma outra ferramenta faça este papel, para isso existe o [Kubernetes Cloud Provider for KIND](https://github.com/kubernetes-sigs/cloud-provider-kind?tab=readme-ov-file#install)
Recomendo fortemente que você instale ele para que seja possível expor serviços em um IP e porta para teste.
Lembrando que após a instalação você precisa [executar o binário do provider](https://github.com/kubernetes-sigs/cloud-provider-kind?tab=readme-ov-file#running-the-provider) enquanto estiver provisionando serviços no cluster k8s.

### Fazendo o deploy de um serviço no cluster

A partir deste ponto, você já pode fazer o deploy de qualquer serviço no seu cluster k8s localmente, para exemplificar segue um arquivo yaml contendo deployment, especificação de pods e service do tipo Load Balancer para uma api de teste em python fast-api

O código desta api está disponível [neste projeto](https://github.com/hebertrfreitas/fastapi-example) e a mesma possui uma pipeline de CI/CD no github actions que faz o push de uma imagem docker para o docker hub.

```yaml
#simple-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pokemon-api
spec:
  selector:
    matchLabels:
      app: pokemon-api
  replicas: 2
  template:
    metadata:
      labels:
        app: pokemon-api
    spec:
      containers:
      - name: pokemon-api
        image: hebertrfreitas/fastapi-example:latest
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
---
kind: Service
apiVersion: v1
metadata:
  name: pokemon-api-deployment
spec:
  type: LoadBalancer
  selector:
    app: pokemon-api
  ports:
  - port: 80
    targetPort: 80
```

Para aplicar no cluster basta executar:

```bash
kubectl apply -f simple-api-deployment.yaml
```

Em seguida para chamar e testar o serviço use os seguintes comandos:

```bash
#obtem o IP do load balancer
POKEMON_API_IP=$(kubectl get svc pokemon-api-deployment -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

#faz uma chamada na rota /pokemon da aplicação, que chama uma API externa e faz um request para trazer os dados de um pokemon aleatório
curl http://$POKEMON_API_IP/pokemon
``` 

### Projeto no github

Se quiser testar todos os comandos e códigos que usei neste post, eles estão disponíveis neste repositorio no github:
https://github.com/hebertrfreitas/kind-labs