# QANGA DreamerV3 Brake-Collapse Audit

## Executive decision

**Decision: revise and adopt the candidate correction, not as originally stated.** The core repair is directionally correct: remove boost/brake from the joint binary score-function term, keep ordinary REINFORCE for throttle and steering, compute binary counterfactual returns under `no_grad`, use the same detached return scale \(S\), retain the world model only provisionally, and start a fresh recovery lineage with actor and critics reset. The revision is that, because there are exactly **two** interacting binary coordinates, the intervention step should use **exact four-combination joint enumeration** of \((\text{boost}, \text{brake}) \in \{(-1,-1),(+1,-1),(-1,+1),(+1,+1)\}\), not two per-head estimators conditioned on the sampled value of the other binary. The per-head conditional estimator is unbiased for the learned-model objective, but the exact four-combination term is also unbiased, removes extra Monte Carlo variance from the “other binary” sample, and costs the same intervention-branch count per decision point. The candidate therefore survives as a **revised** recovery design, not as a literal keep-as-is decision. fileciteturn0file0 fileciteturn0file1

The documented failure signature is strong and specific. Around 2.3 million real environment steps and about 21,200 learner updates, the report records imagined brake-on around 0.953, executed brake-on typically 0.91–0.97, mapped brake mean about 0.905 with statewise standard deviation about 0.014, while continuous saturation remained very low. The same report also localizes the one-step ranking reversal to the continuation-weighted bootstrap product rather than immediate reward prediction, and explicitly states that the current probe does **not** isolate whether the decisive error sits in continuation, successor latent/dynamics, critic value, or some interaction inside \(\gamma \Delta(\hat c \hat V)\). Training longer is not a defensible answer on that evidence. fileciteturn0file1

The retained DreamerV3 backbone is also clear from the paper. DreamerV3 uses a recurrent RSSM, vector encoder/decoder, reward and continuation heads, bootstrapped \(\lambda\)-returns, percentile return normalization, and an actor loss based on detached normalized return advantage plus entropy. The paper’s critic recursion is the \(\lambda\)-return in Eq. 5, the actor loss is Eq. 6, and the return scale is the percentile EMA in Eq. 7; Table 4 fixes \(H=15\), \(\lambda=0.95\), discount horizon \(333\), actor entropy \(3\times10^{-4}\), and actor return normalization with lower limit 1. fileciteturn0file0

### Documented facts

| Item | Status | Evidence basis |
|---|---|---|
| Near state-independent brake collapse is real | Fact | QANGA report telemetry and probe summaries fileciteturn0file1 |
| DreamerV3 official actor objective is detached normalized REINFORCE plus entropy | Fact | DreamerV3 paper Eq. 6, Table 4 fileciteturn0file0 |
| One-step reversal enters through \(\gamma \Delta(\hat c \hat V)\), not immediate reward | Fact | Consolidated report diagnosis fileciteturn0file1 |
| Immediate causal truth in Unreal is not established by imagined counterfactuals | Fact | Consolidated report scope and gate design fileciteturn0file1 |

### Deductions

| Item | Status | Why it follows |
|---|---|---|
| Removing binary log-probs from the joint REINFORCE term is necessary if an exact binary expectation term is added | Deduction | Otherwise the binary coordinates are counted twice under the same estimated objective; the report itself warns about this duplication risk, and the detached-expectation form makes the coordinate-specific gradient explicit. fileciteturn0file1 |
| Joint four-combination enumeration is preferable to sampled-other conditional estimators | Deduction | With exactly two binaries, both approaches need four intervention branches, but full enumeration eliminates extra variance from the sampled other binary while preserving the same binary objective. Supported by the report’s exact Bernoulli-expectation analysis and by the problem structure in the user brief. fileciteturn0file1 |
| The repaired estimator is unbiased only for the **learned world-model objective**, not for causal Unreal truth | Deduction | DreamerV3 actor learning is defined on imagined trajectories, and the report repeatedly separates model-objective correctness from real-environment validity. fileciteturn0file0 fileciteturn0file1 |

### Unresolved hypotheses

The strongest unresolved question is not whether the score-function estimator is noisy. It is whether, after variance reduction and reset, the surviving learned target still points toward pathological braking because the world model, continuation head, reward head, critic, or the actual task reward genuinely prefers it. The documents do **not** prove that the critic alone is wrong, and they do **not** prove that Unreal itself would disagree with the calibrated model signal. That is exactly why the report requires held-out calibration, repeated-draw sign stability, and an authoritative paired Unreal intervention test when exact state restoration exists. fileciteturn0file1

## Mathematical audit

