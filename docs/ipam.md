# [IPAM](https://docs.cilium.io/en/stable/network/concepts/ipam/)

[back](../README.md)

## Modes

- `kubernetes`: (Kubernetes Host Scope) Uses PodCIDRs assigned to each node by Kubernetes.
  Simple but inflexible.
  - Cannot allocate multiple CIDRs to the cluster instead of just one
  - You cannot configure the size of the PodCIDRs allocated to each node, only globally for the cluster.
  - You cannot add CIDRs to or remove them from the cluster or individual nodes.

  ```yaml
  ipam:
    mode: kubernetes
  ```

- `cluster-pool`: (Cluster Scope) Cilium manages PodCIDR allocation via the operator.
  The default mode.
  - Can allocate multiple CIDRs to the cluster instead of just one

  ```yaml
  ipam:
    mode: cluster-pool
    operator:
      clusterPoolIPv4MaskSize: 29
      clusterPoolIPv4PodCIDRList:
        - 10.0.42.0/28
        - 10.0.84.0/28
  routingMode: tunnel
  tunnelProtocol: vxlan
  ```

  ```bash
  kubectl get ciliumpodippools.cilium.io default -o yaml
  kubectl get ciliumnode kind-worker -o yaml | yq '.spec.ipam'

  # Deploy and scale a deployment check assigned ips
  kubectl apply -f examples/multi-pool/nginx-deployment-10-replicas.yaml
  ```

