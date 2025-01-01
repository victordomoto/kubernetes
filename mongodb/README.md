# Mongodb no k8s

# Estrutura
Primeiramente iremos criar o deployment do mongodb, mas utilizando secrets para as credenciais.

# Secret
Nós iremos definir as credenciais dentro de mongo-secret.yaml.
Note que temos os dados sendo um modelo de key-value. Além disso, os dados estão na base 64.
Uma maneira bem fácil de transformar qualque texto nessa base é usando o próprio terminal do linux:

```
echo -n 'text' | base64
```

Depois disso, basta criar o secret no kubernetes.

```
kubectl apply -f mongo-secret.yaml
```

# Service
Vamos criar um service interno para a porta do mongodb, que é a 27017. Podemos fazer isso no próprio deployment, basta separar o deployment do service por `---`
```
kubectl apply -f mongo-deployment.yaml
```

Verifique se o service está de fato ok com o describe:
```
kubectl describe service mongodb-service -n development
```
Aqui teremos os endpoints.

# ConfigMap
Para armazenar valores que não sensíveis de forma centralizada, podemos utilizar o configMap.
Como por exemplo, se quisermos deixar centralizado qual será o server, ou o endpoint do mongodb.

