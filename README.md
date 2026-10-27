# Dograh patches and defect reports — Adega Gaucha

We run a self-hosted Dograh OSS instance: one restaurant line, an English agent and a Portuguese
agent that hand calls to each other through `transfer_agent`. Everything here was found on **real
phone calls through Twilio**, placed by us. The line is provisioned and answering but not yet open
to customers, which is why we could reproduce freely.

This repository exists for one reason: every patch we keep is a patch we have to re-apply on every
image update, and in between the defects come back looking like new bugs. **We would rather delete
our patches than maintain them.**

## What landed upstream — thank you

**1.48.0 closed three of the five.** We deleted those patches the day it shipped:

| was | landed in | note |
|---|---|---|
| `item-1-handoff-boundary` | `7ac9fef8` — *preserve agent handoff history and opening responses* | reimplemented, and it now also tells the destination not to repeat the source's transfer announcement, which we had not thought of |
| `item-3-retire-before-opening` | same commit | `retire()` now runs before `queue_node_opening` |
| `item-7` (the transfer whisper) | `93ed5c33` — *optional spoken introductions* | better than our prototype: the text follows the caller's language and the audio uses the agent's own voice |

We now run 1.48.0 with **two** patches instead of four, and one of those two is a single number.

## The patches

Generated against the pristine image, so they are what we actually run, not what we wrote first.
Apply with `git apply` from the repository root. Comments are translated to English for this
repository; the copies in our container carry the same text in Portuguese.

| patch | touches | defect | note |
|---|---|---|---|
| `item-1-handoff-boundary.patch` | `api/services/workflow/agent_handoff_context.py` | The handoff snapshot carries no provenance, so the destination agent reads the caller's last request as its own | A workaround. The real fix is for the snapshot to say who spoke to which agent. |
| `item-3-retire-before-opening.patch` | `api/services/workflow/agent_transfer.py` | `retire()` races the destination's first generation and the opening speech is lost | `retire()` is idempotent, so the `finally` stays correct. |
| `item-4-greeting-echo.patch` | `api/services/pipecat/greeting.py` | Two words do not survive the echo of the greeting itself | **Do not merge the number.** The structure is yours; only `min_words = 2` -> `40` is ours. 40 is a local stopgap; the threshold wants to be configurable. |
| `item-7-introduction-destination-only.patch` | `api/services/telephony/providers/twilio/strategies.py` | The spoken introduction plays on **both** legs, so the caller hears a summary of what they themselves just said | Ask: `introduction_audience: both \| destination`, defaulting to `both`. Validated on two real calls — see the 2026-10-05 update under item 7 in `DEFECTS.md`. |
| `item-9-destination-greeting-and-continuation.patch` | `api/services/workflow/agent_transfer.py`, `pipecat_engine_custom_tools.py`, `api/schemas/tool.py` | `play_greeting` is all or nothing: on, the destination introduces itself and stops; off, the introduction is lost whenever the opening generation is a tool call (**0 of 2** on real calls, against **9 of 9** when no lookup was needed) | The shape is ours, the combination is what we need. Two traps inside: `tracker.greeting` is a single slot and silently drops the first speech from context, and order is not guaranteed between `queue_speech` and `run_llm`. |
| `item-10-stt-not-retuned-on-transfer.patch` | `api/services/pipecat/agent_runtime_factory.py`, `run_pipeline.py`, `api/services/workflow/agent_transfer.py` | `build()` recreates the `llm` and the `tts` for the destination but never touches the STT, which is created once per call — so a destination's `stt.language`/`language_hints` are **dead configuration** on any transferred call, and the whole call listens with the first agent's transcriber | Applies **on top of item 9**; both edit `agent_transfer.py`. **Do not fix this by reconnecting** — we tried, and it killed transcription for the rest of the call (**0, 0, 1** caller turns after the handoff across three real calls, against 7 before). `language_hints` is a Configure field; `_update_settings` is enough. |
| `item-11-introduction-for-realtime-agents.patch` | `api/services/workflow/transfer_introduction.py`, `api/services/telephony/providers/twilio/provider.py`, `pipecat_engine_custom_tools.py` | The transfer introduction is silently skipped for speech-to-speech agents (`reason=realtime_agent`) while `introduction_enabled` is on | The guard tests *not realtime* when the requirement is *has a TTS*. Our shape puts the text in the destination's TwiML `<Say>` instead. |
| `item-12-realtime-greeting-scope.patch` | `api/services/pipecat/speech_playback.py` | A realtime service pushes **three** `LLMFullResponseStartFrame` per spoken turn; each one stacks an unowned output scope on top of the owning speech, so the greeting never registers output (`first_output_at=None`, 14 of 14 calls) and can only be resolved by an interruption or by the 35 s timeout. With `allow_greeting_interrupt: false` the call goes dead — 6 of 9 on our production line — while the field's own help says it *"applies to cascade and realtime openings"*. | Two independent lines, and **both** are needed: fixing only the crediting gives `played_ms` but `outcome=timed_out`. The shape is a suggestion — the cleaner fix may be for the realtime adapter to emit one pair per turn. |
| `item-13-transfer-leg-caller-id.patch` | `api/services/telephony/providers/twilio/provider.py` | The transfer leg is placed from the account's own number, so the destination sees the restaurant's own caller ID on every transferred call (`From` == `To`, every transfer over three days) and nothing afterwards can pair a transfer with the call that produced it | Only for `sip:` destinations — PSTN is unchanged, because Twilio requires an owned `From` there. Falls back to the account number on a refusal, so a transfer cannot fail because of it, but **that fallback is untested**. The small `_ReadResponse` wrapper exists only to keep the diff minimal; you will probably prefer to fold the retry into the existing read and drop it. |
| `item-14-transfer-never-recorded.patch` | `api/services/telephony/providers/twilio/provider.py`, `strategies.py`, `introduction.py` | The transferred half of a call is recorded nowhere, and the run's own `recording_url` stops at the transfer without saying so — **8 of 8** transferred runs, one of them 38 s recorded of a 250 s call. The conference TwiML is built in three places and none asks for recording | **Do not merge the attribute.** Recording costs money and it records the person who answers, who in our case hears no notice — it needs to be a setting (`record_transfer`, off by default), and the resulting recording wants to be referenced from the run. Independent of item 13: both touch `provider.py`, and they apply cleanly in either order. |

