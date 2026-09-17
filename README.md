# optimize-nv

Local minimisation finds a point where a function takes its smallest
value in some neighbourhood, by evaluating the function and stepping
downhill. This package brings five of the standard methods to
novo-lang: Nelder–Mead, BFGS, Brent's method, golden-section search and
Levenberg–Marquardt least squares. They are the same five
[scipy.optimize](https://docs.scipy.org/doc/scipy/reference/optimize.html)
and [argmin](https://argmin-rs.org/) are built around.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

The function being minimised is the **objective**. It takes a point —
one number for the one-dimensional methods, a list of numbers for the
others — and answers one number. A method starts somewhere, evaluates
the objective, and moves; an **iteration** is one such move, and an
**evaluation** is one call to the objective. The two counts are not the
same, and when the objective is a simulation it is the second that
costs.

A **local** minimum is a point with no lower point nearby. None of
these methods finds a **global** minimum — the lowest point anywhere —
and none of them can tell you whether the one it found is global. A
function with several valleys gives a different answer from each
starting point, and that is a property of the problem rather than a
defect of the method.

A **gradient** is the list of the objective's partial derivatives: how
fast it changes in each coordinate. A method that has one can pick a
direction rather than searching for it, which is why the
gradient-using methods converge in tens of iterations where the others
take thousands. A gradient can be supplied by the caller or
approximated by evaluating the objective at nearby points, which is
called a **finite difference**.

A **least-squares** problem is a special shape: a model with
parameters, a set of observations, and a **residual** for each
observation — what the model predicts minus what was measured. The
objective is the sum of the squares of the residuals. Knowing that
shape lets a method use the derivatives of the residuals in place of
the second derivatives of the sum, which is what makes
Levenberg–Marquardt much faster on a fit than a general method.

**Stopping** is a decision, not an event. An iteration can stop because
a tolerance was met, because a budget ran out, or because it stopped
making progress, and those three mean different things about the point
it returns. This package makes the caller state the rule and reports
which part of it fired.

## Install

```
novo pkg add optimize-nv
```

## Example

```novo
use optsimplex
use optstop

// The objective: Rosenbrock's function, whose minimum is 0 at (1, 1).
fn rosenbrock(x: [Float]) -> Float
    let a = 1.0 - x[0]
    let b = x[1] - x[0] * x[0]
    a * a + 100.0 * b * b

fn main() [io]
    // When to stop: at most 2000 iterations, or when the simplex gets
    // smaller than a ten-thousandth in every coordinate.
    let rule = optstop.with_point_tolerance(optstop.budgeted(2000), 0.0001)

    // The starting simplex: the point (-1.2, 1.0) and one vertex per
    // coordinate, half a unit away. It is an argument, so two runs
    // give the same answer.
    match optsimplex.simplex_around([-1.2, 1.0], 0.5)
        Err(e) => println(e.message())
        Ok(start) =>
            match optsimplex.nelder_mead(rosenbrock, start, optsimplex.classic_moves(), rule)
                Err(e) => println(e.message())
                Ok(r) =>
                    // The status is read before the point: a point
                    // from an exhausted budget is not a minimum.
                    println("${optstop.status_name(r.status)} after ${r.iterations} iterations")
                    println("x = ${r.point[0]}, y = ${r.point[1]}, f = ${r.value}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: optimize-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `optfault` | Every way a call is refused before the first evaluation, and the numbers that refused it. |
| `optstop` | The stopping rule, the gradient option, the three-armed status and the report a run answers. |
| `optscalar` | Brent's method and golden-section search in one dimension, and the bracket they work on. |
| `optsimplex` | Nelder–Mead, its four move coefficients, and the builders for an initial simplex. |
| `optquasi` | BFGS, the Wolfe line-search constants, and the numeric gradient on its own. |
| `optleast` | Levenberg–Marquardt least squares, its damping schedule, and the Jacobian on its own. |

## How to choose an entry point

**One variable, and you have an interval: `optscalar.brent`.** It fits
a parabola through three points and takes the golden-section step when
that would not help, so it is fast on a smooth function and cannot be
worse than golden-section on a rough one. Use
`optscalar.golden_section` instead when the objective is noisy or when
the evaluation count must be predictable, and `optscalar.bracket_from`
first when you have a starting point rather than an interval.

**Several variables and no derivative: `optsimplex.nelder_mead`.** It
uses function values only, so it works on a simulation, a table lookup
or anything else with nothing to differentiate. It converges slowly and
it can stall, and the report says when it did.

**Several variables and a gradient: `optquasi.bfgs`.** It builds an
approximation to the curvature out of the gradients it has seen, and on
a smooth problem it is much faster than Nelder–Mead. Use
`optquasi.bfgs_numeric` when there is no gradient function to hand, and
know that a numeric gradient over a noisy objective is noise.

**Fitting a model to data: `optleast.levenberg_marquardt`.** Hand it
the residual function, not the sum of squares. A general minimiser
given the sum will work and will be throwing away the structure that
makes the fit fast.

## The rules a user needs

1. **Read the status before the point.** `OptBudgetExhausted` means the
   iteration stopped early and the point is the best one so far, not a
   minimum. `OptStalled` means it stopped making progress. Only
   `OptConverged` means a tolerance was met.
2. **A stopping rule with no budget and no tolerance never stops.**
   `optstop.check` catches it where the rule is built. Call it once, at
   start-up.
3. **`gradient_tolerance` is ignored by the methods that have no
   gradient.** Nelder–Mead, Brent and golden-section test the point and
   value tolerances only. Setting it on a rule handed to them does
   nothing, and nothing fails.
4. **The initial simplex is an argument.** `optsimplex.simplex_around`
   builds the usual one from a point and a step. Build the vertices
   yourself when the coordinates have different units, because one step
   for all of them is too large in one and too small in the other.
5. **The numeric gradient's step is yours.** Too large and the
   difference is not the derivative; too small and it is the objective's
   own noise divided by a small number. There is no step that is right
   for every problem's units, which is why none is chosen for you.
6. **`optleast` takes the residuals, not their sum**, and the residual
   function must answer the same number of residuals at every point.
   One that does not is refused with both lengths, because the
   alternative is a silently wrong fit.
7. **The report's `value` for a least-squares fit is the sum of the
   squares**, not the root mean square and not the residual vector.
8. **Every method finds a local minimum.** From a different starting
   point you may get a different answer, and none of these reports can
   tell you which is lower than everything else.

## What is not included

- **Constraints.** No bounds on the variables, no linear or nonlinear
  constraints, no penalty or barrier machinery. Every method here is
  unconstrained. A bounded problem can often be transformed into an
  unconstrained one by the caller, and a general constrained solver is
  a different package.
- **Global optimisation** — simulated annealing, differential
  evolution, basin hopping, particle swarms. Every one of them needs a
  source of random numbers, which a `core` package may not have; they
  belong in a package that takes a generator as an argument or in a
  `host` one.
- **An iteration trace or a progress callback.** Printing, logging or
  calling back on every iteration is `[io]`, and this package's whole
  surface is `[]`. A caller that wants a trace wraps its own objective
  in a counting closure and keeps the effect on its own side, where it
  is visible.
- **Derivative-free trust-region methods** (COBYLA, BOBYQA) and
  **conjugate gradient**. They are the next three to add and none of
  them changes a signature here.
- **Sparse or large-scale least squares.** The damped normal equations
  Levenberg–Marquardt solves are dense here, which is right up to a few
  hundred parameters and wrong above that. L-BFGS, which is the
  large-scale answer for the general problem, is not here either.

## Related packages

- **`ndarray-nv`** is N-dimensional arrays and the linear algebra over
  them. This package works on plain `[Float]` and `[[Float]]` so that a
  caller with neither a matrix library nor a wish for one can use it;
  the two compose by converting at the boundary.
- **`stats-nv`** fits distributions and computes summaries. A maximum
  likelihood fit is a minimisation of a negative log-likelihood, which
  is what this package would do underneath it.

## Tests

The suite is written against the signatures and is red by
construction: every assertion reaches a `todo()`. Its oracles are the
functions every optimisation suite is checked against and whose answers
are exact: `f(x) = (x - 2)^2` has its minimum 0 at x = 2; the sphere
function has its minimum 0 at the origin; Rosenbrock's function has its
minimum 0 at (1, 1) at the bottom of a curved valley; and the constant
model fitted to the observations 1, 2 and 3 has its least-squares
solution at their mean, 2, with a residual sum of squares of 2. The
suite asserts the status and the counts as much as the point, because
that is what the typed report is for.

## Implementation status

| Module | Declared | Implemented |
| --- | --- | --- |
| `optfault` | 7 variants, `message` | no |
| `optstop` | 11 functions | no |
| `optscalar` | 4 functions | no |
| `optsimplex` | 5 functions | no |
| `optquasi` | 5 functions | no |
| `optleast` | 5 functions | no |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
