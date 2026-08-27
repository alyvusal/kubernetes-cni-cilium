# Cilium (eBPF based networking, Observability, Security)

## [eBPF](docs/ebpf.md)

## [Install](docs/install.md)

## [Demo App](docs/demo.md)

## [Egress Gateway](https://docs.cilium.io/en/stable/network/egress-gateway-toc/)

The egress gateway feature routes all IPv4 connections originating from pods and destined to specific cluster-external CIDRs through particular nodes, from now on called “gateway nodes”.

## [Hubble](docs/hubble.md)

## [Tetragon](docs/tetragon.md)

## [Policy](docs/policy.md)

## [CLI](docs/cli.md)

## Misselenous

### [CNI Chaining](https://docs.cilium.io/en/stable/installation/cni-chaining/)

CNI chaining allows to use Cilium in combination with other CNI plugins.

With Cilium CNI chaining, the base network connectivity and IP address management is managed by the non-Cilium CNI plugin, but Cilium attaches eBPF programs to the network devices created by the non-Cilium plugin to provide L3/L4 network visibility, policy enforcement and other advanced features.

### [L2 Announcements / L2 Aware LB](https://docs.cilium.io/en/stable/network/l2-announcements/)

Cilium’s L2 Announcements feature is useful in simpler environments, it has some notable limitations:

- Only one node handles traffic for each VIP, so there is no load balancing between nodes.
- Failover depends on ARP cache expiry, which can be slow.
- Faster failover relies on gratuitous ARP, but many clients ignore these messages for security reasons.

### [BGP](https://docs.cilium.io/en/stable/network/bgp-toc/)

The key [resources](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane-configuration/) involved in managing BGP Control Plane are:

- `CiliumBGPClusterConfig`: Defines one or more BGP instances and associated peer configurations that can be applied to nodes in the cluster
- `CiliumBGPPeerConfig`: Specifies peer configuration settings that can be reused across multiple peers
- `CiliumBGPAdvertisement`: Declares which prefixes (e.g., service IPs or PodCIDRs) should be advertised to the BGP routing table and under what conditions

### [Bandwidth Manager](docs/aks-bandwidth-manager.md)

### Transparent Encryption

Transparent Encryption does not use mutual TLS (mTLS) and does not require a layer 7 proxy to establish a TCP session with the workload. When a packet is sent between managed pods on two different nodes, Cilium encrypts the original packet and encapsulates it in a new WireGuard packet. This new packet is sent to the remote node, and the content is decapsulated and decrypted before being delivered to the target pod.

It’s important to note that packets are only encrypted if they are traveling between two Cilium-managed pods and those pods are on different nodes. Packets are not encrypted if they are traveling between:

- Pods on the same node
- A pod and a host process or host network namespace pod
- Two host processes or host network namespace pods
- Any process and a destination outside of the cluster
- Any communication that does not use the network, such as HTTP over UNIX socket
- Processes within the same pod

A common question is why Cilium does not encrypt traffic between pods on the same node. It may seem like a strange omission, but the rationale is simple: when packets travel between pods on the same node, Cilium’s eBPF datapath moves them directly from one pod’s network interface to the other pod’s, in kernel. There is no Linux bridge, virtual switch, or other networking system between the pods. If Cilium did perform encryption here, it would encrypt traffic only to immediately decrypt it again, so it would provide no benefit.

Transparent Encryption is performed after any resolution of ClusterIP services, so packets bound for a service’s virtual IP will be encrypted only if the selected backend is on another node. Similarly, with layer 7 network policy, the packets are encrypted only after leaving the per-node Envoy proxy on egress and are decrypted on ingress just before entering the Envoy proxy on the remote node.

The encryption and encapsulation in the datapath are performed entirely using the kernel’s implementation of WireGuard. Cilium is responsible for key distribution, configuring the WireGuard tunnels, and steering packets into those tunnels.

## [Kubernetes Without `kube-proxy`](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)

This guide explains how to provision a Kubernetes cluster without `kube-proxy`, and to use Cilium to fully replace it.

### [Performance Optimization](docs/performance.md)

## REFERENCE

- [Requirements](https://docs.cilium.io/en/stable/network/kubernetes/requirements/)
- [System Requirements](https://docs.cilium.io/en/stable/operations/system_requirements/)
- [Kubernetes Compatibility](https://docs.cilium.io/en/stable/network/kubernetes/compatibility/)
- [Docs](https://docs.cilium.io/en/stable)
- [Calico vs. Cilium: 9 Key Differences and How to Choose](https://www.tigera.io/learn/guides/cilium-vs-calico/)
- [Terminology](https://docs.cilium.io/en/stable/gettingstarted/terminology/)
- [Migrating a cluster to Cilium](https://docs.cilium.io/en/stable/installation/k8s-install-migration/)
- [Installation using Kubespray](https://docs.cilium.io/en/stable/installation/k8s-install-kubespray/)
- [Routing](https://docs.cilium.io/en/stable/network/concepts/routing/)
- [IPAM](https://docs.cilium.io/en/stable/network/concepts/ipam/) and [Configuring IPAM Modes](https://docs.cilium.io/en/stable/network/kubernetes/ipam/)
- [Masquerading](https://docs.cilium.io/en/stable/network/concepts/masquerading/)
- [Kubernetes Configuration](https://docs.cilium.io/en/stable/network/kubernetes/configuration/) and [Configuration](https://docs.cilium.io/en/stable/configuration/)
- [Cilium Endpoint](https://docs.cilium.io/en/stable/network/kubernetes/ciliumendpoint/)
- [Cilium CiliumEndpointSlice](https://docs.cilium.io/en/stable/network/kubernetes/ciliumendpointslice/)
- [Troubleshooting](https://docs.cilium.io/en/stable/network/kubernetes/troubleshooting/)
- [Bandwidth Manager](https://docs.cilium.io/en/stable/network/kubernetes/bandwidth-manager/)
- [Local Redirect Policy](https://docs.cilium.io/en/stable/network/kubernetes/local-redirect-policy/)
- [eBPF Datapath](https://docs.cilium.io/en/stable/network/ebpf/)
- [Multi-cluster Networking - Mesh](https://docs.cilium.io/en/stable/network/clustermesh/)
- [External networking - Connect external VM to k8s](https://docs.cilium.io/en/stable/network/external-toc/)
- [Service Mesh](https://docs.cilium.io/en/stable/network/servicemesh/)
- [VXLAN Tunnel Endpoint (VTEP) Integration](https://docs.cilium.io/en/stable/network/vtep/)
- [Node IPAM LB](https://docs.cilium.io/en/stable/network/node-ipam/)
- [Use a Specific MAC Address for a Pod](https://docs.cilium.io/en/stable/network/pod-mac-address/)
- [Multicast](https://docs.cilium.io/en/stable/network/multicast/)
- [Security](https://docs.cilium.io/en/stable/security/)
- [Upgrade Guide](https://docs.cilium.io/en/stable/operations/upgrade/)
- [Performance & Scalability](https://docs.cilium.io/en/stable/operations/performance/)
- [Tuning](https://docs.cilium.io/en/stable/operations/performance/tuning/)
- [Command Reference](https://docs.cilium.io/en/stable/cmdref/)
- [Helm Reference](https://docs.cilium.io/en/stable/helm-reference/)
- [Cilium Network Policy Reference](https://docs.cilium.io/en/stable/security/policy/)
- [Metrics and Monitoring](https://docs.cilium.io/en/stable/observability/metrics/)
- [Gateway API Support](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/)
- [Examples](https://github.com/isovalent/cilium-up-and-running)
