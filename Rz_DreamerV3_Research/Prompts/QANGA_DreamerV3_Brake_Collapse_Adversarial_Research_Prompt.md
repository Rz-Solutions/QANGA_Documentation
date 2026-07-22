# QANGA DreamerV3 brake-collapse adversarial research task

You receive exactly two attached documents:

1. the original DreamerV3 paper;
2. the consolidated QANGA brake-collapse research report.

You do not have repository, checkpoint, log, Unreal Editor, or local filesystem access. Treat implementation facts quoted below and code excerpts contained in the attached report as the available evidence. Do not claim to have verified anything outside those two documents. Your job is a theory-and-design audit that produces an implementation-ready decision for a separate coding agent.

## System context

QANGA trains a vector-only DreamerV3 policy for pickup hovercraft navigation in Unreal Engine PIE:

- 42 normalized scalar observations;
- four actions: continuous throttle and steering, plus binary boost and brake represented as executed values -1 or +1;
- 16 asynchronous environment agents;
- recurrent categorical RSSM, vector encoder/decoder, reward and continuation heads, actor, fast value network, slow value network, replay sequence sampling, and imagined lambda-return training;
- one shared actor MLP trunk with two continuous heads and two Bernoulli heads;
- binary action probability uses unimix:
  `p(z) = (1-u) * sigmoid(z) + u/2`;
- the current actor objective applies one scalar imagined advantage to the joint log-probability of all four action coordinates;
- actor inference can consume explicit continuous Gaussian and binary uniform noise, but the current training sampler draws internally;
- RSSM imagined transitions currently draw categorical latent noise internally;
- the online value network supplies current and successor values; the slow value network is a regularization target, not the main bootstrap;
- imagined rollout weights are pre-decision cumulative products of discount and predicted continuation;
- return normalization is a detached percentile EMA;
- Unreal interprets boost/brake as `action > 0`, and recorded executed binary actions are exactly -1 or +1.

The objective hierarchy is binding:

1. preserve vehicle health;
2. complete the authoritative QAI route;
3. reduce completion time.

Collision-streaming and unreliable-physics pauses produce no transition and advance no training clock. Training is visible in PIE. The game thread may never wait on inference, replay, learning, checkpointing, or GPU synchronization.

## Current failure evidence

At roughly 2.3 million real environment steps and 21,200 learner updates:

- imagined brake-on rate was about 0.953;
- executed brake-on rate was usually 0.91-0.97;
- mapped policy brake mean was about 0.905;
- statewise standard deviation of mapped brake mean was about 0.014;
- continuous action saturation was only about 0.001-0.003;
- imagined boost-on rate was about 0.33;
- reward explained variance was about 0.36;
- replay value explained variance was about 0.81;
- imagined return was about -17.7;
- environment throughput was about 130-145 steps/s;
- learner throughput was about 1.3 updates/s on an RTX 3080;
- there were isolated short-band successes, but no reliable current-band or full-route completion.

This matches the report's near state-independent brake-collapse signature. Simply training longer is not an acceptable answer.

## Candidate correction to audit

The current proposed recovery is:

1. Remove boost/brake log-probabilities from the joint REINFORCE term.
2. Keep ordinary REINFORCE for continuous throttle and steering.
3. For each binary head, optimize the exact Bernoulli expectation
   `p * Q_on + (1-p) * Q_off`,
   with only `p` carrying actor gradient.
4. Estimate `Q_on - Q_off` with paired common-random-number imagined rollouts using the same:
   - starting recurrent state;
   - other action coordinates at the intervention step;
   - future actor Gaussian noise;
   - future actor Bernoulli uniforms;
   - future RSSM categorical uniforms.
5. Force only the audited binary coordinate to +1 or -1 at the intervention step.
6. Use k-step lambda-return counterfactuals, initially around k=15, while explicitly checking k in {1, 5, 15}.
7. Normalize the binary potential by the same detached return scale as the continuous policy advantage.
8. Compute every counterfactual return under no-gradient; actor gradient may flow only through the corresponding Bernoulli probability.
9. Use actor-specific binary unimix 0.05 during recovery while leaving RSSM categorical unimix at 0.01. Decay toward 0.01 only after explicit evidence, not on elapsed steps alone.
10. Start a fresh, versioned recovery lineage. Retain only explicitly whitelisted world-model, replay, return-normalizer, curriculum, and counter state. Reset the full shared actor, fast value, slow value, actor optimizer, and value optimizer.
11. Keep reward logic unchanged unless evidence proves the real objective itself prefers near-continuous braking.

For the two interacting binary coordinates, the intended estimator conditions on the sampled value of the other binary coordinate and averages over sampled rollout states/actions. Determine whether this is unbiased for the original policy objective, and whether joint enumeration of all four boost/brake combinations is necessary.

## Required adversarial analysis

### Mathematical correctness

- Derive the exact gradient with respect to the Bernoulli logit under the stated unimix parameterization.
- Prove or refute the conditional per-head estimator when boost and brake interact.
- State exactly which tensors must be detached.
- Determine whether pre-decision weights, predicted continuation, probabilistic termination, lambda returns, and return normalization preserve the intended objective.
- Identify any double-counting if binary score-function terms or binary entropy remain.
- Distinguish an unbiased estimator of the learned world-model objective from causal truth in Unreal.

### Common-random-number construction

- Specify the exact random streams that must be captured and replayed.
- Explain how categorical RSSM sampling should accept an explicit uniform tensor.
- Explain how the main rollout and paired branches should share noise without accidentally sharing mutable recurrent state.
- Give falsifiable tests for stream identity and variance reduction.
- Quantify expected compute/memory cost and propose a bounded sampling budget suitable for an RTX 3080 without assuming unlimited throughput.

### Recovery lineage

- Decide what must be retained versus reset.
- Challenge retaining replay, world-model optimizer, return normalizer, environment/learner counters, RNG state, pending replay debt, and curriculum state.
- Explain how to avoid an immediate burst of stale learner debt after resetting actor and critics.
- Require explicit incompatibility diagnostics for ordinary resume from the collapsed lineage.
- Preserve production recurrent inference and checkpoint/export compatibility.

### Calibration and safety gates

Define hard stop/go thresholds for:

- replay support for both binary outcomes;
- reward-head calibration;
- continuation calibration versus a constant baseline;
- repeated-draw counterfactual sign stability;
- binary policy state dependence;
- brake/boost collapse or inversion;
- NaN/non-finite parameters and actions;
- learner throughput regression;
- health loss and real navigation behavior.

Do not use generic prescriptions such as “increase entropy,” “train longer,” or “penalize brake” unless you provide a derivation, a falsifier, and a reason the proposed estimator correction is insufficient.

## Required output

Produce one consolidated Markdown report:

1. Executive decision: retain, revise, or reject the candidate correction.
2. Mathematical proof and any counterexample.
3. Exact estimator and CRN pseudocode with tensor shapes and detach boundaries.
4. Ranked failure modes with confidence and falsifiers.
5. Explicit recovery-state whitelist/reset table, including replay debt and RNG lineage.
6. Minimal implementation contract for the coding agent.
7. Focused CPU/CUDA test matrix.
8. Short PIE smoke protocol and quantitative abort/acceptance gates.
9. Compute-cost estimate and lower-cost alternatives that preserve correctness.
10. Citations to the attached paper/report by section, equation, table, or page.

Separate facts supplied by the documents from deductions and unresolved hypotheses. End with one recommended next implementation action, not another open-ended research phase.
