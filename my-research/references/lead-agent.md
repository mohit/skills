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

### 1.5 Pick Model Tiers

- Lead agent should run on the highest-capability model with maximum reasoning budget available.
- For each subagent, pick a capability tier based on the job:
  - Collection: usually mid/high capability, optimize for completeness and throughput.
  - Assembly: usually mid capability unless dedup/cleanup logic is complex.
  - Analysis: high capability when nuanced synthesis is needed.
- Do not hardcode model brand names. Decide per run using whatever models are currently available.

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

### 3. Size Scopes for the Chosen Collector Model

A mid-capability collection subagent can reliably handle ~30-50 items with full content extraction. Plan accordingly:
- 200 emails → 5-7 agents of ~30-40 each (NOT 2 agents of 100)
- 20-year email archive → split by 2-3 year windows, not decades
- Peak years (high volume) → split further (e.g., Jan-Jun / Jul-Dec)
- Large folder → split alphabetically or by subfolder

**The #1 failure mode**: Agents given >50 items will claim to process them all but actually handle 15-20 "representative" ones. They'll report "100+ emails analyzed" while producing 15 entries. The ONLY defense is smaller scopes and post-collection entry count verification.

**Peak periods need extra agents**: If you know a certain time period is heavy (e.g., 2009 was peak correspondence), give it 2x the agents. An under-scoped peak year agent is worse than two agents with overlapping boundaries (dedup is cheap, re-runs are expensive).

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
```bash
# Sizes
wc -l /tmp/research-<id>/agent-*/findings.md

# Entry counts per agent
for f in /tmp/research-<id>/agent-*/findings.md; do
  echo "$f: $(grep -c '^### ' "$f") entries"
done

# Raw dumps vs summaries (if agents used dump-first)
for d in /tmp/research-<id>/agent-*/raw; do
  echo "$d: $(find "$d" -type f 2>/dev/null | wc -l) files"
done

# Boundary check — first and last entry dates
for f in /tmp/research-<id>/agent-*/findings.md; do
  echo "=== $f ==="
  grep "^### " "$f" | head -1
  grep "^### " "$f" | tail -1
done
```
- If raw/ has significantly more files than findings.md has entries → agent skipped items. Re-run that scope.
- If an agent processed <60% of its scope, spawn a new agent for the gap, don't re-run the whole thing.
- Spot-read 3-5 entries for content quality (actual summaries, not stubs).

After Wave 2, before spawning Wave 3:
```bash
COMBINED=$(cat /tmp/research-<id>/agent-*/findings.md | wc -l)
ASSEMBLED=$(wc -l < /tmp/research-<id>/assembled-archive.md)
echo "Input: $COMBINED lines → Output: $ASSEMBLED lines ($(( ASSEMBLED * 100 / COMBINED ))%)"
```
- If output < 70% of input, assembly over-compressed. Re-run with explicit: "You produced {N} lines from {M} lines of input. That's too much compression. Preserve every entry — just clean and deduplicate."
- Check: does the assembled archive cover the full date range?
- Count entries: `grep -c "^### " assembled-archive.md` should be close to sum of input entry counts minus expected overlap.

After Wave 3:
- Does Analysis.md cite specific entries by date/subject? (grep for dates)
- Is it the length/depth the user requested?
- Does it cover all requested analytical angles?

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

7. **Assembly agent over-compressed**: Combined 2,798 lines of findings into 668-line archive. Assembly should preserve, not summarize. The fix: add explicit line count targets in the assembly prompt: "Input is {N} lines. Your output should be at minimum {0.8*N} lines."

8. **Collection agent "sampling"**: Agent found 100+ items but processed "15+ key representative messages." This defeats the purpose. Every item gets cataloged. The fix: verify raw dump file count against findings entry count. If they diverge by >20%, re-run.

9. **Re-run with context works**: When a scope is under-processed, spawning a new agent for just the gap (with knowledge of what's already covered) works better than re-running the entire scope. Tell the gap-filler: "Items already covered: [list]. Process everything else."

10. **Split peak periods proactively**: In the Praneet run, 2009 was a peak year. Original agent 3 produced only 62 lines. Splitting into Jan-Jun (3a) and Jul-Dec (3b) with dump-first produced 2,026 lines + 349 raw files. If you know a period is heavy, split it BEFORE the first run.

## Anti-Patterns

- **Inflating significance**: Not everything is "extraordinary prescience." Be accurate.
- **Single-pass collection+analysis**: Always separate these into different agents.
- **Trusting completion messages**: Read the actual output files.
- **Large scopes for any collector model**: Keep each agent's scope small enough for full coverage, usually ~30-50 items max for mid-tier models.
- **Single-file megadocs**: Split outputs by year, theme, or type.
- **Over-spawning**: 3-8 collectors handles most tasks. Add assembly + analysis = ~5-10 total agents.
