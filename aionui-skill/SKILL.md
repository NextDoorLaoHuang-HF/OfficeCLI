---
name: officecli-track-changes
description: "Track Changes / 修订模式 for .docx files. Use when the user wants to review, audit, redline, or revise a document with tracked changes (修订模式, 审查, redline, revision). MUST load alongside officecli skill."
---

# Track Changes / Revision Mode (Word .docx)

When the user asks to review, audit, redline, or revise a document with tracked changes, you MUST use `--prop trackChange=...`. Do NOT use python-docx, unpack/pack, raw-set, or any other approach for creating revision marks — the CLI handles OOXML revision markup natively.

## CRITICAL: In-place revision workflow

**Do NOT use bare `add run` to create revisions** — it appends to the end of the paragraph, producing: `original text ~~deleted text~~ <u>inserted text</u>`. This is wrong.

### Correct workflow: find → remove → add del+ins at anchor

For each change (e.g. replacing "3个工作日" with "5个工作日"):

```bash
FILE="your-file.docx"

# Step 1: Use `set` with `find` to split the target text into its own run.
#    (The format prop is just to trigger the split; `color=auto` is a no-op.)
officecli set "$FILE" '/body/p[N]' --prop find="3个工作日" --prop color=auto
#    Result: r[1]="...前" r[2]="3个工作日" r[3]="后..."

# Step 2: Remove the original (unmarked) run.
officecli remove "$FILE" '/body/p[N]/r[2]'
#    Result: r[1]="...前" r[2]="后..."

# Step 3: Add ins run AFTER the preceding run (r[1]).
officecli add "$FILE" '/body/p[N]' --type run --after '/body/p[N]/r[1]' \
  --prop trackChange=ins --prop trackChange.author=AI --prop text="5个工作日"
#    Result: r[1]="...前" r[2]=<ins>"5个工作日"</ins> r[3]="后..."

# Step 4: Add del run AFTER the same preceding run (r[1]).
#    Since both use --after r[1], the second one goes after the first → correct order.
officecli add "$FILE" '/body/p[N]' --type run --after '/body/p[N]/r[1]' \
  --prop trackChange=del --prop trackChange.author=AI --prop text="3个工作日"
#    Result: r[1]="...前" r[2]=<ins>"5个工作日"</ins> r[3]=<del>"3个工作日"</del> r[4]="后..."
#    Note: ins appears before del in markup, but Word renders both correctly.
```

### Why this works

1. `set --prop find=X` auto-splits runs so the target text becomes a standalone run
2. `remove` deletes the original unmarked text
3. Two `add run --after` with the same anchor both insert right after r[1]
4. The second add pushes the first one forward, resulting in: ins, del, then the rest

### Verify

```bash
officecli query "$FILE" revision          # list all revisions
officecli view "$FILE" html               # visual: green underline=ins, red strikethrough=del
```

## trackChange properties

| Property | Values | Scope |
|----------|--------|-------|
| `trackChange` | `ins`, `del`, `format`, `moveFrom`, `moveTo` | add run/paragraph |
| `trackChange.author` | Any string | add run/paragraph |
| `trackChange.date` | ISO 8601 (default: now) | add run/paragraph |
| `trackChange.id` | Integer (default: auto) | add run/paragraph |

## Other revision operations

```bash
# Enable Track Changes mode (new edits auto-tracked in Word)
officecli set "$FILE" / --prop trackRevisions=true

# Accept or reject all revisions
officecli set "$FILE" / --prop acceptallchanges=all
officecli set "$FILE" / --prop rejectallchanges=all

# Format change revision (append at end is OK for new paragraphs)
officecli add "$FILE" /body --type paragraph --prop trackChange=format --prop trackChange.author=AI --prop text="reformatted paragraph"
```

## Contract review pattern (common task)

```bash
FILE="contract_reviewed.docx"
cp original.docx "$FILE"
officecli open "$FILE"
officecli view "$FILE" outline                 # see structure
officecli view "$FILE" text                    # read full text

# For each change — use the in-place workflow above:
# 1. set find to split the run
# 2. remove the original run
# 3. add ins after the preceding run
# 4. add del after the preceding run

officecli query "$FILE" revision
officecli view "$FILE" html                    # visual check
officecli close "$FILE"
```

**CRITICAL**: When user mentions "修订模式", "审查", "Track Changes", "redline", "revision", you MUST use the in-place revision workflow above. Never fall back to python-docx, docx.js, unpack/pack, or raw XML for revision marks. Never use bare `add run --prop trackChange=del/ins` without first splitting and removing the original run.
