# learn-probml

Self-study notes on probabilistic ML, built from first principles, growing into a tutorial/course. This file carries context from earlier chat sessions so you don't have to rediscover it.

I am developing a course on Probablistic ML. You can see the course outline in @probabilistic-ml-course-design.md. Help me build this course module by module. As we discuss stuff, keep your responses to the point. If there are additional points you want to make, things you want to emphasize, etc. mention that in a single sentence, and I'll dig into it if needed. Of course if what I am stating is completely off, then do correct me.

## Audience and register

- **Primary audience: my future self.** Secondary: LinkedIn followers who may follow along (I've posted about building this).
- **Be terse.** Skip definitional hand-holding for things I already know. Do expand the derivations that were hard-won — future-me will have forgotten the reasoning, not the vocabulary.
- I have strong mathematical maturity (real analysis, linear algebra, measure theory basics, type theory / category theory from a PL background) but I am **not steeped in statistics jargon**. Define statistical terms plainly on first use; assume "function," "integral," "vector space," "pushforward" are known.
- Background: Senior Staff ML Engineer (generative AI training infra, large-scale ranking and recsys), robotics/RL product leadership, prior founder. So production-systems and ML examples land; textbook coin/dice examples do not.
- **I am a visual learner.** Prefer a figure over paragraphs wherever a concept can be shown. Figures are matplotlib, saved as PNG next to the markdown.
- **Push back directly on errors.** Don't soften, don't pad with praise. Wrong numbers, crossed justifications, and over-strong claims should be called out plainly. This has been the most valuable part of the process.

## Files

- `md/1_probability.md` — Modules 1 & 2: experiment, sample space, event space, probability measure, random variables, support vs. range, distributions (PMF / PDF / CDF), expectation and variance.
- `md/2_joint_dist.md` — Module 3: conditional probability, joint / marginal / conditional distributions, independence, conditional independence, conditional expectation, total probability, Bayes.

Figures use relative paths, so PNGs must sit in the same directory as the markdown.

## Core conceptual commitments

These are settled — don't relitigate them, build on them.

- **A random variable is a chosen function** `X : Ω → ℝ`. The sample space is a modeling decision, not a physical given. Dice are the degenerate case where `X` is the identity, which is exactly what makes them a bad first example.
- **The distribution of `X` is the pushforward** `P_X = P ∘ X⁻¹`, a measure on ℝ. PMF, PDF, and CDF are three descriptions of that one measure, differing in the reference measure (counting / Lebesgue / none). This is the organizing idea of Module 2.
- **The CDF is the object that always exists**, which is why it's the right tool for the mixed case: an atom is just a jump discontinuity.
- **The timeout atom belongs to the recorded variable** `min(T, c)`, produced by right-**censoring**. The latent latency `T` has no point mass at `c`. Keep this distinction sharp; `E[T]` computed from censored data is biased low.
- **Conditioning = restrict then renormalize.** The divisor `1/P(B)` is forced, not chosen: preserve the survivors' relative ratios, require them to sum to 1, solve for the constant. `P'(E) = P(E ∩ B) / P(B)`, and `P'(·) = P(· | B)` is a full probability measure in its own right.
- **Independence in a table = rank 1**, i.e. every column is a scalar multiple of the same vector (the outer product `p_X p_Yᵀ`). Columns being *identical* is only the special case where the conditioning marginal is uniform — do not present that as what independence means.
- **Independence is a property of the model, not of a dataset.** An empirical table never factors exactly; sampling noise guarantees wobble. Rare cells are relatively noisiest.
- **Marginalizing out a common cause destroys factorization.** Two things that are conditionally independent given a shared driver look dependent once the driver is summed away. This is the payoff of the conditional-independence section.

## Running example — the SaaS backend

Two microservices, **Ads** (recommends ads) and **Newsfeed** (assembles the feed), on separate infrastructure, not calling each other. Full request-level trace telemetry: timestamp, client IP, user agent, service, content-length, HTTP status, user ID, request ID, session ID, plus three begin/end timestamp pairs (frontend total, Ads, Newsfeed) for durations. Timeout is a **frontend app-server deadline** returning 504 — so the HTTP status field doubles as the censoring indicator.

**Open decision:** whether to put a Frontend service in front that fans out to both Ads and Newsfeed on each user request. Recommended yes — it makes QPS unambiguous, makes the common cause (frontend traffic) measurable, and makes "they don't call each other" architectural rather than asserted. Caveat to state if adopted: the frontend must call them in parallel with independent connection pools, or head-of-line blocking couples them and conditional independence breaks.

**Three sample spaces, deliberately kept separate.** Which variables can be defined depends on the unit of observation — this is the "Ω is a modeling choice" claim with teeth, and it's worth stating explicitly rather than working around.

| Unit              | `ω` is                      | Variables that live here                                     |
| ----------------- | --------------------------- | ------------------------------------------------------------ |
| Request           | one request's trace         | latency `T` (mixed, atom at `c`), status class, content-length |
| 1-second window   | all requests in that second | `X` = QPS, `Y` = distinct sessions (`Y ≤ X`, and `Y = 0` iff `X = 0`), `Z` = total bytes, `W = Z/X`, error fraction, mean/p99 latency, composite "perf" |
| Visitor / session | one visitor                 | A/B assignment `V`, device type `C`, clicked-CTA outcome     |

Per-request quantities are **not** indexable functions of a window `ω` (there's no `i` in `ω`, and how many exist depends on `ω`). Refer to them collectively — the multiset of latencies/sizes in the window — and define window variables as summaries of that collection.

### Which example teaches which concept

- **Many-to-one collapse** (M1): one `ω` (a full microsecond trace) → one integer QPS.
- **Mixed distribution / CDF** (M2): request latency with its censoring atom at the timeout. Three-figure set: discrete PMF/CDF (QPS), continuous PDF/CDF (height), mixed density/CDF (latency).
- **Joint / marginal / conditional** (M3): `X` and `Y` on the window space.
- **Discrete × continuous joint** (M3): QPS `X` against window mean latency `L̄`. Visual signature is a stack of scaled density curves, not a grid; summing out the discrete variable yields a **mixture** (forward pointer to Module 7).
- **Marginal independence** (M3): A/B test — assignment `V` vs. a pre-treatment covariate `C` (device type). Independent *by construction*, since the randomizer has no access to user attributes. Framed as a **CTA button test**: control in prod plus several color variants (font and position held constant, so a win is attributable). Randomize per **visitor**, not per request. Traffic split 50/20/15/15 as a staged ramp, giving a non-uniform `p_V` — which avoids the uniform-marginal coincidence that would make the columns identical. The same scenario yields three roles: covariate → independence required for validity; guardrail (status class, bounce rate) → independence hoped for; target metric (CTR) → dependence hoped for. Same table, same test, three verdicts.
- **Conditional independence** (M3): `Perf_Ads ⊥ Perf_Newsfeed | traffic`. Marginally dependent (shared demand driver), conditionally independent given total traffic. Requires no direct A→B path — state that as an assumption.
- **Deterministic dependence**: composite "perf" is a function of its inputs, so it's maximally dependent on them. Good for the collapse story, useless for independence.

## Notation conventions

- `Ω` sample space, `ω` outcome, `𝓕` event space, `P` the measure.
- `∩` for named events (`P(overloaded ∩ peak)`); comma for random-variable conditions (`P(X = x, Y = y)`). They mean the same thing; the comma hides the `∩` between the two implied events.
- `p(x, y)` joint PMF, `p_X` marginal, `p_{Y|X}(y | x)` conditional.
- I write "Lets" without the apostrophe. Match my voice; don't normalize it.

## Pending edits

**`md/2_joint_dist.md`**

- The `X = 2` conditional row divides by `0.30`; it must be `p_X(2) = 0.35`. Correct values are `4/7 ≈ 0.57` and `3/7 ≈ 0.43`. Tripwire: a conditional row summing past 1 (the current one sums to 1.17) means the divisor was too small.
- The blockquote justifying the `0.15 / 0.25` choice has crossed subsets: the first bullet bounds by `P(peak)` so it needs `overloaded ∩ peak ⊆ peak`, not `⊆ overloaded` (that's what the *second* bullet uses). Also state the bound as `≤` not `<`; fix two lowercase `p(peak)` in denominators; "3x the overall rate" rather than "3x more"; and note that `P(overloaded) = 0.2` never enters the conditional calculation — it's only the baseline for the contrast.
- Tighten the support claim to `Y = 0` exactly when `X = 0` (drop the looser `0 ≤ y ≤ x`); the tables and the regenerated heatmap already use the strict form.
- Replace the QPS-vs-latency conditional-independence example with Ads/Newsfeed-given-traffic. The old one is circular: "load" and QPS are nearly the same variable, and queueing gives a direct QPS→latency edge anyway.
- Append the multivariate section (chain rule for 3+ variables, `k!` orderings, the 3-way joint as stacked slices, the marginal lattice, conditional independence shortening the chain with parameter counting, and marginalizing the common cause breaking factorization), then reposition it.

**`md/1_probability.md`**

- The timeout is now a frontend server-side deadline, so the lower bound `l` can no longer be motivated by client distance and the cold-start TCP handshake — server duration excludes the network. Either rewrite `l` as minimum server processing time, or keep both a client-observed latency (with its distance floor) and an FE-measured duration, and note that the gap between them is the network time.
- Ensure the atom/censoring distinction is stated where the timeout is introduced, not only later.

## Deferred to later modules

- `W = Z/X` (mean bytes per request) → **Module 5**: it's a sample mean, so `Var(W | X = x) = σ²/x`. Mean-independent of `X` but *not* independent — the conditional mean is flat while the spread shrinks. Use it to introduce sampling distribution, standard error, and the LLN from inside the running example.
- **χ² test of independence** → Module 5: the tool that says whether an empirical table's departure from rank-1 is just noise. In A/B terms it's simultaneously the covariate-balance check and the outcome analysis.
- **Mixture form** of the discrete×continuous marginal → Module 7 (EM un-blends it).
- **Fork / chain / collider** structures → Module 8, which is where graphs earn their place: with three variables there are already several distinct structures and prose stops keeping up.
- **Importance sampling as distribution shift** → Module 9.
- **CVAE for action sequences** as the concrete anchor for the ELBO → Module 10.