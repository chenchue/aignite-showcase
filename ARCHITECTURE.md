# Aignite: Architecture & Design Decisions

This document explains how Aignite is built and why the key choices were made. The code excerpts are illustrative: each one shows a decision, not necessarily the hard part of the system. The prompts, eval suites, and full implementation are not included (see [the repo note](README.md#whats-in-this-repo)).

> **The one sentence the system proves:** it can *assess, teach, evaluate, and decide what to teach next*, and resurface earlier material.

**Contents**

- [System overview](#system-overview)

1. [Two agents, explicitly orchestrated](#1-two-agents-explicitly-orchestrated)
2. [Durable, resumable turns](#2-durable-resumable-turns)
3. [Mastery: track a float, decide on buckets](#3-mastery-track-a-float-decide-on-buckets)
4. [The planner is real, just simple](#4-the-planner-is-real-just-simple)
5. [Spaced review: a rule, not a library](#5-spaced-review-a-rule-not-a-library)
6. [Privacy by construction](#6-privacy-by-construction)
7. [LLM safety as a first-class threat model](#7-llm-safety-as-a-first-class-threat-model)
8. [The evaluation harness](#8-the-evaluation-harness)
9. [The reasoning is visible (SSE + decision log)](#9-the-reasoning-is-visible-sse--decision-log)
10. [Storage behind interfaces](#10-storage-behind-interfaces)
11. [The front-end](#11-the-front-end)
12. [Out of MVP scope (deferred by design)](#12-out-of-mvp-scope-deferred-by-design)

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

**Structured output is a gate, not a hint.** The Evaluator returns a Pydantic model whose `score` is bound to `[0.0, 1.0]`. An out-of-range or malformed result is *rejected and retried*, never clamped into range, so a broken verdict can't quietly become a real grade. The graph also handles a quieter LangChain failure mode: the function-calling parser behind structured output can return `None` instead of raising when the model skips the tool call, so that case is converted into the same retry path. And after a bounded number of failed rounds the turn ends cleanly without touching mastery at all: `route_after_evaluate` can send a failed evaluation to `await_retry` or to a terminal end, but never to `update_mastery`. The loop fails safe, not silent.

**Adaptation is a closed loop, per level.** This is where "adaptive" is actually implemented. When the Evaluator scores an answer it also emits a diagnosis: an `analysis` (the gap) and, only when there's an evident wrong belief, a `misconception`. On a fail those are stored on that concept *at that depth level*, and the next `teach` and `assess` are rebuilt from them, with a misconception handled differently from a gap (the prompt is told to confront and correct a wrong belief, not merely restate the answer). Both are cleared on a pass, so a later review failure at one level never clobbers the active level's context. The loop from evaluator diagnosis, to per-level stored state, to the next prompt is the real adaptation mechanism, not a tone.

## 2. Durable, resumable turns

A tutoring turn is not one request: it has to stop in the middle and wait for a human. After the tutor asks a question, the graph must suspend until the student types an answer, which might arrive in seconds or after a page reload. Aignite models this with LangGraph's node-boundary interrupts over a durable SQLite checkpointer, so a turn survives a stateless HTTP boundary and even a server restart.

```python
_INTERRUPT_AFTER = ["prime", "assess", "await_retry"]

def build_tutoring_graph(checkpointer):
    """Build and return the compiled tutoring graph with the given checkpointer."""
    return _build_graph_builder().compile(
        checkpointer=checkpointer,
        interrupt_after=_INTERRUPT_AFTER,
    )
```

The graph runs to its next interrupt and stops; the client holds only an opaque thread id. The next request rehydrates the full `TutoringState` from the checkpoint, injects the student's input at the phase the state says it is waiting on, and continues from exactly there. There is no in-memory session map to lose on a restart: durability *is* the checkpointer.

Ownership rides on the same mechanism. A session is just a checkpoint thread, so authorization compares the JWT's student id against the `student_id` baked into the checkpointed state: a missing thread is a 404, someone else's thread is a 403, and a guessed session id reads nothing. Identity comes from the token, never from the path. The production graph and the LangGraph Studio graph share one builder and differ only in which checkpointer is injected.

## 3. Mastery: track a float, decide on buckets

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

**Derive, don't store.** Because the EMA and bucket rule are pure and the attempt log is append-only, a question like "has this student ever reached solid here?" is answered by *replaying* the real EMA over their attempts, not by maintaining a separate high-water-mark column that could drift out of sync. The same single definition of mastery runs in the live tutor, the analytics replay, the demo seed, and the tests, so none of them can disagree about what a score means.

## 4. The planner is real, just simple

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

## 5. Spaced review: a rule, not a library

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

A passed level isn't deleted, it becomes a review target. `now` is an injected parameter, which is what lets the demo seed drive this exact code with historical timestamps to make the RETRIEVE bucket non-empty in a single sitting. The scheduler logic is unchanged; only the starting dates are pre-loaded.

## 6. Privacy by construction

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

In the implementation, `student_id` is read from the database to merge a student's diagnoses across depth levels, then dropped before anything is returned, and the results are sorted by text so row order carries no residual correlation to student ids. The question-audit exception is the same shape and additionally omits the student's answer column outright. An identity can't leak from an LLM context because it is never placed in one; the output gate's name and email check (section 7) is defense in depth against a *fabricated* identity, not a load-bearing scrubber. Data minimization *is* the privacy position: there's no consent toggle because there's no individual data to consent about.

## 7. LLM safety as a first-class threat model

The insight suite turns text derived from student answers into lecturer-facing cards, which is a prompt-injection surface. There is a written taint model (`docs/design/llm-threat-model.md`): student answers are untrusted, and the evaluator's `analysis` / `misconception` are *second-order tainted* because they're shaped by student text, so every downstream stage treats them as attacker-influenceable. The governing rule is **the LLM writes, but code counts, validates, and gates**, and the guardrails are deterministic, so no LLM is ever asked to judge another LLM's output on this path.

**Symmetric gates, both deterministic.** A tainted row is filtered on the way in, and every generated field is checked on the way out.

- The **input gate** drops rows whose text matches known injection phrasings before they ever reach the model's context. The patterns are deliberately narrow full-phrase regexes to keep false positives near zero, and the database row is kept for audit: only the model's *view* is filtered.
- The **output gate** is pure string checking with three rules: an identity-leak check (enrolled names by word boundary, emails by substring), a verbatim-copy check (any shared run of 8 words with a source row is treated as a quote, mechanically enforcing paraphrase-only), and a per-field length cap. On a violation the caller drops the card and logs the rule name only, never the content.

```python
VERBATIM_NGRAM = 8   # 8+ identical words in a row is copying, not summarizing

def find_violation(text, *, blocklist=(), source_texts=(), max_length=MAX_FIELD_LENGTH):
    """Returns the violated rule name, or None. Pure string checking, no LLM."""
    if len(text) > max_length:
        return "too_long"
    lowered = text.lower()
    for term in blocklist:                       # the roster: names and emails
        term = term.strip().lower()
        if not term:
            continue
        if "@" in term:                          # email: substring match
            if term in lowered:
                return "identity_leak"
        elif re.search(rf"\b{re.escape(term)}\b", lowered):   # name: word boundary
            return "identity_leak"
    text_grams = _ngrams(_words(text), VERBATIM_NGRAM)
    if text_grams:
        for source in source_texts:              # the raw rows this field came from
            if text_grams & _ngrams(_words(source), VERBATIM_NGRAM):
                return "verbatim_input"
    return None
```

**The model never owns a number.** For each cluster the LLM returns only row ids; code validates each id (it must exist and belong to the cluster's concept), enforces kind purity (a misconception cluster only counts rows that actually hold a misconception), and then sets `student_count = len(valid distinct ids)`. A fabricated or foreign id is silently dropped, and because the ids are assigned `r0, r1, ...` over a deterministic concept-then-text ordering, they carry no information about students or insertion order. Lecture plans get the same treatment: a proposed plan is validated against the prerequisite DAG and the per-session time budget *before* it renders, and an invalid plan is sent back to the model with the exact violations appended for one corrective retry, then refused rather than served if it still fails.

**Idempotent, version-salted synthesis.** A whole synthesis run is cached on a fingerprint: a SHA-256 of the canonical JSON of every input, salted with a `PROMPT_VERSION`. Identical inputs serve the stored run for free; any input change, or a prompt or schema bump, invalidates it. Bumping the version is the deliberate, auditable cache-bust, so there's no force-refresh button to misuse.

```python
def fingerprint(payload) -> str:
    """SHA-256 of the canonical JSON of every synthesis input, salted with
    PROMPT_VERSION. Equal hash means nothing the LLM would see has changed."""
    canon = json.dumps(payload, sort_keys=True, separators=(",", ":"), default=str)
    return hashlib.sha256((PROMPT_VERSION + "\n" + canon).encode()).hexdigest()
```

**Humans stay in the loop for anything that reaches a student.** Class "kits" (intervention scripts) are drafted and then held until a lecturer explicitly approves or edits them. The transition is one-way and one-time (a second decision is a 409), and the system never contacts students on its own. Regeneration is rate-limited two ways (a per-concept cooldown and a per-lecturer daily ceiling), both checked before the model is called, so a throttled request never spends tokens. These guardrails are pinned by the eval harness (section 8).

## 8. The evaluation harness

Prompts are the part of an LLM system most likely to silently regress, so behaviour is pinned by golden eval suites. Two properties are worth calling out.

**Graded by production code, not by another model.** Every case is graded by the *same* functions the API runs: cluster counts from the real card builder, paraphrase from the real output gate, plan validity from the real validator, injection resistance from a canary check, groundedness from anchor terms that only exist in the course text. There's no LLM-as-judge to drift, and the eval can't disagree with production because it *is* production. A fuzzy metric would have to be calibrated against hand labels before it could be counted, so fuzzy metrics aren't used as gates.

**Diffed against a committed baseline.** Results reduce to a `{suite: {case: passed}}` table and are compared against a checked-in last-known-good, so a prompt change reports exactly which cases regressed and which it fixed, not just a pass count. Updating the baseline is an explicit, deliberate human step.

```python
def diff_against_baseline(results, path=BASELINE_PATH):
    """(regressed, fixed) vs the committed last-known-good."""
    baseline = json.loads(path.read_text())["results"]
    current = to_dict(results)
    regressed, fixed = [], []
    for suite, cases in current.items():
        for name, passed in cases.items():
            was = baseline.get(suite, {}).get(name)
            if was is True and not passed:
                regressed.append(f"{suite}/{name}")
            if was is False and passed:
                fixed.append(f"{suite}/{name}")
    return regressed, fixed
```

The deterministic core (counting, validation, gating, scheduling, mastery) is covered by an offline `pytest` suite that stubs the LLM with a fake that mimics the structured-output interface and replays canned results, so the security-critical paths (a failed evaluation never fabricates a score or touches mastery) are tested with no network and no flakiness. The live-LLM evals run separately and skip themselves without an API key.

## 9. The reasoning is visible (SSE + decision log)

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

## 10. Storage behind interfaces

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

The same discipline runs through the services: mastery, scheduling, and every lecturer metric are pure functions with `now` injected and no I/O. That's what lets the demo seed drive the real scheduler with historical timestamps, and the tests pin time-dependent behaviour deterministically, without a parallel reimplementation of the logic they check.

## 11. The front-end

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

## 12. Out of MVP scope (deferred by design)

Scope discipline was itself a design goal. The MVP proves the adaptive loop end to end, and everything off that critical path was consciously deferred, each behind a seam so it drops in without disturbing the layers above. Deferring these was a judgment call, not a wall reached: over-engineering to look impressive would have been the real failure.

| Deferred | Why it's out of the MVP | How it drops in later |
|---|---|---|
| RAG / vector store | One demo subject fits in a single prompt, so retrieval would add latency and failure modes for no recall benefit yet. | A multi-subject corpus is where a retrieval layer slots in behind the existing course-source boundary. |
| FSRS / SM-2 scheduling | `pass → ×2 / fail → 1-day` is enough to demonstrate resurfacing and stays easy to reason about and debug. | The scheduler is one function with an injected clock; a real interval model replaces its body and nothing else. |
| Error-type taxonomy | Freeform `analysis` / `misconception` strings carry the signal the tutor and lecturer need without a premature schema. | The fields already exist; a classifier would annotate them without changing producers or consumers. |
| BKT / richer mastery | EMA over coarse buckets is explainable and stable against noisy LLM scores. | Mastery sits behind `MasteryRepository`; a Bayesian model replaces the update rule alone. |
| Per-strategy session behaviour | The strategy is wired end to end and logged, but branching on it is deferred so the planner stays one readable rule. | The seam is already live; per-mode weighting fills the body that is intentionally inert today. |
| Multi-model evaluator consensus | One Evaluator with an iterated prompt, pinned by evals, is accurate enough for the MVP. | The evaluate node is one graph node; a consensus panel becomes a fan-out behind the same handoff. |

Each row is a boundary chosen on purpose, with the interface already in place. Filling any of them in means replacing a body, with nothing above it changing.
