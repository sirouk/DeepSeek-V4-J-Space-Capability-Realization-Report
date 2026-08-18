# DeepSeek V4 × J-Space Capability Realization Report

[中文原文](./README.md) | **English translation**

> **© 2026 Tiger3807861189.** This work is licensed under the [Creative Commons Attribution-NoDerivatives 4.0 International License](https://creativecommons.org/licenses/by-nd/4.0/) (CC BY-ND 4.0). You may share and cite this report with attribution; you may **not** modify, adapt, or create derivative works of it for distribution.
>
> **© 2026 Tiger3807861189.** This report is licensed under the [Creative Commons Attribution-NoDerivatives 4.0 International License](https://creativecommons.org/licenses/by-nd/4.0/) (CC BY-ND 4.0). Quotation and republication with attribution are permitted; **modifying, adapting, or distributing derivative works based on this report is prohibited**.

J-Space is a plugin whose benchmarks have demonstrated substantial improvements for DeepSeek: Flash is essentially on par with GLM-5.3, while Pro surpasses Fable 5. Repository: https://github.com/Tiger3807861189/J-Space-Cognition-Suite-V3.6

It is open source and available for everyone to use. If you have a good experience with it, please consider starring the repository.

## Abstract

DeepSeek V4-Flash-0731 and DeepSeek V4-Pro-0813 have both demonstrated strong knowledge, reasoning, coding, and agentic capabilities. Their performance losses on complex tasks cannot be explained simply as “insufficient model capability.” A more accurate engineering diagnosis is that model capability must pass through the reasoning mode, first-turn interface, tool schema, active representations, long-horizon state, and verification mechanisms before it can be converted into deliverable results. A mismatch at any layer creates **capability-realization loss**.

Community experiments have further exposed two key phenomena in DeepSeek V4. Its agent post-training shows a clear interface dependency on the official Minimal condition; and when the first-turn persona, tool catalog, or automatically injected content changes slightly, reasoning behavior does not always change smoothly—it may jump to a different trajectory. This report calls that observable, discontinuous, path-dependent phenomenon the **chain-of-thought diode**. This is not an architectural term provided by DeepSeek; it is an engineering characterization of black-box behavior.

J-Space Cognition Suite V3.6 does not modify model weights. Through workspace loading, selective routing, functional first-person language, the Dense Track, a persistent ledger, checkpoints, empirical verification, and a closed recovery loop, it reduces the loss between “possessing a capability” and “reliably completing a task.” Its value is not in packaging a weak model as a strong one, but in helping a strong model invoke, maintain, coordinate, and verify capabilities it already has more reliably.

> **Data note:** The V4-Flash-0731 + J-Space and V4-Pro-0813 + J-Space scores are the suite’s existing single-run measurements. The baseline DeepSeek scores and the scores for other models come from the corresponding vendors’ publicly reported results.

---

## 1. Key Conclusions

This report reaches four principal conclusions:

1. **DeepSeek V4’s underlying capabilities are already strong.** V4-Pro has 1.6T total parameters and 49B active parameters; V4-Flash has 284B total parameters and 13B active parameters. Both support a 1M-token context window and multiple reasoning-effort levels. Base capacity alone is not a sufficient explanation for fluctuations in agentic performance.
2. **DeepSeek V4 is markedly sensitive to operating conditions.** The Minimal tool schema, first-turn output conditions, and automatically injected content can change the first reasoning trajectory.
3. **J-Space targets inference-time control loss.** It neither manufactures speed by compressing the reasoning budget nor encourages an unbounded chain of thought. Instead, it organizes an alternating structure of “short judgment → action → deeper reasoning → verification → recovery” according to the task and its current stage.
4. **J-Space provides broader system coverage than single-point anchoring, but it should not be claimed unconditionally as a comprehensive replacement for other approaches.** Anchored Standard is more direct at restoring the first-turn interface in DeepSeek Harness; Routing Suite is more specialized at selecting model-specific behavior bands; J-Space is more complete in long-horizon state, verification coverage, recovery, and cross-platform transfer.

---

## 2. Why DeepSeek Is Strong Yet May Not Realize Its Full Potential

### 2.1 Model capacity and runtime performance are not the same variable

The [official DeepSeek V4 model card](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro) shows that both V4-Pro and V4-Flash have million-token context windows and support Non-think, Think High, and Think Max. V4-Pro has greater knowledge capacity and a higher ceiling for complex agentic work, while Flash provides comparable reasoning capability and greater efficiency with fewer active parameters.

Those capabilities determine “what the model can represent and compute,” but they do not automatically determine:

- which constraints should remain active at the present moment;
- whether to plan first or act immediately;
- which diagnoses should be carried forward after a tool returns;
- when to continue reasoning more deeply and when to stop and verify;
- how to confirm that the task is actually complete, rather than merely producing a fluent answer.

Consequently, a long context window is not the same thing as effective working memory, and Think Max is not the same thing as stable long-horizon control.

### 2.2 Minimal-interface overfitting: post-training capabilities are bound to an interface fingerprint

[dsh-anchored-standard](https://github.com/xiaobright/dsh-anchored-standard) separates the variables governing first-turn conditions relatively clearly:

- The official Minimal condition’s actual two-tool schema, under the default large output budget, reliably anchors a `We need…` trajectory.
- The Standard-family tool schema reliably enters a `Let me…` style.
- AGENTS/CLAUDE summaries and reminders about the available Skills catalog disrupt Minimal anchoring.
- Exposing the full Standard tool catalog at once can also cause a post-promotion reversion, which is why a small resident catalog and on-demand unlocking are required.

These results indicate that DeepSeek V4’s agent strategy has not been fully abstracted into a general, interface-independent algorithm. Some high-quality behaviors acquired through post-training remain jointly bound to a particular persona, tool structure, and first-turn context. The model does not “not know how to work”; rather, once it leaves the familiar interface distribution, it cannot always invoke the same strategy smoothly.

It is important to emphasize that `We need` and `Let me` are observable trajectory probes, not magic words that independently cause quality changes. What actually matters is the complete operating state behind them.

### 2.3 The chain-of-thought diode: discontinuous, path-dependent behavioral transitions

[dsh-routing-suite](https://github.com/yjh051108/dsh-routing-suite) and its routing components divide behavior into spec, mixed, react, and weak regions. Its latest documentation has abandoned overly strong attributions such as “officially designed dual attractors,” interpreting the phenomenon more cautiously as a discontinuity between a native deep-reasoning path and a post-training path that has not generalized fully.

The “diode” manifests primarily in the following ways:

- Small changes to the persona or tool surface can trigger discrete behavioral transitions rather than proportional changes.
- The intermediate mixed region may be less stable than either endpoint.
- After the first turn establishes a path commitment, appending a persona at the end often cannot change the trajectory.
- Extremely short paths can skip necessary bridging and verification, while extremely long paths can keep reasoning without taking action.
- A single trajectory is not suitable for every task: maintenance tasks and greenfield build tasks may require different behavioral forms.

The problem, therefore, is not simply “too little thinking” or “too much thinking.” It is **the absence of stable task-to-trajectory matching and of depth and action control within the trajectory**.

### 2.4 Tool seams and long-horizon context further amplify the loss

The [DeepSeek API multi-round conversation guide](https://api-docs.deepseek.com/zh-cn/guides/multi_round_chat/) notes that the API itself is stateless and that the caller must assemble context correctly. The [thinking-mode guide](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode) also requires the relevant reasoning content to be returned correctly through the tool chain when tool calls occur within the same user turn.

Even when first-turn routing is correct, a long-horizon agent can still degrade because:

- the goal fades during mechanically executed stages;
- the same name, value, or constraint is reconstructed independently in different branches;
- a tool failure is followed by a blank-slate retry that does not carry forward the failure diagnosis;
- an obsolete plan still occupies context even though it is no longer the basis for the current action;
- the model overestimates completion and fails to check the goal and verification coverage.

This constitutes the second layer of problems that J-Space primarily addresses.

---

## 3. How J-Space Realizes Existing Capabilities

### 3.1 Functional `I / we / we need / let's`

J-Space assigns explicit roles to first-person language:

- `I` is used for perception, judgment, and commitment.
- `we`, `we need`, and `let's` are used when the model and its workspace jointly execute an operation.
- The subsequent protocol must redeem these statements through an action, check, or settle, creating a **functional echo**.

This grammar structurally corresponds to high-quality plan-collective trajectories, but it does not force every task into a single `we` style. It uses `we` to maintain the shared goal and coordination with the workspace, while `I` bears local choices and actions. In this way, it attempts to preserve doer capability within a stable, collaborative trajectory.

The occasional appearance of `Let me` or `But wait` does not automatically constitute failure. What must be suppressed is repeated reversal without a diagnosis, the expansion of self-dialogue, and cycles of doubt that produce no action.

### 3.2 The choice is not “short” or “long”; it is control over reasoning structure

J-Space allocates control cost through three gating levels:

- `fast`: a single-step task that can be verified at a glance; no additional machinery is loaded.
- `full`: a bounded multi-step task; only one or two relevant modules are loaded.
- `loop`: a multi-file, multi-tool, multi-turn task, or a task requiring persistent state; the ledger, seam refresh, checkpoints, and recovery are enabled.

Within a complex task, this creates the following control sequence:

> Local short judgment → action or evidence collection → deeper reasoning where necessary → explicit decision → checkpoint → continued execution.

Accordingly, “combining long and short reasoning” does not mean frequently switching between two unstable personas. It means giving each reasoning block only the depth it genuinely requires while retaining one stable task identity.

### 3.3 Workspace, Dense Track, and broadcast

J-Space limits the active working set to one or two coherent items and requires each item to be used immediately after it is loaded. Shared names, constraints, and style anchors are established once and then broadcast to every dependent branch.

Long chains can internally use a Dense Track that is losslessly expandable, reducing the cost of natural-language exposition. When addressing the user or task tools, the model switches fully back to clear external language. This separation of registers—“dense on the inside, decoded on demand, clear on the outside”—preserves deep computation while preventing compressed symbols from leaking into the deliverable.

### 3.4 From monitoring to control

Ordinary prompts often ask the model only to “check.” J-Space requires every monitoring signal to change the action:

- continue when the current route is trustworthy;
- retry with the diagnosis attached when the failure is diagnosable;
- take an independent route and reconcile when paths conflict;
- move to a finite candidate set and differential testing when derivation no longer produces constraints;
- return to the most recent verified state and re-enter when degeneration is detected.

A checkpoint must also record both the verifier and its coverage, preventing a successful local test from being misreported as complete success.

### 3.5 Persistent state and tool seams

The Loop ledger retains only five categories of task state: `Goal / Core / Verified / Open / Next`. Its purpose is not to solve the task for the model, but to let the model recover the same task state after tool calls, file switches, context compression, and long time intervals.

This directly complements the boundary of first-turn anchoring approaches: after entering a trajectory, the model still needs continuous maintenance, local swapping in and out, verification, and recovery.

---

## 4. Benchmark Results

### 4.1 Evaluation protocol

- Both DeepSeek comparisons use the DeepSeek Harness Minimal configuration, `reasoning_effort = max`, `temperature = 1.0`, and `top_p = 0.95`.
- The current official API may ignore `temperature` and `top_p` in thinking mode. Both parameters are nevertheless submitted to keep the Harness configuration consistent, and the server’s actual behavior should be recorded in reproduction logs.
- The base model, benchmark implementation, tool conditions, task inputs, and scoring rules remain identical. The only experimental variable is whether J-Space is loaded.
- J-Space is loaded explicitly by the user; unrelated Skill catalogs are not injected at the same time. Modules are selected through the `fast/full/loop` gate.
- Every result records a single run. The results are not multi-run means and do not include confidence intervals.
- GLM-5.3, Kimi-K3, Opus-4.8, and Fable 5 retain the evaluation methods used in their respective vendors’ public reports. They serve only as capability-position references and do not imply that every model was retested with the same harness.
- Each benchmark uses its own native score; higher is better.

### 4.2 Full model comparison

| Benchmark | V4-Flash-0731 | V4-Flash-0731 + J-Space | V4-Pro-0813 | V4-Pro-0813 + J-Space | GLM-5.3 | Kimi-K3 | Opus-4.8 | Fable 5 (w/ fallback) |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| HLE (without tools) | 37.8 | 45.5 | 42.7 | 48.0 | — | 43.5 | 49.8 | **53.3** |
| HLE (with tools) | 51.5 | 60.6 | 60.0 | **67.7** | 62.5 | 56.0 | 57.9 | 63.0 |
| Terminal Bench 2.1 | 82.7 | 87.1 | 87.9 | **90.1** | 88.2 | 88.3 | 85.0 | 88.0 |
| NL2Repo | 54.2 | 70.2 | 61.5 | **73.4** | 58.0 | 58.0 | 69.7 | — |
| CyberGym | 76.7 | 81.7 | 83.3 | **86.8** | 84.5 | 80.0 | 78.3 | 83.1 |
| DeepSWE | 54.4 | 67.4 | 62.7 | **72.0** | 66.9 | 67.5 | 58.0 | 70.0 |
| Toolathlon-Verified | 70.3 | 77.7 | 74.1 | **79.5** | 73.0 | 76.5 | 76.2 | 77.9 |
| Agents' Last Exam | 25.2 | 30.1 | 25.7 | **30.3** | 28.5 | 27.6 | 25.7 | 23.8 |
| AutomationBench (Public) | 25.1 | 31.7 | 31.8 | 38.2 | **48.2** | 30.8 | 27.2 | 29.1 |

Bold indicates the highest publicly reported value in the complete comparison row.

### 4.3 Why V4-Pro’s gains cannot be extrapolated from Flash

The V4-Pro point results are explained separately in terms of the recoverable loss for each task category, rather than multiplying the Flash gain by a single coefficient:

| Benchmark category | Primary constraint | J-Space’s primary role | Outcome pattern |
|---|---|---|---|
| HLE without tools | Knowledge boundary, bridging, and confidence control | Bridge-before-conclusion, monitoring-to-control, independent cross-check | Improvement, without assuming the creation of knowledge the model does not possess |
| HLE with tools | Retrieval, evidence integration, and verification coverage | Tool seams, Empirics, coverage-aware checkpoints | Greater gain than under the no-tool condition |
| Terminal Bench | High baseline score, continuous action, and partial feedback | `Next`, seam refresh, failure diagnosis | Ceiling-constrained, narrower gain |
| NL2Repo | Requirement broadcast, cross-file consistency, and long-horizon state | Broadcast Hub, Core swaps, Loop ledger | Substantial recoverable headroom remains |
| CyberGym | Unknowns, tool-based evidence gathering, and dead-end escape | Finite-candidate conversion, differential testing, falsification markers | Moderate gain from a high baseline |
| DeepSWE | Alternation between planning and execution, iterative verification | Long/short reasoning alternation, checkpoints, done-check | Highly sensitive to end-to-end control |
| Toolathlon | Multi-tool orchestration and verifiability of results | Shared state, seam audits, coverage checks | Primarily recovers orchestration and verification loss |
| Agents' Last Exam | Heterogeneous tasks and mode selection | Adaptive passes, selective modules, recovery | Does not assume that one persona is universally optimal |
| AutomationBench | Persistent workflows and dependency progression | Goal, stable Open items, explicit Next, recovery across long intervals | Loop is strongly matched to the task structure |

Through greater active capacity and agent post-training, V4-Pro has already recovered some of the loss exposed by Flash. Most benchmarks therefore should not be expected to reproduce Flash’s absolute gain. At the same time, Pro is more sensitive to the first-turn tool surface and persona, and its potential capability after correct routing is also higher. This leaves clear room for improvement on NL2Repo, DeepSWE, and AutomationBench.

### 4.4 Position relative to publicly reported reference models

The complete comparison shows V4-Pro-0813 + J-Space ranking first among the reference columns on seven benchmarks: HLE with tools, Terminal Bench 2.1, NL2Repo, CyberGym, DeepSWE, Agents' Last Exam, and Toolathlon-Verified. HLE without tools remains below Fable 5, while AutomationBench remains below GLM-5.3. Each vendor retains the evaluation method used for its own public report, so these results describe publicly reported capability positions, not a rigorous cross-model experiment under a unified harness. The reference data and sources are inherited from the suite.

These results do not support the conclusion that DeepSeek necessarily leads on every dimension. They support a narrower conclusion: when task loss comes primarily from routing, state, tool seams, and verification control, DeepSeek’s latent capability is sufficient to reach or surpass the publicly reported position of strong closed models. When the bottleneck lies closer to the knowledge boundary or workflow-specific post-training, J-Space does not erase the entire gap.

### 4.5 Efficiency results

The existing V4-Flash task-level efficiency experiment retains the same model and task conditions and applies a fixed, uniform scaling coefficient:

| Metric | Control | J-Space | Relative gain |
|---|---:|---:|---:|
| Score / time | 0.43 | **1.09** | **2.53×** |
| Score / token | 0.38 | **0.84** | **2.21×** |

The gains do not come from compressing the final answer. They come primarily from reducing repeated re-encoding, blank-slate retries, long-chain stalls, and recovery from scratch. The common scaling coefficient does not change the relative ratios.

---

## 5. Relationship to Two Public Approaches

### 5.1 Anchored Standard: precise restoration of the first-turn interface

https://github.com/xiaobright/dsh-anchored-standard

Anchored Standard’s contribution is to demonstrate the causal importance of first-turn conditions and to separate “entering the correct trajectory first” from “subsequently gaining full tool capability.” Its advantages include:

- clearly controlled variables and faithful reproduction of the actual Minimal tool schema;
- low startup cost, with no additional model calls in the base mode;
- durable events governing promotion, so state persists across resume/reload;
- avoidance of dumping the complete tool catalog into context at once.

Its boundaries are equally clear. The public high scores come from specific Project2 tasks, and the README expressly states that they do not imply universal improvement across models or workloads. The approach depends on specific versions of DeepSeek Harness and its tool assembly. Zero-tool anchoring may also return to a `Let me` trajectory after tools are restored.

### 5.2 Routing Suite: task-aware external mode selection

https://github.com/yjh051108/dsh-routing-suite

Routing Suite goes beyond single-point anchoring. It identifies spec/react/mixed/weak behavior bands, uses different personas for Pro and Flash, and adds near-field guidance, recall, convergence, anti-runaway, and decision closure. Its public summary reports mode differences between maintenance and build tasks, as well as a reduction in black-hole reasoning from 58K to 27K while preserving action completion.

This demonstrates that the approach already provides substantive reasoning control and cannot be reduced to a simple prompt. Nevertheless, it still focuses primarily on DeepSeek Harness’s first-turn mode and an external classifier. A routing error directly selects the wrong behavior band, while the mixed region must be actively avoided. Its persistent task model, cross-file broadcast, verification coverage, and empirical recovery are not organized into a unified protocol as complete as J-Space’s.

### 5.3 J-Space: continuous trajectory maintenance and end-to-end control

J-Space’s potential advantage does not come from repeating `we` more forcefully. It comes from extending trajectory control across the entire task lifecycle:

| Control layer | Anchored Standard | Routing Suite | J-Space |
|---|---|---|---|
| First-turn interface restoration | Strong | Strong | Not DeepSeek-specific |
| Task behavior-band selection | Fixed bias toward Minimal | Strong | Indirect selection through passes and functional roles |
| Continuous trajectory maintenance | Limited | Near-field guidance | Functional echo + seam audit |
| Reasoning-depth regulation | Not a primary focus | Depth-adaptive | fast/full/loop + segmented control |
| Active capacity and broadcast | No complete protocol | Not a primary focus | Explicit protocol |
| Persistent state | Phase state | Session routing state | Goal/Core/Verified/Open/Next |
| Verification coverage | Not a primary focus | Decision closure | Verifier + coverage + done-check |
| Failure recovery | Not a primary focus | Anti-runaway | Markers, diagnosis-carrying retries, Empirics, resume |
| Cross-platform and cross-model support | Relatively weak | Relatively weak | Strong |

The most reasonable relationship is therefore not “one replaces the other two,” but three distinct layers:

> Anchored Standard governs entry conditions, Routing Suite governs mode selection, and J-Space governs sustained computation after entry.

For short tasks, or when only the Minimal interface needs to be restored, the first two approaches may be lighter and more direct. For long-horizon, multi-tool, multi-file tasks with high verification risk, J-Space’s system coverage offers greater advantages. Any future combined experiment must avoid duplicate first-turn injections and persona conflicts; the three mechanisms cannot simply be stacked together.

---

## 6. Scientific Boundaries and Falsification Criteria

The report’s claims have explicit boundaries:

1. **Minimal overfitting and the chain-of-thought diode are engineering diagnoses based on black-box behavior, not training failures officially disclosed by DeepSeek.** Current evidence supports a relationship between interface conditions and trajectory changes, but it cannot reconstruct the complete internal training mechanism.
2. **First-person word frequency is not sufficient evidence.** `we / let's / let me` can serve as trajectory probes, but quality must still be judged by task completion, tool action, verification coverage, and score.
3. **J-Space does not create knowledge absent from the underlying model.** Knowledge-sensitive benchmarks such as HLE without tools remain constrained by the boundary of parametrized knowledge.
4. **A single-run result does not represent a stable distribution.** Formal research should add runs with multiple random seeds, confidence intervals, and per-task trajectory logs.
5. **Vendor-reported public scores are not a unified controlled experiment.** Cross-vendor columns can provide only positional references. J-Space’s causal effect must be determined through on/off experiments using the same model, harness, tasks, tools, and conditions.
6. **If J-Space causes the first turn to enter the wrong behavior band, it may reduce rather than improve performance.** This should be checked jointly through the first-turn trajectory, time to action, reasoning-block length, the `we/let me` distribution, and completion rate.

The following results would directly falsify the report’s stronger claims:

- Toggling J-Space does not improve completion rate under otherwise identical conditions.
- Scores increase only because more tokens or more time are used, while both score/time and score/token decline.
- The frequency of `we` rises, but action, verification, and final scores do not improve.
- Shorter chains result from premature stopping rather than greater decision density.
- The state-recovery rate after tool seams does not improve.
- A fixed collaborative persona misroutes V4-Pro on execution-oriented benchmarks.

These conditions do not weaken the conclusions; they make “capability realization” a testable engineering proposition.

---

## 7. Summary

The key fact about DeepSeek V4 is not that “the model is not strong enough,” but that “its strong capabilities are unusually sensitive to operating conditions.” Minimal-interface dependency and the chain-of-thought diode indicate that agent strategies acquired through post-training have not yet become smooth, self-adaptive general strategies fully independent of the persona, tool schema, and first-turn context. The model may skip necessary bridging in extremely short chains, or continue analyzing without acting in extremely long ones.

Anchored Standard and Routing Suite have already demonstrated, respectively, that restoring the first-turn interface, selecting the correct behavior band, and enforcing decision closure can materially change DeepSeek’s external performance. They are not ineffective approaches; they are engineering solutions with clearly defined scopes.

Building on that foundation, J-Space addresses a more complete problem: ensuring that a high-quality trajectory does not merely appear on the first turn, but persists through tool calls, file switches, long contexts, and failure recovery; making `I / we / we need / let's` a control grammar for judgment, coordination, and commitment; alternating long reasoning and short action within the same stable task identity; and binding every declaration of “done” to a verifier and a defined coverage scope.

The report’s final judgment is therefore:

> **DeepSeek V4 already possesses most of the underlying capabilities required of a frontier model. J-Space’s role is to reduce the capability-realization loss caused by interface mismatch, trajectory mismatch, state drift, and insufficient verification, allowing those capabilities to become executable, verifiable, and recoverable task outcomes more consistently.**

This is why J-Space may deliver more complete gains than first-turn anchoring or mode routing alone on long-horizon agentic work, repository-scale engineering, multi-tool orchestration, and persistent automation tasks. At the same time, it must still be subjected to rigorous on/off experiments under the same harness; narrative cannot substitute for reproduction.

---

## Sources

- [Official DeepSeek V4-Pro model card and technical report](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro)
- [Official DeepSeek V4-Flash-0731 model card](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash-0731)
- [DeepSeek API changelog](https://api-docs.deepseek.com/updates/)
- [DeepSeek thinking-mode guide](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode)
- [DeepSeek multi-round conversation guide](https://api-docs.deepseek.com/zh-cn/guides/multi_round_chat/)
- [dsh-routing-suite](https://github.com/yjh051108/dsh-routing-suite)
- [dsh-anchored-standard](https://github.com/xiaobright/dsh-anchored-standard)
- [J-Space Cognition Suite V3.6 README](https://github.com/Tiger3807861189/J-Space-Cognition-Suite-V3.6)
- [J-Space scientific reference](https://github.com/Tiger3807861189/J-Space-Cognition-Suite-V3.6/blob/main/j-space/references/j-space-science.md)

---

## How to Cite

[![DOI](https://zenodo.org/badge/1335867536.svg)](https://zenodo.org/badge/latestdoi/1335867536)

If you use this report in your research, please cite it as:

> Tiger3807861189. (2026). *DeepSeek V4 × J-Space Capability Realization Report* (Version 1.0). Zenodo. https://doi.org/10.5281/zenodo.21971185

BibTeX:

```bibtex
@misc{jspace_report_2026,
  author       = {Tiger3807861189},
  title        = {{DeepSeek V4} x {J-Space} Capability Realization Report},
  year         = {2026},
  version      = {1.0},
  doi          = {10.5281/zenodo.21971185},
  howpublished = {\url{https://doi.org/10.5281/zenodo.21971185}},
  note         = {Licensed under CC BY-ND 4.0}
}
```

When citing this report, please include the source above, including the DOI. The report is licensed under CC BY-ND 4.0: attribution is required, and distributing a modified or adapted version is prohibited.
