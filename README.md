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


