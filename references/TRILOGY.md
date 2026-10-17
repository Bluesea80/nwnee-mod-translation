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

### batch_01 整批错位教训（10-17 二审）
试点批（apply 加固前手工对位）cn 缺行 → jrl 区块标题/正文**整体滞后一格**，且 1 条 [Resolved] 变体彻底漏译。
- 修复模式：错位有规律时 `key[i] ← value[i+1]` 整批重排 + 断言自检（见 ww/_fix_b01_shift.py）。
- **铁律：每章第 1 批必须直接用索引字典重建法**，不走手工对位。
- **铁律：候选→译表必须经「全量导出文件逐条过目」**，截断的终端输出（head/tail）不得作为建表依据——ncs 三审 24 条喊话漏译即栽在这。

### deploy 路由（bug 已修，10-17）
旧版把所有产物全投 `data\mod` → hak 落错位置。**已修**：按 src_paths 逐文件回各家：
- mod → `data\mod`
- hak → `data\hk`

部署后用 sha 校验全等再收工。

---

## WW 交付数据（2026-10-17，含收尾三件套）

| 阶段 | 数据 |
|---|---|
| 翻译 | 10 批全过，已译 2,640 条（batch_01 二审重排 + 2 条 jrl 标题补译） |
| 回填 | mod 2,533 处 / hak 3,032 处 |
| ncs 等长汉化 | 138 条 / 946 处 / 272 脚本（三审补 24 条喊话亚种变体） |
| strref 重指 | mod 1,110 + hak 205 = 1,315 处（内嵌含中文→-1） |
| verify | 剩余英文 262 uniq：全量终检=内部名（亚种匹配键 107 条为禁译区） |
| 部署 | Steam，sha 全等，⏳ 实机回归（清单见 10-17 日志） |

## 收尾三件套（WW 定版流程，SG/KM 复用）

1. **ncs 台词审计**：`ww/ncs_audit_scan.py`（[len前缀]守卫扫 .ncs，custom/official 分类）→ 人工甄别 → `ww/ww_ncs_patch.py` **等长原位补丁**（UTF-8 ≤ 原 ASCII 字节数，尾随空格填充；三校验：英文归零 / NCS 头大小字段 / 体积不变）。
   - **禁译区：亚种名匹配串**（'nomad half-elf' 等 ~200 条）——脚本比对键，翻译会弄坏亚种机制。
   - **建表必须全量导出过目**（见上铁律）。
2. **strref 重指**：`ww/ww_strref_fix.py`——gff_reembed 保留原 strref，TLK 优先场景（Examine）会显示官方英文遮蔽译文；凡 type12 strref 有效 + 内嵌含中文 → strref=-1（等长 dword 补丁）。白名单：内嵌空 / 纯 ASCII。
3. **实机回归**：浮字 / 战斗喊话 / 墓志铭拼接 / Examine 专项 / jrl 标题 / 粉笔留言拼接。

⚠️ **重建流水线（SG/KM 同口径）**：`tri.py build` → `ww_ncs_patch.py` → `ww_strref_fix.py` → `deploy`——重 build 会还原 ncs 补丁，两步不可省。

## 译名定案
- 🔒 Witch's Wake = 「女巫苏醒」（老板 10-17 裁定；官方未译，TLK row 110655 保留英文）。
- 生物/规则术语查表铁律见 NAMING.md；SRD 镜像已迁 `wiki/DND-3.5-中文SRD/topics/`。

## 待办
- ShadowGuard（4,901 条）翻译（ncs/strref 两工具改路径重扫即可复用）
- Kingmaker 批 5–47（余 5,594 条；批 1–4 已于 10-17 完成）
- WW 实机回归（清单见 10-17 日志）

---

## KM 拥王者实战（2026-10-17 晚，47 批 7,803 条全译交付）

### 子代理流水线 = 大批量翻译的正解（>5,000 条时）
1. **先固化规范再派发**：`work/trilogy/km/_TANSPEC.md` = 硬性规则（token 原样保留/引号风格/口吻/留英文判据）+ 产出流程（_mkview 视图 → _bNN_cn.py 索引字典 → apply → 自检脚本）+ **唯一裁定表（专名一对一，含禁用变体）**。规范先行是防专名分裂的**根本解**。
2. **两波并行派发**：general-purpose 子代理每 2–6 批（3–5 万字符/代理），批 1–4 主控自译垫定口吻基准。
3. **主控收货统一**：跨代理专名漂移必现（同一堡名 4 种拼法）→ 按裁定表 34 组替换对批量修正 → 全量 token/脏文本复查 → 一次 build/deploy。
4. **独立审查闭环**：交付后跑专名一致性全扫（本例 3 组分裂 28 处，见 `三部曲汉化/docs/KM专名分裂修正清单.md` 口径），修正后重 build+两补丁+deploy。

### 索引字典重建法（每批标配）
- `_mkview.py km NN`：批次 → 带序号视图（#N [已译/待译]）。
- `_bNN_cn.py`：`T = {view序号: "译文"}`（只含空值条目）→ walk 批结构产出 cn 文件；**dict 值必须纯字符串字面量**（见坑 21）；文件尾带 unseeded 校验段（missing/extra 双向断言）。
- 收尾自检三连：未填=0 / `'replace(' not in v` / token 数一致。

