# 项目级 `.claude` 配置

本目录由 [ECC](https://github.com/claude-code) installer 以 `claude-project` 目标**最小定制接入**生成，为 linux-doc 文档仓库提供研究与文档处理能力。所有 skills 均来自 ECC 上游，保持与上游同步，请勿直接改动其中的 `SKILL.md`。

## 已安装 Skills

### 研究 / 查证

| Skill | 用途 |
|---|---|
| `deep-research` | 基于 firecrawl 与 exa 的多源深度研究，输出带引用与来源标注的综述报告 |
| `exa-search` | Exa 神经搜索引擎：网页/代码/公司信息检索、人物查找、AI 深度研究 |
| `research-ops` | 以证据为先的现状调研工作流：最新事实、方案对比、带引用的建议 |
| `documentation-lookup` | 通过 Context7 读取库/框架的**最新**文档，避免依赖训练数据 |
| `pubmed-database` | PubMed / NCBI E-utilities：生物医学文献检索、MeSH 查询、PMID 查证 |
| `uspto-database` | USPTO 专利与商标官方记录查询、PatentSearch、TSDR 核查 |
| `gget` | 基因组数据库快速查询、序列检索、BLAST、生信证据记录 |
| `literature-review` | 系统性文献综述：检索规划、来源筛选、综合、引用核验 |
| `scholar-evaluation` | 学术成果结构化评估：论文/提案/综述的方法、证据质量、引用支撑 |

### 文档处理 / 翻译

| Skill | 用途 |
|---|---|
| `nutrient-document-processing` | 基于 Nutrient DWS API 处理 PDF/DOCX/XLSX/PPTX/HTML/图片：转换、OCR、提取、编辑 |
| `visa-doc-translate` | 签证申请文档（图片）翻译为英文，生成双语 PDF |

## 使用方式

在 Claude Code 会话中直接通过斜杠命令调用，例如：

```bash
/deep-research  Btrfs CoW 与硬件生态的最新研究
/documentation-lookup  tar 手册
```

或由 Claude 在相关任务中自动触发对应的 skill。

## 目录结构

```
.claude/
├── README.md                 ← 本说明
├── marketplace.json          ← 插件 marketplace 清单
├── plugin.json               ← 插件 manifest（Claude Code 自动按约定加载，勿手动声明 hooks/agents）
├── mcp-configs/              ← MCP 服务器配置（当前为空目录表）
├── scripts/                  ← ECC 平台辅助脚本
├── skills/                   ← 已安装 skills 与 learned 已学技能
└── ecc/install-state.json    ← ECC 本地安装状态（含本机路径，已被 .gitignore 忽略）
```

## 维护

- **更新 skills**：从 ECC 上游仓库重新运行 installer 即可同步：

  ```bash
  node /home/l/.claude/plugins/marketplaces/ecc/scripts/install-apply.js \
    --target claude-project --modules document-processing,research-apis
  ```

- **`install-state.json`** 记录本机安装状态，含绝对路径，**不纳入版本控制**。
- 项目根 `CLAUDE.md` 为文档撰写约定；`settings.local.json` 为本机权限配置，均不受本目录管理。

## 常见问题

**Q：删除某个 skill 会影响仓库文档吗？**
不会。这些 skills 是 Claude Code 的辅助能力，仓库中的文档本身不依赖它们。

**Q：为什么这里没有代码规则（rules）？**
本项目为纯文档仓库，未接入 `rules-core` 模块，避免无关语言规则干扰文档工作流。

**Q：skills 与全局安装的 `ecc:*` 重复吗？**
全局插件已提供同能力。项目级安装让这些 skills 在本仓库内显式可用，与全局共存，不冲突。