DreamerV3’s paper defines actor learning with detached normalized return advantage, entropy regularization, and imagination under a learned world model. The critic target is the bootstrapped \(\lambda\)-return \(R_t^\lambda = r_t + \gamma c_t \big((1-\lambda)v_t + \lambda R_{t+1}^\lambda\big)\), terminalized by the final value prediction, and the actor loss uses a detached \((R_t^\lambda - v_t)/\max(1,S)\) multiplier on \(\log \pi_\theta(a_t\mid s_t)\) with entropy added separately. That means any modified binary estimator must preserve three invariants if it is intended to optimize the same learned objective: identical pre-decision weighting, identical continuation semantics, and identical detached scaling by \(S\). fileciteturn0file0

### Exact binary gradient under unimix

For one binary head with logit \(z\), unimix \(u\), and mixed Bernoulli probability

\[
p(z) = (1-u)\sigma(z) + \frac{u}{2},
\qquad \sigma(z)=\frac{1}{1+e^{-z}},
\]

the derivative of \(p\) with respect to the logit is

\[
\frac{\partial p}{\partial z}
= (1-u)\sigma(z)\bigl(1-\sigma(z)\bigr).
\]

Using \(p=(1-u)\sigma + u/2\), this can also be written exactly as

\[
\frac{\partial p}{\partial z}
=
\frac{\bigl(p-\frac{u}{2}\bigr)\bigl(1-\frac{u}{2}-p\bigr)}{1-u}.
\]

That is the correct unimix-logit Jacobian. The report explicitly rejects the common but wrong substitute \(p(1-p)/(1-u)\), because that expression does not vanish at the mixed ceiling \(p\to 1-u/2\) and therefore overstates gradient near saturation. fileciteturn0file1

If \(Q_{\text{on}}\) and \(Q_{\text{off}}\) are detached action-values for forcing the executed binary action to \(+1\) and \(-1\), then the exact binary contribution to the **maximization** objective is

\[
J_{\text{bin}}(z)
=
p(z)\,Q_{\text{on}} + \bigl(1-p(z)\bigr)Q_{\text{off}},
\]

so

\[
\frac{\partial J_{\text{bin}}}{\partial z}
=
\frac{\partial p}{\partial z}\,\bigl(Q_{\text{on}}-Q_{\text{off}}\bigr).
\]

With actor loss written as a minimization and using Dreamer’s detached scale and pre-decision weight, the binary term for one head should therefore contribute

\[
\frac{\partial L_{\text{bin}}}{\partial z}
=
-\frac{w_t}{S}\,
\frac{\partial p}{\partial z}\,
\bigl(Q_{\text{on}}-Q_{\text{off}}\bigr),
\]

with \(w_t\), \(S\), \(Q_{\text{on}}\), and \(Q_{\text{off}}\) detached. That derivative is exact for the learned-model expectation represented by the supplied \(Q\)-values. fileciteturn0file0 fileciteturn0file1

### Interacting boost and brake heads

The candidate’s main open mathematical question is whether, when boost and brake interact, a per-head estimator that conditions on the sampled value of the other binary is unbiased for the original policy objective.

The answer is **yes**, under the factorized mixed policy described in the request and in the report’s mathematical setup. If \(b\) is boost, \(r\) is brake, and \(c\) denotes the sampled continuous actions, then for the brake logit \(z_r\),

\[
J
=
\mathbb E_{c,b,r}\bigl[Q(c,b,r)\bigr]
=
\mathbb E_{c,b}\Bigl[
p_r Q(c,b,1) + (1-p_r)Q(c,b,0)
\Bigr].
\]

Differentiating with respect to \(z_r\) gives

\[
\frac{\partial J}{\partial z_r}
=
\mathbb E_{c,b}\Bigl[
\frac{\partial p_r}{\partial z_r}
\bigl(Q(c,b,1)-Q(c,b,0)\bigr)
\Bigr].
\]

So if \(b\) is sampled from the current policy and the estimator averages over those samples, the conditional per-head form is unbiased for the **same learned-model objective**. Joint enumeration of all four binary combinations is therefore **not necessary for unbiasedness**. fileciteturn0file1

But unbiasedness is not the full decision criterion here. Because there are only two binary coordinates, exact joint enumeration of all four intervention-step combinations gives a strictly cleaner objective at the same intervention branch count. Define the detached values

\[
Q_{--},\;Q_{+-},\;Q_{-+},\;Q_{++}
\]

for \((\text{boost},\text{brake})=(-1,-1),(+1,-1),(-1,+1),(+1,+1)\), with continuous throttle/steering frozen to the sampled intervention-step values. Then the exact joint binary expectation is

