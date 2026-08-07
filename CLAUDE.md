# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Personal study vault, not a software project. User was approved in a Go/Python + AI hiring process, has zero Go experience (Python is intermediate), and has 2 weeks to get productive. This directory is where the day-by-day learning happens: notes, exercises, and small practice projects.

Study plan lives in [GO_STUDY_PLAN.md](GO_STUDY_PLAN.md) — a 14-day curriculum (fundamentals week 1, applied/job-relevant week 2: APIs, infra/concurrency, LLM integration). Check it for current progress before assuming where the user is in the plan; update the "Notas de progresso" section as topics get covered.

## Folder layout

- `go/` — Go exercises and mini-projects for the study plan.
- `python/` — Python-side material (used as the reference language to explain Go concepts by comparison, and for any Python parts of the study).
- `infra/` — infra/concurrency-focused material (worker pools, rate limiting, deployment-adjacent topics), matching the "infra" angle of the target role.
- `ia/` — AI-focused material: LLMs, MCP (Model Context Protocol), generative AI, and related topics — including how these get integrated from Go/Python (ties into the plan's Day 9/Day 13 LLM integration work).
- `contexts/` — saved conversation contexts (session recaps, decisions made, state to resume from).
- `plans/` — saved plans (implementation/study plans produced during sessions, beyond the top-level GO_STUDY_PLAN.md).

Folders are currently empty scaffolding — populate them as each day's exercises are written, one subfolder or file per topic/day as it comes up naturally (don't pre-create structure ahead of need).

## Working conventions

- Teach by analogy to Python where it shortens the explanation (user's existing mental model).
- Prioritize runnable code over pure theory — each topic should land with something the user can execute (`go run`, `go test`).
- Standard Go tooling applies once code exists: `go run <file>.go`, `go test ./...`, `go build`, `gofmt`/`golangci-lint` for style. No custom build system.
- Keep code in this repo idiomatic Go (per plan's Day 14 focus), not Python-flavored Go.
