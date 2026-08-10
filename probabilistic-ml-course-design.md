# Probabilistic Machine Learning — Course Design

*A rigorous, example-driven course, built from first principles.*

This is the architecture, not the lectures. It fixes the arc, the dependencies, the
running examples, and — for each concept — which medium (animation, live code, or
worked-by-hand derivation) actually teaches it. Lecture content gets produced module by
module once the presentation style is nailed down.

---

## Design principles

**Calibration.** The audience has strong mathematical maturity — real analysis,
linear algebra, functions as first-class objects — but is not steeped in *statistics
jargon*. So: be rigorous where the math earns it (treat a random variable as a function,
introduce measure only where it removes hand-waving), and define every statistical term
plainly the first time it appears. Never assume "unbiased," "sufficient," or "conjugate"
is known; always assume "function," "integral," and "vector space" are.

**Two running examples, threaded end to end.** Every module returns to the same two
objects rather than inventing a fresh toy each time:

- **QPS** — requests processed by a server in one second. Discrete. Carries the thread
  through counts → Poisson → exponential family → count regression (GLM).
- **Latency** — time for one HTTP request, with an *atom* at the timeout. Mixed
  (continuous below the cutoff, a point mass at it). Carries the thread through the
  unified CDF view → censoring → survival-style likelihoods → a latent-variable model of
  response time.

The payoff: concepts stop looking like disconnected machinery and start looking like
successive questions about *the same two things you already understand*.

**Medium heuristic.** Animation for anything geometric or about *change* (a
distribution deforming, a bound tightening, samples filling a region). Live code for
anything where seeing real numbers/tensors is the point (sampling, fitting, an EM loop
converging). Hand-derivation for anything where the *steps* are the lesson (deriving
EM as a lower bound, the ELBO, conjugacy). Most modules use all three.

---

## Part I — Foundations

### Module 1 — Probability and Random variables
**Objective.** Undo the coin/dice damage: a random variable is a function
`X : Ω → ℝ` that you *chose*, and the sample space is a modeling decision, not a
physical given. PMF, PDF, and CDF as three views, unified by the CDF — and use the
latency atom to force that unification early, before it becomes a special case students
fear.
**Concepts.** Outcome vs. sample space vs. support; `X` as a many-to-one function; the
information `X` discards; why symmetry-given distributions (dice) are the exception, not
the rule. PMF, PDF, CDF; the CDF as the object that always exists; the mixed
distribution (continuous part + atoms); expectation as an integral *against a measure*
(measure-theoretic view introduced here, lightly, precisely because the atom makes the
elementary view creak). Define plainly: "density," "mass," "expectation."
**Running thread.** QPS: one outcome ω is a full microsecond-level trace; `X` collapses
it to a count. Latency: same experiment, different `X`, hence a different sample space —
motivating "enlarge Ω when you add a variable." Latency is the star: a smooth density below the timeout with a
literal spike (atom) sitting on top at the cutoff. QPS gives the pure-PMF contrast.

### Module 3 — Several random variables at once
**Objective.** Joint, marginal, conditional; independence and *conditional*
independence (planted here as the seed for graphical models).
**Concepts.** Joint distributions; marginalization as "summing out"; conditioning;
independence vs. conditional independence; covariance; conditional expectation as a
projection (leans on the linear-algebra strength — `E[Y|X]` as the best `X`-measurable
predictor).
**Running thread.** Joint of (QPS, mean latency): they're dependent — high load raises
latency — so the joint doesn't factor, which is exactly what makes conditioning
interesting.
**Medium.** Hand-derivation (conditional expectation as projection).

---

## Part II — From probability to inference

### Module 4 — Common distributions and the exponential family
**Objective.** Stop enumerating distributions and see the organizing abstraction: the
exponential family, with conjugacy and sufficiency falling out of it.
**Concepts.** Bernoulli/Binomial, Poisson, Gaussian, Gamma, Beta; the exponential-family
form; natural parameters; sufficient statistics; conjugate priors. (Framing that lands
for a PL mind: the exponential family is an *interface* several distributions implement,
and conjugacy is a closure property of that interface.)
**Running thread.** QPS → Poisson, derived from a memoryless-arrivals argument;
latency's continuous part → Exponential/Gamma, same family. Show both as instances of
one form.
**Medium.** Hand-derivation (Poisson from first principles; exponential-family form).
Live code (fit Poisson to simulated QPS, watch the sufficient statistic be all you need).

### Module 5 — Estimation: MLE, MAP, and Bayes
**Objective.** Three answers to "what are the parameters?" — and an honest account of the
frequentist/Bayesian split as both philosophy and practice.
**Concepts.** Likelihood; MLE; the prior; MAP; the full posterior; point estimate vs.
distribution over parameters; bias, variance, consistency (defined plainly). Connect to
A/B testing / eval methodology: what a confidence interval does and doesn't claim vs. a
credible interval.
**Running thread.** Estimate the Poisson rate for QPS three ways; show MLE = MAP under a
flat prior, and the posterior narrowing as data accrues.
**Medium.** Animation (posterior tightening around the true rate as samples stream in).
Live code (the three estimators side by side).

### Module 6 — Regression, probabilistically
**Objective.** Recast familiar models as probabilistic ones — the "oh, *that's* what I
was doing" module for someone from ranking/recsys.
**Concepts.** Linear regression as Gaussian-noise MLE; ridge as a Gaussian prior (MAP);
logistic regression; generalized linear models; the link function. Count regression
(Poisson GLM) makes QPS a *predictable* quantity, not just a described one.
**Running thread.** Model QPS as a function of covariates via a Poisson GLM; model the
probability a request exceeds the timeout via logistic regression (the latency atom
becomes a label).
**Medium.** Hand-derivation (least squares = Gaussian MLE; ridge = MAP). Live code
(fit the Poisson GLM on synthetic load data).

