# 疑難排解

## 存取 / kubectl

| 症狀 | 原因 | 解法 |
|---|---|---|
| `kubectl` connection refused、找不到 `/etc/kubernetes/admin.conf` | 你連到 **worker** 節點了 | `kubectl get nodes -o wide` 找 `ROLES=control-plane` 的 `INTERNAL-IP`，改連那台 |
| `sudo: a password is required` | 節點**沒有** passwordless sudo | `echo '<VSP_PW>' \| sudo -S -p '' <cmd>` |
| plink 報 `POTENTIAL SECURITY BREACH` | IP pool 重用、host key 換了 | `-hostkey "SHA256:<新指紋>"`；Linux 用 `-o StrictHostKeyChecking=no` |
| 記死的 control-plane IP 突然連不上 | **重建後 IP↔角色會變** | 一律照 `ROLES` 欄認，別記 IP |
| `kubectl exec … -- awk` 報 `command not found` | 容器是精簡映像，**沒有 awk 等工具** | 改用 shell built-in，或換到有工具的容器（`-c file-server`） |
| `logs` 報要指定 container | `depot-service` 是 **3-container** pod | 加 `-c download-service` 或 `-c file-server` |

---

## 設定改了又變回去

| 症狀 | 原因 | 解法 |
|---|---|---|
| 改 `platform-trust` configmap 後被還原 | **trust-manager** 會覆蓋 target configmap | 改建「帶 label 的 Secret」→ [TRUST-CA.md](TRUST-CA.md) |
| `kubectl patch depot-service-secret` 後被還原 | 由 `depot-service-manager` + **Helm/Flux** 管 | 只能走 **VCF Operations UI** → [DEPOT-SERVICE.md](DEPOT-SERVICE.md) |
| 任何 Flux 管的資源手改被還原 | GitOps reconcile 是設計行為 | 找正確入口（UI / CR），不要跟 operator 對抗 |

> 判斷某資源是不是被管：
> ```bash
> kubectl -n <ns> get <res> <name> -o jsonpath='{.metadata.labels}'
> ```
> 看到 `helm.toolkit.fluxcd.io/*` 或 `app.kubernetes.io/managed-by` 就是。

---

## 憑證 / TLS

| 症狀 | 原因 | 解法 |
|---|---|---|
| `tls_verify=1`、`HTTP=000` | CA 不在 bundle | 依 [TRUST-CA.md](TRUST-CA.md) 匯入 |
| `HTTP=401`，但 `tls_verify=0` | **TLS 已通**，只是缺帳密 | **不是** CA 問題，去補 basic auth |
| `HTTP=404` | 路徑錯 | 與憑證無關；例如 nginx `location /PROD/` 就一定要帶 `/PROD/` |
| 匯入後憑證數沒 +1 | 同一張 CA **已經在 bundle 內** | `kubectl -n vmsp-platform get secret -l trust.vmsp.vmware.com/bundle=platform-trust` 確認 |
| 匯好了但服務還是不信任 | pod 尚未拿到新 bundle（約 1 分鐘），或服務有快取 | 等，或 `rollout restart deploy depot-service distribution-service` |

---

## Depot 下載

| 症狀 | 原因 | 解法 |
|---|---|---|
| CA 匯好了，Fleet 還是抓不到東西 | **depot 設定指到錯的位址** —— 匯 CA 只解決「信任」，不等於指向 | UI 改 Software Depot URL |
| UI 的 Authentication 開關打不開 | URL 還是 `http://` | 先改成 `https://`（只有 HTTPS 支援認證） |
| depot access.log **完全沒有新紀錄** | Fleet 根本沒打到這台 | URL 指錯（例如指到不存在的埠） |
| access.log 只有 `401`、第 3 欄是 `-` | 帳密沒設或沒帶 | UI 補 Username / Password |
| 分不清是 Fleet 還是自己在抓 | — | 看 UA：`Apache-HttpClient/…(Java/…)` + 來源是 VSP 節點 IP 才是 Fleet |

---

## Online（Connected）模式

| 症狀 | 原因 | 解法 |
|---|---|---|
| 切 Connected 後一直失敗，`activationCode` 為空 | **沒有啟用碼** | 用 `machineId` 到 `vcf.broadcom.com` 換 activation code |
| 打 `dl.broadcom.com` 根路徑得 403/500 就以為斷網 | **根路徑本來就這樣** | 測真實 catalog 路徑；看 header 有沒有 `Server: cloudflare` |
| VALIDATE 過但下載 403 | 該帳號**沒有 binary 下載 entitlement** | 換一組有權限的帳號產 code。**與指令無關** |
| 之前可以現在不行 | activation code **有時效** | 用**同一顆 machineId** 重產 code |
| 重建 fleet 後舊 code 失效 | `machineId` 變了 | 用**新的** machineId 重產 |

> 🔴 **不要重產 `machineId`** —— ID 一改，既有 code 立刻作廢。

---

## 重啟相關

| 症狀 | 原因 | 解法 |
|---|---|---|
| 重啟後任務中斷 | 有 in-flight 的 bring-up / upgrade | 重啟前先確認 UI **Build → Tasks** 沒有進行中工作 |
| 重啟 VIDB 後登不進 vCenter / NSX | 那些走 **VCF SSO federated 登入** | 等 `vidb-service` Ready 再試 |
| 重啟 StatefulSet 後 DB 異常 | 那些是 **Postgres 叢集** | 不要 `rollout restart` sts；必要時先備份、逐台觀察 |
| 重啟了但問題沒好 | 問題根本不在快取 | 見上面「設定改了又變回去」與「Depot 下載」 |

---

## 快速健檢

```bash
kubectl get nodes -o wide
```
```bash
kubectl get pod -A --field-selector=status.phase!=Running --no-headers
```
```bash
for ns in vcf-fleet-depot vcf-fleet-lcm vcf-sddc-lcm vidb-external; do echo "--- $ns"; kubectl -n $ns get deploy,sts --no-headers; done
```
```bash
kubectl get bundles.trust.cert-manager.io platform-trust
```
```bash
kubectl -n vcf-fleet-depot logs deploy/depot-service -c download-service --tail=50 | grep -iE 'error|fail|denied|unauthor|forbidden'
```
