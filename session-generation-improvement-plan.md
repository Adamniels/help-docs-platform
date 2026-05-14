# Session Generation Improvement Plan

The core problem is that the current prompts frame the LLM as a **session designer**
instead of a **teacher**. The system prompt is 3 sentences. The schema instruction says
"content (plain text ok)" which signals to the model that thin content is acceptable.
The YouTube query sits at the same level as the teaching content, implying they are
equivalent — they are not.

The fix has two layers: better prompts (immediate, high impact) and a two-call architecture
for the context section (medium effort, maximum quality).

---

## What each section should actually contain

Before changing anything in code, this is the target:

**Goal (5–10 min)**
What the user will concretely know and be able to do after the session. Why this topic
matters and how it connects to their situation. Two or three real-world examples of the
skill applied. What prior knowledge is assumed. Short but meaningful — not a one-liner.

**Context (30–45 min) — the teaching section**
This is where the user learns. It should read like a mini article or a focused textbook
chapter written by someone who knows the topic deeply. It must:
- Explain what the concept is and why it exists (the problem it solves)
- Build from first principles to practical understanding
- Use concrete examples, analogies, and progressively increasing depth
- Cover 4–6 distinct conceptual points with real explanations, not bullet headers
- Include code snippets, formulas, or diagrams-in-text where relevant
- The YouTube query is a complementary resource — it does not replace teaching

**Hands-on (45–60 min)**
A concrete task with a clear success condition. The first 2–3 steps spelled out so the
user is not staring at a blank page. What "done" looks like. One stretch goal.

**Reflection (10 min)**
Unchanged — already well structured.

---

## Layer 1 — Rewrite the prompts (immediate)

All changes in `workers-platform/app/workflows/side_learning/session_stage_b.py`.

### New system prompt

Replace `build_session_generation_system_prompt()`:

```python
def build_session_generation_system_prompt() -> str:
    return (
        "You are an expert teacher writing a self-contained learning session for one student. "
        "Your job is to actually teach the subject, not to outline it. "
        "The context section is the core of the session — the student learns by reading it, "
        "not by watching videos. Videos and other resources are supplementary only. "
        "Write content in Markdown. Be specific, concrete, and substantive. "
        "Return strict JSON only, no markdown outside of content field values. "
        "Output exactly four sections in order: goal, context, hands-on, reflection."
    )
```

### New output schema instruction

Replace the `## Output schema` block at the end of `build_session_generation_user_prompt()`:

```python
lines.append(
    "\n## Output schema\n"
    'Return JSON: {"sections":[...]} with exactly 4 objects in this order of "id": '
    '"goal", "context", "hands-on", "reflection".\n'
    "\n"
    "### goal\n"
    "Fields: id, label, estimatedMinutes (5–10), type, content, example.\n"
    "content: State clearly what the user will understand and be able to do after this session. "
    "Explain why this topic matters and how it connects to their background. "
    "Give 2–3 concrete real-world examples of the skill in use. "
    "State any assumed prior knowledge. Minimum 120 words.\n"
    'example: One specific, concrete example of the end result (a sentence or short snippet).\n'
    "\n"
    "### context\n"
    "Fields: id, label, estimatedMinutes (30–45), type, content, youtubeQuery.\n"
    "content: THIS IS THE TEACHING SECTION. Write this as a focused article or textbook chapter. "
    "Do not produce a list of headers with one sentence each. "
    "Explain what the concept is and why it exists. "
    "Build from fundamentals to practical understanding using concrete examples and analogies. "
    "Cover 4–6 distinct ideas with full explanations, not just names. "
    "Use Markdown — headers, code blocks, inline code, bold for emphasis. "
    "Minimum 500 words. The youtubeQuery is a supplementary resource, not a substitute for teaching.\n"
    "youtubeQuery: A specific search query for a video that complements the written content.\n"
    "\n"
    "### hands-on\n"
    "Fields: id, label, estimatedMinutes (45–60), type, content, outputType.\n"
    "content: Define a concrete task with a clear success condition. "
    "Write out the first 2–3 steps explicitly so the user knows exactly how to start. "
    "State what the finished result looks like. Include one stretch goal. Minimum 150 words.\n"
    "outputType: code | diagram | notes | demo — whatever fits the task.\n"
    "\n"
    "### reflection\n"
    "Fields: id, label, estimatedMinutes (10–15), type, content, prompts.\n"
    "content: Brief framing for the reflection (2–3 sentences).\n"
    "prompts: 3–4 questions. At least one should be specific to this topic, not generic.\n"
)
```

This single change will significantly improve output quality with no architectural change.

---

## Layer 2 — Separate the context call (medium effort, maximum quality)

The fundamental constraint of a single LLM call is that the model is balancing four
sections at once. The context section competes for token budget with the others. A
dedicated call for context — with a prompt written entirely for teaching — produces
noticeably better output.

### New activity: `generate_context_section`

Add to `activities.py`. This call gets a much higher timeout and a teaching-only prompt.

