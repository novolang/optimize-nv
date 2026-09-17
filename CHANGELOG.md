# Changelog

All notable changes to optimize-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `optstop` — the load-bearing interface. The stopping rule is one
  value a program builds once and names, rather than five optional
  arguments whose defaults were chosen by someone who did not know the
  caller's units; `check` catches a rule that could never stop where
  the rule is built. The result is an `OptReport` whose `OptStatus`
  has three arms — converged, budget exhausted, stalled — so a caller
  can tell a minimum from a time-out, which returning only a point
  never lets it do. The gradient option is a value too, and it carries
  the finite-difference step, because that step decides whether a
  numeric gradient is a gradient or noise.
- `optscalar` — Brent and golden-section over a bracket, with
  `is_bracket` stating the property both assume and `bracket_from`
  searching for one when the caller has a point instead of an
  interval.
- `optsimplex` — Nelder–Mead with the initial simplex as an argument.
  Every other implementation perturbs the starting point, usually at
  random; that would make a run irreproducible and would have put this
  package in `host`. `simplex_around` and `simplex_with_steps` build
  the deterministic ones, and the second exists because one step for
  coordinates in different units is too large in one and too small in
  the other.
- `optquasi` — BFGS, taking the objective, a gradient function and an
  `OptGradient` that says which of the two is used, so a call site
  shows it. `numeric_gradient` is exposed on its own, because checking
  an analytic gradient against a differenced one is how a wrong
  derivative gets found.
- `optleast` — Levenberg–Marquardt taking the RESIDUAL function rather
  than the sum of squares, since the residuals are what make the
  problem fast; a residual vector that changes length between points
  is refused with both lengths rather than silently fitted.
- `optfault` — seven refusals, every one about the CALL rather than
  about the function, and every one findable before the first
  evaluation.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  optimize-nv.<module>.<fn>`.
- **No constraints and no global methods.** Every method here is
  unconstrained and local. The global methods all need a source of
  random numbers, which a `core` package may not have.
- **No iteration trace.** A progress callback or a printed trace is
  `[io]` and this package's whole surface is `[]`. A caller that wants
  one wraps its own objective and keeps the effect on its own side.
- **No L-BFGS and no sparse least squares.** The linear algebra here
  is dense, which is right up to a few hundred parameters and wrong
  above that.
