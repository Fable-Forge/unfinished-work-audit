---
name: unfinished-work-audit
description: Use when the user asks what work is still unfinished across their local AI sessions — 检查我本地的所有 claude 对话还有哪些工作没干完, 盘点 Codex 对话未完工作, 哪些项目停在半截, 有哪些后续没开发, 上次那个做完了吗 — or wants to resume a dormant project and needs to know where it stopped. Reads local Codex rollouts and Claude Code session transcripts, then verifies each claim against the real repo before listing it.
---

# Unfinished Work Audit

## Overview

Scan local AI session history across tools, then report what is genuinely still open. The hard part is not finding sessions — it is that a session's last message is a *claim* about state, often weeks stale. Verify before listing.

## Evidence Sources

| Source | Path | What it gives |
|---|---|---|
| Codex thread index | `~/.codex/session_index.jsonl` | `thread_name` + `updated_at`, cheapest first pass |
| Codex rollouts | `~/.codex/sessions/<YYYY>/**/rollout-*.jsonl` | full turns when a thread looks open |
| Codex rollout summaries | `~/.codex/memories/rollout_summaries/*.md` | pre-digested outcomes and learnings |
| Claude Code transcripts | `~/.claude/projects/<encoded-path>/*.jsonl` | per-project sessions; dir name encodes the repo path |
| Project memory | `~/.claude/projects/*/memory/MEMORY.md` | durable facts already captured |

Read `references/scan-recipes.md` for the extraction snippets (first user message, last assistant message, per-project grouping, mojibake-safe output).

## Workflow

1. Set the window and the roots.
   - Default to 60 days. If that yields under ~20 sessions, widen to all history and say so.
   - Enumerate both Codex and Claude Code. Reporting only one tool's sessions is a wrong answer to "所有对话", and the same project often has threads in both.

2. Shortlist candidate-open threads.
   - Signals: an ending that promises a next step (`下一步`, `待办`, `未完成`, `TODO`, `pending`, `接下来`), an explicit stop (`我现在停止`, `不再追加功能`), a `pending_external` marker, or an error/blocker as the final state.
   - A thread that ends with a delivered result and no promised follow-up is closed. Do not pad the list.

3. Verify each candidate against the repo — this is the step that makes the report worth anything.
   - Check whether the promised artifact exists now: the file, the export, the test, the commit.
   - A session claiming "还需重新导出 EXE" is not evidence the EXE is still stale; compare the binary's mtime against the last source change.
   - Mark each item `已完成（会话未更新）`, `确认未完成`, or `无法判定` with the reason.
   - Never infer completion from `git status` cleanliness, and never infer incompleteness from the session text alone.

4. Separate what is blocked from what is merely unfinished.
   - `pending_external` (real-device QA, store review, human playtest, another person) is not backlog the user can clear tonight — list it apart.
   - Keep the user's own `已停止开发` decisions out of the backlog; a deliberate stop is not a loose end.

5. Report smallest-useful.
   - Group by project, ordered by how cheap it is to close.
   - Per item: 项目 / 停在哪一步 / 证据（文件路径、时间戳、会话日期）/ 验证结论 / 下一步一句话.
   - State coverage honestly: sessions scanned, tools covered, and anything unreadable.

## Common Mistakes

- Trusting the last assistant message as current state. It is the single biggest source of false backlog here.
- Listing every session that mentions a TODO. Most were closed later, often in a different thread or the other tool.
- Printing Chinese thread names straight to a Windows console — they mojibake. Write to a UTF-8 file and read that back.
- Dumping raw transcript excerpts into the report; follow `reporting-discipline` and give conclusion plus evidence pointer.

## Output Contract

Report: window and roots scanned, session counts per tool, verified-open items grouped by project, `pending_external` listed separately, items reclassified as already-done, and anything that could not be verified.
