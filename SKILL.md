---
name: nwn-mod-translation
displayName: NWN:EE 模组汉化流水线
description: NWN:EE（无冬之夜增强版）模组（.mod/.erf/.hak）简体中文汉化的完整工程流水线——抽取 GFF 内 CExoLocString、按批翻译、字典式回填、结构级验收。覆盖对话(.dlg)、非对话(uti/utc/are/jrl/ifo/git/utp/ute/utt/utw)与premium三部曲(mod+hak双层)三条线。当需要继续/复查 Aielund Saga 任意章节、做 Kingmaker/ShadowGuard/Witch's Wake 三部曲汉化，或对任何 NWN:EE 模组做同类汉化时使用。实测来自 Aielund Saga 全 7 章汉化全闭环 + 三部曲 Witch's Wake 交付。
agent_created: true
version: 1.6.0
summary: NWN:EE 模组简体中文汉化流水线——抽取 GFF CExoLocString、按批翻译、字典式回填、结构级验收。对话 + 非对话 + 三部曲(mod+hak)三条线，含四铁律、引擎 token 保护、译名纪律（官方TLK→SRD 查表铁律）、TLK/strref 速查与 23 个坑（strref 复用错配 / 改译双写 / 审计假阳性 / .ncs 台词 / 共享临时文件 / 整批错位 / 截断输出建表 / 子代理专名漂移 / dict 值表达式污染 / 关键字批量修误伤）。**三部曲全交付：WW 2,640 / KM 7,803 / SG 4,901 条**；大批量翻译用子代理流水线（规范+唯一裁定表先行→并行派发→主控统一）。ncs 台词四层盲区（单字节守卫/rtype/同句多形态/混合常量）已根治，终检=双口径（2 字节大端 + 混合残段），工具 lib/ncs_strings.py + _ncs_gate.py + _ncs_patch2/3 定版可复用。23+4 个坑。
---

# NWN:EE 模组汉化流水线

工程实证来自 **Aielund Saga 全 7 章汉化**（Act I / II / III / IV Pt1·Pt2·Pt3 / Finale），全部交付并验收 **结构错误 0 / CExoString 变更 0**；以及 **premium 三部曲**（Kingmaker / ShadowGuard / Witch's Wake，mod+hak 双层，WW 已交付，见 references/TRILOGY.md）。本技能是这套流程的**操作手册 + 指挥中枢**；底层工具链（纯 Python）自包含在 `汉化问题/work/` 与 `三部曲汉化/work/`，本技能直接调用它们，不复制代码（避免腐化）。

## 0. 工作目录与运行环境

- **工作区根**：`汉化问题/` 的上一级（即 NWNEE 项目根）。
- **工具链**：`汉化问题/work/*.py`（脚本靠 `HERE = dirname(__file__)`、`BASE = dirname(dirname(HERE))`、`MODS = BASE/MOD/modules` 自定位，彼此用 `sys.path.insert` 兄弟导入，无需改路径）。三部曲工具链在 `三部曲汉化/work/tri.py`。
- **模组目录**：`汉化问题/MOD/modules/`。
- **译名事实源**：Aielund=`汉化问题/work/译名表.md`；三部曲=`三部曲汉化/三部曲译名表.md`（见 references/NAMING.md）。
- **Python**（两台机器实测，均为 UTF-8 原生 3.13）：
  - macOS：`/Users/sea/.workbuddy/binaries/python/versions/3.13.12/bin/python3`
  - Windows：`C:\Users\zhou_\.workbuddy\binaries\python\versions\3.13.12\python.exe`
- ⚠️ **Mac 上不装游戏 → 不回写创意工坊目录**（老板指示）。游戏内通读需真实客户端实测（见 HANDOFF「仍待验证」）。

