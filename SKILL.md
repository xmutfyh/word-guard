---
name: word-guard
description: "对已有 Word (.docx) 文档做安全的局部编辑与验收：结构扫描（标题层级/段落索引/公式/图片）、按章节或段落范围做 run-aware 文本替换、编辑后范围完整性校验、OOXML 包完整性校验、学术三线表、原生公式审计、格式指纹备份。适用于修改论文/报告的某几段、润色结果写回、让 LLM 改文后落地、字号与表格格式调整。不用于从零新建 Word 文档、插图、修订痕迹或批注。触发场景：扫描 docx、修改第三章、替换某个段落、safe edit、scope check、三线表、修改字号、编辑后验证范围、docx edit、word 安全编辑、AI 改文写回。"
version: 1.0.0
author: Yuanhao Feng
---

# Word Guard — 已有 Word 文档的安全编辑

改 docx 的正确姿势：**只做"局部替换 + 范围校验"，从不重建文档**。
LLM（包括网页版 GPT）只负责产出"改后的文字"，docx 永远由本工具在本地写回。

## 什么时候用 / 不用

| 用 | 不用（换别的工具） |
|---|---|
| 修改已有 .docx 的某几段/某章节 | 从零新建 Word（目录、页眉页脚、脚注、多栏排版） |
| 润色/AI 改文结果写回原文档 | 插入图片、修订痕迹（tracked changes）、批注 |
| 调字号、统一格式、表格改三线表 | `.doc` 转换、PDF 转换 |
| 编辑后验证"只改了该改的" | 内容质量审计（查 AI 味、数字引用是否被动等） |

## 依赖

- Python 3.9+，只依赖一个包：`pip install python-docx`
- 不需要 Node、不需要任何插件、完全离线。

## 目录结构

```
word-guard/
├── SKILL.md                  ← 本文件
├── manifest.yaml
└── scripts/
    └── word_guard/           ← 纯 Python 模块（可单独拷走）
        ├── cli.py            ← CLI 总入口
        ├── scanner.py        结构扫描
        ├── editor.py         run-aware 替换（保留原格式）
        ├── scope.py          章节/段落范围解析
        ├── protected.py      保护节点（公式/图片/交叉引用/参考文献）
        ├── manifest.py       变更清单
        ├── integrity.py      范围完整性比对
        ├── package_guard.py  OOXML 包校验
        ├── fingerprint.py    格式指纹
        ├── equation_audit.py 公式审计
        └── table_format.py   三线表
```

`cli.py` 自己会修好 import 路径，**从任意工作目录直接调用即可**：

```bash
python scripts/word_guard/cli.py <命令> ...
```

## 命令一览

```text
scan <file.docx>                            结构扫描：profile、标题层级、段落索引、表格、公式、警告
edit <file.docx> <reps.json> [out.docx] [mode] [scope.json]   局部编辑
scope-check <before.docx> <after.docx> [scope.json]           编辑后范围完整性校验
package-validate <file.docx>                OOXML 包完整性（relationships / XML 合法性）
format-tables <file.docx> [out.docx] [baseline.docx]          学术三线表（版式表自动跳过）
fingerprint <file.docx>                     生成格式基线指纹（页边距/样式/字体/表格边框签名）
equation-audit <file.docx> [baseline.docx]  原生 OMML 公式审计（连续式/章节式编号）
audit <file.docx>                           提取正文 + 基础写作检查（修改残留词、AI 套话、单位）
```

编辑模式 `mode`：

- `text_only`（默认，推荐）：只替换文字，保留段落原有格式；
- `structural`：章节级插入/删除/整段替换；
- `format_normalization`：全文格式统一（慎用，影响全文）。

范围 `scope`（JSON 字符串，五种写法任选）：

```json
{"heading": "3 方法"}
{"startHeading": "1 引言", "endHeading": "3 方法"}
{"startParagraph": 10, "endParagraph": 25}
```

不传 scope = 全文范围（仍受保护节点约束）。

## 标准四步管线（推荐流程）

```bash
# 1) 扫描：拿到结构和原文段落
python scripts/word_guard/cli.py scan 论文.docx

# 2) 写回：把改好的文字放进 replacements.json
#    [{"old": "逐字来自原文的句子", "new": "改写后的句子"}, ...]
python scripts/word_guard/cli.py edit 论文.docx reps.json 论文_改.docx text_only '{"heading":"3 方法"}'

# 3) 范围校验：范围外修改必须为 0
python scripts/word_guard/cli.py scope-check 论文.docx 论文_改.docx

# 4) 包完整性：确认文件没被改坏
python scripts/word_guard/cli.py package-validate 论文_改.docx
```

Windows PowerShell 下 JSON 参数用单引号包住（如上），bash 下同样用单引号。

## 让 LLM（含网页版 GPT）改文的用法

网页版 GPT 收不到、也回不了真正的 .docx，所以分工必须是——**云端出文字，本地动文件**：

1. **提取**：`cli.py scan`（或 `audit`）拿到要改的段落原文；
2. **发给 LLM**：附上原文段落 + 修改要求，并明确要求
   > 只返回 JSON 替换对：`[{"old":"原文","new":"改后"}]`，old 必须逐字引用我给你的原文，禁止输出整篇文档；
3. **写回**：把返回内容存成 `reps.json`，执行 `edit`；
4. **验收**：`scope-check` + `package-validate`，两个都通过才交付。

这样做的原因：整篇重写 docx 会毁掉字体、公式、交叉引用、页眉页脚；逐段 old→new 替换则只动文字。

## 硬性纪律

- `old` 必须**逐字**匹配（含标点、空格），建议直接从 `scan` 输出里复制，不要凭记忆改写；
- 每次 `edit` 自动生成 `<文件>.docx.bak` 备份和变更清单，出问题先还原；
- 公式、图片、交叉引用、参考文献等复杂段落会被**拒绝替换**并明确报告，不要绕过；
- 范围歧义（如章节标题重复）会报 `Ambiguous scope` 并给出候选项，选定后重跑，不要猜；
- 交付前必须跑 `scope-check`：范围外修改数 = 0 才算通过。

## 验收输出怎么读

- `scope-check`：`unintended_changes: []` 或 count 为 0 → 通过；有内容 → 说明改动泄漏到范围外，还原重来；
- `package-validate`：`valid: true` → 通过；false → 文件结构已损坏，用 `.bak` 回滚；
- `edit` 返回的 manifest 会列出每一处实际替换与跳过原因，交付前扫一眼。
