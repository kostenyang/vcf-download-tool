# 走企業 Proxy

**只要在原本指令加 `--proxy-server`，其他都不用改。**

## 旗標

| 旗標 | 說明 |
|---|---|
| `-s, --proxy-server=<FQDN:Port>` | proxy 位址，**port 必填** |
| `-r, --proxy-user=<user>` | proxy 需認證時的帳號 |
| `--proxy-user-password-file=<file>` | 單行密碼檔；不給會互動詢問 |
| `--proxy-https` | **「連到 proxy 這一段用 HTTPS」** —— 見下方警告 |

---

## 🔴 `--proxy-https` 最常被誤加

它的意思**不是**「我要抓 HTTPS 網站」，而是「**我到 proxy 的那一段**走 HTTPS」。

- 一般 HTTP proxy（squid `:3128`）→ **不要加**，加了會壞
- 真的要加 → 還得把 **proxy 的憑證匯入工具的 JRE trust store**

---

## 指令（每步都加 proxy）

> `<AUTH>` = `--depot-download-activation-code-file=actcode.txt`
> 或 `--depot-download-token-file=token.txt`

**① 驗憑證**
```bash
vcf-download-tool releases list --proxy-server=<PROXY_IP>:3128 <AUTH>
```
成功會看到 `Proxy configuration completed` + `Depot credentials are valid`。

**② 列版本**
```bash
vcf-download-tool binaries list --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --proxy-server=<PROXY_IP>:3128 <AUTH>
```

**③ 下載（精準）**
```bash
vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 --proxy-server=<PROXY_IP>:3128 <AUTH> --id=<id1>,<id2> --ceip=DISABLE
```

**③' 下載（整批）**
```bash
vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 --proxy-server=<PROXY_IP>:3128 <AUTH> --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --ceip=DISABLE
```

**Proxy 需要認證時再加**
```bash
--proxy-user=<PROXY_USER> --proxy-user-password-file=/root/proxy-pw.txt
```

**Windows**
```bash
C:\VCF9\bin\vcf-download-tool.bat binaries download --depot-store=C:\VCF9\depot --proxy-server=<PROXY_IP>:3128 --depot-download-activation-code-file=C:\VCF9\actcode.txt --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --ceip=DISABLE
```

---

## 驗證「真的有走 proxy」

在 proxy 機器上看 log：

```bash
tail -f /var/log/squid/access.log
```

自建 systemd proxy 的話：
```bash
journalctl -u <proxy-service> -f
```

工具的**每一種對外連線**都會經過 proxy，應看到四類 CONNECT：

```
CONNECT eapi.broadcom.com:443         # 認證 / entitlement
CONNECT dl.broadcom.com:443           # binary 下載主機
CONNECT vvs.broadcom.com:443          # metadata / compatibility
CONNECT vsanhealth.vmware.com:443     # vSAN HCL
```

> 這是唯一可靠的證明。指令沒報錯**不等於**流量走了 proxy。

---

## 實測結論（2026-07-28）

自建 CONNECT proxy、工具在 Windows：

| 動作（帶 `--proxy-server`） | 結果 |
|---|---|
| `releases list`（驗憑證） | ✅ exit=0，`Proxy configuration completed` |
| `binaries list` | ✅ exit=0，回完整目錄（16 元件） |
| `binaries download --id=…` | 請求**確實穿過 proxy 送達** `dl.broadcom.com` → 回 **403** |

proxy journal 共記錄 **38 筆 CONNECT**，涵蓋上述四個主機。

### 那個 403 不是 proxy 的問題

是**當下那顆憑證沒有 binary 下載 entitlement**（proxy log 證明請求已送達）。
換一顆有下載權限的，**同一條 `--proxy-server` 指令**就會實際落檔。

佐證：同一專案先前（不經 proxy、用有權限的憑證）實測
`binaries download --id=<16 元件>` → **16/16 SUCCESS、66 GB、含完整 metadata**。

→ **有權限的憑證 + proxy 環境 = 可完整下載**。

---

## 官方限制

- ❌ **不支援 Kerberos / NTLMv2 認證的 proxy**
- 🔐 `--proxy-https` 需要把 proxy 憑證放進**工具的 JRE trust store**
- ⏱ 長時間下載請設 **TCP keepalive**，避免連線 timeout

---

## 用設定檔取代旗標（免每次帶）

工具的 `conf/` 底下有 proxy 設定，預設關閉：

```bash
grep -n proxy conf/application-prod.properties
```
```properties
lcm.depot.adapter.proxyEnabled=false
lcm.depot.adapter.proxyHost=proxy.vmware.com
lcm.depot.adapter.proxyPort=3128
```

把 `proxyEnabled` 改 `true` 並填好 host/port 即可。

> ⚠️ 同樣的區塊在 `application-prodv2.properties`、`application-asyncpatch.properties`、
> `app_new_1.properties` 都有一份 —— **改你實際跑的那個 profile**。

---

## 排查

| 症狀 | 原因 | 解法 |
|---|---|---|
| 加了 proxy 就連不上 | 誤加 `--proxy-https` | 一般 HTTP proxy 拿掉它 |
| proxy log 沒有任何 CONNECT | 旗標沒生效 / 打錯 port | `--proxy-server` 的 **port 必填** |
| 認證步驟就過不了 | proxy 擋 `eapi.broadcom.com` | 開放該網域 |
| list 可以、download 403 | **憑證沒有下載權限** | 換憑證，**與 proxy 無關** |
| proxy 要 Kerberos/NTLMv2 | **官方不支援** | 改用支援 basic auth 的 proxy，或開白名單直連 |
| 下載中途斷掉 | 連線被 timeout | 設 TCP keepalive；重跑會**跳過已下載的**續傳 |
