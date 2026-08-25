# Changelog

## 0.1.0

Initial release.

- `node_prep` role: prepares a Debian-family host to become a kubeadm node.
  Disables swap, loads and persists the required kernel modules and sysctls,
  installs and configures containerd with the systemd cgroup driver, and stages
  the Kubernetes packages **without installing them**.
- `prepare_nodes` playbook as a manual entry point.
- Defaults to Kubernetes 1.33.
