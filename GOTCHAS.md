# 踩雷合輯

> 建議下載前先掃一遍這份。多數「工具壞掉」其實是這幾條之一。

## Filter 相關

| # | 症狀 | 原因 | 解法 |
|---|---|---|---|
| 1 | 印出 usage 然後 `exit 2` | **沒給 filter**，或**混用了互斥群組** | `[VCF VERSION]` / `[BUNDLE ID]` / `[DOWNLOAD SPEC]` **三選一**，且至少要給一個 |
| 2 | `--component-version` 單獨用沒反應 | 它屬 `[VCF VERSION]` 群組 | **必須同時帶 `--vcf-version`** |
| 3 | `--vcf-version=9.1.0.0400` 回 **0 elements** | `0400` 是 patch **不是 release** | 用 release 版 **`9.1.0.0`** |
| 4 | 「最新」誤挑到 `9.0.2` / NSX `4.2.4` 等別線版本 | 只看尾碼 build number，跨產品線會誤判 | 用 **`--id`** 釘各元件正確版 |

## 認證相關

| # | 症狀 | 原因 | 解法 |
|---|---|---|---|
| 5 | `Missing required argument: --depot-download-activation-code-file` | **每個** list / download 動作都要帶認證檔 | 補上 |
| 6 | 產完 ID 後 code 立刻失效 | 用了 `configuration generate --software-depot-id --force`，**ID 一改舊 code 作廢** | 換 code 請用 **`get`** 讀原 ID 去 portal 重產 |
| 7 | `Can't access Broadcom depot with provided activation code` | code **有時效** | 用**同一顆 ID** 重產一顆新 code |
| 8 | list / metadata 讀得到，**binary 下載 403** | 該憑證的帳號**沒有 binary 下載 entitlement**（site/tenant 權限不同） | 換一顆有下載權限的。**與指令、proxy 都無關** —— `--id` 仍會正確命中（印 `1 element`） |
| 9 | Installer sync 卡 `Vmware compatibility data download failed` | 用 **token** 下載，抓不到 Compatibility 互通矩陣 | 改用 **activation code**，或另補 metadata zip |

## 指令 / 環境

| # | 症狀 | 原因 | 解法 |
|---|---|---|---|
| 10 | `releases list --vcf-version=X` 噴 `NoSuchElementException` | 工具**單版本 detail 的 bug** | **不帶版本**列全部即可 |
| 11 | 第一次跑卡住不動 | 在等你回答 **CEIP** | 加 `--ceip=DISABLE`（選過會記住） |
| 12 | 以為要 root / admin | **不用** | 工具綠色可攜，一般使用者即可跑 |
| 13 | 認證檔讀不到 / 認證失敗但值看起來對 | 檔案有 **CRLF 或結尾換行** | `sed -i 's/\r$//' <file>`，確保**單行無結尾換行** |

## Proxy

| # | 症狀 | 原因 | 解法 |
|---|---|---|---|
| 14 | 加了 proxy 反而連不上 | 誤加 `--proxy-https`（那是「連 proxy 用 HTTPS」） | 一般 HTTP proxy **拿掉它** |
| 15 | proxy log 沒有任何 CONNECT | `--proxy-server` **少了 port** | 格式是 `<FQDN:Port>`，port 必填 |
| 16 | proxy 需要 Kerberos / NTLMv2 | **官方不支援** | 換支援 basic auth 的 proxy，或開白名單直連 |

## 下載結果

| # | 症狀 | 原因 | 解法 |
|---|---|---|---|
| 17 | 「重下」什麼都沒發生 | `download` 是**累加**，已存在的會跳過 | 先 `cleanup` → 再 `download`（見 [CLEANUP.md](CLEANUP.md)） |
| 18 | 下到一半空間爆掉 | 沒先估量體 | 全元件約 **130 GB**；先 `df -h` |
| 19 | depot 架好但 Installer / Fleet 讀不到 | **權限** | `chmod -R a+rX <depot>` |
| 20 | 補料後客戶原本的版本抓不到了 | **覆蓋了 `PROD/metadata/`**，型錄序號被蓋掉 | 補料**只搬 `PROD/COMP/`**；先備份 metadata |

---

## 最貴的一條：#20

實際發生過的事故 ——

交付 tar 用 `binaries export` 打包，**連新版 metadata 一起包進去**（seq48）。
客戶原始 depot 是 seq43（用舊版工具抓的、cap 在客戶自己的版本集）。
客戶原樣解開 → **型錄被蓋成 seq48** → installer 要的版本在新型錄裡不存在 → **Failed**。

**正確做法**：交付**只含 binaries**（`PROD/COMP/`），讓客戶保留自己的 `PROD/metadata/`。
匯入腳本要「先備份 metadata → 只解 COMP → 驗證型錄序號沒變」。

---

## 快速自檢

下載前：
```bash
df -h <depot-store 所在磁碟>
```
```bash
vcf-download-tool releases list <AUTH>
```

下載後：
```bash
grep -E 'SUCCESS|FAILED' /tmp/vcf-dl.log | tail -5
```
```bash
ls <depot-store>/PROD/COMP/SDDC_MANAGER_VCF/Compatibility/
```
```bash
chmod -R a+rX <depot-store>
```
