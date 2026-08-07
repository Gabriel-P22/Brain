# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Personal study vault, not a software project. User was approved in a Go/Python + AI hiring process, has zero Go experience (Python is intermediate), and has 2 weeks to get productive. This directory is where the day-by-day learning happens: notes, exercises, and small practice projects.

Two complementary files, both in `contexts/`:
- [contexts/GO_STUDY_PLAN.md](contexts/GO_STUDY_PLAN.md) — the schedule ("when"): 14-day curriculum, fundamentals week 1, applied/job-relevant week 2.
- [contexts/GO-MODULES.md](contexts/GO-MODULES.md) — the content ("what"): topics grouped in modules, each with its own status checkbox and accumulating "Contexto/notas" (persists regardless of which day it was actually studied on). This is the vault's pattern for subject content — future subjects (Python, arquitetura, IA) get their own `*-MODULES.md` file the same way, also under `contexts/`.

Check both before assuming where the user is: GO_STUDY_PLAN.md for pacing/deadline, GO-MODULES.md for what's actually been covered and what notes exist on it.

## Folder layout

- `go/` — Go exercises and mini-projects for the study plan.
- `python/` — Python-side material (used as the reference language to explain Go concepts by comparison, and for any Python parts of the study).
- `infra/` — infra/concurrency-focused material (worker pools, rate limiting, deployment-adjacent topics), matching the "infra" angle of the target role.
- `ia/` — AI-focused material: LLMs, MCP (Model Context Protocol), generative AI, and related topics — including how these get integrated from Go/Python (ties into the plan's Day 9/Day 13 LLM integration work).
- `contexts/` — living reference docs the study workflow reads/writes: GO_STUDY_PLAN.md (schedule) and GO-MODULES.md (per-topic content, one `*-MODULES.md` per subject), plus saved conversation contexts (session recaps, decisions made, state to resume from).
  - `contexts/plans/` — saved plans (implementation/study plans produced during sessions).

Folders are currently empty scaffolding — populate them as each day's exercises are written, one subfolder or file per topic/day as it comes up naturally (don't pre-create structure ahead of need).

## Working conventions

- Teach by analogy to Python where it shortens the explanation (user's existing mental model).
- Prioritize runnable code over pure theory — each topic should land with something the user can execute (`go run`, `go test`).
- Standard Go tooling applies once code exists: `go run <file>.go`, `go test ./...`, `go build`, `gofmt`/`golangci-lint` for style. No custom build system.
- Keep code in this repo idiomatic Go (per plan's Day 14 focus), not Python-flavored Go.

## .claude/ tooling

- `permissions.allow` in `.claude/settings.json` — `go run/test/build`, `gofmt`, `golangci-lint`, `python -m venv`, `pip install`, `pytest` run without a permission prompt.
- Slash commands (`.claude/commands/`): `/topic <nome>` loads a topic from `contexts/GO-MODULES.md` (or another `contexts/*-MODULES.md`) and runs the lesson/exercise, syncing status back to both files; `/review [arquivo]` reviews the latest (or given) Go exercise; `/quiz [topico]` quizzes on topics already covered.
- `.claude/agents/go-tutor.md` — dedicated subagent for Go teaching, same conventions as above, usable standalone for focused Go Q&A without pulling in unrelated session context.
