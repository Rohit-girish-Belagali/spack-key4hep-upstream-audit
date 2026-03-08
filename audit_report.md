# Upstream Readiness Audit Report

## 1. Environment Setup

### Repository Registration
```bash
$ git clone --depth=1 https://github.com/spack/spack.git ~/spack
$ source ~/spack/share/spack/setup-env.sh
$ git clone --depth=1 https://github.com/key4hep/key4hep-spack.git ~/key4hep-spack
$ spack repo add ~/key4hep-spack
==> Added repo to config with name 'k4'.

$ spack repo list
[+] k4         v1.0    /Users/rohitgirishbelagali/key4hep-spack
 -  builtin            https://github.com/spack/spack-packages.git
```

### Confirming Packages Are Absent from Builtin

With `k4` repo removed, all three packages are not found in Spack's builtin repository:

```
=== k4fwcore ===
==> Error: Package 'spack_repo.builtin.packages.k4fwcore.package' not found
    in repository '...' Use 'spack create' to create a new package.

=== k4geo ===
==> Error: Package 'spack_repo.builtin.packages.k4geo.package' not found
    in repository '...' Use 'spack create' to create a new package.

=== k4simdelphes ===
==> Error: Package 'spack_repo.builtin.packages.k4simdelphes.package' not found
    in repository '...' Use 'spack create' to create a new package.
```

**All three packages exist only in the `key4hep-spack` repo and are candidates for upstreaming.**

---

## 2. Automated `spack audit` Output

```bash
$ spack audit packages k4fwcore
PKG-DIRECTIVES: passed
PKG-ATTRIBUTES: passed
PKG-PROPERTIES: passed

$ spack audit packages k4geo
PKG-DIRECTIVES: passed
PKG-ATTRIBUTES: passed
PKG-PROPERTIES: passed

$ spack audit packages k4simdelphes
PKG-DIRECTIVES: passed
PKG-ATTRIBUTES: passed
PKG-PROPERTIES: passed
```

> **Note:** `spack audit` checks only structural Python issues (e.g., invalid
> directive syntax, missing required class attributes, property correctness).
> It does **not** check for Spack PR guidelines such as missing `maintainers()`,
> missing `license()`, incorrect dependency types, or non-standard version
> schemes. The meaningful issues below were found through **manual recipe
> inspection**.

---

## 3. Manual Critique — Per-Package Analysis

### 3.1 k4fwcore

**Source:** `~/key4hep-spack/packages/k4fwcore/package.py` (74 lines)

#### Issue 1: Missing `maintainers()` directive

- **grep output:** `grep -n "maintainers" ... → MISSING`
- **Impact:** Spack requires at least one maintainer for upstream packages.
  Without it, the PR will be rejected.

**Before** (line 5–6):
```python
class K4fwcore(CMakePackage, Ilcsoftpackage):
    """Core framework components of the Key4HEP project"""
```

**After:**
```python
class K4fwcore(CMakePackage):
    """Core framework components of the Key4HEP project"""

    ...

    maintainers("jmcarcell", "tmadlener")
```

#### Issue 2: Missing `license()` directive

- **grep output:** `grep -n "license" ... → MISSING`
- **Impact:** All upstream Spack packages must declare their license.

**Before:** *(no license line)*

**After** (added after `maintainers()`):
```python
    license("Apache-2.0")
```

#### Issue 3: Inherits `Ilcsoftpackage` — key4hep-only mixin

- **grep output:** Line 2: `from spack.pkg.k4.key4hep_stack import Ilcsoftpackage`
- **grep output:** Line 5: `class K4fwcore(CMakePackage, Ilcsoftpackage):`
- **Impact:** `Ilcsoftpackage` is defined in the `k4` repo namespace
  (`spack.pkg.k4.key4hep_stack`). It cannot exist in Spack builtin.
  This is a **critical blocker** for upstreaming.

**Before** (lines 1–5):
```python
from spack.package import *
from spack.pkg.k4.key4hep_stack import Ilcsoftpackage


class K4fwcore(CMakePackage, Ilcsoftpackage):
```

**After:**
```python
from spack.package import *


class K4fwcore(CMakePackage):
```

#### Issue 4: Missing `depends_on("cmake", type="build")`

