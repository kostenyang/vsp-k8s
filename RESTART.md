# 重啟 Fleet 服務

> 前置：已登入 control-plane 並 `export KUBECONFIG=/etc/kubernetes/admin.conf`（見 [ACCESS.md](ACCESS.md)）。

## ⚠️ 先確認你真的需要重啟

大多數情況**不用**：

| 你改了什麼 | 會自己生效嗎 |
|---|---|
| UI 改 depot 設定 | ✅ 會 |
| 匯入 CA（trust-manager） | ✅ 會，約 **1 分鐘**（kubelet 同步 configmap） |
| 改了 Secret / ConfigMap 但服務有**快取** | ❌ 才需要重啟 |

而且**有進行中的作業時不要重啟** —— bring-up / upgrade / bundle 下載會中斷：

```bash
kubectl -n vcf-fleet-lcm get pod
```
```bash
kubectl -n vcf-sddc-lcm get pod
```
同時確認 VCF Operations UI 的 **Build → Tasks** 沒有 in-flight 任務。

---

## 1. Depot（最常用）

改完 depot 設定或憑證後：

```bash
kubectl -n vcf-fleet-depot rollout restart deploy depot-service distribution-service
```
```bash
kubectl -n vcf-fleet-depot rollout status deploy/depot-service --timeout=180s
```

## 2. Fleet LCM

```bash
kubectl -n vcf-fleet-lcm rollout restart deploy vcf-fleet-build-service-fleetbuild vcf-fleet-upgrade-service-fleetupgrade
```
```bash
kubectl -n vcf-fleet-lcm rollout status deploy/vcf-fleet-upgrade-service-fleetupgrade --timeout=180s
```

## 3. SDDC LCM

```bash
kubectl -n vcf-sddc-lcm rollout restart deploy vcf-sddc-build-service-sddcbuild vcf-sddc-upgrade-service-sddcupgrade
```

## 4. Identity Broker（VIDB / VCF SSO）

```bash
kubectl -n vidb-external rollout restart deploy vidb-service
```

> ⚠️ 重啟 VIDB 期間**所有走 VCF SSO 的登入都會失敗**（vCenter / NSX 的 federated 登入也包含在內）。

## 5. 整組 Fleet（只動 Deployment，不碰 DB）

```bash
for ns in vcf-fleet-depot vcf-fleet-lcm vcf-sddc-lcm vidb-external; do kubectl -n $ns rollout restart deploy; done
```
```bash
for ns in vcf-fleet-depot vcf-fleet-lcm vcf-sddc-lcm vidb-external; do kubectl -n $ns rollout status deploy --timeout=300s; done
```

---

## 🔴 不要重啟這些（StatefulSet = Postgres 叢集）

```
vcf-fleet-lcm/vcf-fleet-lcm-db          (3/3)
vcf-sddc-lcm/vcf-sddc-lcm-db            (3/3)
vidb-external/vidb-postgres-instance
salt-raas/pgdatabase
```

`rollout restart` 對 StatefulSet 會**逐一輪替 pod**，有掉資料與選主（failover）風險。
上面第 5 條刻意只寫 `deploy`、**不含 `sts`**。

真的要動 DB，先確認備份，並逐台觀察：
```bash
kubectl -n vcf-fleet-lcm get pod -l app=vcf-fleet-lcm-db -w
```

---

## 回滾

重啟後更糟時：

```bash
kubectl -n vcf-fleet-depot rollout undo deploy/depot-service
```
```bash
kubectl -n vcf-fleet-depot rollout history deploy/depot-service
```

---

## 檢查（唯讀，隨時可下）

```bash
kubectl -n vcf-fleet-depot get pod -o wide
```
```bash
for ns in vcf-fleet-depot vcf-fleet-lcm vcf-sddc-lcm vidb-external salt salt-raas telemetry; do echo "--- $ns"; kubectl -n $ns get deploy,sts --no-headers; done
```
```bash
kubectl get pod -A --field-selector=status.phase!=Running --no-headers
```
```bash
kubectl -n vcf-fleet-depot logs deploy/depot-service -c download-service --tail=50
```
```bash
kubectl -n vcf-fleet-depot describe pod -l app=depot-service | tail -30
```

> 💡 `depot-service` 是 **3-container** pod，看 log 要指定 `-c`：
> `download-service`（Java 主體）、`file-server`（curl 等工具在這個容器裡）。

---

## 重啟不會解決的事

| 症狀 | 真正原因 |
|---|---|
| 手改 Secret / ConfigMap 後又變回去 | **Flux / operator reconcile** —— 重啟不會改變這件事，要從正確入口改 |
| depot 抓不到東西 | depot **設定指錯位址**，見 [DEPOT-SERVICE.md](DEPOT-SERVICE.md) |
| TLS 憑證不受信任 | CA 沒進 bundle，見 [TRUST-CA.md](TRUST-CA.md) |
