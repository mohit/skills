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
│ (max-capacity)│  NEVER does primary collection
└──────┬───────┘
       │ sessions_spawn (parallel)
       ▼
┌────────────┐ ┌────────────┐ ┌────────────┐
│ Collector 1 │ │ Collector 2 │ │ Collector N │  ← Wave 1: Collection
│ (lead picks)│ │ (lead picks)│ │ (lead picks)│
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
              │  (lead picks)    │     Output: archive files
              └────────┬────────┘
                       │
              ┌─────────────────┐
              │  Analysis Agent  │  ← Wave 3: Interpret, analyze, synthesize
              │  (lead picks)    │     Output: analysis files
              └────────┬────────┘
                       │
                 Lead verifies → delivers to user
```

**Critical**: Collection, assembly, and analysis are SEPARATE agents. Combining them produces either data-without-analysis or analysis-without-data.

## Model Strategy

- Lead agent should run with the highest-capability model and maximum reasoning budget available.
- Lead agent decides model choice for each subagent based on task complexity, expected volume, and cost/speed tradeoffs.
- Do not hardcode specific model names. Use capability tiers (max, high, mid, fast) and let the lead map those tiers to currently available models.

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

**Scope sizing**: A mid-capability collection agent can reliably process ~30-50 items with full content extraction, or ~50-80 items with metadata-only. For larger scopes, split into smaller agents. A 6-month email window is better than a 2-year window. If a scope might have 100+ items, split it further — agents WILL claim to have processed everything but actually handle 15-20 "representative" items. This is the single most common failure mode. If using weaker/faster models, reduce scope further.

### Phase 3: Verify & Fill Gaps

After all collectors complete, the lead agent MUST run these concrete checks:

```bash
# 1. Check output sizes — anything under 50 lines is suspicious
wc -l /tmp/research-<id>/agent-*/findings.md

# 2. Count actual entries vs reported items
grep -c "^### " /tmp/research-<id>/agent-*/findings.md

# 3. Check raw dump counts vs summary counts
find /tmp/research-<id>/agent-*/raw -type f | wc -l  # files dumped
grep -c "^### " /tmp/research-<id>/agent-*/findings.md  # entries written

# 4. Check date boundaries
head -20 /tmp/research-<id>/agent-*/findings.md  # earliest entries
tail -20 /tmp/research-<id>/agent-*/findings.md  # latest entries
```

Verification checklist:
1. **Raw count vs summary count**: If raw/ has 80 files but findings.md has 25 entries, the agent skipped 55 items. Re-run with tighter scope.
2. **Content depth**: Spot-read 3-5 entries — do they have actual content summaries or just subject lines? "Content: likely sharing interesting article" is a failure.
3. **Boundary coverage**: Are the earliest/latest dates what we expected? If you asked for 2005-2009 but earliest entry is 2008, there's a gap.
4. **Gap section**: Did the agent report gaps? If not but counts don't add up, the agent silently skipped items.
5. Spawn gap-filling agents for any under-processed scopes. Don't move to assembly with known gaps.

### Phase 4: Assemble (Wave 2)

Spawn an **assembly agent** that reads ALL findings files and produces clean output:
- Deduplicate (same item from overlapping agent scopes)
- Merge chronologically
- Clean formatting artifacts
- Filter false-positive links (email header domains, etc.)

**Assembly mandate**: The assembly agent's job is to CLEAN and DEDUPLICATE, not to SUMMARIZE. Output should be LONGER than or equal to total input minus duplicates. If the combined findings are 3,000 lines, the assembled archive should not be 600 lines.

After assembly completes, verify:
```bash
# Combined input size
cat /tmp/research-<id>/agent-*/findings.md | wc -l

# Assembly output size
wc -l /tmp/research-<id>/assembled-archive.md

# Entry counts
grep -c "^### " /tmp/research-<id>/agent-*/findings.md  # input entries
grep -c "^### " /tmp/research-<id>/assembled-archive.md  # output entries
```
If output entries < 80% of input entries (after expected dedup), the assembly agent over-compressed. Re-run with an explicit instruction: "You dropped entries. Your previous output had {N} entries but input had {M}. Preserve every entry this time."

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
| < 30 items | 1 | All items | "Summarize my emails with X this month" |
| 30-100 items | 2-4 | ~30-40 items each | "Analyze all emails between me and X this year" |
| 100-300 items | 4-8 | ~30-50 items each | "Build a report from this folder of notes" |
| 300+ items | 8-15 | ~30-50 items each | "Synthesize themes across my entire vault" |

**Err on the side of more, smaller agents.** An agent with 30 items will process all 30. An agent with 100 items will process 20 and claim it did 100. The overhead of extra agents is cheap; missing data costs full re-runs.

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

Run these concrete checks — not vibes, not spot-reads alone:

```bash
# 1. Coverage: entry count vs expected corpus size
EXPECTED=300  # set from your initial count query
ACTUAL=$(grep -c "^### " /path/to/assembled-archive.md)
echo "Coverage: $ACTUAL / $EXPECTED entries ($(( ACTUAL * 100 / EXPECTED ))%)"
# FAIL if <80%

# 2. Content depth: entries with actual summaries vs stubs
grep -A2 "^### " /path/to/assembled-archive.md | grep "Content:" | head -10
# FAIL if entries say "likely sharing" or "content not available"

# 3. Link hygiene
grep -E "http://(gmail|yahoo|google|outlook)\\.com" /path/to/links.md
# FAIL if any matches

# 4. Date boundaries
head -5 /path/to/assembled-archive.md  # earliest
tail -20 /path/to/assembled-archive.md  # latest
# FAIL if outside expected range
```

Full checklist:
1. **Coverage**: Entry count >= 80% of estimated corpus size. If not, there are gaps.
2. **Content depth**: Entries have actual content summaries, not subject-line guesses. "Content: likely sharing interesting content" is a failure.
3. **Data + analysis**: Both the archive AND the analysis must exist. One without the other is incomplete.
4. **Link hygiene**: No `http://gmail.com` or email-domain artifacts in link lists.
5. **Source type coverage**: If source has emails AND chats, both should be present.
6. **File size sanity**: Each cataloged item → ~5-10 lines. 300 items → archive should be 1,500-3,000 lines.
7. **Boundary verification**: Earliest and latest entries match expected date range.
8. **Assembly compression check**: Assembled output line count >= 80% of combined findings input (minus dedup).

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

## Attribution

This skill's orchestration pattern is inspired by Anthropic's engineering post:
https://www.anthropic.com/engineering/multi-agent-research-system

This implementation is adapted for local skill workflows and corpus-first research.
