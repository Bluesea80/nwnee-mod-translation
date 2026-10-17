# 非对话（nondlg）汉化流水线 · 逐步详解

覆盖 `uti / utc / are / jrl / ifo / git / utp / ute / utt / uts / utw` 等玩家可见文本。
对话（`.dlg`）走 §1 流程；这里是非对话。非对话**最易在"行数配对"和"单/多行拆分"上翻车**，逐步讲清。

工具链根：`汉化问题/work/`，下文 `PY=...python3`。

---

## 总览（7 步）

```
extract_nondlg.py   → nondlg_<ext>.json （全部混在一起，含 \n 与不含 \n）
_actN_nondlg_reuse  → nondlg_<ext>_remain.json （剔除 TM 复用项）
split_nondlg_remain → nondlg_<ext>_single.json + nondlg_<ext>_multi.json
手写 cn_<ext>.txt          （单行，行序配对 single，空行=留英文）
手写 nondlg_<ext>_multi_cn.json  （多行，dict{原文:译文}）
apply_nondlg.py    → nondlg_<ext>_filled.json
backfill.py --all  （合并所有 filled + batch → 打包）
verify_struct_all  （结构错误 0 / CExoString 变更 0）
```

---

## ① 抽取

```bash
$PY extract_nondlg.py <ch> "<原版 mod 文件名>"
# 想先看样本： $PY extract_nondlg.py <ch> "<mod>" --sample 30
```

产物（都在 `work/<ch>/`）：
- `nondlg_<ext>.json`：`{"英文": ""}` 待填，**所有类型混在一个文件里按 ext 分文件**（uti 一个、utc 一个……）。
- `_nondlg_report.txt`：每段/每类型的段数、唯一数、字符数统计（先看这个定翻译策略）。

只取 **玩家看得见的类型**（`WANT` 集：uti/utc/are/jrl/ifo/git/utp/ute/utt/uts/utd/utw/fac）。
判断"要不要收"**看 LocString 有没有非空 substring**（坑 9：strref 有效但内嵌英文 substring 的也要收）。

---

## ② 跨章复用（TM）

```bash
$PY _act4_nondlg_reuse.py      # Act IV；Act III 用 _act3_nondlg_reuse.py
```

逻辑：拿前几章已译的 `*_filled.json` 建 TM，命中项直接从 `nondlg_<ext>.json` 剔除，剩**未译**项写 `nondlg_<ext>_remain.json`。
**只对 remain 翻译**，已复用的不碰。

---

## ③ 拆单 / 多行（关键，别漏）

```bash
$PY split_nondlg_remain.py <ch>
```

把每个 `nondlg_<ext>_remain.json` 拆成：
- `nondlg_<ext>_single.json` —— **不含 `\n`** 的条目（短名/短描述，行序配对翻译）
- `nondlg_<ext>_multi.json`  —— **含 `\n`** 的条目（长文本/多段，按 key 翻译）

> 为什么拆：单行用"行序配对"零转义最稳；多行含换行，行序配对会错位，改用"字典按原文键"匹配。

---

## ④ 译单行 → `cn_<ext>.txt`

- 第 i 行 ↔ `nondlg_<ext>_single.json` 第 i 个键（严格行序）。
- **行数必须 == 键数**。
- **空行 = 该条留英文不译**（内部标识、脚本引用键、音效名等）。
- ⚠️ **末行空行陷阱**：若最后一个 key 应留空，文件必须以 `…\n\n` 结尾（**多写一个空行**）。
  脚本只 pop "文件末尾换行"产生的那一个空元素，不会多 pop —— 首轮 `cn_git.txt` 就因末 7 条 tech token 漏写空行导致行数不符。
  **正解**：用 Python 生成，`assert len(keys)==len(values)`，并用"应留空集合"双向校验（`set(blank_keys) == set(实际空行对应的 key)`）。

