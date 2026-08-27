# Cilium CLI

[back](../README.md)

```bash
# check status of Cilium
cilium status
cilium-health status

kubectl -n kube-system exec -it ds/cilium -- cilium-dbg service list
kubectl -n kube-system exec -it ds/cilium -- cilium-dbg status --verbose
kubectl -n kube-system exec -it ds/cilium -- cilium-dbg bpf ipmasq list
kubectl -n kube-system exec -it ds/cilium -- cilium-dbg bpf egress list

# This collects logs, configuration, and runtime information from all Cilium components
cilium sysdump --node-list kind-worker2 --logs-since-time 2h

# run below command to see visual flow
cilium connectivity test

# Transparent Encryption public key, shows us the public key for the node
cilium status -o json | jq '.cilium_status[].encryption.wireguard.interfaces'

kubectl get ciliumnode -o json | jq "$(cat <<EOF
.items[] | {
"name": .metadata.name,
"wg-pub-key": .metadata.annotations["network.cilium.io/wg-pub-key"]
}
EOF
)"

# test connectivity between pods
cilium connectivity test
cilium config view

# which programs are attached to which devices
bpftool net show
```
