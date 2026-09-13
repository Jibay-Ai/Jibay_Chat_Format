# Jibay Chat Format (JCF)

**Version:** 1.0.0

**Author:** JibayAI

**Status:** Production — used to train select Jibay models

**File:** `chat_template.jinja`

Jibay Chat Format (JCF) is a purpose-built chat templating specification designed as a lighter, clearer, and more structurally explicit alternative to ChatML and OpenAI's Harmony format. It was created to solve real problems observed while operating a production, self-hosted, multi-user LLM inference stack (Qwen-family models on `llama.cpp`/`ik_llama.cpp`) — problems around token overhead, prefix-cache stability, ambiguous role boundaries, and inconsistent reasoning/thinking control.

---

## Table of Contents

1. [Why JCF Exists](#why-jcf-exists)
2. [JCF vs ChatML vs Harmony](#jcf-vs-chatml-vs-harmony)
3. [Core Design Principles](#core-design-principles)
4. [Full Role Reference](#full-role-reference)
5. [Fixed Ordering Contract](#fixed-ordering-contract)
6. [Reasoning & Thinking Control](#reasoning--thinking-control)
7. [History Sanitization Rule](#history-sanitization-rule)
8. [Defaults](#defaults)
9. [Full Examples](#full-examples)
10. [Error Resilience](#error-resilience)
11. [⚠️ Critical Warning: Training Requirement](#️-critical-warning-training-requirement)
12. [Integration Notes](#integration-notes)
13. [FAQ](#faq)

---

## Why JCF Exists

Every general-purpose chat template in wide use today (ChatML, Harmony, Llama's own formats) was designed around one or two labs' internal conventions and then generalized outward. In doing so they inherited compromises that make sense for their original use case but hurt when you are:

- Running a **CPU-bound, self-hosted, multi-tenant** inference server where every extra token in the system prefix costs real, measurable latency across every concurrent user.
- Injecting **many distinct context types** per turn (memory, retrieval, environment, notes, files) that need to be **individually identifiable** to the model, not flattened into one undifferentiated system blob.
- Needing **fine-grained, per-request control** over whether the model thinks at all, how hard it thinks, and whether that thinking is exposed to the end user or the training log.
- Wanting a format a human engineer can **visually parse in under a second** while reading raw logs, without mentally parsing nested JSON-in-text or memorizing multi-token special sequences.

JCF was built directly from these operational pressures, not as an academic exercise. It is opinionated, and every opinion maps to a specific failure mode JCF avoids.

---

## JCF vs ChatML vs Harmony

| Property | ChatML | OpenAI Harmony | **Jibay Chat Format** |
|---|---|---|---|
| Role delimiter style | `<\|im_start\|>role ... <\|im_end\|>` | Nested structured channels (`<\|start\|>`, `<\|channel\|>`, `<\|message\|>`, `<\|end\|>` combinations) | Single flat tag pair per role: `<[role_x]> ... <[role_x_end]>` |
| Visual scanability | Medium — role name is inline text | Low — channel/recipient metadata is interleaved with content, hard to skim | **High** — tag name alone tells you the section instantly |
| Number of context types supported natively | 1 (system) + user/assistant | Channels (analysis/commentary/final) but no dedicated memory/retrieval/env/notes primitives | **11 dedicated roles**, each single-purpose |
| Reasoning exposure control | None built-in (bolt-on via prompting) | Channel-based (`analysis` vs `final`) but binary, no effort scale | **Three independent controls**: `enable_thinking`, `show_thinking`, `show_summary` |
| Reasoning effort levels | None | None natively | **7-level enum**: Low / Medium / High / xHigh / Max / Pro / Ultra |
| History reasoning leakage | N/A | Requires manual channel-stripping logic downstream | **Built into the template**: reasoning/summary auto-stripped from every message except the one being generated |
| Malformed/missing-field behavior | Undefined — most implementations error or hallucinate structure | Undefined at the template level | **Defined fallback chain** for every optional field (see [Error Resilience](#error-resilience)) |
| Token overhead per role tag | 3–5 tokens (`<|im_start|>`, role word, `<|im_end|>`) | Higher — channel + recipient + constrain metadata adds multiple extra tokens per turn | **Minimized** — short bracket tags tokenize compactly and cache identically every turn |
| Default system prompt behavior | Model-specific, often silently empty | Model-specific | **Defined**: injects a guaranteed identity string if the caller sends none |
| Ordering guarantees | Loose convention only | Loose convention only | **Fixed, documented contract** (see below) |
| Empty/optional sections | Often emitted anyway (wasted tokens) | Often emitted anyway | **Never emitted unless populated** |

**Bottom line:** ChatML is minimal but under-specified — it gives you three roles and leaves everything else (memory, retrieval, reasoning control, ordering, sanitization) to be improvised per-project, which is exactly where production bugs come from. Harmony is more structured but pays for that structure with heavier channel/recipient metadata on every single message and still leaves memory/retrieval/notes/environment context with no dedicated home. JCF keeps the token-per-turn cost close to ChatML's while giving you Harmony-level (and beyond) structural guarantees, plus first-class primitives for the context types a real production assistant actually needs.

---

## Core Design Principles

1. **One tag, one meaning.** Every role has exactly one opening tag and one closing tag. No nested channel/recipient sub-syntax to parse.
2. **Nothing is emitted unless it has content.** Optional roles (developer, memory, input_files, tools, safety, notes, environment, context, retrieval) are completely absent from the rendered prompt when unused — they never waste a single token "just in case."
3. **The system block is a container, not a single string.** Safety, notes, environment, context, and retrieval are logically part of the system framing and are nested *inside* the system block, before it closes — so the model reads one coherent instructional frame instead of scattered top-level tags competing for attention.
4. **Reasoning visibility is a first-class, three-axis control**, not a prompting trick.
5. **History is always sanitized.** A model must never see its own historical hidden reasoning replayed back as if it were normal conversation content — this teaches bad habits and blows up context length for no benefit.
6. **Fail soft, never fail hard.** Missing fields, malformed booleans, absent message arrays, invalid enum values — none of these should crash the template. Every field has a defined fallback.
7. **Deterministic prefix caching.** Because the system frame's structure and ordering never change turn-to-turn, inference servers relying on prefix/prompt caching (e.g. `llama-server`, vLLM prefix cache) get maximal cache hits, directly reducing latency in multi-turn conversations.

---

## Full Role Reference

| Tag | Role | Purpose | Cardinality |
|---|---|---|---|
| `<[role_developer]>` | developer | Highest-priority operational instructions (above system) — sampling hints, platform-level constraints, debug flags | 0–1, first message only |
| `<[role_system]>` | system | Core persona/instruction block. Contains safety/notes/environment/context/retrieval nested inside it. Always present. | Exactly 1 |
| `<[role_safety]>` | safety | Safety policy or guardrail text, nested inside system | 0–1 |
| `<[role_notes]>` | notes | Metadata about the session/user (name, date, locale, tier), nested inside system | 0–1 |
| `<[role_environment]>` | environment | Runtime/platform environment description (device, app version, OS), nested inside system | 0–1 |
| `<[role_context]>` | context | Ambient situational context not tied to retrieval or memory, nested inside system | 0–1 |
| `<[role_retrieval]>` | retrieval | Retrieved documents/snippets (RAG), nested inside system | 0–1 |
| `<[role_reasoning_effort]>` | reasoning_effort | Declares the requested reasoning depth for this turn | Exactly 1 |
| `<[role_memory]>` | memory | Long-term memory injected from a persistent store | 0–1 |
| `<[role_input_files]>` | input_files | References/content of files the user attached | 0–1 |
| `<[role_tools]>` | tools | Tool/function schema definitions available this turn | 0–1 |
| `<[role_user]>` | user | End-user message | 0–N |
| `<[role_tools_response]>` | tool response | Result returned from a tool call | 0–N |
| `<[role_assistant]>` | assistant | Model output, optionally wrapping `<[reasoning]>` / `<[summary]>` | 0–N |

### Why each role earns its own tag

- **developer vs system** — developer instructions (e.g. sampling parameters, platform overrides) are operationally different from persona/behavior instructions. Conflating them (as ChatML effectively forces you to) makes it impossible to change one without re-parsing the other.
- **memory vs context vs retrieval** — these three are semantically distinct and often come from entirely different subsystems (a persistent vector-store memory service, ephemeral session context, and a RAG pipeline, respectively). Flattening them into one "system" string makes debugging which subsystem caused a bad response far harder. JCF keeps them addressable independently in your backend code and independently visible in raw logs.
- **input_files** — file content has different lifecycle and truncation rules than chat text; giving it a dedicated tag lets you truncate/summarize it without touching the rest of the prompt.
- **notes vs environment** — session metadata (user tier, date) and runtime environment (device/app) come from different code paths in most backends and are useful to reason about separately when debugging.

---

## Fixed Ordering Contract

JCF defines a **strict, non-negotiable rendering order**. This is what makes prefix caching reliable and what makes the format trainable — a model can learn "safety always comes right before the system block closes" only if that is *always* true.

```
1. <[role_developer]>            (only if first message role == developer)
2. <[role_system]>
     ├─ system content (or default identity string if empty)
     ├─ <[role_safety]>          (only if provided)
     ├─ <[role_notes]>           (only if provided)
     ├─ <[role_environment]>     (only if provided)
     ├─ <[role_context]>         (only if provided)
     └─ <[role_retrieval]>       (only if provided)
   <[role_system_end]>
3. <[role_reasoning_effort]>      (always present)
4. <[role_memory]>                (only if provided)
5. <[role_input_files]>           (only if provided)
6. <[role_tools]>                 (only if provided)
7. conversation turns (user / tool / assistant, in original order)
8. generation prompt              (<[role_assistant]>, opened for the model to continue)
```

This order is **fixed by design**, never conditionally reordered based on content — reordering based on content is exactly what destroys prefix cache hit rates in production.

---

## Reasoning & Thinking Control

JCF separates three independent questions that other formats conflate into one:

| Flag | Question it answers | Type | Default |
|---|---|---|---|
| `reasoning_effort` | *How hard should the model think?* | enum: `Low`, `Medium`, `High`, `xHigh`, `Max`, `Pro`, `Ultra` | `Medium` |
| `enable_thinking` | *Should the model produce hidden reasoning at all?* | boolean | `true` |
| `show_thinking` | *Should that reasoning be rendered into the visible transcript/log?* | boolean | `true` |
| `show_summary` | *If reasoning is shown, should a condensed summary also be shown?* | boolean | `true` |

This four-way split matters because in production these are genuinely different decisions made by different parts of a system:

- A **billing/tier system** might set `reasoning_effort` based on the user's plan.
- A **product UI toggle** ("Show thinking") controls `show_thinking` independent of whether thinking is happening at all.
- A **safety/compliance layer** might force `show_thinking=false` for a specific user-facing surface while still wanting `enable_thinking=true` internally for quality.
- A **UX designer** may want the condensed `show_summary` without the full raw chain-of-thought, to keep the interface clean.

When `enable_thinking` is `false`, the template proactively closes the reasoning tag (`<[reasoning_end]>`) immediately after opening the assistant turn in the generation prompt, guaranteeing the model cannot emit a reasoning block even if it was trained with one — a defensive measure against reasoning leaking into surfaces that must not see it.

---

## History Sanitization Rule

**This is one of JCF's most important production-hardening features.**

In any multi-turn conversation, only the **most recent assistant message** (the one currently being generated or just generated) is allowed to carry `<[reasoning]>` / `<[summary]>` content. Every prior assistant turn in the replayed history has its reasoning and summary **stripped entirely**, regardless of what flags are set.

Why this matters:

- **Prevents reasoning replay contamination.** If a model sees its own past hidden reasoning presented as normal historical content across many turns, it starts to imitate the *style* of exposed reasoning as if it were expected user-facing output — a known degradation pattern in reasoning-model fine-tuning.
- **Keeps context length under control.** Reasoning traces are often 3–10x longer than the final answer. Replaying them every turn compounds context usage catastrophically in long conversations.
- **Matches training-time expectations.** Jibay models trained on JCF are trained with this exact stripping behavior in the data pipeline (see `jibay5_dataset_v10` construction), so inference-time replay *must* match training-time replay or the model sees an out-of-distribution history shape.

---

## Defaults

| Setting | Default | Rationale |
|---|---|---|
| `reasoning_effort` | `Medium` | Balanced cost/quality; matches the median training distribution |
| `enable_thinking` | `true` | Jibay models are trained thinking-first; disabling should be an explicit opt-out |
| `show_thinking` | `true` | Transparency by default; product surfaces that must hide it opt out explicitly |
| `show_summary` | `true` | Condensed reasoning is cheap to show and generally desirable |
| System prompt (if empty/missing) | `You are Jibay, developed by JibayAI. You are a helpful and safe assistant and have a natural and human-like tone.` | Guarantees the model always has a coherent identity frame, even if the calling code forgets to set one |
| Any invalid/out-of-enum `reasoning_effort` value | Falls back to `Medium` | Never let a malformed request silently produce an unpredictable prompt shape |
| Any non-boolean `enable_thinking`/`show_thinking`/`show_summary` | Falls back to that flag's documented default | Same resilience principle |
| Optional roles with no content | **Omitted entirely** | Never spend tokens on empty tags |

---

## Full Examples

### Example 1 — Minimal (system omitted, all defaults apply)

**Input messages:**
```json
[{"role": "user", "content": "سلام"}]
```

**Rendered prompt:**
```
<[role_system]>
You are Jibay, developed by JibayAI. You are a helpful and safe assistant and have a natural and human-like tone.
<[role_system_end]>

<[role_reasoning_effort]>
Reasoning Effort is Medium
<[role_reasoning_effort_end]>

<[role_user]>
سلام
<[role_user_end]>

<[role_assistant]>
```

### Example 2 — Full context stack

**Input (conceptual):**
- `system`: "You are a coding assistant."
- `safety_message`: "Never execute untrusted code."
- `notes`: `{"user":"parsa","tier":"pro"}`
- `environment`: "Android app v3.2, ChatSDK"
- `retrieval`: "Doc: llama.cpp slot reuse causes KV bleed if id_slot isn't pinned."
- `memory_context`: "User previously mentioned working with Qwen models."
- `reasoning_effort`: `"High"`

**Rendered prompt:**
```
<[role_system]>
You are a coding assistant.
<[role_safety]>
Never execute untrusted code.
<[role_safety_end]>
<[role_notes]>
{"user":"parsa","tier":"pro"}
<[role_notes_end]>
<[role_environment]>
Android app v3.2, ChatSDK
<[role_environment_end]>
<[role_retrieval]>
Doc: llama.cpp slot reuse causes KV bleed if id_slot isn't pinned.
<[role_retrieval_end]>
<[role_system_end]>

<[role_reasoning_effort]>
Reasoning Effort is High
<[role_reasoning_effort_end]>

<[role_memory]>
User previously mentioned working with Qwen models.
<[role_memory_end]>

<[role_user]>
چرا KV cache بین چت‌ها قاطی می‌شه؟
<[role_user_end]>

<[role_assistant]>
```

### Example 3 — Multi-turn with history sanitization

**Input messages (2nd assistant turn is the one being generated; 1st had reasoning):**
```json
[
  {"role": "user", "content": "۲+۲ چند میشه؟"},
  {"role": "assistant", "content": "۴", "reasoning": "جمع ساده، ۲+۲=۴"},
  {"role": "user", "content": "حالا ۴ برابرش کن"},
  {"role": "assistant", "content": "۱۶", "reasoning": "۴ * ۴ = ۱۶", "summary": "ضرب ساده"}
]
```

**Rendered prompt (note: first assistant's reasoning is stripped, only the last one keeps it):**
```
<[role_system]>
You are Jibay, developed by JibayAI. You are a helpful and safe assistant and have a natural and human-like tone.
<[role_system_end]>

<[role_reasoning_effort]>
Reasoning Effort is Medium
<[role_reasoning_effort_end]>

<[role_user]>
۲+۲ چند میشه؟
<[role_user_end]>
<[role_assistant]>
۴
<[role_assistant_end]>
<[role_user]>
حالا ۴ برابرش کن
<[role_user_end]>
<[role_assistant]>
<[reasoning]>
۴ * ۴ = ۱۶
<[summary]>
ضرب ساده
<[summary_end]>
<[reasoning_end]>
۱۶
<[role_assistant_end]>
```

### Example 4 — Thinking fully disabled

**Flags:** `enable_thinking = false`

**Rendered generation prompt tail:**
```
<[role_assistant]>
<[reasoning_end]>
```

The model receives an already-closed reasoning tag and proceeds directly to producing the final answer — it structurally cannot open a reasoning block after this point in the turn.

---

## Error Resilience

JCF's template is written to **never throw and never produce an undefined/garbage prompt shape**, even with malformed input:

- If `messages` is missing, `undefined`, or not iterable → treated as an empty list, and the template still renders a valid system + reasoning_effort block.
- If `reasoning_effort` is missing or not one of the 7 valid values → silently falls back to `Medium`.
- If `enable_thinking`, `show_thinking`, or `show_summary` are missing or not strictly boolean → silently fall back to their documented defaults.
- If `system` content is present but empty/whitespace → the default Jibay identity string is used instead of emitting an empty tag.
- If an assistant message has no `content` and no `tool_calls` → nothing is printed inside the assistant tag body (no `None`/`null` leaking into the prompt).
- If an optional context field (`safety_message`, `notes`, `environment`, `context`, `retrieval`, `memory_context`, `input_files`, `tools`) is absent → its tag is omitted entirely, never emitted empty.

This resilience is intentional: a template that crashes on a missing field takes down inference for every concurrent user on a shared server. JCF is designed for exactly the kind of multi-tenant, high-concurrency deployment where that failure mode is unacceptable.

---

## ⚠️ Critical Warning: Training Requirement

**JCF is not a drop-in prompt format for an arbitrary pretrained model.**

This cannot be overstated: **a model must be fine-tuned/trained on JCF-formatted data to use JCF safely and effectively in production.** JCF defines a specific token sequence, a specific nesting structure, and a specific history-sanitization behavior. A model that has never seen this structure during training has no learned prior for:

- What `<[role_reasoning_effort]>` means or how "High" vs "Low" should actually change its behavior.
- How to interpret nested `<[role_safety]>` / `<[role_notes]>` / `<[role_environment]>` / `<[role_context]>` / `<[role_retrieval]>` tags living *inside* the system block rather than as top-level turns.
- The expected relationship between `<[reasoning]>`, `<[summary]>`, and the final answer content.
- That historical assistant turns will never contain reasoning, and current-turn ones might.

**Consequences of swapping JCF onto a model trained on ChatML, Harmony, or any other format without retraining or at minimum extensive fine-tuning/adaptation:**

- Severe quality degradation — the model treats the tags as meaningless literal text tokens instead of structural signals, effectively adding noise to every prompt.
- Broken instruction-following — persona/safety instructions nested inside `<[role_system]>` may be ignored if the base model was never trained to look for sub-structure inside its system turn.
- Reasoning collapse or hallucinated reasoning — a model never trained with `<[reasoning]>`/`<[summary]>` boundaries may generate malformed, incomplete, or nonsensical content when the generation prompt opens a reasoning-adjacent context it doesn't recognize.
- Repetition and formatting corruption — similar in nature to the Persian-morphology corruption issues previously diagnosed in Jibay's own `run_model.py` sampler stack when a mismatch existed between what the serving stack expected and what the model was actually trained on.
- Unpredictable tool-calling behavior — a model not trained to expect `<[role_tools]>` / `<[role_tools_response]>` in these exact positions may fail to invoke tools correctly or hallucinate tool syntax.

**Do not treat JCF as a universal prompt template you can point at any checkpoint.** It is a training-time contract first, and a serving-time template second. Only models explicitly trained (or carefully adapted via continued fine-tuning) on JCF-formatted conversations should be served with `chat_template.jinja`.

**Some Jibay models have already been trained on this exact chat template** and are the reference implementation for correct JCF behavior. Any new base model considered for the Jibay stack should go through a dedicated JCF fine-tuning pass before this template is wired in as its default chat template.

---

## Integration Notes

- Designed for `llama-server` / `llama.cpp` / `ik_llama.cpp` deployment via the `--jinja` flag, using this file as the model's `chat_template`.
- Because the system frame's shape never varies (fixed ordering, no conditional reshuffling), prefix caching (`--slot-prompt-similarity`, KV cache reuse) behaves predictably turn-to-turn — a direct latency win on CPU-only serving.
- Backends should pass `reasoning`/`summary` as explicit fields on assistant message objects (not embedded in `content`) so the template's history-sanitization logic can operate on them independently of the visible answer text.
- Recommended to validate `reasoning_effort` server-side as well (not only template-side) so invalid values are caught and logged before reaching the model, rather than silently downgraded to `Medium` without visibility.

---

## FAQ

**Q: Why not just use Harmony's channel system for memory/retrieval/notes?**
A: Harmony's channels (`analysis`, `commentary`, `final`) are designed around reasoning-visibility routing, not around typed context injection. Repurposing them for memory/retrieval/notes conflates two unrelated concerns and still leaves you writing custom convention on top of Harmony to distinguish "this is a memory snippet" from "this is a retrieval snippet" — which is exactly the ambiguity JCF eliminates with dedicated tags.

**Q: Doesn't having 13 possible tags increase token overhead compared to ChatML's 3 roles?**
A: No — because unused tags are never emitted. A simple single-turn chat with no memory/retrieval/tools renders with the same two structural tags (system, reasoning_effort) plus user/assistant, comparable to ChatML's overhead. The extra roles only cost tokens when you actually use the corresponding feature, which is when you'd be paying for that context regardless of format.

**Q: Can I mix reasoning_effort levels within one conversation?**
A: Yes — it is evaluated per-render, so each turn can request a different effort level based on live product logic (e.g. escalate to `High` after a user says "explain in more detail").

**Q: What happens if I set `show_thinking=false` but `enable_thinking=true`?**
A: The model still reasons internally (server-side), but the reasoning block is omitted from what's rendered back into the transcript for that turn — useful for hiding chain-of-thought from end users while still benefiting from it internally, and while still logging it separately server-side if desired.

By JibayAI – MIT – jibay.ir
