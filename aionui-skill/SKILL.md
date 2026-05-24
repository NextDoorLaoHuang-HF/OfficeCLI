---
name: officecli-track-changes
description: "Track Changes / 修订模式 for .docx files. Use when the user wants to review, audit, redline, or revise a document with tracked changes (修订模式, 审查, redline, revision). MUST load alongside officecli skill."
---

# Track Changes / Revision Mode (Word .docx)

When the user asks to review, audit, redline, or revise a document with tracked changes, you MUST use `--prop trackChange=...`. Do NOT use python-docx, unpack/pack, raw-set, or any other approach for creating revision marks — the CLI handles OOXML revision markup natively.

## CRITICAL: In-place revision workflow

**Do NOT use bare `add run --prop trackChange=del/ins` on an existing paragraph** — it appends to the end, producing: `original text ~~deleted~~ <u>inserted</u>`. This duplicates text instead of marking the original.

### Step-by-step: Replace text with tracked change

For each change (e.g. replacing "8" with "4"):

```bash
# Step 0: Read the paragraph run structure FIRST
officecli get "$FILE" '/body/p[N]' --depth 2
# This shows you each run and its text. The target text may be split
# across multiple runs (e.g. "8" in r[3], "小时" in r[4]).

# Step 1: Use `set` with `find` to isolate the target text into its own run.
# find only matches within a SINGLE run, so use the minimal unique text.
# If the target is a number like "8" that's already in its own run, find="8" works.
# If text spans runs, find the portion that IS in one run.
officecli set "$FILE" '/body/p[N]' --prop find="8" --prop color=auto
#   Result: r[1]="...前" r[2]="8" r[3]="...后"

# Step 2: Remove the original (unmarked) run.
officecli remove "$FILE" '/body/p[N]/r[2]'

# Step 3: Add ins run AFTER the preceding run (the run just before where the text was).
officecli add "$FILE" '/body/p[N]' --type run --after '/body/p[N]/r[1]' \
  --prop trackChange=ins --prop trackChange.author=AI --prop text="4"

# Step 4: Add del run AFTER the same preceding run (r[1]).
officecli add "$FILE" '/body/p[N]' --type run --after '/body/p[N]/r[1]' \
  --prop trackChange=del --prop trackChange.author=AI --prop text="8"
# Final: r[1]="...前" r[2]=<ins>"4"</ins> r[3]=<del>"8"</del> r[4]="...后"
```

### Handling cross-run text

Real-world documents often split text across runs (e.g. `"【"` + `"8"` + `"】小时"`). Strategy:

1. **Read run structure first** with `officecli get '/body/p[N]' --depth 2`
2. **Find the smallest unique portion** that exists in a single run
3. For numbers in brackets like `【8】`, the number `8` is typically its own run — just `find="8"`
4. If `find` fails, try a shorter/adjacent portion that IS within one run
5. NEVER fall back to unpack/pack or python-docx — always find a way with officecli

### When find doesn't match

If `find` reports 0 matches:
- The text is split across runs. Use `get --depth 2` to see actual run boundaries
- Try matching just the number or keyword portion (e.g. `find="8"` instead of `find="8小时"`)
- If multiple runs contain the same text, use `find` with enough context to be unique within a single run

## trackChange properties

| Property | Values | Scope |
|----------|--------|-------|
| `trackChange` | `ins`, `del`, `format`, `moveFrom`, `moveTo` | add run/paragraph |
| `trackChange.author` | Any string | add run/paragraph |
| `trackChange.date` | ISO 8601 (default: now) | add run/paragraph |
| `trackChange.id` | Integer (default: auto) | add run/paragraph |

## Other revision operations

```bash
# Format change revision
officecli add "$FILE" /body --type paragraph --prop trackChange=format --prop trackChange.author=AI --prop text="reformatted paragraph"

# Move revision
officecli add "$FILE" '/body/p[1]' --type run --prop trackChange=moveFrom --prop trackChange.author=AI --prop text="moved text"
officecli add "$FILE" '/body/p[2]' --type run --prop trackChange=moveTo --prop trackChange.author=AI --prop text="moved text"

# Enable Track Changes mode (new edits auto-tracked in Word)
officecli set "$FILE" / --prop trackRevisions=true

# Accept or reject all revisions
officecli set "$FILE" / --prop acceptallchanges=all
officecli set "$FILE" / --prop rejectallchanges=all
```

## Verify

```bash
officecli query "$FILE" revision          # list all revisions
officecli view "$FILE" html               # visual: green underline=ins, red strikethrough=del
```

## Contract review pattern (common task)

```bash
FILE="contract_reviewed.docx"
cp original.docx "$FILE"
officecli open "$FILE"
officecli view "$FILE" outline
officecli view "$FILE" text

# For each change:
# 0. get --depth 2 to see run structure
# 1. set find to isolate target text (use minimal unique portion within a single run)
# 2. remove original run
# 3. add ins after preceding run
# 4. add del after preceding run

officecli query "$FILE" revision
officecli view "$FILE" html
officecli close "$FILE"
```

**CRITICAL**: When user mentions "修订模式", "审查", "Track Changes", "redline", "revision", you MUST use the in-place revision workflow above. Never use bare `add run --prop trackChange=del/ins` without first splitting and removing the original run. Never fall back to python-docx, docx.js, unpack/pack, or raw XML for revision marks. If `find` fails, read the run structure and adjust — do NOT switch tools.
