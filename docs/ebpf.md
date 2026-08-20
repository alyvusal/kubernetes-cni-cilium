# eBPF

[back](../README.md)

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