- `multi-pool`: Allows pods to be assigned IPs from multiple IP pools based on namespaces or
  annotations. Ideal for multitenant setups.
  - Multi-Pool mode functions similarly to Cluster Scope IPAM mode.
  - One advantage it has over other IPAM modes, however, is that it gives you more control over how IP addresses are assigned to pods. You can force a pod—or all pods in a particular namespace—to receive IPs from a particular range

  ```yaml
  ipam:
    mode: multi-pool
    operator:
      autoCreateCiliumPodIPPools:
        default:
          ipv4:
            cidrs: ["10.10.0.0/16"]
            maskSize: 27
        other:
          ipv4:
            cidrs:
              - 10.20.0.0/16
            maskSize: 24

  routingMode: native
  endpointRoutes:
    enabled: true

  autoDirectNodeRoutes: true
  ipv4NativeRoutingCIDR: 10.0.0.0/11
  bpf:
    masquerade: true
    hostLegacyRouting: true
  ```

  Multi-pool gives each node many small prefixes from different pools, so a single node CIDR and VXLAN tunnel are not enough.
  These native-routing settings make those prefixes reachable and keep tenant IPs visible on the wire:
  - The `autoCreateCiliumPodIPPools` option configures Cilium to create a default pool of IP addresses at start-up and to assign prefixes to nodes based on the `ipam.operator.autoCreateCiliumPodIPPools.ipv4.maskSize` value.
  Once Cilium is installed, verify that a default pool has been created:
  - `routingMode: native` forwards packets with the real pod IP instead of encapsulating them. That is what lets egress traffic show whether it came from ACME (`10.20.x`) or Foobar (`10.30.x`).
  - `endpointRoutes.enabled` installs a `/32` host route per pod so the kernel can deliver traffic to the correct veth no matter which pool the IP came from. Without it, Cilium only has one route per node CIDR toward `cilium_host`, which does not work with many disjoint `/27`s.
  - `autoDirectNodeRoutes` installs routes to other nodes' allocated prefixes on a shared L2 network (`10.20.0.32/27 via <node-B-IP>`). Prefixes appear and disappear as pods scale; without this (or BGP) cross-node traffic to non-default pools blackholes.
  - `ipv4NativeRoutingCIDR` is the supernet Cilium uses to decide "in-cluster, do not SNAT" vs "external, masquerade". It must cover every pool (`10.10.0.0/16`, `10.20.0.0/16`, `10.30.0.0/16`) but must not swallow the k3s service CIDR `10.43.0.0/16`, which is why `10.0.0.0/11` is set here instead of `/8`.
  - `bpf.masquerade` is required in multi-pool. iptables masquerade panics unless `egressMasqueradeInterfaces` is set; eBPF masquerade does not need that.
  - `bpf.hostLegacyRouting` is needed on k3d/kind. BPF host routing sends pod-to-API traffic (`10.43.0.1:443`) out `eth0` instead of delivering it to the local kube-apiserver, so CoreDNS never becomes ready.

  This is handy for multitenant environments. For example, suppose you have two tenants, ACME Corp. and Foobar Inc. Namespaces are often used in Kubernetes to differentiate tenants.
  While the IPAM modes we looked at previously are not namespace-aware, with Multi-Pool IPAM each namespace can draw pod IPs from its own defined address pool.
  This capability is particularly useful when traffic exits the cluster, as it helps engineers understand which tenant the traffic originated from

  For per node pool configuration, see [Per-Node Default Pool](https://docs.cilium.io/en/stable/network/concepts/ipam/multi-pool/#per-node-default-pool)

  ```yaml
  apiVersion: cilium.io/v2
  kind: CiliumNodeConfig
  metadata:
    name: ip-pool-dc1
    namespace: kube-system
  spec:
    defaults:
      ipam-default-ip-pool: foobar-pool
    nodeSelector:
      matchLabels:
        # topology.kubernetes.io/zone: dc1
        topology.kubernetes.io/hostname: k3d-k3s-default-agent-0
    ```

  To ensure that all workloads in a particular namespace receive an IP address from a particular IP pool, you can annotate the namespace

  ```bash
  kubectl get ciliumpodippools.cilium.io default -o yaml
  kubectl get ciliumnode k3d-k3s-default-agent-0 -o yaml | yq '.spec.ipam'

  kubectl apply -f examples/multi-pool/namespaces.yaml
  kubectl apply -f examples/multi-pool/acme-pool.yaml
  kubectl apply -f examples/multi-pool/foobar-pool.yaml

  kubectl annotate ns acme-corp ipam.cilium.io/ip-pool=acme-pool
  kubectl annotate ns acme-corp ipam.cilium.io/require-pool-match="true"
  kubectl annotate ns foobar-inc ipam.cilium.io/ip-pool=foobar-pool
  kubectl annotate ns foobar-inc ipam.cilium.io/require-pool-match="true"

  kubectl apply -f examples/multi-pool/nginx-deployment-10-replicas.yaml -n acme-corp
  kubectl apply -f examples/multi-pool/nginx-deployment-10-replicas.yaml -n foobar-inc
  kubectl get pods -o wide -n acme-corp
  kubectl get pods -o wide -n foobar-inc

  # Check the CiliumNode to see which PodCIDRs are assigned to each node
  kubectl get ciliumnode k3d-k3s-default-agent-0 -o yaml | yq .spec.ipam
  ```

  If you need more IPs, Cilium dynamically assigns additional subnets to the node.
  For example, let's scale up the ACME deployment from 10 replicas to 40:

  ```bash
  kubectl scale -n acme-corp deployment nginx-deployment --replicas=40

  kubectl get ciliumnode k3d-k3s-default-agent-0 -o yaml | yq '.spec.ipam'
  ```

  Note the needed field in the following output. Since the number of required IPs significantly increased, another `/27` subnet from the cluster-wide `10.20.0.0/16` was allocated to the node:

  ```bash
  $ kubectl get ciliumnode k3d-k3s-default-agent-0 -o yaml | yq '.spec.ipam'
  pools:
    allocated:
      - cidrs:
          - 10.20.0.0/27
          - 10.20.0.32/27
        pool: acme-pool
      - cidrs:
          - 10.10.0.32/27
        pool: default
      - cidrs:
          - 10.30.0.0/27
        pool: foobar-pool
    requested:
      - needed:
          ipv4-addrs: 40
        pool: acme-pool
      - needed:
          ipv4-addrs: 16
        pool: default
      - needed:
          ipv4-addrs: 10
        pool: foobar-pool
  ```

- `eni`: (ENI IPAM) Exclusive to Amazon Elastic Kubernetes Service (EKS) environments.
  Allocates pod IPs from from secondary IP addresses on AWS elastic network interfaces (ENIs).
  - The number of IPs that can be allocated per node is limited by the instance type’s ENI and secondary IP capacity (e.g., an m5.large instance supports only 3 ENIs with 10 IPs each). This restricts the number of pods a node can run (with an m5.large instance able to support just 30 pods—far below the 110 pods per node limit Kubernetes imposes by default).
  - Cilium supports [AWS prefix delegation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-prefix-eni.html), a feature that assigns entire CIDR blocks (e.g., a `/28`) to each ENI. This significantly increases per-node pod density, without requiring you to scale the cluster horizontally or vertically.

  Sample EKS configuration to use with `eksctl`:

  ```yaml
  apiVersion: eksctl.io/v1alpha5
  kind: ClusterConfig

  metadata:
    name: cuar
    region: eu-west-1

  managedNodeGroups:
  - name: ng-1
    desiredCapacity: 2
    privateNetworking: true
    taints:
    - key: "node.cilium.io/agent-not-ready"
      value: "true"
      effect: "NoExecute"
  ```

  You can also see the IPs being populated on the CiliumNode resource:

  ```bash
  kubectl get cn ip-192-168-132-54.eu-west-1.compute.internal -o yaml
  ```

## Reference

- [Configuring IPAM Modes](https://docs.cilium.io/en/stable/network/kubernetes/ipam/)