```python
@activity.defn
async def generate_context_section(
    context_dict: dict[str, Any],
    topic_title: str,
    user_feedback: str | None,
) -> str:
    """LLM: write the full teaching content for the context section."""
    settings = get_settings()
    context = MemoryContextV1.model_validate(context_dict)
    system = build_context_section_system_prompt()
    user = build_context_section_user_prompt(context, topic_title, user_feedback)
    model = _side_learning_session_llm_model(settings)
    payload = {
        "model": model,
        "messages": [
            {"role": "system", "content": system},
            {"role": "user", "content": user},
        ],
        "temperature": 0.6,
    }
    # plain text response — no json_object mode, which removes the JSON overhead
    # and lets the model focus entirely on prose quality
    async with httpx.AsyncClient(timeout=300.0) as client:
        response = await client.post(url, headers=headers, json=payload)
    response.raise_for_status()
    return response.json()["choices"][0]["message"]["content"].strip()
```

### New prompt builders for context-only call

Add to `session_stage_b.py`:

```python
def build_context_section_system_prompt() -> str:
    return (
        "You are a domain expert writing one focused teaching section for a student. "
        "Write in Markdown as if writing a high-quality blog post or textbook chapter. "
        "Do not produce headers with thin bullet points. Explain things properly. "
        "Use examples, analogies, and progressively increasing depth. "
        "Your output is plain Markdown text — no JSON wrapper, no preamble."
    )


def build_context_section_user_prompt(
    context: MemoryContextV1,
    topic_title: str,
    user_feedback: str | None,
) -> str:
    lines = [
        f"Topic: {topic_title.strip()}",
        "",
        "Write the full teaching content for this topic. Cover:",
        "- What it is and why it exists (the problem it solves)",
        "- How it works — the core mechanics, from simple to complex",
        "- 2–3 concrete examples or analogies the student can follow",
        "- Common misunderstandings or things that trip people up",
        "- Where this fits in the bigger picture of the field",
        "",
        "Requirements:",
        "- Minimum 600 words",
        "- Use Markdown: headers (##, ###), code blocks, bold for key terms",
        "- Write to teach, not to outline",
        "- Do not mention YouTube or external resources",
    ]

    if user_feedback and user_feedback.strip():
        lines.insert(1, f"User focus: {user_feedback.strip()}")

    # Inject relevant memory context
    learning_semantics = [
        s for s in context.semantic_memories if (s.domain or "").lower() == "learning"
    ]
    if learning_semantics:
        lines.append("\n## What the student already knows (adjust depth accordingly)")
        for s in learning_semantics[:12]:
            lines.append(f"- {s.claim}")

    if context.profile_facts:
        lines.append("\n## Student background signals")
        for p in context.profile_facts[:10]:
            lines.append(f"- ({p.source}) {p.text}")

    return "\n".join(lines)
```

### Updated workflow: run both calls, merge result

In `workflow.py`, Stage B becomes:

```python
# Fetch context once
ctx = await workflow.execute_activity(
    "fetch_memory_context_for_session_generation",
    args=[request.topic_title, request.user_feedback],
    start_to_close_timeout=timedelta(seconds=60),
)

# Run structure and context generation in parallel
sections_task = workflow.execute_activity(
    "generate_learning_session",
    args=[ctx, request.topic_title, request.user_feedback],
    start_to_close_timeout=timedelta(seconds=300),
)
context_task = workflow.execute_activity(
    "generate_context_section",
    args=[ctx, request.topic_title, request.user_feedback],
    start_to_close_timeout=timedelta(seconds=300),
)
sections, context_content = await asyncio.gather(sections_task, context_task)

# Merge: replace the context section's content with the dedicated output
for section in sections:
    if section.get("id") == "context":
        section["content"] = context_content
        break
```

This runs both calls in parallel, so latency is the max of the two rather than the sum.

---

## Layer 3 — Model configuration (quick win)

The `openai_side_learning_session_model` setting exists specifically so session generation
can use a different model than the default. If it is currently unset, it falls back to
`openai_model` which may be a smaller/cheaper model.

Session generation is the highest-value LLM call in the entire product — the quality of
what the user reads directly determines whether they come back. Use the strongest model
available here, even if other workflow calls use a cheaper one.

```bash
# .env
OPENAI_SIDE_LEARNING_SESSION_MODEL=gpt-4o  # or whatever the best available model is
```

---

## What to do and in what order

**Step 1 — Rewrite the prompts (Layer 1)**
This is one function in `session_stage_b.py`. High impact, zero risk, do this first.
Verify output quality before proceeding.

**Step 2 — Set the model (Layer 3)**
Check what `OPENAI_SIDE_LEARNING_SESSION_MODEL` is currently set to. If it is unset or
pointing at a non-frontier model, fix it. This compounds with better prompts.

**Step 3 — Separate context call (Layer 2)**
Once Layer 1 output looks good structurally but you want more depth in context, add the
dedicated context activity. Because it runs in parallel with the structure call, the only
cost is a small increase in token spend — latency stays the same.

---

## What not to do

Do not add more sections. The four-section structure is the right product decision —
adding sections would make sessions feel heavier, not richer.

Do not try to make the goal section long. It should be clear and concrete, not exhaustive.
The improvement there is specificity and real examples, not word count.

Do not change the reflection section. It works well as-is.