### 收尾两件套（km 版，SG 同口径复用）
- `trilogy/km/ncs_audit_scan.py` + `km_ncs_patch.py`：ncs 台词审计（KM 仅 17 条/26 处/17 脚本，全 mod 层）+ 等长 UTF-8 中文+空格填充（译文须 ≤ 原文字节数，先跑等长校验）。
- `trilogy/km/km_strref_fix.py`：strref 重指（内嵌含中文且 strref 有效 → -1；KM 实测 mod 799 + hak 138 = 937 处）。
- ⚠️ **坑15 的 fix_strref_mismatch.py 不适用三部曲**：那是 Aielund「同 strref 复用」场景的专用工具；tri.py 按英文原文精确匹配回填，不依赖 strref 对位。三部曲需要的是上面的**泛化重指**（防 TLK 遮蔽），两者别混。

### 重建流水线（三部曲通用）
```
tri.py build <name> → <name>_ncs_patch.py → <name>_strref_fix.py → tri.py deploy <name>
```
重 build 会还原 out/ 层两补丁，中间两步不可省。

## 译名定案（KM，🔒 老板/审查裁定）
- Banner=**旗主**、Hand=**辅手**（10-17 老板裁定）。
- Cyan=赛恩（赛恩堡/北赛恩/南赛恩）、Dynschall=丁斯考尔、Alias=艾莉丝、Gelani=杰拉妮、Thaddeus=塔迪厄斯、Serah=瑟拉、Levio'Sa=莱维欧莎、Jaess=雅斯、The Stan=老斯坦。
- Dylan=**迪伦**（弃卓兰/狄兰）、Manticore=**蝎狮**（弃蝎尾狮/曼提柯尔/曼提克尔/毒之鞭）、Intellect Devourer=**噬智魔**（SRD 官方）。
- 行会两家勿混：Trades Guild=贸易行会/The Trades=贸易行（老斯坦）vs Merchants Guild=商人工会（杰拉妮→塔迪厄斯）。

## 待办（更新）
- ~~Kingmaker~~ ✅ 全译交付（7,803/7,803，含两补丁）⏳ 实机回归
- ShadowGuard（4,901 条）：开工前**一次列全高频专名入裁定表**（防分裂根本解）；ncs/strref 两工具改路径重扫即可复用
- WW/KM 实机回归

---

## ncs 台词终极口径（10-18 定案：四层盲区 + 双口径终检）

> SG/KM/WW 实战迭代四轮后定版。**任何 ncs 扫描/终检代码禁止再用单字节守卫。**

### 四层盲区（每层都是真漏过译的）
1. **单字节守卫漏长段**：NCS CONST_S 是 **2 字节大端长度前缀**；串长 32–126B 时前缀低位是可打印 ASCII，被 `[\x20-\x7e]` 正则吞进串、守卫失败 → 整常量丢弃。三部曲因此漏 421 条（SG184/KM43/WW194）。
2. **资源类型误判**：ncs 实际 rtype=**1323**（非 2010）——过滤一律用 magic `b'NCS '`（`lib/ncs_strings.is_ncs`），勿按 rtype。
3. **同句多形态**：同一逻辑句在不同脚本以「整段 / ~146B 拆段 / 前后缀碎片」多种形态存放；整句 key 只命中一种形态。**同句必须按实际常量形态逐个建 key**（用 `ncs_audit/_custom_all.json` 精确串，勿 trim）。
4. **混合常量免疫**：短 key 命中后常量变「中文+英文残段」，长度前缀对应含中文整串 → 任何长度守卫对 ASCII 残段失效。检测：ASCII 段且前邻字节 ≥0x80（中文 UTF-8 尾）。

### 工具链（现行版）
- `work/lib/ncs_strings.py`：`iter_strings()` 2 字节大端解析（形态1 前缀完整 / 形态2 前缀低位被吞）。
- `work/_ncs_gate.py`：三部统一终检（2 字节口径），残留须逐条解释（dev/SKIP 属正常）。
- `work/trilogy/_ncs_patch2.py` / `_ncs_patch3.py`：漏译清欠补丁（直接作用于 out/，无需重 build；patch3 处理混合残段）。
- 混合残段扫描脚本：见 TRILOGY 流程（ASCII 段 + 前邻 ≥0x80 + like_speech）。

### 终检双口径（缺一不可）
1. 2 字节 gate：残留逐条解释（dev/引擎标记/at_tester 断言/TourNode 键 Isle of the Dead·Tree of Life/DM 限定/SKIP <12B 碎片——均不译）。
2. 混合残段扫描：应为 0。
**铁律：终检先验证「能抓到已知残留」再信 0**（rtype 事故教训）；终检对象必须是**当前部署 sha 的产物**——审查/复扫引用旧 sha 数据会得出假结论（10-18 实例）。

### DEV 不译清单（审查定案）
TourNode 状态机键（Isle of the Dead / Tree of Life）、at_tester 断言、`<<...>>`/`[全大写]` 引擎标记、DM 限定文本、`should be` 开发者注释、<12B 拼接碎片。
