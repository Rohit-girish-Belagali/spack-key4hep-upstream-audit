# Spack Upstreaming Audit — Key4HEP Packages

> **GSoC 2026 Qualification Task: "The Upstream Readiness Audit"**
>
> Audit of three key4hep-spack packages (`k4fwcore`, `k4geo`, `k4simdelphes`)
> for readiness to be upstreamed into the Spack builtin repository.

---

## Project Overview

The [key4hep-spack](https://github.com/key4hep/key4hep-spack) repository
contains Spack package recipes for the Key4HEP software ecosystem — a
framework for future high-energy physics (HEP) collider experiments. These
recipes currently live in an external Spack repository and are not part of
Spack's builtin package collection.

This project audits three representative packages to assess their upstream
readiness, identifies specific issues blocking acceptance, provides corrected
recipes, and outlines a strategy for upstreaming the broader key4hep-spack
ecosystem.

---

## Deliverables

| File | Description |
|------|-------------|
| [`audit_report.md`](audit_report.md) | **Part 1** — Repository mining & gap analysis. Real `spack audit` output, manual critique of each recipe with line-level issues, before/after code fixes, and summary table. |
| [`dependency_analysis.txt`](dependency_analysis.txt) | **Part 2** — Dependency graph analysis. Full package list, static recipe dep analysis, blocker identification table, and recommended topological upstreaming order. |
| [`dependency_analysis.png`](dependency_analysis.png) | **Part 2** — Rendered dependency graph of k4fwcore (generated from `spack graph --dot`). |
| [`packages/k4fwcore/package.py`](packages/k4fwcore/package.py) | **Part 3** — Corrected recipe for k4fwcore. |
| [`packages/k4geo/package.py`](packages/k4geo/package.py) | **Part 3** — Corrected recipe for k4geo. |
| [`packages/k4simdelphes/package.py`](packages/k4simdelphes/package.py) | **Part 3** — Corrected recipe for k4simdelphes. |
| [`mock_pr.md`](mock_pr.md) | **Part 4** — Mock pull request for k4geo following Spack PR conventions. |
| [`action_plan.md`](action_plan.md) | **Part 5** — Prioritization strategy for upstreaming 50 packages. |

---

## Key Findings

### Packages Audited

| Package | In Builtin? | Key Issues Found |
|---------|:-----------:|------------------|
| **k4fwcore** | ❌ No | Missing `maintainers()`, missing `license()`, inherits `Ilcsoftpackage` (k4-repo-only mixin), missing `depends_on("cmake", type="build")`, no test method |
| **k4geo** | ❌ No | Missing `license()`, unconstrained `dd4hep`/`lcio` deps, f-string cmake args, missing cmake build dep, debug `print(self)` |
| **k4simdelphes** | ❌ No | Missing `license()`, inherits `Ilcsoftpackage`, missing cmake build dep, f-string/`.format()` cmake args, missing explicit `root` dep, no test method |

### Critical Blockers for Upstreaming

1. **`Ilcsoftpackage` mixin** — Imported from `spack.pkg.k4.key4hep_stack`,
   which only exists in the key4hep-spack repo. Must be removed entirely.
2. **`k4fwcore`** — Blocks `k4simdelphes` (when `+framework`). Must be
   upstreamed before `k4simdelphes`.
3. **`k4gen`** — Blocks `k4simdelphes` (when `+integration_tests`). Not part
   of this audit but must be handled.

### Recommended Upstreaming Order

```
k4geo → k4fwcore → k4gen → k4simdelphes
```

---

## Reproduction Commands

```bash
# 1. Install Spack
git clone --depth=1 https://github.com/spack/spack.git ~/spack
source ~/spack/share/spack/setup-env.sh

# 2. Clone key4hep-spack and register as a Spack repo
git clone --depth=1 https://github.com/key4hep/key4hep-spack.git ~/key4hep-spack
spack repo add ~/key4hep-spack
spack repo list

# 3. Confirm packages are absent from builtin
spack repo remove k4
for pkg in k4fwcore k4geo k4simdelphes; do
  spack info $pkg 2>&1 | head -3
done
spack repo add ~/key4hep-spack

# 4. Run spack audit on each package
for pkg in k4fwcore k4geo k4simdelphes; do
  spack audit packages $pkg
done

# 5. Manual inspection grep commands
for pkg in k4fwcore k4geo k4simdelphes; do
  echo "=== $pkg ==="
  grep -n "maintainers" ~/key4hep-spack/packages/$pkg/package.py || echo "MISSING"
  grep -n "license"     ~/key4hep-spack/packages/$pkg/package.py || echo "MISSING"
  grep -n "depends_on.*cmake" ~/key4hep-spack/packages/$pkg/package.py || echo "MISSING"
  grep -n "Ilcsoftpackage" ~/key4hep-spack/packages/$pkg/package.py || echo "not used"
done

# 6. Generate dependency graph
spack graph k4fwcore > dependency_graph_raw.txt
spack graph --dot k4fwcore > k4fwcore.dot
dot -Tpng k4fwcore.dot -o dependency_analysis.png

# 7. Install flake8 and validate corrected recipes
pip3 install flake8
python3 -m flake8 packages/k4fwcore/package.py \
                  packages/k4geo/package.py \
                  packages/k4simdelphes/package.py
```

---

## Validation

### flake8 Configuration

Spack's own `.flake8` configuration uses:
- `max-line-length = 99`
- `per-file-ignores` for `F403`, `F405`, `F821` on package files

Our `.flake8` matches these conventions:
```ini
[flake8]
ignore = F403,F405
max-line-length = 99
```

### flake8 Output (All Three Corrected Recipes)

```
$ python3 -m flake8 packages/k4fwcore/package.py \
                    packages/k4geo/package.py \
                    packages/k4simdelphes/package.py

$ echo $?
0
```

**Result: All three corrected recipes pass flake8 with zero errors.** ✅

> **Note on F403/F405:** Spack recipes use `from spack.package import *` by
> design — this is the official pattern documented in the
> [Spack Packaging Guide](https://spack.readthedocs.io/en/latest/packaging_guide.html).
> Spack's own CI ignores F403/F405 for package files. Our flake8 config
> mirrors this convention.

> **Note on E501 at max-line-length=79:** When run with the strict PEP 8
> default of 79 characters, sha256 hash lines trigger E501 (82 chars = 8
> spaces indent + `sha256="` + 64-char hash + `",`). This is unavoidable
> for sha256 checksums and Spack's own codebase uses `max-line-length = 99`.

---

## Tools & Versions

| Tool | Version |
|------|---------|
| Spack | develop (latest, shallow clone 2026-03-09) |
| key4hep-spack | main (shallow clone 2026-03-09) |
| Python | 3.9 (macOS system) |
| flake8 | 7.3.0 |
| Graphviz (dot) | Homebrew install |
| OS | macOS Big Sur (aarch64) |

---

*Generated on 2026-03-09 for GSoC 2026 Qualification Task.*
