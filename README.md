[English](README_EN.md) | **简体中文**

# book-mind — 书籍章节思维导图大纲生成

将书籍指定章节原文提取为「章 → 节 → 子节 → 核心要点」四层结构的 **mindmap 格式 Markdown 大纲**（mindmap 格式 = Markdown 大纲，即导入格式本体），可直接导入 **XMind / MindMaster / 幕布 / Notion**。

v2.3 起升级为**三模式整书处理管道**：A 索引模式（整本书从目录页快速建章节索引，禁止逐页扫描）→ B 检索模式（按章节号或主题定位，只返回位置、不擅自提取）→ C 直提模式（仅对命中章节输出四层大纲）。索引一次建好驻留会话，之后每次只处理目标章节，不再反复全书扫描——这是 v2.3 解决“接入新书慢”的核心机制；子节要点复杂时允许加深一级子目录展开（层级弹性），解决“节点死板”。核心资产《书籍内容处理规范》v2.3 零改写封装为 Claude Skill，同一份规范文本同时用于各平台智能体与本地批量脚本，三端行为完全一致。

## 仓库结构

```
skill-book-mind/
├── book-mind/        # Skill 源文件夹（上传 claude.ai 用）
│   ├── SKILL.md                 # frontmatter + v2.3 规范正文（逐字保留）
│   └── examples.md              # 完整示例 + 8 项压测用例
├── book-mind.zip     # claude.ai Skills 上传包
├── prompt.md                    # v2.3 规范纯文本（平台系统提示词 / 脚本外部 prompt）
├── book_to_mindmap.py           # 多格式批量脚本（PDF/Word/EPUB/TXT，MOBI 尽力）
├── requirements.txt             # 脚本依赖
├── 压测报告.md                   # 8 项上线压测记录（8/8 通过）
├── 书籍思维导图skill 设计说明2.3.txt # v2.3 规范定稿原文（唯一事实来源）
├── 书籍思维导图skill 设计说明2.2.txt # v2.2 历史定稿（已被 2.3 整体替换）
└── 单干-第01章-变现篇-思维导图.md # 真实书籍输出样例
```

## 使用方式一：Claude Skill（推荐）

1. 下载 `book-mind.zip`；
2. claude.ai → 设置 → Capabilities → Skills → 上传 zip；
3. 对话中说 **“帮我把这章做成思维导图大纲”** 并粘贴章节原文，即自动触发。

> 已按规范要求本地安装到 `~/.zcode/skills/book-mind/`（与其余用户技能同目录，优先级最高），ZCode 等工具新会话启动时自动发现。
> 注意：skill 的 name 只能含小写字母、数字、连字符；description 上限 1024 字符，须同时包含“做什么”与“何时用”（本仓库已按此写好，无需改动）。

## 使用方式二：平台智能体（清言 / 扣子 / Dify 等）

1. 新建智能体，将 `prompt.md` 全文整体粘贴进「系统提示词 / 人设与回复逻辑」栏；
2. **不做任何改写，平台的“优化提示词”按钮一律不点**；
3. 开场白建议：`请提供书籍章节原文，我将输出可直接导入 XMind/幕布的 mindmap 大纲`；
4. 若平台支持知识库，可把 `examples.md` 作为示例文件上传，并在系统提示词里加一句“输出样式参考知识库中的示例文件”。

## 使用方式三：本地批量脚本（多格式）

支持从电子书站点下载的常见格式：

| 格式 | 选取方式 | 说明 |
|------|----------|------|
| `.pdf` | `--pages 10-25` | 按页码范围选取（1 起始） |
| `.docx` | `--chapters 3-5` | 按 Word 标题样式或「第X章/Chapter N」自动识别章节 |
| `.epub` | `--chapters 3-5` | 按 spine 阅读顺序切章 |
| `.txt` / `.md` | `--chapters 3-5` | 按「第X章/Chapter N」识别；无标题时按约 4000 字切段 |
| `.mobi` / `.azw3` | 同 EPUB | 尽力支持：需 `pip install mobi`；失败时建议先用 Calibre 转 EPUB |