\[
J_{\text{boost,brake}}
=
(1-p_b)(1-p_r)Q_{--}
+
p_b(1-p_r)Q_{+-}
+
(1-p_b)p_r Q_{-+}
+
p_b p_r Q_{++}.
\]

Its gradients are

\[
\frac{\partial J}{\partial z_r}
=
\frac{\partial p_r}{\partial z_r}
\Bigl[
p_b(Q_{++}-Q_{+-}) + (1-p_b)(Q_{-+}-Q_{--})
\Bigr],
\]

\[
\frac{\partial J}{\partial z_b}
=
\frac{\partial p_b}{\partial z_b}
\Bigl[
p_r(Q_{++}-Q_{-+}) + (1-p_r)(Q_{+-}-Q_{--})
\Bigr].
\]

This is the exact joint binary marginal for the learned-model objective, captures interaction **without** conditioning on a sampled other binary, and uses the same four intervention combinations that two separate per-head on/off evaluations would already require. My recommendation is therefore to **revise** the candidate from sampled-other conditional estimators to a single exact four-combination binary objective. The sampled-other form remains a correct fallback if implementation simplicity wins, but it is not the best design under the stated action space. Deduction supported by the report’s exact Bernoulli-expectation framework and the given two-binary structure. fileciteturn0file1

### What must be detached

The detach boundary is load-bearing. The DreamerV3 paper’s actor loss uses a stop-gradient on the normalized return term, and the consolidated QANGA report repeatedly warns that counterfactual returns must be computed under `no_grad`, with only the Bernoulli probability carrying actor gradient. The following tensors must therefore be detached for the binary term: the start state fed into learner-only actor evaluation, the intervention-step sampled continuous action values, all paired counterfactual returns \(Q_{\cdot}\), all \(\lambda\)-returns, current and successor value predictions used inside those returns, continuation predictions used inside those returns and weights, pre-decision weights \(w_t\), the return-normalizer scale \(S\), all batch offsets if any, and all replay/world-model outputs that define the branch target. The only differentiable quantities in the binary exact term should be the binary probabilities \(p_b\) and \(p_r\). Continuous head log-probabilities and analytic entropies remain differentiable in their own unchanged term. fileciteturn0file0 fileciteturn0file1

### Weights, continuation, termination, lambda returns, and normalization

Using pre-decision cumulative products of discount and predicted continuation is consistent with the report’s intended estimator and with DreamerV3’s imagined weighting structure, **provided** the weight used at decision \(t\) is formed **before** multiplying by the continuation generated by action \(a_t\). That detail matters twice. First, it preserves the intended state-occupancy weighting of the learned objective. Second, it prevents the same-step action from leaking into the common scalar weight, which would create avoidable estimator coupling and would also invalidate the report’s “batch offset is only conditionally harmless” argument. fileciteturn0file1

Predicted continuation and probabilistic termination do **not** invalidate the estimator mathematically as long as the returns and weights use the same continuation semantics and all of that is detached. They do, however, narrow the meaning of “correctness.” The estimator is unbiased for the **learned world-model objective with learned continuation**, not for physical Unreal truth. The paper itself defines the imagined objective through learned rewards, learned continuation, and critic bootstrap in imagination; the report explicitly distinguishes that from real-environment causality and therefore introduces external calibration gates. fileciteturn0file0 fileciteturn0file1

Detached percentile return normalization by the same positive scalar \(S=\max(1,\text{EMA}(P95-P5))\) also preserves the intended objective class. It rescales gradients but does not change action ordering inside a state as long as the same detached \(S\) is used for both the continuous REINFORCE term and the binary exact term. If the binary term used a different scale, the repaired actor would no longer optimize the same normalized objective as DreamerV3’s continuous coordinates. fileciteturn0file0 fileciteturn0file1

### Double counting

There are only two legitimate regularization pieces after the repair: continuous-head REINFORCE and entropy. Binary score-function terms must be removed from the joint log-prob sum if the binary exact expectation is added. Keeping both would count the same binary coordinate twice under the actor objective. By contrast, leaving **binary entropy** in place is **not** double counting, because entropy is a separate regularizer, already part of DreamerV3’s actor loss, and the report explicitly preserves it. The implementation rule is simple: binary log-probs out, binary entropy in once, exact binary expectation in once, same detached \(S\), same detached pre-decision \(w_t\). fileciteturn0file0 fileciteturn0file1

### Ranked failure modes

