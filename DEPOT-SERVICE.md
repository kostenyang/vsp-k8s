# depot-service：設定存哪、為什麼只能走 UI 改

## Pod 結構

`depot-service` 是 **3-container** 的 Deployment：

| Container | 角色 |
|---|---|
| `download-service` | Java 主體（Spring），實際去 depot 抓東西 |
| `file-server` | 提供檔案服務；**`curl` 等工具在這個容器裡**，做連線測試用它 |
| （第三個為 sidecar） | |

看 log 一定要指定容器：
```bash
kubectl -n vcf-fleet-depot logs deploy/depot-service -c download-service --tail=50
```

---

## 設定分兩層

### 第一層：`download-service-configs`（ConfigMap，靜態）

```bash
kubectl -n vcf-fleet-depot get cm download-service-configs -o yaml
```

重點欄位：
```properties
lcm.bundle.download.root.dir=/data/depot
root.dir=/data/depot
root.tmp.dir=/data/tmp

lcm.access_token.broadcom.authorization.server.url=https://eapi.broadcom.com/vcf/generateToken
lcm.depot.adapter.host=dl.broadcom.com          # online 模式的來源
lcm.depot.adapter.remote.v2.rootDir=/PROD
lcm.depot.adapter.remote.vcfMetadataDir=/metadata
lcm.depot.adapter.enableSignatureValidation=true
lcm.depot.adapter.trustedBundleSignatureFilePath=conf/trusted_sig.crt
```

> 💡 `lcm.depot.adapter.host` 永遠是 `dl.broadcom.com`（online 用），
> **offline 模式的位址不在這裡**，在下面的 Secret。

### 第二層：`depot-service-secret`（Secret，實際生效的 depot 設定）

```bash
kubectl -n vcf-fleet-depot get secret depot-service-secret -o jsonpath='{.data}' | tr ',' '\n' | grep -oE '"[a-zA-Z0-9._-]+":' | tr -d '":'
```

| Key | 意義 |
|---|---|
| `depotType` | `OFFLINE` / （connected） |
| `offlineOrigin` | **離線 depot 的 URL** |
| `offlineBasePath` | 基底路徑 |
| `offlineUsername` / `offlinePassword` | basic auth |
| `offlineCertificate` | 憑證 |
| `activationCode` | **online 模式用**的啟用碼 |
| `machineId` | **Software depot ID**（拿去 portal 換 activation code） |

唯讀查看（不印密碼）：
```bash
for k in depotType offlineOrigin offlineUsername offlineBasePath machineId; do printf '%-16s = %s\n' "$k" "$(kubectl -n vcf-fleet-depot get secret depot-service-secret -o jsonpath="{.data.$k}" | base64 -d)"; done
```

---

## 🔴🔴 不要用 kubectl patch 改這個 Secret

```bash
kubectl -n vcf-fleet-depot get secret depot-service-secret -o jsonpath='{.metadata.labels}'
```

會看到：
```
app.kubernetes.io/managed-by: depot-service-manager
helm.toolkit.fluxcd.io/name: depot-service
helm.toolkit.fluxcd.io/namespace: vcf-fleet-depot
```

→ 由 **`depot-service-manager` + Helm/Flux** 管理，**手改會被 reconcile 還原**。

**正確入口：VCF Operations UI**
**Build → Software Depot → 選 fleet → EDIT**

---

## UI 設定流程（offline depot）

1. **Connection Mode** → `Offline Depot` → NEXT
2. **URL** → `https://<DEPOT_FQDN>`（用 FQDN）
3. **Authentication** → 打開
   > UI 明示「Authentication is supported only with an **HTTPS** offline depot」
   > → HTTP 位址時這個開關是關的，帳密欄不會出現
4. **Username / Password**
5. **VALIDATE** → `Validation completed successfully. No errors found.`
6. **FINISH**

## 觸發下載

**Build → Lifecycle → VCF Management** → 右上 **Sync**
（`Last lifecycle metadata sync time` 從 `N/A` 變成實際時間）

---

## 驗證真的在下載：看 depot 的 access log

在 **depot server** 上：
```bash
grep 'Apache-HttpClient' /var/log/nginx/access.log | tail -10
```

實測長相：
```
<VSP_NODE_IP> - <USER> "GET /PROD/metadata/productVersionCatalog/v1/productVersionCatalog.json" 200 1276766 "Apache-HttpClient/5.5.1 (Java/21.0.9)"
<VSP_NODE_IP> - <USER> "GET /PROD/metadata/productVersionCatalog/v1/productVersionCatalog.sig"  200    1930 "Apache-HttpClient/5.5.1 (Java/21.0.9)"
<VSP_NODE_IP> - <USER> "GET /PROD/metadata/manifest/v1/vcfManifest.json"                        200  214215 "Apache-HttpClient/5.5.1 (Java/21.0.9)"
```

| 判讀點 | 意義 |
|---|---|
| 來源是 **VSP 節點 IP** + UA `Apache-HttpClient/…(Java/…)` | ✅ 真的是 Fleet 在抓，不是你的瀏覽器 / curl |
| 第 3 欄是帳號（不是 `-`） | ✅ basic auth 有帶且通過 |
| `.sig` 也被抓 | ✅ **簽章驗證**路徑有在走（`enableSignatureValidation=true`） |
| 只有 `401`，或第 3 欄是 `-` | ❌ 帳密沒設 |
| **完全沒有新紀錄** | ❌ URL 指到別的位址，Fleet 根本沒打到這台 |

---

## Offline → Online（Connected）

Online 模式需要 **activation code**，流程：

1. 取得 Software depot ID：
```bash
kubectl -n vcf-fleet-depot get secret depot-service-secret -o jsonpath='{.data.machineId}' | base64 -d; echo
```
2. 到 `https://vcf.broadcom.com` → **Software depot Registration** → 貼 ID → 產 activation code
3. UI 切 **Connection Mode = Connected** → 貼 code → VALIDATE → FINISH

### 診斷 online 連不上

從 pod 內測真實端點（**不要測根路徑** —— 根路徑本來就回 403/500）：
```bash
kubectl -n vcf-fleet-depot exec deploy/depot-service -c download-service -- curl -sS -m 10 -o /dev/null -w "HTTP=%{http_code}\n" https://dl.broadcom.com/PROD/metadata/productVersionCatalog/v1/productVersionCatalog.json
```

| 結果 | 判讀 |
|---|---|
| 回應 header 有 `Server: cloudflare` / `CF-RAY` | ✅ **網路通**，真的連到 Broadcom CDN |
| `HTTP=403` 且 `activationCode` 為空 | 🔴 **沒有憑證** —— 這是最常見原因 |
| `HTTP=403` 但 code 有填且 VALIDATE 過 | 該帳號**沒有 binary 下載 entitlement**（site/tenant 不同）→ 換一組 |
| `Can't access … with provided activation code` | code **有時效** → 用**同一顆 machineId** 重產 |
| DNS 解不出 / connect timeout | 真的沒有 egress → 需要 proxy |

> 🔴 **不要重產 machineId** —— ID 一改，既有 activation code 立刻失效。
> 🔴 切成 Connected **不會刪掉 offline 設定**（`offlineOrigin`/帳密都留著），可以安全來回切。
> 💡 proxy 若做 **TLS 攔截**，要把 proxy 的 CA 用 [TRUST-CA.md](TRUST-CA.md) 的方式匯入。
