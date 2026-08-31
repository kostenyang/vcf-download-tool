# 常用配方

> 以下用 `<AUTH>` 代表認證旗標，替換成其中一個：
> `--depot-download-activation-code-file=actcode.txt`　或　`--depot-download-token-file=token.txt`
> 走 proxy 就在任一指令**再加** `--proxy-server=<PROXY_IP>:3128`（見 [PROXY.md](PROXY.md)）。

---

## 1. 驗證憑證（每次都先做這步）

```bash
vcf-download-tool releases list <AUTH>
```
`Depot credentials are valid.` = 成功。失敗就別往下浪費時間。

> 🔴 這裡**不要**加 `--vcf-version`（單版本 detail 有 bug → `NoSuchElementException`）。

---

## 2. 列出可下載項目、挑 bundle ID

```bash
vcf-download-tool binaries list --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL <AUTH>
```

輸出每列：`ID | Component | Full Name | Version | Release Date | Size | Type`
**ID 在最前面**，複製下來給 `--id` 用。

> 🔴 `--vcf-version` 要填 **release 版 `9.1.0.0`**，不是 `9.1.0.0400`
> （`0400` 是它底下的 patch，填了會回 **0 elements**）。

---

## 3. 下載

### A) 精準下（**建議** —— 量可控、版本不會挑錯）

```bash
vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 <AUTH> --id=<id1>,<id2>,<id3> --ceip=DISABLE
```

### B) 整批下（filter 命中的全下）

```bash
vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 <AUTH> --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --ceip=DISABLE
```

### C) 指定單一元件的特定版本

```bash
vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 <AUTH> --vcf-version=9.1.0.0 --sku=VCF --type=INSTALL --component=VCF_LICENSE_SERVER --component-version=9.1.0.0200 --ceip=DISABLE
```

> 🔴 `--component-version` 屬 `[VCF VERSION]` 群組，**單獨用不行**，一定要同時帶 `--vcf-version`。

### D) 版本範圍

```bash
vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 <AUTH> --vcf-version=9.1.0..9.1.0.0300 --sku=VCF --type=UPGRADE --ceip=DISABLE
```

---

## 4. 大檔背景跑

```bash
nohup vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 <AUTH> --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --ceip=DISABLE > /tmp/vcf-dl.log 2>&1 &
```
```bash
tail -f /tmp/vcf-dl.log
```
```bash
grep -E 'SUCCESS|FAILED' /tmp/vcf-dl.log | tail -5
```

> ⏱ 長時間下載，SSH 連線請開 **TCP keepalive**，避免被中途砍掉。

---

## 5. 下完之後

### 修權限（要給 nginx / apache 讀）
```bash
chmod -R a+rX /opt/vcf-depot/vcf9
```

### 看結構
```bash
ls /opt/vcf-depot/vcf9/PROD/
```
```bash
du -sh /opt/vcf-depot/vcf9/PROD/COMP/* | sort -h | tail -20
```

典型結構：
```
<depot-store>/PROD/
├── COMP/
│   ├── VCENTER/  NSX_T_MANAGER/  ESX_HOST/  VIDB/  VRA/  VSP/ ...
│   └── SDDC_MANAGER_VCF/Compatibility/   ← Installer sync 需要這個
└── metadata/
    ├── productVersionCatalog/v1/productVersionCatalog.json (+ .sig)
    ├── manifest/v1/vcfManifest.json
    └── ...
```

---

## 6. 補料到既有 depot（只加不動 metadata）

情境：客戶 depot 已存在、只缺幾個元件。

```bash
vcf-download-tool binaries download -d=/tmp/scratch-depot <AUTH> --id=<缺的 id 們> --ceip=DISABLE
```
```bash
rsync -a /tmp/scratch-depot/PROD/COMP/ <user>@<DEPOT_HOST>:/opt/vcf-depot/vcf9/PROD/COMP/
```
```bash
ssh <user>@<DEPOT_HOST> 'chmod -R a+rX /opt/vcf-depot/vcf9/PROD/COMP'
```

> 🔴🔴 **只搬 `PROD/COMP/`，絕對不要碰 `PROD/metadata/`**。
> 曾發生實際事故：交付 tar 含新版 metadata（seq48），客戶原樣解開把自己的
> seq43 型錄蓋掉 → installer 找不到對應版本 → 部署失敗。
> 先備份再說：
> ```bash
> tar czf metadata-backup-$(date +%F).tgz -C /opt/vcf-depot/vcf9/PROD metadata
> ```

---

## 7. 進 SDDC Manager（不透過 depot URL）

**須在 SDDC Manager appliance 上執行**：

```bash
vcf-download-tool binaries upload -d=/path/to/depot --sddc-manager-fqdn=<FQDN> --sddc-manager-user=<USER> --sddc-manager-user-password-file=<PWFILE>
```

依升級計畫產 spec，再拿去下載：
```bash
vcf-download-tool binaries create-download-spec --sddc-manager-fqdn=<FQDN> --sddc-manager-user=<USER> --sddc-manager-user-password-file=<PWFILE> --domain-name=<DOMAIN> --download-spec-dir=/tmp/spec
```
```bash
vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 <AUTH> --download-spec-file=/tmp/spec/<file>.json --ceip=DISABLE
```

---

## 8. 只要 metadata

```bash
vcf-download-tool metadata download -d=/opt/vcf-depot/vcf9 <AUTH>
```
```bash
vcf-download-tool metadata export -d=/opt/vcf-depot/vcf9
```
`export` 會把本地 metadata 打包成 tar，方便搬到 air-gap 環境。
