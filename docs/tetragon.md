# [Tetragon](https://tetragon.io/docs/)

[back](../README.md)

```bash
helm upgrade -i tetragon cilium/tetragon \
  -n kube-system \
  --version 1.2.0 \
  --set tetragon.hostProcPath=/procHost

# test app
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.15.3/examples/minikube/http-sw-app.yaml
```
