# Defects on our production instance, and the patches we run to work around them

**From:** Adega Gaucha — self-hosted OSS
**Image:** `dograhai/dograh-api:main-1e47eb54`
**Date:** 2026-09-29

One Dograh instance answering a restaurant line, with an English agent and a Portuguese agent that
hand calls to each other via `transfer_agent`. Everything below was found on **real phone calls
through Twilio**, placed by us over one afternoon — the line is provisioned and answering but not
yet open to customers, which is why we could reproduce so freely.

To keep those calls working we patch your Python **inside the running container**. Every image
update reverts us silently, and the defect comes back looking like a new bug. That is the only
reason this document exists: we would rather delete our patches than maintain them.

| # | defect | status | evidence |
|---|---|---|---|
| 1 | Handoff context carries no provenance | **patched** | 2 failure modes, real calls |
| 2 | A refused transfer consumes the turn | not patched | 13 s of dead air |
| 3 | `retire()` races the destination's first generation | **patched** | 65 ms vs 2.056 s, every call |
| 4 | Greeting cut by the echo of the agent's own voice | **patched** | 4 of 13 calls |
| 5 | `turn_start_strategy` inert with external-turn STT | not patched | in every call's log |
| 6 | `max_call_duration` cuts with no goodbye | not patched | present in this image |
| 7 | No way to brief the person who answers a transfer | **feature request** | the hook is already in your TwiML |

Three diffs accompany this document and apply with `git apply` from the repository root. Happy to
open PRs.

---

## 1. Handoff context carries no provenance — the destination inherits the source's turns as its own

**Patched. Severity: high — it broke the call in two different ways in one afternoon.**

**Where:** `api/services/workflow/agent_handoff_context.py`, `build_handoff_snapshot()` /
`conversation_messages()` / `_carries_tool_traffic()`.

Every message handed to the destination agent arrives as a plain `user`/`assistant` turn with
nothing marking that it was addressed to a *different* agent. `_carries_tool_traffic()` filters
only `tool`/`function`/`system` roles and `tool_calls`. So the destination reads the source's
conversation as if it were spoken to itself, and acts on it.

### 1a. The destination completes the source's transfer announcement → language loop

`AgentTransfer._announce()` calls `queue_speech(request.announcement, append_to_context=True, ...)`,
so the transfer tool's configured `message` lands in the context as an assistant turn. The
destination reads *"One moment, I'll connect you with our Portuguese-speaking line."* as a promise
**it** made and has not kept — and keeps it, by calling the transfer tool back.

Real call, 30 s. **The caller spoke once.**

```text
15:25:03 assistant: Thank you for calling ... This is Claudia, our virtual assistant ...
15:25:05 user:      Gostaria de falar com o atendente português.      <- the only caller turn
15:25:08 assistant: One moment, I'll connect you with our Portuguese-speaking line.
15:25:14 assistant: Um momento, vou te passar para o nosso atendimento em inglês.
15:25:20 assistant: One moment, I'll connect you with our Portuguese-speaking line.
                    ... call dropped
```

`agent_transfers` = 3 (`completed`, `completed`, `user_hangup`), `nodes_visited` = `["Greeting"]`.

### 1b. The destination applies its own language rule to the caller's inherited turn → mute agent

Subtler, and more common: the caller asks to switch **in the language they are leaving**, which is
the natural way to ask. The destination inherits that sentence as fresh input.

```text
16:24:57 user:      Você consegue me transferir para atendente inglês?   (Portuguese)
16:25:01 assistant: Só um momento, por favor.
16:25:04 [transfer] installed visit-3b73149e5e92: 6 handoff messages (summarized=False) + 0 live messages
16:25:05 Transfer Agent Tool EXECUTED: adega_para_agente_portugues
16:25:05 Transfer via 'adega_para_agente_portugues' refused: transfer_in_progress
16:25:14 user:      Olá.        <- 13 s of hold audio; the agent never spoke
```

