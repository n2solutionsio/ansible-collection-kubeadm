# Changelog

## 0.2.0

- Assert each requested package was staged by name, rather than counting `.deb`
  files. A count passes even when dependencies alone satisfy it and a requested
  package is silently missing.
- Report the staged filenames instead of a count, so `cri-tools`,
  `kubernetes-cni` and friends are visible in the run output.
- Pre-pull the pause image into containerd's `k8s.io` namespace
  (`node_prep_prepull_sandbox_image`), removing one network dependency from
  `kubeadm init`.
- Mount `bpffs` at `/sys/fs/bpf` and persist it (`node_prep_mount_bpffs`).
- Assert cgroup v2 (`node_prep_require_cgroup_v2`).
- New `node_prep_prepull_images` for pre-staging arbitrary images — CNI-agnostic
  and empty by default.

The eBPF groundwork makes Cilium or Calico's eBPF dataplane possible without
choosing one. CNI deployment itself remains out of scope: it is a Kubernetes
workload and cannot exist before the cluster does.

## 0.1.0

Initial release.

- `node_prep` role: prepares a Debian-family host to become a kubeadm node.
  Disables swap, loads and persists the required kernel modules and sysctls,
  installs and configures containerd with the systemd cgroup driver, and stages
  the Kubernetes packages **without installing them**.
- `prepare_nodes` playbook as a manual entry point.
- Defaults to Kubernetes 1.33.
