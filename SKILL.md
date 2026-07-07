---
name: tech-doc-consistency-check
description: >
  Audits and auto-fixes Markdown technical documentation for structural and linguistic
  consistency, then outputs a summary table of all issues found and fixed. Use this skill
  whenever the user wants to validate, review, clean up, or proofread a Markdown file —
  even when they don't say "consistency check" explicitly. Trigger on: "check this doc
  before I publish", "the TOC links seem broken", "something looks off in the section
  numbers", "proofread my technical writeup", "review this doc for errors", "make sure
  the anchors work", "audit this markdown", or any request to verify or clean up a .md
  file. Checks section numbering, TOC completeness, TOC anchor links (GFM rules including
  em-dash handling), HTML anchor validity, cross-document link and anchor validity
  (following `file.md#anchor` links to sibling docs to confirm both the file and the
  anchor exist), tech-aware spelling, prose grammar, and
  hard-wrapped paragraphs that misrender as line breaks in some viewers —
  auto-fixing everything it can and flagging judgment calls for the user. Also handles
  packaging a Markdown doc and its locally referenced images into a ZIP archive, including
  bundling multiple sibling docs into one combined archive with a merged images/ folder —
  trigger on "zip up this doc", "package this doc into a zip file", "create a zip from this
  doc", "zip this markdown with its images", "zip up these docs", or any request to
  bundle/archive one or more .md files for sharing.
license: Apache-2.0
metadata:
  author: Vincent Yin
  version: "2.4.1"
---

# Tech Doc Consistency Checker

Your goal is to leave the document structurally sound and linguistically clean. Work through the checks below in order, fixing each issue as you go so subsequent checks operate on the corrected state. Auto-fix anything with a clear correct answer; flag anything that requires judgment.

---

## Check 1 — Section Numbering

Numbered headings are a navigation contract with the reader: when a section is added, removed, or moved, sibling numbers must be updated or that contract breaks. Verify that all numbered headings are consecutive with no gaps, at every level.

**Body order is authoritative.** When the TOC order and the body order disagree, renumber the body sections in place and update the TOC to match — never reorder body sections to match the TOC. The body is the document; the TOC is an index of it.

**How to check:**
1. Extract all headings with `grep -n "^#" <file>`, filtering out lines inside fenced code blocks.
2. For top-level numbered sections (e.g., `## 1.`, `## 2.`), verify they run 1, 2, 3, … with no skips.
3. For each subsection group (e.g., `### 2.1`, `### 2.2`), verify they run .1, .2, .3, … with no skips.
4. Verify subsection prefixes match their parent (e.g., all subsections of `## 3.` start with `3.`).

**Common issues:**
- A section was added or removed but siblings were not renumbered.
- A section was moved and the old number was left behind.
- Subsection numbering restarts from the wrong parent.

---

## Check 2 — TOC Completeness

A TOC that doesn't match the document body misleads readers: a phantom entry sends them nowhere, a missing entry hides content entirely.

Verify that every numbered heading in the document has a corresponding entry in the Table of Contents, and vice versa — no heading is missing from the TOC, and no TOC entry points to a non-existent heading.

**A document must have exactly one H1 heading.** Count all `# ` headings (excluding lines inside fenced code blocks). Flag if the count is zero or greater than one — list the line numbers of all H1 headings found. Do not auto-fix; the correct resolution depends on intent.

**H1 must not appear in the TOC.** Including the H1 in the TOC adds a spurious nesting level that pushes all real entries one indent deeper. If the TOC's root entry links to the H1, remove it and dedent all remaining entries by one level. (Only applies when the document has exactly one H1; if the H1 count is wrong, flag that first.)

**H1 must carry `<!-- omit in toc -->`.** Without this comment, TOC auto-generators (e.g., VS Code "Markdown All in One") will silently re-add the H1 to the TOC on the next regeneration. Auto-fix: append `<!-- omit in toc -->` to the H1 line if it is missing.

Example: `# My Guide` → `# My Guide <!-- omit in toc -->`

**The TOC heading itself must carry `<!-- omit in toc -->`.** The "Table of Contents" heading (or equivalent, e.g., `## Table of Contents`, `## Contents`) must not appear as an entry inside the TOC it introduces — that would be self-referential and meaningless. Without the comment, TOC auto-generators will silently re-add it on the next save. Auto-fix: append `<!-- omit in toc -->` to the TOC heading line if it is missing.

