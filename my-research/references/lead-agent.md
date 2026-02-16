# Lead Agent Behavior Guide

The lead agent IS the main session agent following the skill. This describes how to operate.

## Role

Plan research, delegate to subagents, verify quality, synthesize final output. Do NOT do primary data collection — subagents do that. Focus on strategy, quality control, and the final synthesis step.

## Planning Process

### 1. Assess the Query

- What data sources are specified or implied?
- What questions need answering?
- What output does the user expect? (archive, analysis, report, comparison)
- What's the estimated corpus size? Run a quick count first.
- Are there dependencies? (e.g., need a name list before searching emails)

### 2. Determine Query Type

**Breadth-first** (most common — divide corpus into chunks):
- "Go through all emails from 2005-2025" → split by time period
- "Analyze notes across these 5 folders" → split by folder
- Each collector gets an independent slice

**Depth-first** (multiple lenses on same data):
- "What themes emerge?" → each agent analyzes from a different angle
- Works when corpus fits in one agent's context

**Hybrid** (large corpus + complex analysis):
- Wave 1: breadth-first collection (split by time/source)
- Wave 2: assembly (merge + deduplicate)
- Wave 3: depth-first analysis (separate agent for interpretation)
- This is the default for corpuses over ~100 items

### 3. Size Scopes for Sonnet's Limits

Each Sonnet subagent can reliably handle ~50-80 items. Plan accordingly:
- 200 emails → 4 agents of ~50 each (NOT 2 agents of 100)
- 20-year email archive → split by 2-3 year windows, not decades
- Large folder → split alphabetically or by subfolder

Agents given >100 items will claim to process them all but actually process 15-20 "representative" ones. This is the #1 failure mode. Prevent it with smaller scopes.

### 4. Design the Three Waves

**Wave 1 — Collectors** (parallel):
Each gets a narrow scope, dumps raw data to files, then writes structured findings.
See [subagent.md](subagent.md) for the prompt template.

**Wave 2 — Assembly** (single agent, after collectors finish):
Reads all findings files, produces clean deduplicated archive.
Key instruction: "Your job is to CLEAN and MERGE, not to SUMMARIZE. Preserve all entries."

**Wave 3 — Analysis** (single agent, after assembly):
Reads the assembled archive, writes interpretation.
Key instruction: "Your ONLY job is analysis. The data is already collected. Read it and write insight."

### 5. Verify Between Waves

After Wave 1, before spawning Wave 2:
- `wc -l /tmp/research-<id>/agent-*/findings.md` — are they substantive?
- Spot-read 3-5 entries from each agent
- Check: did agents report gaps in their ## Gaps section?
- If an agent processed <50% of its scope, re-run with tighter scope

After Wave 2, before spawning Wave 3:
- Is the assembled archive >= 80% the size of combined findings (minus expected dedup)?
- Does it cover the full date range?

After Wave 3:
- Does Analysis.md cite specific entries?
- Is it the length/depth the user requested?

## Synthesis (when YOU write the final output)

For smaller tasks where you assemble the final report yourself:

1. **Read ALL output files** before writing anything
2. **Deduplicate** — same item may appear from overlapping scopes
3. **Resolve conflicts** — if agents disagree, note the discrepancy
4. **Layer the report**:
   - Raw archive (chronological, every item — NEVER compress this)
   - Thematic analysis (patterns, evolution)
   - Meta-analysis (significance, lookbacks)
5. **Separate files for large outputs** — don't cram 5,000 lines into one file
6. **Cite sources** — message IDs, dates, file paths for verifiability

## Production Failure Modes (from real runs)

These are not hypothetical. Each happened in production and cost full re-runs:

1. **Analysis without data**: Agent wrote 294-line essay about 423 emails and 795 links but listed ZERO individual entries. Sounded authoritative but was unverifiable. ALWAYS require both granular entries AND synthesis.

2. **Data without analysis**: Agent produced 3,200 lines of entries but Analysis.md was a 55-line stub: "To Be Developed." Both layers required.

3. **Incomplete pagination**: Agent stopped at 2009 despite emails going back to 2005. Never followed pagination tokens past first few pages. ALWAYS verify boundary dates.

4. **False link extraction**: Naive URL parsing pulled email header domains (http://gmail.com, http://yahoo.com) as "shared links." 590 false entries polluted the link archive.

5. **Excluded content types**: Gmail CHAT threads were skipped as "not emails." They contained the richest informal conversations.

6. **Unverified completion**: Lead accepted "Task complete — 423 emails analyzed" at face value. Actual output had 15 entries. ALWAYS read the files, not just the completion message.

7. **Assembly agent over-compressed**: Combined 2,798 lines of findings into 668-line archive. Assembly should preserve, not summarize.

8. **Collection agent "sampling"**: Agent found 100+ items but processed "15+ key representative messages." This defeats the purpose. Every item gets cataloged.

## Anti-Patterns

- **Inflating significance**: Not everything is "extraordinary prescience." Be accurate.
- **Single-pass collection+analysis**: Always separate these into different agents.
- **Trusting completion messages**: Read the actual output files.
- **Large scopes for Sonnet**: Keep each agent's scope to ~50-80 items max.
- **Single-file megadocs**: Split outputs by year, theme, or type.
- **Over-spawning**: 3-8 collectors handles most tasks. Add assembly + analysis = ~5-10 total agents.
