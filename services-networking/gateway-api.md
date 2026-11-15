# Kubernetes Gateway API

The **Gateway API** is a family of Kubernetes resources for configuring
network traffic. For HTTP ingress, the main resources are `GatewayClass`,
`Gateway`, and `HTTPRoute`. A Gateway API implementation (controller/data
plane) must be installed to make these resources actually handle traffic.

For the CKA, focus on recognizing the resource relationships, creating or
editing an `HTTPRoute`, attaching it to a listener, directing traffic to a
Service, and checking status when routing does not work. Use the API version
and implementation already installed in the cluster or named in the task.

## Gateway API vs. Ingress

| Ingress | Gateway API |
|---|---|
| `Ingress` describes HTTP(S) host/path rules. | A set of resources models infrastructure and traffic routes separately. |
| Needs an Ingress controller. | Needs a Gateway API implementation/controller. |
| Most implementations use annotations for implementation-specific behavior. | Provides typed resources, roles, attachment controls, and route types. |
| One resource combines entry-point configuration and routing rules. | `GatewayClass` selects an implementation, `Gateway` configures listeners, and route resources define traffic rules. |

Neither API routes traffic on its own. Installing the API CRDs only makes the
resource types available to the API server; a compatible controller and
working data plane are needed to program listeners and forward requests.

## Main resources and responsibilities

| Resource | Scope | Purpose |
|---|---|---|
| `GatewayClass` | Cluster | Identifies a Gateway controller implementation. |
| `Gateway` | Namespaced | Requests/configures network listeners, such as HTTP or HTTPS. |
| `HTTPRoute` | Namespaced | Defines HTTP host, path, header, and backend routing rules. |
| `ReferenceGrant` | Namespaced | Allows selected cross-namespace references to resources in its namespace. |

The common relationship is:

```text
GatewayClass
    └── Gateway
          └── HTTPRoute(s)
                 └── Service backend(s)
```

In more detail:

1. A controller watches a `GatewayClass` whose `spec.controllerName` it owns.
2. A `Gateway` selects that class with `spec.gatewayClassName` and declares
   one or more listeners.
3. Routes attach to a Gateway listener, subject to the listener's
   `allowedRoutes` policy.
4. Route rules match requests and refer to backend Services.
5. The controller programs the implementation to send matching traffic to
   those backends.

`GatewayClass` is cluster-scoped; Gateways, HTTPRoutes, and ReferenceGrants
are namespaced. A Gateway's `status` and each attached route's `status` report
whether the controller accepted and programmed the configuration.

## Minimal HTTP routing example

This example assumes a compatible controller is already installed and the
`GatewayClass` named `example-gateway-class` exists. Class names and
`controllerName` values are implementation-specific.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
  namespace: apps
spec:
  gatewayClassName: example-gateway-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: apps
spec:
  parentRefs:
  - name: web-gateway
    sectionName: http
  hostnames:
  - shop.example.test
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-service
      port: 80
```

Apply and inspect the resources:

```bash
kubectl apply -f gateway-and-route.yaml
kubectl get gatewayclass
kubectl get gateway,httproute -n apps
kubectl describe gateway web-gateway -n apps
kubectl describe httproute web-route -n apps
```

The referenced Service must exist in the route's namespace in this example,
and its Service port must match the `backendRefs.port`. The Service then
selects the application Pods through its selector and target port.

## Important fields

### GatewayClass

```yaml
spec:
  controllerName: example.net/gateway-controller
