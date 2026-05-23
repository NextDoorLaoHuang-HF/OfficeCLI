---
name: officecli-track-changes
description: "Track Changes / 修订模式 for .docx files. Use when the user wants to review, audit, redline, or revise a document with tracked changes (修订模式, 审查, redline, revision). MUST load alongside officecli skill."
---

# Track Changes / Revision Mode (Word .docx)

When the user asks to review, audit, redline, or revise a document with tracked changes, you MUST use `--prop trackChange=...`. Do NOT use python-docx, unpack/pack, raw-set, or any other approach for creating revision marks — the CLI handles OOXML revision markup natively.

### Creating revisions

```bash
# Insert revision (green underline in Word/preview)
officecli add "$FILE" '/body/p[1]' --type run --prop trackChange=ins --prop trackChange.author=AI --prop text="new clause"

# Delete revision (red strikethrough in Word/preview)
officecli add "$FILE" '/body/p[1]' --type run --prop trackChange=del --prop trackChange.author=AI --prop text="removed text"

# Format change revision
officecli add "$FILE" '/body/p[1]' --type run --prop trackChange=format --prop trackChange.author=AI --prop bold=true --prop text="formatted"

# Move revision
officecli add "$FILE" '/body/p[1]' --type run --prop trackChange=moveFrom --prop trackChange.author=AI --prop text="moved text"
officecli add "$FILE" '/body/p[2]' --type run --prop trackChange=moveTo --prop trackChange.author=AI --prop text="moved text"

# Paragraph-level format revision
officecli add "$FILE" /body --type paragraph --prop trackChange=format --prop trackChange.author=AI --prop text="reformatted paragraph"

# Enable Track Changes mode (new edits auto-tracked in Word)
officecli set "$FILE" / --prop trackRevisions=true
```

### trackChange properties

| Property | Values | Scope |
|----------|--------|-------|
| `trackChange` | `ins`, `del`, `format`, `moveFrom`, `moveTo` | add run/paragraph |
| `trackChange.author` | Any string | add run/paragraph |
| `trackChange.date` | ISO 8601 (default: now) | add run/paragraph |
| `trackChange.id` | Integer (default: auto) | add run/paragraph |

### Revision workflow

```bash
# 1. Read the document
officecli view "$FILE" text

# 2. For each change: delete old text, insert new text
officecli add "$FILE" '/body/p[3]' --type run --prop trackChange=del --prop trackChange.author=AI --prop text="3个工作日"
officecli add "$FILE" '/body/p[3]' --type run --prop trackChange=ins --prop trackChange.author=AI --prop text="5个工作日"

# 3. Verify revisions
officecli query "$FILE" revision

# 4. Accept or reject all
officecli set "$FILE" / --prop acceptallchanges=all
officecli set "$FILE" / --prop rejectallchanges=all
```

### Contract review pattern (common task)

```bash
FILE="contract_reviewed.docx"
cp original.docx "$FILE"
officecli open "$FILE"
officecli view "$FILE" text                    # read full text
officecli view "$FILE" outline                 # see structure

# For each issue found:
# 1. Mark old text as deleted
officecli add "$FILE" '/body/p[5]' --type run --prop trackChange=del --prop trackChange.author=AI --prop text="old clause"
# 2. Mark new text as inserted
officecli add "$FILE" '/body/p[5]' --type run --prop trackChange=ins --prop trackChange.author=AI --prop text="revised clause"

# After all changes, verify
officecli query "$FILE" revision
officecli view "$FILE" html                    # visual check: green=inserted, red=deleted
officecli close "$FILE"
```

**CRITICAL**: When user mentions "修订模式", "审查", "Track Changes", "redline", "revision", you MUST use `--prop trackChange=ins/del/format`. Never fall back to python-docx, docx.js, unpack/pack, or raw XML for revision marks.
