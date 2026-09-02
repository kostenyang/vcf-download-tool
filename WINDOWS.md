# Windows 使用方式（含 proxy）

> 實機驗證：2026-09-02，工具版本 `9.1.0.0400.25570101`。
> 憑證一律以 `<...>` 佔位符呈現。

## 1. 安裝 —— 解壓即用

```
C:\VCF9\
├── bin\vcf-download-tool.bat    ← 執行檔
├── conf\                         ← proxy 等設定
├── jre\win32\                    ← 自帶 JRE，不用另裝 Java
├── lib\
└── log\vdt.log                   ← log 在這
```

**不需要 admin**、不用裝 Java、不用設 PATH。

```bash
C:\VCF9\bin\vcf-download-tool.bat --version
```

| 呼叫方式 | 寫法 |
|---|---|
| cmd（完整路徑） | `C:\VCF9\bin\vcf-download-tool.bat --version` |
| cmd（已 cd 到 bin） | `.\vcf-download-tool.bat --version` ← **cmd 不搜尋當前目錄** |
| PowerShell | `& "C:\VCF9\bin\vcf-download-tool.bat" --version` ← **要加 `&`** |

---

## 2. 憑證檔 —— Windows 最容易踩的地方

### 存檔要用這個寫法

```bash
[IO.File]::WriteAllText("C:\VCF9\actcode.txt", "<ACTIVATION_CODE>")
```

🔴 **不要**用 `Set-Content` / `>` / 記事本另存 —— 會加 **BOM** 或 **CRLF**。

檢查檔案是否乾淨：
```bash
$f = "C:\VCF9\actcode.txt"; "$((Get-Item $f).Length) bytes; BOM=$((Get-Content $f -Encoding Byte -TotalCount 3) -join ',')"
```
（BOM 會是 `239,187,191`）

### 🔴 Activation code 綁 Software depot ID

```bash
C:\VCF9\bin\vcf-download-tool.bat configuration get --software-depot-id
```
輸出：
```
Use this link to register https://vcf.broadcom.net/vcf/clm/download-manager/register?serviceId=<UUID>
Alternatively login at https://vcf.broadcom.com, select Software depot Registration
and use this Software depot ID: <UUID>
```

**activation code 綁定產生它的那顆 ID**。把別台機器產的 code 拿來用，會得到：
```
ERROR: Can't access Broadcom depot with provided Customer activation code eyJhc.....
```

> ⚠️ **實測經驗**：遇到這個錯誤時，很容易誤判成憑證檔格式問題（BOM / CRLF）。
> 我們實際做過對照 —— 把檔案的 CR 與結尾換行清掉後**結果完全相同**，
> 所以格式不是原因，**是 ID 不符**。判斷方式：解開 code 的 base64 看 `asset_id`
> 是否等於本機的 Software depot ID。

> 🔴 **不要用 `configuration generate --software-depot-id --force` 來「換 code」** ——
> 那會產生**新的 ID**，等於要重新註冊。要換 code 時用 `get` 讀出現有 ID 去 portal 重產即可。

---

## 3. 驗證憑證

```bash
C:\VCF9\bin\vcf-download-tool.bat releases list --depot-download-activation-code-file=C:\VCF9\actcode.txt
```

成功長相（實測）：
```
Validating depot credentials.
Depot credentials are valid.
Downloading unified release manifest file.
Successfully downloaded unified release manifest file.
9.1.0.0
9.0.2.0
...
```

> 🔴 這裡**不要**加 `--vcf-version`（單版本 detail 有 bug → `NoSuchElementException`）。

---

## 4. 列出與下載

```bash
C:\VCF9\bin\vcf-download-tool.bat binaries list --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --depot-download-activation-code-file=C:\VCF9\actcode.txt
```
```bash
C:\VCF9\bin\vcf-download-tool.bat binaries download --depot-store=C:\VCF9\depot --depot-download-activation-code-file=C:\VCF9\actcode.txt --id=<id1>,<id2> --ceip=DISABLE
```
```bash
C:\VCF9\bin\vcf-download-tool.bat binaries download --depot-store=C:\VCF9\depot --depot-download-activation-code-file=C:\VCF9\actcode.txt --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --ceip=DISABLE
```

成功長相（實測，單一 bundle）：
```
Validating available free space.
Required disk space: 1122.3, available disk space : 20621.3
Successfully validated available free space.
Starting binaries download.
Download Progress of : telemetry-acceptor-...tgz : 49.6 MB, Average Speed: 8.18 Mbps
Binary Download Summary:
0 CANCELLED | 1 SUCCESS | 0 FAILED | 0 ALREADY_DOWNLOADED
Successfully downloaded 1 binaries.
```

> 💡 下載前會**自動檢查磁碟空間**，不夠會直接擋下來。

---

## 5. 走 Proxy

**只要在原本指令加 `--proxy-server`，其他都不用改。**

### 驗證憑證
```bash
C:\VCF9\bin\vcf-download-tool.bat releases list --proxy-server=<PROXY_IP>:3128 --depot-download-activation-code-file=C:\VCF9\actcode.txt
```

### 列出
```bash
C:\VCF9\bin\vcf-download-tool.bat binaries list --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --proxy-server=<PROXY_IP>:3128 --depot-download-activation-code-file=C:\VCF9\actcode.txt
```