> 用变量简化命令（按机器选一个）：
> ```bash
> # macOS
> cd /Users/sea/WorkBuddy/game/NWNEE/汉化问题/work
> PY=/Users/sea/.workbuddy/binaries/python/versions/3.13.12/bin/python3
> # Windows (Git Bash)
> cd /d/WorkBuddy_Workspace/game/NWNEE/汉化问题/work
> PY=/c/Users/zhou_/.workbuddy/binaries/python/versions/3.13.12/python.exe
> ```

---

## 1. 对话（.dlg）汉化流水线

故事文本是玩家真正阅读的内容，优先做。**完整流程 = ①→⑧，⑦ fix_strref_mismatch 与 ⑥ backfill 绑定，缺一步都会留缺陷**。

```bash
# ① 抽取：仅取 .dlg 的自定义(strref=-1)英文，去重 → work/<ch>/uniq.json + stats.txt
$PY dlg_uniq.py <ch> "<原版 mod 文件名>"

# ② 切批：默认 12000 英文字符/批，同一来源 NPC 不跨批；自动剔除官方通用对话
$PY mkbatches.py <ch> [每批字符数]

# ③ 跨章复用（TM）：必须【在 mkbatches 之后】跑；命中条目塞进 batch_00.json，并从各批剔除
#    Act III 用 _act3_reuse.py；Act IV 用 _act4_reuse.py（TM 池含前几章）
$PY _act4_reuse.py        # 或 _act3_reuse.py

# ④ 翻译：手写 work/<ch>/cn_NN.txt —— 一行一条，行数 == batch_NN.json 键数，空行=留英文
#    （用 Python 生成最稳，带 assert len(keys)==len(values) 校验，见下方"译法"）

# ⑤ 填回：apply_cn.py 会把中文【原地覆盖】写回 batch_NN.json
$PY apply_cn.py <ch> NN            # NN 两位，如 04

# ⑥ 全量回填 + 追加式打包 → 产出 "<mod>(汉化).mod" + <ch>/_backfill_report.txt
$PY backfill.py <ch> "<原版 mod 文件名>" --all        # ⚠️ 后台跑，见 §7

# ⑦ 重指被复用的官方 strref（坑 15）——【每次 backfill 后必须重跑】
$PY fix_strref_mismatch.py "<汉化 mod 文件名>"        # 先 --dry-run 核对数量

# ⑧ 结构级验收（铁律：结构错误 0 且 CExoString 变更 0）
$PY verify_struct_all.py "<原版 mod 文件名>" "<汉化 mod 文件名>"
```

⚠️ **`dlg_uniq.py` 本体仍是旧口径**（只抽 strref=-1）：若要**重做**某章对话，必须先改成「凡 lang=0 非空且非中文即提取」（把坑 9 的「strref 有效 + 内嵌英文 substring」整类也抽进来），否则重做会整类漏抽。

**对话翻译写 `cn_NN.txt` 的契约**
- 第 i 行 ↔ `batch_NN.json` 第 i 个键（严格行序配对）。
- **行数必须 == 键数**，否则 `apply_cn.py` 报错。
- **空行 = 该条留英文不译**（内部标识、引擎 token 等）。
- 英文原文含引号/换行/特殊符号 → 用行序文本零转义风险（**别写 JSON**）。
- `apply_cn.py` 内 `protect_tokens()` 会自动还原引擎占位符（见 §4），译文中误写的 token 也会被还原。

**译法建议**（避免漏条/错位）：不要手敲长清单——用 Python 先把 `batch_NN.json` 的键 dump 出来，逐条翻译后按 `dict(zip(keys, values))` 写回，并 `assert len(keys)==len(values)`；用"应留空集合"双向校验（哪些 key 故意不译）。三部曲批次用「索引字典 T[i]→walk 批次结构产出 cn 文件」的重建法，免疫物理行错位（见 references/TRILOGY.md）。

---

## 2. 非对话（nondlg）汉化流水线 ⚠️ 易错

覆盖 `uti/utc/are/jrl/ifo/git/utp/ute/utt/uts/utw` 等玩家可见文本。**完整 7 步，缺 `split_nondlg_remain` 这一步是常见翻车点**。详见 references/NONDLG.md。

