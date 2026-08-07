---
name: notebooklm-meta-study
version: "0.1.0"
description: "Conduct scalable, secure scientific meta-studies on any topic via Gemini Notebook (NotebookLM) using the nlm CLI. Produces systematic reviews, meta-analyses, evidence syntheses, literature surveys, and multi-source research reports with automated error recovery. Triggered by: \"meta-study\", \"systematic review\", \"meta-analysis\", \"evidence synthesis\", \"literature survey\", \"multi-source research report\", \"wissenschaftliche Metastudie\", \"Literaturrecherche\", \"Evidenzsynthese\". Do NOT use for: one-off queries, non-research content generation, chat/REPL sessions, or cinematic video production."
---

# NotebookLM Meta-Study Skill

Conduct scalable, secure scientific meta-studies on any research topic using Gemini Notebook (NotebookLM) via the `nlm` CLI (notebooklm-mcp-cli v0.9.6+). Orchestrates the complete pipeline: authentication → notebook provisioning → source ingestion (100+ documents) → multi-artifact generation → quality assurance → download.

## When to Use

- The user requests a systematic review, meta-analysis, evidence synthesis, literature survey, or multi-source research report.
- The user provides 3+ source documents (URLs, PDFs, Drive docs, text) and wants a synthesized research output.
- The user wants structured research artifacts (report + data table + infographic + audio + quiz + flashcards + slides) generated from a curated source corpus.
- The user says: "meta-study on [topic]", "systematic review of [topic]", "meta-analyze [topic]", "synthesize evidence on [topic]", "literature survey for [domain]", "multi-source report on [topic]".

## When NOT to Use

- Single-source Q&A or one-off queries — use `nlm notebook query` directly.
- Non-research content generation (marketing copy, creative writing, casual summaries).
- Cinematic video production — quota-limited (~2/day), not suited for meta-studies.
- Interactive REPL sessions — `nlm chat start` is a human-only REPL; this skill never invokes it.
- Deleting notebooks, sources, or artifacts — deletion always requires explicit user confirmation outside this skill.

## Prerequisites

1. **`nlm` CLI ≥ 0.9.6**: Installed and authenticated (`nlm login --check` passes).
2. **Authenticated Google account**: With Gemini Notebook access.
3. **Source materials available**: URLs, local file paths (accessible on the machine running `nlm`), Google Drive document IDs, or text content.
4. **Disk space**: Sufficient for downloaded artifacts (reports ~100KB, audio ~20-50MB, infographics ~1-5MB each).

## Workflow

This skill executes a strict 6-phase pipeline. No phase may be skipped.

### Phase 0: Pre-Flight

**Goal:** Verify authentication, identify research type, validate source availability.

```
Step 0.1 — Authenticate
  nlm login --check
  If auth_status=stale → nlm login (1x retry, then escalate to user)
  If auth_status=unverified → nlm notebook list --quiet (test API call)

Step 0.2 — Identify research type from user input
  Map user phrasing to one of five types (see Research Types table below).
  If ambiguous → ask user: "Which research type: systematic review, meta-analysis,
  evidence synthesis, literature survey, or multi-source report?"

Step 0.3 — Validate source availability
  Count URLs, file paths, Drive IDs, text blocks.
  If 0 sources → ask user to provide at least one source.
  If >100 sources → warn about potential notebook limits; plan split-notebook strategy.
```

**Research Types:**

| ID | Type | Prompt Profile | Primary Artifacts |
| ---- | ------ | --------------- | ------------------- |
| SR | Systematic Review | PRISMA-informed, structured search strategy, inclusion/exclusion criteria | Report (Create Your Own), Data Table, Infographic |
| MA | Meta-Analysis | Effect-size extraction, heterogeneity assessment, forest-plot description | Report (Create Your Own), Data Table, Quiz |
| ES | Evidence Synthesis | GRADE-informed evidence rating, PICO framework | Report (Create Your Own), Infographic (scientific), Flashcards |
| LS | Literature Survey | Thematic clusters, chronological organization | Report (Study Guide), Mind Map, Slides |
| MR | Multi-Source Report | Executive summary, cross-source patterns, policy implications | Report (Briefing Doc), Audio (deep_dive), Slides |

