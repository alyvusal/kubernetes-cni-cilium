# Cilium CLI

[back](../README.md)

```bash
# check status of Cilium
kubectl -n kube-system exec -ti cilium-jhhpp -- cilium-dbg status --verbose

# test connectivity between pods
cilium connectivity test
cilium config view

# which programs are attached to which devices
bpftool net show
```
