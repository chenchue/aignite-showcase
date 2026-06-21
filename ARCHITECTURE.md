# LearnOS — Architecture & Design Decisions

This document explains *how* LearnOS is built and *why* the key choices were
made. The code excerpts below are illustrative — chosen because they show a
design decision, not because they're the hard part. The prompts, evaluation
suites, and full implementation are not included (see
[the repo note](README.md#whats-in-this-repo)).

> **The one sentence the system proves:** it can *assess, teach, evaluate, and
> decide what to teach next* — and resurface earlier material.

---

## 1. Two agents, explicitly orchestrated

The intelligence is two cooperating LLM agents, not one prompt:

- **Tutor** — runs the inner loop for one concept at one depth level:
  **Prime** (surface what the student already knows) → **Teach** (a minimal,
  adapted explanation) → **Assess** (a *live-generated* question). Questions
  are never stored in the course model; they're generated per turn and adapted
  to the student's prior mistakes.
- **Evaluator** — called by the Tutor *as a tool*. It scores a free-text answer
  `0.0–1.0` and returns a structured result: one field per audience —
  `feedback` (student-facing), `analysis` and `misconception` (for tutor
  adaptation and the lecturer dashboard). Freeform strings, deliberately **no
  error-type taxonomy**.

This is wired as a small **LangGraph** — enough to express the real two-agent
handoff and tool use, and no more. A multi-subgraph framework would have been
over-engineering for two agents; a bare `while` loop would have hidden the
structure the design is *about*. The orchestration is scoped to match the
actual agent structure.

## 2. Mastery: track a float, decide on buckets

Mastery per concept *per depth level* is a single float, updated with an
exponential moving average so recent performance dominates without erasing
history:

```python
def update_mastery(overlay, concept_id, level, score, logger):
    """EMA update: new = old*0.7 + score*0.3."""
    level_mastery = overlay.concepts[concept_id].levels[level]
    old_score = level_mastery.score
    new_score = ema_update(old_score, score)   # old*0.7 + score*0.3
    level_mastery.score = new_score
    logger.log("mastery_updated", {
        "concept": concept_id, "level": level.value,
        "before": old_score, "after": new_score,
        "raw_score": score, "bucket": level_mastery.bucket.value,
    })
```

The float is stored, but **no decision is ever made on the raw float.** Every
decision — unlock a concept, reteach vs. move on, prerequisite met, mastered? —
reads a **coarse bucket**:

```
score < 0.4   → not-yet
0.4 – 0.7     → shaky
score > 0.7   → solid
```

Why this matters: an LLM-graded score is noisy. Branching on `0.71 vs 0.69`
would make the system jittery and unexplainable. Branching on three buckets
makes every decision robust and easy to narrate to a student ("this is solid —
let's go deeper") or a grader.

## 3. The planner is real, just simple

A study session is three kinds of work, in order: **RETRIEVE** (due reviews),
**GO DEEP** (push a started concept to its next level), **NEW** (teach an
unlocked concept from scratch). The planner is a flat graph-walk gated by
mastery buckets and due-dates — a *real* planner, deliberately not a time-budget
optimizer:

```python
def plan_session(course, overlay, strategy, logger, session_duration=30, now=None):
    # RETRIEVE: levels whose review is due and not not-yet  (max 3)
    # DEEP:     started concepts, next non-solid level       (max 3)
    # NEW:      unlocked (all prereqs solid), not started    (max 3)
    ...
    logger.log("session_planned", {
        "strategy": strategy.value,          # threaded through, intentionally inert
        "session_duration": session_duration,
        "retrieve": ..., "deep": ..., "new": ...,
    }, student_id=overlay.student_id)
    return plan
```

Note `strategy` and `session_duration` are accepted, threaded, and **logged but
unused**. They're a deliberate *seam*: the interface for per-strategy behavior
exists and is wired (the log line proves it's connected), but the behavior
behind it is intentionally not built yet. Simplicity lives in the body, never in
the interface — so filling it in later changes nothing above it.

A prerequisite check is one bucket read:

```python
def prerequisites_met(concept_id, course, overlay):
    for prereq_id in course.prerequisites_for(concept_id):
        if overlay.concepts[prereq_id].levels[KNOW].bucket != SOLID:
            return False
    return True
```

## 4. Spaced review: a rule, not a library

Each depth level keeps its own review date and decays independently. The
scheduler is a single rule — no FSRS, no SM-2, no dependency:

```python
def schedule_review(level_mastery, passed, ...):
    """pass -> interval x2, fail -> reset to 1 day."""
    if not passed:
        new_interval = 1
    elif level_mastery.next_review is not None:
        old = max((level_mastery.next_review - now).days, 1)
        new_interval = old * 2
    else:
        new_interval = 1
    level_mastery.next_review = now + timedelta(days=new_interval)
```

A passed level isn't deleted — it becomes a *review* target. (For the demo,
state is seeded so the RETRIEVE bucket is non-empty in a single sitting; the
scheduler logic is unchanged.)

## 5. Privacy by construction

The lecturer surface is **aggregates only** — and that's enforced structurally,
not by a policy doc or a code-review rule. The lecturer's *only* read path is
one interface whose method signatures can return counts, means, and
distributions, but **never accept or return a student identity**. Individual
rows are not "forbidden" — they're *inexpressible*:

```python
class LecturerAnalyticsRepository(ABC):
    """The lecturer's only read path. Every method returns aggregate shapes
    (counts, means, distributions) — no method accepts or returns a student
    identity, so individual rows are inexpressible through this boundary.
    At scale, this is the slot where database-enforced access control lands
    (a lecturer DB role granted SELECT only on aggregate views)."""

    @abstractmethod
    def enrolled_count(self) -> int: ...

    @abstractmethod
    def level_bucket_counts(self) -> dict[tuple[str, str], dict[str, int]]:
        """{(concept_id, level): {"not_yet": n, "shaky": n, "solid": n}}."""

    @abstractmethod
    def stuck_counts(self) -> dict[str, int]: ...
    # ...all returns are aggregate shapes; no student_id anywhere
```

The two cases that *need* text (clustering misconceptions, auditing weak
questions) return identity-stripped, text-sorted dataclasses with **no student
field** — so even those exceptions can't carry an identity:

```python
@dataclass(frozen=True)
class ActiveIssue:
    """One student's current diagnosis at one concept — identity-stripped by
    construction: the dataclass has no student field, so a student id cannot
    cross this boundary even by accident."""
    concept_id: str
    level: str
    analysis: str | None
    misconception: str | None
```

Data minimization *is* the privacy position: there's no consent toggle because
there's no individual data to consent about.

## 6. LLM safety as a first-class threat model

The insight suite reads text *derived from student answers* — a prompt-injection
surface. The design treats generated text as tainted and stages the pipeline so
the **LLM writes, but code counts, validates, and gates**:

- the model proposes clusters and prose; **code** computes every count from
  validated row ids — the model never controls a number a lecturer sees;
- proposed lecture plans are validated against the prerequisite DAG and the time
  budget *before* they render;
- a deterministic **output gate** enforces length caps and verifies no generated
  card smuggled out an identity (defense in depth — identities never enter an
  LLM context in the first place);
- class "kits" (intervention scripts) are **human-in-the-loop**: drafted, then
  held until a lecturer explicitly approves — the system never contacts
  students on its own.

Behavior is pinned by **golden eval suites** (clustering counts, injection
resistance via canary tokens, paraphrase enforcement, DAG-valid plans, grounded
kits, evaluator null-discipline). Prompt changes are gated on these evals.
*(The eval cases and prompts themselves are kept private.)*

## 7. The reasoning is visible (SSE + decision log)

The whole point is *adaptive decisions*, so every decision is inspectable. One
centralized `logEvent(type, detail)` emits structured events at each decision
point (`session_planned`, `concept_selected`, `assess`, `evaluated`,
`mastery_updated`, `review_scheduled`, …). These stream to the client over
**Server-Sent Events** *as they happen* — the tutoring API never pretends a
multi-step agent is a single request/response call. The UI and a toggleable
**dev-trace panel** update mid-turn.

This one event stream does three jobs: it's how the adaptive flow is verified,
how the system's reasoning is shown live in a demo, and the captured evidence
for the writeup. Student-facing surfaces get friendly phrasing ("Let's quickly
review Demand — it's been a while"); raw values (mastery floats, strategy codes)
stay in the dev panel only.

## 8. Storage behind interfaces

All storage sits behind **repository interfaces** — the logic depends on
`MasteryRepository`, never on a concrete store. Today that's one SQLite database
via SQLAlchemy (plus a static `course.json` and a durable LangGraph
checkpointer). Swapping to Postgres later is a connection string, not a
redesign. Students are keyed by integer id; identity always comes from the JWT
(`get_current_student`), never from a client-supplied name or id.

## 9. What was deliberately *not* built

Scope discipline was an explicit design goal — over-engineering to look
impressive is itself a judgment failure. Chosen *against*, on purpose:

| Not built | Why |
|---|---|
| RAG / vector store | The whole source text fits in the prompt. |
| FSRS / SM-2 spaced repetition | `pass→×2 / fail→1-day` is enough for the MVP. |
| Error-type taxonomy | `analysis`/`misconception` stay freeform strings. |
| BKT mastery | EMA + coarse buckets is sufficient and explainable. |
| Per-strategy session behavior | Wired as an inert seam; behavior deferred. |
| Multi-model evaluator consensus | One Evaluator, prompt iterated instead. |

Each of these *exists as a seam* where the demo path needs the boundary, with a
trivial body — so the architecture stays honest now and upgrades stay clean
later.
