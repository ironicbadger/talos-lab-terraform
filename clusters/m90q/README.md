# M90q: infrastructure-only cluster

Three Talos 1.13.8 / Kubernetes 1.36.2 VMs on the M90q Proxmox hosts. VM disks
are local; app storage is supplied by the existing Ceph cluster via CSI.
Provisioning inventory and scripts: `ironicbadger/infra`, `talos/m90q/`.

Flux tracks `codex/m90q-cluster`, reconciling this directory and its `platform/`
child. This intentionally does not include old app overlays, in-cluster Ceph,
monitoring stacks, dashboards or application ingress. No applications deployed.

Installed: Flux 2.9.4 (source, kustomize and Helm controllers), Cilium 1.20.1,
Ceph-CSI RBD 3.17.1. HelmRelease versions are pinned. The Ceph driver consumes
pool `k8s-m90q` on existing Ceph FSID `1f4d427c-77c5-47bc-8d63-b507b05afc47`.
Default class `ceph-rbd` retains data when a claim is deleted.

## Access

```sh
export TALOSCONFIG="$PWD/clusters/m90q/talos/clusterconfig/talosconfig"
export KUBECONFIG="$PWD/clusters/m90q/talos/clusterconfig/kubeconfig"
kubectl get nodes
flux get all -A
```

The LAN API VIP is `10.42.1.120`. Talos API endpoints are `.121`, `.122`, `.123`;
use individual nodes rather than the Kubernetes VIP for Talos administration.

## Tailscale access

Tailscale operator 1.102.3 is reconciled by Flux. Its authenticated API proxy is
`https://m90q-ts-operator.ktz.ts.net`. OAuth credentials are SOPS-encrypted in
`platform/tailscale-secret.sops.yaml` and mounted through `operator-oauth`.

This cluster intentionally uses only existing `tag:k8s-operator` and `tag:k8s`
tags. The per-cluster tag convention is unnecessary for this installation; no
additional tag ownership or tailnet policy changes were needed. Hostname identifies
this operator. Future proxy devices use `tag:k8s` by default.

```sh
# Optional: configure your normal kubectl context for tailnet access.
tailscale configure kubeconfig m90q-ts-operator.ktz.ts.net
kubectl get nodes
```

The build verified access using an isolated config at
`talos/clusterconfig/tailscale-kubeconfig`, preserving the default context.
API authorization uses existing tailnet impersonation grants. The operator is
one replica; the independent LAN VIP remains available during operator downtime.

## Recovery

Private generated configs are ignored under `talos/clusterconfig/`. New secrets
are encrypted to the existing infra age recipient and the dedicated key at
`~/.config/sops/age/m90q.txt`. Back up that key separately. Flux's decryption Secret
contains the dedicated key. `infra/talos/m90q/generate.py` regenerates configs from
the encrypted secrets. Retain etcd snapshots and app-consistent volume backups;
Git does not contain Kubernetes's complete live state. Initial encrypted etcd
snapshot: `~/.local/share/talos-backups/m90q/`. Recurring backups are not configured.

Do not delete the tracked branch while this cluster follows it. Changing to main
requires merging these files and explicitly changing the GitRepository ref.
