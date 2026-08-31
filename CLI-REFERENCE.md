# CLI 速查（實測 `9.1.0.0400.25570101`）

```bash
vcf-download-tool --version
```
```bash
vcf-download-tool --help
```

---

## 命令樹

| 命令 | 用途 |
|---|---|
| `binaries download` | 下載 binary + 所需 metadata |
| `binaries list` | 列出可下載項目（ID / 元件 / 版本 / 日期 / 大小 / 型別） |
| `binaries cleanup` | **刪掉 filter 命中的** binary |
| `binaries upload` | 上傳進 SDDC Manager（**須在 SDDC Manager appliance 上執行**） |
| `binaries create-download-spec` | 依升級計畫產 spec（**須在 SDDC Manager 上執行**） |
| `metadata download / upload / export` | 只處理 metadata；`export` 打包成 tar |
| `esx download / export / import / metadata / configuration` | ESX binary 與 depot |
| `depot binaries {upload\|cleanup\|list}` / `depot metadata` / `depot esx` | Software depot 端管理 |
| `releases list` | 列出 VCF releases（5.0 以後） |
| `configuration generate --software-depot-id` | 產生 Software depot ID |
| `configuration get --software-depot-id` | 讀回 Software depot ID |

---

## 通用旗標

| 旗標 | 說明 |
|---|---|
| `-d, --depot-store=<dir>` | 本地 depot 目錄（`download` / `cleanup` **必填**） |
| `--ceip=<ENABLE\|DISABLE>` | 首次不帶會互動詢問；選過會記住 |
| `-h, --help` / `-v, --version` | — |

## 認證（`[DEPOT]` 群組，二選一）

| 旗標 | 說明 |
|---|---|
| `--depot-download-activation-code-file=<file>` | **現行**做法 |
| `--depot-download-token-file=<file>` | 舊做法，**superseded**（留到 5.x EOL） |

詳見 [AUTH.md](AUTH.md)。

## Proxy（`[PROXY]` 群組）

| 旗標 | 說明 |
|---|---|
| `-s, --proxy-server=<FQDN:Port>` | proxy 位址，**port 必填** |
| `-r, --proxy-user=<user>` | proxy 帳號 |
| `--proxy-user-password-file=<file>` | 單行密碼檔；不給會互動詢問 |
| `--proxy-https` | **「連到 proxy 這段用 HTTPS」** —— 一般 HTTP proxy **不要加** |

詳見 [PROXY.md](PROXY.md)。

---

## Filter —— 三組互斥，至少要給一組

`binaries list` / `binaries download` / `binaries cleanup` 共用同一套 filter。

### `[VCF VERSION]`

| 旗標 | 說明 |
|---|---|
| `--vcf-version=<a.b[.c[.d]][..[end]]>` | `9.1` → 9.1.0.0 及其下所有維護版；`9.1.0.0` → 精準；`9.0..` → 9.0 以上全部 |
| `--sku=<VCF\|VVF>` | 產品型別 |
| `-t, --type=<INSTALL\|UPGRADE>` | 安裝 or 升級 |
| `--automated-install` | VCF **Installer** 需要的那一組 |
| `--component=<KEY>` | 元件（清單見下） |
| `--component-version=<ver>` | 元件版本 —— **必須同時帶 `--vcf-version`** |
| `--lifecycle-managed-by=<SDDC_MANAGER_VCF\|VRSLCM\|VCF_FLEET_LCM\|SELF>` | 依生命週期管理者 |
| `--patches-only` / `--upgrades-only` | 只要 patch / 只要 upgrade |

### `[BUNDLE ID]`

| 旗標 | 說明 |
|---|---|
| `--id=<bundleId>[,<bundleId>...]` | 可逗號分隔或重複給。**最精準**，一個 ID = 一個確切版本 |

### `[DOWNLOAD SPEC]`

| 旗標 | 說明 |
|---|---|
| `--download-spec-file=<file>` | 由 `binaries create-download-spec` 產生 |

---

## `--component` 可選值（實測全集，大小寫不拘）

```
VCENTER                 SDDC_MANAGER_VCF        NSX_T_MANAGER
ESX_HOST                VRSLCM                  VRA
VROPS                   VRLI                    VRNI
VSAN_OSA_WITNESS        VSAN_ESA_WITNESS        VSAN_FILE_SERVICES
VMTOOLS                 VCFDT                   VCF_OPS_CLOUD_PROXY
VIDB                    HCX                     VMRC
VRO                     VSP                     DEPOT_SERVICE
VCF_LICENSE_SERVER      TELEMETRY_ACCEPTOR      VCF_FLEET_LCM
VCF_SDDC_LCM            VCF_CONSUMPTION_CLI     VCF_CONSUMPTION_CLI_PLUGINS
VCFMS_METRICS_STORE     VCF_SERVICE_VCD_MIGRATION_BACKEND
VCF_SALT                VCF_SALT_RAAS           VCF_OBSERVABILITY_DATA_PLATFORM
```

---

## SDDC Manager 專用（須在 appliance 上跑）

`binaries upload` / `binaries create-download-spec` 另需：

| 旗標 | 說明 |
|---|---|
| `--sddc-manager-fqdn=<fqdn>` | 目標 SDDC Manager |
| `--sddc-manager-user=<user>` | 帳號 |
| `--sddc-manager-user-password-file=<file>` | 單行密碼檔 |
| `--domain-name=<name>[,<name>...]` | *(create-download-spec)* 要規劃升級的 domain |
| `--download-spec-dir=<dir>` | *(create-download-spec)* spec 輸出目錄 |

---

## 行為備註

- `binaries download` **同時會下 metadata**（product version catalog、unified release manifest、vSAN HCL、Compatibility），產出 `<depot-store>/PROD/`。
- **UMDS binaries 一律排除**，不會下載。
- 同一個 `--depot-store` 重跑是**累加**（跳過已下載的），不覆蓋。
- 每個檔會**對簽章 catalog 驗 sha256**，結尾印 `N SUCCESS / 0 FAILED`。
- **不需要 root / admin** —— 工具綠色可攜。
