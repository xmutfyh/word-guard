# word-guard

Agent Skill：**已有 Word (.docx) 文档的安全局部编辑与验收**。

核心原则：只做「局部替换 + 范围校验」，从不重建文档。LLM 只负责产出改后的文字，
docx 始终由本技能在本地写回，因此原文档的样式、编号、公式、图片都不会被破坏。

## 能力

| 能力 | 说明 |
|---|---|
| 结构扫描 | 标题层级、段落索引、公式、图片、表格清单 |
| run-aware 替换 | 按章节或段落范围替换文本，保留原有 run 级格式 |
| 范围校验 | 编辑后比对，确认「只改了该改的地方」 |
| 包完整性校验 | OOXML 包级校验，防止文档损坏 |
| 学术三线表 | 非版式表格统一为三线表（清旧样式 + 单元格居中） |
| 公式审计 | 原生 OMML 公式排版审计（公式居中、编号右对齐） |
| 格式指纹 | 编辑前备份格式指纹，便于回滚与比对 |

## 安装

把本仓库内容放到 Agent Skill 目录下，目录名保持 `word-guard`：

```text
~/.dsh/skills/word-guard/
├── SKILL.md
├── manifest.yaml
└── scripts/word_guard/
```

Linux / macOS：

```bash
git clone https://github.com/xmutfyh/word-guard.git ~/.dsh/skills/word-guard
```

Windows (PowerShell)：

```powershell
git clone https://github.com/xmutfyh/word-guard.git "$env:USERPROFILE\.dsh\skills\word-guard"
```

## 依赖

- Python 3.9+
- 仅需 `pip install python-docx`（无需 Node、无需插件、完全离线）

## 使用

技能的具体调用流程、参数与验收标准见 [SKILL.md](SKILL.md)。
CLI 入口为 `scripts/word_guard/cli.py`。

## 不适用

从零新建 Word 文档、插图、修订痕迹（tracked changes）、批注、`.doc` / PDF 转换。

## 作者

Yuanhao Feng
