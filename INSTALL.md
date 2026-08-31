# 取得與安裝

## 特性

- **綠色可攜** —— 解開就能跑，不用安裝
- **免 admin / root**
- **自帶 JRE** —— 同一包內含 `jre/lin64` 與 `jre/win32`，Linux / Windows 共用

## 目錄結構

```
<tool-root>/
├── bin/
│   ├── vcf-download-tool          ← Linux
│   ├── vcf-download-tool.bat      ← Windows
│   ├── lcm-bundle-transfer-util
│   └── lcm-bundle-transfer-util.bat
├── conf/                          ← properties(含 proxy 設定)、logback、憑證
├── jre/
│   ├── lin64/
│   └── win32/
├── lib/                           ← lcm-tools-uber.jar 等
├── esximage/
├── osl/
└── log/                           ← vdt.log 在這
```

## 安裝

### Linux
```bash
tar xzf vcf-download-tool-<版本>.tar.gz -C /opt/vcf-depot/
```
```bash
chmod +x /opt/vcf-depot/<tool-root>/bin/vcf-download-tool
```
```bash
/opt/vcf-depot/<tool-root>/bin/vcf-download-tool --version
```

### Windows
解壓到任意路徑（例如 `C:\VCF9`），直接跑：
```bash
C:\VCF9\bin\vcf-download-tool.bat --version
```

## 驗證可用

```bash
vcf-download-tool --version
```
```bash
vcf-download-tool --help
```

應輸出版本（例如 `9.1.0.0400.25570101`）與命令清單：
```
Commands:
  binaries       Management of the binaries files within the system.
  configuration  Manage vcf-download-tool configuration.
  metadata       Management of the metadata files within the system.
  releases       Operations related to the VCF releases.
  esx            Management of ESX binaries and depots.
  depot          Software depot management commands
```

## Log

```bash
tail -f <tool-root>/log/vdt.log
```
每次執行結尾也會印出 log 路徑。

## 包裝腳本的額外旗標

`bin/vcf-download-tool` 本身是 wrapper，支援：

| 旗標 | 說明 |
|---|---|
| `--jre-home` | 指定 JRE |
| `--path` | 指定路徑 |
| `--unix` / `--windows` | 平台 |
| `--sddc-manager-fqdn` | SDDC Manager |
| `--ignoreConsent` | 略過同意提示 |

一般用不到，直接跑就好。

---

## 從 Windows 搬到 Linux 目標機

```bash
scp vcf-download-tool-<版本>.tar.gz <user>@<HOST>:/opt/vcf-depot/
```
```bash
ssh <user>@<HOST> 'tar xzf /opt/vcf-depot/vcf-download-tool-<版本>.tar.gz -C /opt/vcf-depot/'
```

> 🔴 **要用 activation code 的話，順序很重要** ——
> 必須**先把工具放上目標機**，再在**那台機器**上 `configuration generate --software-depot-id`，
> 因為 code 綁該 ID。在自己筆電產的 ID 換到的 code，拿到目標機**無效**。
> 細節見 [AUTH.md](AUTH.md)。

## 隨手腳本

從 Windows 取得的 `.sh` 記得先轉 LF：
```bash
sed -i 's/\r$//' *.sh
```