---

## Part III — Latent structure and graphical models

### Module 7 — Latent-variable models and EM
**Objective.** Introduce variables you never observe, and derive EM as coordinate ascent
on a lower bound — deliberately, because that lower bound *is* the ELBO you'll reuse for
variational inference.
**Concepts.** Mixture models; the latent assignment variable; the EM algorithm derived as
maximizing a lower bound on the log-likelihood; probabilistic PCA / factor analysis.
**Running thread.** Latency is bimodal — cache hits vs. misses. Model it as a mixture with
a latent hit/miss variable; EM recovers the two modes without ever seeing the label.
**Medium.** Animation (EM iterations: responsibilities re-coloring points, components
sliding into place). Hand-derivation (the lower bound — set up so it visibly *becomes*
the ELBO next).

### Module 8 — Probabilistic graphical models
**Objective.** Read and write the conditional-independence structure of a model.
**Concepts.** Directed models (Bayes nets), undirected models (MRFs), plate notation,
d-separation, exact inference via message passing.
**Running thread.** Draw the QPS → latency → timeout chain as a small Bayes net; use
d-separation to reason about what observing latency tells you about load.
**Medium.** Animation (message passing as beliefs flowing along edges). Hand-drawn graphs,
built live.

---

## Part IV — Approximate inference (the practical core)

### Module 9 — Monte Carlo and MCMC
**Objective.** When you can't integrate, sample. Build the sampling toolkit and its
failure modes.
**Concepts.** Monte Carlo estimation; importance sampling and density ratios;
Metropolis–Hastings; Gibbs sampling; a visual intuition for Hamiltonian Monte Carlo;
diagnosing non-convergence.
**Running thread + connection.** Importance sampling is the natural hook to
*distribution shift* — reweighting samples from one distribution to answer questions
about another. That is precisely the failure mode in the imitation-learned-policy
research (a policy trained under one state distribution evaluated under a shifted one),
so this module can double as the probabilistic framing of that problem.
**Medium.** Animation (a chain exploring a density; importance weights resizing samples).
Live code (Metropolis–Hastings on the QPS-latency posterior).

### Module 10 — Variational inference and the VAE
**Objective.** Turn inference into optimization; arrive at the VAE as amortized VI.
**Concepts.** The ELBO (now recognized from Module 7); mean-field VI; the
reparameterization trick; amortization; the variational autoencoder.
**Connection.** Anchor the whole module in the conditional VAE from the policy-learning
work — the course meets the research here. The CVAE that generates action sequences is a
concrete, already-familiar instance of "decoder conditioned on context, latent trained by
the ELBO," which makes the abstract VI machinery immediately legible.
**Medium.** Hand-derivation (ELBO, reparameterization). Animation (the approximate
posterior deforming toward the true one as the bound tightens). Live code (a small VAE on
latency traces).

---

## Part V — Modern probabilistic ML

### Module 11 — Deep generative and Bayesian models
**Objective.** Survey the modern landscape as *variations on inference*, not a zoo.
**Concepts.** Normalizing flows (exact likelihood via change of variables); diffusion
models as learned denoising / score matching; Gaussian processes; Bayesian neural nets;
uncertainty quantification. Frame diffusion and flows against the VAE so they read as
answers to "how else can we make the latent-to-data map tractable?"
**Medium.** Animation (change of variables warping a density; the forward/reverse
diffusion process). Live code (a 1-D flow or diffusion on latency data).

### Module 12 — Decision theory and calibration (capstone)
**Objective.** Close the loop: a probabilistic model exists to support *decisions*, and
its probabilities have to be trustworthy.
**Concepts.** Bayesian decision theory; loss and risk; proper scoring rules;
calibration (reliability diagrams); a taste of bandits / active learning. Tie back
explicitly to evaluation methodology.
**Running thread.** Decide a timeout threshold from the latency model to trade tail
latency against error rate — a real decision, made with the full machinery the course
built.
**Medium.** Live code (calibration curves on a real-ish classifier). Animation (a
reliability diagram assembling; the risk surface over a decision threshold).

---

## Dependency structure

```
1 → 2 → 3 → 4 → 5 → 6
             ↓
             7 → 8
             ↓    ↓
             9 → 10 → 11 → 12
```

Modules 1–6 are a strict spine. Module 7's lower bound is a hard prerequisite for
Module 10's ELBO — teach them close together. Modules 11 and 12 can be reordered or
trimmed to taste without breaking anything upstream.

---

## Reference spine

Murphy, *Probabilistic Machine Learning* (Book 1: *Introduction*; Book 2: *Advanced
Topics*) tracks this arc closely and is the natural companion text. Bishop's *PRML* is
the alternative for Parts II–IV. Neither dictates the sequencing above — the running
examples do.

---

## Production notes

The most animation-worthy segments, in rough priority order (best return on the
Manim pipeline first): the many-to-one collapse (M1, done), the mixed density with the
atom (M2), posterior tightening (M5), EM iterations (M7), message passing (M8),
importance weights under shift (M9), the ELBO bound tightening (M10). These are where
motion teaches something a static figure can't. The derivations (M4, M6, M7, M10)
are better as build-it-by-hand segments than as pre-rendered animation. Estimation and
GLM fitting want live code with real output.