```bash
# ① 抽取：所有非 dlg 资源的 CExoLocString，按类型落盘 → nondlg_<ext>.json + _nondlg_report.txt
$PY extract_nondlg.py <ch> "<原版 mod 文件名>"      # 可加 --sample 30 看样本

# ② 跨章复用：命中项剔除，剩 nondlg_<ext>_remain.json
$PY _act4_nondlg_reuse.py          # 或 _act3_nondlg_reuse.py

# ③ 拆单/多行：remain → nondlg_<ext>_single.json(无\n) + nondlg_<ext>_multi.json(含\n)
$PY split_nondlg_remain.py <ch>

# ④ 译单行：手写 cn_<ext>.txt —— 行序配对 nondlg_<ext>_single.json，空行=留英文
#    ⚠️ 末条若应留空，必须显式多写一个空行（脚本只 pop 文件末尾那个换行产生的空元素）

# ⑤ 译多行：写 nondlg_<ext>_multi_cn.json ——【JSON dict，键=原文，值=译文】（不是行序！）
#    {"<原文整串>": "<译文>", ...}

# ⑥ 填回：apply_nondlg.py 需 4 个文件齐备 → nondlg_<ext>_filled.json
$PY apply_nondlg.py <ch> <ext>      # ext 如 uti / git / utc / jrl / are / ifo / ute / utp / uts / utw

# ⑦ 回填 + 验收（同对话 §1 ⑥⑦⑧，backfill 会一并读所有 *_filled.json + batch_*.json）
```

**非对话关键坑**
- `cn_<ext>.txt` 行数 == `nondlg_<ext>_single.json` 键数；**末尾有意留空的 key 必须显式写空行**（同对话的末行空行陷阱放大版——曾因漏 7 条 tech token 空行导致整文件行数不符）。
- 内部标识（如 `git` 的 `Alarm`/`CASTER`/`FIREPOINT`/`LUCAS`/`TWILIGHT_MERCHANT`/`Zap point`/`beam1..4`/`beampoint`/`bloodsound`，`utp`/`uts`/`utw` 大量内部键）**按规矩留英文空行**，不要翻。
- 多行用 **dict keyed by 原文**，不是行序；`apply_nondlg.py` 按原文字典匹配。
- 🔴 **改译必须"双写"（坑 16）**：`backfill.py` 实际消费的是 `nondlg_<ext>_filled.json`（生成物），不是 `cn_<ext>.txt`（源）。改定案时**两边都要改**——只改 cn 源不改 filled.json，重跑 backfill 会把**旧值**写回成品（act4p3 area 名漏"界"×11 实锤）。对话线同理：`cn_NN.txt` 与 `batch_NN.json` 双写。

---

## 3. 四条铁律（回嵌器正确性的根基）

1. **GFF 只翻 type 12 CExoLocString。** type 10 `Tag` 是 `GetObjectByTag()` 查找键、`Speaker` 是说话动画的 Tag 匹配键——**都不是显示文本**。回嵌器 `_build_fd` 必须 `if ft == 12`。翻掉 type 10 = 脚本引用失配、NPC 点不动。
2. **判断"要不要译"看 substring，不看 strref。**「strref 有效 + 内嵌英文 substring」整类必须译。⚠️ 反向同样成立（坑 15）：**strref 有效 ≠ TLK 内容就对**——作者常复用官方 strref 当"万能描述槽"，内嵌了完全不同的文本；内嵌译完后还必须重指 StrRef（见坑 15），否则 TLK 优先的客户端仍显示错配的官方中文。
3. **不能按 strref 查官方中文 TLK 自动套译文。** 命中率 99% 但语义大量错配（标准物名可用、自创名一律手译）。官方中文包只作参考（检索工具 `_q_tlk2.py`，字节级）。
4. **整串精确匹配回填的三处前置修正**：多语言 substring 用 `_pick_src()`（langid 0 非空优先）；`backfill.load_mapping` 自动补英文双引号变体别名；对不含尾空格的 key 生成 `s + ' '` 变体（mod 实际子串常带尾空格）。

