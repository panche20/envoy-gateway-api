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
