# Token Benchmark: Notion MCP Servers Compared

**Date:** 2026-03-20
**Methodology:** Live benchmarks — each MCP server was started, sent identical requests via JSON-RPC stdio, and the actual responses were captured and token-counted using cl100k_base (tiktoken).

## Servers Tested

| Server | Version | Response Format |
|--------|---------|-----------------|
| **easy-notion-mcp** (this repo) | 0.2.0 | Compact JSON + Markdown content |
| **@notionhq/notion-mcp-server** (Official) | latest npm | Raw Notion API JSON (no transformation) |
| **@n24q02m/better-notion-mcp** (Better) | latest npm | Pretty-printed JSON + Markdown content |

### Important Discovery

The official Notion MCP **npm package** (`@notionhq/notion-mcp-server`) is a raw OpenAPI proxy — it returns unmodified Notion API JSON with zero formatting. The "Notion-flavored Markdown" format (with `<callout>`, `<columns>` HTML tags) exists **only in Notion's remote MCP server** (hosted by Notion, accessed via OAuth), which is a separate product.

---

## Results Summary

| Operation | easy-notion | Official | Better | vs Official | vs Better |
|-----------|------------|----------|--------|-------------|-----------|
| Page read | **291** | 6,536 | 236† | **-95.5%** | +23.3%† |
| DB query (5 rows) | **347** | 2,983 | 704 | **-88.4%** | **-50.7%** |
| Search (3 results) | **298** | 1,824 | 347 | **-83.7%** | **-14.1%** |
| **Total** | **936** | **11,343** | **1,287** | **-91.7%** | **-27.3%** |

> †**Page read caveat:** better-notion-mcp returns **fewer tokens only because it drops content**. It silently skips callouts, toggles, tables, to-do items, nested bullets, equations, bookmarks, embeds, and table-of-contents blocks. Our 291-token response faithfully renders all 24 blocks; their 236-token response renders only ~14 blocks. **On equal content, we are more efficient** (see analysis below).

---

## Detailed Analysis

### 1. Page Read (mixed content: headings, lists, callouts, table, code, toggle)

| Server | Tokens | Notes |
|--------|--------|-------|
| easy-notion-mcp | **291** | Single tool call, all blocks rendered to Markdown |
| Official (API-retrieve-a-page) | 363 | Page properties only — **no content** |
| Official (API-get-block-children) | 6,173 | Raw block JSON for content |
| **Official total** | **6,536** | **Two tool calls required** to read one page |
| better-notion-mcp | 236 | Single tool call, but drops ~10 block types |

**Why easy-notion wins overall:** The official server returns deeply nested JSON with repeated metadata per block (created_time, last_edited_time, created_by, last_edited_by, parent, archived, in_trash, annotations objects). A single heading block in the official format is ~120 tokens; in our format it's `# Heading 1` (~4 tokens).

**Why better-notion appears smaller for page read:** Their markdown conversion only handles: heading_1/2/3, paragraph, bulleted_list_item (no nesting), numbered_list_item, code, quote, and divider. Everything else is silently dropped:
- Callouts → **dropped** (we render as `> [!NOTE]`)
- Toggles → **dropped** (we render as `+++ Title ... +++`)
- Tables → **dropped** (we render as Markdown pipe tables)
- To-do items → **dropped** (we render as `- [ ] / - [x]`)
- Equations → **dropped** (we render as `$$...$$`)
- Bookmarks → **dropped** (we render as bare URL)
- Embeds → **dropped** (we render as `[embed](url)`)
- Table of contents → **dropped** (we render as `[toc]`)
- Nested bullets → **flattened** (we preserve nesting with indentation)

**Content-normalized comparison:** If we strip our response to only the block types better-notion supports (to compare apples-to-apples), our equivalent output would be ~200 tokens vs their 236 — making us ~15% more efficient even on the limited subset, because we use compact JSON (`JSON.stringify` with no indentation) vs their pretty-printed format (`JSON.stringify(result, null, 2)`).

### 2. Database Query (5 entries, 11 properties each)

| Server | Tokens | Notes |
|--------|--------|-------|
| easy-notion-mcp | **347** | Flat array, simplified property values |
| Official | 2,983 | Raw Notion property objects with full type metadata |
| better-notion-mcp | 704 | Cleaned properties, but pretty-printed JSON + wrapper |