**哪些该留空（按规矩不译）**—— 常见内部标识：
- `git`：`Alarm` `CASTER` `FIREPOINT` `LUCAS` `TWILIGHT_MERCHANT` `Zap point` `beam1` `beam2` `beam3` `beam4` `beampoint` `bloodsound`
- `utp` / `uts` / `utw`：大量内部键（每章数量不同，逐章看 remain 决定）

---

## ⑤ 译多行 → `nondlg_<ext>_multi_cn.json`

**格式 = JSON dict，键是原文整串，值是译文**（不是行序、不是 `cn_*.txt`）。

```python
import json
src = json.load(open("nondlg_uti_multi.json", encoding="utf-8"))
cn = {}
for k, _ in src.items():
    cn[k] = "这里填译文（保留原文里的换行）"
json.dump(cn, open("nondlg_uti_multi_cn.json","w",encoding="utf-8"),
          ensure_ascii=False, indent=1)
```

- 键必须 **exact match** 原文字符串（含里面的 `\n`）。
- 多行里的引擎 token（`<FirstName>` 等）同样保留英文。
- 验证：`set(src.keys()) == set(cn.keys())`。

---

## ⑥ 填回 → `nondlg_<ext>_filled.json`

```bash
$PY apply_nondlg.py <ch> <ext>
```

`apply_nondlg.py` 需要 **4 个文件齐备**才能跑：
1. `nondlg_<ext>_single.json`
2. `cn_<ext>.txt`（行序配对 single）
3. `nondlg_<ext>_multi.json`
4. `nondlg_<ext>_multi_cn.json`（dict 配对 multi）

→ 输出 `nondlg_<ext>_filled.json`（合并非空译文）。

缺任一文件会报错；多行若没翻译就**不要**生成 `multi_cn.json`（或生成空 dict `{}`），否则配对失败。

---

## ⑦ 回填 + 验收

```bash
$PY backfill.py <ch> "<原版 mod 文件名>" --all
$PY verify_struct_all.py "<原版 mod 文件名>" "<汉化 mod 文件名>"
```

`backfill.py --all` 会读 `work/<ch>/` 下**所有** `batch_*.json` + `nondlg_*_filled.json`，合并非空译文 → 字典匹配回嵌 → 追加式打包成 `(汉化).mod`。
验收铁律见 SKILL.md §5。

## ⑧ 改定案 / 补译时的「双写」铁律（坑 16，2026-09-15 实锤）

`backfill.py` **实际消费的是 `nondlg_<ext>_filled.json`（生成物），不是 `cn_<ext>.txt`（源）**。
改定案/补译时：

1. `cn_<ext>.txt`（源，保流程可复现）；
2. `nondlg_<ext>_filled.json`（**回填直接消费物，漏改 = 旧值入包**——act4p3 area 名漏"界"×11 实锤，verify 全 PASS 抓不到）；
3. 涉及专名的同步改 `译名表.md`。

改完重跑 backfill（后台）→ verify → **成品级 GFF 复扫归零**（复扫才抓得到内容对错）。
若改动涉及 strref 复用错配（坑 15），backfill 后还须重跑 `fix_strref_mismatch.py`。

---

## 实例：Act IV Pt3 非对话统计

| 类型 | 实译(单行) | 多行 | 留空 |
|---|---:|---:|---:|
| utc | 130 | 0 | 0 |
| jrl | 86 | 7 | 0 |
| uti | 274 | 3 | 0 |
| git | 338 | 3 | 12 |
| are | 36 | 0 | 0 |
| ifo | 1 | 1 | 0 |
| ute | 20 | 0 | 0 |
| utp | 8 | 0 | 10 |
| uts | 0 | 0 | 11 |
| utw | 1 | 0 | 4 |
| utt | 0 | 0 | 0 |

`utt` 为 0 条 → 跳过 `apply_nondlg.py utt`。`git` 的 12 条内部标识按规矩留空。
