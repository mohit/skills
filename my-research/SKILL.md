---
name: deep-research
description: >
  Multi-agent deep research over specific data sources: email archives, note folders, document collections,
  academic papers, codebases, or any structured corpus. Use when asked to research, analyze, or synthesize
  information from a user's own data (Gmail, Obsidian, files, etc.) or a defined set of sources. Web search
  is used only for cross-checking, enrichment, and context — not as primary source. Triggers on: "research",
  "deep dive", "analyze these emails", "go through my notes on", "synthesize", "build a report from",
  "what patterns do you see in", or similar corpus-analysis tasks.
---

# Deep Research

Multi-agent research system for corpus-level analysis of specific data sources.

## When to Use

- Analyzing email archives between people or on topics
- Synthesizing themes from a folder of notes or documents
- Building reports from academic papers or technical docs
- Cross-referencing multiple data sources (emails + calendar + notes)
- Any task requiring systematic processing of more content than fits in one context window

## Architecture

Orchestrator-worker pattern. Three phases, strict separation of concerns:

```
User Query
    │
    ▼
┌──────────────┐
│  Lead Agent   │  Plans, delegates, verifies, synthesizes
│  (main/opus)  │  NEVER does primary collection
└──────┬───────┘
       │ sessions_spawn (parallel)
       ▼
┌────────────┐ ┌────────────┐ ┌────────────┐
│ Collector 1 │ │ Collector 2 │ │ Collector N │  ← Wave 1: Collection
│ (sonnet)    │ │ (sonnet)    │ │ (sonnet)    │
└──────┬──────┘ └──────┬──────┘ └──────┬──────┘
       │               │               │
       ▼               ▼               ▼
   /tmp/research-<id>/agent-1/raw/  agent-2/raw/  agent-N/raw/
       │               │               │
       └───────────────┴───────────────┘
                       │
                 Lead verifies → spawns gap-fillers if needed
                       │
                       ▼
              ┌─────────────────┐
              │  Assembly Agent  │  ← Wave 2: Clean, deduplicate, structure
              │  (sonnet)        │     Output: archive files
              └────────┬────────┘
                       │
              ┌─────────────────┐
              │  Analysis Agent  │  ← Wave 3: Interpret, analyze, synthesize
              │  (sonnet)        │     Output: analysis files
              └────────┬────────┘
                       │
                 Lead verifies → delivers to user
```

**Critical**: Collection, assembly, and analysis are SEPARATE agents. Combining them produces either data-without-analysis or analysis-without-data.

## Process

### Phase 1: Plan

1. Parse the user's query — what sources, what questions, what output format
2. Identify data sources and access methods (see [references/data-sources.md](references/data-sources.md))
3. Estimate corpus size. Run a quick count query first (e.g., `gog gmail search ... --limit 1` to see total, or `find ... | wc -l`)
4. Determine subagent count and scope splits (see scaling guidelines)
5. Create scratchpad: `mkdir -p /tmp/research-<timestamp>/agent-{1..N}/raw`

### Phase 2: Collect (Wave 1)

Spawn **collection agents** in parallel. Each collector:
- Has a specific, narrow scope (time range, folder, account)
- Dumps ALL raw content to files FIRST: `/tmp/research-<id>/agent-N/raw/<item-id>.txt`
- THEN reads through raw files and writes structured findings to `/tmp/research-<id>/agent-N/findings.md`
- Reports statistics: items found, items processed, gaps detected

**The dump-first pattern is mandatory.** Agents that try to fetch-and-summarize in a single pass invariably skip items when they hit context limits. Dumping to files first means the data is preserved even if the agent runs out of budget for summarization.

**Scope sizing**: Each Sonnet agent can reliably process ~50-80 items. For larger scopes, split into smaller agents rather than trusting one agent with 200+ items. A 6-month email window is better than a 2-year window.

### Phase 3: Verify & Fill Gaps

After all collectors complete, the lead agent MUST:
1. Check each agent's output: `wc -l /tmp/research-<id>/agent-*/findings.md`
2. Spot-read 3-5 entries from each agent — do they have actual content or just metadata stubs?
3. Compare item counts: if an agent found 100 items but its findings.md has 15 entries, it under-processed
4. Check boundary coverage: are the earliest/latest dates what we expected?
5. Spawn gap-filling agents for any under-processed scopes

### Phase 4: Assemble (Wave 2)

Spawn an **assembly agent** that reads ALL findings files and produces clean output:
- Deduplicate (same item from overlapping agent scopes)
- Merge chronologically
- Clean formatting artifacts
- Filter false-positive links (email header domains, etc.)

