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
| 8 | Template variables cannot take part of a value | **feature request** | the agent guesses the caller's digits |
| 9 | `play_greeting` is all or nothing | **patched** | 0 of 2 vs 9 of 9 real calls |
| 10 | The STT is never retuned on a transfer | **patched** | a call ended on a mis-transcribed "Sim" |
| 11 | The introduction never runs for speech-to-speech agents | **feature request** | enabled and silently skipped, 2 languages measured |
| 12 | A realtime greeting never registers output, so it can only end by interruption | **patched** | `first_output_at=None` in 14 of 14 calls |
| 13 | The transfer leg is placed from the account's own number, so the destination never learns who is calling | **patched** | `From` == `To` on every transfer, 3 days |
| 14 | Nothing records the transferred part of a call, and the run's own recording stops at the transfer | **patched** | 8 of 8 runs: 38 s recorded of a 250 s call |
| 15 | A realtime provider's cumulative transcription is published once per partial, so the caller's sentence is repeated in the transcript | **patched** | 8 events for one spoken sentence |

Eleven diffs accompany this document and apply with `git apply` from the repository root. Happy to
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

### Update 2026-10-05 — you shipped it, and it works. One adjustment.

`introduction_enabled` / `introduction_prompt` (#843) landed in 1.48.0 and it is better than
what we had prototyped: the text follows the caller's language instead of being pinned to
`pt-BR`, and the audio comes out in the agent's own voice through `create_tts_service`, not the
default `<Say>` voice. The 25-word and 300-character validation, the two attempts and the
"failures must never block the transfer" guarantee all match what we had found the hard way.
Thank you.

The one thing we changed is **who hears it**. `introduction_urls` is attached twice:

| call site | leg | wanted? |
|---|---|---|
| `provider.py`, outbound `Calls.json` | the destination | yes |
| `strategies.py`, inside `TwilioConferenceStrategy` | the **caller's** existing leg | no |

The summary is for whoever answers. The caller just said all of it, so hearing it read back is
only hold time before they are connected. `item-7-introduction-destination-only.patch` drops the
second attachment; the first is untouched.

**On the concern that the caller would not be able to join the conference.** We measured it on
two real calls, 1.48.0 with the patch, Twilio, destination a FreePBX ring group:

```
transfer_introduction.py:158 | Transfer introduction ready words=19 audio_bytes=88960   (prepared in 1.8s)
transfer_introduction.py:158 | Transfer introduction ready words=17 audio_bytes=96170   (prepared in 1.5s)
```

Both dialled, both answered, `destination_answered` / `status=success`, and the caller joined the
conference normally. Confirmed by ear on our side: the summary was heard on one leg only, the
right one, and the caller heard hold for those couple of seconds.

That is consistent with the comment already in `process_introduction_status` — *"During Play the
recipient hasn't joined the conference yet, so `endConferenceOnExit` cannot remove the waiting
caller"* — which describes a caller already waiting in the conference while the destination is
still playing. And the caller path the patch restores is the one that already runs whenever
`introduction_enabled` is false.

**What we would ask for instead of this patch:** `introduction_audience: both | destination`,
defaulting to `both` so nothing changes for existing users. Happy to open the PR.

**This shipped feature turned out to be unreachable for speech-to-speech agents.** We moved this line to a realtime model and the introduction stopped existing, silently. That is **item 11**, with the measurements and a patch.

**One loose end we have not investigated.** On the second call the `transfer-status` callback
published `TRANSFER_FAILED` 24 seconds *after* the destination had already answered — the
`completed` callback arriving at the end of the call, and `process_introduction_status` treats
`completed` as a cancellation, including `end_transfer_leg(original_call_sid)`. Harmless for us
because the call had ended anyway, but on a long call it looks like it could tear down a transfer
that is already up.

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

## 9. `play_greeting` is all or nothing: the destination either introduces itself or continues, never both

**Where:** `api/services/workflow/agent_transfer.py` — `queue_node_opening()` at line 193;
`api/services/workflow/pipecat_engine.py` — `queue_node_opening()`, the greeting branch returning
at line 1101 and the generation branch at line 1103.

`play_greeting` decides which of two things the destination does, and there is no way to ask for
both:

- **true** — `queue_node_opening` plays the Start Call greeting and **returns before the LLM runs**.
  The destination introduces itself and then waits for the caller to speak again. The caller has
  just been transferred and already said what they wanted; now they have to say it twice.
- **false** — the opening is an LLM generation, and the introduction depends on the model
  remembering to produce it. **Whenever that generation comes out as a tool call, the opening turn
  is consumed by the tool and the introduction is lost.**

We hit the second one on a real line. Our English agent hands over to a Portuguese agent; the
Portuguese prompt orders the introduction in four separate places, one of them in position zero of
all three entry nodes. Read off the traces of eleven real calls:

| first Portuguese turn | introduced itself |
|---|---|
| **needed a knowledge-base lookup** | **0 of 2** |
| did not | **9 of 9** |

The prompt was in context — 60,293 characters, the rule present — and the model skipped it, because
the opening turn had already been spent on `retrieve_from_knowledge_base`. This is not a prompting
problem: a question that needs a lookup is the single most common way a transferred call starts.

**What we would ask for:** a way to have the destination say a fixed line **and** carry on. We added
an optional `transfer_greeting` to the transfer tool — a string spoken by the destination, in its
own voice, before its first generated turn, used only when `play_greeting` is false. The node's own
Start Call greeting stays for direct entry; this is the sentence for the handover, which is a
different sentence (ours ends with "and I will continue your call in Portuguese", which makes no
sense when nobody was transferred).

**It works.** On real calls: introduction present on every handover since, exactly once, and the
answer still arrives in the same breath. The gap between the end of the fixed line and the start of
the answer went from 9.4 s to 7.4–7.9 s once we stopped blocking on playback (below).

**Two things we found while building it, in case they save you time. Both cost us a deploy and a
rollback.**

1. **`tracker.greeting` is a single slot, and the second writer silently wins.**
   `SpeechPlaybackTracker.create()` does `self.greeting = speech` for any speech flagged
   `greeting=True` (`api/services/pipecat/speech_playback.py:157-158`). The generation branch of
   `queue_node_opening` creates its own greeting-flagged speech, so ours is overwritten; then
   `if self is self._owner.greeting` (`speech_playback.py:111`) is false, `on_greeting_finished`
   never fires, and the text **never reaches the assistant context**
   (`api/services/pipecat/greeting.py:131-132`). The agent has spoken a sentence it does not know it
   spoke. Nothing logs this. We avoid it by not passing `greeting=True` at all — `mute_user` already
   blocks interruption, and it is checked **before** the greeting exception in `should_mute_user`
   (`pipecat_engine.py:1546-1551`).
   Worth noting that `greeting=True` also silently overrides the caller's arguments —
   `append_to_context` to `False` and `persist_to_logs` to `True` (`pipecat_engine.py:1462-1464`).
   We passed `append_to_context=True` for two days and it did nothing.

2. **Order is not guaranteed between `queue_speech` and `run_llm`.** `queue_speech` enqueues at the
   head of the agent worker, so the `TTSSpeakFrame` must traverse the LLM processor to reach the
   TTS; `run_llm` queues **straight onto the LLM** (`agent_runtime.py:160-167`). Whichever reaches
   the LLM's process queue first wins, and the shortcut belongs to the LLM — so dropping the wait
   altogether can play the answer **before** the greeting. Waiting for the speech to finish is
   correct but costs the silence. The right synchronisation point is `speech.started`, set when the
   opening boundary drains at the transport (`speech_playback.py:318-324`): by then the frame has
   passed the LLM and anything queued after lands behind it. There is no public future for it, so we
   poll with a ceiling.

**The patch:** `item-9-destination-greeting-and-continuation.patch`. It touches
`agent_transfer.py`, `pipecat_engine_custom_tools.py` and the `TransferAgentConfig` schema. We are
not attached to the shape — a third value on `play_greeting`, or a `greeting_mode`, would do the
same job. What we need is the combination.

## 10. The STT is created once per call, so a transferred call keeps listening with the source agent's transcriber

**Where:** `api/services/pipecat/agent_runtime_factory.py` — `build()` creates an `llm` at line 171
and a `tts` at line 172, and then only *reports* the STT at lines 224-225;
`api/services/pipecat/run_pipeline.py:761` — `create_stt_service()`, called once, whose result goes
into `build_pipeline(transport, stt, ...)`.

A transfer rebuilds the per-agent services and attaches a new generation pipeline
(`build_agent_generation_pipeline(llm, tts, ...)`). The STT is not one of them: it belongs to the
*call* pipeline and is never rebuilt, retuned or even consulted again. The consequence:

**the destination workflow's `stt.language` and `stt.language_hints` are dead configuration on any
transferred call.**

Two further effects worth naming:

- `runtime_configuration` records `stt_provider` and `stt_model` for the destination
  (`agent_runtime_factory.py:224-225`), so the run reports an STT that is **not** the one running.
- The TTS *does* switch, so the asymmetry is invisible from the outside: the destination speaks in
  its own voice while listening with somebody else's ears.

### What it did on a real call

Our English agent hands to a Portuguese agent. Both run `flux-general-multi`; the English one with
`language_hint=en`, the Portuguese one with `pt` and `en`. After the handoff the call kept the
English hint, and the damage fell entirely on **short** utterances — a full Portuguese sentence
survived, a one-word answer did not:

| what the caller said (Portuguese) | what the transcript got | length |
|---|---|---|
| "Qual tipo de restaurante?" | `Qual tipo de restaurante?` | 256 ms |
| "Sim, pode mandar." | `Sim, pode mandar.` | 256 ms |
| "você tem vinhos" | `você tem vinhos,` | 399 ms |
| **"Sim"** | **`See.`** | 256 ms |
| **"Sim"** (answering "anything else?") | **`Thank`** | **95 ms** |

That last one ended the call: `Thank` read as a goodbye, the agent called `move_to_goodbye()`, and
the caller — who had just said *yes, help me more* — heard the closing line and was hung up on.

**Isolated with the recording.** Same `user.wav`, same window, same thresholds, only the hint
changed:

| `language_hint` | what Flux returned for that "Sim" |
|---|---|
| none | `Sim` |
| **`en` only** (what was running) | **`See`** |
| `en` + `pt` | `Sim` |
| `pt` only | `Sim` |

And the reverse holds on the English leg, from the recording of a different call: with `pt` in
play, English audio comes back as `Lata` and `O Atomix` in the eager hypotheses — the final
transcript is correct, but the agent acts on the eager one. So **there is no single hint list that
serves both legs**, which is why this cannot be configured around.

### THE TRAP: do not fix it by reconnecting

Our first patch set `_settings.language_hints` and then called `_disconnect()` / `_connect()`. The
log said the socket reconnected in 99-210 ms and reported `Connected to Flux - ready to stream
audio`. **Transcription was dead for the rest of the call.** Three real calls, and the number of
usable caller turns after the handoff:

| | caller turns after handoff |
|---|---|
| before the patch (shared transcriber) | 7 |
| with the reconnect patch, 3 calls | **0, 0, 1** |

On the third the caller spoke and the agent asked *"are you still there?"* 22 seconds later. The
cause is most likely the hazard your own comment describes at
`pipecat/services/deepgram/flux/stt.py:195-203`: `_receive_task` is created inside
`_connect_websocket`, and a reconnect raised from the receive handler can leave the first
`_receive_task` alive. Our log shows exactly that race — `_disconnect()` closed the socket with an
audio frame in flight, `send_with_retry` reconnected on its own, and our `_connect()` then hit the
already-open guard.

**No reconnect is needed.** `language_hints` is in `_CONFIGURE_FIELDS`
(`flux/stt_base.py:187`), so `_update_settings` sends `{"type": "Configure", "language_hints":
["pt"]}` on the open socket — which is what the field's own docstring says
(*"Can be updated mid-stream via STTUpdateSettingsFrame"*, against `numerals`/`redact`, which are
*"read only from the connection URL"*). One line:

```python
await stt._update_settings(type(settings)(language_hints=hints))
```

`apply_update` only touches *given* fields, so a delta carrying just `language_hints` leaves
`keyterm` and the end-of-turn thresholds alone — we verified that with the real `Settings` class,
because losing `keyterm` mid-call would cost us the dish names.

**It works.** One real call so far on the fixed version: the hints changed with no reconnect, and
the first Portuguese turn after the handoff came back whole — *"Para o dia vinte e nove."* — where
the reconnect version had produced nothing. One call is not a sample; we will say so if it turns
out otherwise.

**What we would ask for:** retune or rebuild the STT as part of the handoff, next to the `llm` and
`tts` that `build()` already creates. Our patch does it from the factory because that is where the
destination's `user_config` already is, but the shape is yours to choose — what matters is that a
destination's STT settings stop being inert.

**The patch:** `item-10-stt-not-retuned-on-transfer.patch`. It touches `agent_runtime_factory.py`,
`run_pipeline.py` and `agent_transfer.py`, and it applies **on top of item 9** (both edit
`agent_transfer.py`). The composition of the hint list is copied from `create_stt_service`; one
shared helper there would be better than two copies kept in step by hand.

**Not visible from text chat**, for the same reason as the rest of this document: `transfer_agent`
never runs there, and text chat has no STT at all.

## 11. The transfer introduction never runs for speech-to-speech agents

**Feature request, and a small one: the hook is the same `<Say>` as item 7.**

We moved this line to **Grok Voice 3.0** (realtime) and the introduction stopped existing. Not
degraded — gone, and silently:

```
transfer_introduction.py | Transfer introduction skipped reason=realtime_agent
```

`introduction_enabled` is `true`, the transfer succeeds, and nothing tells the operator that the
feature they switched on produces nothing. The UI shows it enabled. Only a container log line,
at INFO, says otherwise.

**The guard is sound; its conclusion is too broad.** The early return exists because
`prepare_transfer_introduction` ends in `create_tts_service(agent.user_config, ...)`, and a
speech-to-speech model has no separate TTS stage. We confirmed the premise in your registry:

```python
get_service_configuration(ServiceType.TTS, ServiceProviders.GROK_REALTIME)  # -> None
```

So the real requirement is **a TTS**, not **not realtime**. Two things follow, and we measured
both:

1. **A realtime configuration can carry a `tts` block.** `ModelConfigurationSpec.required_services`
   only *requires* `realtime` in realtime mode; `tts` stays optional and validates fine. Your API
   accepted the PATCH.
2. **But it is then discarded.** `_active_roles(mode)` returns `{"llm", "embeddings", "realtime"}`
   for realtime, and `if role in active` drops `tts` from the executable services — so
   `user_config.tts` arrives `None` even though the operator configured one. Relaxing only the
   early return is not enough; we tried, and got
   `skipped reason=realtime_agent_without_tts` with a TTS plainly configured.

**What we would ask for.** Not the TTS path — the `<Say>` one, the slot this item was originally
about. When there is no clip, put the generated text in the destination's TwiML instead of the
fixed sentence. No TTS provider, no synthesis, no Redis storage, no second HTTP fetch, and it
works for every speech-to-speech provider at once.

We built it and ran it on real Twilio calls against `main-35b38849`:

```
transfer_introduction.py | Whisper introduction ready words=15 language=pt-BR   (2.02 s)
transfer_introduction.py | Whisper introduction ready words=17 language=en-US   (1.58 s)
provider.py              | Twilio transfer API response status: 201
```

A human answered and heard it. Three notes, in case they save you time:

- **The `<Say>` needs a `language` attribute**, and the summary follows the caller, so the
  language has to come back with the text. We ask the model for a two-letter prefix (`pt|`,
  `en|`, `es|`) in the same response rather than spending a second round trip before the dial,
  and fall back to the default on anything unrecognised.
- **Escape the text.** It comes from the conversation: one `&` or `<` in what the caller said
  makes the TwiML invalid and Twilio **rejects the call**. We check two things, not one — that
  the TwiML parses *and* that the spoken text comes back identical, because over-escaping would
  have Twilio read "ampersand" out loud. 6/6.
- **Generate it inside the hold-audio window.** Our first attempt produced the text after the
  `finally` that stops the hold loop, which turned ~2 s of preparation into silence for a caller
  who is already waiting.

**It stays additive.** Pipeline agents keep the native clip in the agent's own voice; the `<Say>`
only fills in when there is no clip. Proved by contrast on two workflows on the same image — the
pipeline one logged `Transfer introduction ready ... audio_bytes=101164`, the realtime one logged
`Whisper introduction ready ... language=pt-BR`.

And it makes `introduction_audience` moot for this path: the `<Say>` lives only in the
destination's TwiML, so the caller never hears it and `strategies.py` is not involved at all.

Happy to open the PR — it is three files and no new dependency.

## 12. A realtime greeting never registers output, so it can only end by an interruption

**Patched, in two lines. This one breaks a documented feature, not a nice-to-have.**

`Allow Greeting Interruption` says, in its own help text:

> *"Let the caller interrupt the opening greeting. When off, caller speech is retained until the
> greeting finishes. **Applies to cascade and realtime openings.**"*

Turn it off on a Grok Voice 3.0 agent and **the call goes dead**. We measured it on our production
line before we understood why: **6 of 9 calls** in that window had an assistant greeting and
**zero** caller turns in the transcript. The caller hangs up; the operator sees a silent agent.

### What actually happens

Every realtime greeting logs this, with the flag **on** or **off** — 14 of 14 calls:

```
Greeting finished: speech_id=... outcome=interrupted first_output_at=None played_ms=0
```

`first_output_at=None` while the caller plainly hears the greeting. With the flag **on** nothing
breaks, because the caller's own speech resolves the playback — which is why this has been
invisible. With it **off** there is nothing left to resolve it, so `release_opening_input` waits
out the 35 s `timeout` with the caller's audio gate shut.

**The cause is that a realtime service pushes three `LLMFullResponseStartFrame` for one spoken
turn.** We confirmed it at TRACE level and then with two `logger.warning` lines in
`speech_playback.py`:

```
Greeting requested:   speech_id=9557792d...
abertura: boundary_id=9557792d...  in_pending=True    <- the greeting's scope opens, correctly
abertura: boundary_id=bbb3ca0d...  in_pending=False   <- an orphan stacks on top
abertura: boundary_id=032dfc7b...  in_pending=False   <- and another
```

Two independent consequences, and each needed its own line:

1. **`note_output` credits `self._output_scopes[-1]`** — the top of the stack. By the time audio
   arrives the top is an orphan uuid, the `pending` lookup misses, and the greeting's audio is
   credited to nobody.
2. **`_mark_response_boundaries` assumes one start/end pair per source.** The first start consumes
   the expectation; the next two mint orphan uuids and **overwrite** `_response_sources[source]`.
   The `end` therefore carries the id of the *last* one, and **the owning speech never gets an end
   boundary at all**.

Fixing only the first gets `played_ms=10560` and `outcome=timed_out` — audio counted, call still
useless. Both are needed.

### After the patch, on real calls

`allow_greeting_interrupt: false`, **text** greeting with a `{{template}}` in it:

| check | result |
|---|---|
| the greeting resolves on its own | `outcome=played played_ms=10560` (before: never, 14 of 14) |
| the call is not dead | caller turns present, agent answers |
| the greeting cannot be cut | caller spoke at **3.12 s** and at **1.20 s** — greeting intact both times |
| speech over the greeting is not lost | the question asked during the greeting was answered |
| interruption *after* the greeting | still cuts normally, verified |

### Why this matters beyond our line

The workaround your docs suggest — *"static text greetings are not spoken by realtime
speech-to-speech models; use an audio override"* — does not help, because a recorded greeting
cannot carry a template, so a single workflow serving three locations cannot name the location.
With this patch a **text** greeting works, template and all.

**Not visible from text chat**, and not visible with the flag at its default, which is why we
suspect nobody has reported it. The same question was asked in `#oss-support` on 2026-10-14
(*"does the VAD and interruption settings are applicable to realtime models as well"*) and is
still unanswered.

Measured against `adega/dograh-api:main-35b38849` (our build of `main-35b38849`), Grok Voice 3.0,
inbound Twilio calls.

## 13. The transfer leg is placed from the account's own number, so the destination never learns who is calling

**Patched. Nothing crashes — the information is simply absent, and nothing downstream can recover
it afterwards.**

`TwilioProvider.transfer_call()` picks the outbound leg's caller ID at
`api/services/telephony/providers/twilio/provider.py:644-645`:

```
from_number = self.select_from_number()
logger.info(f"Selected phone number {from_number} for transfer call")
```

and posts it as `"From": from_number` to `/Calls.json`. That is one of the account's own numbers.
Nothing in the file reads the inbound call — even though `transfer_context.original_call_sid` is
already in hand in all three transfer modules: `provider.py`, `strategies.py`
(`_find_transfer_context_for_call()`) and `introduction.py` (`process_introduction_status()`).

### What actually happens

Our destination is a PBX reached over SIP. Read back from the Twilio API, every transfer over three
days looks like this — numbers masked, but `From` and `To` really are the same line:

```
From=+1555XXX1111   To=sip:+1555XXX1111@pbx.example.com    94 s   completed
From=+1555XXX1111   To=sip:+1555XXX1111@pbx.example.com    40 s   completed
From=+1555XXX1111   To=sip:+1555XXX1111@pbx.example.com   273 s   completed
```

The phone that rings shows the restaurant's own name, on every transferred call, and the PBX call
log records the restaurant as the origin of a call that came from a customer.

Three consequences, and the third is the one that costs us:

1. **Whoever answers cannot tell a transferred customer from an internal call.** They pick up blind
   to who is on the line.
2. **The PBX call log cannot be used for this.** Every transferred call carries the same origin, so
   the CDR joins to nothing.
3. **Nothing can pair a transfer with the call that produced it.** We match the two to know whether
   escalations are actually being answered; with one constant `From` the only key left is the
   timestamp, and that collides as soon as two lines transfer in the same minute.

### The patch, and the one thing it is careful about

`From` becomes the inbound caller's number, looked up by `original_call_sid`, **only when the
destination is a `sip:` URI**. On a PSTN destination Twilio requires `From` to be a number the
account owns, so the caller's number is not usable there and nothing changes.

Even on SIP we do not assume Twilio will accept it: the account's configuration is the authority.
On a refusal the request is retried **once** with the account number — exactly the value the
transfer used before this change. A transfer therefore cannot fail because of this patch.

### After the patch, on a real call

Transfer to an extension through a ring group, destination `sip:`:

| check | result |
|---|---|
| the phone shows who is calling | the caller's number and city (before: the restaurant's own name, every call) |
| the PBX call log agrees | CDR origin = the caller's number; answered on the extension, 22 s |
| the transfer still succeeds | Twilio `201`, first attempt |
| **the retry path** | **not exercised** — Twilio accepted the caller's number, so the fallback never ran. It is the branch we are least sure of, and we would rather say so than imply we tested it. |