---

## 4. 引擎层硬约束（token 保护）

- GFF 内嵌中文 = **UTF-8 + langid 0**（目标运行时 `.37` 版定案；不要套用 `.20` 版的 GBK/langid 130）。
- 以下引擎 token **译成中文会原样显示字面量**，必须保留英文（译文中误写也会被 `apply_cn.py::protect_tokens()` 还原）：
  `<sir/madam>` `<man/woman>` `<FirstName>` `<FullName>` `<Class>` `<race>`
- 结构性标签 **标签名保留英文、内容可译**：`<StartAction>…</Start>`、`<StartCheck>…</Start>`。
- 技能/阵营名保持英文：`[Persuade]` `[Insight]` `[Lawful]` `[Evil]` `[Chaotic]` `[Spellcraft]` `[Lore]` `[Success]` `[Failure]` `[Bluff]` `[Intimidate]` `[Loot]`。
- 玩家要照抄输入的值（如 `Sub-race Syntax:` 之后）保留英文。
- **token 数校验必做**：译文里的 `<`/`>` 占位符数量必须与原文一致（三部曲 WW 实测抓出漏 token）。

---

## 5. 验收铁律（交付前必过）

- **结构错误 0 且 CExoString 变更 0** —— 两者都必须为 0，缺一不可。
- **成品级复扫归零**（verify 之外的独立一道）：任何修复/回填后，用独立 GFF 解码扫描器对 `(汉化).mod` 逐资源抽取 CExoLocString，统计目标错串命中——必须 = 0，正确译文落位 > 0（act4p3 漏改 filled.json 就是复扫抓出来的，verify 抓不到——它只比对结构不比对内容对错）。
- 校验器**必须独立于被测实现**（曾因复用被测代码解析器而"自证清白"，漏掉 List 结构崩坏）。
- **空翻译重建体积必须与原版完全相等** → 证明打包无副作用（append-only，不重写 ERF 头/keylist/loc/未改资源）。
- `backfill.py` stdout 只打印报告**最后 6 行**；完整命中分布读 `<ch>/_backfill_report.txt`。
- `verify_struct_all.py` 传参**只给文件名**（相对 `MOD/modules/`），传 `../MOD/modules/x.mod` 会拼成双份路径 → `FileNotFoundError`。
- **PASS ≠ 译全**：审计必须覆盖 **dlg + ncs** 两条线（`.ncs` 编译脚本里的台词是独立缺陷面，见坑 18；成品级审计工具 `work/_audit_*_en.py`）。
- 三部曲 hak 侧残留英文须逐条裁定：verify 剩余英文若全为**内部名**（脚本键/资源名）才算干净。

---

## 6. 译名纪律（唯一事实源：译名表）

详见 references/NAMING.md。要点：
- **翻译前先读译名表**（Aielund 用 `work/译名表.md`，三部曲用 `三部曲汉化/三部曲译名表.md`），别凭印象写专名（同一专名会在不同批次反复被引进同一种错写）。
- **查表铁律（三部曲定案，10-17）**：生物/规则术语**先查官方 TLK**（`三部曲汉化/work/server_tlk_utf8.tsv` EN 列精确匹配，实测 116,150 行），官方无再查 3.5 SRD 中文镜像。devil=魔鬼 / demon=恶魔；官方定名：Rakshasa=邪兽鬼、Invisible Stalker=隐形潜伏怪、Dire=凶暴、Huge=巨型、Greater=超大型（前缀）。详见 NAMING.md §查表铁律。
- **审计/修复判定前也先查译名表判决**：表里的禁令（如 §七之七 `Scimitar→弯刀` 不变、**禁对"弯刀"盲替换**）优先于工具输出——`_name_audit.py` 不区分 `Scimitar(弯刀)` 与 `Falchion(弯刃大刀)`，纯子串计数会把正确译名当错报（129 处假阳性实锤）。**专名一致性必须按英文键分类判定，不能按中文子串计数。**
- 改定案三步顺序：**先改表 → 再批量替换 `cn_*.txt` 与 `batch_*.json`（两处都要改）→ 最后重新 backfill**。
- ⚠️ 非对话改译是**三处同改**：`cn_<ext>.txt`（源）+ `nondlg_<ext>_filled.json`（回填消费物）+ 译名表（事实源）——漏 filled.json 就是坑 16。
- 每章译完 / 任何批量改名后跑一次 **`_name_audit.py <ch...>`** 复验；跨章复用池引入的错译用 **`_variant_scan.py`** 抓。
- 子串假阳性陷阱：「影德鲁伊」⊂「暗影德鲁伊」、「陈」⊂「陈先生」、「游侠」⊂「游侠团」、「晨」常在「清晨/晴晨」里——必须逐条看原文再判。

