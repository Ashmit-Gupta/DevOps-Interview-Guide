# How do you migrate an entire Kubernetes cluster from 1.34 to 1.37 if it has 400+ pods running and 40 nodes?

## Answer

You **cannot skip minors**. Path is **1.34 → 1.35 → 1.36 → 1.37**. Each hop: **control plane first**, then **workers**, staying inside the version-skew window (kubelet may lag the API server by **up to 3 minors**, but kube-apiserver / kube-controller-manager / kube-scheduler / kube-proxy should match). With **~400 pods / 40 nodes** this is a **rolling, drain-based** upgrade, not a big-bang rebuild — unless you deliberately blue/green a second cluster.

### 0. Decide the model

- **In-place rolling** (kubeadm / EKS / AKS / GKE version upgrade): keep the same cluster, drain node by node. Default for this size.
- **Blue/green / new cluster**: new 1.37 cluster, move workloads with GitOps + PV migration. Safer if you can spare capacity; more work for stateful apps.
- Managed Kubernetes still follows the same **one-minor** rule; the cloud does control-plane hops for you, you still own **node groups, addons, and PDBs**.

### 1. Pre-checks (do this once, before 1.34→1.35)

1. **Backup etcd** (self-managed) or confirm cloud snapshots / cluster backups.
2. Run **`kube-no-trouble` (kubent)** / deprecated-API scan against **1.35, 1.36, and 1.37**. Fix leftover `v1beta1` VolumeAttributesClass, ServiceCIDR, IPAddress, ClusterTrustBundle alpha, etc.
3. Confirm **CNI, CSI, CRI (containerd), ingress, metrics-server, operators** support **each** target version. Upgrade addons **before or with** the hop they require.
4. **cgroup v2** on every node (1.35+ kubelet fails on cgroup v1 by default).
5. **kube-proxy mode**: if `ipvs`, plan move to **nftables** (IPVS deprecated in 1.37).
6. **Static Pods** must not reference Secrets/ConfigMaps (blocked in 1.37).
7. Workload safety:
   - **PodDisruptionBudgets** on every critical Deployment/StatefulSet (`minAvailable` or `maxUnavailable` so 400 pods don’t evacuate at once).
   - **maxUnavailable** on Deployments; **podManagementPolicy** / `maxUnavailable` on StatefulSets.
   - Enough **spare CPU/mem** so 1–2 nodes can be drained (~10 pods/node average here — drain is cheap if PDBs are sane).
8. Maintenance window + **rollback**: keep previous component binaries / AMI / node image until the hop is proven.

### 2. Each minor hop (repeat 3 times)

**Control plane**

- kubeadm: `kubeadm upgrade plan` then `kubeadm upgrade apply v1.3x.y` on the first CP node; upgrade additional CP nodes; then `kubectl drain` is **not** how you upgrade CP if they are dedicated — upgrade kubelet/kubectl on CP nodes to the same minor.
- Managed: trigger **control plane version** only; wait until API is healthy (`kubectl get --raw=/readyz`, etcd, controllers).

**Workers (40 nodes, rolling)**

1. Cordon + drain **one node** (or a small batch of 2–3 if PDBs and spare capacity allow):

```bash
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --grace-period=120
```

2. Upgrade kubelet, kube-proxy, OS/runtime on that node; uncordon.
3. Watch:
   - `kubectl get pods -A | grep -v Running`
   - Deployments/StatefulSets ready counts
   - HPA, ingress, PVC attach/detach
   - node `Ready`, CNI, CSI
4. Only then drain the next node. **Do not drain 40 nodes in parallel.**

DaemonSets (CNI, kube-proxy, logging) roll with the node; they are why `--ignore-daemonsets` is used.

**After each minor is fully rolled**

- Upgrade kubectl on jump hosts.
- Re-run deprecated API scan for the **next** minor.
- Confirm storage version / encryption migrator if you use encryption-at-rest (built-in StorageVersionMigration is Stable in 1.37).

### 3. What “400 pods / 40 nodes” actually changes

| Risk | What you do |
| --- | --- |
| Eviction storm | PDBs + drain **1–3 nodes** at a time; never cordon the whole pool |
| StatefulSets / single-replica DBs | `maxUnavailable: 1`, wait for Ready before next drain; CSI driver already upgraded |
| EmptyDir / local data | `--delete-emptydir-data` **will wipe** it — call that out; those pods reschedule empty |
| Cluster autoscaler | Pause or pin during control-plane hop so it doesn’t mix old/new templates wrongly |
| Long drain | `--grace-period` + preStop hooks; 400 pods is fine if you drain ~10 at a time |
| Skew | Finish **all 40 kubelets** on 1.35 before starting 1.36 control plane |

### 4. 1.37-specific landmines on the last hop

- Clients/controllers must handle **API 429** (watch cache / APF).
- Drop **kube-dns** leftovers; CoreDNS only.
- Confirm **no IPVS** kube-proxy.
- Confirm **cgroup v2**.
- Prefer **nftables** kube-proxy going forward.

### 5. Rollback

- **During a hop**: restore etcd snapshot taken **before that hop** (self-managed) or revert the managed control-plane if the provider allows it **before** workers are mixed badly.
- **Do not** run a 1.37 API server against a 1.34 etcd snapshot and expect a clean skip — rollback is **per minor**, which is another reason you hop 1.34→1.35→1.36→1.37.

**Interview one-liner:** Version skew forbids a skip; treat it as three production rolling upgrades, control plane then 40 nodes with PDBs, addon compatibility, cgroup v2 and kube-proxy/API deprecations checked before the 1.37 hop.