**Assembly mandate**: The assembly agent's job is to CLEAN and DEDUPLICATE, not to SUMMARIZE. Output should be LONGER than or equal to total input minus duplicates. If the combined findings are 3,000 lines, the assembled archive should not be 600 lines.

### Phase 5: Analyze (Wave 3)

Spawn an **analysis agent** that reads the assembled archive and writes interpretation:
- Thematic analysis, patterns, evolution over time
- The specific analytical angles the user requested
- Honest assessment — not everything is significant
- Cites specific entries by date/subject as evidence

**Analysis agents work best when they ONLY analyze.** They should read finished data, not collect it. This separation consistently produces better writing.

### Phase 6: Deliver & Clean Up

1. Lead verifies final output files exist and pass quality gates
2. Delivers to user-specified location
3. Cleans up: `rm -rf /tmp/research-<id>/`

## Scaling Guidelines

| Corpus Size | Collectors | Scope per Agent | Example |
|---|---|---|---|
| < 50 items | 1-2 | All items | "Summarize my emails with X this month" |
| 50-200 items | 3-5 | ~40-50 items each | "Analyze all emails between me and X" |
| 200-500 items | 5-8 | ~50-70 items each | "Build a report from this folder of notes" |
| 500+ items | 8-15 | ~50-80 items each | "Synthesize themes across my entire vault" |

Plus 1 assembly agent + 1 analysis agent. Never exceed 15 collectors.

## Subagent Task Design

Each collection agent prompt MUST include:

1. **Objective**: "Collect and summarize all items in scope"
2. **Source scope**: Exact commands with date ranges or folder paths
3. **Collection method**: "Dump raw content to `/tmp/research-<id>/agent-N/raw/` FIRST, then summarize"
4. **Output location**: `/tmp/research-<id>/agent-N/findings.md` and `links.md`
5. **Output structure**: Chronological entries with date, source, summary, links, quotes
6. **Web policy**: "Supplementary only — verify, enrich, check links"
7. **Completeness mandate**: "Process EVERY item. If you hit limits, write what you have and report the gap in a ## Gaps section"
8. **Budget**: Tool calls for fetching (unlimited for file I/O, ~50-80 for API calls)

See [references/subagent.md](references/subagent.md) for the full template.

## Quality Gates

Before declaring research complete, the lead MUST verify:

1. **Coverage**: Count entries in assembled archive vs. estimated corpus size. If <80%, there are gaps.
2. **Content depth**: Spot-read 5 entries. Do they have actual content summaries or just subject lines? Entries saying "Content: likely sharing interesting content" are failures.
3. **Data + analysis**: Both the archive AND the analysis must exist. One without the other is incomplete.
4. **Link hygiene**: No `http://gmail.com` or email-domain artifacts in link lists.
5. **Source type coverage**: If source has emails AND chats, both should be present.
6. **File size sanity**: 400 items → archive should be 2000+ lines. 20 items → should not be 5000 lines.
7. **Boundary verification**: Earliest and latest entries should match expected date range.

If any gate fails → spawn a fix-up agent. Don't ship incomplete work.

## Key Principles

### Data-Source-First
Web is supplementary. The primary value is in the user's own data. Always exhaust the specified sources before reaching for the web.

### Dump First, Summarize Second
Collection agents dump raw content to files, THEN read files and write summaries. This preserves data even when agents hit context limits. Never rely on an agent to fetch-and-summarize in a single pass over large corpuses.

### Separate Collection from Analysis
The agent that collects data should NOT be the agent that interprets it. Collection agents optimize for completeness. Analysis agents optimize for insight. Combining them degrades both.

### Filesystem as Communication Bus
Subagents write to files, lead reads files. Avoids the "telephone game" of passing findings through conversation context. Files also serve as checkpoints if a run fails partway.

### Honest Analysis
Not everything is "prescient" or "extraordinary." Report what's actually there. Include the mundane alongside the significant. Let the reader draw conclusions.

### Preserve, Don't Compress
Assembly agents clean and deduplicate — they do NOT summarize. If you started with 3,000 lines of findings, the assembled archive should not be 600 lines. The raw data is the archive. Analysis is separate.

## Reference Files

- **Data source access patterns**: [references/data-sources.md](references/data-sources.md)
- **Lead agent behavior**: [references/lead-agent.md](references/lead-agent.md)
- **Subagent prompt template**: [references/subagent.md](references/subagent.md)