**修复已交付成品的标准流程**（高影响，先备份再动）：
```bash
# ① 备份：受影响 (汉化).mod + 整个 work/ 源 → backups/<日期>_<事由>/
# ② 按英文键定向改（禁中文子串盲替换）→ cn 源 + *_filled.json 双写
# ③ 重跑 backfill（后台！61MB 约 5 分钟）
# ④ backfill 后必须重跑 fix_strref_mismatch.py（见坑 15）
# ⑤ verify_struct_all + 成品级 GFF 复扫归零 + 审计报告勘误留痕（误报作废要在报告里写明）
```

---

## 7. 环境坑（跨机实测：macOS + Windows）

- **PowerShell stdout 常被吞** → 脚本结果一律 `| tee`/`Out-File` 落盘再 Read，别信回显。
- **PowerShell 内联 python 的 `%s` 会被当 cmd 变量语法拦下** → 写脚本落盘再执行，别用 `-c` 含 `%` 的代码。
- ⚠️ **`backfill.py` 启动即一次性 load 全部 `batch_*.json` / `nondlg_*_filled.json`**。回填**运行期间**才 `apply_cn` 写入的批**不会进本次成品** → 必须等回填结束再重跑一遍；终态校验务必等**最后一次**回填跑完再 `verify_struct_all`。
- ⚠️ **34MB 整包重嵌前台 120s 超时会被 SIGKILL** → 后台跑（`run_in_background` / `> log 2>&1 &`），约 5 分钟。
- 抽查成品别用"通用专名"当探针（假阳性）——挑**该批独有**字符串，用 Python `b.find(s.encode('utf-8'))` 查二进制 mod（grep 对二进制不可靠）。
- ⚠️ **`erf_pack` 追加式重建会累积膨胀**（CEP 注入实测）→ 大规模重做必须**从干净备份出发**，不要在已膨胀的成品上继续追加（回滚点 `MOD/modules_bak_cep_20261013/`）。
- ⚠️ **临时文件必须唯一名（坑 19）**：多资源循环里共享一个临时文件名，一旦某轮异常退出，下一轮会读到上轮残留 → 439 个 utc 被写成同一只半龙（10-13 实锤，已重建）。

---

## 8. 交付前全批审计（防止漏译 shipped English）

曾因未审计导致 `batch_46` 整批 64 条漏译上包。最终交付前必须跑**全批审计**：

```bash
$PY -c "
import json, os
def cjk(s): return any('一'<=c<='鿿' for c in s)
for i in range(57):                       # 按本章批数调整
    b='%02d'%i; fn='batch_%s.json'%b
    if not os.path.exists(fn): continue
    d=json.load(open(fn,encoding='utf-8'))
    tot=len(d)
    cn=sum(1 for v in d.values() if isinstance(v,str) and v.strip() and cjk(v))
    empty=sum(1 for v in d.values() if not (isinstance(v,str) and v.strip()))
    print(b,'tot',tot,'cn',cn,'empty',empty)
# 非对话同样审计 nondlg_*_filled.json 的 translated 数
"
```

