# Cilium (eBPF based networking, Observability, Security)

**Note on eBPF**:
Check if the necessary eBPF features are enabled and see also troubleshooting [guide](https://docs.cilium.io/en/latest/reference-guides/bpf/debug_and_test/):

```bash
sudo bpftool feature
```

- Embedded in the Kernel:
  - eBPF is not a standalone software or package; it is a subsystem built into the Linux kernel.
  - If your Linux distribution runs a kernel version of 4.1 or higher, the kernel includes support for eBPF.
- Progressive Enhancements:
  - Kernel versions up to 4.4 introduced the basic capabilities of eBPF.
  - Advanced features, such as support for tracing, networking, and security applications, require kernel versions 4.9, 4.14, 5.x, or later.
  - Features critical to tools like Cilium often depend on capabilities introduced in kernel versions 4.19 or later.
- Compatibility with Cilium:
  - Cilium uses eBPF extensively for networking, security policies, and observability.
  - Cilium recommends running kernel versions 5.3 or later to fully leverage its eBPF features.
  - Some distributions (e.g., Ubuntu, Red Hat) backport certain eBPF features into their kernels, allowing older kernels to support some newer eBPF functionality.
- User-Space Tools:
  - To interact with eBPF, you typically need user-space tools like `bpftool` or `bcc`.
  - These tools are not always installed by default but can be installed via package managers (e.g., `apt`, `yum`, etc.).

**While Hubble** provides excellent network-level observability, **Tetragon** takes security observability to the next level by providing kernel and process-level insights. Tetragon, another eBPF-powered tool under Cilium, can: Monitor process executions and file access. Detect and prevent unauthorized binaries from running.

## Install

### [Install with CLI](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)

```bash
# Install
cilium install
cilium status
```

### [Install with HELM](https://docs.cilium.io/en/stable/installation/k8s-install-helm/#installation-using-helm)

```bash
helm repo add cilium https://helm.cilium.io/

helm upgrade -i cilium cilium/cilium \
  --version 1.18.2 \
  -n kube-system

# after any change in helm
kubectl -n kube-system rollout restart deployment cilium-operator
kubectl -n kube-system rollout restart ds cilium
```

### Verify installation

```bash
# Validate connectivity in cluster with CLI
cilium connectivity test

# Validate connectivity in cluster with deployment
kubectl create ns cilium-test
kubectl apply -n cilium-test -f https://raw.githubusercontent.com/cilium/cilium/1.16.3/examples/kubernetes/connectivity-check/connectivity-check.yaml
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

### Demo App

Use samples from [Getting Started with the Star Wars Demo](https://docs.cilium.io/en/stable/gettingstarted/demo/)

- [Apply an L3/L4 Policy](https://docs.cilium.io/en/stable/gettingstarted/demo/#apply-an-l3-l4-policy)
- [Apply and Test HTTP-aware L7 Policy](https://docs.cilium.io/en/stable/gettingstarted/demo/#apply-and-test-http-aware-l7-policy)

## [CNI Chaining](https://docs.cilium.io/en/stable/installation/cni-chaining/)

CNI chaining allows to use Cilium in combination with other CNI plugins.

With Cilium CNI chaining, the base network connectivity and IP address management is managed by the non-Cilium CNI plugin, but Cilium attaches eBPF programs to the network devices created by the non-Cilium plugin to provide L3/L4 network visibility, policy enforcement and other advanced features.

## [Kubernetes Without `kube-proxy`](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)

This guide explains how to provision a Kubernetes cluster without `kube-proxy`, and to use Cilium to fully replace it. For simplicity, we will use `kubeadm` to bootstrap the cluster.

## [Egress Gateway](https://docs.cilium.io/en/stable/network/egress-gateway-toc/)

The egress gateway feature routes all IPv4 connections originating from pods and destined to specific cluster-external CIDRs through particular nodes, from now on called “gateway nodes”.

## [Hubble](https://docs.cilium.io/en/stable/observability/hubble/)

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
```

Troubleshoot

```bash
cilium status
kubectl -n kube-system exec ds/cilium -- cilium-dbg service list
```

## [Tetragon](https://tetragon.io/docs/)

```bash
helm upgrade -i tetragon cilium/tetragon \
  -n kube-system \
  --version 1.2.0 \
  --set tetragon.hostProcPath=/procHost

# test app
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.15.3/examples/minikube/http-sw-app.yaml
```

## Policy

Sample app

```bash
kubectl apply -f examples/apps.yaml
```

### [Network Policy Editor](https://editor.networkpolicy.io/)

[Editor](https://editor.networkpolicy.io/)

### Structure

- [Endpoint-based](https://docs.cilium.io/en/stable/security/policy/language/#endpoints-based): can define connectivity rules based on pod labels.
- [Service-based](https://docs.cilium.io/en/stable/security/policy/language/#services-based): use Kubernetes service endpoints to define connectivity rules.
- [Entity-based](https://docs.cilium.io/en/stable/security/policy/language/#entities-based): categorizing remote peers without knowing their IP addresses.
- [IP/CIDR-based](https://docs.cilium.io/en/stable/security/policy/language/#cidr-based): define connectivity rules for external services using hardcoded IP addresses or subnets.
- [DNS-based](https://docs.cilium.io/en/stable/security/policy/language/#dns-based): can define connectivity rules based on DNS names resolved to IP addresses.

Syntax

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: ...
  namespace: ...
spec:
  endpointSelector:
    matchLabels:
      app: hubble-ui
  ingress:
    - {}
  egress:
    - {}
```

Note that if the endpoint selector field is empty, the policy will be applied to all pods in the namespace.

**Default deny vs explicit deny (very important):**

- Default deny is the implicit effect of providing no allow rules for a given direction (e.g., `ingress: []` or omitting `ingress` entirely) — traffic not explicitly allowed is denied.
- Explicit deny uses `ingressDeny`/`egressDeny` sections to actively block traffic that would otherwise be allowed. In Cilium, explicit denies take precedence over allows (Deny > Allow).
- Allow-all in Cilium uses an empty rule object (e.g., `- {}`) which is a wildcard rule matching everything for that direction. This is different from an empty list `[]` which means deny-all by default.

**Empty {}:**

1. Empty endpointSelector: {}
An empty endpointSelector with {} means the policy applies to all endpoints within the namespace where the CiliumNetworkPolicy is defined. This acts as a wildcard selector for endpoints.
2. Empty ingress: - {} or egress: - {}
When an ingress or egress section contains an empty rule {} (represented as a list item - {}), it signifies a default deny for that direction of traffic for the endpoints selected by the policy.
Specifically, if ingress: - {} is present, all incoming traffic to the selected endpoints will be denied by default, unless explicitly allowed by other rules within the ingress section.
Similarly, if egress: - {} is present, all outgoing traffic from the selected endpoints will be denied by default, unless explicitly allowed by other rules within the egress section.

```bash
kubectl get cnp  # cnp is short for the CiliumNetworkPolicy
```

## Compare NetworkPolicy & CiliumNetworkPolicy

| Feature / Behavior | **Kubernetes NetworkPolicy** | **CiliumNetworkPolicy** |
|---------------------|-----------------------------|--------------------------|
| API Group | `networking.k8s.io/v1` | `cilium.io/v2` |
| CRD Type | `NetworkPolicy` | `CiliumNetworkPolicy` / `CiliumClusterwideNetworkPolicy` |
| Default behavior (no policy) | All traffic allowed | All traffic allowed |
| Policy type | Allow-only (implicit deny) | Allow + Explicit deny supported |
| Deny rules supported | ❌ No | ✅ Yes (`ingressDeny`, `egressDeny`) |
| Rule precedence | N/A (union of allows) | **Deny > Allow** |
| `ingress: []` meaning | Deny all ingress | Deny all ingress |
| `ingress: - {}` meaning | ❌ Invalid YAML or no effect | Allow all ingress (wildcard) |
| `ingress:` omitted | Deny all ingress | Deny all ingress |
| Egress control | Supported (since v1.8) | Fully supported + deny semantics |
| CIDR filtering | ✅ Basic (`ipBlock`) | ✅ Advanced (`fromCIDRSet`, `toCIDRSet`) |
| L4 (port) filtering | ✅ Yes | ✅ Yes |
| L7 (HTTP, DNS, Kafka, etc.) | ❌ No | ✅ Yes (via Envoy integration) |
| Namespace scope | Namespaced | Namespaced or Clusterwide (`CCNP`) |
| Policy merging behavior | Additive (union of allows) | Additive (union of allows + denies) |
| Order of evaluation | Irrelevant (all allows combined) | Deny → Allow (deny evaluated first) |
| Visibility / Metrics | Basic (via CNI logs) | Rich observability (Hubble, metrics, flow visibility) |
| Example default deny | `ingress: []`, `egress: []` | `ingress: []`, `egress: []` |
| Example allow all | ❌ Not possible via policy | `ingress: - {}`, `egress: - {}` |
| Advanced match (labels, CIDR, FQDN, etc.) | Limited | Extensive (label, CIDR, FQDN, service, identity, etc.) |

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