The English agent took over, saw a whole Portuguese sentence as the last turn, applied its own "a
Portuguese sentence hands off immediately" instruction, and called transfer back. Your
`transfer_in_progress` guard correctly refused it — so no loop — but the turn was spent on a failed
tool call and no speech came out. See defect 2.

**This is not fixable by prompting.** Our node prompt already states, explicitly and with a worked
example, that the language rule applies only to turns addressed to the current agent. The model
cannot comply, because nothing in the context distinguishes inherited turns.

### Our patch — `01-handoff-boundary.patch`

A single `role: "system"` boundary marker appended to the **end** of the handoff messages, in both
return paths of `build_handoff_snapshot()`. The end is deliberate: it is the last thing read before
generation, and it occupies the position where the inherited foreign-language turn used to sit.
Validated on real calls in both directions — no loop, no silence.

**It is a workaround, not the fix we would choose.** The structural fix is for the snapshot to
carry **who said what to whom** — tag inherited messages with their source visit, or render them as
a transcript block rather than as live `user`/`assistant` turns. A cheaper independent improvement:
`_announce()` should not put the announcement into the inherited context at all. It is the source
agent's speech about a mechanism, not conversation content.

---

## 2. A refused transfer consumes the turn and the agent goes mute

**Not patched — we have no fix from outside. Severity: medium.** Same evidence as 1b.

When `transfer_in_progress`, or any other guard, refuses a transfer tool call, the turn ends with
no speech. From the caller's side that is indistinguishable from a dropped call: 13 seconds of
nothing until they spoke again.

A refusal is a recoverable condition and the engine already knows the tool did not run. It should
let the agent produce speech instead of ending the turn silently. Fixing this alone would have
contained the damage of defect 1.

---

## 3. `source.retire()` races the destination's first generation, losing its opening speech

**Patched. The race was firing on every handoff. Severity: medium.**

**Where:** `api/services/workflow/agent_transfer.py` — `queue_node_opening()` at line 187,
`retire()` at line 217.

`queue_node_opening()` only *queues* the destination's first generation. The `finally` block then
runs `source.retire("transferred")`, which drains the old worker on the same shared pipeline. If
the drain catches that generation mid-flight **and** that generation is a tool call, the following
generation — the actual speech — is lost, and the agent stays mute until the caller speaks.

The asymmetry is reproducible here, measured from `installed visit` to `xfer completed`:

| call | English → Portuguese | Portuguese → English |
|---|---|---|
| 1373 | 65 ms | **2.056 s** |
| 1374 | 66 ms | **2.056 s** |
| 1375 | 60 ms | — |

On the fast leg `retire()` finishes before the generation exists and nothing is lost. On the slow
leg it lands in the middle. We have one call where the destination's first action was a tool call
on the *fast* leg and the speech survived; we have not yet caught the slow leg in that state, but
the window is two seconds wide.

`03-retire-before-opening.patch` moves `await source.retire("transferred")` to **before**
`queue_node_opening()`. `retire()` is idempotent, so the `finally` block stays correct — it simply
awaits the same task. Cost: up to ~2 s more hold audio on the switch, a good trade for not going
mute. Applied here after the measurements above.

---

## 4. The greeting is cut by the echo of the agent's own voice — the 2-word guard is too low

**Patched. Severity: high — 4 of 13 calls (31%) in one afternoon.**
**This is the one where you already have the right mechanism and only the number is wrong.**

**Where:** `api/services/pipecat/greeting.py`, `GreetingController._update_strategies()`.

While a greeting plays you swap in `MinWordsUserTurnStartStrategy(min_words=2)`. That is exactly
the right idea. But two words do not survive the echo of the greeting itself. Every affected call
has a "caller" turn timestamped within milliseconds of the bot's own line, containing **words from
the greeting**:

