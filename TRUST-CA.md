# 憑證信任：9.1 用 trust-manager，不再是 keytool

## 機制對照

| | 匯入 CA 的方式 |
|---|---|
| 舊版 appliance | `keytool` 匯入 `/etc/vmware/vcf/commonsvcs/trusted_certificates.store`（KB 316056） |
| **9.1 Fleet** | K8s 上由 **cert-manager `trust-manager`** 的 **`platform-trust` Bundle** 管理 |

`platform-trust` Bundle（cluster-scoped）的組成：

```yaml
sources:
  - useDefaultCAs: true                      # 公開 CA（本 lab 實測 181 張）
  - secret:
      key: ca.crt
      selector:
        matchLabels:
          trust.vmsp.vmware.com/bundle: platform-trust   # ← 你要建的 Secret 帶這個 label
target:
  configMap: platform-trust                  # 產出 bundle.pem + bundle.jks
```

pod 掛載於 **`/etc/platform/trust`**。

> 🔴 **直接改 configmap 沒有用** —— `trust-manager` 會覆蓋回去。
> 正解 = 在 trust namespace（`vmsp-platform`）建一個**帶該 label、key 為 `ca.crt` 的 Secret**，
> `trust-manager` 會自動合併進 bundle 並推送到所有 pod。

---

## 匯入一張自簽 CA

### 1. 取得 CA（PEM）

從線上抓（build-agnostic，最保險）：
```bash
openssl s_client -connect <DEPOT_IP>:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform pem > /root/depot-ca.pem
```
```bash
openssl x509 -in /root/depot-ca.pem -noout -subject -issuer
```

> 💡 若 `subject == issuer`，那是**自簽單張**憑證（`openssl req -x509` 直接自簽，
> 不是 root CA + leaf 兩層）→ **這張本身就是要匯入的信任錨**。

### 2. 記下匯入前的憑證數

```bash
kubectl -n vcf-fleet-depot get cm platform-trust -o jsonpath='{.data.bundle\.pem}' | grep -c 'BEGIN CERTIFICATE'
```

### 3. 建帶 label 的 Secret

```bash
kubectl -n vmsp-platform create secret generic depot-ca --from-file=ca.crt=/root/depot-ca.pem
```
```bash
kubectl -n vmsp-platform label secret depot-ca trust.vmsp.vmware.com/bundle=platform-trust
```

### 4. 等同步並確認 +1

```bash
sleep 20 && kubectl -n vcf-fleet-depot get cm platform-trust -o jsonpath='{.data.bundle\.pem}' | grep -c 'BEGIN CERTIFICATE'
```
```bash
kubectl get bundles.trust.cert-manager.io platform-trust
```

本 lab 實測：**181 → 182**，Bundle CR `SYNCED=True`。

---

## 驗證真的生效（從 pod 內打）

```bash
kubectl -n vcf-fleet-depot exec deploy/depot-service -c file-server -- ls /etc/platform/trust
```
應看到 `bundle.pem`、`bundle.jks`。

```bash
kubectl -n vcf-fleet-depot exec deploy/depot-service -c file-server -- curl -sS --cacert /etc/platform/trust/bundle.pem -u "<USER>:<PW>" -o /dev/null -w "HTTP=%{http_code} tls_verify=%{ssl_verify_result}\n" https://<DEPOT_FQDN>/
```

### 判讀

| 結果 | 意義 |
|---|---|
| `tls_verify=0` | ✅ **憑證驗證通過** |
| `tls_verify=1` + `HTTP=000` | ❌ CA 沒生效 |
| `HTTP=401` + `tls_verify=0` | ⚠️ TLS 已通，只是缺帳密 —— **不是** CA 失敗 |
| `HTTP=404` | ⚠️ 路徑錯，與憑證無關 |

> 🔴 **判斷 CA 成敗只看 `tls_verify`**（0=通過、1=失敗）。
> 看到 `401` 就以為 CA 沒成功是最常見的誤判。

### 對照組（證明成功真的來自 CA）

```bash
kubectl -n vcf-fleet-depot exec deploy/depot-service -c file-server -- curl -sS -u "<USER>:<PW>" -o /dev/null -w "no-CA: HTTP=%{http_code} tls_verify=%{ssl_verify_result}\n" https://<DEPOT_FQDN>/
```
不給 `--cacert` **應該要失敗**（`HTTP=000 tls_verify=1`）。若這樣還成功，代表你驗到的不是你以為的東西。

---

## 確認某張 CA 是否已在 bundle

```bash
kubectl -n vcf-fleet-depot get cm platform-trust -o jsonpath='{.data.bundle\.pem}' > /tmp/b.pem && openssl crl2pkcs7 -nocrl -certfile /tmp/b.pem | openssl pkcs7 -print_certs -noout | grep -i <關鍵字>
```

列出目前所有提供 CA 的 Secret：
```bash
kubectl -n vmsp-platform get secret -l trust.vmsp.vmware.com/bundle=platform-trust
```

---

## 移除

```bash
kubectl -n vmsp-platform delete secret depot-ca
```
`trust-manager` 會自動從 bundle 移掉。

---

## 生效時間與快取

- configmap 更新後，**pod 約 1 分鐘**才拿到新 bundle（kubelet 同步週期）。
- 服務若有 TLS 快取，才需要重啟：
```bash
kubectl -n vcf-fleet-depot rollout restart deploy depot-service distribution-service
```

---

## 適用範圍

同一套機制可用於任何要讓 Fleet 信任的私有 CA，不限離線 depot：

- HTTPS 離線 depot 的自簽憑證
- **企業 proxy 做 TLS 攔截**時的 proxy CA
- 內部 PKI 簽發的服務憑證
