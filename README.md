# nwnee-mod-translation — NWN:EE 模组汉化技能（WorkBuddy Skill）

**NWN:EE（Neverwinter Nights: Enhanced Edition）模组简体中文汉化**的完整工程流水线，以 WorkBuddy Skill 形式沉淀（`SKILL.md` + `references/`）。

工程实证来源：

- **Aielund Saga 全 7 章**（Act I / II / III / IV Pt1·Pt2·Pt3 / Finale）——全部交付，验收 **结构错误 0 / CExoString 变更 0**
- **Premium 三部曲**（Kingmaker / ShadowGuard / Witch's Wake，mod+hak 双层）——Witch's Wake 已交付（2,639 条译文，mod 2,532 处 / hak 3,032 处回填）

## 能力覆盖

| 线 | 内容 | 文档 |
|---|---|---|
| 对话 | `.dlg` 抽取 → 切批 → 翻译 → 回填 → strref 重指 → 结构验收 | SKILL.md §1 |
| 非对话 | `uti/utc/are/jrl/ifo/git/utp/ute/utt/uts/utw` 七步流水线 | SKILL.md §2 + references/NONDLG.md |
| 三部曲 | premium 模组 mod+hak 双层，官方 TLK 种子复用 | references/TRILOGY.md |

## 核心知识

- **四条铁律**：只翻 type 12 CExoLocString；看 substring 不看 strref；TLK 不能自动套译文；整串精确匹配的空白/引号对齐
- **TLK 与 strref 速查**：`strref=(tlk_index<<20)|row` 编码、cep260.tlk(tlk_index=16)、4095 空哨兵过滤、TLK V3.0 offset 基准
- **CEP 物品名注入**（方案 A）：972 条注入 12,363 处的内嵌格式
- **19 个致命坑全表**（references/PITFALLS.md）：GFF List 结构 / ERF append-only / type10 禁令 / strref 复用错配（坑 15，backfill 后必须重跑 fix）/ 改译双写（坑 16）/ 审计假阳性（坑 17）/ `.ncs` 台词（坑 18）/ 共享临时文件污染（坑 19）
- **译名纪律**（references/NAMING.md）：译名表唯一事实源 + 官方 TLK → 3.5 SRD 查表铁律

## 目录结构

```
SKILL.md                  # 操作手册 + 指挥中枢（v1.3.0）
references/
  NONDLG.md               # 非对话流水线逐步详解
  PITFALLS.md             # 19 个致命坑 + 环境坑全表
  NAMING.md               # 译名纪律 + 官方 TLK 查表铁律
  TRILOGY.md              # 三部曲 mod+hak 双层流水线
```

## 使用

本技能为 WorkBuddy 项目级技能，放置于项目 `.workbuddy/skills/nwn-mod-translation/`。配套的 Python 工具链（抽取/切批/回填/验收脚本，约 20 个纯 Python 文件）随项目工作区维护，本仓库收录方法论与纪律文档。

## 环境实测

macOS（Mac 侧生产）与 Windows（Git Bash / PowerShell）双机验证；Python 3.13（UTF-8 原生）。
