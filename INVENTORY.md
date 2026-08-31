# VSP 叢集實機盤點

> 全部來自實機 `kubectl` 查詢。環境 `vcf-m02`（nested lab），盤點日 **2026-08-31**。
> 版本/數量會隨環境不同，**重點是結構與各元件的角色**，不是這些數字本身。

## 平台基本

```
Kubernetes    v1.34.2+vmware.1
節點          4（1 control-plane + 3 worker）
CNI           Antrea
CSI/CPI       vsphere-csi / vsphere-cpi
StorageClass  vmsp-default   ← 唯一，且是 default
```

重新盤點指令：
```bash
kubectl get nodes -o wide
```
```bash
kubectl version -o json | head -20
```
```bash
kubectl get sc
```

---

## 節點

```
NAME                  STATUS   ROLES           VERSION            INTERNAL-IP
vcf-m02-vsp01-ftck5   Ready    control-plane   v1.34.2+vmware.1   10.0.0.228
vcf-m02-vsp01-zfp57   Ready    <none>          v1.34.2+vmware.1   10.0.0.229
vcf-m02-vsp01-q278k   Ready    <none>          v1.34.2+vmware.1   10.0.0.230
vcf-m02-vsp01-6kdzn   Ready    <none>          v1.34.2+vmware.1   10.0.0.231
```

> 節點名是**隨機後綴**（`vsp01-<5碼>`），重建就變；IP 也會重新分配。

---

## Namespace 與 workload

```bash
kubectl get ns
```
```bash
for ns in vcf-fleet-depot vcf-fleet-lcm vcf-sddc-lcm vidb-external salt salt-raas telemetry; do echo "--- $ns"; kubectl -n $ns get deploy,sts --no-headers; done
```

### `vcf-fleet-depot` —— depot / bundle
| Workload | 型別 |
|---|---|
| `depot-service` | Deployment（**3 containers**：含 `download-service`、`file-server`） |
| `distribution-service` | Deployment |

Service（皆 ClusterIP，無 Ingress）：
```
depot-service          7443, 8080, 8443, 9080
distribution-service   18080, 18443
```

### `vcf-fleet-lcm` —— Fleet 升級
| Workload | 型別 |
|---|---|
| `vcf-fleet-build-service-fleetbuild` | Deployment |
| `vcf-fleet-upgrade-service-fleetupgrade` | Deployment |
| `vcf-fleet-lcm-db` | **StatefulSet 3/3（Postgres）** |

### `vcf-sddc-lcm` —— SDDC 升級
| Workload | 型別 |
|---|---|
| `vcf-sddc-build-service-sddcbuild` | Deployment |
| `vcf-sddc-upgrade-service-sddcupgrade` | Deployment |
| `vcf-sddc-lcm-db` | **StatefulSet 3/3（Postgres）** |

### `vidb-external` —— VCF SSO / Identity Broker
| Workload | 型別 |
|---|---|
| `vidb-service` | Deployment |
| `vidb-postgres-instance` | **StatefulSet** |

### `salt` / `salt-raas` / `telemetry`
```
salt        salt-master, salt-minion
salt-raas   raas, redis, pgdatabase(sts)
telemetry   telemetry-acceptor
```

### `vmsp-platform` —— 平台底層（最重要的支撐層）

```
cert-manager, cert-manager-cainjector, cert-manager-webhook   ← 憑證
helm-controller                                                ← Flux（GitOps）
capi-controller-manager, capv-controller-manager,
capi-kubeadm-bootstrap/control-plane-controller-manager,
capi-ipam-in-cluster-controller-manager                        ← Cluster API（叢集生命週期）
cluster-autoscaler-clusterapi-cluster-autoscaler
argo-workflows-workflow-controller                             ← 工作流
kube-prometheus-stack-operator, -kube-state-metrics,
metrics-server, logging-operator                               ← 可觀測性
envoy-gateway, proxy-service                                   ← 閘道 / 代理
ndc-controller-manager, hooks-server-synthetic-checker
reloader-reloader (0/0)                                        ← 注意:副本數 0
```

> 🔴 **CA 信任的 Secret 放在這個 namespace**（trust-manager 的 trust-namespace）。
> 見 [TRUST-CA.md](TRUST-CA.md)。

---

## 關鍵 CRD

```bash
kubectl get crd | grep -E 'trust|helm|source|cert-manager|vmsp'
```

| CRD | 用途 |
|---|---|
| `bundles.trust.cert-manager.io` | **trust-manager 的信任 bundle** —— CA 從這裡合併出去 |
| `bundles.bundle.vmsp.vmware.com` | VCF 元件 bundle（每個元件一個 CR，狀態 `Successful`） |
| `certificates.cert-manager.io` / `clusterissuers.cert-manager.io` | cert-manager |
| `helmreleases` / `kustomizations` / `buckets.source.toolkit.fluxcd.io` | **Flux** —— 這就是「手改會被還原」的元凶 |
| `components.api.vmsp.vmware.com` | 平台元件 |
| `accessgrants.identity.vmsp.vmware.com` | 存取授權 |
| `backupconfigurations.backuprestore.vmsp.vmware.com` | 備份設定 |

看已安裝的 VCF 元件版本：
```bash
kubectl get bundles.bundle.vmsp.vmware.com -A
```

---

## 兩個「這個叢集不含」的東西

| 元件 | 在哪 |
|---|---|
| **VCF Operations** (`ops01`) | **獨立 appliance**，不在此叢集。重啟服務用 appliance 的 systemd |
| **Supervisor / VKS** | 另一套 workload 叢集，與 VSP 無關 |