| Rank | Failure mode | Confidence | Falsifier |
|---|---|---:|---|
| Highest | Wrong learned action ranking enters through the continuation-weighted bootstrap product rather than immediate reward | High | Exported decomposition shows no durable sign issue in \(\Delta(\hat c\hat V)\) and paired real/model signs agree on calibrated states fileciteturn0file1 |
| High | Joint binary score-function variance plus saturation prevents recovery once collapse begins | High | Exact-joint binary expectation removes the binary sampling tier, but collapse still recurs with healthy support and calibrated heads fileciteturn0file1 |
| High | Brake-off support starvation in replay destabilizes off-branch values | High | Replay brake-off support rises above gate while repeated-draw sign instability remains or worsens fileciteturn0file1 |
| Medium | Critic-specific extrapolation is the dominant inner cause inside \(\gamma\Delta(\hat c\hat V)\) | Medium | Critic reset leaves the sign pathology unchanged while continuation/reward/dynamics ablations implicate other heads fileciteturn0file1 |
| Medium-low | The real reward itself prefers heavy braking even after model/critic repair | Medium-low | Exact or authoritative paired-state real tests show the repaired, calibrated model still agrees with non-completing brake-heavy behavior and route performance does not recover fileciteturn0file1 |

## CRN rollout design and compute

The report’s CRN idea is sound, but it must be implemented with sharper stream discipline than the candidate text implies. The required shared randomness is not generic “same seed.” It is a specific replay of identical sampled noise tensors at specific points in the imagined rollout, with independent state tensors per branch. The report already states that actor inference can consume explicit continuous Gaussian and binary uniform noise, while current training draws internally, and that RSSM imagined transitions currently draw categorical latent noise internally. That is the design gap the coding agent must close. fileciteturn0file1

### Streams that must be captured and replayed

For exact four-combination intervention at decision point \(t\), the required streams are:

| Stream | Needed at intervention step | Needed for future steps | Shape |
|---|---|---|---|
| Sampled continuous action values for throttle/steering | Yes, fixed from base rollout | No | \([M,2]\) |
| Actor Gaussian noise for throttle/steering | No, if continuous values are frozen at step \(t\) | Yes | \([k-1,M,2]\) |
| Actor Bernoulli uniforms for boost/brake | No, if both binaries are explicitly enumerated at step \(t\) | Yes | \([k-1,M,2]\) |
| RSSM categorical uniforms | Yes | Yes | \([k,M,L]\) or \([k,M,L,1]\) depending on sampler API |

Here \(M\) is the number of sampled decision points per learner update and \(L\) is the number of independent categorical latent variables used by the RSSM transition. If the implementation falls back to the per-head sampled-other estimator instead of the recommended four-combination exact term, then the intervention-step Bernoulli uniform for the **non-audited** binary must also be captured and replayed. Under the revised exact four-combination design, it is unnecessary because both boost and brake are forced at the intervention step. Deduction supported by the action structure in the request and the report’s CRN design notes. fileciteturn0file1

### RSSM categorical sampling API

The current problem is that the report says RSSM imagined transitions still sample categorical latent noise internally. That must be replaced, in learner-only code, by an overload that accepts explicit uniforms. The correct interface is conceptually

```text
next_state = rssm.image_step(state, action, latent_uniform=u_lat)
```

where `u_lat` provides one uniform draw per categorical latent variable for that step. If the latent logits or probabilities have shape \([M,L,C]\) for \(C\) classes, then `u_lat` should have shape \([M,L]\) or \([M,L,1]\), and the sampler should perform inverse-CDF selection along the class axis from the cumulative probabilities. The report already notes that a `categorical_sample_from_uniform` path exists conceptually; the coding task is to expose it in the imagined transition API. fileciteturn0file1

### Branch-state isolation

Noise should be shared. State should **not** be shared. That distinction is a common failure point.

The safe CUDA strategy is to materialize the four intervention combinations as an explicit combo dimension \(C=4\), then flatten \(C\times M\) into the batch dimension for purely functional rollout code. Each branch receives the same future Gaussian noise, the same future Bernoulli uniforms, and the same future RSSM uniforms, but its own copied recurrent state tensors and forced first-step executed boost/brake values. Avoid `expand()` without `clone()` on any mutable state container. If the recurrent state is a tree of tensors, create a new tree per combo or vectorize into a new tensor batch. Never let two branches point to the same tensor storage if any downstream op might mutate in place. This is directly aligned with the report’s warning to share noise without accidentally sharing mutable recurrent state. fileciteturn0file1

### Exact estimator and pseudocode with shapes

Below is the revised estimator. It keeps continuous REINFORCE unchanged and replaces the candidate’s two conditional per-head binary estimators with one exact joint binary term:

