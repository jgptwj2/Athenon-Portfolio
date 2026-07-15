# Athenon

R&D project I've been building that combines physics simulation, 3D
geometry generation, and applied AI. The full implementation and
product concept are private — this is just an overview of the
engineering side, for anyone curious about the technical work.

---

## Simulation work

I don't trust a simulation result just because it looks reasonable.
Every solver here got checked against something known before it was
allowed anywhere near real output — a textbook formula, a published
table, an independently derived closed-form answer.

- 1D beam finite-element analysis, checked against Euler-Bernoulli
  beam theory: came out to 0.000% error.
- 2D meshed finite-element analysis, checked against closed-form
  theory: around 6%, depending on mesh density.
- Closed-form plate-bending (Navier series) for two different load
  cases, checked against published deflection coefficients: 0.06% and
  0.01% error.
- Gear tooth bending and contact stress, checked against published
  engineering tables.
- An industrial FEA solver (CalculiX) driven by a fully automated
  meshing pipeline, checked against the same beam benchmark: got it to
  within 1-2% once mesh density was actually under control.

## Bugs I actually hit and had to dig out of

**Shear locking.** First pass at the finite elements gave deflections
40-95% below the right answer. Wasted time suspecting the boundary
conditions before realizing it was the element order itself — linear
elements just can't represent bending correctly. Switching to
quadratic elements fixed it outright.

**Mesh format mismatch.** A mesher and a solver I was gluing together
disagreed on how they number the nodes of a 10-node tetrahedron. Tried
fixing it by hand twice with a signed-volume orientation check —
flipped every single element on the second attempt and it made zero
difference, which told me the problem wasn't per-element orientation
at all. Gave up hand-deriving it and routed the conversion through a
dedicated mesh library instead, which handled it correctly.

**Parser choking on its own numbers.** After the above fix, the solver
started crashing on node coordinates — turned out the conversion tool
was writing 16-digit scientific notation, and the (older, Fortran-based)
solver has a hard field-width limit it silently exceeds. Only showed up
on negative coordinates, because the minus sign was the extra character
that pushed it over. Fixed by writing coordinates myself in plain
fixed-decimal instead of trusting the library's formatting.

**Mesh too coarse without knowing it.** A slender test shape, meshed
with default auto-sizing, only got 2 elements through its thinnest
dimension and under-predicted deflection by something like 44%. Now I
size the mesh off the smallest bounding dimension of whatever geometry
comes in, instead of trusting a mesher's default judgment.

**Geometry that wouldn't stay one solid piece.** Features placed to
exactly touch (not overlap) sometimes left visible gaps, and in one
case a cut operation split a solid into disconnected chunks. Fixed by
always giving generated features a small deliberate overlap before any
boolean operation.

## Also built

A first pass at automatically figuring out boundary conditions (what's
fixed, where the load goes) directly from raw mesh geometry, with no
hardcoded dimensions — tested it against a couple of differently
proportioned, differently loaded shapes and it held up.

## Stack

Python, FastAPI, Next.js/React/TypeScript, Three.js, build123d
(OpenCASCADE), scikit-fem, Gmsh, CalculiX, Claude.

## How I work

Nothing gets trusted until it's checked against a known answer.
Regression tests catch whole categories of bugs automatically instead
of relying on me noticing by eye. And I'd rather the system say "I'm
not sure" honestly than confidently give a wrong number.

---

Still actively building this. Happy to talk through any of it in more
detail if you're curious.
