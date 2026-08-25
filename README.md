# n2solutions.kubeadm

Ansible collection that prepares Linux hosts to become kubeadm Kubernetes nodes.

It stops deliberately short of bootstrapping. Everything a node needs *before*
`kubeadm init` or `kubeadm join` is done; the Kubernetes packages are downloaded
but left uninstalled, so the operator runs the install and the bootstrap by hand.

That split exists because kubeadm bootstrap is the step worth watching — join
tokens expire, CA hashes have to match, and the first control plane has to be up
before any worker can join. Preparation is repeatable and boring, which is what
Ansible is good at. Bootstrap is neither.

## What it does

| | |
|---|---|
| Swap | Disabled at runtime, in `/etc/fstab`, and via systemd `.swap` units |
| Kernel modules | `overlay`, `br_netfilter` — loaded and persisted |
| Sysctls | `bridge-nf-call-iptables`, `bridge-nf-call-ip6tables`, `ip_forward` |
| Container runtime | containerd, `SystemdCgroup = true`, pinned `sandbox_image` |
| Kubernetes packages | Downloaded to `/opt/kubeadm-staging`, **not installed** |
| Notes | `INSTALL-NOTES.md` written to each node with the exact install command |

## What it deliberately does not do

- Install `kubelet`, `kubeadm`, or `kubectl`
- Run `kubeadm init` or `kubeadm join`
- Install a CNI
- Touch an existing cluster

## Requirements

- Ansible >= 2.15
- Debian-family, x86_64 (asserted at the start of the run)
- `ansible.posix`, `community.general`

## Install

```bash
ansible-galaxy collection install git+https://github.com/n2solutionsio/ansible-collection-kubeadm.git
```

Or via `requirements.yml`, which is what an AWX project should use:

```yaml
collections:
  - name: https://github.com/n2solutionsio/ansible-collection-kubeadm.git
    type: git
    version: main
  - name: ansible.posix
```

## Use

```bash
cp inventory/demo.ini.example inventory/demo.ini   # edit to taste
ansible-playbook -i inventory/demo.ini n2solutions.kubeadm.prepare_nodes
```

Target a subset:

```bash
ansible-playbook -i inventory/demo.ini n2solutions.kubeadm.prepare_nodes \
  -e target_hosts=kubeadm_control_plane
```

Then, on each node:

```bash
sudo apt-get install -y /opt/kubeadm-staging/*.deb
sudo apt-mark hold kubelet kubeadm kubectl
```

kubelet will restart-loop until a cluster is initialised or joined. That is
expected — it has no configuration yet.

## Variables

Full list in [`roles/node_prep/defaults/main.yml`](roles/node_prep/defaults/main.yml).
The ones you are most likely to change:

| Variable | Default | Notes |
|---|---|---|
| `node_prep_k8s_version` | `"1.33"` | Minor version. Selects the `pkgs.k8s.io` repo, so it is not cosmetic. |
| `node_prep_k8s_package_version` | `""` | Pin an exact version, e.g. `1.33.4-1.1`. Worth setting before a demo. |
| `node_prep_sandbox_image` | `registry.k8s.io/pause:3.10` | Must match `kubeadm config images list`. |
| `node_prep_staging_dir` | `/opt/kubeadm-staging` | Where the `.deb` files land. |
| `node_prep_containerd_version` | `""` | Pin containerd, e.g. `1.7.27-1`. |

Each stage can be toggled off: `node_prep_disable_swap`,
`node_prep_configure_kernel`, `node_prep_install_containerd`,
`node_prep_stage_packages`.

## Version selection

`pkgs.k8s.io` publishes a **separate repository per minor version**, so
`node_prep_k8s_version` changes where packages come from, not just which are
selected. Bumping a minor means the next run pulls from a different repo.

The default of `1.33` matches the k3s cluster this was built alongside, so a
single `kubectl` works against both (skew policy tolerates ±1 minor).

## Testing

```bash
cd roles/node_prep && molecule test
```

**A container is not a faithful stand-in for a kubeadm node**, and the scenario
does not pretend otherwise. `modprobe` cannot load host kernel modules, there is
no swap to disable, and containerd will not run meaningfully under another
container runtime. Those three sections are disabled in the converge rather than
asserted against a hollow pass.

What CI genuinely covers: base packages, `pkgs.k8s.io` repository wiring, the
download-without-install behaviour (verified by asserting `dpkg-query` reports
kubeadm and kubelet as *not* installed), and idempotence. The kernel, swap, and
containerd paths are exercised on real VMs.

## Licence

MIT
