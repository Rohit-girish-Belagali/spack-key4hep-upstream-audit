# Mock Pull Request: Add k4geo package

## PR Title
**New package: k4geo — DD4hep geometry models for future colliders**

---

## PR Checklist

- [x] **Maintainers**: `maintainers("jmcarcell")` — existing, verified
- [x] **License**: Added `license("Apache-2.0")` directive
- [x] **Checksums**: All versions have `sha256` checksums ✅
- [x] **Style**: Passes `flake8` with zero errors
- [x] **Tests**: Added `test_load_detector` method for `spack test run`
- [x] **Build deps**: Added explicit `depends_on("cmake", type="build")`
- [x] **Dep constraints**: Constrained `lcio@2.17:` and split `dd4hep` into
      version-ranged entries (`@1.25:` for `@:0.21`, `@1.31:` for `@0.22:`)
- [x] **No external repo deps**: Recipe uses only `from spack.package import *`
      — no key4hep-specific imports

---

## Package Description

**k4geo** provides DD4hep geometry models for future collider experiments,
including CLIC, ILC, FCC-ee, FCC-hh, and CEPC detector concepts. It is a core
component of the Key4HEP software ecosystem.

- **Homepage**: https://github.com/key4hep/k4geo
- **License**: Apache-2.0
- **Build system**: CMake (with Ninja generator)
- **Key dependencies**: dd4hep, lcio, ROOT, podio

---

## Full Corrected Recipe

```python
# Copyright 2013-2024 Lawrence Livermore National Security, LLC and other
# Spack Project Developers. See the top-level COPYRIGHT file for details.
#
# SPDX-License-Identifier: (Apache-2.0 OR MIT)

from spack.package import *


class K4geo(CMakePackage):
    """DD4hep geometry models for future colliders."""

    homepage = "https://github.com/key4hep/k4geo"
    git = "https://github.com/key4hep/k4geo.git"
    url = "https://github.com/key4hep/k4geo/archive/v00-16-07.tar.gz"

    generator = "Ninja"

    maintainers("jmcarcell")

    license("Apache-2.0")

    version("main", branch="main")
    version(
        "00-24",
        sha256="3eefd973c0e534cc5cbb4d8fc079455508986bba49f859c30e0c23ac3e732f19",
    )
    version(
        "00-23",
        sha256="dd0c6300a6a2190a089012dfea271bd31050e8d4134ce09d896ebd81ef7391c5",
    )
    version(
        "00-22",
        sha256="95712eaf3452d29d35ac8156c37e5b4ea6449eb04073fb330bddc5df686f2cb3",
    )
    version(
        "0.21",
        sha256="0451e532fd22b2b9ea93a71f7036ea6de44386ecb10a84f28bc1d9fd557c6ad1",
        url="https://github.com/key4hep/k4geo/archive/refs/tags/v00-21.tar.gz",
    )
    version(
        "0.20.0",
        sha256="40d5842faa4767cc1b8c19f9b710713ba6a128ecd94fb9682e3afe3145e20511",
    )
    version(
        "0.19.0",
        sha256="6e8101e5991870484988f9fcb0299076a30f9b5f37e4e51141e50dfd30f32314",
    )

    version(
        "0.18.1",
        sha256="2bcdcbb772b9672994ac3cf8e9691f55f23a898d67c6f6c84ae0ae1b5416d893",
    )
    version(
        "0.18",
        sha256="50cd058e80baba21748156f3603a45a2388c6f3a8823d9aaa3f419eb58038fc9",
    )

    variant("compact", default=True, description="Install compact files")

    depends_on("cxx", type="build")
    depends_on("cmake", type="build")

    depends_on("lcio@2.17:")
    depends_on("dd4hep@1.25:", when="@:0.21")
    depends_on("dd4hep@1.31:", when="@0.22:")
    depends_on("root")
    depends_on("python", type="build")
    depends_on("ninja", type="build")
    depends_on("podio")

    def cmake_args(self):
        args = []
        args.append(
            self.define(
                "CMAKE_CXX_STANDARD",
                self.spec["root"].variants["cxxstd"].value,
            )
        )
        args.append(
            self.define_from_variant("INSTALL_COMPACT_FILES", "compact")
        )
        # Automatically install the CAD beampipe files
        # if we install the compact files
        args.append(
            self.define(
                "INSTALL_BEAMPIPE_STL_FILES",
                self.spec.variants["compact"].value,
            )
        )
        args.append(self.define("BUILD_TESTING", self.run_tests))
        return args

    def setup_run_environment(self, env):
        env.set("LCGEO", self.prefix.share.k4geo)
        env.set("K4GEO", self.prefix.share.k4geo)
        env.set("lcgeo_DIR", self.prefix.share.k4geo)
        env.set("k4geo_DIR", self.prefix.share.k4geo)
        env.prepend_path(
            "LD_LIBRARY_PATH",
            self.spec["k4geo"].libs.directories[0],
        )

    def setup_build_environment(self, env):
        env.set("LCGEO", self.prefix.share.k4geo)
        env.set("lcgeo_DIR", self.prefix.share.k4geo)
        env.prepend_path(
            "LD_LIBRARY_PATH",
            self.spec["lcio"].libs.directories[0],
        )
        env.prepend_path("LD_LIBRARY_PATH", self.prefix.lib)

    def setup_dependent_run_environment(self, env, dependent_spec):
        env.set("LCGEO", self.prefix.share.k4geo)
        env.set("lcgeo_DIR", self.prefix.share.k4geo)
        env.prepend_path(
            "LD_LIBRARY_PATH",
            self.spec["k4geo"].libs.directories[0],
        )
        env.prepend_path(
            "LD_LIBRARY_PATH",
            self.spec["lcio"].libs.directories[0],
        )

    def setup_dependent_build_environment(self, env, dependent_spec):
        env.set("LCGEO", self.prefix.share.k4geo)
        env.set("lcgeo_DIR", self.prefix.share.k4geo)
        env.prepend_path(
            "LD_LIBRARY_PATH",
            self.spec["k4geo"].libs.directories[0],
        )
        env.prepend_path(
            "LD_LIBRARY_PATH",
            self.spec["lcio"].libs.directories[0],
        )

    # dd4hep tests need to run after install step:
    # disable the usual check
    def check(self):
        pass

    # instead add custom check step that runs after installation
    @run_after("install")
    def install_check(self):
        with working_dir(self.build_directory):
            if self.run_tests:
                ninja("test")

    def test_load_detector(self):
        """check that k4geo compact files are installed"""
        assert self.prefix.share.k4geo.isdir()
```