### Why this matters beyond our line

Any deployment that hands calls to a PBX has the same gap, and it is invisible from inside Dograh:
`transfer_success` reports on the INVITE, not on what the destination saw. We only found it by
looking at the phone that rang.

It is also the opposite of the usual caller-ID problem — there is no setting to get wrong.
`select_from_number()` does exactly what its name says. Nothing ever asks for the caller.

Measured against `adega/dograh-api:main-35b38849` (our build of `main-35b38849`), inbound Twilio
calls, SIP transfer to a FreePBX ring group. The file is byte-identical in `35b38849` and in
today's `main`, so the patch applies to `main` unchanged.

**On the recording of a transfer:** this patch deliberately does not touch it. When we first
published this item we left recording out, on the grounds that trunk-level recording already
covered the transferred conversation. That was wrong — item 14 has the measurement that shows it,
and the patch.


## 14. Nothing records the transferred part of a call, and the run's own recording stops at the transfer without saying so

**Patched with one attribute in three places. But the attribute is the wrong shape for you — what
we are asking for is a setting. See the end of this item.**

A transferred call has two halves. The first, caller and agent, is in the run's own
`recording_url`. The second, caller and whoever answered, is in **no artifact at all**, and nothing
in the workflow or the tool configuration can ask for it.

### What actually happens

