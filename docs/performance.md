# [Performance Optimization](https://docs.cilium.io/en/stable/operations/performance/)

## [Bottleneck Bandwidth and Round-trip propagation time (BBR)](https://docs.cilium.io/en/stable/operations/performance/tuning/#bottleneck-bandwidth-and-round-trip-propagation-time-bbr)

improves throughput and reduces latency on long or high-bandwidth connections.

## [BIG TCP](https://docs.cilium.io/en/stable/operations/performance/tuning/#big-tcp)

increases efficiency for large-scale data transfers by allowing larger packet sizes, reducing per-packet overhead.

- [Accelerate network performance with Cilium BBR](https://isovalent.com/blog/post/accelerate-network-performance-with-cilium-bbr/)
- [BIG Performances with BIG TCP on Cilium](https://isovalent.com/blog/post/big-tcp-on-cilium/)

## [eBPF host routing](https://docs.cilium.io/en/stable/operations/performance/tuning/#ebpf-host-routing)

Bypasses parts of the traditional networking stack to deliver faster packet forwarding directly within the kernel.

eBPF host-routing is enabled by default if your kernel supports it, so it does not require further explanation here.

## Service Traffic Distribution [area/topology]

Prioritizes endpoints in the same area when available, helping reduce latency and inter-area costs.
Note that KPR (Kube Proxy replacement) is required for Service Traffic Distribution.

For a given service, if `externalTrafficPolicy` or `internalTrafficPolicy` is set to `Local`, that policy takes precedence over `trafficDistribution` for the corresponding traffic type

```bash
kubectl apply -f examples/performance/std/std-deployment.yaml

kubectl apply -f examples/performance/std/std-poller-distribution.yaml
kubectl apply -f examples/performance/std/std-service.yaml
# check logs what pods curls

# enable trafficDistribution in service reapply and check curl logs again
```

## Local Redirect Policy (LRP) [node]

### Using LRP to Keep Traffic on the Same Node

Prioritizes node-local backends so traffic stays on the same node, reducing crossnode hops and latency

LRP enables traffic destined for a particular IP address and port/protocol tuple or Kubernetes service to be redirected locally to backend pods within a node, using eBPF.

There are two policy types:

- `AddressMatcher`: Matches an IP:port destination and redirects locally. AddressMatcher is handy for node-local metadata caches on managed Kubernetes platforms such as EKS and GKE. For example, you can match 169.254.169.254:80 (instance metadata) and redirect those requests to a node-local cache pod. Cache misses will then go to the external metadata service.
- `ServiceMatcher`: Matches a specific service and redirects to node-local backends for that service. This is the most common policy type and the one used in the two examples covered in this chapter.

`localRedirectPolicies.enabled: true` in `k8s/helm/k3d-values.yaml` is required. After a Helm upgrade, restart `cilium-operator` so it registers the `CiliumLocalRedirectPolicy` CRD (Helm does not roll the operator unless `operator.rollOutPods: true`). Otherwise `kubectl apply` fails with `no matches for kind "CiliumLocalRedirectPolicy"`.

```bash
kubectl -n kube-system rollout restart deploy/cilium-operator
kubectl get crd ciliumlocalredirectpolicies.cilium.io
kubectl apply -f examples/performance/lpr/lpr-deployment.yaml

kubectl apply -f examples/performance/lpr/netshoot-client.yaml
kubectl exec -it netshoot-client -- curl echo-server:8080  # run twice to see the difference
kubectl get svc echo-server

$ kubectl exec -it -n kube-system ds/cilium -- cilium-dbg service list
ID Frontend                Service Type  Backend
6  10.96.219.113:8080/TCP  ClusterIP     1 => 10.0.0.209:8080/TCP (active)
                                         2 => 10.0.2.27:8080/TCP (active)

kubectl apply -f examples/performance/lpr/clrp-svc.yaml
```

When a policy of this type is applied, the existing service entry created by Cilium will be replaced with a new service entry of type LocalRedirect. This entry can only have node-local backend pods.

Let’s verify the behavior by executing `cilium-dbg service list` on the Cilium agent running on the same node as our client pod. The following output confirms that Cilium will forward traffic to the echo-server service only to the local backend (echo-server-78fbb56cc7-pvkgf, with IP 10.0.2.27)

```bash
$ kubectl -n kube-system exec -it cilium-5d5jq -- cilium-dbg service list
ID Frontend                Service Type   Backend
7  10.96.219.113:8080/TCP  LocalRedirect  1 => 10.0.2.27:8080/TCP (active)
```

```txt
Pod
 │
 ▼
Service
 │
 ├── internalTrafficPolicy
 │
 ▼
Cilium service handling
 │
 ├── LRP match → local redirected endpoint
 │
 ▼
Backend
```

- `internalTrafficPolicy: Local` → tells the Kubernetes Service to only use endpoints on the same node.
  - So if your goal is simply "don't send Service traffic to another node", internalTrafficPolicy: Local is the native Kubernetes mechanism.
- `Cilium LRP` → tells Cilium to redirect matching Service traffic to a specific local backend, usually selected by LRP rules.
  - LRP is more useful when you need special node-local redirection, e.g. redirecting traffic destined for a Service to a particular local daemon/pod.

### Using LRP to Redirect DNS Traffic Locally

The most common use case for LRP is to optimize DNS performance, alongside the [NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns) Kubernetes architecture. This architecture relies on a DNS caching agent running on each cluster node as a DaemonSet. With LRP you can be sure that DNS traffic from a pod goes to the DNS cache running on the same node as the pod, reducing latency, among [other benefits](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/#motivation)

See [config sample](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/#configuration), [yaml](https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml)

```bash
# Need to fix
# __PILLAR__LOCAL__DNS__: 169.254.20.10
# __PILLAR__DNS__SERVER__: 10.96.0.10
# in deployment

kubedns=$(kubectl get svc kube-dns -n kube-system -o jsonpath={.spec.clusterIP})
domain="cluster.local"
localdns=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

wget https://raw.githubusercontent.com/kubernetes/kubernetes/refs/heads/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml

# If kube-proxy is running in IPTABLES mode
sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/__PILLAR__DNS__SERVER__/$kubedns/g" nodelocaldns.yaml

# __PILLAR__CLUSTER__DNS__ and __PILLAR__UPSTREAM__SERVERS__ will be populated by the node-local-dns pods. In this mode, the node-local-dns pods listen on both the kube-dns service IP as well as <node-local-address>, so pods can look up DNS records using either IP address.

# If kube-proxy is running in IPVS mode
sed "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/,__PILLAR__DNS__SERVER__//g; s/__PILLAR__CLUSTER__DNS__/$kubedns/g" nodelocaldns.yaml

# this file ready to use, just apply
kubectl apply -f examples/performance/lpr/nodelocaldns.yaml

# or use ready yaml
# __PILLAR__LOCAL__DNS__: 169.254.20.10
# __PILLAR__DNS__SERVER__: 10.96.0.10
kubectl apply -f examples/performance/lpr/node-local-dns.yaml

$ kubectl exec -it ds/cilium -n kube-system -- cilium-dbg service list
root@k3d-k3s-default-agent-0:/home/cilium# cilium service list | grep 53
9    10.43.0.10:53/TCP       ClusterIP      1 => 10.10.0.6:53/TCP (active)
10   10.43.0.10:53/UDP       ClusterIP      1 => 10.10.0.6:53/UDP (active)

# apply LRP to redirect DNS traffic locally
kubectl apply -f examples/performance/lpr/clrp-dns.yaml

$ kubectl exec -it ds/cilium -n kube-system -- cilium-dbg service list
root@k3d-k3s-default-agent-0:/home/cilium# cilium service list | grep 53
9    10.43.0.10:53/TCP       LocalRedirect   1 => 172.18.0.5:53/TCP (active)
10   10.43.0.10:53/UDP       LocalRedirect   1 => 172.18.0.5:53/UDP (active)
```

Let’s also get the IP of the “non-local” DNS cache pod:

```bash
$ DNS_NON_LOCAL="$(kubectl -n kube-system get po \
-l k8s-app=node-local-dns \
--field-selector spec.nodeName=kind-worker2 \
-o jsonpath='{.items[].status.podIP}')"
$ echo "$DNS_NON_LOCAL"
10.0.2.20
```

We can now make DNS requests from the client pod and check the DNS request
counts on the local DNS cache pod and the one running on the other worker node:

```bash
$ kubectl exec netshoot-client -- curl "$DNS_LOCAL:9253/metrics" | \
grep 'coredns_dns_request_size_bytes_count'
coredns_dns_request_count_total{zone="cluster.local."} 1 # local
$ kubectl exec netshoot-client -- curl "$DNS_NON_LOCAL:9253/metrics" | \
grep 'coredns_dns_request_size_bytes_count'
coredns_dns_request_count_total{zone="cluster.local."} 1 # non-local
```

The values here indicate that only one request has been made to each DNS cache.
Let’s run a few DNS requests, and then check the counts again:

```bash
$ kubectl exec netshoot-client -- nslookup oreilly.com
$ kubectl exec netshoot-client -- nslookup isovalent.com
$ kubectl exec netshoot-client -- nslookup cilium.io
$ kubectl exec netshoot-client -- curl "$DNS_LOCAL:9253/metrics" | \
grep 'coredns_dns_request_size_bytes_count'
coredns_dns_request_count_total{zone="cluster.local."} 14 # local
$ kubectl exec netshoot-client -- curl "$DNS_NON_LOCAL:9253/metrics" \
| grep 'coredns_dns_request_size_bytes_count'
coredns_dns_request_count_total{zone="cluster.local."} 1 # non-local
```

As expected, it increases for the local DNS cache, while the number for the other
cache stays the same.

```bash
$ kubectl exec netshoot-client -- curl "$DNS_LOCAL:9253/metrics" | \
grep 'coredns_dns_request_size_bytes_count'
coredns_dns_request_count_total{zone="cluster.local."} 14 # local
$ kubectl exec netshoot-client -- curl "$DNS_NON_LOCAL:9253/metrics" \
| grep 'coredns_dns_request_size_bytes_count'
coredns_dns_request_count_total{zone="cluster.local."} 1 # non-local
```

## Direct Server Return (DSR) [edge]

Preserves the client source IP and sends responses directly from backend pods to clients, improving return path performance

```bash
kubectl apply -f examples/performance/dsr/dsr-deployment.yaml
kubectl apply -f examples/performance/dsr/dsr-service.yaml

docker run -d --network k3d-k3s-default --rm --name client nicolaka/netshoot:v0.8 sleep infinity
docker exec -it client ip -4 -o a show eth0 | awk '{print $4}'

kubectl exec -it ds/cilium -n kube-system -- cilium-dbg status --verbose | grep -A 7 "KubeProxyReplacement Details"
kubectl -n kube-system exec -it ds/cilium -- cilium-dbg bpf lb list
docker exec -it client curl 172.18.0.4:30080
```

## eXpress Data Path (XDP) [node]

Processes packets at the earliest possible point in the kernel to reduce per-packet overhead and lower latency

The eXpress Data Path (XDP) feature allows for eBPF code to be executed in hard‐ware directly on the physical network interface, processing packets before they even transit to the CPU and the kernel.

The main use cases for XDP are protection against distributed denial-of-service (DDoS) attacks and firewalling, since XDP enables packets to be dropped early with minimal overhead. XDP also supports load balancing and forwarding, including redirection to other network interfaces or CPU cores, and it can be used for pre-stack filtering, such as dropping unwanted protocols like UDP or Stream Control Trans‐mission Protocol (SCTP) before they reach the kernel’s networking stack.

Unlike user space frameworks such as Data Plane Development Kit (DPDK), XDP still works in cooperation with the kernel. This means you can offload the fast path to XDP while leaving the kernel to handle more complex processing, such as full TCP connection tracking or policy enforcement.

XDP applies only to packets that arrive through a physical network interface. Traffic between pods on the same node, for example, does not pass through the network interface controller (NIC) and will not benefit. However, for ingress traffic from external clients or LoadBalancer services, XDP can reduce per-packet latency and CPU usage by processing traffic earlier in the stack.

## Maglev [edge]

Provides consistent hashing so the same flow lands on the same backend, improving connection stability during node changes

Maglev provides consistent hashing, ensuring that client connections identified by the same 5-tuple—source IP address, source port, destination IP address, destination port, and transport protocol—are routed to the same backend, regardless of which external node handles the request. This helps avoid connection drops during node failures or load balancer changes. Operators should consider enabling it when session stickiness and stability are critical, especially for TCP-based workloads.

## Netkit [node]

Replaces the traditional veth path with a kernel-native device model to cut overhead and bring container networking performance closer to host levels

- [Cilium netkit: The Final Frontier in Container Networking Performance](https://isovalent.com/blog/post/cilium-netkit-a-new-container-networking-paradigm-for-the-ai-era/)