```python
# Main imagined rollout outputs
# feat        : [H, B, D]          detached pre-decision features/states
# a_cont      : [H, B, 2]          sampled throttle, steering executed values
# w           : [H, B]             detached pre-decision weights
# adv_cont    : [H, B]             detached continuous advantage
# scale       : [] or [1]          detached return-normalizer scale
# idx         : [M]                uniform sample over H*B decision points

# Gather sampled decision points
feat_m   = gather_flat(feat, idx)          # [M, D] detached
a_cont_m = gather_flat(a_cont, idx)        # [M, 2] detached
w_m      = gather_flat(w, idx)             # [M] detached

# Differentiable actor probabilities at sampled points
dist_m   = actor(feat_m)                   # learner-only forward
p_boost  = (1-u) * sigmoid(dist_m.boost_logit) + u/2   # [M]
p_brake  = (1-u) * sigmoid(dist_m.brake_logit) + u/2   # [M]

# CRN streams
u_lat_0      = sample_latent_uniform(M)                 # [M, L]
eps_cont_fut = sample_gaussian(k-1, M, 2)              # [k-1, M, 2]
u_bin_fut    = sample_uniform(k-1, M, 2)               # [k-1, M, 2]
u_lat_fut    = sample_latent_uniform(k-1, M, L)        # [k-1, M, L]

# Enumerate four binary combinations at intervention step
combo = tensor([
    [-1., -1.],   # Q--
    [+1., -1.],   # Q+-
    [-1., +1.],   # Q-+
    [+1., +1.],   # Q++
], device=device)                                         # [4, 2]

# Vectorize branches: C=4, flattened to batch C*M
state_0 = gather_states(flattened_states, idx)            # detached
state_c = repeat_state(state_0, C=4)                      # [4*M, ...] independent copies
a0_cont = repeat_interleave(a_cont_m, repeats=4, dim=0)   # [4*M, 2]
a0_bin  = combo[:, None, :].expand(4, M, 2).reshape(4*M, 2)

# First forced step
a0_full = pack_action(a0_cont, a0_bin)                    # [4*M, 4]
state   = rssm.image_step(state_c, a0_full, latent_uniform=repeat_u(u_lat_0, 4))

# Roll forward k-1 future steps under shared future noise, no_grad
# Collect reward, continuation, successor value each step, all detached
q = rollout_lambda_returns_no_grad(state, eps_cont_fut, u_bin_fut, u_lat_fut, k)  # [4, M]

q_mm, q_pm, q_mp, q_pp = q[0], q[1], q[2], q[3]          # each [M], detached

# Exact joint binary expectation
j_bin = (
    (1 - p_boost) * (1 - p_brake) * q_mm +
    p_boost       * (1 - p_brake) * q_pm +
    (1 - p_boost) * p_brake       * q_mp +
    p_boost       * p_brake       * q_pp
) / scale                                                 # [M]

# Continuous REINFORCE remains unchanged
logp_cont = dist_full.throttle.log_prob(a_throttle_det) + dist_full.steering.log_prob(a_steering_det)

objective = (
    (w * logp_cont * adv_cont).mean() +
    (w_m * j_bin).mean() +
    beta * continuous_entropy +
    beta * binary_entropy
)

actor_loss = -objective
```

All four \(q\)-values are computed under `no_grad`. Only \(p_{\text{boost}}\) and \(p_{\text{brake}}\) carry actor gradient in `j_bin`. Continuous REINFORCE still uses detached features, detached action samples, and detached normalized continuous advantage. This preserves the report’s intended detach boundary and the paper’s actor normalization semantics while improving the candidate’s binary interaction handling. fileciteturn0file0 fileciteturn0file1

### Falsifiable CRN tests

The documents already prescribe most of the right test logic. The minimal set is:

| Test | Pass condition | What it falsifies if it fails |
|---|---|---|
| Stream identity | Every replayed noise tensor in paired branches is bitwise identical | The implementation is not actually using CRN fileciteturn0file1 |
| Duplicate-branch determinism | Same state, same forced action, same noise, run twice → identical \(Q\) | State mutation or hidden nondeterminism fileciteturn0file1 |
| Shared-vs-independent variance | On fixed sampled states, \(\mathrm{Var}_{\text{shared}}/\mathrm{Var}_{\text{indep}}\) has one-sided 95% upper bound \(<1\) | CRN provides no measurable variance reduction fileciteturn0file1 |
| Other-binary Monte Carlo elimination | Four-combo exact term shows lower repeated-draw variance than sampled-other per-head term on the same states | The revision to exact joint enumeration produced no practical gain |

### Compute-cost estimate and lower-cost alternatives

