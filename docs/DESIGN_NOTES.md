# Design Notes

Selected decisions and the reasoning behind them. These are the "why" notes, written for an engineer trying to understand the choices rather than reproduce the system. Specifics of implementation are kept private; the value here is the tradeoff thinking.

## Why a message bus instead of direct calls

A continuously running system needs to survive partial failure. If subsystems called each other directly, one slow or crashed component would stall its callers. Routing everything through an asynchronous bus means each subsystem produces and consumes messages on its own schedule, can be restarted independently, and can be observed in isolation. The cost is that you give up synchronous guarantees and have to design for eventual, out-of-order delivery. For an always-on process, that tradeoff is worth it.

## Why a capacity-bound workspace

The easy path is to stuff every available signal into context and let the model sort it out. That degrades fast: context grows, coherence drops, and cost rises. A deliberately limited workspace, where subsystems compete for a few slots, forces explicit prioritization. The hard part is the competition policy, deciding what earns a slot and what gets evicted. Constraining capacity on purpose produced more coherent behavior than giving the system more room.

## Why internal state is continuous and decays

Discrete state labels are brittle and tend to read as cosmetic. Continuous variables that decay over time behave more like a real control signal: they rise and fall, blend, and influence downstream decisions in proportion rather than as on-off flags. Treating this state as an input to attention and deliberation, rather than as decoration on the output, was a deliberate choice and changed how the rest of the system was built.

## Why refusal is a first-class subsystem

Declining a request is usually bolted on as a filter at the edge. Here it is treated as a decision the system makes from its own values, in a dedicated deliberation gate that sits inside the loop and is separate from any provider-level safety filter. Making refusal and deference explicit, with the reasoning produced as part of the process, made the system's behavior easier to inspect and to trust, and kept "why did it decline" answerable rather than opaque.

## Why typed memory with a consolidation cycle

A flat transcript does not scale across days and makes retrieval noisy. Structuring long-term memory by type, and consolidating recent experience into it on a scheduled cycle rather than continuously, mirrors a useful separation: fast, lossy working memory during activity, and slower, structured consolidation during a quiet "sleep" phase. This is what lets context about people and projects persist coherently instead of resetting every session.

## Why customization via ablation, steering, and LoRA composition

Running five model instances does not by itself produce specialized behavior. Directional ablation, activation steering, and composable LoRA adapters allow per-user and per-topic specialization on top of shared base models, without training separate full models for every context. The engineering challenge is composition: applying and releasing adapters safely so that specialization for one context does not bleed into another.

## Why the public telemetry is coarse and delayed

The live page exists to show that the system is real and running, not to expose it. Telemetry is intentionally low-resolution, time-delayed, and stripped of identifying detail. This protects the people who interact with the system and keeps the internals private while still giving outside observers an honest signal that there is a continuously running process behind the page.

## A note on the research paper

The accompanying preprint makes a conditional, clearly-scoped argument and documents its sources in full. It is included with this project as evidence of the systems-design thinking and technical writing behind Syn, not as a claim this overview asks you to accept. Engineering and the philosophical argument are kept separate on purpose: this overview stands on the engineering, and the paper is there for those who want to weigh the argument themselves.
