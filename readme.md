# Master Gateway API with Envoy Gateway

Before Gateway API traditional Kubernetes Ingress had several limitations. 
Ingress manily supports:
- host routing
- path routing
- TLS termination

But modern requirements need:
- Traffic splitting
- Header matching
- Rate limiting
- Authentication
- Retry policies
- TCP/UDP routing
- gRPC
- Multi-tenant ownership

Different ingress controllers solved this with annotations: eg.

```
nginx.ingress.kubernetes.io/rewrite-target: /
nginx.ingress.kubernetes.io/rate-limit: "100"
```

Problems:

- Vendor-specific
- Hard to migrate
- No standardization

**Gateway API solves this** 
It separates responsibilities:

**Infrastructure Team**
Creates:
GatewayClass
Gateway

**Application Team**

Creates:
HTTPRoute
TCPRoute
GRPCRoute

This separation enables multi-tenancy.

Architecture:

```
               GatewayClass
                     |
                     ↓
                 Gateway
                     |
        ----------------------------
        |            |             |
    HTTPRoute     TCPRoute      GRPCRoute
        |
     Services
        |
       Pods
```

**Envoy Gateway Architecture**

```
                Kubernetes API
                       |
                 Envoy Gateway
                  (Controller)
                       |
               xDS Configuration
                       |
                   Envoy Proxy
                       |
                   Applications
```

**Install Gateway API Standard CRDs**
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

**Install Envoy Gateway**

```
kubectl create ns envoy-gateway-system

helm install eg oci://docker.io/envoyproxy/gateway-helm \
-n envoy-gateway-system \
--create-namespace
```

**Create gatewayclass**

Create gateway-class.yaml file & paste in below code:
```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

**Create Gateway**

Create file name gateway.yaml & paste below code:

```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: external-gw
  namespace: envoy-gateway-system
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
```
