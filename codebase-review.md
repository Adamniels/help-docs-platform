# Codebase Review — Structural Problems and Fix Proposals

Early-stage audit of the .NET backend and Python workers. Two issues already in progress
are excluded (WorkflowRunRequest typed subclasses, IMemoryContextProvider routing).
Everything below is a fresh finding.

Issues are ordered by priority: fix the high ones before adding new features.

---

## 1. Stage strings hardcoded across .NET and Python — HIGH

**Problem:**
Stage values like `"propose_topics"`, `"generate_session"`, `"analyze_reflection"` are
written as raw string literals in every .NET command handler that starts a workflow.
The Python side has `SideLearningStage` enum but the .NET side has no equivalent. If a
stage is renamed or a typo is introduced, the workflow silently fails at Temporal with no
compile-time error.

**Files:**
- `Platform.Application/Features/SideLearning/Sessions/Create/CreateSideLearningSessionCommandHandler.cs:54` — `stage = "propose_topics"`
- `Platform.Application/Features/SideLearning/Sessions/SelectTopic/SelectSideLearningTopicCommandHandler.cs:55` — `stage = "generate_session"`
- `Platform.Application/Features/SideLearning/Sessions/Reflect/SubmitSideLearningReflectionCommandHandler.cs:54` — `stage = "analyze_reflection"`
- `Platform.Application/Features/SideLearning/Sessions/RefreshTopicProposals/RefreshSideLearningTopicProposalsCommandHandler.cs:59` — `stage = "propose_topics"`

**Fix:**
Add stage constants to `SideLearningWorkflowTypes.cs` so .NET and Python share a single
source of truth for each workflow.

```csharp
// Platform.Application/Features/SideLearning/SideLearningWorkflowTypes.cs
public static class SideLearningWorkflowTypes
{
    public const string WorkflowTypeName = "SideLearningWorkflow";

    public static class Stages
    {
        public const string ProposeTopics   = "propose_topics";
        public const string GenerateSession = "generate_session";
        public const string AnalyzeReflection = "analyze_reflection";
    }
}
```

Replace every string literal in the handlers with `SideLearningWorkflowTypes.Stages.*`.

---

## 2. Stage dispatch is a growing if-chain in workflow.py — HIGH

**Problem:**
`SideLearningWorkflow.run()` is one large method with three sequential `if` blocks, each
~40 lines of validation, activity calls, and result building. Adding a new stage means
copying the block and editing in place. The method will become unreadable as stages grow.

**File:** `workers-platform/app/workflows/side_learning/workflow.py:16-163`

**Fix:**
One file per stage. Each stage is a plain async function that takes the typed request and
returns a `WorkflowRunResult`. The workflow only dispatches.

```python
# app/workflows/side_learning/stages/propose_topics.py
async def run_propose_topics(request: SideLearningWorkflowRequest) -> WorkflowRunResult:
    if not request.session_id:
        return _failed(request, "missing_session_id")
    ctx = await workflow.execute_activity("fetch_memory_context_for_learning", ...)
    # rest of the current Stage A logic
    return _completed(request, titles)

# app/workflows/side_learning/workflow.py
_STAGES = {
    SideLearningStage.PROPOSE_TOPICS:    run_propose_topics,
    SideLearningStage.GENERATE_SESSION:  run_generate_session,
    SideLearningStage.ANALYZE_REFLECTION: run_analyze_reflection,
}

@workflow.run
async def run(self, payload: str) -> WorkflowRunResult:
    request = SideLearningWorkflowRequest.model_validate_json(payload)
    stage = (request.stage or "").strip()
    handler = _STAGES.get(stage)
    if not handler:
        return _failed(request, f"unknown_stage:{stage or 'empty'}")
    return await handler(request)
```

New stages are new files. The workflow itself never changes again.

---

## 3. Domain and workflow filtering duplicated across prompt builders — MEDIUM

**Problem:**
The same two filter expressions appear identically in `memory_mapper.py` and
`session_stage_b.py`. Any future prompt builder for a new stage will copy them again.

