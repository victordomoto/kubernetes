## Criação do cluster:
* Minikube:
    ```
    minikube start
    ```
* Verificar os nós:
    ```
    kubectl get nodes
    ```

## Subindo uma aplicação no cluster kubernetes:
Vamos subir uma aplicação simples de quote, onde ela irá retornar para nós frases.
Podemos utilizar o manifesto YAML para fazer essa implementação.

### Criação dos namespaces
Vamos criar dois namespaces bastante utilizados, que é o development e o production, apenas para simulação.

Dentro de namespace.yaml os definimos e basta aplicá-lo.
``` 
kubectl apply -f namespace.yaml
```

### Criação do deployment
Vamos criar um deployment que irá utilizar a imagem datawire/quote:0.5.0. Além disso, vamos definir no namespace development com o nome do app de quote-service e nomear o container como quote-container. Podemos ainda definir duas replicas e aceitar as requisições na porta 8080. Portando, definindo esses pontos no quote.yaml, basta aplicar.

```
kubectl apply -f quote.service
```

### Criação do busybox
O busybox nos permite utilizar uma série de comandos básicos dentro do cluster. Então vamos criar um deployment para ele para poder testar a aplicação.
```
kubectl apply -f busybox.yaml
```

Agora basta entrar dentro do container e utilizar suas ferramentas, como por exemplo o comando `wget` para fazer uma requisição HTTP.
```
kubectl get pods -n development -o wide     # Verifique o IP_POD
kubectl get pods                            # Verifique o busybox-hash
kubectl exec -it busybox-hash -- /bin/sh    # Troque hash pelo nome
wget IP_POD:PORT                            # Troque IP_POD e PORT
```

### Service 
Até o momento só conseguimos acessar essa aplicação de dentro do cluster. Para deixar ela pública, precisamos criar um service do tipo loadbalancer, para ele direcionar o tráfego para esses pods da nossa aplicação de quote.
Lembre-se que no nosso deployment, definimos a label da aplicação como quote-service, então essa deve ser a label que o service irá direcionar. Além disso, definimos a porta 8080. Então podemos colocar o acesso público na porta 8000 e direcionar para a porta 8080.

```
kubectl apply -f service.yaml
minikube tunnel
```

Agora basta acessar a aplicação fora de cluster, ou seja, da sua máquina local. O `minikube tunnel` permite a utilização dessas portas.
```
http://127.0.0.1:8000
```


### Debugando os pods
Temos 3 principais maneiras de descobrir o que há de errado com um pod:
* Verificando os logs do console:
```
kubectl log [pod_name]
```

* Se conectando de forma interativa no pod:
```
kubectl exec -it [pod_name] -- /bin/sh  
```

* Verificando informações sobre o pod:
```
kubectl descibre pod [pod_name]
```

### Verificando status dos componentes do k8s
Lembrando que se não for o default, precisamos informar o namespace com -n
```
kubectl get nodes
kubectl get pods
kubectl get services
kubectl get replicasets
kubectl get deployments
```

### Trabalho com os componentes do k8s sem ser por arquivos
É possível criar diretamente pelo kubectl algum deployment e service, ou até mesmo editá-lo ou apagá-lo:
```
kubectl create deployment nginx-develop-deployment --image=nginx
kubectl create service loadbalancer nginx-service --tcp=9000:80
kubectl edit service nginx-service
kubectl edit deployment nginx-develop-deplyment
kubectl delete service nginx-service
kubectl delete deployment nginx-develop-deployment
```

### Exportar um componente para um arquivo yaml
```
kubectl get deployment pod-info-deployment -n development -o yaml > pod-info-deployment.yaml
```
