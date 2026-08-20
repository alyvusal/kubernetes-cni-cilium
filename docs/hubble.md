# [Hubble](https://docs.cilium.io/en/stable/observability/hubble/)

[back](../README.md)

```bash
helm upgrade -i cilium cilium/cilium \
  --version 1.16.5 \
  -n kube-system \
  --reuse-values \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# validate access
cilium hubble port-forward&
hubble status
hubble observe
cilium hubble ui

# run below command to see visual flow
cilium connectivity test

cilium hubble port-forward&
hubble status
hubble observe --pod testpod -f
hubble observe --http-header "X-Header-Add-1:header-add-1"
```

Troubleshoot

```bash
cilium status
kubectl -n kube-system exec ds/cilium -- cilium-dbg service list
```

## [Hubble CLI](https://docs.cilium.io/en/stable/observability/hubble/hubble-cli/)

```bash
cilium hubble ui
cilium hubble port-forward &
hubble status
hubble list nodes
hubble observe
hubble observe --from-pod netshoot-client
hubble observe --from-pod netshoot-client --http-path "/index.html"
hubble observe --label app.kubernetes.io/name=nginx
hubble observe -t policy-verdict
```