### Phase 1: Notebook Provisioning

**Goal:** Create a dedicated research notebook with alias and tags.

```
Step 1.1 — Create notebook
  nlm notebook create "Meta-Study: <topic> — $(date +%F)" --json
  Capture NOTEBOOK_ID from output.

Step 1.2 — Set alias
  nlm alias set meta-<type> <NOTEBOOK_ID>

Step 1.3 — Tag for organization
  nlm tag add <NOTEBOOK_ID> --tags "meta-study,<research-type>"
```

### Phase 2: Source Ingestion

**Goal:** Ingest all sources with rate-limiting compliance.

> **Why:** The `nlm` API enforces rate limits. Source operations need 2-second inter-command delays. Batch operations without delays will trigger rate-limit errors.

```
Step 2.1 — Add URL sources (bulk)
  For each URL:
    nlm source add <NOTEBOOK_ID> --url "<URL>"
    sleep 2  # MANDATORY: rate-limit throttle

Step 2.2 — Add file sources (local PDFs, DOCX, etc.)
  For each file:
    nlm source add <NOTEBOOK_ID> --file "<absolute-path>" --wait --wait-timeout 120
    sleep 2
  If file upload fails with path error:
    Fallback: extract text content and use --text instead.
    User message: "File path must be accessible on the machine running nlm.
    Consider using --text for inline content."

Step 2.3 — Add text sources (abstracts, notes)
  For each text block:
    nlm source add <NOTEBOOK_ID> --text "<content>" --title "<descriptive title>"
    sleep 2

Step 2.4 — Add Google Drive sources
  For each Drive ID:
    nlm source add <NOTEBOOK_ID> --drive <doc-id> --type <doc|slides|sheets|pdf>
    sleep 2

Step 2.5 — Complementary deep research (optional but recommended)
  nlm research start "<research query>" --notebook-id <NOTEBOOK_ID> --mode deep --auto-import
  This discovers additional relevant sources (~40+ in deep mode, ~5 min).
  Monitor with: nlm research status <NOTEBOOK_ID> --max-wait 900
```

**Source Error Recovery:**

| Failure | Recovery |
| --------- | ---------- |
| URL unreachable | Retry 3x with 5s delay, then skip with warning |
| File path not found | Suggest --text fallback, do not retry |
| Rate limit | Wait 120s, retry once |
| Drive permission error | Skip, report to user |

### Phase 3: Source Quality Assurance

**Goal:** Inventory all ingested sources and verify readiness for generation.

```
Step 3.1 — Inventory sources
  nlm source list <NOTEBOOK_ID> --json
  Count total sources. If < 3 → warn user about limited corpus.

Step 3.2 — AI-generated source summaries (sample key sources)
  nlm source describe <source-id>  # For 3-5 most important sources

Step 3.3 — Notebook overview
  nlm notebook describe <NOTEBOOK_ID>
  Review suggested topics to validate source coverage.
```

### Phase 4: Multi-Artefact Generation

**Goal:** Generate all research artifacts sequentially with rate-limiting.

> **Why sequential:** Parallel Studio generation triggers rate limits. Enforce 5-second inter-generation delays. Each `--confirm` gate requires user confirmation before proceeding.

**CRITICAL:** Every generation command requires `--confirm` or `-y`. The agent MUST present each generation request to the user for approval before execution.

```
Step 4.1 — Primary Research Report (MANDATORY)
  nlm report create <NOTEBOOK_ID> --format "Create Your Own" \
    --prompt "<type-specific report prompt from references/prompts.md>" --confirm
  sleep 5

Step 4.2 — Data Table (SR, MA, ES types)
  nlm data-table create <NOTEBOOK_ID> "<type-specific schema from references/prompts.md>" --confirm
  sleep 5

Step 4.3 — Scientific Infographic
  nlm infographic create <NOTEBOOK_ID> --orientation landscape \
    --detail standard --style scientific --focus "<topic summary>" --confirm
  sleep 5

Step 4.4 — Audio Podcast (Deep Dive, ~20 min)
  nlm audio create <NOTEBOOK_ID> --format deep_dive --length long \
    --language de --focus "<topic>" --confirm
  sleep 5

Step 4.5 — Quiz (10 questions, medium difficulty)
  nlm quiz create <NOTEBOOK_ID> --count 10 --difficulty 3 \
    --focus "Key concepts from sources" --confirm
  sleep 5

Step 4.6 — Flashcards
  nlm flashcards create <NOTEBOOK_ID> --difficulty medium \
    --focus "Core terminology and concepts" --confirm
  sleep 5

Step 4.7 — Slide Deck
  nlm slides create <NOTEBOOK_ID> --format detailed_deck --length default \
    --focus "Research findings presentation" --confirm
```

