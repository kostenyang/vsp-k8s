# 登入 VSP 節點並取得 kubectl

## 座標

| 項目 | 值 |
|---|---|
| VSP 節點 IP | **外層網段**（本 lab `10.0.0.227–231`）—— 是 outer network，Windows / 一般 Linux 都路由得到，**不是** nested 內網 |
| 帳號 | `vmware-system-user`（節點允許 password 登入） |
| 密碼 | `<VSP_PW>` |
| kubeconfig | `/etc/kubernetes/admin.conf` —— **只在 control-plane 節點** |
| Fleet API VIP | `<VIP>:443` —— 直打 IP 回 **404**，要用 FQDN 或帶 `Host:` header |

---

## 從 Linux / macOS

```bash
ssh -o StrictHostKeyChecking=no vmware-system-user@<VSP_NODE_IP>
```
```bash
sudo -i
```
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf && kubectl get nodes -o wide
```

應看到類似：

```
NAME                  STATUS   ROLES           VERSION            INTERNAL-IP
vcf-m02-vsp01-ftck5   Ready    control-plane   v1.34.2+vmware.1   10.0.0.228   <- admin.conf 在這台
vcf-m02-vsp01-zfp57   Ready    <none>          v1.34.2+vmware.1   10.0.0.229
vcf-m02-vsp01-q278k   Ready    <none>          v1.34.2+vmware.1   10.0.0.230
vcf-m02-vsp01-6kdzn   Ready    <none>          v1.34.2+vmware.1   10.0.0.231
```

> 🔴 **連錯節點的症狀**：`kubectl` 報 connection refused、或找不到 `/etc/kubernetes/admin.conf`
> → 你在 worker 上。改 SSH 到 `ROLES=control-plane` 那個 `INTERNAL-IP`。
>
> 🔴 **IP↔角色每次重建都會變**。本輪 control-plane 是 `.228`，
> 而且 `.227` 可能是**同一台的第二個 IP**（同一把 host key）。務必照 `ROLES` 欄認。

---

## sudo 沒有 passwordless

節點的 `sudo` **會問密碼**，非互動情境要用 `-S` 從 stdin 餵：

```bash
echo '<VSP_PW>' | sudo -S -p '' <command>
```

---

## 從 Windows（PuTTY plink / pscp）

### 上傳腳本再執行（推薦 —— 避免引號地獄）

```bash
"/c/Program Files/PuTTY/pscp" -pw '<VSP_PW>' script.sh vmware-system-user@<VSP_NODE_IP>:/tmp/
```
```bash
"/c/Program Files/PuTTY/plink" -ssh -batch -pw '<VSP_PW>' vmware-system-user@<VSP_NODE_IP> "echo '<VSP_PW>' | sudo -S -p '' bash /tmp/script.sh"
```

腳本開頭記得：

```bash
#!/usr/bin/env bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### 單行指令

```bash
"/c/Program Files/PuTTY/plink" -ssh -batch -pw '<VSP_PW>' vmware-system-user@<VSP_NODE_IP> "echo '<VSP_PW>' | sudo -S -p '' bash -c 'export KUBECONFIG=/etc/kubernetes/admin.conf; kubectl get nodes'"
```

### 🔴 host key 雷

VSP 節點的 IP pool（例如 `10.0.0.226-240`）**會重用**。重建叢集後同一個 IP 換了新主機，
plink 會擋下來報 **`POTENTIAL SECURITY BREACH`**。

用 `-hostkey` 釘住當前指紋（指紋每次重建會變，先連一次看它印出來的新指紋）：

```bash
"/c/Program Files/PuTTY/plink" -ssh -batch -hostkey "SHA256:<當前指紋>" -pw '<VSP_PW>' vmware-system-user@<VSP_NODE_IP> "hostname"
```

Linux 端對應寫法：

```bash
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null vmware-system-user@<VSP_NODE_IP>
```

---

## 把 kubeconfig 拉回本機用（選用）

```bash
"/c/Program Files/PuTTY/plink" -ssh -batch -pw '<VSP_PW>' vmware-system-user@<CP_IP> "echo '<VSP_PW>' | sudo -S -p '' cat /etc/kubernetes/admin.conf" > vsp-kubeconfig.yaml
```

⚠️ 檔案裡的 `server:` 多半指向節點內部位址，本機要用得先確認路由得到，
必要時改成 control-plane 的 `INTERNAL-IP`。**這份檔等同 cluster-admin 憑證，請當機密保管。**

---

## 常用起手式

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```
```bash
kubectl get nodes -o wide
```
```bash
kubectl get ns
```
```bash
kubectl get pod -A --field-selector=status.phase!=Running --no-headers | head -20
```
