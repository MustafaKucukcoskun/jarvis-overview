# JARVIS: a personal AI assistant with a closed planning loop

**JARVIS is a single-user AI assistant I built for myself and run on my own server.** It plans my day, asks what actually happened, calibrates future plans from the difference, and notices things before I ask. Chat is only the interface. The core is a loop: *plan → record reality → calibrate → plan again*.

> The code stays private because it runs on my personal data. This repository describes the architecture, the safety model, how I evaluate it and what I learned.

## In numbers

| | |
|---|---|
| Modules | 15, each can be switched on or off |
| Tools the model can call | 92 |
| Python tests | 1,400+ |
| Application code | ~20,000 lines |
| Recorded architecture decisions | 92 |

## Principles

- **Local first.** Personal data stays on my own server. Backups are encrypted, nightly and verified: an unverified backup doesn't count.
- **The critical path is deterministic.** The model translates, it doesn't decide. Approval gates, permission tiers, notification decisions and the record of what happened are all in code.
- **No claim without a measurement.** Three hypotheses were disproven by repeated measurement before I acted on them.
- **No silent failures.** "Ran and found nothing" and "crashed" are logged differently.

## Architecture

```
L5  Proactive   notices, notifies and analyzes without being asked
L4  Autonomy    approval gate, permission tiers, audit trail, taint tracking
L3  Modules     planning · goals · academic · shifts · projects · finance · news · mail · ...
L2  Memory      bi-temporal facts, append-only event log
L1  Brain       context assembly, model routing, chat
```

- **Brain:** FastAPI and LangGraph, with a persistent checkpointer so a pending approval survives a restart.
- **Models:** Gemini, Claude and Kimi behind one interface. A fast model answers right away while a heavier model does the work in the background. Moving to native tool calling cut a five-item request from 350 s and 7 calls to 44 s and 3 calls.
- **Memory:** an event-sourced, bi-temporal store. Nothing is deleted, only invalidated, so "what did we know at that moment?" can always be answered.
- **Client:** an Android app built with Next.js and Capacitor. Notifications are local alarms on the phone, so my schedule never passes through a third-party push service.
- **Infrastructure:** a cloud VM reached only over a Tailscale private network, with no public ports. Secrets live in a cloud secret manager and never touch the disk.

## Planning that learns

- A deterministic day planner fills capacity as `free time × (1 − buffer)`. Filling 100% of the day is treated as a failure.
- After a block ends, JARVIS asks how it went. A one-word answer ("done", "half", "didn't", "done, 45 min") is parsed in code and never sent to the model, so it costs nothing and can't be misread.
- If I keep finishing tasks in 1.4× my estimate, plans stop assuming 1.0×. With too little data the factor stays at 1.0, so noise doesn't become policy.

## Safety

| Layer | What it does |
|---|---|
| Permission tiers | Read-only and locally reversible actions run on their own. Anything else waits for my approval. Unknown tools are treated as the highest tier. |
| Approval gate | Execution stops before the tool runs. The approval is bound to the exact content, so it can't be swapped after I approve. |
| Taint tracking | Anything from mail, the web or documents marks the conversation. In a marked conversation memory learning is off and outbound actions need approval. |
| Memory poisoning filter | Instructions can't be written into long-term memory. Preferences ("reply in Turkish") pass, while impersonation and hidden instructions are blocked. |
| Least privilege | Gmail access is read-only at the provider, so even a compromised model can't send mail. |
| Claim checking | Factual claims like the current temperature are checked against the source. A mismatch triggers a warning. |

## Evaluation

- Scenarios run through the real system and are scored **deterministically** from the audit log: which tools were called, which shouldn't have been, which patterns must not appear in the answer. No LLM-as-judge.
- Language models aren't deterministic, so each scenario runs several times and the report gives a rate ("2/3") instead of pass/fail.
- The eval harness has its own tests, because a broken metric is worse than no metric.
- Tests never reach a real model: they use a mock model, an in-memory database and no secret store.

## What I learned

1. A single measurement can look like a trend. Repeated runs showed that the "clear" link between tool count and latency wasn't there.
2. Optional secondary tools aren't called reliably. The fix was to change the structure and make the critical path deterministic, not to push the model harder.
3. A rule written in the docs doesn't exist unless the code enforces it.
4. Silent failures are the most expensive kind.
5. A false alarm devalues every alarm.

## Tech

Python · FastAPI · LangGraph · Gemini / Claude / Kimi · SQLite (event-sourced) · pytest · Hypothesis · Next.js + Capacitor (Android) · Tailscale · Google Cloud