```python
# memory_mapper.py:42-47  AND  session_stage_b.py:38-43 — identical
learning_semantics = [
    s for s in context.semantic_memories if (s.domain or "").lower() == "learning"
]
other_semantics = [
    s for s in context.semantic_memories if (s.domain or "").lower() != "learning"
]
# Also:
rules = [r for r in context.procedural_rules if (r.workflow_type or "").lower() == "side_learning"]
```

**Files:**
- `workers-platform/app/workflows/side_learning/memory_mapper.py:42-65`
- `workers-platform/app/workflows/side_learning/session_stage_b.py:38-61`

**Fix:**
One shared helper. Both prompt builders call it.

```python
# app/workflows/side_learning/memory_filters.py
def split_semantics_by_domain(
    memories: list[SemanticMemoryContextV1],
    domain: str,
) -> tuple[list[SemanticMemoryContextV1], list[SemanticMemoryContextV1]]:
    target = domain.lower()
    matching = [m for m in memories if (m.domain or "").lower() == target]
    other    = [m for m in memories if (m.domain or "").lower() != target]
    return matching, other

def filter_rules_by_workflow(
    rules: list[ProceduralRuleContextV1],
    workflow_type: str,
) -> list[ProceduralRuleContextV1]:
    target = workflow_type.lower()
    return [r for r in rules if (r.workflow_type or "").lower() == target]
```

---

## 4. All LLM calls copy the same 20-line HTTP block — MEDIUM

**Problem:**
Every activity that calls OpenAI repeats the same setup: validate API key, build URL,
build headers, build payload dict, post, raise for status, parse choices. The only things
that differ are the prompts, model, temperature, and timeout.

**Files:**
- `workers-platform/app/workflows/side_learning/activities.py:78-108` (propose topics)
- `workers-platform/app/workflows/side_learning/activities.py:190-221` (generate session)
- `workers-platform/app/workflows/side_learning/activities.py:238-267` (topic memory)
- `workers-platform/app/workflows/side_learning/activities.py:315-346` (reflection)

**Fix:**
One shared async helper. Activities just call it.

```python
# app/workflows/shared/llm_client.py
async def call_openai_json(
    system_prompt: str,
    user_prompt: str,
    model: str,
    temperature: float = 0.7,
    timeout: float = 120.0,
) -> dict[str, Any]:
    settings = get_settings()
    if not settings.openai_api_key:
        raise RuntimeError("OPENAI_API_KEY is not set.")
    url = f"{settings.openai_base_url.rstrip('/')}/chat/completions"
    headers = {
        "Authorization": f"Bearer {settings.openai_api_key}",
        "Content-Type": "application/json",
    }
    payload = {
        "model": model,
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt},
        ],
        "response_format": {"type": "json_object"},
        "temperature": temperature,
    }
    async with httpx.AsyncClient(timeout=timeout) as client:
        response = await client.post(url, headers=headers, json=payload)
    try:
        response.raise_for_status()
    except httpx.HTTPStatusError:
        logger.exception("OpenAI call failed status=%s", response.status_code)
        raise
    return json.loads(response.json()["choices"][0]["message"]["content"])
```

Activities shrink to prompt building + parsing.

---

## 5. Memory proposal types validated with if-chains — MEDIUM

**Problem:**
`try_wire_memory_proposal()` in `session_stage_b.py` checks `if pt == "NewSemantic"` and
`if pt == "NewProceduralRule"` with no extension point. Adding a third proposal type means
adding another if-block and another `_validate_*` function in the same file.

**File:** `workers-platform/app/workflows/side_learning/session_stage_b.py:276-313`

**Fix:**
A small registry keyed by proposal type string.

```python
# app/workflows/side_learning/session_stage_b.py

_PROPOSAL_VALIDATORS: dict[str, Callable[[dict], dict | None]] = {
    "NewSemantic":       _validate_new_semantic,
    "NewProceduralRule": _validate_new_procedural,
}

def try_wire_memory_proposal(item: TopicSelectionMemoryProposalLlmItem) -> SideLearningMemoryProposalWire | None:
    pt = (item.proposal_type or "").strip()
    validator = _PROPOSAL_VALIDATORS.get(pt)
    if validator is None or not isinstance(item.proposed_change, dict):
        return None
    fixed = validator(item.proposed_change)
    if fixed is None:
        return None
    # ... rest of the wiring logic
```

