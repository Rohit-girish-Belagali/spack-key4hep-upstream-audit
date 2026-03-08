# Action Plan: Prioritizing 50 Packages for Upstreaming

Given a backlog of 50 key4hep-spack packages to upstream into Spack builtin,
I would prioritize using four criteria applied in sequence:

**1. Dependency Graph Traversal.** Build the full dependency DAG across all 50
packages. Identify *leaf nodes* — packages with zero key4hep-only dependencies.
These can be upstreamed immediately without blockers. Topological sorting
reveals the natural submission order: leaves first, then packages whose only
key4hep deps are already-upstreamed leaves, and so on layer by layer.

**2. Blast-Radius Scoring.** For each package, count how many *other* packages
in the set depend on it (reverse dependency count). High blast-radius packages
(e.g., `k4fwcore`, `edm4hep`) unblock the most downstream work. Among packages
at the same topological depth, prioritize higher blast-radius first.

**3. Recipe Readiness Filtering.** Score each recipe's upstream-readiness:
does it have `maintainers()`, `license()`, proper version checksums, no
external repo imports (like `Ilcsoftpackage`), `cmake` build dep, and a test
method? Recipes scoring 6/6 go first; others need fixes before submission.

**4. Topological PR Submission Order.** Combine the above into a ranked queue:
submit ready leaf packages with highest blast-radius first, batch 3–5 PRs per
wave, wait for merge, then advance to the next topological layer. This
minimizes reviewer fatigue while maximizing unblocked downstream packages
per merge cycle.
