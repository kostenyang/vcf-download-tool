# 清資料與重新下載

## 先理解一件事：`download` 是「累加」不是「重下」

同一個 `--depot-store` 重跑，**已存在的檔會被跳過**。
所以「我重下一次」通常什麼都不會發生 —— 這是最常見的誤解。

| 你的目的 | 該做什麼 |
|---|---|
| 檔案壞掉 / 下到一半 | `cleanup` 那幾個 `--id` → 再 `download` |
| 想換版本 | `cleanup` 舊版 → `download` 新版（直接 download **不會**刪舊的） |
| 只是想確認完整性 | **不用重下** —— 重跑會驗 sha256 並補缺檔 |
| 真的要全新 | `rm -rf` 目錄（**先備份 metadata**） |

---

## `binaries cleanup`

```bash
vcf-download-tool binaries cleanup --help
```

**語意：刪掉 filter 命中的 binary**，不是「保留命中的、刪掉其他」。

- filter 規則與 `download` **完全相同**（`[VCF VERSION]` / `[BUNDLE ID]` / `[DOWNLOAD SPEC]` 三選一互斥）
- **至少要給一個 filter**
- **沒有 `--all` 之類的清空旗標**
- **不需要**認證檔（純本機檔案操作，help 裡沒有 `[DEPOT]` 群組）

### 刪特定 bundle（最精準）
```bash
vcf-download-tool binaries cleanup -d=/opt/vcf-depot/vcf9 --id=<id1>,<id2> --ceip=DISABLE
```

### 刪某個元件
```bash
vcf-download-tool binaries cleanup -d=/opt/vcf-depot/vcf9 --vcf-version=9.1.0.0 --sku=VCF --type=INSTALL --component=VCENTER --ceip=DISABLE
```

### 刪整組 release 命中的
```bash
vcf-download-tool binaries cleanup -d=/opt/vcf-depot/vcf9 --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --ceip=DISABLE
```

### 清完重下
同一組 filter 換成 `download` 再跑一次：
```bash
vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 --depot-download-activation-code-file=actcode.txt --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --ceip=DISABLE
```

> 💡 第一次用 `cleanup` 建議**先拿一個不重要的 `--id` 試**，確認刪除範圍符合預期再放大。

---

## 整個重來

因為沒有清空旗標，最乾淨可靠的做法是直接砍目錄：

```bash
rm -rf /opt/vcf-depot/vcf9 && mkdir -p /opt/vcf-depot/vcf9
```
```bash
vcf-download-tool binaries download -d=/opt/vcf-depot/vcf9 --depot-download-activation-code-file=actcode.txt --vcf-version=9.1.0.0 --sku=VCF --automated-install --type=INSTALL --ceip=DISABLE
```
```bash
chmod -R a+rX /opt/vcf-depot/vcf9
```

### 🔴 砍之前務必確認兩件事

**① 這個 depot 是不是正在服務中**
若 VCF Installer / Fleet 正指著它，砍掉期間會抓不到東西。先確認沒有進行中的
bring-up / upgrade / sync。

**② metadata 是不是客戶原始的**

```bash
tar czf /root/metadata-backup-$(date +%F).tgz -C /opt/vcf-depot/vcf9/PROD metadata
```

這正是實際發生過的事故根因：交付包含了新版 metadata（seq48），客戶原樣解開，
把自己原本的 seq43 型錄**蓋掉** → installer 要的版本在型錄裡不存在 → 部署失敗。

**教訓：補料只搬 `PROD/COMP/`，`PROD/metadata/` 一律保留客戶原本的。**

---

## 空間管理

```bash
du -sh /opt/vcf-depot/vcf9/PROD/COMP/* | sort -h | tail -20
```
```bash
df -h /opt/vcf-depot
```

參考量體（實測）：
- 指定 2 個元件（VIDB + VRA）≈ **32 GB**
- 全部 17 個元件 ≈ **130 GB**
- `--automated-install` 的 16 元件 ≈ **66 GB**

> 下載前先確認空間 —— 這是最容易忽略、也最痛的一步。

---

## 驗證下載完整性

不用清掉重來，**重跑同一條 download 指令**即可：
- 已存在且 sha256 正確的 → 跳過
- 缺的 / 壞的 → 補下

結尾會印 `N SUCCESS / 0 FAILED`。