### 安装

```bash
# 使用 conda py312 环境（Windows 示例）
D:/dev_tools/miniconda3/envs/py312/python.exe -m pip install -r requirements.txt
```

PDF 提取默认使用 **pymupdf**（v2.3 规范要求；未安装时自动回退 pypdf）。

### 常用命令

```bash
PY=D:/your/path/envs/py312/python.exe

# 1. 先查看可选的页/章节清单（不调 API）
$PY book_to_mindmap.py book.epub --list

# 2. 干跑预览提取文本（不调 API）
$PY book_to_mindmap.py book.pdf --pages 10-25 --dry-run

# 3. 正式提取：EPUB 第 3~5 章，每 2 章一次调用
$PY book_to_mindmap.py book.epub --chapters 3-5 --chunk-pages 2

# 4. Word / TXT 同理
$PY book_to_mindmap.py book.docx --chapters 1-2
$PY book_to_mindmap.py book.txt
```

### API 配置

| 提供方 | `--base-url` | `--model` 示例 | 密钥环境变量 |
|--------|--------------|----------------|--------------|
| GLM（默认） | `https://open.bigmodel.cn/api/paas/v4` | `glm-4.6` | `ZHIPUAI_API_KEY` |
| Claude | `https://api.anthropic.com/v1/` | `claude-sonnet-4-5` | `ANTHROPIC_API_KEY` |
| 其他兼容服务 | 自定义 | 自定义 | `OPENAI_API_KEY` |

也可用 `--api-key` 直接传入。system prompt 从 `prompt.md` 读取（可用 `--prompt` 换路径），**不硬编码在代码里**。

### 输出、日志与重试

- 结果保存为 `<书名>_p10-25.md`（按页）/ `<书名>_c3-5.md`（按章）/ `<书名>_all.md`（全文）；
- 每个分块失败自动**重试 3 次**（指数退避 2/4/8 秒）；
- 全过程记录到同名 `.log` 文件（控制台同步输出）；
- 某分块最终失败时：已完成的分块照常写入结果文件并退出码 1，日志标明失败位置。

## 验收与压测

8 项上线压测（v2.2 原 5 项：篇级目录 / 无子节章节 / 问答体 / 仅目录无正文 / 代码密集；v2.3 新增 3 项：整书建索引 / 主题检索仅定位 / 按篇提取不混章）**全部通过**，详见[压测报告.md](压测报告.md)。压测任一项失败时，先区分是**规则被执行偏差**（回到封装环节对照定稿逐字校对）还是**规则本身冲突**（此时才出 v2.3）。

## 规范要点（v2.3）

- 输出仅含 Markdown 大纲（`## / ### / #### / -`），无任何对话内容；**禁止 Mermaid** 等无法被 XMind/MindMaster/幕布/Notion 直接导入的语法；
- 章号归一化为两位阿拉伯数字（第三章 → `## 第03章`），节编号章内从 1 重计；
- 仅给目录时输出 `- 待补充：需提供章节正文` 骨架；原书本无内容用 `- 内容空缺`，二者不得混用；
- 代码密集章节不保留代码块，API 名压缩进 ≤30 字要点；
- **索引 / 检索 / 直提三模式**：整书先建索引（从目录页生成，禁止逐页扫描），检索命中才提取；任何模式下禁止全书一次性全量输出；
- **层级弹性**：子节要点复杂或较多时，可增设更深一级子目录（如 `##### N.M.K`）分组展开，不局限于「子节＋要点」两级；
- PDF 提取使用 pymupdf，失败重试 2 次后停止技术攻关、提示用户粘贴文本，禁止手写解析器；
- 规范原文以 `书籍思维导图skill 设计说明2.3.txt` 中 v2.3 定稿为唯一事实来源；复制与校验一律以原始文本为准，不以渲染视图为准。