**Artifact selection by research type:**

| Artifact | SR | MA | ES | LS | MR |
|----------|:--:|:--:|:--:|:--:|:--:|
| Report (Create Your Own) | ● | ● | ● |   |   |
| Report (Study Guide)     |   |   |   | ● |   |
| Report (Briefing Doc)    |   |   |   |   | ● |
| Data Table               | ● | ● | ● |   |   |
| Infographic              | ● |   | ● |   |   |
| Audio (deep_dive)        |   |   |   |   | ● |
| Quiz                     |   | ● |   |   |   |
| Flashcards               |   |   | ● |   |   |
| Mind Map                 |   |   |   | ● |   |
| Slides                   |   |   |   | ● | ● |

**Generation Error Recovery:**

| Failure | Recovery |
| --------- | ---------- |
| `studio_status=failed` | Read `error_reason`, retry 1x with narrower scope (add --source-ids for key sources) |
| Rate limit during generation | Wait 120s, retry once |
| "No sources" error | Verify source list; if empty, abort and report |

### Phase 5: Status Monitoring & Download

**Goal:** Poll until all artifacts complete, then download.

```
Step 5.1 — Poll completion
  nlm studio status <NOTEBOOK_ID> --json --full
  Poll every 30s until all artifacts show "completed" or "failed".
  Max wait: 15 minutes (audio generation can take 2-5 min).

Step 5.2 — Download all completed artifacts
  nlm download all <NOTEBOOK_ID> --output-dir ./meta-studies/<topic-slug>/
  This creates per-notebook subdirectories with all completed artifacts.

Step 5.3 — Export chat transcript (if queries were made)
  nlm chats export <NOTEBOOK_ID> --format md -o ./meta-studies/<topic-slug>/chat-transcript.md
```

### Phase 6: Finalization

**Goal:** Verify outputs, report results to user.

```
Step 6.1 — Verify downloads
  Check that ./meta-studies/<topic-slug>/ contains expected files.
  Count completed vs failed artifacts from studio status.

Step 6.2 — Report summary to user
  Format:
    "Meta-Study Complete: <topic>
     - Notebook: <NOTEBOOK_ID> (alias: meta-<type>)
     - Research Type: <SR|MA|ES|LS|MR>
     - Sources ingested: <N>
     - Artifacts generated: <X>/<Y> completed
     - Output directory: ./meta-studies/<topic-slug>/
     - Failed artifacts: <list if any, with error reasons>"

Step 6.3 — Quality check
  Spot-check the primary report:
    - Claims are source-grounded (no invented statistics)
    - Structure matches the research type template
    - Contradictions are flagged
  If quality fails → offer to regenerate with adjusted prompt.
```

## Failure Modes

**Level 1 (Local Retry — automatic):**

| Condition | Action | Max Retries |
| ----------- | -------- | ------------- |
| Rate limit ("Rate limit exceeded") | Wait 120s, retry | 3 |
| Source add network error | Wait 5s, retry | 3 |
| Auth unverified probe | Test API call | 2 |
| Studio status timeout | Extend poll to 900s | 1 |

**Level 2 (Local Patch — automatic with fallback):**

| Condition | Action |
| ----------- | -------- |
| File upload path not found | Fallback to `--text` with extracted content |
| Import timeout | Retry with `--timeout 600` |
| Generation failed with `error_reason` | Retry with narrower `--source-ids` scope |
| Research mode deep too slow (>15 min) | Fallback to `--mode fast` |

**Level 3 (Replan/Escalate — user intervention required):**