`recording_url` is presented as the recording of the call. On a transferred call it stops at the
transfer, and nothing says so. Every run with a transfer on our production line, against the same
call's duration in Twilio:

| run | the run's recording covers | the call lasted |
|---|---|---|
| 1990 | 38 s | **250 s** |
| 1975 | 104 s | **373 s** |
| 1980 | 165 s | 223 s |
| 1988 | 33 s | 124 s |
| 1978 | 70 s | 124 s |
| 1970 | 31 s | 75 s |
| 1982 | 35 s | 72 s |
| 1974 | 44 s | 41 s |

Seven of the eight are short by the whole transferred conversation — in the worst one, 212 of 250
seconds are missing. The eighth is the only one where the two agree; we did not chase why, and we
are not going to claim it was a transfer that never connected without checking.

There is nothing to find on the provider side either. Every transfer on one day, read back from
the Twilio API:

```
13 of 13 transfer legs (outbound-api to a sip: URI)  ->  no recording
76 of 76 inbound calls the agent answered            ->  no recording
```

We got this wrong ourselves first, and it is worth stating because the setup is common: **automatic
trunk recording does not cover either of them.** An inbound call arrives on a Programmable Voice
number rather than over the SIP trunk, and the transfer leg is a fresh outbound API call, so a
trunk-level `record-from-answer` sees neither. The only reason it looked covered to us is that our
first test call was dialled from our own PBX, which does traverse the trunk. A real caller does not.

