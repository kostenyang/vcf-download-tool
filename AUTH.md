# 認證：token vs activation code

工具有**兩條**認證路徑。搞清楚差別，可以省掉很多白工。

| | **Download token** | **Activation code** |
|---|---|---|
| 旗標 | `--depot-download-token-file` | `--depot-download-activation-code-file` |
| 綁機器 | ❌ 不綁 | ✅ **綁 Software depot ID** |
| 換機器 | 直接拿去用 | **失效**，要重產 |
| 有時效 | — | ✅ 會過期 |
| 抓 binary | ✅ | ✅ |
| 抓 **Compatibility 互通矩陣** | ❌ **抓不到** | ✅ |
| 9.1 狀態 | **superseded**（工具明示留到 5.x EOL） | 現行 |

> 🔴 **這不是「新舊取代」而已，是能力差異** ——
> VCF Installer 的 offline depot sync **需要**
> `COMP/SDDC_MANAGER_VCF/Compatibility/VmwareCompatibilityData.json`。
> 只用 token 下載，這個檔不會有，Installer sync 會卡在
> **`Vmware compatibility data download failed`**。

---

## A. Activation code 路徑（建議）

### 1. 取得 Software depot ID

```bash
vcf-download-tool configuration get --software-depot-id
```

輸出：
```
Software depot ID: <UUID>
註冊連結 https://vcf.broadcom.net/vcf/clm/download-manager/register?serviceId=<UUID>
```

首次若還沒有，可產生：
```bash
vcf-download-tool configuration generate --software-depot-id
```
```bash
vcf-download-tool configuration generate --software-depot-id --force
```

> 🔴 **`--force` 會產一顆新 ID → 既有 activation code 立刻作廢**。
> 要換 code 時請用 **`get`** 讀出原本那顆 ID 去 portal 重產，**不要** `generate --force`。

### 2. 換 activation code

開上面那個註冊連結，或登入
**`https://vcf.broadcom.com` → Software depot Registration** → 貼 ID → 產 code。

存成**單行文字檔**（無結尾換行）：
```bash
echo '<ACTIVATION_CODE>' > actcode.txt && chmod 600 actcode.txt
```

### 3. 驗證

```bash
vcf-download-tool releases list --depot-download-activation-code-file=actcode.txt
```
看到 `Validating depot credentials.` → `Depot credentials are valid.` 就成功。

> 🔴 **不要加 `--vcf-version`** —— 工具的單版本 detail 有 bug，會噴 `NoSuchElementException`。不帶版本列全部即可。

---

## B. Token 路徑

從 Broadcom Support Portal 的 **Generate Download Token** 取得，存成單行檔：

```bash
echo '<TOKEN>' > token.txt && chmod 600 token.txt
```

Windows 取得的檔先轉 LF：
```bash
sed -i 's/\r$//' token.txt
```

之後把所有指令的認證旗標換成：
```
--depot-download-token-file=token.txt
```

適用情境：
- 只要 binary、**不需要** Compatibility（例如補幾個元件到既有 depot）
- 要在**多台機器**上下載（token 不綁機器）

---

## 在 K8s Fleet 上的對應

VSP 上的 `depot-service` 也是同一套概念：

```bash
kubectl -n vcf-fleet-depot get secret depot-service-secret -o jsonpath='{.data.machineId}' | base64 -d; echo
```

那個 `machineId` **就是 Software depot ID**，拿去 portal 換 code，填進
VCF Operations UI 的 **Connected** 模式。細節見
[`vsp-k8s` / DEPOT-SERVICE.md](https://github.com/kostenyang/vsp-k8s)。

---

## 認證相關錯誤

| 訊息 / 現象 | 原因 | 解法 |
|---|---|---|
| `Missing required argument: --depot-download-activation-code-file` | **每個** list/download 動作都要帶認證檔 | 補上 |
| `Can't access Broadcom depot with provided activation code` | code **過期** | 用**同一顆 ID** 到 portal 重產 |
| 產完 ID 後 code 突然失效 | 用了 `generate --force`，ID 換了 | 拿新 ID 重產 code |
| list / metadata 正常，但 binary **HTTP 403** | 該帳號**沒有 binary 下載 entitlement**（site/tenant 權限不同） | 換一顆有下載權限的 code。**與指令、proxy 都無關** |
| Installer sync 卡 `Vmware compatibility data download failed` | 用 token 下的，缺 Compatibility | 改用 activation code，或另外補 metadata zip |