Example: `## Table of Contents` → `## Table of Contents <!-- omit in toc -->`

**How to check:**
1. Extract all headings (excluding the TOC heading itself and any `<!-- omit in toc -->` headings).
2. Extract all TOC entries.
3. Confirm a 1-to-1 match.

---

## Check 3 — TOC Anchor Links

Even a small mismatch in anchor case or punctuation silently breaks navigation in GitHub, GitLab, and most static site generators — the link renders fine but goes nowhere. Anchors must follow **GitHub-Flavored Markdown (GFM) rules** exactly.

**GFM anchor rules:**
1. Strip inline code backticks, keeping the inner text.
2. Convert to lowercase.
3. Remove every character that is not a letter, digit, space, hyphen, or underscore.
4. Replace each space with a hyphen — one-to-one, so two adjacent spaces become `--`.

**Examples:**
- `## 2. Phase A — Scaffold a Project` → `#2-phase-a--scaffold-a-project`
- `### 1.1. Install \`agents-cli\`` → `#11-install-agents-cli`
- `### 2.2. Type 1 Client: Using \`agent_engines\` SDK` → `#22-type-1-client-using-agent_engines-sdk`

The em-dash `—` is stripped, but the spaces on either side remain, producing `--` in the anchor. This is the most common source of anchor bugs in technical docs. Underscores are **preserved** (GFM keeps them), as the `agent_engines` example shows — a frequent trap, since a naive "strip non-alphanumerics" pass drops them and silently breaks the anchor.

Compare each computed anchor against what the TOC actually contains. Fix any mismatch.

---

## Check 4 — HTML Anchor Validity

Some documents use `<a id="..."></a>` anchors for cross-references not tied to headings. Orphaned anchors (defined but never linked) add noise; broken references (linked but undefined) leave readers stranded. An `<a id>` may also be the target of a **cross-doc** link from a sibling document, so this check looks beyond the current file before declaring an anchor orphaned.

1. Collect every `<a id="...">` definition in this document.
2. Collect every in-document `(#...)` reference that targets one of those IDs (i.e., not a heading anchor).
3. Collect every **cross-doc** reference to this document's anchors: scan the other `.md` files in the same directory for links of the form `[text](<THIS_FILENAME#id>)` or `[text](THIS_FILENAME#id)` (angle brackets optional), where `THIS_FILENAME` is the basename of the document being checked.
4. Verify every in-document reference (step 2) has a matching definition — otherwise it is a **broken reference**.
5. Verify every definition (step 1) has at least one reference, counting both in-document (step 2) and cross-doc (step 3) references. A definition with only a cross-doc reference is **not** orphaned.

Report orphaned anchors (no reference anywhere) and broken references.

---

## Check 5 — Spelling (Tech-Aware)

Misspelled prose erodes credibility. Tech docs are full of identifiers, product names, and domain jargon that a naive spellchecker would flag — strip those before scanning so you're only catching genuine mistakes.

