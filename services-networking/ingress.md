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