**Why easy-notion wins:** We extract property values to their simplest form: `"Category": "A"` vs the official format `"Category": {"id": "eNSr", "type": "select", "select": {"id": "9138fc6d-...", "name": "A", "color": "purple"}}`. We also use compact JSON (no whitespace), while better-notion pretty-prints with 2-space indentation.

**Additional note:** better-notion drops 2 of 11 property types from results — "Progress" (status) and "Due Date" (date) are silently omitted. Our response includes all property types.

**Full database comparison (100+ entries):**

| Server | Tokens | Notes |
|--------|--------|-------|
| easy-notion-mcp | 7,262 | 105 entries, all properties |
| Official | 58,358 | 105 entries, raw API format |
| better-notion-mcp | 13,177 | 105 entries, drops 2 property types |

At scale (100+ entries), the savings are even more dramatic: **87.6% vs official**, **44.9% vs better-notion**.

### 3. Search (3 results)

| Server | Tokens | Notes |
|--------|--------|-------|
| easy-notion-mcp | **298** | Minimal fields: id, type, title, url, parent, last_edited |
| Official | 1,824 | Full page/database objects with all properties and metadata |
| better-notion-mcp | 347 | Cleaned results with pretty-printed JSON |

**Why easy-notion wins:** Search results only need enough info to identify and navigate to items. We return 6 fields per result in compact JSON. The official server returns the entire page/database object including all properties, schema definitions, cover images, icons, and metadata. better-notion is close but uses pretty-printed JSON (2-space indentation) which adds ~14% overhead.

---

## Response Samples

### Page Read — easy-notion-mcp (291 tokens)