| run | transcribed as the caller | words | greeting |
|---|---|---|---|
| 1386 | `Thank` | 1 | cut at "Kissimmee." |
| 1382 | `Thank you.` | 2 | cut |
| 1381 | `Adega, gotcha, Cassini?` | 3 | cut — "Adega Gaucha Kissimmee" misheard |
| 1360 | `Thank you for calling.` | 4 | cut |

After the cut the agent answers itself — `assistant: "You're welcome! Is there anything else I can
help you with today?"` — three seconds into the call.

**`04-greeting-echo.patch` raises the number from 2 to 40. Please do not merge that number.** It is
a stopgap for one deployment. What we would like is for the greeting threshold to be
**configurable per workflow**, with a default high enough to survive an echo of your own greeting.

Raising it is safe, and your own code is why: when the greeting finishes you already restore the
normal strategies and hand them any pending speech onset — *"Let the restored strategy adopt that
onset so it can finish the turn on the final transcript."* A caller who talks over the greeting is
not lost; their turn simply starts when the greeting ends. Both behaviours confirmed on real calls
after the change.

**Two things we tried first, in case it saves you time:** a prompt instruction cannot do this at
all — the cut happens in the audio pipeline, before the model decides anything. And
`allow_interrupt: False` on the node does not cover the greeting either: `pipecat_engine.py` has
`if self.speech_playback.greeting_pending or self.greeting.awaiting_turn: return False` *before*
the `allow_interrupt` check.

---

## 5. `turn_start_strategy` is inert whenever the STT reports its own turns

**Not patched. This is a request, not a bug report — your code says the behaviour is deliberate.**

**Where:** `api/services/pipecat/run_pipeline.py`,
`_create_non_realtime_user_turn_start_strategies()`. The function checks `uses_external_turns`
**before** reading `turn_start_strategy`, with the comment *"An STT that reports its own turn
boundaries decides the turn start, whatever `turn_start_strategy` asks for."*

Every call here logs `requested=default resolved=ExternalUserTurnStartStrategy
uses_external_turns=True` with `flux-general-multi`. We have `turn_start_min_words: 3` configured
on the workflow and it has no effect, by design.

We understand the intent. The problem is that it leaves **no configuration at all** to guard
against a false turn start from the STT — which is exactly what defect 4 is, outside the greeting
window. Inside the greeting you already solved it with a word threshold; outside it there is
nothing.

**What we would like:** a way to keep the external-turn path *and* require a minimum of real words
before a turn opens — even an opt-in flag.

---

## 6. `max_call_duration` cuts the call with no spoken goodbye

**Not patched. Present in this image; we have not yet let a call here run to the limit.**

**Where:** `api/services/workflow/pipecat_engine_callbacks.py`, `create_max_duration_callback` —
`handle_max_duration()` calls `engine.end_call_with_reason(EndTaskReason.CALL_DURATION_EXCEEDED…)`.
The audio stops mid-call; from the caller's side it is indistinguishable from a dropped line.

We once patched this elsewhere to speak a goodbye first, using what the engine already has:
`queue_text_message()` between `arm_speech_playback()` and `wait_for_speech_playback()`, so the
sentence completes before the transport drops. A configurable closing line on the workflow would
cover it properly.

---

## 7. Let the transfer tell the person who answers what the call is about

**This one is not a defect. It is a feature request, and a small one, because the
hook is already in your code and so is the machinery.**

When `transfer_call` dials out, the TwiML you send already speaks to the person who
answers, privately, before they join the conference:

```xml
<Response>
    <Say>You have answered a transfer call. Connecting you now.</Say>
    <Dial>
        <Conference endConferenceOnExit="true">{conference_name}</Conference>
    </Dial>
</Response>
```

That `<Say>` is a whisper slot in everything but name. Today it carries a fixed
sentence that tells the answerer nothing.

For us the call is transferred to a restaurant, into a FreePBX ring group. Whoever
picks up gets a live caller with no idea why they are on the line — the caller then
repeats everything they already told the agent. A single sentence would fix that:
*"Transfer from the virtual assistant. The caller is asking about a birthday booking
for twelve people on Saturday."*