- 所有批次应 **100%（0 empty）**，除了**故意保留的引擎 token 键**（如 `batch_00` 里的 `<FullName>`/`<CUSTOM99xx>` 等，确认是 token 而非漏译）。
- `nondlg_*_filled.json` 同理：translated 数应 == 实译数（内部标识留空的除外）。

**成品级残留英文审计**（`audit_residual_en.py <mod>`，对成品而非源跑）：
- 扫三类：资源类型覆盖缺口 / 已覆盖 GFF 类型残留英文成句 / 脚本资源可见英文。
- ⚠️ **命中 ≠ 漏译，必须 strref 级复核真伪**（坑 17 教训）：
  - `strref=-1` 内联自定义串 → **确定真漏译**，必须补；
  - `strref` 有效且官方 TLK 内容一致 → TLK 兜底，非缺陷；
  - `strref` 有效但 TLK 内容**错配**（作者复用官方 strref）→ 按铁律 2 补译内嵌 + **重指 StrRef**（坑 15）；
  - `strref` 超出 TLK 范围 → 确定真漏译。
- 技术串有意不译：作者署名（`volition's Bio Placeables…`）、隐形工具容器、开发残留名（报老板定夺）。
- ⚠️ ③ 脚本段只认 `.nss` 源码；成品全是编译后 `.ncs`（0 源码）→ 脚本英文是**未证实风险**，不是已确认漏译；但 **`.ncs` 台词须单独审计**（坑 18，工具 `work/_audit_*_en.py`）。
- ⚠️ ② 显式跳过 `.dlg` → 「②=0」不代表对话干净。

---

## 9. TLK 与 strref 速查（2026-09/10 定论）

- **规模**：英文 TLK=112,228 条；中文 TLK=112,526 条（多出的 298 条为中文专属区，**英文客户端越界读**）。
- **strref 编码**：`strref = (tlk_index << 20) | row`。`tlk_index`：`0`=dialog.tlk、**`16`=cep260.tlk（CEP 内容都在这）**、`4095`=空哨兵（实测占 56%，**必须过滤**）。
- **TLK V3.0 格式**：entry 的 offset 字段（+28 处）**相对字符串区起点**，不是相对文件头。判据：row 0 应为 `Bad Strref`。
- **Examine 覆盖范围**（实测）：NPC 标题❌ / NPC 描述✅ / placeable 标题✅——**不可跨类型外推**。
- `SetTlkOverride` 传空串 = 恢复默认；要"清空"某条必须写一个空格。
- `myriad_*.tlk` 是老服自定义表，**不是官方 TLK**，别拿它当基准。
- `encoding.2da` 与本工程无关；中文显示走官方语言包。

---

## 10. CEP 物品名注入（方案 A，2026-10-13 交付）

CEP（Community Expansion Pack）内容物名存于 cep260.tlk（tlk_index=16），中文客户端无对应译文 → 972 条 CEP 物品名以**内嵌 CExoLocString** 方式注入 12,363 处资源：

- **写入格式**：`CExoLocString = [size][strref 不动][count=1][langid=0][len][UTF-8 中文]`（保留原 strref，count 置 1 后追加中文 substring）。
- 工具：`work/_zz_cep_*.py`；字典：`work/cep_cn_all.txt`。
- 7 章 verify 全 PASS、strref/Tag 零改动，⏳ 待实机。
- ⚠️ 两个 CEP 订阅并存时 `tlk_index=16` 会随加载顺序漂移。
- ⚠️ **strref 遮蔽风险（10-17 WW 实锤）**：注入保留了原 strref → TLK 优先场景（Examine）可能显示 TLK 英文而非内嵌中文。WW 的 `ww_strref_fix.py`（内嵌含中文→strref=-1 等长补丁）可直接复用到 CEP 注入产物上——CEP 侧⏳待实机，若 Examine 显英文即跑它。
- ⚠️ 注入用 erf_pack 追加式重建 → mod 体积膨胀；重做**必须从备份出发**（`MOD/modules_bak_cep_20261013/`）。