### Where

The conference TwiML is built in three places, and none of them asks for recording:

- `api/services/telephony/providers/twilio/provider.py:658` — the destination leg, in the inline TwiML posted to `/Calls.json`
- `api/services/telephony/providers/twilio/strategies.py:72` — the caller's leg, in `TwilioConferenceStrategy.execute_transfer`
- `api/services/telephony/providers/twilio/introduction.py:27` — the introduction path, built with `SubElement`

### After the patch

| check | result |
|---|---|
| a recording exists for the transferred part | yes, `source: Conference` (before: none, 13 of 13) |
| it begins when both legs are in the conference | 12 s of a 22 s transfer — the whisper played to the person answering happens before that and is not captured |
| the caller-and-agent half is untouched | still in the run's own recording; the two together cover the call |

### What we are actually asking for

Not this attribute. `record="record-from-start"` hardcoded is wrong for a product, for three
reasons:

1. **It costs money** — per minute recorded and per month stored.
2. **It records the person who answers**, not only the caller. Florida, where we operate, requires
   all-party consent; our caller hears *"you're on a recorded line"* in the greeting and the staff
   member who answers hears nothing. Whether to record that leg is the operator's decision, which
   means it has to be a toggle, not a constant.
3. **The recording is born unreachable.** It appears in Twilio with nothing tying it to the
   workflow run, so finding it means searching by timestamp.

