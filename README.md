# Kubernetes
A ideia inicial é resumir de forma prática a criação de clusters no K8s e quais conceitos/tecnolgias são importantes para que isso seja feito.

## O que é o Kubernetes?
De forma resumida podemos dizer que é um orquestrador de containers.

## Arquitetura do k8s
Iremos seguir com um modelo control plane/workers para a criação dos clusters, de modo que o ideal seria ter pelo menos 3 redundâncias, de modo que o control plane seria responsável pelo gerenciamento do próprio cluster enquanto os workers seriam para executar as aplicações que desejamos.

### Como rodar o k8s no ambiente local?
* Kind: consiste basicamente em simular o cluster do kubernetes dentro do docker, geralmente o utilizamos apenas para desenvolvimento de testes, e não de produção.

* Minikube: também é uma ferramenta para testes, de modo que a criação do cluster é feita com apenas 1 único nó. 

### Outros conceitos de arquitetura 
* API server: é um dos principais componentes do Kubernetes, onde é fornecido uma API que utiliza o JSON sobre o HTTP como comunicação, de modo que na sua grande maioria das vezes funciona juntamente com o kubectl, no que diz respeito à administração. Esse componente permite a comunicação entre os nós do cluster.

* etcd: é utilizado pelo k8s para o armazenamento de dados, tais como especificações, status, configurações do cluter e dentre outras opções. Ele nada mais é do que um datastore chave-valor, e por segurança ele é executado apenas em nós do tipo control plane, ou se preferir em nós específicos para o etcd.

* Scheduler: é responsável por selecionar o nó que irá hospedar um determinado pod, de modo que a seleção é feita baseando-se na quantidade de recursos disponíveis por nó, além também do estado de cada nó do cluster, garantindo assim que os recursos sejam distribuídos corretamente.

* Controller Manager: é quem garante que o cluster esteja no último estado definido pelo etcd. Então a ideia dele é fazer verificações se o estado atual do cluster corresponde com o deploy que foi feito.

* kubelet: nada mais é do que um agente do kubernetes que é executado nos nós workers, de modo que em cada um deles deverá existir um agente kubelet em execução. O kubelete é responsável por gerenciar de fato os pods que foram direcionados pelo controller do cluster, de forma que para isso o kubelet pode iniciar, para e manter os containers e os pods em funcionamento de acordo com o instruído pelo controlador do cluster.

* kube-proxy: age como proxy e um loadbalancer. Este componente é responsável por efetuar o roteamento de requisições para os pods corretos, como também cuidar da parte de rede do nó.


## Instalação
* Instalar o docker:
    ```
    sudo apt-get update -y
    sudo apt-get install docker -y
    ```
* Instalar o kubectl:
    ```
    sudo apt-get install kubectl -y
    ```
* Instalar o minikube:
    ```
    sudo apt-get install minikube -y
    ```


## Criação do cluster local com o Minikube
```
minikube start      # inicia o cluster
kubectl get nodes   # verifica os nós
minibuke stop       # para o cluster
minikube delete     #deleta o cluster
```
### Kubectl
Permite a interação com o cluster.
```
kubectl cluster-info           # infos do cluster
kubectl get nodes              # retorna nós      
kubectl get namespaces         # ou apenas ns
kubectl get pods -A            # pods de todos ns
kubectl get services -A        # services de todos ns
kubectl get deployments -n development 
```

### Namespaces
São usados para isolar e gerenciar aplicações e serviços que você deseja que permaneça em segundo lugar.

### Services
Services são como loadbalancers de dentro do cluster e direcionam o tráfego para os pods.

### Manifestos YAML
Utilizamos esses manifestos para fazer implementações ou alterações no cluster, como por exemplo fazer o deploy de uma aplicação, de um service, criação de namespace e dentre outros.
Todos os componentes que vamos trabalhar podemos implementar usando esses arquivos YAML.
Eles têm uma estrutura padronizada, podendo ser dividia em 3 grandes partes:
Nas duas primeiras linhas apenas definimos o que queremos criar, se é um deployment, service, namespace etc.
```
---
apiVersion: apps/v1
kind: Deployment
```

```
apiVersion: v1
kind: Service
```

1. **metadata:** onde podemos definir o nome, label, namespace e etc.
2. **spec:** especificação que varia de acordo com o tipo de componente, se é um deployment, um service ou namespace. Nele podemos definir por exemplo quantidade de réplicas, imagem a ser utilizada, limitações de recursos de hardware de etc.
3. **status:** é automaticamente gerado e adicionado pelo próprio kubernetes. Basicamente o k8s fica constantemente verificando se há alguma diferença entre as especificações atuais e as desejadas, que foram definidas no manifesto, como por exemplo o número de réplicas. Caso esteja diferente, então ele sabe que tem algo para ser corrigido.

```
kubectl apply -f [name_file]
kubectl apply -f namespace.yaml
kubectl delete -f [name_file]
```

## Log e Debbug
Verifique a integridade de um pod observando os logs de eventos. Ao subir novos pods, é muito capaz dele dar algum erro, como por exemplo de falta de espaço, imagem indisponível e etc.
* Identificar o nome do pod:
    ```
    kubectl get pods -n development
    ```
* Usar o kubectl describe:
    ```
    kubectl describe pod pod-info-deveploment-hash -n development
    ```

## Busybox
Podemos também utilizar o busybox para fazer testes locais e práticos, como uma requisição HTTP numa aplicação que fizemos o deploy.
* Descubra o IP do pod:
    ```
    kubectl get pods -n development -o wide
    ```
* Entre dentro do busybox:
    ```
    kubectl exec -it busybox-hash -- /bin/sh
    ```
* Utilize dos comandos gerais, como wget:
    ```
    wget IP-POD:PORT
    ```