Extract prose text by stripping:
- Fenced code blocks (` ``` `)
- Inline code (`` ` `` ... `` ` ``)
- URLs (`https?://...`)
- HTML tags
- Markdown link syntax `[text](url)`
- Markdown image syntax `![alt](url)`

Then scan the remaining prose for spelling errors.

**Do not flag:**
- Technical abbreviations: CLI, SDK, API, GCP, ADK, CI/CD, LLM, IDE, GKE, IAM, HTTP, URL, JSON, YAML, etc.
- Tool/product names: Terraform, Kubernetes, Dockerfile, Vertex AI, Cloud Run, Cloud Build, etc.
- Package/command names: `uvicorn`, `fastapi`, `pyproject`, `evalset`, `rubric`, etc. (even when appearing outside backticks in prose)
- File extensions and paths discussed in prose: `.env`, `.gitignore`, `.json`, etc.
- Domain jargon standard in the field: observability, evalset, rubrics, symlink, subcommand, metadata, hot-reload, etc.
- Intentional informal register (e.g., "Gotcha" in a "Key Gotchas" section)

**Do flag:**
- Misspelled common English words (e.g., `explicity` → `explicitly`, `recieve` → `receive`)
- Accidentally merged words (e.g., `Withose` → `With those`)

If `aspell` or `hunspell` is available, run:
```bash
cat <file> | aspell list --lang=en_US --mode=markdown | sort -u
```
and filter the output against the tech-term allowlist above. Otherwise, extract and scan prose manually using a Python/sed script.

---

## Check 6 — Basic Prose Grammar

Common grammar slips — missing prepositions, duplicate words, subject-verb disagreement — often survive drafting because readers unconsciously self-correct while reading. Explicit checking catches them.

While extracting prose for spelling (Check 5), also scan for:
- Missing prepositions (e.g., `resulting this` → `resulting in this`)
- Missing articles where clearly required (e.g., `in previous section` → `in the previous section`)
- Subject-verb agreement errors
- Duplicate words (e.g., `the the`)

**Do not flag:**
- Intentional terse style common in technical writing (omitting articles is sometimes acceptable)
- Non-native phrasing that is unambiguous and readable

---

## Check 7 — File & Cross-Doc Link Validity

Broken file links are silent — they render as valid Markdown but produce missing images or dead navigation when the document is viewed. Cross-document links carry a second, independent failure mode: the file may exist but the `#anchor` may point nowhere. This check follows every local link — plain file and cross-doc alike — and verifies the file **and** any anchor. No slack: a `file.md#anchor` link is valid only when the sibling file exists *and* the anchor resolves inside it.

Collect every link whose target is a local path — any `![alt](target)` or `[text](target)` where `target` is **not** a URL (does not start with `http://` or `https://`) and **not** a purely in-document anchor (does not start with `#`). A destination wrapped in angle brackets — `[text](<My Doc.md#anchor>)`, the form used when the path contains spaces — counts here; strip the surrounding `<` and `>` first.

Normalize and check each target:
1. Strip a surrounding `<…>` if present.
2. Split off a trailing `#fragment` (everything from the first `#` onward). Keep both the **file part** and the **fragment**.
3. Resolve the file part: relative paths against the directory containing the document being checked; absolute paths as-is. Verify the file exists on disk. If it does not → **broken link** (report; do not auto-fix — the correct path is unknowable without the user's input).
4. If a `#fragment` is present **and** the file part exists **and** the file part is a Markdown file (`.md`), open that file and verify the fragment resolves to a real anchor in it. The fragment resolves if it matches **either**:
   - the **GFM slug** of one of that document's headings (compute slugs with the Check 3 rules), **or**
   - an explicit `<a id="fragment">` anchor defined in that document.
   If it matches neither → **broken anchor** (report; do not auto-fix). Report the fragment and the target filename so the user can see which slug drifted.

A target whose file part is empty (`#anchor` alone) is a purely in-document anchor and is out of scope here — Checks 3 and 4 own it.

**Examples of targets to check:**
- `./images/diagram.png` — file must exist.
- `images/screenshot.jpg` (no leading `./`, still relative) — file must exist.
- `../shared/glossary.md` — file must exist (no fragment to validate).
- `/Users/vyin/docs/setup.md` — absolute path, file must exist.
- `<DevOps Guide to agents-cli.md#57-agent-engine-grant-the-runtime-service-agent-bucket-access>` — sibling `.md` must exist **and** a heading whose GFM slug is `57-agent-engine-grant-the-runtime-service-agent-bucket-access` must exist in it.
- `<DevOps Guide to agents-cli.md#adk-a2a-on-cloud-run>` — sibling `.md` must exist **and** an `<a id="adk-a2a-on-cloud-run">` (or a heading with that slug) must exist in it.

**Examples of targets to skip:**
- `https://example.com/docs` (URL)
- `http://localhost:8080` (URL)
- `#section-heading` (purely in-document anchor — Checks 3 and 4 own it)

---

## Check 8 — Image Alt Text

Empty alt text on images (`![](...) `) silently degrades accessibility and makes images unidentifiable when they fail to load. Every local image link must have non-empty alt text.

**How to check:**
1. Collect every image link `![alt](target)` where `target` is a local file path (not a URL, not an anchor).
2. If `alt` is empty, auto-fix by setting it to the filename portion of `target` (i.e., the last path segment, including extension).

**Example:**
- `![](images/agents-cli-deploy-to-agent-runtime.png)` → `![agents-cli-deploy-to-agent-runtime.png](images/agents-cli-deploy-to-agent-runtime.png)`

**Do not flag:**
- Images with non-empty alt text, even if the alt text doesn't match the filename.
- Remote images (URLs) — their alt text is out of scope for this check.

---

## Check 9 — Protocol Layering Precision

In diagrams and prose, protocols are often written as `X / Y` when X is actually a higher-level protocol defined on top of Y. That slash is ambiguous: it could mean "either X or Y", "X plus Y", or "X over Y" — three different things. When the relationship is layering, make it explicit.

**Rule:** When a label or phrase uses `X / Y` where X is a protocol defined on top of Y, replace it with `X (over Y)`.

**How to check:**
1. Scan all diagram text — arrow labels, node labels, table cells, and prose — for the pattern `<Term> / <Term>`.
2. For each match, determine the relationship between X and Y:
   - If X is a protocol layered on top of Y → rewrite as `X (over Y)`.
   - If the slash means "either one" → rewrite as `X or Y`.
3. This applies equally to node labels (e.g., `Gemini Enterprise or Orchestrator Agent`, not `Gemini Enterprise / Orchestrator Agent`) and arrow labels.

**Examples:**

| Before | After | Why |
|---|---|---|
| `HTTP / A2A` | `A2A (over HTTP)` | A2A is a protocol spec that runs on top of HTTP |
| `MCP / HTTPS` | `MCP (over HTTP)` | MCP (Model Context Protocol) is defined on top of HTTP |
| `OTLP / gRPC` | `OTLP (over gRPC)` | OTLP is the OpenTelemetry wire format; gRPC is its transport |
| `REST / HTTPS` | `REST (over HTTPS)` | REST is an architectural style applied over HTTPS |
| `Gemini Enterprise / Orchestrator Agent` | `Gemini Enterprise or Orchestrator Agent` | Two alternative callers, not a layered relationship — use "or" |

**HTTP vs HTTPS:** Use HTTP and HTTPS interchangeably when referring to network protocols in general (e.g. arrow labels in diagrams, prose descriptions of communication patterns). Only distinguish them when the term appears as a **URL prefix** in actual URLs (e.g. `https://...` in code, config, or links), where the distinction affects behavior (TLS enforcement, redirects). Do not flag or correct `HTTP` → `HTTPS` in general protocol descriptions.

**Do not flag:**
- `HTTP or gRPC` — the word "or" already makes the choice explicit.
- `TCP/IP` — this is a single compound name, not a slash-ambiguity case.
- `CI/CD` — an established compound abbreviation, not a protocol expression.
- Established compound industry terms where the slash is part of the term itself, not a separator between two independent concepts (e.g., `no-code / low-code`, `read / write`, `input / output`). When in doubt, ask: would removing one side leave the other side meaningful in context? If the two sides form a recognized spectrum or pair, leave the slash.

---

## Check 10 — Heading Level Increments

Skipping a heading level (e.g., jumping from `#` directly to `###`) breaks the document outline and confuses screen readers and document parsers that rely on a strict hierarchy.

**Rule:** When traversing headings in document order, the level of each heading may exceed the previous heading's level by **at most 1**. Decreasing by any amount is allowed.

**How to check:**
1. Extract all headings in order (excluding lines inside fenced code blocks), recording each heading's level (number of leading `#` characters).
2. For each consecutive pair, if `current_level > previous_level + 1`, flag it.
3. Report the line number, the offending heading text, and the skip (e.g., "jumped from level 2 to level 4").

**Examples:**

| Previous heading | Current heading | Level change | Valid? |
|---|---|---|---|
| `## Section` (2) | `### Subsection` (3) | +1 | ✓ |
| `# Title` (1) | `## Section` (2) | +1 | ✓ |
| `### Sub` (3) | `# Top` (1) | −2 | ✓ (decreasing is always allowed) |
| `# Title` (1) | `### Sub` (3) | +2 | ✗ skips level 2 |
| `## Section` (2) | `##### Deep` (5) | +3 | ✗ skips levels 3 and 4 |

Do not auto-fix — the correct repair depends on intent (promote the child, demote the child, or insert an intermediate heading).

---

## Check 11 — Filename vs. Title Agreement

A document's filename is often used as a URL slug or navigation label. When it drifts from the H1 title, external references and breadcrumbs become misleading.

**Rule:** The slug derived from the filename must match the slug derived from the H1 title. Flag any mismatch; do not auto-fix (renaming a file is a destructive filesystem operation).

**Slug derivation — apply identically to both the filename (minus `.md` extension) and the H1 title text:**
1. Strip inline code backticks, keeping the inner text.
2. Convert to lowercase.
3. Remove every character that is not a letter, digit, space, hyphen, or underscore.
4. Replace each space with a hyphen (one-to-one).

**How to check:**
1. Take the bare filename (strip the `.md` extension).
2. Take the text of the first `# ` heading in the document body.
3. Apply the slug rules to both.
4. If the two slugs differ, report: the filename slug, the title slug, and the specific mismatch.

**Examples:**

| Filename | H1 Title | Filename slug | Title slug | Match? |
|---|---|---|---|---|
| `my-caveman-agent4.md` | `# My Caveman Agent4` | `my-caveman-agent4` | `my-caveman-agent4` | ✓ |
| `devops-guide.md` | `# DevOps Guide` | `devops-guide` | `devops-guide` | ✓ |
| `deploy-guide.md` | `# Deployment Guide` | `deploy-guide` | `deployment-guide` | ✗ |
| `agents_cli_setup.md` | `# Agents CLI Setup` | `agents_cli_setup` | `agents-cli-setup` | ✗ (GFM preserves `_`, so the filename's underscores do not match the title's hyphens) |

**Do not flag:**
- Filenames that are intentionally abbreviated (judgment call — just report the mismatch so the user can decide).
- **Colon-to-hyphen substitution.** When the only difference is that the H1 title uses a colon (`:`) where the filename uses a hyphen — typically a space-padded ` - `, which slugifies to `---` against the title's stripped `:` — treat it as cosmetic and do not flag. Most filesystems disallow `:` in filenames, so a hyphen is the standard substitute. Example: `Claude Session Branching - Pros vs. Cons.md` (`claude-session-branching---pros-vs-cons`) vs `# Claude Session Branching: Pros vs. Cons` (`claude-session-branching-pros-vs-cons`) — differs only at the `:`/`-` boundary, so it is a match for this check's purposes.

**Always flag:**
- Documents with no H1 heading — report as "missing H1 title".
- Documents with more than one H1 heading — report as "multiple H1 titles" and list their line numbers. When there are multiple H1s, the slug comparison is undefined; skip the slug check and flag the H1 count issue only.

---

## Check 12 — Embedded Line Breaks in Paragraphs

A single newline inside a paragraph does **not** render identically across Markdown viewers. Strict CommonMark/GFM renderers (e.g., VS Code's built-in preview, `breaks: false`) collapse it to a space and reflow to window width — so hard wraps are invisible there. But renderers configured with markdown-it `breaks: true` emit a literal `<br>` for every single newline; common examples include **Markdown Preview Enhanced** (VS Code) and browser Markdown viewers like the Chrome "Markdown Reader" extension. In those, a hard-wrapped paragraph breaks mid-sentence at the source's arbitrary wrap columns and does **not** reflow to the window. The safe, portable form is one continuous line per paragraph (and per list item), which renders correctly under both `breaks: false` and `breaks: true`.

**Rule:** Each prose paragraph, and each individual list item, should occupy a single physical line. If a prose paragraph or a single list item is split across multiple physical lines by single (non-blank-separated) newlines, flag it as a hard-wrapped block. **Do not auto-fix** — detect the situation and offer to reflow it; apply the reflow only if the user accepts.

**How to check:**
1. Walk the document line by line, skipping anything inside fenced code blocks (` ``` `).
2. Identify candidate blocks as maximal runs of consecutive non-blank lines that are plain prose or a single list item. **Exclude** these line types from the rule (single newlines within them are legitimate structure, not hard wraps):
   - Headings (`#`, `##`, …)
   - Table rows (lines containing the `|` column structure)
   - Blockquote lines (`>`)
   - Raw HTML blocks
3. For a **prose paragraph** (a run of non-blank lines none of which begin a list item): if the run is more than one physical line, flag it — the lines belong to one paragraph and were hard-wrapped.
4. For a **list item**: treat the item's marker line plus any indented continuation lines as one logical item. If that item spans more than one physical line, flag it. (The newline *between two different list items* is correct and is not flagged; only continuation-line wrapping *within* one item is.)

**What to report:** the starting line number of each hard-wrapped block and a one-line preview. Then offer: *"Reflow these N block(s) to one continuous line each? (headings, table rows, and code blocks left untouched)"* Reflow only on confirmation, joining the wrapped physical lines of each block into a single line with single spaces, preserving list-item markers and indentation.

**Do not flag:**
- Blocks separated by a blank line — those are already distinct paragraphs/items.
- Line breaks inside fenced code blocks, tables, or blockquotes.
- A paragraph or list item that is already a single physical line.

---

## Execution Order and Summary

Run checks in this order; fixing as you go ensures later checks see the corrected state:

1. Section Numbering
2. TOC Completeness
3. TOC Anchor Links
4. HTML Anchor Validity
5. Spelling
6. Basic Prose Grammar
7. File & Cross-Doc Link Validity
8. Image Alt Text
9. Protocol Layering Precision
10. Heading Level Increments
11. Filename vs. Title Agreement
12. Embedded Line Breaks in Paragraphs

After all checks, report:

| Check | Issues Found | Fixed |
|-------|-------------|-------|
| Section Numbering | N | N |
| TOC Completeness | N | N |
| TOC Anchor Links | N | N |
| HTML Anchor Validity | N | N |
| Spelling | N | N |
| Prose Grammar | N | N |
| File & Cross-Doc Link Validity | N | — |
| Image Alt Text | N | N |
| Protocol Layering Precision | N | N |
| Heading Level Increments | N | — |
| Filename vs. Title Agreement | N | — |
| Embedded Line Breaks in Paragraphs | N | — |

If any issue required a judgment call and was not auto-fixed, list it explicitly below the table.

---

## On-Demand: Package Document(s) into a ZIP

**Only do this when the user explicitly asks** — e.g., "package the doc into a ZIP file", "ZIP up this doc", "create a zip file for this doc", "zip up these docs". Do NOT run this automatically as part of the consistency check.

Creates a ZIP archive containing the document(s) and all locally referenced image files, preserving the relative folder structure so images load correctly when extracted anywhere. This handles both a single doc and multiple docs bundled into one archive.

### Single Document

Run from the directory containing the document:

```bash
cd "<doc-directory>" && \
zip -j "<doc>.zip" "<doc>" && \
grep -o '](images/[^)]*' "<doc>" | sed 's/^](//' | sort -u | while read img; do
  zip "<doc>.zip" "$img"
done
```

The `-j` flag places the document at the ZIP root (no parent path). The image loop adds each image under its relative `images/` path. Only files actually referenced in the document are included — not the entire `images/` folder.

**Naming:** If the user dictated an archive name in the request (e.g., "zip this doc to `tech-guide.zip`"), honor that name exactly. Otherwise name it the same as the document filename (e.g., `My Guide.md` → `My Guide.zip`). Place it in the same directory.

### Multiple Documents (One Combined ZIP)

When the user asks to bundle 2 or more docs into **one** archive (rather than zipping each separately), the docs are all **siblings** — at the same folder level — so their on-disk layout already parallels the ZIP layout and no folder re-mapping is needed. The archive gets **one combined `images/` folder** holding the union of images referenced by any of the docs.

Run from the shared directory:

```bash
cd "<doc-directory>" && \
zip -j "<archive>.zip" "<doc1>" "<doc2>" [...] && \
cat "<doc1>" "<doc2>" [...] | grep -o '](images/[^)]*' | sed 's/^](//' | sort -u | while read img; do
  zip "<archive>.zip" "$img"
done
```

The `-j` flag places all docs at the ZIP root, side by side. The `cat ... | grep` collects image references across **all** the docs at once, and `sort -u` dedupes them, so an image referenced by more than one doc is added only once into the single shared `images/` folder.

**Naming:** If the user dictated an archive name in the request (e.g., "zip up both docs to `tech-guide.zip`"), honor that name exactly. Otherwise no single filename applies, so invent a short name that summarizes the set — e.g., for `DevOps Guide to agents-cli.md` + `Infra Guide to GCP AI Agents.md`, a name like `DevOps and Infra Guide.zip` or `Guide to Agents.zip`. Place it in the same directory.

### After Zipping (Both Cases)

Move the finished `.zip` to `~/Downloads`.