```

- `controllerName` identifies the implementation responsible for the class.
- GatewayClasses are usually installed or provisioned by the Gateway API
  implementation; do not invent a class name or controller name.
- Inspect available classes with `kubectl get gatewayclass -o wide` and use
  the one required by the task.

### Gateway listeners

Each listener declares an address/port and protocol. Common protocols include
`HTTP` and `HTTPS`; implementation support and optional features vary.

- `name` uniquely identifies a listener within the Gateway. A route can use
  `parentRefs.sectionName` to attach specifically to it.
- `port` is the listener port on the Gateway data plane; it is not the backend
  Service's port.
- `hostname` optionally restricts which hostnames the listener accepts.
- `allowedRoutes` controls which route kinds and namespaces may attach.
- An HTTPS listener typically uses `tls` configuration with a certificate
  reference. The certificate Secret is normally in the Gateway's namespace;
  follow the implementation's supported TLS configuration.

Example HTTPS listener:

```yaml
listeners:
- name: https
  hostname: shop.example.test
  port: 443
  protocol: HTTPS
  tls:
    mode: Terminate
    certificateRefs:
    - kind: Secret
      name: shop-tls
  allowedRoutes:
    namespaces:
      from: Same
```

### HTTPRoute

- `parentRefs` identifies the Gateway to which the route attaches. Use
  `sectionName` to select a listener when needed.
- `hostnames` limits the HTTP hostnames matched by the route. A hostname must
  be compatible with the selected listener's hostname, if one is set.
- `rules` contain request matches and actions, including backend references.
- `matches` can match paths, headers, query parameters, and methods depending
  on the API version and implementation support.
- `PathPrefix` is the common path match type. Do not confuse it with the
  Ingress `pathType` field; Gateway API uses its own match schema.
- `backendRefs` refer to Services by name and Service port. Weights can split
  traffic among backends when supported and configured.
- `filters` can modify or redirect requests. Check support for optional or
  extended filters in the selected implementation.

When multiple routes match, the Gateway API defines precedence rules, but
implementation-specific behavior and supported features still matter. For
basic exam tasks, make the hostname/path match unambiguous and verify the
route's status.

## Route attachment and namespaces

A route attaches only if the target Gateway allows it. A listener's
`allowedRoutes.namespaces.from` commonly uses:

| Value | Meaning |
|---|---|
| `Same` | Only routes in the Gateway's namespace may attach. |
| `All` | Routes from any namespace may attach. |
| `Selector` | Routes from namespaces matching the supplied label selector may attach. |

The default attachment policy can depend on the listener and API defaults;
set `allowedRoutes` explicitly when the task requires a particular policy.
Setting a namespace selector does not itself permit a route to reference
arbitrary resources across namespaces.

By default, references such as an `HTTPRoute` backend point to a Service in
the route's own namespace. Cross-namespace references generally require an
explicit `ReferenceGrant` in the namespace containing the referenced
resource. A grant is an authorization boundary: it declares which source
namespaces and resource kinds may refer to which target kinds/names.

For example, if an HTTPRoute in namespace `apps` needs to reference a Service
in namespace `shared`, a `ReferenceGrant` belongs in `shared`. Its `from`
entry identifies `HTTPRoute` in `apps`; its `to` entry identifies `Service`
in `shared`. Use the exact API schema and scope required by the task rather
than granting broad access.

If a reference is not permitted or cannot be resolved, inspect the route's
`ResolvedRefs` condition and the Gateway API implementation's events.

## Install and discover

Gateway API CRDs and a Gateway controller/data plane are separate concerns.
The CRDs may be installed by a cluster administrator or bundled with an
implementation, but installing CRDs alone does not provide traffic handling.
Follow the task's instructions for the required implementation and version.

```bash
# See whether Gateway API types are registered
kubectl api-resources --api-group=gateway.networking.k8s.io
kubectl get crd | grep gateway.networking.k8s.io

# Discover installed classes and Gateways
kubectl get gatewayclass
kubectl get gateways -A
kubectl get httproutes -A
kubectl get referencegrants -A