| Condition | Action |
| ----------- | -------- |
| Auth expired (`auth_status=stale`) after `nlm login` retry | "Gemini Notebook authentication expired. Please run `nlm login` manually." |
| All 3 rate-limit retries exhausted | "NotebookLM rate limit persists. Wait 5 minutes and retry, or reduce artifact count." |
| >100 source notebook limit reached | "Source limit reached. Split into multiple notebooks and use `nlm cross query` for synthesis." |
| RPC_ID_Drift detected | "Gemini Notebook changed internal RPC IDs. Run with --debug and set NOTEBOOKLM_RPC_OVERRIDES." |
| All sources fail to ingest | "No sources could be added. Verify URLs, file paths, and network connectivity." |

## Output Contract

This skill produces:

1. **A Gemini Notebook notebook** containing all ingested sources, tagged `meta-study,<type>`.
2. **Downloaded artifacts** in `./meta-studies/<topic-slug>/`:
   - Primary research report (Markdown)
   - Data table (where applicable)
   - Infographic (PNG/PDF)
   - Audio podcast (MP3)
   - Quiz (HTML/JSON)
   - Flashcards (HTML/JSON)
   - Slide deck (PDF)
   - Chat transcript (Markdown)
3. **A structured summary** to the user with notebook ID, artifact inventory, and output path.

## Verification Gate

Before declaring a meta-study complete, ALL of these must be true:

- [ ] `nlm login --check` passed (or re-authenticated successfully).
- [ ] Notebook created with alias and tags.
- [ ] All Phase 2 source adds completed (with skip-tracking for failures).
- [ ] Phase 4 primary report artifact shows `studio_status=completed`.
- [ ] At least 1 additional artifact completed beyond the report.
- [ ] Downloaded files exist in `./meta-studies/<topic-slug>/`.
- [ ] Report spot-check: at least 80% of verifiable claims map to uploaded sources.

## Side Effects

| Action | Type | Blast Radius | Human Approval? |
| -------- | ------ | ------------- | ----------------- |
| `nlm login --check` | Read-only | Low | No |
| `nlm login` (reauth) | Reversible | Medium | Yes — browser opens |
| `nlm notebook create` | Reversible | Low | No (can delete later) |
| `nlm source add` | Reversible | Low | No (can delete later) |
| `nlm research start` | Reversible | Low | No |
| `nlm report create --confirm` | Compensatable | Medium | Yes — `--confirm` required |
| `nlm audio create --confirm` | Compensatable | Medium | Yes — `--confirm` required |
| `nlm data-table create --confirm` | Compensatable | Low | Yes — `--confirm` required |
| `nlm infographic create --confirm` | Compensatable | Low | Yes — `--confirm` required |
| `nlm quiz create --confirm` | Compensatable | Low | Yes — `--confirm` required |
| `nlm flashcards create --confirm` | Compensatable | Low | Yes — `--confirm` required |
| `nlm slides create --confirm` | Compensatable | Low | Yes — `--confirm` required |
| `nlm download all` | Pure | Low | No (writes to local disk) |
| `nlm source delete --confirm` | Irreversible | High | Yes — explicit user confirmation OUTSIDE this skill |

## Rate Limiting & Throttling

| Operation | Minimum Delay | Max Retries on Rate-Limit |
| ----------- | -------------- | --------------------------- |
| Source add (URL/file/text/Drive) | 2 seconds | 3 (120s backoff) |
| Content generation (any artifact) | 5 seconds | 1 (120s backoff) |
| Research operations | 2 seconds | 2 (60s backoff) |
| Query operations | 2 seconds | 2 (30s backoff) |
| Download operations | None required | 2 (30s backoff) |

**Daily limits (Free Tier):** ~50 queries/operations per day. Warn user when approaching limit: "Approaching NotebookLM free-tier daily limit (~50 ops). Consider upgrading to Pro for extended usage."

## Security Architecture

Based on `nlm-skill/references/remote-mcp.md`:

