# LearnOS — Adaptive AI Tutoring System

> An AI tutor that **assesses, teaches, evaluates, and decides what to teach
> next** — and resurfaces earlier material for review — built as an explicit
> multi-agent system with the adaptive reasoning made visible.

<!-- Replace the line below with media/demo.gif once recorded -->
<p align="center">
  <img src="media/demo.gif" alt="LearnOS walkthrough" width="800">
  <br><em>90-second walkthrough: concept map → adaptive tutoring → live decision trace → lecturer insight suite</em>
</p>

> **Status:** Working system, in active development and preparing for a pilot
> with a higher-education institution. This is a showcase repository — see
> [What's in this repo](#whats-in-this-repo). For a live walkthrough or access
> to the full system, [contact me](#contact).

---

## What it is

A web app where a student learns a subject (demo: **Supply & Demand**) with an
AI tutor. The tutor teaches concepts in prerequisite order, checks
understanding with live-generated questions, adapts what to teach next based on
how the student does, and brings earlier material back for spaced review. The
home screen is a **concept map** that makes the system's decisions visible —
mastery state per concept, prerequisite edges, nodes unlocking as the student
progresses.

Lecturers get a separate surface: a class metrics board plus an **AI insight
suite** that clusters the class's misconceptions, proposes next-lecture plans,
and drafts classroom interventions — all behind a privacy boundary where
**no individual student data is even expressible**.

## Why it's interesting (engineering highlights)

- **Two-agent system, explicitly orchestrated.** A **Tutor** agent runs a
  Prime → Teach → Assess loop and calls an **Evaluator** agent as a tool to
  score free-text answers. Wired as a small LangGraph, not a bare prompt loop —
  proportional to the real structure (two agents + a handoff + tools).
- **Adaptive decisions on coarse buckets, not raw floats.** Mastery is tracked
  as a float (EMA-updated) but every *decision* — unlock a concept, move on vs.
  reteach, prerequisite met, schedule review — is made on coarse buckets
  (`not-yet` / `shaky` / `solid`). Robust by design.
- **The reasoning is the product, so it's inspectable.** Every decision emits a
  structured event; a live **dev-trace panel** streams them over **SSE** during
  a turn — you watch the agent decide in real time, not just see the answer.
- **Privacy by construction, not by policy.** The lecturer's only data path is
  an interface whose method signatures can return *only* aggregates
  (counts / means / distributions). A student identity literally cannot cross
  the boundary — it's structurally inexpressible. See
  [ARCHITECTURE.md](ARCHITECTURE.md#privacy-by-construction).
- **LLM safety treated as a first-class threat model.** The insight suite that
  reads student-derived text defends against prompt injection with identity
  stripping, delimiter discipline, structured output, a deterministic output
  gate, canary-token tests, and a code-counts/LLM-writes split where the model
  never controls counts or gating. Behavior is pinned by golden eval suites.
- **Deliberate scope discipline.** RAG was *dropped* (the source text fits in
  context); spaced repetition is a simple `pass→×2 / fail→1-day` rule, not
  FSRS; study-modes are a wired-but-inert seam. The grade-worthy move was
  choosing *not* to over-build — and documenting why.

## Architecture at a glance

```
                 ┌──────────────────────────────────────────┐
   Student  ──▶  │  React + Vite UI  (concept map, tutoring) │
                 └───────────────┬──────────────────────────┘
                                 │  SSE (live agent steps + decision events)
                 ┌───────────────▼──────────────────────────┐
                 │  FastAPI  ·  JWT auth (httpOnly cookie)    │
                 ├───────────────────────────────────────────┤
                 │  LangGraph orchestration                  │
                 │    Tutor agent ──tool──▶ Evaluator agent  │
                 ├───────────────────────────────────────────┤
                 │  Planner · Scheduler · Mastery updater    │
                 │  (decisions on coarse buckets)            │
                 ├───────────────────────────────────────────┤
                 │  Repository interfaces (storage-agnostic) │
                 │    · student / mastery / attempts         │
                 │    · LecturerAnalytics (aggregates only)  │
                 └───────────────┬──────────────────────────┘
                                 │
                 ┌───────────────▼──────────────────────────┐
                 │  SQLite (SQLAlchemy) + course.json        │
                 │  durable LangGraph checkpointer           │
                 └───────────────────────────────────────────┘
```

Full design reasoning and illustrative code excerpts:
**[ARCHITECTURE.md](ARCHITECTURE.md)**.

## Tech stack

**Backend:** Python · FastAPI · LangGraph · Anthropic Claude · SQLAlchemy /
SQLite · argon2 + JWT · Server-Sent Events
**Frontend:** React · TypeScript · Vite
**Quality:** pytest (offline, stubbed LLM) + golden **eval suites** for live
LLM behavior (clustering counts, injection resistance, grounded outputs)

## What's in this repo

This is a **showcase**, not the deployable source. The full LearnOS codebase is
a private repository while the project heads into a pilot. Kept private on
purpose: the **prompts** (the core IP of an LLM product), the **eval suites**,
and the full implementation. What you'll find here:

- this overview and a [demo walkthrough](#what-it-is)
- **[ARCHITECTURE.md](ARCHITECTURE.md)** — the design decisions and trade-offs,
  with a few illustrative, non-deployable code excerpts
- screenshots in [`media/`](media/)

Happy to walk through the live system or the private codebase on request.

## Contact

**Chen Swissa** — chennys25@gmail.com
<!-- add LinkedIn / portfolio link here -->

---

*LearnOS was originally built as a graduate gen-AI course project and is being
developed toward a real-world pilot. © 2026 — all rights reserved (see
[LICENSE](LICENSE)).*
