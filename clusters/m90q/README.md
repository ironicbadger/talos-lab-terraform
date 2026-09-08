# M90q: infrastructure-only cluster

Three Talos 1.13.8 / Kubernetes 1.36.2 VMs on the M90q Proxmox hosts. VM disks
are local; app storage is supplied by the existing Ceph cluster via CSI.
Provisioning inventory and scripts: `ironicbadger/infra`, `talos/m90q/`.

Flux tracks `codex/m90q-cluster`, reconciling this directory and its `platform/`
child. This intentionally does not include old app overlays, in-cluster Ceph,
application workloads. Headlamp and Grafana provide cluster administration and monitoring.

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

## Headlamp

Headlamp 0.45.0: https://m90q-headlamp.ktz.ts.net (tailnet only).
Select **Sign in** to authenticate through the existing https://idp.ktz.ts.net.
The dedicated tsidp client is `m90q-headlamp`; its callback is
`https://m90q-headlamp.ktz.ts.net/oidc-callback`. Credentials are SOPS-encrypted in
`platform/headlamp-secret.sops.yaml`, mounted via an external Secret; PKCE is enabled.

All three Kubernetes API servers trust the issuer and client ID in `talos/oidc.json`.
The infra config generator includes these arguments. Apply regenerated Talos configs
one node at a time with `--mode=no-reboot`, waiting for the new API pod and `/readyz`
on that node before continuing. The VIP can briefly refuse connections when its
owner's API server restarts; other API nodes and KubePrism remain available.

RBAC maps `alexktz@gmail.com` to cluster-admin. Headlamp's service account has no
cluster-admin binding; access uses the logged-in user's token. Group claims are
prefixed `oidc:` to keep them distinct from Kubernetes built-in groups.

Verified HTTPS, the full OIDC redirect/token exchange, and an authenticated request
through Headlamp returning all three Kubernetes nodes. No persistent volume needed.

## Grafana and metrics

Open https://m90q-grafana.ktz.ts.net and sign in with Tailscale. The home dashboard
is **M90q / Cluster overview**, with RAM/CPU usage, scheduling requests, namespace
usage, PVC utilisation and restarts. The bundled Kubernetes dashboards provide
pod, namespace, kubelet and API server drilldowns. Node capacity is the Talos VM
capacity (8 GiB each), not the full Proxmox hosts.

`platform/monitoring-release.json` pins kube-prometheus-stack 90.0.0. Prometheus
keeps up to seven days / 8 GB of metrics on a 10 GiB Ceph RBD PVC; Grafana has a
1 GiB PVC. Both use the existing Retain storage class. The dashboard is managed
in Git via `platform/grafana-dashboard.json`; persist edits there.

Grafana uses a separate `m90q-grafana` tsidp client. Only `alexktz@gmail.com` is
mapped to Admin; other identities are denied. OAuth and recovery admin credentials
are SOPS-encrypted in `platform/grafana-secret.sops.yaml`.

Scrapes cover nodes, kubelet/cAdvisor, kube-state-metrics and the Kubernetes API.
The stack does not expose Talos's local-only etcd, scheduler or controller-manager
metrics. kube-proxy is absent. Their scrapes and associated rules are disabled.
Alertmanager is disabled; this installation does not send notifications.

## Custom domain ingress

`*.m90q.ktz.me` resolves to **10.42.0.54**, an independent Keepalived VIP on
core-pi5/core-zima. Core HAProxy forwards TCP to Traefik NodePorts 30080/30443;
Traefik runs two replicas on different nodes and terminates TLS. Cert-manager
renews the Let's Encrypt wildcard via Cloudflare DNS-01. All resources are
managed by platform and ingress Flux Kustomizations. See infra's
`talos/m90q/ingress/README.md` for core deployment, DNS and failover details.

New app Ingresses use `ingressClassName: traefik`, a host such as
`smokeping.m90q.ktz.me`, and `spec.tls.hosts` with that hostname. Omit secretName
to use Traefik's default wildcard certificate in namespace traefik. No per-app
certificate or NodePort needed. Reachability is private LAN/Tailscale subnet
routing; custom DNS names do not themselves provide user authentication.

The staged SmokePing draft now uses this route and remains at zero replicas.
Its files and root apps wiring have not been pushed as part of ingress setup.

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
