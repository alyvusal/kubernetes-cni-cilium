# [Policy](https://docs.cilium.io/en/stable/security/policy/)

[back](../README.md)

## 🧠 CiliumNetworkPolicy (CNP / CCNP)

Cilium extends the basic Kubernetes NetworkPolicy model by operating at **Layer 3–7**, providing **more granular control**, **observability**, and **explicit deny** capabilities.
CiliumNetworkPolicies (CNPs) leverage **eBPF** for efficient enforcement directly in the Linux kernel.

### 🔹 Key Concepts and Enhancements

- **Policy Types**
  - `CiliumNetworkPolicy` → Namespace-scoped
  - `CiliumClusterwideNetworkPolicy` → Cluster-wide (applies across namespaces)

- **L3–L7 Aware**
  Cilium adds **application-layer (L7)** awareness, supporting filtering based on:
  - HTTP methods, paths, headers
  - DNS names (`toFQDNs`)
  - Kafka topics, gRPC services, etc.

- **Explicit Deny Rules**
  Supports both **allow** and **deny** rules (`ingressDeny`, `egressDeny`).
  Deny rules have **higher precedence** than allow rules.

- **Advanced Match Options**
  Match traffic by:
  - Pod, namespace, or service labels
  - CIDRs, IP sets
  - FQDNs, ports, protocols
  - Identities (Cilium internal label system)

- **Layer 7 Proxying**
  Uses **Envoy** as an embedded L7 proxy to enforce HTTP/DNS/Kafka policies.

- **Visibility and Observability**
  Integrated with **Hubble** — offering flow visibility, metrics, and tracing of policy decisions.

- **Policy Additivity with Deny Semantics**
  Policies are additive, but **deny rules override allows**.
  Order matters only in that deny is always evaluated first.

- **Clusterwide Scoping**
  Cilium can define cluster-wide network security with **CCNP**, something not possible with standard NetworkPolicy.

- **CIDR Sets and Identity Awareness**
  Supports rich CIDR filtering via `fromCIDRSet` / `toCIDRSet` and **identity-based** rules for workload-aware networking.

### 📘 Summary

> **CiliumNetworkPolicy = L3–L7 aware, supports allow + explicit deny, clusterwide scope, and deep observability via eBPF and Hubble.**

---

## 🔐 [**Kubernetes NetworkPolicy**](https://github.com/alyvusal/kubernetes/blob/main/network-policy/README.md) vs Cilium Network Policies

Kubernetes **NetworkPolicies** and **CiliumNetworkPolicies** define how Pods can communicate with each other and with external endpoints.
While Kubernetes provides **L3/L4 isolation**, **Cilium** extends this to **L7**, enabling deep visibility, explicit deny rules, and advanced identity-based filtering.

⚖️ See in detail [Comparison of NetworkPolicy & CiliumNetworkPolicy](https://github.com/alyvusal/kubernetes/blob/main/network-policy/README.md#-kubernetes-network-policies-vs-cilium-network-policies)

---

Unlike standard Kubernetes network policies, Cilium network policies can apply at
up to layer 7.

Sample app

```bash
kubectl apply -f examples/apps.yaml
```

## [Network Policy Editor](https://editor.networkpolicy.io/)

## Structure

- [Endpoint-based](https://docs.cilium.io/en/stable/security/policy/language/#endpoints-based): can define connectivity rules based on pod labels.
- [Service-based](https://docs.cilium.io/en/stable/security/policy/language/#services-based): use Kubernetes service endpoints to define connectivity rules.
- [Entity-based](https://docs.cilium.io/en/stable/security/policy/language/#entities-based): categorizing remote peers without knowing their IP addresses.
- [IP/CIDR-based](https://docs.cilium.io/en/stable/security/policy/language/#cidr-based): define connectivity rules for external services using hardcoded IP addresses or subnets.
- [DNS-based](https://docs.cilium.io/en/stable/security/policy/language/#dns-based): can define connectivity rules based on DNS names resolved to IP addresses.

## Syntax

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