So: a setting beside `introduction_enabled` — `record_transfer`, off by default — and the resulting
recording referenced from the run, the way `recording_url` already is. Then the operator decides,
and whoever reviews the call can actually hear the half that matters most: the one with a human in
it.

Measured against `adega/dograh-api:main-35b38849` (our build of `main-35b38849`), inbound Twilio
calls, SIP transfer to a FreePBX ring group. All three files are byte-identical in `35b38849` and
in today's `main`, so the patch applies to `main` unchanged.

## 15. A realtime provider's cumulative transcription is published once per partial, so the caller's sentence is repeated in the transcript

**Patched. The conversation is unaffected — what this damages is everything that reads the call
afterwards.**

Grok Voice sends **several** `input_audio_transcription.completed` events for one utterance: same
`item_id`, each carrying everything transcribed so far.
`_handle_evt_input_audio_transcription_completed` publishes each of them as a finalized
`TranscriptionFrame`, the user aggregator joins them into the turn text, and the stored transcript
reads:

```
user: Have a Have a question. Have a question.
user: Do you still Do you still have the promotion? Do you still have the promotion for
      birthdays? Do you still have the promotion for birthdays where the birthday person
      doesn't pay? ...
```

### What actually happens

Two `logger.warning` lines in the handler, one bench call, one sentence spoken:

```
item=888810a2  'I would like to'
item=888810a2  'I would like to book a table for six'
item=888810a2  'I would like to book a table for six people this Saturday.'
item=888810a2  '... this Saturday at seven in the evening'
item=888810a2  '... and one of them'
item=888810a2  '... and one of them is celebrating a birth'
item=888810a2  '... and one of them is celebrating a birthday.'
item=888810a2  '... and one of them is celebrating a birthday.'   <- identical to the previous one
```

**Eight events, one utterance, one `item_id`.** And note the fourth against the fifth: `at 7 in the
evening` becomes `at seven in the evening`. The provider also **revises** what it already sent, so
publishing only the new suffix is not an option.

Two explanations we ruled out before patching, in case they look likely to you too:

1. `transcript_log_coordinator.record_user_transcript` concatenates with `\n` — but the stored text
   contains no newline. The join that produces it happens in the aggregator, with spaces.
2. Nothing accumulates transcription `delta`s, anywhere in `api/` or in pipecat. Only the
   `completed` handler exists.

### Why `response.created` is not the boundary

Our first design flushed the utterance when the model started answering. Measured, it does not hold:
`response.created` arrives **before** the last transcription event, fires once before any caller
speech at all, and arrives **twice** per turn — the same chattiness as item 12, where this service
pushes three `LLMFullResponseStartFrame` for one spoken turn.

The signal that does hold is the text itself. When an event repeats the previous text, the text has
stopped growing and the utterance is over.

### After the patch, on bench calls

| check | result |
|---|---|
| a long sentence | stored once, in full — before, eight growing prefixes of it |
| the same in the context-derived log | also once; both storage paths are fed from these frames |
| **control: three distinct utterances** | all three preserved, none merged, none repeated |
| an utterance whose text never repeats | flushed when the next utterance arrives, so it is not lost |

### Why this matters beyond our line

It does not degrade the conversation: with a speech-to-speech model the dialogue lives in the
provider's session, not in this context. It degrades everything read afterwards — the stored
transcript, the QA node's summary, call analysis, and any operator-facing view of a call. On our
line a repeated fragment reached an automated report, and we stripped it there for four days before
understanding where it came from.

The guard belongs in the provider wrapper because the frame is published with `finalized=True`:
nothing downstream can tell a partial from a final, and the aggregator is deliberately generic
across providers. A provider that sends one final per utterance is unaffected by this patch.

Measured against `adega/dograh-api:main-35b38849` (our build of `main-35b38849`), Grok Voice 3.0,
inbound Twilio calls.

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
