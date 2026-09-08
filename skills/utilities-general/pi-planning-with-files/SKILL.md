# pi-planning-with-files

> Persistent file-based planning for multi-step AI-agent work. Keeps task_plan.md, findings.md, and progress.md on disk; lifecycle hooks inject selected project planning context. Automatic recovery reads project planning files only. Explicit session-catchup.py --metadata reads same-project local agent session records and emits aggregate counts only; --replay may emit bounded nonce-framed excerpts. Optional gated mode can request continuation only when the host supports it and never runs commands declared in Markdown. The skill has no network upload path. Use for research or work needing 5+ tool calls.

## Overview
Part of the Gopyr Skills Hub under `General & Utilities`. Designed for autonomous execution and prompt engineering workflows.
