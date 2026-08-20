# Demo App

[back](../README.md)

Use samples from [Getting Started with the Star Wars Demo](https://docs.cilium.io/en/stable/gettingstarted/demo/)

- [Apply an L3/L4 Policy](https://docs.cilium.io/en/stable/gettingstarted/demo/#apply-an-l3-l4-policy)
- [Apply and Test HTTP-aware L7 Policy](https://docs.cilium.io/en/stable/gettingstarted/demo/#apply-and-test-http-aware-l7-policy)

```bash
kubectl apply -f examples/demo-app/deployment.yaml  # deploys on node0
kubectl apply -f examples/demo-app/netshoot-client.yaml  # deploys on node1

kubectl exec pod/netshoot-client -- curl -s -o /dev/null -w "%{http_code}\n" http://nginx-service # or pod ip

$ kubectl get endpoints
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME            ENDPOINTS         AGE
kubernetes      172.18.0.2:6443   11m
nginx-service   10.42.0.14:80     6m14s

$ kubectl get endpointslices.discovery.k8s.io
NAME                  ADDRESSTYPE   PORTS   ENDPOINTS    AGE
kubernetes            IPv4          6443    172.18.0.2   11m
nginx-service-jcztj   IPv4          80      10.42.0.14   6m19s
```

## Test L4 Policy

```bash
kubectl apply -f examples/demo-app/policy.yaml
# check url from allowed client
kubectl exec pod/netshoot-client -- curl -s -o /dev/null -w "%{http_code}\n" --max-time 3 http://nginx-service
# check url from denied client (--max-time 3 is to avoid hanging request)
kubectl apply -f examples/demo-app/unauthorized-client.yaml
kubectl exec pod/unauthorized-client -- curl -s -o /dev/null -w "%{http_code}\n" --max-time 3 http://nginx-service
```

## Test HTTP-aware L7 Policy

By default, the NGINX server we deployed includes an error page named 50x.html.
The current CiliumNetworkPolicy lets you access it with no restrictions

```bash
# Below allowed
kubectl exec -t netshoot-client -- curl http://nginx-service/50x.html

# Below denied with same timeout
kubectl exec -t unauthorized-client -- curl http://nginx-service/50x.html

# apply policy to deny access to 50x.html
kubectl apply -f examples/demo-app/policy-with-l7.yaml

# Test HTTP-aware L7 Policy
$ kubectl exec -t netshoot-client -- curl -s http://nginx-service/
$ kubectl exec -t netshoot-client -- curl -s http://nginx-service/50x.html
Access denied

# Below allowed
kubectl exec -t netshoot-client -- curl -s http://nginx-service/index.html
```

Check [hubble](hubble.md) to see the traffic.