The report’s work formula is the right starting point: with two binary heads and two branches per head, or equivalently one four-combination joint enumeration, the extra rollout cost is \(4kM\) RSSM transitions for \(M\) sampled decision points and counterfactual horizon \(k\). Main imagination cost is \(HB\), where \(H=15\) and \(B\) is the number of imagined decision points in the base actor rollout. To cap total transition work at about \(3\times\) the main rollout, require

\[
4kM \le 2HB
\quad\Longrightarrow\quad
M \le \frac{HB}{2k}.
\]

That exact bound appears in the report’s implementation outline. fileciteturn0file1

If the local imagination batch follows DreamerV3’s default sequence geometry of \(16\times 64\) start states, then \(B\approx 1024\) and \(H=15\), so the main imagined rollout is about \(15{,}360\) model transitions per learner update. Under the revised exact four-combination design:

| \(k\) | \(M\) | Extra transitions \(4kM\) | Total as multiple of main |
|---:|---:|---:|---:|
| 15 | 64  | 3,840  | 1.25× |
| 15 | 128 | 7,680  | 1.50× |
| 15 | 256 | 15,360 | 2.00× |
| 15 | 512 | 30,720 | 3.00× |
| 5  | 128 | 2,560  | 1.17× |
| 5  | 256 | 5,120  | 1.33× |
| 5  | 512 | 10,240 | 1.67× |

That gives a practical RTX 3080 starting budget: **\(k=15\), \(M=128\)** as the default, with **\(M=256\)** allowed if learner throughput remains above the regression gate. This keeps the estimator exact for the sampled decision-point mean and avoids assuming unlimited throughput. The report’s live gate already says measured learner throughput below 50% of pre-fix is a hard engineering stop, and below 80% is a regression. fileciteturn0file1

Lower-cost alternatives that preserve correctness are limited. Uniformly reducing \(M\) preserves unbiasedness for the sampled decision-point mean. Reducing \(k\) to 5 preserves estimator correctness for the \(k\)-step target but changes the target itself, so it is a **target ablation**, not a free compute trick. Dropping from exact four-combo to sampled-other per-head estimators also preserves unbiasedness for the learned objective, but it **adds variance** and therefore should be treated as a fallback, not the preferred implementation. fileciteturn0file1

## Recovery lineage

The report is right that this must be a fresh, versioned recovery lineage, and it is also right that the shared actor trunk forces a **full actor reset**, not a binary-head-only reset. The retained-reset decision should be driven by whether the state encodes supervised knowledge on real data, bootstrapped self-referential targets, or merely lineage-specific optimizer/scheduler residue. fileciteturn0file1

### Whitelist and reset table

| State item | Decision | Reason |
|---|---|---|
| RSSM, encoder, decoder parameters | Retain provisionally | Supervised/model-learning state from replay; still subject to calibration gates fileciteturn0file1 |
| Reward head parameters | Retain provisionally | Real-target supervised, but low EV means retain behind calibration only fileciteturn0file1 |
| Continuation head parameters | Retain provisionally | Real-target supervised, but explicitly implicated inside \(\gamma\Delta(\hat c\hat V)\) and must be calibrated separately fileciteturn0file1 |
| Full actor parameters | Reset | Shared MLP trunk received collapsed updates; no clean binary-only checkpoint boundary fileciteturn0file1 |
| Fast value parameters | Reset | Bootstrapped in collapsed imagination fileciteturn0file1 |
| Slow value parameters | Reset | Derived from fast value and would re-import old critic state fileciteturn0file1 |
| Actor optimizer | Reset | Moments encode collapsed direction; fresh lineage required fileciteturn0file1 |
| Value optimizer | Reset | Same reason as actor optimizer fileciteturn0file1 |
| World-model optimizer | **Reset** | Revision to the candidate: moments are lineage-specific, not inference-critical, and retaining them adds stale-curvature/schema-coupling risk without preserving any unique supervised knowledge |
| Replay buffer | Retain | Truthful environment record; contains scarce brake-off support that should not be destroyed fileciteturn0file1 |
| Return-normalizer state | Retain | Positive detached scale does not flip ordering; report explicitly retains it fileciteturn0file0 fileciteturn0file1 |
| Pending learner debt / backlog | Reset to zero | Prevent immediate burst of stale catch-up updates against collapsed-distribution replay |
| Learner-side RNG lineage | Reset with new master seed | Fresh recovery lineage and reproducible CRN streams |
| Env-worker recurrent hidden states | Reset | Ephemeral runtime state, not checkpoint knowledge |
| Curriculum descriptor that is externally authoritative | Retain | User objective says preserve authoritative QAI route |
| Curriculum counters derived from old policy success/failure | Reset or version separately | Old performance must not auto-advance or auto-block the new lineage |
| Historical total counters | Retain as metadata only | Needed for auditability |
| Scheduler counters used for debt, annealing, or recovery logic | Reset to lineage-local zero | Avoids inherited scheduling effects |