---

## Diff Table — Changes from Original key4hep-spack Recipe

| Line(s) | What Changed | Original | Corrected |
|---------|-------------|----------|-----------|
| 21 | Added `license()` directive | *(missing)* | `license("Apache-2.0")` |
| 63 | Added `cmake` build dependency | *(missing)* | `depends_on("cmake", type="build")` |
| 60 | Constrained `lcio` minimum version | `depends_on("lcio")` | `depends_on("lcio@2.17:")` |
| 61 | Split bare `dd4hep` into version range | `depends_on("dd4hep")` | `depends_on("dd4hep@1.25:", when="@:0.21")` |
| 71 | Replaced f-string with `self.define()` | `f"-DCMAKE_CXX_STANDARD={...}"` | `self.define("CMAKE_CXX_STANDARD", ...)` |
| 116 | Removed debug `print(self)` | `print(self)` | *(removed)* |
| 139 | Added `test_load_detector` method | *(missing)* | `def test_load_detector(self): ...` |

---

## Validation Plan — Testing on a Clean Ubuntu 22.04 Environment

The following step-by-step plan tests the corrected k4geo package from scratch
on a minimal Ubuntu 22.04 system with no pre-existing key4hep-spack repo.

### Prerequisites

A clean Ubuntu 22.04 machine (or Docker container) with `build-essential`,
`git`, `curl`, and `python3` installed.

### Step-by-Step Commands

```bash
# 1. Install system prerequisites
sudo apt-get update
sudo apt-get install -y build-essential git curl python3 python3-pip \
    gfortran unzip

# 2. Clone Spack (latest develop branch)
git clone --depth=1 https://github.com/spack/spack.git ~/spack
source ~/spack/share/spack/setup-env.sh

# 3. Verify Spack is functional
spack --version
spack compiler find

# 4. Create a local Spack repo to hold the corrected k4geo recipe
mkdir -p ~/my-spack-repo/packages/k4geo
cat > ~/my-spack-repo/repo.yaml << 'EOF'
repo:
  namespace: my_repo
  version: 1.0
EOF

# 5. Copy the corrected package.py into the local repo
cp packages/k4geo/package.py ~/my-spack-repo/packages/k4geo/package.py

# 6. Register the local repo with Spack (highest priority)
spack repo add ~/my-spack-repo

# 7. Verify the package is found and parseable
spack info k4geo

# 8. Check that spack audit passes
spack audit packages k4geo

# 9. Run Spack style checks (if available)
spack style --root ~/my-spack-repo

# 10. Concretize the package to verify dependency resolution
spack spec k4geo

# 11. Install with tests enabled
spack install --test=root k4geo

# 12. Run the standalone test
spack test run k4geo

# 13. Verify the installation
spack find k4geo
spack location -i k4geo
ls $(spack location -i k4geo)/share/k4geo/

# 14. Clean up
spack uninstall -y k4geo
spack repo remove my_repo
```

### Expected Results

- **Step 7**: `spack info k4geo` shows the package metadata, versions, and
  dependencies correctly.
- **Step 8**: `spack audit packages k4geo` returns all PASSED.
- **Step 10**: `spack spec k4geo` concretizes successfully with all deps
  resolved from builtin.
- **Step 11**: `spack install --test=root k4geo` completes without errors.
  (Note: full build requires ~2-4 hours depending on hardware, as it builds
  ROOT, dd4hep, and other heavy deps from source.)
- **Step 12**: `spack test run k4geo` executes `test_load_detector` and passes.

---

*Mock PR generated on 2026-03-09 for GSoC 2026 Qualification Task.*
