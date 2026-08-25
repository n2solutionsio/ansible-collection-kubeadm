# Changelog

## 0.3.0

First release exercised against a real node, which found two things molecule
could not.

- **containerd was left unusable by kubelet.** The `containerd.io` package ships
  a stub config -- a licence header and `disabled_plugins = ["cri"]` -- rather
  than a full default. Generation was skipped whenever the file merely existed,
  so CRI stayed disabled and `kubeadm init` would have failed with "container
  runtime is not running". The stub is now detected by the absence of
  `SystemdCgroup` and the default generated over it. Verification reads the file
  back and asserts both `SystemdCgroup = true` and that `cri` is not disabled,
  because a `replace` whose regexp matches nothing reports ok, not failed.
- **Dependencies do not resolve themselves.** `kubeadm` declares no `Depends`
  at all, and `kubelet` lists only iptables, kubernetes-cni, mount, util-linux
  and libc6. `cri-tools`, `conntrack` and `socat` were therefore neither
  installed nor staged. `conntrack` is a `kubeadm init` preflight requirement,
  so a manual install would have failed. `cri-tools` is now staged with the
  other Kubernetes packages; `conntrack` and `socat` are installed as OS
  prerequisites.
- Install notes no longer carry a timestamp, which made every run report
  changed.

Verified on a real node: idempotent second run (`changed=0`), Kubernetes
packages staged and confirmed *not* installed, containerd active with CRI
enabled, bpffs mounted, pause image pre-pulled, swap off.

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
