# KILL-002 — Add exact line numbers and source code to business-rules-inventory.md

## Source
Follow-up to `KILL-001` (business rules & unique-IP inventory), requested by the user to make each
entry independently verifiable without re-opening the cited source file.

## Summary
For every rule in `planning/research/business-rules-inventory.md`, add three new lines immediately
after the existing `Description:` line: `Line Numbers:`, `Source Code File Name:`, and
`Source Code:`. These must pin down the exact business-logic lines the rule's description already
references in prose (e.g. "AccountModelDao.java:184-189"), and reproduce the literal source code at
those lines.

## Assessment
`planning/research/business-rules-inventory.md` (857 lines, 166 rules across 10 modules) already
embeds file:line citations inline in each `Description:` line's prose (e.g. `Kill-BR-0001` cites
`account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:184-189`). No entry
currently has a structured `Line Numbers:`, `Source Code File Name:`, or `Source Code:` line — this
task adds them using the file paths and ranges already present in each description, verified against
current source (line numbers may have drifted since KILL-001 was written).

**Location:** `planning/research/business-rules-inventory.md` — every `- [Kill-BR-<nnnn>]` entry.

## Plan

1. Read `business-rules-inventory.md` end to end and, for each `- [Kill-BR-<nnnn>]` entry, extract
   the file path(s) and line range(s) already cited in its `Description:` text.
2. For entries citing more than one file/range (duplicated logic across two classes), decide
   whether to list both under one `Line Numbers:`/`Source Code File Name:`/`Source Code:` block or
   split into multiple such blocks — pick whichever keeps the entry unambiguous, and apply that
   choice consistently.
3. Open each cited source file and re-verify the line range still contains the described logic
   (files may have drifted since KILL-001 was written); adjust the range if it has shifted.
4. Insert three new lines directly after each entry's `Description:` line (and before its
   `Applies to:` line):
   - `Line Numbers: <start> to <end>`
   - `Source Code File Name: <path>`
   - `Source Code: <the literal source lines, verbatim>`
5. Update the file in place, preserving every other line (Methodology, Summary, rule text) unchanged.

## Acceptance Criteria
All criteria verified 2026-08-27 before commit.
- [x] Every one of the 166 `- [Kill-BR-<nnnn>]` entries in `business-rules-inventory.md` has a
      `Line Numbers:`, `Source Code File Name:`, and `Source Code:` line inserted immediately after
      its `Description:` line and before its `Applies to:` line.
- [x] Each `Line Numbers:` range matches the file cited in that same entry's `Source Code File Name:`
      line, and the `Source Code:` content is the literal, verbatim code at that range in the current
      repo (re-verified, not copied from the old prose citation without checking).
- [x] No existing content (Methodology, Summary, rule names, `Applies to:`, `Confidence score:`) is
      altered or reordered — only the three new lines are added.
- [x] The file remains valid Markdown and every rule ID (`Kill-BR-0001`...`Kill-BR-0166`) is still
      present and in its original order.
