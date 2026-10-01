# Dograh patches and defect reports — Adega Gaucha

We run a self-hosted Dograh OSS instance on `dograhai/dograh-api:main-1e47eb54`: one restaurant
line, an English agent and a Portuguese agent that hand calls to each other through
`transfer_agent`. Everything here was found on **real phone calls through Twilio**, placed by us.
The line is provisioned and answering but not yet open to customers, which is why we could
reproduce freely.

This repository exists for one reason: we keep three patches **inside the running container**, in
the writable layer. Every image update reverts us silently and the defects come back looking like
new bugs. **We would rather delete our patches than maintain them.**

## The patches

Generated from the running container against the pristine image, so they are what is actually in
production here, not what we wrote first. Apply with `git apply` from the repository root.

| file | defect | note |
|---|---|---|
| `item-1-handoff-boundary.patch` | The handoff snapshot carries no provenance, so the destination agent reads the caller's last request as its own | A workaround. The real fix is for the snapshot to say who spoke to which agent. |
| `item-3-retire-before-opening.patch` | `retire()` races the destination's first generation and the opening speech is lost | `retire()` is idempotent, so the `finally` stays correct. |
| `item-4-greeting-echo.patch` | Two words do not survive the echo of the greeting itself | **Do not merge the number.** 40 is a local stopgap; the threshold wants to be configurable. |

### On the boundary marker

Our first version of `item-1` read *"Never call a transfer tool because of anything above"*. It
stopped the loop and broke something else: a caller who had been handed over and then asked, in
the other language, to be served in that language could no longer be transferred — the request
looked like the inherited one. The text in this patch is the corrected version, which closes the
restriction after the first turn. Worth knowing if you build the real fix.

## The write-up

`DEFECTS.md` is the full document: eight items, with logs, timings and call counts. Four of them
have no patch here, either because they cannot be fixed from outside the container or because they
are feature requests.

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