Everything needed exists on your side:

- the `<Say>` slot, in `telephony/providers/twilio/provider.py`;
- one call site, `pipecat_engine_custom_tools.py`, where the transfer is placed;
- the summarisation path you already use for agent handoffs (`run_inference` with a
  system instruction, the same one behind `HANDOFF_SUMMARY_PROMPT`).

**What we would ask for:** an optional per-tool field — a whisper prompt, or simply a
boolean that turns on "summarise the call for the answering party" — plus a language
attribute on the `<Say>`, since without `language="pt-BR"` a Portuguese sentence comes
out read by an English voice.

**We built it and it works.** On a real call the caller asked, in Portuguese, for a
Portuguese agent; was handed over; and only then asked for a table for thirty. The
whisper the restaurant heard was *"Cliente pediu para falar com um atendente em
português e deseja fazer uma reserva para trinta pessoas"* — sixteen words joining
both halves of a conversation that had spanned two agents. Generating it cost
**1.1 seconds** before the dial, well inside our 4-second cap. `destination_answered`,
`status=success`.

**Three things we found while prototyping it, in case they save you time:**

1. **The text has to be XML-escaped.** It comes from the conversation, so an `&` or a
   `<` in what the caller said produces malformed TwiML and Twilio rejects the call.
   The transfer would fail because of a stray character in someone's sentence.
2. **It has to be time-boxed with a fallback.** Generating the summary happens *before*
   the dial, so a slow inference is dead air added to a caller who is already on hold.
   Ours caps at 4 seconds and falls back to a fixed sentence — a summary is a nicety,
   the transfer is not.
3. **Word count is a latency budget, not a style choice.** The caller waits through the
   whisper. We cap at about 25 words, roughly eight seconds.

Also worth noting for anyone who builds this: the whisper is heard by whoever answers,
which in a ring group is not a known individual. It will speak the caller's business
out loud to whoever picks up. That is the point, but it deserves to be a deliberate
setting rather than always-on.

## 8. Template variables have no way to take part of a value, so the agent guesses the digits

`{{initial_context.caller_number}}` can only be rendered whole. The only filter is the
default one, `{{field | fallback}}`. There is no slice, no `last4`, no formatter.

We need the last four digits, for one line the agent says near the end of every booking:
*"the confirmation goes by text to the number you're calling from, ending X X X X — or a
different number?"* That line exists for a concrete reason: a landline cannot receive a
text, and someone calling from a hotel room or an office otherwise leaves the call with
nothing and no way of knowing.

Since the template cannot cut the value, the prompt asked the model to do it. It does not.
Measured on our test instance, gpt-4.1, five repetitions per cell:

| caller number | correct last four | what the agent said |
|---|---|---|
| +1 555 123-**4567** | 4567 | "five, five, five, five" (4/5), "five, five, six, seven" (1/5) |
| +1 305 555-**9999** | 9999 | "eight, eight, nine, one" |

Two things are worth separating, because only the second is yours.

**The first was ours.** The prompt carried a worked example with literal digits — "ending
eight, eight, nine, one" — three times in what the model reads, against one mention of the
variable. It copied the example. That is a prompt-authoring trap rather than a platform
defect, and we removed the digits from every example. Worth a line in the docs, though:
**a worked example containing something that looks like data will be read as data.**

**The second is the platform.** With no example at all, and only the variable, it still
reads the wrong four — it receives `+15551234567` whole and picks four digits out of it.
No wording fixes that. We have taken the digits out of the sentence entirely; the question
survives, the courtesy does not.

What would fix it, smallest first:

- a slice or `last4` filter on template variables, e.g. `{{caller_number | last4}}`;
- or a built-in `{{caller_number_last4}}` alongside the existing `{{caller_number}}`;
- or, more generally, a way to derive a context value before it reaches the prompt.

