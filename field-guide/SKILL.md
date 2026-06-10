---
name: field-guide
description: Orient a researcher in an unfamiliar research area — producing a structured report covering foundational papers, competing approaches, open problems, a sequenced reading order, and a subfield map — by composing get_field_orientation (cheap retrieval), search_papers, get_paper, and fetch_fulltext with in-context synthesis. Use when the user asks "I'm new to X, where do I start," "orient me in research area Y," "what should I read to get into Z," "give me a reading path for topic W," "map the subfields of A," "foundational papers for B," "survey of approaches to C," or "help me understand the landscape of D." get_field_orientation returns CANDIDATE PAPERS only (cheap retrieval, not a pre-synthesized report) — the synthesis is your job. Requires the scholar-feed MCP server; falls back to web search on arxiv.org if it is not connected. Free API keys & docs: https://www.scholarfeed.org/developers
---

# Field Guide — research-area orientation skill

This skill teaches you to answer "I'm new to area X — where do I start?" by producing a structured orientation report. The output covers the five things a researcher needs to enter a new field: foundational papers with rationale, competing approaches and their trade-offs, open problems with supporting evidence, a sequenced reading order, and a subfield map showing the major communities and how they relate.

**The synthesis is done in-context, not by a tool.** `get_field_orientation` is a cheap retrieval primitive: it returns a ranked list of *candidate* foundational papers via fast vector search over the corpus. It does not cluster, sequence, identify open problems, or map subfields. You do all of that — clustering the candidates, deepening coverage with `search_papers` and `fetch_fulltext`, and assembling the report.

Why is the synthesis in the skill rather than a tool? Because different researchers entering the same field need different framings. A systems researcher entering NLP needs a different reading path than a theorist entering NLP. A single pre-synthesized report can't serve both; in-context synthesis adapts to the user's background and goals. The retrieval half is deterministic and corpus-dependent, so it's a tool; the synthesis half is judgment, so it's this skill.

---

## Tool composition

The scholar-feed MCP exposes its tools prefixed `mcp__scholar-feed__`; bare names are used below for readability.

| Step | Tool | Purpose |
|---|---|---|
| Get candidate foundational papers | `get_field_orientation(topic="<area>")` | CHEAP retrieval only — a ranked list of candidate foundational papers via vector search. Does NOT synthesize, cluster, sequence, or map. You do that in the steps that follow |
| Find recent activity per subfield | `search_papers(q="<subtopic>", sort='recent', limit=10)` | Surfaces papers from the last 12–24 months in each subfield, biasing toward active threads and open problems |
| Surface papers that cite an anchor and discuss limits | `search_papers(scope_to_citations_of=<anchor_arxiv_id>, q="limitation OR challenge OR open problem OR future work", limit=5)` | Papers that build on a foundational anchor AND name what's unsolved — the best source for the open-problems section |
| Retrieve structured metadata for the shortlist | `get_paper(arxiv_ids=[...])` | `title`, `authors`, `abstract`, `llm_summary`, `llm_novelty_score`, `citation_count` for the papers selected for the report |
| Deep-read the 3–5 most foundational papers | `fetch_fulltext(arxiv_id=X, sections='all')` | Full paper text grouped by section — the substrate for synthesis. Intro/related-work reveal positioning; conclusion/discussion name open problems; method sections ground the approach comparison |
| In-context synthesis | (no tool — you assemble the report) | Combine candidates, recent activity, metadata, and deep reads into the five-part orientation report |

---

## Workflow (6 steps)

### Step 1 — Call get_field_orientation for the candidate set

Call `get_field_orientation(topic="<the user's area>")`. Be specific: "diffusion models for protein structure prediction" beats "AI for biology." The call is cheap (fast vector retrieval) and returns a ranked list of candidate foundational papers — those the corpus considers most representative of the topic's foundational literature.

**This is candidates, not the final output.** The response is a retrieval result, not a synthesized report. Do NOT present it directly to the user. Treat it as a reading list to cluster, evaluate, deepen, and synthesize in the steps below.

Scan the candidates and note:
- Which papers look most foundational by citation count and `llm_novelty_score`
- Which papers cluster naturally by method, time period, or sub-problem
- Which time periods and approach families are well-represented vs. sparse

### Step 2 — Cluster candidates into subfields and schools of thought

In context (no tool call), cluster the Step 1 candidates into groups that reflect how researchers actually partition the field:

- **By school of thought** (e.g. NLP: statistical → neural → pre-trained → instruction-tuned)
- **By sub-problem** (e.g. RL: model-free vs. model-based; on-policy vs. off-policy; discrete vs. continuous action spaces)
- **By approach family** (e.g. protein structure: homology-based vs. energy-based vs. ML-based → within ML: CNN vs. GNN vs. transformer)
- **By recency tier** (foundational / matured / actively evolving)

This clustering becomes the skeleton of the subfield map in Step 5. Write the clusters down before Step 3 — the cluster labels determine the `search_papers` queries you run next.

### Step 3 — Probe recent activity and open problems via search_papers

For each cluster from Step 2, run `search_papers(q="<subfield name>", sort='recent', limit=10)` to surface papers from the last 12–24 months. These serve two purposes:

1. **Currency** — they show the reading path leads to where the field is working today, not just its history.
2. **Open problems** — recent papers spend real space naming what is still unsolved; their introductions and related-work sections are the best source for a field's current open problems.

For each cluster, also run `search_papers(scope_to_citations_of=<canonical_foundational_paper>, q="limitation OR challenge OR open problem OR future work", limit=5)`. Papers that cite a foundational anchor and discuss its limits are often the ones that shaped the next generation of work — exactly what a new reader should read to see the frontier.

### Step 4 — Deep-read the 3–5 most foundational papers