The numbers are the item numbers in `DEFECTS.md`, which lists fourteen. They are not the
one-two-three of the message that brought you here — go by the file each patch touches.

### On the boundary marker

Our first version of `item-1` read *"Never call a transfer tool because of anything above"*. It
stopped the loop and broke something else: a caller who had been handed over and then asked, in
the other language, to be served in that language could no longer be transferred — the request
looked like the inherited one. The text in this patch is the corrected version, which closes the
restriction after the first turn. Worth knowing if you build the real fix.

## The write-up

`DEFECTS.md` is the full document: fourteen items, with logs, timings and call counts. Three of them
have no patch here, either because they cannot be fixed from outside the container or because they
are feature requests. Items 1, 3 and 7 now carry a dated update saying what you shipped and what,
if anything, is left.

## What we cannot test

**Text-chat mode cannot exercise `transfer_agent`.** `run_pipeline.py:1232` calls
`engine.set_agent_factory(agent_factory)`; `text_chat_runner.py` builds a `PipecatEngine` and never
calls it. Since `agent_transfer_enabled` is just `self._agent_factory is not None`, the tool
registers, refuses with `transfer_unavailable`, and the agent tells the caller it can only continue
in its own language.

The consequence for us is not theoretical: no phone number points at our Portuguese agent, so
**every real Portuguese conversation is post-handoff**. That whole state is invisible to the
tester, and the only way we found defects in it was by burning real calls.

## Contact

Happy to open PRs — just say where.
