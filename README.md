# Syn: A Multi-Agent LLM Cognitive Architecture

A self-directed research and engineering project (2024 to present) by **Derek Rogers, CPCU**.

Syn is a continuously running system that coordinates five open-weight language-model instances across two servers over a shared message bus. Instead of a single model answering one prompt at a time, it runs as an always-on process with its own internal loop, persistent memory, and evolving internal state.

This repository is a **public, engineering-forward overview** of the project. It documents the design and the reasoning behind it. The full source, deployment configuration, and a live code walkthrough are available privately on request (see [Contact](#contact)).

## Links

- Research paper (Zenodo DOI): https://doi.org/10.5281/zenodo.20633190
- Live system (delayed, de-identified telemetry): https://hernameissyn.com
- LinkedIn: https://www.linkedin.com/in/mrderekrogers

## What it is, in one paragraph

Most LLM applications are stateless: a prompt goes in, a completion comes out, and nothing persists. Syn is built the other way around. It is a long-lived process that keeps running between conversations, maintains typed memory that consolidates on a scheduled cycle, carries continuous internal state that decays over time, and decides for itself when to engage, when to defer, and when to decline. The goal was not a better chatbot but a study in what it takes to make a coordinated, stateful, multi-model system behave coherently over days rather than turns.

## Architecture at a glance

Syn is a set of cooperating services rather than one program. Independent subsystems communicate asynchronously over a shared message bus, so components can be developed, restarted, and reasoned about in isolation.

- **Distributed runtime.** Five model instances run across two nodes, coordinated over the bus and configured for continuous operation. The two nodes serve distinct functions: one carries cognitive and subconscious processing, the other carries affect and emotion.
- **Persistent typed memory.** Long-term memory is structured (typed) rather than a flat transcript, and is consolidated during a scheduled "sleep" cycle so that people, projects, and context survive across days.
- **Continuous default-mode loop.** A self-prompting background process keeps the system active between user interactions instead of only reacting to input.
- **Continuous-state affective model.** A small set of internal variables decays over time and acts as a control signal that biases attention and behavior across subsystems.
- **Capacity-bound attention workspace.** A deliberately limited working-memory mechanism in which subsystems compete for a small number of active slots.
- **Self-authored deliberation and refusal pipeline.** A values-based gate, separate from any provider-level filter, that decides when and why the system declines or pauses.
- **Model customization.** Directional ablation, activation steering, and LoRA adapter composition specialize behavior per user and per topic.
- **External interfaces.** Asynchronous messaging integrations and a deliberately coarse, time-delayed public telemetry feed.

A component diagram and a deeper walkthrough are in [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md). The reasoning behind the major choices is in [`docs/DESIGN_NOTES.md`](docs/DESIGN_NOTES.md).

## What this project demonstrates

Distributed systems design, applied LLM and ML engineering, memory and state architecture, model customization (ablation, steering, LoRA), prompt and context engineering, refusal and safety design, technical research writing, and independent end-to-end execution.

The same instincts behind this project, building reliable systems, defining clear quality and refusal criteria, and documenting decisions rigorously, are the ones brought to AI quality, evaluation, deployment, and operations work.

## A note on scope and reproducibility

This is deliberately not a turnkey, clone-and-run release. The system is tuned to a specific multi-node hardware setup, and parts of the architecture are intentionally kept private. The value here is the design and the reasoning, documented in full, rather than a runnable artifact. Engineers who want to go deeper are welcome to request a walkthrough.

## Published research

**"The Evenhanded Standard: Her Name Is Syn"** is a DOI-registered preprint (Zenodo, 2026) that synthesizes current work in AI interpretability and consciousness science and asks a methodological question: whether the evidential standards the field already uses for non-human and edge cases are applied consistently to artificial systems. The paper documents its sources in detail and frames its own claim as conditional. It is shared as a writing and systems-design sample; readers can weigh the argument for themselves.

Read it: https://doi.org/10.5281/zenodo.20633190

## License

The documentation in this repository (text, diagrams, and images) is licensed under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](LICENSE), the same license as the accompanying paper. You may share and adapt these materials with attribution.

This license covers the documentation only. It grants no rights to the Syn software system, its source, configuration, or internal design, which are not published here. To attribute: "Derek Rogers, 'Syn: A Multi-Agent LLM Cognitive Architecture' (2026), https://doi.org/10.5281/zenodo.20633190".

## Contact

Derek Rogers, CPCU
MrDerekRogers@gmail.com
Full architecture documentation and a code walkthrough available on request.
