# VCF Download Tool — 實戰手冊

> Broadcom **VCF Download Tool**（`vcf-download-tool`）的完整用法：認證、下載、走 proxy、清資料、踩雷。
>
> 基準版本：**`9.1.0.0400.25570101`** —— 所有旗標與子命令**都是實跑 `--help` 抓的**，不是抄文件。
> 憑證（token / activation code / 密碼）一律以 `<...>` 佔位符呈現。

## 文件索引

| 文件 | 內容 |
|---|---|
| [INSTALL.md](INSTALL.md) | 取得與安裝（綠色可攜、免 admin、Windows/Linux 共用一包） |
| [AUTH.md](AUTH.md) | **token vs activation code** —— 差在哪、怎麼選、Software depot ID 流程 |
| [CLI-REFERENCE.md](CLI-REFERENCE.md) | 完整命令樹與旗標速查（實測全集） |
| [WORKFLOWS.md](WORKFLOWS.md) | 常用配方：列版本、精準下載、整批下載、背景跑、進 depot |
| [PROXY.md](PROXY.md) | 走企業 proxy、如何驗證真的有走、官方限制 |
| [CLEANUP.md](CLEANUP.md) | `binaries cleanup`、重新下載、整個重來（含備份警告） |
| [GOTCHAS.md](GOTCHAS.md) | 踩雷合輯 —— **建議先看這份** |

---

## 30 秒版

```bash
vcf-download-tool configuration get --software-depot-id
```
拿到的 UUID → `https://vcf.broadcom.com` → **Software depot Registration** → 換 activation code。

```bash
vcf-download-tool releases list --depot-download-activation-code-file=actcode.txt
```
看到 `Depot credentials are valid` 就成功。

```bash
vcf-download-tool binaries list --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --depot-download-activation-code-file=actcode.txt
```
```bash
vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 --depot-download-activation-code-file=actcode.txt --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --ceip=DISABLE
```

---

## 三件最該先知道的事

**① 至少要給一個 filter，而且三組互斥**

`binaries list` / `download` / `cleanup` 都要 filter，且以下三組**只能擇一**：

```
[VCF VERSION]   --vcf-version / --sku / --type / --component / --automated-install …
[BUNDLE ID]     --id=<bundleId>[,<bundleId>…]
[DOWNLOAD SPEC] --download-spec-file
```
混用會印 usage 並 `exit 2`。

**② 同一個 `--depot-store` 重跑是「累加」，不是「重下」**

已存在的檔會**跳過**。所以「重新下載」通常不會如你預期 → 見 [CLEANUP.md](CLEANUP.md)。

**③ token 抓不到 Compatibility 互通矩陣**

而 VCF Installer 的 offline depot sync **需要**
`COMP/SDDC_MANAGER_VCF/Compatibility/VmwareCompatibilityData.json`，
少了會卡 `Vmware compatibility data download failed` → 見 [AUTH.md](AUTH.md)。

---

## 命令樹

```
vcf-download-tool
├── binaries
│   ├── download              下載 binary + metadata
│   ├── list                  列出可下載的東西
│   ├── upload                上傳進 SDDC Manager(要在 appliance 上跑)
│   ├── cleanup               刪掉 filter 命中的 binary
│   └── create-download-spec  依升級計畫產 spec(要在 SDDC Manager 上跑)
├── metadata
│   ├── download / upload / export
├── esx
│   ├── download / export / import / metadata / configuration
├── depot                     Software depot 端管理
│   ├── binaries { upload | cleanup | list }
│   ├── metadata
│   └── esx
├── releases list             列出 VCF releases(5.0+)
└── configuration
    ├── generate --software-depot-id
    └── get --software-depot-id
```

---

## 相關

- 離線 depot 的架設與餵料：[`vcf9offlinescript`](https://github.com/kostenyang/vcf9offlinescript)
- Fleet / VSP K8s 端怎麼吃 depot：[`vsp-k8s`](https://github.com/kostenyang/vsp-k8s)