The main revision above is the **world-model optimizer**. The attached report leaves room for retaining it in skeleton pseudocode, but that is not the cleanest recovery choice. The optimizer state is not part of the agent’s supervised knowledge, is not required for production inference, and creates unnecessary resume incompatibility. Resetting it preserves the retained world-model parameters while making the new lineage cleaner and easier to reason about. That is a design deduction from the report’s own whitelist logic, not a claim directly stated in the documents. fileciteturn0file1

### Replay debt, counters, and RNG lineage

The user explicitly asked to challenge replay debt, counters, and RNG state. The correct recovery behavior is:

A fresh lineage must **not** inherit pending learner debt. If the learner has accumulated updates “owed” under the previous collapsed actor/critic schedule, spending that debt immediately after reset would train the new actor/critics against the old replay mix before fresh brake-off-rich post-reset data arrives. So backlog goes to zero at the lineage fork, and any replay-ratio scheduling restarts from the new lineage counters. Historical totals may still be stored for dashboards, but they must not drive recovery scheduling. This is consistent with the report’s requirement for a fresh schema, explicit whitelist, and new lineage ownership of gate state. fileciteturn0file1

RNG also needs explicit lineage handling. The new recovery run should start from a new master seed; learner-side actor-noise streams, RSSM uniform streams, and any replay subsampling RNG should derive from that seed and be serialized under the new checkpoint format. Old RNG state should not be resumed, because it belongs to the old policy/critic lineage and because deterministic CRN replay now becomes a first-class learner feature. fileciteturn0file1

### Resume incompatibility and production compatibility

The report requires explicit incompatibility diagnostics, and that is the correct operational stance. Ordinary resume from the collapsed lineage must fail loudly if the checkpoint does not declare the new recovery schema version, whitelist contents, dual counter semantics, and fresh optimizer ownership. Silent “best effort” restore is wrong here. The recovery loader should emit a structured incompatibility report covering missing retained keys, unexpected actor/critic keys, optimizer ownership mismatch, and absent lineage metadata. fileciteturn0file1

At the same time, production recurrent inference and checkpoint/export compatibility should be preserved by **keeping the runtime actor/RSSM signatures intact**. The new explicit-noise actor and RSSM APIs are learner-only overloads. The standard inference path used by PIE should still accept ordinary observations and recurrent state without any CRN plumbing. The exported actor and retained world model remain the same modules; only the training-time recovery loader and learner-time imagined branching interfaces change. This is consistent with the report’s separation between present implementation, learner-only CRN integration, and production path compatibility. fileciteturn0file1

## Implementation contract

The separate coding agent should not be asked to “figure out the approach.” It needs a narrow contract with hard invariants.

### Minimal implementation contract

| Contract item | Mandatory behavior |
|---|---|
| Binary estimator | Replace binary REINFORCE with one exact four-combination binary expectation term at sampled decision points |
| Continuous estimator | Leave throttle/steering REINFORCE unchanged |
| Intervention step | Freeze sampled throttle/steering; enumerate four executed \((\text{boost},\text{brake})\) combinations as \(\pm1\) values |
| Future rollout randomness | Replay identical future Gaussian, Bernoulli, and RSSM latent uniforms across all four branches |
| CRN integration | Add learner-only explicit-noise path to actor and RSSM image step |
| Detach boundary | Only \(p_{\text{boost}}\) and \(p_{\text{brake}}\) from the exact binary term carry actor gradient |
| Critic source | Use fast/current value for current and successor value targets; slow value remains regularizer only |
| Scaling | Divide binary exact term by the same detached return scale \(S\) used by continuous advantage |
| Weighting | Use the same detached pre-decision weights \(w_t\) as the continuous actor term |
| Recovery loader | New schema, explicit whitelist, actor/value reset, backlog zeroed, lineage counters reset, historical counters archived |
| Gates | Binary exact term stays disabled until replay support, reward/continuation calibration, repeated-draw stability, and T9 logic pass |

Everything above is implementation-ready and comes directly from the report’s core design, except for the one substantive revision: exact four-combination binary enumeration replaces sampled-other per-head enumeration. fileciteturn0file1

### Focused CPU and CUDA test matrix

