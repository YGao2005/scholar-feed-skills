# Scholar Feed — Agent Skills

Agent Skills that turn Claude into a research copilot over **600,000+ CS/AI/ML
arXiv papers** — literature reviews, prior-art search, tracing how a method
evolved through the citation graph and foundational lineage, and full-text
reading — going beyond what a single web search returns.

These skills are built on the **Scholar Feed MCP connector**. Each skill is a
self-contained `SKILL.md`; Claude activates it automatically when a request
matches its description.

## Skills in this repository

| Skill | Description |
|-------|-------------|
| [`scholar-feed`](./scholar-feed/SKILL.md) | Deep CS/AI/ML research over the Scholar Feed MCP. Find, survey, read, and reason about papers: literature reviews, "recent work / SOTA in X", prior art, tracing how a method evolved, finding what supersedes a paper, comparing methods, and reading full text. Runs the deep-research loop (search → foundational lineage → cited-by → trending → full text) instead of stopping at one search, and grounds every claim in retrieved papers. |
| [`field-guide`](./field-guide/SKILL.md) | Orient a researcher in an unfamiliar area — "I'm new to X, where do I start?" Produces a structured report: foundational papers with rationale, competing approaches and trade-offs, open problems, a sequenced reading order, and a subfield map. Composes the cheap `get_field_orientation` retrieval primitive with `search_papers` + `fetch_fulltext` deep-reads, then synthesizes the report in context so the framing adapts to the reader's background. |

## Requirements

These skills call the **Scholar Feed MCP server**:

- **Remote connector:** `https://mcp.scholarfeed.org/mcp` (OAuth — connect it in Claude)
- **npm (stdio):** [`scholar-feed-mcp`](https://www.npmjs.com/package/scholar-feed-mcp) · source: [YGao2005/scholar-feed-mcp](https://github.com/YGao2005/scholar-feed-mcp)
- **Free API keys & docs:** https://www.scholarfeed.org/developers

If the connector is not connected, the `scholar-feed` skill falls back to a
plain web search on arxiv.org, so it degrades gracefully.

## Using a skill

Add a skill by placing its folder (e.g. `scholar-feed/`) where your Claude
client loads Agent Skills, then connect the Scholar Feed MCP server. Claude
picks the skill up automatically when your request matches it — for example:
"survey recent work on long-context transformers," "trace the lineage of
attention," or "what's the prior art for my idea about X."

## License

[MIT](./LICENSE) © 2026 Scholar Feed
