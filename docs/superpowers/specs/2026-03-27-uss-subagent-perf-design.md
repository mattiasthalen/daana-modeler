# USS Subagent Performance Fix

**Date:** 2026-03-27
**Issue:** #53 — USS subagent takes too long: 73 tool calls for 17 file generation

## Problem

The Phase 2 USS generation subagent makes ~73 tool calls (re-reading embedded reference files, running validation queries, iterating on corrections) when it should make ~19 calls. Takes 17m32s instead of <2 minutes.

## Root Cause

The subagent prompt in `skills/uss/SKILL.md`:
1. Includes connection details and dialect instructions — enabling unnecessary DB queries
2. Embeds ~24K tokens of USS patterns/examples — yet the subagent re-reads them anyway
3. Has no explicit tool constraints — the subagent uses Bash, Read, and validation queries freely

## Design

### Changes to `skills/uss/SKILL.md`

1. **Remove line 17** — parent agent no longer reads `@references/uss-patterns.md` and `@references/uss-examples.md` (it doesn't use them; the subagent will read them itself)
2. **Remove subagent prompt item #4** (Connection details) — subagent doesn't need host/port/password
3. **Remove subagent prompt item #5** (Dialect instructions) — execution commands irrelevant for Write-only agent
4. **Replace subagent prompt items #6 and #7** (embedded USS patterns/examples) with a single instruction telling the subagent to Read the two reference files from disk
5. **Add tool constraints** to the subagent prompt:
   - Only use the Read tool to read the two reference files
   - Only use the Write tool to create SQL files
   - Do NOT use the Bash tool or run any database queries
6. **Renumber** remaining items

### What stays the same

- Bootstrap data (embedded in prompt — subagent needs TYPE_KEYs, table names, physical columns)
- Interview answers (entity classification, temporal mode, versioning, materialization, output folder)
- Column naming conventions
- File naming rules
- DDL wrapping rules
- Generation order
- Output instructions

## Expected Result

- ~19 tool calls: 2 Read (reference files) + ~17 Write (SQL files)
- Under 2 minutes execution time
- Smaller subagent prompt (~24K fewer tokens)