From the candidate set (Step 1) and the recent probes (Step 3), pick the 3–5 papers that are most foundational — the ones any colleague in the area would expect you to have read. Call `fetch_fulltext(arxiv_id=X, sections='all')` for each and read:

- **Introduction** → how the paper frames the problem and positions itself against alternatives (primary source for the competing-approaches framing in Step 5)
- **Related work** → which prior approaches it engages with and why it finds them insufficient (the most concise survey of the landscape at publication time)
- **Conclusion and Discussion** → what the authors say they did not solve, what assumptions limit them, what they suggest next (best source for open problems)
- **Method** → enough to characterize how this approach is architecturally distinct from alternatives in its cluster

Cap the deep read at 3–5 papers to stay within a reasonable context budget. Additional candidates stay at abstract+summary depth from Steps 1–3.

### Step 5 — Synthesize the orientation report in context

Assemble the five-part report from what you read:

**Foundational papers with rationale** (3–7, ordered by importance): for each give `arxiv_id`, title, one-sentence positioning ("what problem does this solve and why does it matter"), and why it's foundational ("why anyone entering this field needs to read it"). These are not necessarily the most-cited papers — they are the ones that collectively establish the vocabulary, problem framing, and primary approaches current work builds on.

**Competing approaches** (the major families from your Step 2 clustering): name each, give a 2–3 sentence characterization of its core assumptions, cite its 1–2 canonical papers, and describe the key trade-off vs. the other families. Answer "why does this family exist and what does it give up," not "which is best" (that depends on the user's task).

**Open problems** (3–5 named problems): name each, cite the papers that defined it, and describe what makes it hard. Draw from the conclusion/discussion sections (Step 4) and recent search results (Step 3). Prefer problems that are concrete ("robustness to distribution shift") over vague ("making models more capable") and currently active (recent papers exist on them).

**Reading order** (a sequenced path): order the foundational papers so a reader building from the start never needs knowledge a later paper provides. Label each with a depth cue ("skim the intro," "read in full," "focus on the method," "read only if you'll work on X"). A good reading order is not citation order or chronological order — it is the order that minimizes how often the reader has to stop and look something up.

**Subfield map** (prose or a list of communities): describe the major communities, how they differ (problem framing, benchmarks, venues, naming conventions), and how they relate (do one community's methods feed another's? competing approaches to one problem, or adjacent problems?). A reader who knows this map can navigate conferences, understand why a paper uses a given benchmark, and know who to talk to about what.

### Step 6 — Emit the structured orientation report

Output the five-part report in a form the user can act on. Open with a one-paragraph executive summary: what the field is fundamentally about, where it is now, and what the reading path will teach. Close with the open problems — they show where the frontier is and what contributions are still needed.

Format each section with clear headers or bold labels so the user can navigate non-linearly. Someone planning three weeks of reading will jump between sections — make that easy.

---

## Anti-patterns

- **Don't present `get_field_orientation` output as the answer.** It returns candidate papers — a ranked retrieval result, not a synthesized report. It does not cluster, sequence, name open problems, or map subfields. That synthesis is Steps 2–6, done in context. Handing the raw candidate list to the user skips the entire value of the skill.
- **Don't stop at one `search_papers` call.** Orientation needs coverage across subfields. Probe each cluster from Step 2 separately; the corpus's value is the citation graph and rising-work signal, not a single keyword hit.
- **Find papers near an anchor with `search_papers(anchor_paper_id=<arxiv_id>)`**, and **search within a paper's citations with `search_papers(scope_to_citations_of=<arxiv_id>, q=...)`** — these are parameters on `search_papers`, not separate tools.
- **Don't fabricate the reading order from citation dates.** Chronological ≠ pedagogical. Sequence by prerequisite depth (Step 5).
- **A method-comparison request is different.** If the user wants a head-to-head benchmark table of specific named methods, that's a comparison task, not an orientation — this skill produces an overview of approach *families*, not benchmark scores.

---

## Graceful degradation — when the MCP isn't connected

If `mcp__scholar-feed__*` tools return errors indicating the server is not available:

1. Tell the user: "Scholar Feed MCP isn't connected — Field Guide uses it for the `get_field_orientation` retrieval and the `fetch_fulltext` deep-reads. To connect it: https://www.scholarfeed.org/developers"
2. Fall back to web search via arxiv.org, Google Scholar, and Semantic Scholar (semanticscholar.org). You can still produce an orientation report, but warn the user: without `get_field_orientation`, candidate selection rests on your training-data knowledge rather than live corpus retrieval, so for fast-moving areas the reading path may miss recent work.
3. The five-part structure (foundational papers, competing approaches, open problems, reading order, subfield map) is still the right output shape in fallback mode — the structure is the skill's value, independent of how papers are retrieved.
4. The synthesis-in-context constraint still applies. Don't try to find a backend endpoint that returns a pre-built report; the adaptive in-context synthesis is the point.

---

## Quick reference

- `get_field_orientation(topic=...)` → CANDIDATES, not a report. Cluster and synthesize in Steps 2–6.
- Deep-read only the 3–5 most foundational papers via `fetch_fulltext(sections='all')`; keep the rest at abstract+summary depth.
- Open problems come from the conclusion/discussion sections of deep-read papers plus `search_papers(scope_to_citations_of=ANCHOR, q="limitation OR challenge OR open problem")`.
- Reading order minimizes prerequisite look-ups — it is neither citation order nor chronological order.
- The synthesis is yours, in context, on purpose: different users need different orientation framings.

---

## See also

- General Scholar Feed research patterns (the deep-research loop): the [`scholar-feed`](../scholar-feed/SKILL.md) skill — a good follow-up once the user has oriented and wants to go deep on a specific thread.
- Free API keys & MCP setup: https://www.scholarfeed.org/developers