- **grep output:** `grep -n "depends_on.*cmake" ... → MISSING`
- **Impact:** CMakePackage classes should explicitly declare the cmake dependency
  with `type="build"` for correct DAG resolution.

**Before** (line 46–47):
```python
    depends_on("c", type="build", when="@:1.2")
    depends_on("cxx", type="build")
```

**After:**
```python
    depends_on("c", type="build", when="@:1.2")
    depends_on("cxx", type="build")
    depends_on("cmake", type="build")
```

#### Issue 5: No `test` method

- **grep output:** `grep -n "def.*test\|def.*check" ... → MISSING`
  (only `BUILD_TESTING` string reference found)
- **Impact:** Spack upstream packages are expected to have a `test()` or
  `test_*()` method for `spack test run`.

**Before:** *(no test method)*

**After:**
```python
    def test_import(self):
        """check k4fwcore installation"""
        python = self.spec["python"].command
        python("-c", "import k4FWCore")
```

---

### 3.2 k4geo

**Source:** `~/key4hep-spack/packages/k4geo/package.py` (120 lines)

#### Issue 1: Missing `license()` directive

- **grep output:** `grep -n "license" ... → MISSING`
- **Impact:** Required for upstream acceptance.

**Before:** *(no license line; `maintainers("jmcarcell")` on line 18, but no
license)*

**After** (added after `maintainers(...)`):
```python
    maintainers("jmcarcell")

    license("Apache-2.0")
```

#### Issue 2: Missing `depends_on("cmake", type="build")`

- **grep output:** `grep -n "depends_on.*cmake" ... → MISSING`
- **Impact:** Same as k4fwcore — must be explicit.

**Before** (line 58):
```python
    depends_on("cxx", type="build")
```

**After:**
```python
    depends_on("cxx", type="build")
    depends_on("cmake", type="build")
```

#### Issue 3: Unconstrained `depends_on("dd4hep")` and `depends_on("lcio")`

- **grep output:**
  - Line 60: `depends_on("lcio")` — no version constraint
  - Line 61: `depends_on("dd4hep")` — no version constraint (separate from
    the `dd4hep@1.31:` entry on line 62 which uses `when="@0.22:"`)
- **Impact:** Open-ended deps can pull incompatible ancient versions.
  Best practice is to constrain to known-compatible minimum versions.

**Before** (lines 60–62):
```python
    depends_on("lcio")
    depends_on("dd4hep")
    depends_on("dd4hep@1.31:", when="@0.22:")
```

**After:**
```python
    depends_on("lcio@2.17:")
    depends_on("dd4hep@1.25:", when="@:0.21")
    depends_on("dd4hep@1.31:", when="@0.22:")
```

#### Issue 4: f-string for `CMAKE_CXX_STANDARD` instead of `self.define()`

- **grep output:** Line 71:
  `f"-DCMAKE_CXX_STANDARD={self.spec['root'].variants['cxxstd'].value}"`
- **Impact:** Using raw f-strings bypasses Spack's define() mechanism.
  `self.define()` is the idiomatic and safer approach.

**Before** (line 70–72):
```python
        args.append(
            f"-DCMAKE_CXX_STANDARD={self.spec['root'].variants['cxxstd'].value}"
        )
```

**After:**
```python
        args.append(
            self.define(
                "CMAKE_CXX_STANDARD",
                self.spec["root"].variants["cxxstd"].value,
            )
        )
```

#### Issue 5: Debug `print(self)` in `install_check`

- **Line 116:** `print(self)` — leftover debug statement.

**Before:**
```python
    @run_after("install")
    def install_check(self):
        print(self)
        with working_dir(self.build_directory):
```

**After:**
```python
    @run_after("install")
    def install_check(self):
        with working_dir(self.build_directory):
```

---

### 3.3 k4simdelphes

**Source:** `~/key4hep-spack/packages/k4simdelphes/package.py` (116 lines)

#### Issue 1: Missing `license()` directive

- **grep output:** `grep -n "license" ... → MISSING`

**Before:** *(no license line)*

**After** (added after `maintainers(...)`):
```python
    maintainers("vvolkl", "tmadlener")

    license("Apache-2.0")
```

#### Issue 2: Inherits `Ilcsoftpackage` — key4hep-only mixin