| Platform | Test | Pass criterion |
|---|---|---|
| CPU float64 | Unimix derivative grid | \(\partial p/\partial z\) matches closed form to tight tolerance across \(z\in[-12,12]\) fileciteturn0file1 |
| CPU float64 | Exact binary gradient sign/scale | Autograd on exact binary term matches analytic gradient with shared \(w/S\) factor |
| CPU | Zero-signal test | If all four \(Q\) combinations equal, binary gradient is exactly zero |
| CPU | Double-counting mutant | Reintroducing binary log-probs doubles binary gradient in the test mutant; shipped build has zero binary-log-prob contribution |
| CPU | Detach audit | No world-model/reward/critic grad leakage after actor backward |
| CUDA | CRN stream identity | Paired branches replay identical stored noise tensors bitwise |
| CUDA | Branch determinism | Duplicate same branch with same state/action/noise gives same \(Q\) |
| CUDA | Shared-vs-independent variance | Shared-noise paired difference variance is statistically lower than independent-noise variance |
| CUDA | Throughput profile | \(M=128,k=15\) stays above 80% of pre-fix learner throughput target; below 50% is hard stop fileciteturn0file1 |
| CUDA + PIE integration | Recovery loader roundtrip | New checkpoint schema restores retained modules and resets forbidden modules exactly |

## PIE smoke gates

The attached report already contains the right gate philosophy: binary learning must be gated by support, calibration, and stability instead of being trusted by default. The smoke run should therefore be short, narrow, and quantitative. fileciteturn0file1

### Short smoke protocol

Run a **50k-step PIE smoke** from the fresh recovery lineage with binary exact-term updates initially **closed**. Collect fresh replay, train the retained world model and reset actor/critics under the new lineage rules, and open binary exact-term learning only if the support/calibration gates below pass. Keep the actor binary unimix at \(u=0.05\) throughout this smoke. Do not decay on wall clock. fileciteturn0file1

### Hard stop and go thresholds

| Gate | Go threshold | Hard stop |
|---|---:|---:|
| Replay support for both binary outcomes | Brake-off replay fraction \(\ge 5\%\) and rising before binary exact-term opens | \(<2\%\) after reset window or executed brake-off below the hard floor implied by unimix |
| Reward-head calibration | Held-out EV \(\ge 0.3\) before gate opens | EV \(<0.1\) or non-finite fileciteturn0file1 |
| Continuation calibration | Brier score better than constant baseline and action-stratified calibration gap \(<0.05\) | Worse than constant baseline or non-finite fileciteturn0file1 |
| Repeated-draw counterfactual sign stability | \(>80\%\) for non-near-ties, e.g. \(|\Delta Q/S|\ge 0.005\) | \(\le 80\%\) keeps binary training closed fileciteturn0file1 |
| Binary state dependence | Brake-probability statewise std \(>0.05\) before decaying unimix | After checkpoint 1, mean brake-on \(>0.95\) or \(<0.03\) with std \(<0.03\) for two consecutive checkpoints fileciteturn0file1 |
| Brake/boost collapse or inversion | Neither binary pinned near deterministic globally | Logit \(z\ge 4\) or \(z\le -4\) with low dispersion after first checkpoint fileciteturn0file1 |
| NaN / non-finite params or actions | None | Any occurrence is immediate stop |
| Learner throughput | \(\ge 80\%\) of pre-fix baseline | \(<50\%\) of pre-fix baseline fileciteturn0file1 |
| Health loss and real navigation behavior | No regression in damage or low-health terminations; stuck terminations trend down | Health regression or route behavior clearly degrades fileciteturn0file1 |

If exact Unreal state restoration exists, add the authoritative T9 gate from the report: on restored paired real states, model-vs-real brake-sign agreement should exceed 60% raw with one-sided 95% Wilson lower bound above 50%. If exact restoration does **not** exist, record that T9 is unavailable and keep the lineage labeled “model-causal unresolved.” That distinction matters. fileciteturn0file1

### Acceptance threshold for continuing beyond smoke

Proceed to the longer recovery run only if all of the following hold at two consecutive smoke checkpoints: no non-finite issues, no global binary pinning, brake-off replay support above 5% and rising, reward and continuation gates passing, repeated-draw sign stability above 80%, and learner throughput within the engineering budget. If those pass, keep \(u=0.05\) until the report’s unimix-exit set also passes, including statewise brake dispersion \(>0.05\). Decay \(u\) from 0.05 to 0.01 only on those measured conditions, not on elapsed steps alone. fileciteturn0file1

## Recommended next action

Implement the learner-only **exact four-combination binary counterfactual module** and the **versioned recovery loader with reset actor/critics, reset world-model optimizer, zeroed learner debt, and explicit CRN-capable RSSM sampling**, then run the CPU/CUDA gate suite before any new PIE training.