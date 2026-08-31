# VSP 內部：VCF 9.1 Fleet 跑在什麼樣的 K8s 上

> VCF 9.1 的 **Fleet Manager 不是一台可以 SSH 的 appliance** ——
> 它是跑在 **VSP（VCF Services Platform）K8s 叢集**上的一組服務。
> 這個 repo 記錄那個叢集**實際長什麼樣**、怎麼進去、怎麼操作、以及踩過的雷。
>
> 資料來源：實機 `kubectl` 查詢，不是文件抄寫。
> 環境：`vcf-m02`（nested lab, home.lab）　最後盤點：2026-08-31
> 密碼一律以 `<..._PW>` 佔位符呈現。

## 文件索引

| 文件 | 內容 |
|---|---|
| [ACCESS.md](ACCESS.md) | 怎麼登入、拿 `kubectl`（含 Windows plink 用法與 host key 雷） |
| [INVENTORY.md](INVENTORY.md) | 叢集實際盤點：節點、namespace、workload、CRD、儲存 |
| [RESTART.md](RESTART.md) | 重啟各服務的指令，與**不能重啟什麼** |
| [TRUST-CA.md](TRUST-CA.md) | `trust-manager` 憑證信任機制（9.1 換掉了舊的 keytool 做法） |
| [DEPOT-SERVICE.md](DEPOT-SERVICE.md) | depot-service 設定存哪、為什麼只能走 UI 改 |
| [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | 症狀 → 原因 → 解法 |

---

## 30 秒版

```bash
ssh -o StrictHostKeyChecking=no vmware-system-user@<VSP_NODE_IP>   # 10.0.0.227-231
sudo -i
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes -o wide          # 認 ROLES=control-plane 那台
```

🔴 **只有 control-plane 節點有 `/etc/kubernetes/admin.conf`**。
worker 節點 SSH 進得去但沒有 kubeconfig，`kubectl` 會報連線失敗。
**哪個 IP 是 control-plane 每次重建都會變**，一定要用 `kubectl get nodes -o wide` 的 `ROLES` 欄認，別記死。

---

## 叢集一覽（實測）

```
Kubernetes  v1.34.2+vmware.1
節點        4 台（1 control-plane + 3 worker）
CNI         Antrea
儲存        vsphere-csi（StorageClass: vmsp-default，唯一且為預設）
GitOps      Flux（helm-controller / source-controller）
憑證        cert-manager + trust-manager
叢集生命週期 Cluster API（capi / capv / capi-kubeadm-*）
工作流       Argo Workflows
可觀測性     kube-prometheus-stack、logging-operator、metrics-server
閘道        envoy-gateway
```

### Namespace 對照

| Namespace | 裝什麼 | 你多半會動這裡 |
|---|---|---|
| `vcf-fleet-depot` | `depot-service`、`distribution-service` | ✅ depot / bundle 下載 |
| `vcf-fleet-lcm` | `vcf-fleet-build-service-fleetbuild`、`vcf-fleet-upgrade-service-fleetupgrade`、`vcf-fleet-lcm-db`(sts 3/3) | ✅ fleet 升級 |
| `vcf-sddc-lcm` | `vcf-sddc-build-service-sddcbuild`、`vcf-sddc-upgrade-service-sddcupgrade`、`vcf-sddc-lcm-db`(sts 3/3) | ✅ SDDC 升級 |
| `vidb-external` | `vidb-service`、`vidb-postgres-instance`(sts) | ✅ VCF SSO / Identity Broker |
| `vmsp-platform` | **平台底層**：cert-manager、trust-manager、Flux、CAPI、Argo、Prometheus… | ⚠️ CA 信任的 Secret 放這 |
| `vmsp-policies` | 平台政策 | 通常不動 |
| `salt` / `salt-raas` | `salt-master`、`salt-minion` / `raas`、`redis`、`pgdatabase`(sts) | 少動 |
| `telemetry` | `telemetry-acceptor` | 少動 |
| `kube-system` | Antrea、CoreDNS、kube-proxy、vsphere-cpi/csi | 少動 |

---

## 最常見的三個需求

**1. 讓 Fleet 信任自簽憑證（離線 depot 的 CA）**
→ [TRUST-CA.md](TRUST-CA.md)。
🔴 **不是** keytool，也**不能改 configmap**（`trust-manager` 會覆蓋回去）。

**2. 改 depot 設定**
→ [DEPOT-SERVICE.md](DEPOT-SERVICE.md)。
🔴 `depot-service-secret` 由 `depot-service-manager` + Helm/Flux 管，
**`kubectl patch` 會被 reconcile 還原**，只能走 VCF Operations UI。

**3. 重啟服務**
→ [RESTART.md](RESTART.md)。
🔴 **不要 `rollout restart` StatefulSet** —— 那些是 Postgres 叢集。

---

## 這個 repo 不涵蓋什麼

- **VCF Operations（`vcf-m02-ops01`）** 是獨立 appliance，**不在這個叢集裡**，重啟它的服務要用 appliance 上的 systemd，不是 `kubectl`。
- **Supervisor / VKS 的 K8s** 是另一套（workload 叢集），與 VSP 無關。
- 客戶專屬的 IP / 密碼 —— 一律佔位符。