---

## 11. 三部曲流水线（premium mod + hak 双层）

Kingmaker / ShadowGuard / Witch's Wake 走 `三部曲汉化/work/tri.py`，**七命令**：`extract / seed / batches / apply / build / verify / deploy`。要点：

- **双层结构**：文本同时落在 .mod（GFF）与 .hak（TLK 侧/资源）里，build 两层都做。
- **build 源 = 备份原版**（`backups/premium_trilogy_20261017/`），防 erf 追加膨胀。
- **官方种子**：星空娱动 server.tlk（GBK）→ `server_tlk_utf8.tsv`，seed 进 uniq——官方已译的条目直接复用。
- **批次对位**：用「索引字典 T[i] → walk 批次结构产出 cn 文件」重建法，免疫物理行错位。
- **E 集**（内部名）须按内容逐条裁定留英文，不能整集盲留。
- ⚠️ deploy bug 已修（10-17）：按 src_paths 逐文件回各家（mod→`data\mod`、hak→`data\hk`），旧版全投 data\mod 是错的。
- ⚠️ batch_01（试点批）曾整批错位（cn 缺行→标题/正文滞后一格）→ **每章第 1 批必须直接用索引字典重建法**；候选→译表必须全量导出过目，禁用截断终端输出建表。
- WW 已交付+收尾：10 批全过 2,640 条；**收尾三件套**①ncs 等长汉化（138 条/946 处，禁译区=亚种匹配串）②strref 重指（1,315 处，内嵌含中文→-1）③实机回归⏳。
- **KM 已交付（10-17 晚）**：47 批全译 7,803/7,803（批1-4 主控 + 批5-47 两波 13 子代理）；strref 重指 937 处 + ncs 26 处；审查修正 3 组专名分裂（28 处）后重部署 sha 全等 ⏳实机回归。详见 references/TRILOGY.md。
- ⚠️ **重建流水线四步**：`tri.py build` → `ww/ww_ncs_patch.py` → `ww/ww_strref_fix.py` → `deploy`（重 build 会还原 ncs 补丁，中间两步不可省）。详见 references/TRILOGY.md。

---

## 12. 实例：Act IV Pt3（末章，2026-09-15 收官）

| 阶段 | 数据 |
|---|---|
| 规模 | dlg 去重 4,871 唯一串 / 515,903 英文字符；非对话 2,894 唯一串 |
| 跨章复用 | `_act4_reuse.py` 命中 1,601，剩 945（单行 931 + 多行 14） |
| 对话批 | `batch_00–56` 共 57 批（batch_00 = TM 复用） |
| 非对话 | 单行 10 类 + 多行 4 类；git 12 条内部标识按规矩留空 |
| 回填 | 1,962 文件 / 10,813 处替换 → 34,097,137 → **47,746,381 B** |
| 验收 | 对比 2,167 文件 / 改写 10,799 字段 / **结构错误 0 / CExoString 变更 0 / [PASS]** |

成品：`汉化问题/MOD/modules/The Aielund Saga Act IV - The Fall of Aielund Pt3 (汉化).mod`。
至此 Aielund Saga 全 7 章全部交付并验收 0 错误。

---

## 13. 工具链索引（`汉化问题/work/`）

**流水线核心**
- `dlg_uniq.py <ch> <mod>` 抽取对话 → `uniq.json` ⚠️ 本体旧口径，重做前先改（见 §1）
- `mkbatches.py <ch> [chars]` 切批 → `batch_NN.json`
- `extract_nondlg.py <ch> <mod>` 抽取非对话 → `nondlg_<ext>.json`
- `split_nondlg_remain.py <ch>` remain → single/multi
- `apply_cn.py <ch> NN` 对话填回（原地覆盖）
- `apply_nondlg.py <ch> <ext>` 非对话填回 → `*_filled.json`
- `backfill.py <ch> <mod> [--all]` 合并 + 回嵌 + 打包
- `verify_struct_all.py <orig> <zh> [timeout]` 全类型结构验收
- `verify_struct.py` / `verify_mod.py` / `verify_all.py` 单章/单类型验收

