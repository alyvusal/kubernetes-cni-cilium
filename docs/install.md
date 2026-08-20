# Install

[back](../README.md)

## [Install with CLI](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)

```bash
# Install
cilium install
cilium status
```

## [Install with HELM](https://docs.cilium.io/en/stable/installation/k8s-install-helm/#installation-using-helm)

```bash
helm repo add cilium https://helm.cilium.io/

helm upgrade -i cilium cilium/cilium \
  --version 1.20.0 \
  -n kube-system

# after any change in helm
kubectl -n kube-system rollout restart deployment cilium-operator
kubectl -n kube-system rollout restart ds cilium
```

## Verify installation

```bash
# Validate connectivity in cluster with CLI
cilium connectivity test

# Validate connectivity in cluster with deployment
kubectl create ns cilium-test
kubectl apply -n cilium-test -f https://raw.githubusercontent.com/cilium/cilium/1.20.0/examples/kubernetes/connectivity-check/connectivity-check.yaml
# The pod name indicates the connectivity variant and the readiness and liveness gate indicates success or failure of the test
kubectl get pods -n cilium-test

# Test network performance
cilium connectivity perf

# check endpoints
kubectl -n kube-system get pods -l k8s-app=cilium  # single node
kubectl -n kube-system exec cilium-7h44q -- cilium-dbg endpoint list  # multi node
kubectl get ciliumendpoints -A

cilium-dbg status --verbose
```
