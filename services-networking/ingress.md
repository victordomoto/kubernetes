# Ingress
An Ingress is a Kubernetes resource that manages external HTTP/HTTPS access to applications running inside a cluster.

Instead of exposing each application with its own LoadBalancer or NodePort service, multiple applications can be exposed through a single entry point.

## Common use cases:

- Host-based routing (app.example.com, api.example.com)
- Path-based routing (/app, /api)
- TLS/HTTPS termination
- Centralized traffic management

## How traffic flows
```
Client
  |
  v
Load Balancer
  |
  v
Ingress Controller
  |
  v
Ingress Rules
  |
  v
Service
  |
  v
Pods
```

## Ingress resource
The Ingress itself is only a Kubernetes object that defines routing rules.
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```
The Ingress resource does not process traffic by itself. It requires an Ingress Controller.

## IngressController
An Ingress Controller is the component responsible for implementing the routing rules defined by Ingress resources.

Popular controllers:

- NGINX Ingress Controller
- Traefik
- HAProxy Ingress
- AWS Load Balancer Controller

Without an Ingress Controller, Ingress resources have no effect.

## IngressClass
An IngressClass defines which Ingress Controller should manage a specific Ingress.
```
spec:
  ingressClassName: nginx
```
This is especially useful when multiple controllers exist in the same cluster.

## Key concepts
| Component          | Purpose                                          |
| ------------------ | ------------------------------------------------ |
| Ingress            | Defines HTTP/HTTPS routing rules                 |
| Ingress Controller | Implements and enforces Ingress rules            |
| IngressClass       | Associates an Ingress with a specific controller |
| Service            | Provides a stable endpoint for Pods              |
| Load Balancer      | Exposes the Ingress Controller externally        |
| TLS                | Enables secure HTTPS communication               |


## Ingress nginx install
Using helm, it's pretty easy to install.
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort
```

## Ingress haproxy install
```
helm repo add haproxytech https://haproxytech.github.io/helm-charts
helm repo update

helm install haproxy-ingress haproxytech/kubernetes-ingress \
  --namespace haproxy-ingress \
  --create-namespace \
  --set controller.service.type=NodePort
```


## Cert-manager install
```
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

## Creating ingress TLS
```
# clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
```

## Creating cert with commonName
```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: hello-php-tls
  namespace: dev-hello-php
spec:
  secretName: hello-php-tls
  commonName: hello-php.homelab.local
  dnsNames:
  - hello-php.homelab.local
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
```
