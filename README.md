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
- [BGP](https://docs.cilium.io/en/stable/network/bgp-toc/)
- [eBPF Datapath](https://docs.cilium.io/en/stable/network/ebpf/)
- [Multi-cluster Networking - Mesh](https://docs.cilium.io/en/stable/network/clustermesh/)
- [External networking - Connect external VM to k8s](https://docs.cilium.io/en/stable/network/external-toc/)
- [Service Mesh](https://docs.cilium.io/en/stable/network/servicemesh/)
- [VXLAN Tunnel Endpoint (VTEP) Integration](https://docs.cilium.io/en/stable/network/vtep/)
- [L2 Announcements / L2 Aware LB](https://docs.cilium.io/en/stable/network/l2-announcements/)
- [Node IPAM LB](https://docs.cilium.io/en/stable/network/node-ipam/)
- [Use a Specific MAC Address for a Pod](https://docs.cilium.io/en/stable/network/pod-mac-address/)
- [Multicast](https://docs.cilium.io/en/stable/network/multicast/)
- [Security](https://docs.cilium.io/en/stable/security/)
- [Network Observability with Hubble](https://docs.cilium.io/en/stable/observability/hubble/)
- [Upgrade Guide](https://docs.cilium.io/en/stable/operations/upgrade/)
- [Performance & Scalability](https://docs.cilium.io/en/stable/operations/performance/)
- [Tuning](https://docs.cilium.io/en/stable/operations/performance/tuning/)
- [Command Reference](https://docs.cilium.io/en/stable/cmdref/)
- [Helm Reference](https://docs.cilium.io/en/stable/helm-reference/)
- [Cilium Network Policy Reference](https://docs.cilium.io/en/stable/security/policy/)
- [Hubble Observability](https://docs.cilium.io/en/stable/observability/hubble/)
- [Gateway API Support](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/)
- [Examples](https://github.com/isovalent/cilium-up-and-running)