### 下載（精準）
```bash
C:\VCF9\bin\vcf-download-tool.bat binaries download --depot-store=C:\VCF9\depot --proxy-server=<PROXY_IP>:3128 --depot-download-activation-code-file=C:\VCF9\actcode.txt --id=<id1>,<id2> --ceip=DISABLE
```

### 下載（整組安裝集）
```bash
C:\VCF9\bin\vcf-download-tool.bat binaries download --depot-store=C:\VCF9\depot --proxy-server=<PROXY_IP>:3128 --depot-download-activation-code-file=C:\VCF9\actcode.txt --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --ceip=DISABLE
```

### Proxy 需要帳密
```bash
[IO.File]::WriteAllText("C:\VCF9\proxypw.txt", "<PROXY_PASSWORD>")
```
```bash
--proxy-user=<PROXY_USER> --proxy-user-password-file=C:\VCF9\proxypw.txt
```

### 🔴 兩個必記的 proxy 陷阱

| 陷阱 | 說明 |
|---|---|
| **`--proxy-https` 不要亂加** | 它的意思是「**連到 proxy 那一段**走 HTTPS」，**不是**「要抓 HTTPS 網站」。一般 squid（HTTP proxy on 3128）加了會壞。真要加，還得把 proxy 憑證匯進工具的 JRE trust store |
| **port 不能省** | 格式必須是 `<FQDN或IP>:<Port>`。少了 port，旗標等於沒生效，流量會**直連** |

### 驗證真的走了 proxy

指令沒報錯**不等於**流量走了 proxy。在 proxy 機器上看 log 才是唯一證明：
```bash
tail -f /var/log/squid/access.log
```
工具的每種對外連線都會經過，應看到四類 CONNECT：
```
CONNECT eapi.broadcom.com:443       # 認證 / entitlement
CONNECT dl.broadcom.com:443         # binary 下載
CONNECT vvs.broadcom.com:443        # metadata / compatibility
CONNECT vsanhealth.vmware.com:443   # vSAN HCL
```

### 官方限制
- ❌ **不支援 Kerberos / NTLMv2** 認證的 proxy
- 🔐 `--proxy-https` 需要 proxy 憑證在工具的 **JRE trust store**
- ⏱ 長時間下載請設 **TCP keepalive**

---

## 6. 背景跑（大檔）

```bash
Start-Process -FilePath "C:\VCF9\bin\vcf-download-tool.bat" -ArgumentList "binaries","download","--depot-store=C:\VCF9\depot","--proxy-server=<PROXY_IP>:3128","--depot-download-activation-code-file=C:\VCF9\actcode.txt","--vcf-version=9.1.0.0","--sku=VCF","--automated-install","--type=INSTALL","--ceip=DISABLE" -NoNewWindow -RedirectStandardOutput C:\VCF9\dl.log
```
```bash
Get-Content C:\VCF9\dl.log -Wait -Tail 20
```

---

## 7. 產出結構（實測）

單獨下一個 bundle，metadata 會一併帶下來：

```
C:\VCF9\depot\PROD\
├── COMP\
│   ├── <COMPONENT>\                       ← 你下的 binary + yaml
│   └── SDDC_MANAGER_VCF\Compatibility\
│       └── VmwareCompatibilityData.json   ← 🔑 Installer sync 必需
└── metadata\
    ├── Compatibility\v1\VmwareCompatibilityData.json
    ├── Compatibility\v2\VmwareCompatibilityData.json
    ├── productVersionCatalog\v1\productVersionCatalog.json (+ .sig)
    ├── manifest\v1\vcfManifest.json
    └── vsan\hcl\all.json
```

> ✅ **activation code 路徑會抓到 `Compatibility`** —— 這是 token 路徑做不到的。
> 少了它，VCF Installer 的 offline depot sync 會卡 `Vmware compatibility data download failed`。
> 詳見 [AUTH.md](AUTH.md)。

搬到 Linux depot 後記得修權限：
```bash
chmod -R a+rX /opt/vcf-depot/vcf9
```

---

## Windows 特有問題排查

| 症狀 | 原因 / 解法 |
|---|---|
| `Can't access Broadcom depot with provided Customer activation code` | **多半是 ID 不符**（code 綁別台）。先 `configuration get --software-depot-id` 比對 code 內的 `asset_id`。其次才是 code 過期 |
| `'vcf-download-tool.bat' 不是內部或外部命令` | cmd 不搜尋當前目錄 → `.\vcf-download-tool.bat` 或完整路徑 |
| PowerShell 找不到指令 | 加 `&` 呼叫運算子 |
| 路徑含空白（`C:\Program Files\`） | 整個路徑用**雙引號**包起來 |
| 憑證檔格式看起來對卻失敗 | BOM / CRLF → 用 `[IO.File]::WriteAllText()`。**但先排除 ID 不符** |
| `--id` 很長時報錯 | cmd 單行上限 **8191 字元** → 改用 `--download-spec-file` 或分批 |
| 防毒攔截 | 大量 .tgz/.iso 寫入可能被擋 → 把 depot 目錄加入排除清單 |
| 下載中斷 | 沒有續傳旗標，但同一 `--depot-store` **重跑會跳過已完成的**，直接重下即可 |