The same failure mode has a worse version we fixed by construction rather than by
measurement, because measuring it means creating a real booking: step 9 of our flow carried
*"Your confirmation number is one, two, three, four, five, six"* as an example. A model that
copies "eight, eight, nine, one" will copy that too, and the caller writes down a
confirmation number that does not exist.

## Other things we work around, on this same instance

- **Knowledge-base retrieval is a per-node tool, and the model skips it about half the time.**
  Measured over 16 long-tail questions, fresh session each, gpt-4.1: 41 % / 31 % / 47 % retrieval
  depending on prompt variant — prompt reinforcement made no measurable difference (the same
  variant swings 31–41 % between days). The failures are deterministic: the model never looks up
  the questions where a generic steakhouse answer sounds plausible, and almost every unlooked-up
  answer was wrong — invented addresses, invented prices. We ended up putting the knowledge base
  into the global prompt (16/16) or forcing `tool_choice` through a proxy on `base_url`
  (41 % → 100 %). **A pipeline-stage retrieval option would remove the need for both.**
- **No way to influence `tool_choice`.** It exists only in the realtime wrappers, hardcoded to
  `"none"`, with tests pinning that value. In pipeline mode it is never sent. The only route in is
  `base_url`.
- **The HTTP tool only ever sends JSON.** The executor calls `httpx` with `json=body`; setting
  `Content-Type: application/x-www-form-urlencoded` in the headers does not change the body. Twilio
  requires form encoding, so we run a bridge service. Proven both ways: JSON returns Twilio error
  21604, form returns 21211.
- **The HTTP tool reports `status: success` over any HTTP response, including 400.**
- **Text-chat mode does not support `transfer_agent`.** The tool is called, the agent replies *"I
  can't connect you to a Portuguese-speaking agent right now"*, and `agent_transfers` stays empty.
  That makes every defect above **untestable without spending a real phone call**, which is most of
  why they took so long to characterise. Supporting transfers in text chat would be the single
  biggest quality-of-life improvement for anyone building multi-agent flows.
- **An agent handoff costs about 12 seconds of hold audio.** Measured on five handoffs, from the
  end of the announcement to the destination's first word: 11.5, 12.1, 13.7, 12.1 and 12.7 s.
  Roughly 3 s from tool execution to `installed visit`, then the destination's first generation
  against a 27,000–32,000 character global prompt. Not silence — but long enough that our own
  testers assumed something had broken.

## Operational papercuts

- **`GET /api/v1/health` answers while the API is down**, because the request stops at the UI
  proxy. Probing it after a restart gives a false "ready" and the next call returns `502 ... fetch
  failed while proxying to http://api:8000`. Only `docker inspect <container> --format
  '{{.State.Health.Status}}'` is trustworthy.
- **The version string does not identify the running image.** `/health` reports a stale version,
  and the git release text under `/opt/dograh` is whatever was last pulled — ours says "1.35.0"
  while running `main-1e47eb54`. A build identifier in `/health` would remove a real source of
  misdiagnosis: we wasted time assuming this instance was older than another, when the API surface
  proved the opposite.
- **`GET /api/v1/tools` returns 307** — the working path is `/tools/` with a trailing slash.
- **API keys come back masked and are refused on write** ("The api_key appears to be masked"). The
  server accepts a masked value only when it can match it to the key it already holds *for that
  provider*, so copying a workflow configuration between instances fails as soon as the provider
  changes. A write-only key field, or an explicit "leave unchanged", would make configuration
  portable.
- **The MinIO bucket is created with anonymous read, write and delete.** The `voice-audio` bucket
  policy grants `s3:GetObject`, `s3:PutObject` and `s3:DeleteObject` to `"AWS": ["*"]`, and nginx
  exposes it at `location /voice-audio/`, so call recordings and transcripts are downloadable — and
  deletable — by anyone. We would like to know whether this is the default in the OSS deployment,
  because it is not visible from the outside: `mc anonymous list` reports `none`, and only
  `mc anonymous get-json` shows the real policy.