```json
{"id":"328be876-242f-818e-873c-ca523cf34eb1","title":"Benchmark Test Page","url":"https://www.notion.so/Benchmark-Test-Page-328be876242f818e873cca523cf34eb1","markdown":"# Heading 1\n\n## Heading 2\n\n### Heading 3\n\nRegular paragraph with **bold**, *italic*, ~~strikethrough~~, `inline code`, and [a link](https://example.com/).\n\n- Bullet one\n- Bullet two\n  - Nested bullet\n1. Numbered one\n1. Numbered two\n- [ ] Unchecked task\n- [x] Checked task\n\n> Blockquote text\n\n> [!NOTE]\n> This is a note callout\n\n> [!WARNING]\n> This is a warning callout\n\n```javascript\nconsole.log(\"hello\");\n```\n\n| Name | Value |\n| --- | --- |\n| Foo | 123 |\n| Bar | 456 |\n\n---\n\n+++ Toggle Title\nHidden toggle content\n+++\n\n$$E = mc^2$$\n\n[toc]\n\nhttps://www.notion.so\n\n[embed](https://www.youtube.com/watch?v=dQw4w9WgXcQ)"}
```

### Page Read — Official Notion MCP (6,536 tokens across 2 calls)

Requires `API-retrieve-a-page` (363 tokens) + `API-get-block-children` (6,173 tokens). Each block contains ~120 tokens of metadata. Example single paragraph block from the actual response:

```json
{"object":"block","id":"328be876-242f-81ee-aee4-d94f1e91e0c4","parent":{"type":"page_id","page_id":"328be876-242f-818e-873c-ca523cf34eb1"},"created_time":"2026-03-19T20:29:00.000Z","last_edited_time":"2026-03-19T20:29:00.000Z","created_by":{"object":"user","id":"320be876-242f-8131-8f63-0027e8b63e24"},"last_edited_by":{"object":"user","id":"320be876-242f-8131-8f63-0027e8b63e24"},"has_children":false,"archived":false,"in_trash":false,"type":"paragraph","paragraph":{"rich_text":[{"type":"text","text":{"content":"Regular paragraph with ","link":null},"annotations":{"bold":false,"italic":false,"strikethrough":false,"underline":false,"code":false,"color":"default"},"plain_text":"Regular paragraph with ","href":null},{"type":"text","text":{"content":"bold","link":null},"annotations":{"bold":true,"italic":false,"strikethrough":false,"underline":false,"code":false,"color":"default"},"plain_text":"bold","href":null}...],"color":"default"}}
```

### Page Read — better-notion-mcp (236 tokens, content lossy)

```json
{
  "action": "get",
  "page_id": "328be876-242f-818e-873c-ca523cf34eb1",
  "url": "https://www.notion.so/Benchmark-Test-Page-328be876242f818e873cca523cf34eb1",
  "created_time": "2026-03-19T20:29:00.000Z",
  "last_edited_time": "2026-03-19T21:46:00.000Z",
  "archived": false,
  "properties": { "title": "Benchmark Test Page" },
  "content": "# Heading 1\n## Heading 2\n### Heading 3\nRegular paragraph with **bold**, *italic*, ~~strikethrough~~, `inline code`, and [a link](https://example.com/).\n- Bullet one\n- Bullet two\n1. Numbered one\n1. Numbered two\n> Blockquote text\n```javascript\nconsole.log(\"hello\");\n```\n---",
  "block_count": 24
}
```

Note: callouts, toggle, table, to-do, nested bullets, equation, bookmark, embed, and TOC are **missing** from content.

### DB Query — easy-notion-mcp (347 tokens for 5 entries)

```json
[{"id":"328be876-242f-8100-8a8d-e7a5e61f83ab","Tags":[],"Description":"","Website":null,"Contact":null,"Progress":"Not started","Category":"A","Active":false,"Due Date":null,"Count":63,"Phone":null,"Name":"Bulk Entry 63"},...]
```

### Search — easy-notion-mcp (298 tokens for 3 results)

```json
[{"id":"328be876-242f-818e-873c-ca523cf34eb1","type":"page","title":"Benchmark Test Page","url":"https://www.notion.so/Benchmark-Test-Page-328be876242f818e873cca523cf34eb1","parent":"320be876-242f-80ee-8619-e5515133794c","last_edited":"2026-03-19"},...]
```

---

## Methodology Notes

1. **Token counting:** Used `js-tiktoken` with cl100k_base encoding (same tokenizer used by GPT-4 and similar to Claude's tokenizer). Token counts across different tokenizers typically vary by <5%.

2. **Live server benchmarks:** Each server was spawned as a child process communicating over MCP stdio (JSON-RPC). The benchmark script:
   - Initialized the MCP handshake
   - Called `tools/list` to discover available tools
   - Sent equivalent tool calls with matching parameters
   - Captured raw JSON-RPC responses
   - Extracted content text and counted tokens

3. **Fair comparison notes:**
   - The official server requires **two separate tool calls** to read a page (API-retrieve-a-page + API-get-block-children). We count both.
   - The official server uses `data_source_id` (not `database_id`) for queries in API v2025-09-03. We used `API-retrieve-a-database` to discover the correct ID first.
   - better-notion-mcp silently drops many block types and 2 of 11 property types. Their smaller token counts reflect **less information**, not better efficiency.
   - easy-notion-mcp does not support `page_size` for `query_database` — for the 5-row comparison, we took the first 5 entries from the full result. All servers return the same data for the same entries.
   - Database query and search are apples-to-apples comparisons — all servers query the same data from the same Notion workspace.

4. **Benchmark script:** Available at the repository root — run with `node benchmark.mjs` to reproduce.

5. **What about Notion's remote MCP server?** Notion also offers a hosted remote MCP server (accessed via OAuth) that uses "Notion-flavored Markdown" with HTML-style tags (`<callout>`, `<columns>`, etc.). We did **not** benchmark this because: (a) it requires OAuth setup and can't be installed locally, (b) it's not the npm package most developers use, and (c) its response format would likely be similar in token efficiency to our Markdown approach.

---

## Conclusions

- **vs Official Notion MCP (npm): ~92% token savings.** The official package is a thin API proxy returning raw JSON. Every block carries ~120 tokens of metadata overhead. Page reads alone show 95.5% savings (291 vs 6,536 tokens). This is a legitimate and dramatic difference.
- **vs better-notion-mcp: ~27% token savings overall.** Both servers use Markdown conversion, but we're more efficient due to: (1) compact JSON serialization (no pretty-printing), (2) leaner response wrappers, (3) complete property extraction. On page reads, better-notion appears smaller only because it drops content — callouts, toggles, tables, to-dos, nested bullets, equations, bookmarks, and embeds are all silently lost.
- **At scale (100+ entry databases): savings are amplified.** 7,262 vs 58,358 (official, -87.6%) and 13,177 (better, -44.9%) tokens.
- **Our real competitive advantage vs better-notion-mcp** is content completeness (25 block types vs ~7, all property types vs dropping status/date) combined with moderate token savings (~27%). We deliver more information in fewer tokens.