# Inspect the schema supported by this cluster
kubectl explain gateway --api-version=gateway.networking.k8s.io/v1
kubectl explain httproute.spec.rules
```

The `grep` example is convenient in a Unix-like shell. In PowerShell, use:

```powershell
kubectl get crd | Select-String gateway.networking.k8s.io
```

If the API resources are missing, determine whether the Gateway API CRDs
should be installed as part of the task. Do not apply a random CRD bundle or
controller version; match the Kubernetes cluster, Gateway API release, and
implementation version specified by the environment.

## Status and troubleshooting

Start by checking whether the class, Gateway, and route are recognized and
accepted:

```bash
kubectl get gatewayclass <class-name> -o yaml
kubectl get gateway <gateway-name> -n <gateway-namespace> -o yaml
kubectl get httproute <route-name> -n <route-namespace> -o yaml
kubectl describe gateway <gateway-name> -n <gateway-namespace>
kubectl describe httproute <route-name> -n <route-namespace>
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Look at the Gateway API status conditions, especially:

- `Accepted`: the controller accepts the resource or route attachment.
- `Programmed`: the Gateway configuration has been programmed by the
  implementation.
- `ResolvedRefs`: references such as backend Services and TLS Secrets resolve
  and are permitted.

Condition availability can differ by resource, controller, and version. Check
the condition's `reason` and `message`, not just `status`.

If requests fail, check in this order:

1. **Implementation:** Is the controller/data plane installed and healthy?
   Inspect its Pods, Services, events, and logs in its namespace.
2. **Class:** Does the named GatewayClass exist, and does its controller
   report it accepted?
3. **Gateway:** Is the Gateway using the right class? Is the listener valid,
   programmed, and listening on the expected port/protocol?
4. **Attachment:** Does the route's `parentRefs` name the right Gateway and
   listener? Does `allowedRoutes` permit the route's namespace and kind?
5. **Matching:** Do listener hostname, route hostname, request host, and path
   agree?
6. **References:** Does the backend Service exist in the expected namespace?
   Does the backend port name/number match a Service port? Is a
   `ReferenceGrant` needed?
7. **Backend health:** Does the Service have endpoints and healthy Pods?
   Inspect `kubectl get svc,endpoints,endpointslices,pods -n <namespace>`.
8. **External access:** Is the implementation's data plane reachable through
   its Service or external load balancer? A valid route does not guarantee
   external connectivity.

Gateway API status explains configuration acceptance and programming; it
does not prove an application is healthy or reachable from a client.

## CKA quick checklist

1. Confirm the active context, required namespace, and requested API version.
2. Check if Gateway API resources and an implementation are already installed.
3. Inspect `GatewayClass` objects and use the exact available class.
4. Create/update the Gateway listener with the required protocol, port,
   hostname, TLS, and route attachment policy.
5. Create/update the HTTPRoute with the correct parent/listener, hostname,
   path match, and Service backend port.
6. For cross-namespace attachment or references, check `allowedRoutes` and
   the necessary `ReferenceGrant`.
7. Verify Gateway and route status conditions, then verify the Service and
   backend endpoints.
8. Do not equate an accepted YAML manifest with a programmed Gateway or a
   reachable application.

## Quick reference

| Goal | Command or field |
|---|---|
| List available GatewayClasses | `kubectl get gatewayclass` |
| Inspect Gateways in all namespaces | `kubectl get gateways -A` |
| Inspect HTTPRoutes in all namespaces | `kubectl get httproutes -A` |
| Inspect route attachment | `spec.parentRefs` |
| Select a listener by name | `parentRefs[].sectionName` |
| Set listener route namespace policy | `listeners[].allowedRoutes.namespaces.from` |
| Match a URL path prefix | `rules[].matches[].path.type: PathPrefix` |
| Route to a Service | `rules[].backendRefs[]` |
| Allow a cross-namespace reference | `ReferenceGrant` in the target namespace |
| Check controller decision | Gateway/route `status.conditions` |

Official references:

- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [Gateway API concepts](https://gateway-api.sigs.k8s.io/concepts/)
- [Gateway API guides](https://gateway-api.sigs.k8s.io/guides/)
