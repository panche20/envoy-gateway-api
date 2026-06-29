# Host-based Routing

Main goal of host based routing is to serve multiple applications through the same gateway.

```
api.demo.local -> api-service
web.demo.local -> frontend-service
```

This is exactly similar to sharing one Load balancer for :
- api.company.com
- www.company.com

## Step 1 : Deploy two applications

**Frontend**

```
kubectl create deployment frontend \
--image=nginx

kubectl expose deployment frontend \
--port=80
```

**API**

```
Use hashicorp/http-echo:

kubectl create deployment api \
--image=hashicorp/http-echo \
-- /http-echo -text="Hello from API"

Expose deployment using:
kubectl expose deployment api --port=5678

Verify using:
kubectl get svc
```

## Step 2 : Create HTTPRoute for frontend

Create file frontend-route.yaml

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
spec:
  parentRefs:
  - name: external-gw
    namespace: envoy-gateway-system

  hostnames:
  - web.demo.local

  rules:
  - backendRefs:
    - name: frontend
      port: 80
```

```
kubectl apply -f frontend-route.yaml
```

## Step 3: Create API Route

Create file called api-route.yaml

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
  - name: external-gw
    namespace: envoy-gateway-system

  hostnames:
  - api.demo.local

  rules:
  - backendRefs:
    - name: api
      port: 5678
```

```
kubectl apply -f api-route.yaml
```

## Step 4 : Verify what is created

```
kubectl describe gateway external-gw -n envoy-gateway-system

Should now show as:
Attached Routes: 3
```

## Step 5: Find Envoy NodePort

```
kubectl get svc -n envoy-gateway-system

Example:
envoy-external-gw    NodePort
80:31567/TCP
```

## Step 6: Test Host Header Routing

Since DNS doesn't exist, simulate with curl.

**Web request**

```
curl \
-H "Host: web.demo.local" \
http://<NODE-IP>:31567
```

```
Welcome to Nginx
```

**API Request**

```
curl \
-H "Host: api.demo.local" \
http://<NODE-IP>:31567
```

```
Hello from API
```

**Now, what happens internally is:**

Envoy listener receives request : Host: api.demo.local

Envoy performs:

```
Virtual Host Match --> HTTPRoute (api-route) --> Cluster (api-service) --> POD
```

Similarly,

```
If Host: web.demo.local ---- then

Virtual Host Match --> HTTPRoute (frontend-route) --> Cluster (frontend-service) --> POD
```

### Verify Envoy Config

You can inspect generated resources using below commands:

```
kubectl get httproute

kubectl describe httproute api-route

Look for:
Accepted=True
ResolvedRefs=True
```

**Architecture after Lab 2**

```
               Gateway
                   |
      ---------------------------
      |                         |
api.demo.local           web.demo.local
      |                         |
HTTPRoute api             HTTPRoute frontend
      |                         |
api-service              frontend-service
      |                         |
pods                     pods
```