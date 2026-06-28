# Aignite: Architecture & Design Decisions

This document explains how Aignite is built and why the key choices were made. The code excerpts are illustrative: each one shows a decision, not necessarily the hard part of the system. The prompts, eval suites, and full implementation are not included (see [the repo note](README.md#whats-in-this-repo)).

> **The one sentence the system proves:** it can *assess, teach, evaluate, and decide what to teach next*, and resurface earlier material.

**Contents**

- [System overview](#system-overview)

1. [Two agents, explicitly orchestrated](#1-two-agents-explicitly-orchestrated)
2. [Mastery: track a float, decide on buckets](#2-mastery-track-a-float-decide-on-buckets)
3. [The planner is real, just simple](#3-the-planner-is-real-just-simple)
4. [Spaced review: a rule, not a library](#4-spaced-review-a-rule-not-a-library)
5. [Privacy by construction](#5-privacy-by-construction)
6. [LLM safety as a first-class threat model](#6-llm-safety-as-a-first-class-threat-model)
7. [The reasoning is visible (SSE + decision log)](#7-the-reasoning-is-visible-sse--decision-log)
8. [Storage behind interfaces](#8-storage-behind-interfaces)
9. [The front-end](#9-the-front-end)
10. [What was deliberately not built](#10-what-was-deliberately-not-built)

---

## System overview

Aignite is layered: a React client; a FastAPI edge that authenticates with a JWT cookie and streams agent steps over SSE; a LangGraph state machine that holds the two-agent tutoring loop; a set of pure-logic services (planner, scheduler, mastery updater) that decide on buckets; and repository interfaces over SQLite. Each layer depends only on the interface beneath it, so the store, the model, and the UI can each change without disturbing the others.

```
  React + TypeScript  (Vite, React Flow)
    concept map · tutoring view · dev-trace panel
        │   fetch + ReadableStream reader
        │   ◄──  text/event-stream: one frame per agent step, then a final state frame
        ▼
  FastAPI edge   ·   JWT in httpOnly cookie   ·   get_current_student / require_lecturer
        │
        ├─ tutoring ─►  LangGraph state machine  (durable SQLite checkpointer)
        │                 prime ─► teach ─► assess ─► evaluate ─┬─► update_mastery ─► END
        │                                    ▲                  └─► await_retry ──────┘
        │                              route_entry          route_after_evaluate
        │
        ├─ services ─►  planner · scheduler · mastery updater   (pure functions, decide on buckets)
        │
        └─ lecturer ─►  LecturerAnalyticsRepository (aggregates only)
                          └─ insight suite: the LLM writes, code counts / validates / gates
        │
        ▼
  Repository interfaces ─►  SQLite (SQLAlchemy)  +  course.json  +  durable checkpoints
```

### How one tutoring turn flows

1. The student opens a concept, or accepts the planned queue. The client POSTs to the tutoring endpoint and immediately begins reading a `text/event-stream` response.
2. FastAPI resumes the LangGraph run for that student's thread from its durable checkpoint. `route_entry` enters at `prime` for a new concept, or jumps straight to `assess` for a retrieve-only review.
3. The Tutor nodes run: **prime** (surface what the student already knows), **teach** (a short, adapted explanation), **assess** (a question generated live). Each node completes as one streamed `step` frame, so the UI and the dev-trace panel update mid-turn rather than only at the end.
4. The student answers. The graph routes to **evaluate**, a separate agent that scores the free-text answer and returns `{score, feedback, analysis, misconception}`.
5. `route_after_evaluate` branches on the result: a pass goes to **update_mastery** (EMA update, recompute the bucket, schedule the next review); a fail with retries left goes to **await_retry**; an exhausted retry ends the turn.
6. The final graph state is sent as one `state` frame. The client applies it: mastery moves, newly unlocked concepts light up on the map, and the next session's buckets are recomputed.

## 1. Two agents, explicitly orchestrated

The intelligence is two cooperating LLM agents, not one prompt.

- **Tutor** runs the inner loop for one concept at one depth level: **Prime** (surface what the student already knows), **Teach** (a minimal, adapted explanation), **Assess** (a live-generated question). Questions are never stored in the course model. They're generated per turn and adapted to the student's prior mistakes.
- **Evaluator** is a separate agent the graph routes to after Assess (its own graph node and prompt, run at a lower temperature and bound to a structured-output schema; it shares a base model with the Tutor but is a distinct agent). It scores a free-text answer from 0.0 to 1.0 and returns a structured result with one field per audience: `feedback` for the student, `analysis` and `misconception` for tutor adaptation and the lecturer dashboard. Freeform strings, with no error-type taxonomy.

This is wired as a small **LangGraph** state machine: a node per phase, with explicit conditional edges for routing. The two agents are two nodes with a handoff between them, not a tool call. A bare `while` loop would have hidden the structure the design is about; a multi-subgraph framework would have been over-engineering for two agents.

```python
def _build_graph_builder() -> StateGraph:
    builder = StateGraph(TutoringState)

    builder.add_node("prime", prime_node)
    builder.add_node("teach", teach_node)
    builder.add_node("assess", assess_node)
    builder.add_node("evaluate", evaluate_node)
    builder.add_node("await_retry", await_retry_node)
    builder.add_node("update_mastery", update_mastery_node)

    builder.add_conditional_edges(START, route_entry, {"prime": "prime", "assess": "assess"})
    builder.add_edge("prime", "teach")
    builder.add_edge("teach", "assess")
    builder.add_edge("assess", "evaluate")
    builder.add_conditional_edges(
        "evaluate", route_after_evaluate,
        {"update_mastery": "update_mastery", "await_retry": "await_retry", END: END},
    )
    builder.add_edge("await_retry", "evaluate")
    builder.add_edge("update_mastery", END)
    return builder
```

Routing is data-driven (`route_entry`, `route_after_evaluate`), not a chain of `if`s buried in a node. A retrieve-only review skips Prime and Teach and enters at `assess`; a failed evaluation can route to `await_retry` instead of updating mastery.

## 2. Mastery: track a float, decide on buckets

Mastery per concept *per depth level* is a single float, updated with an exponential moving average so recent performance dominates without erasing history. The bucket mapping and the EMA are two pure functions, defined once and reused everywhere (the live update, the analytics replay over attempt history, the seed script).

```python
def bucket_from_score(score: float) -> MasteryBucket:
    if score > 0.7:
        return MasteryBucket.SOLID
    if score >= 0.4:
        return MasteryBucket.SHAKY
    return MasteryBucket.NOT_YET


def ema_update(old: float, score: float) -> float:
    """new = old*0.7 + score*0.3."""
    return old * 0.7 + score * 0.3
```

The float is stored, but **no decision is ever made on the raw float.** Every decision (unlock a concept, reteach or move on, prerequisite met, mastered?) reads the coarse bucket.

Why it matters: an LLM-graded score is noisy. Branching on `0.71` vs `0.69` would make the system jittery and unexplainable. Branching on three buckets makes every decision stable and easy to narrate, to a student ("this is solid, let's go deeper") or to a grader.

## 3. The planner is real, just simple

A study session is three kinds of work, in order: **RETRIEVE** (due reviews), **DEEP** (push a started concept to its next level), **NEW** (teach an unlocked concept from scratch). The planner is a flat graph-walk gated by mastery buckets and due-dates. It's a real planner, deliberately not a time-budget optimizer.

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

`strategy` and `session_duration` are accepted, threaded, and **logged but unused**. They're a deliberate seam: the interface for per-strategy behaviour exists and is wired (the log line proves it's connected), but the behaviour behind it isn't built yet. Simplicity lives in the body, never in the interface, so filling it in later changes nothing above it.

A prerequisite check is one bucket read:

```python
def prerequisites_met(concept_id, course, overlay):
    for prereq_id in course.prerequisites_for(concept_id):
        if overlay.concepts[prereq_id].levels[KNOW].bucket != SOLID:
            return False
    return True
```

## 4. Spaced review: a rule, not a library

Each depth level keeps its own score and review date and is scheduled independently. The scheduler is a single rule: no FSRS, no SM-2, no dependency.

```python
def schedule_review(level_mastery, passed, logger, concept_id, level, now=None, student_id=None):
    """pass -> interval x2, fail -> reset to 1 day. Mutates in place."""
    now = now or datetime.now()
    if not passed:
        new_interval = DEFAULT_INTERVAL_DAYS
    elif level_mastery.next_review is not None:
        old_interval = max((level_mastery.next_review - now).days, DEFAULT_INTERVAL_DAYS)
        new_interval = old_interval * 2
    else:
        new_interval = DEFAULT_INTERVAL_DAYS
    level_mastery.next_review = now + timedelta(days=new_interval)
    logger.log("review_scheduled", {
        "concept": concept_id, "level": level.value,
        "next_review": level_mastery.next_review.isoformat(),
    }, student_id=student_id)
```

A passed level isn't deleted, it becomes a review target. (For the demo, state is seeded so the RETRIEVE bucket is non-empty in a single sitting. The scheduler logic is unchanged; only the starting dates are pre-loaded.)

## 5. Privacy by construction

The lecturer surface is aggregates only, and that's enforced structurally, not by a policy doc or a code-review convention. The lecturer's *only* read path is one interface whose method signatures can return counts, means, and distributions, but never accept or return a student identity. Individual rows aren't forbidden; they're *inexpressible*.

```python
class LecturerAnalyticsRepository(ABC):
    """The lecturer's only read path. Every method returns aggregate shapes
    (counts, means, distributions): no method accepts or returns a student
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

The two features that genuinely need text (clustering misconceptions, auditing weak questions) return identity-stripped, text-sorted dataclasses with no student field, so even those exceptions can't carry an identity.

```python
@dataclass(frozen=True)
class ActiveIssue:
    """One student's current diagnosis at one concept, identity-stripped by
    construction: the dataclass has no student field, so a student id cannot
    cross this boundary even by accident."""
    concept_id: str
    level: str
    analysis: str | None
    misconception: str | None
```

In the implementation, `student_id` is read from the database to merge a student's diagnoses across depth levels, then dropped before anything is returned, and the results are sorted by text so row order carries no residual correlation to student ids. Data minimization *is* the privacy position: there's no consent toggle because there's no individual data to consent about.

## 6. LLM safety as a first-class threat model

The insight suite reads text derived from student answers, which is a prompt-injection surface. The design treats generated text as tainted and stages the pipeline so the **LLM writes, but code counts, validates, and gates**.

- The model proposes clusters and prose. **Code** computes every count from validated row ids, so the model never controls a number a lecturer sees.
- Proposed lecture plans are validated against the prerequisite graph and the time budget *before* they render.
- A deterministic **output gate** enforces length caps and verifies that no generated card smuggled out an identity. This is defense in depth: identities never enter an LLM context in the first place.
- Class "kits" (intervention scripts) are **human-in-the-loop**. They're drafted, then held until a lecturer explicitly approves. The system never contacts students on its own.

Behaviour is pinned by **golden eval suites**: clustering counts, injection resistance via canary tokens, paraphrase enforcement, DAG-valid plans, grounded kits, evaluator null-discipline. Prompt changes are gated on these evals. (The eval cases and prompts themselves are kept private.)

## 7. The reasoning is visible (SSE + decision log)

The whole point is *adaptive decisions*, so I instrumented every decision to be inspectable, for building and demoing the system rather than as a student-facing feature. Every decision on the path emits a structured event (`session_planned`, `concept_selected`, `assess`, `evaluated`, `mastery_updated`, `review_scheduled`): planner, selection, mastery, and review events go through a durable event logger, while the in-graph steps (`assess`, `evaluated`) are emitted as the graph runs. The graph is run with LangGraph's streaming API, which yields one event per node as it completes, and the FastAPI endpoint relays those over **Server-Sent Events**.

```python
async def _stream_run(graph, run_input, config, thread_id):
    """Run the graph to its next interrupt, yielding one step per node, then the state."""
    async for chunk in graph.astream(run_input, config, stream_mode="updates"):
        for node, update in chunk.items():
            if node.startswith("__"):       # interrupt markers, not real nodes
                continue
            yield "step", _format_step(node, update)
    snapshot = await graph.aget_state(config)
    yield "state", _format_response(snapshot.values, thread_id)
```

The tutoring API never pretends a multi-step agent is one request/response call. The UI and a toggleable **dev-trace panel** update mid-turn. This one event stream does three jobs: it's how the adaptive flow is verified, how the system's reasoning is shown live in a demo, and the captured evidence for the writeup. Student-facing surfaces get friendly phrasing ("Let's quickly review Demand, it's been a while"); raw values (mastery floats, strategy codes) stay in the dev panel only.

## 8. Storage behind interfaces

All storage sits behind **repository interfaces**. The logic depends on `MasteryRepository`, never on a concrete store.

```python
class MasteryRepository(ABC):
    @abstractmethod
    def get(self, student_id: int) -> MasteryOverlay | None: ...

    @abstractmethod
    def save(self, overlay: MasteryOverlay) -> None: ...

    @abstractmethod
    def init_fresh(self, student_id: int, concept_ids: list[str]) -> MasteryOverlay: ...
```

Today the concrete store is one SQLite database via SQLAlchemy (plus a static `course.json` and a durable LangGraph checkpointer). The session factory is injected, so tests instantiate their own against a temp file. Swapping to Postgres later is a connection string, not a redesign. Students are keyed by integer id, and identity always comes from the JWT (`get_current_student`), never from a client-supplied name or id.

## 9. The front-end

A React + TypeScript app (Vite, [@xyflow/react](https://reactflow.dev/) for the concept map). Two parts are worth calling out.

**Consuming the decision stream.** `EventSource` only supports GET, so the client reads the `text/event-stream` body itself: a reader loop, a buffer split on `\n\n`, and typed dispatch. Each `step` frame updates the UI live; the final `state` frame resolves the turn.

```typescript
const reader = res.body.getReader();
const decoder = new TextDecoder();
let buffer = "";

for (;;) {
  const { done, value } = await reader.read();
  if (done) break;
  buffer += decoder.decode(value, { stream: true });
  let sep;
  while ((sep = buffer.indexOf("\n\n")) !== -1) {
    const frame = buffer.slice(0, sep);
    buffer = buffer.slice(sep + 2);
    handleFrame(frame); // event: "step" → onStep(...) ; event: "state" → finalState
  }
}
```

The domain is modelled with explicit string-literal unions rather than loose strings, so an impossible bucket or phase is a compile error:

```typescript
// node-level state ("not_started") and per-level bucket ("not_yet") are
// deliberately different unions, so the two can't be mixed up by accident.
export type MasteryState = "not_started" | "shaky" | "solid";
export type LevelBucket = "not_yet" | "shaky" | "solid";
export type DepthLevelKey = "know" | "apply" | "transfer";

export interface MapNode {
  id: string;
  title: string;
  mastery: MasteryState;
  locked: boolean;
  depth_reached: number;
  levels: Record<DepthLevelKey, LevelBucket>;
}
```

**Rendering the map.** The concept map is one component reused at two sizes (a side peek and the full interactive view). Fitting the graph to its container reliably means coordinating the order in which React Flow initializes, the flex layout settles, and the data updates: the fit is deferred and re-triggered by whichever signal lands last (nodes measured, container resized, init), so the view stays correct at any size.

## 10. What was deliberately not built

Scope discipline was an explicit design goal. Over-engineering to look impressive is itself a judgment failure. Chosen against, on purpose:

| Not built | Why |
|---|---|
| RAG / vector store | This single demo subject fits in one prompt; a multi-subject corpus at scale is where retrieval returns. |
| FSRS / SM-2 spaced repetition | `pass → ×2 / fail → 1-day` is enough for the MVP. |
| Error-type taxonomy | `analysis` / `misconception` stay freeform strings. |
| BKT mastery | EMA plus coarse buckets is sufficient and explainable. |
| Per-strategy session behaviour | Wired as an inert seam; behaviour deferred. |
| Multi-model evaluator consensus | One Evaluator, prompt iterated instead. |

Each of these exists as a seam where the demo path needs the boundary, with a trivial body, so the architecture stays honest now and upgrades stay clean later.