- **grep output:**
  - Line 7: `from spack.pkg.k4.key4hep_stack import Ilcsoftpackage`
  - Line 10: `class K4simdelphes(CMakePackage, Ilcsoftpackage):`
- **Impact:** Same as k4fwcore — `Ilcsoftpackage` is in the `k4` repo
  namespace and **cannot** exist in Spack builtin.

**Before** (lines 6–10):
```python
from spack.package import *
from spack.pkg.k4.key4hep_stack import Ilcsoftpackage


class K4simdelphes(CMakePackage, Ilcsoftpackage):
```

**After:**
```python
from spack.package import *


class K4simdelphes(CMakePackage):
```

#### Issue 3: Missing `depends_on("cmake", type="build")`

- **grep output:** `grep -n "depends_on.*cmake" ... → MISSING`

**Before** (line 83):
```python
    depends_on("cxx", type="build")
```

**After:**
```python
    depends_on("cxx", type="build")
    depends_on("cmake", type="build")
```

#### Issue 4: No `test` method

- **grep output:** `grep -n "def.*test\|def.*check" ... → MISSING`

**Before:** *(no test method)*

**After:**
```python
    def test_import(self):
        """check k4SimDelphes installation"""
        assert self.prefix.lib.isdir()
```

#### Issue 5: f-string and `.format()` used for cmake args instead of `self.define()`

- **Line 105:** `f"-DCMAKE_CXX_STANDARD={self.spec['root']...}"`
- **Line 107:** `"-DBUILD_TESTING={0}".format(self.run_tests)`
- **Line 108:** `"-DCMAKE_INSTALL_LIBDIR=lib"` — hardcoded string
- **Impact:** Should use `self.define()` for consistency and correctness.

**Before** (lines 105–108):
```python
            f"-DCMAKE_CXX_STANDARD={self.spec['root'].variants['cxxstd'].value}",
            "-DUSE_EXTERNAL_CATCH2=ON",
            "-DBUILD_TESTING={0}".format(self.run_tests),
            "-DCMAKE_INSTALL_LIBDIR=lib",
```

**After:**
```python
            self.define(
                "CMAKE_CXX_STANDARD",
                self.spec["root"].variants["cxxstd"].value,
            ),
            self.define("USE_EXTERNAL_CATCH2", True),
            self.define("BUILD_TESTING", self.run_tests),
            self.define("CMAKE_INSTALL_LIBDIR", "lib"),
```

#### Issue 6: Missing explicit `depends_on("root")`

- **Impact:** The recipe uses `self.spec["root"].variants["cxxstd"]` in
  `cmake_args()` but does not declare `root` as a dependency. It only works
  because `root` is pulled in transitively. An explicit dep is needed.

**Before:** *(no `depends_on("root")` line)*

**After:**
```python
    depends_on("root")
```

---

## 4. Summary Table

| Package | `spack audit` | `maintainers()` | `license()` | `Ilcsoftpackage` | `cmake` dep | Test method | Other issues |
|---------|:---:|:---:|:---:|:---:|:---:|:---:|---|
| **k4fwcore** | ✅ passed | ❌ Missing | ❌ Missing | ❌ Must remove | ❌ Missing | ❌ Missing | — |
| **k4geo** | ✅ passed | ✅ Present | ❌ Missing | ✅ Not used | ❌ Missing | ⚠️ Partial (`install_check`) | Unconstrained dd4hep/lcio, f-string cmake args, debug `print(self)` |
| **k4simdelphes** | ✅ passed | ✅ Present | ❌ Missing | ❌ Must remove | ❌ Missing | ❌ Missing | f-string/`.format()` cmake args, missing explicit `root` dep |

### Error/Warning Counts

| Package | Critical Errors | Warnings | Total Issues |
|---------|:-:|:-:|:-:|
| **k4fwcore** | 3 (maintainers, license, Ilcsoftpackage) | 2 (cmake dep, test method) | **5** |
| **k4geo** | 1 (license) | 4 (cmake dep, dd4hep/lcio constraints, f-string, print debug) | **5** |
| **k4simdelphes** | 2 (license, Ilcsoftpackage) | 4 (cmake dep, test method, f-string, root dep) | **6** |

---

*Report generated on 2026-03-09 using Spack (develop branch) with key4hep-spack repo.*