1. **Loopback-only binding**: The `nlm` MCP server binds exclusively to `127.0.0.1`. Never set `NOTEBOOKLM_ALLOW_EXTERNAL_BIND=1`.
2. **Single-account isolation**: All operations run under one authenticated Google profile. Use `nlm login switch <profile>` to change accounts.
3. **Server-local file paths**: `nlm source add --file` reads from the machine running `nlm`, not the client machine. Paths like `/Users/...` on a remote agent host will fail.
4. **No built-in endpoint auth**: The MCP HTTP endpoint has no authentication, authorization, or TLS. Never expose it to public internet.
5. **GDPR data flow**: `Local machine → nlm CLI → Google NotebookLM API → Local machine (download)`. No research data transits through third-party servers beyond Google's infrastructure.
6. **No auto-delete**: This skill never invokes `nlm * delete` commands. Deletion is irreversible and requires explicit user confirmation outside this workflow.

## References

For detailed prompt templates, CLI command reference, and troubleshooting, load these reference files on demand:

- **[references/prompts.md](references/prompts.md)**: Type-specific prompt templates for SR, MA, ES, LS, MR reports, data tables, and supplementary artifacts.
- **[references/nlm-commands.md](references/nlm-commands.md)**: Abridged `nlm` CLI command reference — the 25 commands relevant to meta-study workflows.

For the full `nlm` documentation, refer to the source files at:

- `nlm-skill-export/nlm-skill/references/command_reference.md`
- `nlm-skill-export/nlm-skill/references/troubleshooting.md`
- `nlm-skill-export/nlm-skill/references/studio-prompting-guide.md`

## Usage Examples

### Example 1: Systematic Review

```
User: "Conduct a systematic review of AI literacy requirements under EU AI Act Article 4."

Agent executes Phase 0-6:
  → Type: SR (Systematic Review)
  → Notebook: "Meta-Study: AI Literacy Art. 4 — 2026-08-07"
  → Sources: 10 PDFs from .trae/documents/ + deep research web sources
  → Artifacts: Report (Create Your Own) + Data Table (PRISMA flow) + Infographic
  → Output: ./meta-studies/ai-literacy-art-4/
```

### Example 2: Evidence Synthesis

```
User: "Evidenzsynthese zu den Auswirkungen des EU AI Act auf deutsche KMU"

Agent executes Phase 0-6:
  → Type: ES (Evidence Synthesis)
  → Notebook: "Meta-Study: EU AI Act KMU Auswirkungen — 2026-08-07"
  → Sources: EU AI Act PDF + KI-MIG Entwurf + KPMG ISO 42001 + deep research
  → Artifacts: Report + Data Table (GRADE profile) + Infographic (scientific) + Flashcards
  → Language: de (all prompts and audio in German)
  → Output: ./meta-studies/eu-ai-act-kmu-auswirkungen/
```

### Example 3: Multi-Source Report

```
User: "Multi-source research report comparing AI governance frameworks (EU AI Act, ISO 42001, US Executive Order)"

Agent executes Phase 0-6:
  → Type: MR (Multi-Source Report)
  → Notebook: "Meta-Study: AI Governance Frameworks Comparison — 2026-08-07"
  → Sources: URLs for each framework + deep research + text notes
  → Artifacts: Report (Briefing Doc) + Audio (deep_dive) + Slides
  → Output: ./meta-studies/ai-governance-comparison/
```

## Troubleshooting

| Symptom | Cause | Solution |
| --------- | ------- | ---------- |
| "Cookies have expired" | Session timeout | `nlm login` — browser opens for Google sign-in |
| "Rate limit exceeded" during generation | Too many sequential requests | The skill automatically waits 120s and retries. If it persists, reduce artifact count. |
| Source add fails for a URL | URL inaccessible or paywalled | Skip with warning. Provide alternative source. |
| File upload "path not found" | Path not on nlm host | Use `--text` fallback with extracted content |
| Deep research takes >15 min | Broad query or server load | Use `--mode fast` or narrow query |
| "Notebook not found" | Wrong ID | Run `nlm notebook list` to find correct ID |
| Artifact status = "failed" | Generation error | Read `error_reason` in `studio_status --full`, retry with adjusted prompt |
| "Import timed out" | Too many sources | Use `--timeout 600` for larger batches |
| Non-English audio has wrong accent | BCP-47 locale mismatch | Use regional code: `es-419` for Latin American, `es-ES` for Spain Spanish. For German always use `de`. |
| `auth_status=unverified` | Network probe failure | This is NOT expired auth. Try an API call first: `nlm notebook list --quiet` |
