# 三部曲（premium）汉化流水线 · mod + hak 双层

Kingmaker / ShadowGuard / Witch's Wake 是 NWN:EE 的 premium 模组，文本同时落在 **.mod（GFF 内嵌）** 与 **.hak（资源包）** 两层，且自带独立 `server.tlk`。与 Aielund 主线工具链不同，走 `三部曲汉化/work/tri.py` 专用七命令。

- 事实源：`三部曲汉化/三部曲译名表.md`
- 官方种子：`三部曲汉化/work/server_tlk_utf8.tsv`（星空娱动官方 server.tlk，GBK→UTF-8，EN 列精确匹配 116,150 行）
- build 源 = **备份原版** `backups/premium_trilogy_20261017/`（防 erf 追加膨胀，见坑 3 反面提醒）

---

## 七命令总览

```bash
PY=...python3            # 见 SKILL.md §0
cd 三部曲汉化/work

# ① extract：从原版 mod+hak 抽取 CExoLocString → uniq（去重）
$PY tri.py extract <name>

# ② seed：官方种子回填——server_tlk_utf8.tsv EN 精确匹配的条目直接带出官方译文
$PY tri.py seed <name>

# ③ batches：切批 → batch_NN.json（结构含索引与来源信息，不只是裸文本）
$PY tri.py batches <name> [chars]

# ④ apply：行序/索引配对写回批结构（配合手工 cn 文件）
$PY tri.py apply <name> <NN>

# ⑤ build：从备份原版重建 mod+hak（双层回嵌 + 打包）
$PY tri.py build <name>

# ⑥ verify：结构验收 + 残留英文清单（剩余须逐条裁定是否内部名）
$PY tri.py verify <name>

# ⑦ deploy：部署到 Steam 客户端目录（sha 校验）
$PY tri.py deploy <name>
```

---

## 关键经验（WW 实战，2026-10-17）

### 批次对位：重建法，免疫物理行错位
不要"逐行对 cn 文件"。用**索引字典**：先从批结构建 `T[i] → 原文`，翻译产出 `{i: 译文}`，再 **walk 批次结构** 按 i 产出 cn 文件。物理行序错乱（空行、注释、换行差异）不再影响配对。

### E 集（内部名）逐条裁定
verify 后剩余英文 = E 集，**不能整集盲留**：逐条看内容裁定是「内部名/脚本键（留英文）」还是「真漏译（补）」。WW 交付时剩余英文全为内部名才算干净。

### token 数校验必做
译文 `<`/`>` 占位符数量必须与原文一致（`<FirstName>`、`<StartAction>` 等）——WW 实测靠这条抓出漏 token。

### deploy 路由（bug 已修，10-17）
旧版把所有产物全投 `data\mod` → hak 落错位置。**已修**：按 src_paths 逐文件回各家：
- mod → `data\mod`
- hak → `data\hk`

部署后用 sha 校验全等再收工。

---

## WW 交付数据（2026-10-17）

| 阶段 | 数据 |
|---|---|
| 翻译 | 10 批全过，已译 2,639 条 |
| 回填 | mod 2,532 处 / hak 3,032 处 |
| verify | 剩余英文全为内部名（E 集裁定通过） |
| 部署 | Steam，sha 全等，⏳ 实机回归 |

## 待办
- ShadowGuard（4,901 条）/ Kingmaker（7,803 条）翻译
- ncs 审计（见 PITFALLS 坑 18）
- fix_strref_mismatch 的 hak 版适配
- 「女巫苏醒 vs 女巫觉醒」Witch's Wake 译名待老板复核
