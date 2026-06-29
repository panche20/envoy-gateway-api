# Path Based Routing

Path-based routing is where Gateway API starts becoming very powerful. Instead of routing based on hostname, we'll route based on URL paths.

```
http://<gateway>/ --> frontend
http://<gateway>/api --> api
http://<gateway>/admin --> admin
```

**Single IP, single Gateway, multiple backends**

**Architecture**

```
                     Gateway
                        |
                 HTTPRoute (Path Matching)
                        |
          -----------------------------------
          |               |                 |
         /             /api             /admin
          |               |                 |
      frontend         api-service      admin-service
```

## Step 1: Deploy Frontend

```
kubectl create deployment frontend --image=nginx

kubectl expose deployment frontend \
  --port=80
```

## Step 2: Deploy API

```
kubectl create deployment api \
--image=hashicorp/http-echo \
-- /http-echo -text="Hello from API"

kubectl expose deployment api \
--port=5678
```

## Step 3: Deploy Admin

```
kubectl create deployment admin \
--image=hashicorp/http-echo \
-- /http-echo -text="Hello from Admin"

kubectl expose deployment admin \
--port=5678

kubectl get svc

Expected result:
frontend    ClusterIP
api         ClusterIP
admin       ClusterIP
```

## Step 4: Create HTTPRoute

Create file path-route.yaml:

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: path-route
spec:
  parentRefs:
  - name: external-gw
    namespace: envoy-gateway-system

  rules:

  - matches:
    - path:
        type: PathPrefix
        value: /api

    backendRefs:
    - name: api
      port: 5678

  - matches:
    - path:
        type: PathPrefix
        value: /admin

    backendRefs:
    - name: admin
      port: 5678

  - matches:
    - path:
        type: PathPrefix
        value: /

    backendRefs:
    - name: frontend
      port: 80
```

```
kubectl apply -f path-route.yaml
```

## Step 5: Verify Route

```
kubectl describe httproute path-route

Accepted=True
ResolvedRefs=True
```

## Step 6: Find Gateway NodePort

```
kubectl get svc -n envoy-gateway-system

Example:
envoy-external-gw
80:31567/TCP

Assume:
Node IP : 172.31.31.10
Node Port : 31567
```

## Step 7: Test Frontend

```
curl http://172.31.31.10:31567/

Expected Result:
Welcome to nginx!
```

**Test API**

```
curl http://172.31.31.10:31567/api

Expected Result:
Hello from API
```

**Test Admin**

```
curl http://172.31.31.10:31567/admin

Hello from Admin
```

**How Envoy Processes This**

```
curl http://gateway/api
```

```
Client
  ↓
Gateway Listener
  ↓
HTTP Connection Manager
  ↓
Path Matching
  ↓
"/api"
  ↓
Cluster(api-service)
  ↓
Pod
```

*For curl http://gateway/admin* --> *Envoy selects: Cluster(admin service)*

*For everything else* --> *Envoy selects: Cluster(frontend service)*

```
Why should / be last? - Because / is a catch-all prefix.
If it is evaluated before: /api & /admin - it would match everything and steal traffic.
Envoy and Gateway API internally prioritize longest prefix matches, but keeping / last improves readability and avoids ambiguity.
```