+++
author = "Hebert Freitas"
title = "Montando um ambiente local para testar LLM's OpenSource "
date = "2025-02-19"
description = "Montando um ambiente local para testar LLM's OpenSource "
tags = [
    "genai",
]
+++

> Estou iniciando uma série de artigos para compartilhar a minha experiência como engenheiro de software atuando focado com tecnologias de Generative AI, fique atento aos próximos posts para aprender um pouco mais sobre este tópico.

Ao pesquisar um pouco sobre o mundo de Large Language Models é natural que você se depare com os grandes fornecedores de modelos como OpenAI, Anthropic, etc. Estas empresas fornecem seus modelos proprietários no formato SaaS através de um serviço que pode ser acessado por uma API desde que você tenha uma conta e pague um valor pelo seu uso.

No entanto, nem só de modelos proprietários vive o universo de LLM's, muitos modelos são disponibilizados de forma open source, o que significa que você pode baixá-los e executá-los como quiser.

Apesar de parece simples, detalhes como arquitetura do modelo, configurações e requisitos de hardware podem tornar a tarefa significativamente árdua para um desenvolvedor que simplesmente quer testar diferentes modelos.

Para atender essa demanda, existem algumas iniciativas que tornam este trabalho mais simples, uma delas é o [ollama](https://github.com/ollama/ollama)

## Ollama

O ollama é um projeto open source que tem como objetivo facilitar a nossa experiência para obter e rodar LLM's.
Ele pode ser [instalado na sua máquina](https://ollama.com/download) e fornece um cli através do qual baixar e executar uma LLM é tão simples quanto executar este comando
```shell
ollama run llama3.2
```
Imediatamente após a execução desde comando, caso você ainda não tenha este modelo localmente ele irá fazer o download do mesmo e logo em seguida fornecer uma iteração com o modelo via linha comando.

Você também pode optar por executar o ollama dentro de um container docker, desde que você já tenha o docker instalado basta executar:
```shell
docker run -it -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```
Em seguida para executar o modelo:
```shell
docker exec -it ollama ollama run llama3.2
```
Nos exemplos acima, estamos executando o modelo `llama3.2`, para saber a lista de modelos disponíveis, consulte o [catálogo do ollama](https://ollama.com/library)

Esta solução já atende grande parte das necessidades, mas ela pode melhorar.
O serviço do ollama também disponibiliza uma API para comunicação com o modelo, e devido a isso podemos tanto fazer requisições http como também podemos projetar um interface gráfica para facilitar nosso dia a dia, e neste ponto entra a nossa próxima ferramenta que irá nos ajudar, a [OpenWebUI](https://github.com/open-webui/open-webui).

## OpenWebUI

O OpenWebUI é um projeto opensource que fornece uma interface padronizada de chat conversacional para testar LLM's, pode ser usada em conjunto com o ollama que assume o papel de servir as LLM's localmente.

Você pode instalar o OpenWebUI via `pip` com o comando (recomendamos que você crie um ambiente virtual antes de instalar
```shell
pip install open-webui
```
Em seguida rodar o servidor web com 
```
open-webui serve
```
Acessando o endereço http://localhost:8080 você verá uma tela parecida com esta.

O OpenWebUI também pode ser executado via docker e tem compatibilidade total com o ollama, caso queira consumir os modelos do ollama, execute o comando abaixo (considerando que você já tem um container docker de nome ollama sendo executado)

```shell
docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```
Neste caso, após a execução, basta esperar alguns segundos (o servidor do OpenWebUI pode demorar um pouco para subir) e acessar o endereço `http://localhost:3000`

Os modelos servidos pelo ollama estarão disponíveis e você poderá conversar com eles, semelhante ao exemplo à seguir:

![print da tela mostrando a interface web do serviço openwebui em um chat com o modelo llama 3.2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s7nemvg10btie6lfyfpl.png)

## Bônus, executando tudo com um comando

Para quem achar muito trabalhoso, eu disponibilizei um projeto no github por onde você pode subir todo este ambiente e os serviços com apenas um comando.
O projeto está disponível aqui: https://github.com/hebertrfreitas/generative-ai-labs/tree/main/examples/ollama