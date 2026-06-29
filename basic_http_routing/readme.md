# Basic HTTP Routing

## Phase 1 : Install Gateway API CRDs

```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

Verify:
kubectl get crd | grep gateway

Expected:
gatewayclasses.gateway.networking.k8s.io
gateways.gateway.networking.k8s.io
httproutes.gateway.networking.k8s.io
grpcroutes.gateway.networking.k8s.io
referencegrants.gateway.networking.k8s.io
```

## Phase 2 : Install Envoy Gateway

```
Create Namespace:
kubectl create ns envoy-gateway-system

Install:
helm install eg oci://docker.io/envoyproxy/gateway-helm \
-n envoy-gateway-system \
--create-namespace

Verify:
kubectl get pods -n envoy-gateway-system
```

## Phase 3 : Create Gateway Class

Create gateway-class.yaml file & paste in below code:

```
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

```
Verify:
kubectl get gatewayclass

Expected:
NAME              CONTROLLER
eg                gateway.envoyproxy.io/gatewayclass-controller
```

## Phase 4 : Create Gateway

Create gateway.yaml and enter below details:

```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: eg

  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

```
Apply:

kubectl apply -f gateway.yaml

Verify:
kubectl get gateway

Expected:
NAME         CLASS   ADDRESS
my-gateway   eg
```

## Phase 5 : Observe Envoy Proxy

Envoy Gateway controller creates an Envoy deployment.

```
Check:
kubectl get all -A | grep envoy

You'll see:
envoy-gateway
envoy-my-gateway

and a service:
kubectl get svc -A

Something like:
envoy-my-gateway
NodePort
```

## Phase 6 : Deploy Sample App

```
kubectl create deployment nginx --image=nginx

Expose service:
kubectl expose deployment nginx \
--port=80

Verify:
kubectl get svc nginx
```

## Phase 7 : Create HTTPRoute

Create route.yaml

```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx-route
spec:
  parentRefs:
  - name: my-gateway

  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /

    backendRefs:
    - name: nginx
      port: 80
```

```
Apply:
kubectl apply -f route.yaml

Verify:
kubectl get httproute

Expected:
NAME          HOSTNAMES
nginx-route
```

## Phase 8 : Test Traffic

```
Find envoy service:
kubectl get svc -A

Example:
envoy-my-gateway NodePort
80:31555/TCP

Get node IP:
kubectl get nodes -o wide

Suppose:
172.31.10.20

Test:
curl http://172.31.10.20:31555

Expected:
Welcome to nginx!
```

**Verify the attachments:**

```
kubectl describe gateway my-gateway

Expected result:
Attached Routes: 1
Programmed: True
Accepted: True
```

Verify Httproute:

```
kubectl describe httproute nginx-route

Expected result:
Accepted=True
ResolvedRefs=True
```
