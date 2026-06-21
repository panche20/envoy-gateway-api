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


# Create Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: external-gw
  namespace: envoy-gatew
ay-system

spec:
  gatewayClassName: eg

  listeners:

  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All

  - name: https
    protocol: HTTPS
    port: 443

    tls:
      mode: Terminate

      certificateRefs:
      - kind: Secret
        name: demo-cert

    allowedRoutes:
      namespaces:
        from: All
```

---

# Labs Covered

---

## 1. Basic HTTP Routing

```text
Client
 ↓
Gateway
 ↓
HTTPRoute
 ↓
Service
 ↓
Pod
```

---

## 2. Host-Based Routing

```text
api.demo.local
        ↓
    api-service

web.demo.local
        ↓
 frontend-service
```

---

## 3. Path-Based Routing

```text
/api
 ↓
api-service

/admin
 ↓
admin-service

/
 ↓
frontend
```

---

## 4. Canary Deployment (Traffic Splitting)

Weighted traffic distribution:

```text
80% → api-v1

20% → api-v2
```

Implemented using:

```yaml
backendRefs:
- name: api-v1
  weight: 80

- name: api-v2
  weight: 20
```

---

## 5. Header-Based Routing

Traffic sent based on request headers.

Example:

```text
version: beta
        ↓
      api-v2

default
        ↓
      api-v1
```

---

## 6. URL Rewrite

Client request:

```text
/api/get
```

Rewritten by Envoy to:

```text
/get
```

using:

```yaml
filters:
- type: URLRewrite
```

---

## 7. HTTPS / TLS Termination

Architecture:

```text
Client
  |
 HTTPS
  |
Envoy Gateway
(TLS termination)
  |
 HTTP
  |
Backend
```

Certificate stored as Kubernetes Secret.

---

## 8. BackendTrafficPolicy

Implemented:

### Request Timeout

```yaml
timeout:
  http:
    requestTimeout: 5s
```

### Retry Policy

```yaml
retry:
  numRetries: 3

  retryOn:
    triggers:
    - connect-failure
    - retriable-status-codes

    httpStatusCodes:
    - 503
```

---

# Envoy Gateway Extension CRDs

Available CRDs:

* BackendTrafficPolicy
* ClientTrafficPolicy
* SecurityPolicy
* EnvoyExtensionPolicy
* EnvoyPatchPolicy
* HTTPRouteFilter

---

# Request Flow

```text
Client
  ↓
Envoy Gateway
  ↓
Gateway
  ↓
HTTPRoute
  ↓
BackendTrafficPolicy
  ↓
Service
  ↓
Pod
```

---

# Troubleshooting

## Attached Routes = 0

Cause:

HTTPRoute not attached to Gateway.

Fix:

```yaml
parentRefs:
- name: external-gw
  namespace: envoy-gateway-system
```

---

## Empty Reply From Server

Cause:

Sending HTTP request to HTTPS listener.

Incorrect:

```bash
curl http://IP:PORT
```

Correct:

```bash
curl -k https://IP:PORT
```

---

## No Endpoints

Check:

```bash
kubectl get endpoints
```

Verify service selector and pod labels.

---

## HTTPRoute Not Accepted

Check:

```bash
kubectl describe httproute
```

Expected:

```text
Accepted=True
ResolvedRefs=True
```

---

## Gateway Not Programmed

Verify:

```bash
kubectl describe gateway -n envoy-gateway-system
```

Expected:

```text
Accepted=True
Attached Routes > 0
```

---

# Future Enhancements

* Rate Limiting
* Circuit Breakers
* Fault Injection
* JWT Authentication
* Prometheus Metrics
* Grafana Dashboards
* Jaeger Tracing
* OpenTelemetry Integration

---

# References

* Kubernetes Gateway API
* Envoy Gateway
* Gateway API CRDs
* BackendTrafficPolicy
* ClientTrafficPolicy

---

## Learning Outcomes

After completing this lab, I gained hands-on experience with:

* Kubernetes Gateway API resources
* Envoy Gateway architecture
* HTTPRoute traffic management
* Canary deployments
* Header-based routing
* URL rewrites
* TLS termination
* Retry and timeout policies
* Production-grade API Gateway concepts

