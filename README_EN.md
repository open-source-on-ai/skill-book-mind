**English** | [简体中文](README.md)

# book-mind — Book Chapter Mindmap Outline Generator

Extracts a specified book chapter into a four-level **mindmap-format Markdown outline** — Chapter → Section → Subsection → Key Points (mindmap format = Markdown outline, i.e. the import format itself) — ready to import into **XMind / MindMaster / Mubu / Notion**.

As of v2.3 the skill is a **three-mode whole-book pipeline**: Mode A *Index* (build a chapter index quickly from the book's TOC page — no page-by-page scanning), Mode B *Retrieve* (locate chapters by number or topic; return locations only, never extract on its own), Mode C *Direct extract* (output the four-level outline only for the matched chapter). The index is built once and stays in the session, so each later request processes only the target chapter — this is the core fix for slow onboarding of a new book; complex subsections may expand one level deeper (hierarchy flexibility). The core *Book Content Processing Spec* v2.3 is wrapped verbatim into a Claude Skill, while the exact same spec text drives platform agents and the local batch script, so all three surfaces behave identically.

## Repository Layout

```
skill-book-mind/
├── book-mind/         # Skill source folder (for claude.ai upload)
│   ├── SKILL.md                  # frontmatter + v2.3 spec body (verbatim)
│   └── examples.md               # full example + 8 acceptance test cases
├── book-mind.zip      # upload package for claude.ai Skills
├── prompt.md                     # plain v2.3 spec text (platform system prompt / script prompt)
├── book_to_mindmap.py            # multi-format batch script (PDF/Word/EPUB/TXT, best-effort MOBI)
├── requirements.txt              # script dependencies
├── 压测报告.md                    # acceptance test report (8/8 passed)
├── 书籍思维导图skill 设计说明2.3.txt  # v2.3 spec source of truth (Chinese)
├── 书籍思维导图skill 设计说明2.2.txt  # v2.2 history (fully replaced by 2.3)
└── 单干-第01章-变现篇-思维导图.md    # real-book output sample
```

## Option 1: Claude Skill (recommended)

1. Download `book-mind.zip`;
2. claude.ai → Settings → Capabilities → Skills → upload the zip;
3. In chat, say **"turn this chapter into a mindmap outline"** (or Chinese “帮我把这章做成思维导图大纲”) and paste the chapter text — the skill triggers automatically.

> The skill is also installed locally at `~/.zcode/skills/book-mind/` (alongside your other user skills, highest precedence); ZCode-style tools discover it automatically on a new session.
> Note: skill names may only contain lowercase letters, digits and hyphens; `description` is capped at 1024 characters and must state both *what it does* and *when to use it* (already done — no edits needed).

## Option 2: Platform Agents (Chatbot builders such as Dify / Coze)

1. Create a new agent and paste the entire `prompt.md` into the *system prompt / persona* field;
2. **Do not rewrite anything; never click the platform's "optimize prompt" button**;
3. Suggested opening line: `请提供书籍章节原文，我将输出可直接导入 XMind/幕布的 mindmap 大纲`;
4. If the platform supports knowledge bases, upload `examples.md` as the sample file and append one line to the system prompt: "output style follows the sample file in the knowledge base".

## Option 3: Local Batch Script (multi-format)

Supports the formats commonly offered by e-book download sites:

| Format | Selection | Notes |
|--------|-----------|-------|
| `.pdf` | `--pages 10-25` | select by 1-based page range |
| `.docx` | `--chapters 3-5` | chapters detected via Word heading styles or “第X章/Chapter N” |
| `.epub` | `--chapters 3-5` | split by spine reading order |
| `.txt` / `.md` | `--chapters 3-5` | detected via “第X章/Chapter N”; fallback ~4000-char segments |
| `.mobi` / `.azw3` | same as EPUB | best effort: requires `pip install mobi`; otherwise convert to EPUB with Calibre first |

### Install

```bash
# Example with the conda py312 environment (Windows)
D:/your/path/envs/py312/python.exe -m pip install -r requirements.txt
```

PDF extraction uses **pymupdf** by default (required by the v2.3 spec; falls back to pypdf when absent).

### Common Commands

```bash
PY=D:/dev_tools/miniconda3/envs/py312/python.exe

# 1. List selectable pages/chapters (no API call)
$PY book_to_mindmap.py book.epub --list

# 2. Dry-run: preview extracted text (no API call)
$PY book_to_mindmap.py book.pdf --pages 10-25 --dry-run

# 3. Real run: EPUB chapters 3-5, one API call per 2 chapters
$PY book_to_mindmap.py book.epub --chapters 3-5 --chunk-pages 2

# 4. Word / TXT work the same way
$PY book_to_mindmap.py book.docx --chapters 1-2
$PY book_to_mindmap.py book.txt
```

### API Configuration

| Provider | `--base-url` | Example `--model` | Key environment variable |
|----------|--------------|-------------------|--------------------------|
| GLM (default) | `https://open.bigmodel.cn/api/paas/v4` | `glm-4.6` | `ZHIPUAI_API_KEY` |
| Claude | `https://api.anthropic.com/v1/` | `claude-sonnet-4-5` | `ANTHROPIC_API_KEY` |
| Other compatible | custom | custom | `OPENAI_API_KEY` |

You may also pass `--api-key` directly. The system prompt is read from `prompt.md` (override with `--prompt`) and is **never hard-coded** in the script.

### Output, Logging and Retries

- Results are saved as `<name>_p10-25.md` (by pages) / `<name>_c3-5.md` (by chapters) / `<name>_all.md` (whole book);
- Each chunk is **retried up to 3 times** on failure (exponential backoff 2/4/8 s);
- Everything is logged to a sibling `.log` file (mirrored to the console);
- If a chunk ultimately fails: completed chunks are still written to the output file, exit code 1, failure position recorded in the log.

## Acceptance Tests

All 8 acceptance cases **passed** — the original 5 (part-level TOC / section-without-subsection / Q&A style / TOC-only input / code-heavy chapter) plus 3 new in v2.3 (whole-book indexing / topic retrieval returns locations only / per-part extraction without cross-chapter leakage) — see [压测报告.md](压测报告.md) (Chinese). On any failure, first determine whether it is *spec-execution drift* (re-verify the packaging against the frozen v2.2 text) or a *genuine rule conflict* (only then draft v2.3).

## Spec Essentials (v2.3)

- Output contains only the Markdown outline (`## / ### / #### / -`) with zero conversational text; **Mermaid and any syntax not directly importable** into XMind/MindMaster/Mubu/Notion is forbidden;
- Chapter numbers are normalized to two Arabic digits (第三章 → `## 第03章`); section numbering restarts at 1 within each chapter;
- TOC-only input yields a skeleton with `- 待补充：需提供章节正文` per level-4 position; genuinely empty source uses `- 内容空缺` — the two markers must never be mixed;
- Code-heavy chapters keep no code blocks; API names are compressed into points of ≤30 characters;
- **Three modes — index / retrieve / direct extract**: a whole book is indexed first (generated from the TOC page; page-by-page scanning forbidden); extraction happens only after retrieval hits; dumping the whole book in one pass is forbidden in any mode;
- **Hierarchy flexibility**: when a subsection's points are complex or numerous, one deeper sub-level (e.g. `##### N.M.K`) may be added to group them;
- PDF extraction uses pymupdf; after 2 failed retries, stop debugging and ask the user to paste text — hand-written parsers forbidden;
- The v2.3 spec in `书籍思维导图skill 设计说明2.3.txt` is the single source of truth; copy and verify against the raw text, never the rendered view.
