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

The numbers are the item numbers in `DEFECTS.md`, which lists eight. They are not the
one-two-three of the message that brought you here — go by the file each patch touches.

### On the boundary marker

Our first version of `item-1` read *"Never call a transfer tool because of anything above"*. It
stopped the loop and broke something else: a caller who had been handed over and then asked, in
the other language, to be served in that language could no longer be transferred — the request
looked like the inherited one. The text in this patch is the corrected version, which closes the
restriction after the first turn. Worth knowing if you build the real fix.

## The write-up

`DEFECTS.md` is the full document: eight items, with logs, timings and call counts. Three of them
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
