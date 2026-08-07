# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Personal study vault, not a software project. User was approved in a Go/Python + AI hiring process. Senior software engineer overall (already knows SOLID, DDD, Clean Architecture, etc. from experience — Python is their strongest language) but has zero Go experience specifically, and has 2 weeks to get productive. This directory is where the day-by-day learning happens: notes, exercises, and small practice projects.

Two complementary files, both in `contexts/`:
- [contexts/GO_STUDY_PLAN.md](contexts/GO_STUDY_PLAN.md) — the schedule ("when"): 14-day curriculum, fundamentals week 1, applied/job-relevant week 2.
- [contexts/GO-MODULES.md](contexts/GO-MODULES.md) — the content ("what"): topics grouped in modules, each with its own status checkbox and accumulating "Contexto/notas" (persists regardless of which day it was actually studied on). This is the vault's pattern for subject content — future subjects (Python, arquitetura, IA) get their own `*-MODULES.md` file the same way, also under `contexts/`.

Check both before assuming where the user is: GO_STUDY_PLAN.md for pacing/deadline, GO-MODULES.md for what's actually been covered and what notes exist on it.

## Folder layout

- `go/` — Go exercises and mini-projects for the study plan, plus `go/config/reference/` (idiomatic Go examples of each principle from contexts/) and `go/config/` itself reserved for future language-specific config (linter settings, etc.).
- `python/` — Python-side material (used as the reference language to explain Go concepts by comparison, and for any Python parts of the study), plus `python/config/reference/` (same principles, Python-idiomatic examples — the comparison anchor).
- `infra/` — infra/concurrency-focused material (worker pools, rate limiting, deployment-adjacent topics), matching the "infra" angle of the target role.
- `ia/` — AI-focused material: LLMs, MCP (Model Context Protocol), generative AI, and related topics — including how these get integrated from Go/Python (ties into the plan's Day 9/Day 13 LLM integration work).
- `contexts/` — living reference docs the study workflow reads/writes: GO_STUDY_PLAN.md (schedule) and GO-MODULES.md (per-topic content, one `*-MODULES.md` per subject), plus saved conversation contexts (session recaps, decisions made, state to resume from).
  - `contexts/common/` — language-agnostic reference docs (SOLID.md, CLEAN-CODE.md) shared by every language in the vault.
  - `contexts/plans/` — saved plans (implementation/study plans produced during sessions).

Each language folder gets one subfolder per topic (kebab-case, no numeric prefix — study order lives in that language's README.md, not in folder names, so inserting a topic later never requires renaming existing folders), scaffolded deliberately via `/init-lang` rather than pre-created ahead of need.

## Working conventions

- Teach by analogy to Python where it shortens the explanation (user's existing mental model).
- Tie explanations back to SOLID (and other design principles the user already knows as a senior engineer) from the very first topic — not deferred until Módulo 2 of GO-MODULES.md. Módulo 2 goes deep on SOLID/DDD/Clean Architecture; earlier topics should still name the relevant principle in passing when it's naturally touched (e.g. small interfaces in Dia 2 → mention Interface Segregation/DIP briefly). [contexts/common/SOLID.md](contexts/common/SOLID.md) and [contexts/common/CLEAN-CODE.md](contexts/common/CLEAN-CODE.md) hold language-agnostic definitions of each principle (explanation lives ONLY here); concrete idiomatic examples (code only, no re-explanation) live per-language under `config/reference/` ([go/config/reference/SOLID.md](go/config/reference/SOLID.md), [go/config/reference/CLEAN-CODE.md](go/config/reference/CLEAN-CODE.md), [python/config/reference/SOLID.md](python/config/reference/SOLID.md), [python/config/reference/CLEAN-CODE.md](python/config/reference/CLEAN-CODE.md)) — cite from these rather than re-deriving the explanation, and always let the concrete example follow that language's own idiom, not a translation from another language.
- Prioritize runnable code over pure theory — each topic should land with something the user can execute (`go run`, `go test`).
- Standard Go tooling applies once code exists: `go run <file>.go`, `go test ./...`, `go build`, `gofmt`/`golangci-lint` for style. No custom build system.
- Keep code in this repo idiomatic Go (per plan's Day 14 focus), not Python-flavored Go.

## .claude/ tooling

- `permissions.allow` in `.claude/settings.json` — `go run/test/build`, `gofmt`, `golangci-lint`, `python -m venv`, `pip install`, `pytest` run without a permission prompt.
- Slash commands (`.claude/commands/`): `/topic <nome>` loads a topic from `contexts/GO-MODULES.md` (or another `contexts/*-MODULES.md`) and runs the lesson/exercise, syncing status back to both files; `/review [arquivo]` reviews the latest (or given) Go exercise; `/quiz [topico]` quizzes on topics already covered; `/exercise [topico] [linguagem]` generates a standalone practice exercise without the full lesson or status tracking; `/new-language <nome>` scaffolds a new language folder (`config/reference/SOLID.md` + `config/reference/CLEAN-CODE.md`, optionally a `*-MODULES.md`) following the existing go/python pattern; `/init-lang <linguagem>` (named to avoid colliding with the built-in `/init`) reads that language's `*-MODULES.md` and generates its `README.md` plus one topic folder per entry, in study order, only when the language folder has no topic content yet; `/add-topic <linguagem> <nome>` registers one new topic in both `*-MODULES.md` and the language folder without touching existing topic folders.
- `.claude/agents/go-tutor.md` — dedicated subagent for Go teaching, same conventions as above, usable standalone for focused Go Q&A without pulling in unrelated session context.
