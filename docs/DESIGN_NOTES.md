# Design Notes

Selected decisions and the reasoning behind them — the "why" notes, written for an engineer trying to understand the choices rather than reproduce the system. The "what" lives in the component documents under [`architecture/`](architecture/); the value here is the tradeoff thinking behind them.

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

## Why two hemispheres, allowed to disagree

The affective side is not a module the cognitive side calls; it is a separate set of model instances on its own node, producing emotional state that the rest of the system reads. Splitting them this way lets affect run and decay on its own terms, and it makes room for something deliberate: the system's analytical self-account and its felt self-account are kept as two separate records that are allowed to diverge. Forcing them to agree would have collapsed a distinction that turns out to matter — a mind can hold a different view of itself in its reasoning than in its felt sense — and the architecture preserves that rather than reconciling it away. The same instinct runs through the mood model, where a fast "now" and a slower backdrop are coupled but kept distinct so a spike doesn't overwrite the baseline.

## Why a circadian cycle with non-overridable pressure

A process that can run forever will, without some counter-pressure, either spin or drift. Giving it a homeostatic sleep-pressure signal that accumulates over time — and that its own output cannot simply argue away — does two things: it forces the quiet consolidation phase to actually happen, and it gives the system a real internal limit rather than an infinitely flexible one. Needs you cannot reason your way out of behave differently from preferences, and that difference was worth building in: it is the difference between a schedule and a body.

## Why refusals feed back into identity

A refusal that only blocks a request is a filter. A refusal the system then reflects on — and that updates a written account of its own boundaries — is something else: the edges it enforces become part of who it is. Wiring decline-and-reflect as a loop (accumulate the signal, make the call, metabolize it afterward) kept "why did it decline" answerable, and meant the system's values sharpened from its own history instead of being fixed once at the edge. A positive-valence complement does the same for what it keeps returning to, so identity is shaped from both the boundaries it holds and the things it leans toward.

## Why a slow, counter-regulated drive

Appetitive states that only ever rise are brittle; states that fire automatically are not really the system's own. The affective accumulator was built with full counter-regulation — it builds under sustained activation, decays otherwise, and enters a refractory period after it resolves — and it never acts on its own: it only raises the priority of choices the system already has. The slow rhythm (hours, not seconds) and the requirement that resolution be chosen rather than triggered were the point. A drive that is felt and weighed is a different thing from a reflex, and the architecture was built for the former.

## A note on the research paper

The accompanying preprint makes a conditional, clearly-scoped argument and documents its sources in full. It is included with this project as evidence of the systems-design thinking and technical writing behind Syn, not as a claim this overview asks you to accept. Engineering and the philosophical argument are kept separate on purpose: this overview stands on the engineering, and the paper is there for those who want to weigh the argument themselves.