Adding a new proposal type is registering a new entry in `_PROPOSAL_VALIDATORS`.

---

## 6. Python activity registry is a manual list — MEDIUM

**Problem:**
`get_registered_definitions()` in the runtime registry is a hand-maintained list of every
activity function. When a new activity is added and not included here, the worker starts
without it and the workflow fails at runtime with no clear error.

**File:** `workers-platform/app/runtime/registry/` (definitions file)

**Fix:**
A decorator that registers activities at import time so the list is always complete.

```python
# app/runtime/registry/activity_registry.py
_activities: list = []

def register_activity(fn):
    """Decorator: marks an activity for automatic registration."""
    _activities.append(fn)
    return fn

def get_all_activities() -> list:
    return list(_activities)

# Usage in activities.py — add @register_activity above @activity.defn
@register_activity
@activity.defn
async def propose_learning_topics(...):
    ...

# Runtime bootstrap — no manual list
from app.runtime.registry.activity_registry import get_all_activities
activities = get_all_activities()
```

---

## 7. WorkflowRunResult status values are bare string literals — LOW

**Problem:**
`status="failed"` and `status="completed"` appear as raw strings at 10+ locations across
`workflow.py`. The .NET side has a proper `WorkflowRunStatus` enum. The Python side has a
`Literal` type annotation but no enum. A new status value requires updating the Literal
and hunting down every usage.

**Files:**
- `workers-platform/app/schemas/workflow_contracts.py:33` — `Literal["completed", "failed", "needs_input"]`
- `workers-platform/app/workflows/side_learning/workflow.py:27, 56, 65, 94, 103, 110, 117, 125, 154, 161`

**Fix:**

```python
# app/schemas/workflow_contracts.py
from enum import StrEnum

class WorkflowStatus(StrEnum):
    COMPLETED   = "completed"
    FAILED      = "failed"
    NEEDS_INPUT = "needs_input"

class WorkflowRunResult(BaseModel):
    ...
    status: WorkflowStatus  # replaces Literal
```

---

## 8. No validation that a workflow type is known before Temporal start — LOW

**Problem:**
`workflowStarter.StartAsync(taskQueue, workflowType, ...)` takes a plain string. A typo
or an unregistered type produces a silent failure at Temporal. The validator for
`StartWorkflowRunCommand` only checks that the string is non-empty, not that it is known.

**Files:**
- `Platform.Application/Features/WorkflowRuns/StartWorkflowRun/StartWorkflowRunCommandValidator.cs`
- `Platform.Application/Abstractions/Workflows/IWorkflowStarter.cs`

**Fix:**
A small registry injected into the validator.

```csharp
// Platform.Application/Abstractions/Workflows/IWorkflowTypeRegistry.cs
public interface IWorkflowTypeRegistry
{
    bool IsKnownType(string workflowType);
}

// Platform.Application/Features/SideLearning/SideLearningWorkflowTypes.cs
// (and any future workflow type file — each registers itself)
public static class SideLearningWorkflowTypes
{
    public const string WorkflowTypeName = "SideLearningWorkflow";
}

// StartWorkflowRunCommandValidator
RuleFor(x => x.WorkflowType)
    .NotEmpty()
    .Must(wt => registry.IsKnownType(wt))
    .WithMessage(wt => $"Unknown workflow type: {wt}");
```

---

## Summary

| # | Problem | Priority | Effort |
|---|---------|----------|--------|
| 1 | Stage strings hardcoded in .NET handlers | HIGH | Small — add constants to one file |
| 2 | Stage dispatch is one giant method in workflow.py | HIGH | Medium — split into per-stage files |
| 3 | Domain / workflow filtering duplicated across prompt builders | MEDIUM | Small — extract one helper |
| 4 | LLM HTTP block copy-pasted in every activity | MEDIUM | Small — extract one helper |
| 5 | Memory proposal types validated with if-chains | MEDIUM | Small — replace with dict registry |
| 6 | Python activity registry is a hand-maintained list | MEDIUM | Small — add decorator |
| 7 | WorkflowRunResult status as bare string literals | LOW | Trivial — add StrEnum |
| 8 | No known-workflow-type validation before Temporal start | LOW | Small — add registry + validator rule |