**复用 / 审计 / 检索**
- `_act3_reuse.py` / `_act4_reuse.py` 对话跨章 TM 复用
- `_act3_nondlg_reuse.py` / `_act4_nondlg_reuse.py` 非对话跨章 TM 复用
- `_name_audit.py <ch...>` 译名一致性审计（表 vs 实文）⚠️ 不区分同子串专名（弯刀/弯刃大刀），判定须结合译名表判决
- `_variant_scan.py <ch...>` 同原文不同译法（抓跨章错译）
- `audit_residual_en.py <mod>` 成品残留英文审计（②不查 dlg、③读不了 .ncs，见 §8 真伪复核）
- `_audit_*_en.py` dlg+ncs 台词审计（.ncs 侧见坑 18）
- `fix_strref_mismatch.py <mod> [--dry-run]` 重指被复用的官方 strref→-1（同尺寸 dword 原位补丁，官方签名白名单；**backfill 后必须重跑**，见坑 15）
- `_zz_cep_*.py` + `cep_cn_all.txt` CEP 物品名注入（见 §10）
- `_fix_archmage.py <ch...>` 定向改名（先备份）
- `_chk_hans.py <ch>` 繁体字扫描（只报需转写字）
- `_q_tlk2.py` 官方中文包字节级检索

**底层（被上面调用，勿直接手改）**
- `erf_extract.py` / `erf_pack.py`（ERF 抽取/追加式打包）
- `gff_text.py` / `gff_reembed.py`（GFF 解析/回嵌）

**三部曲**（`三部曲汉化/work/`）
- `tri.py <extract|seed|batches|apply|build|verify|deploy>` 七命令（见 references/TRILOGY.md）
- `trilogy/{ww,km}/ncs_audit_scan.py` + `*_ncs_patch.py`：.ncs 台词审计 + 等长原位汉化（SG 改路径重扫即可复用）
- `trilogy/{ww,km,sg}/*_strref_fix.py`：strref 泛化重指（内嵌含中文→-1，mod+hak 全 GFF；build 后必跑）
- `work/lib/ncs_strings.py` + `work/_ncs_gate.py`：**ncs 2 字节大端解析 + 三部统一终检门禁（双口径）**——ncs 扫描/终检唯一合法口径
- `work/trilogy/_ncs_patch2.py` / `_ncs_patch3.py`：ncs 漏译清欠补丁范本（out/ 层直接替换，含混合残段）
- `trilogy/km/_TANSPEC.md`：子代理翻译规范范本（硬规则+产出流程+唯一裁定表+自检脚本）
- `trilogy/ww/_fix_b01_shift.py`：整批错位重排范本（key[i]←value[i+1] + 断言）

---

## 14. 参考

- **references/NONDLG.md** —— 非对话流水线逐步详解（含 tech token 留空、末行空行、multi dict 格式）
- **references/PITFALLS.md** —— 19 个致命坑 + 环境坑全表（GFF List 结构 / ERF 追加式 / type10 禁令 / 尾空格匹配 / backfill 快照 / strref 复用错配 / 改译双写 / 审计假阳性 / .ncs 台词 / 共享临时文件等）
- **references/NAMING.md** —— 译名纪律 + 官方 TLK 查表铁律 + 审计工具用法
- **references/TRILOGY.md** —— premium 三部曲（Kingmaker/ShadowGuard/Witch's Wake）mod+hak 双层流水线
- 坑点详表（叙事版）：工作区根 `HANDOFF.md` §⑤
- 工具链长文：《NWNEE_汉化工具链与客户端补丁铁律.md》（工作区根）
- 项目长期记忆：`.workbuddy/memory/MEMORY.md`
- 每日流水：`.workbuddy/memory/YYYY-MM-DD.md`
