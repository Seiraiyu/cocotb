# RyuSim Simulator Support Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add RyuSim as a supported simulator in cocotb so users can use `SIM=ryusim` or `get_runner("ryusim")`.

**Architecture:** Follow the existing DSim/Icarus patterns (Verilog-only, VPI-only). RyuSim compiles SV to C++ producing a shared library, then runs it with cocotb VPI loaded via `LD_PRELOAD` of RyuSim's VPI shim (`libryusim_vpi.so`). The runner and Makefile hide this from the user.

**Tech Stack:** Python (runner class), Makefile (build flow), C++ (VPI library extension build)

---

### Task 1: Add RyuSim version class

**Files:**
- Modify: `src/cocotb_tools/sim_versions.py:143` (append after NvcVersion)

**Step 1: Add the version class**

Append after line 142 (end of `NvcVersion`):

```python


class RyusimVersion(LooseVersion):
    """Version numbering class for RyuSim."""
```

**Step 2: Verify syntax**

Run: `python -c "from cocotb_tools.sim_versions import RyusimVersion; print(RyusimVersion('1.0.0') > RyusimVersion('0.9.0'))"`
Expected: `True`

**Step 3: Commit**

```bash
git add src/cocotb_tools/sim_versions.py
git commit -m "feat: add RyusimVersion class for version detection"
```

---

### Task 2: Register RyuSim in config.py

**Files:**
- Modify: `src/cocotb_tools/config.py:117-131` (supported_sims list)

**Step 1: Add "ryusim" to the supported_sims list**

In `lib_name()` function, add `"ryusim"` to the `supported_sims` list at line 131 (after `"dsim"`):

```python
    supported_sims = [
        "icarus",
        "verilator",
        "questa",
        "modelsim",
        "ius",
        "xcelium",
        "vcs",
        "ghdl",
        "riviera",
        "activehdl",
        "cvc",
        "nvc",
        "dsim",
        "ryusim",
    ]
```

No mapping needed — `ryusim` maps to itself (falls through to `else: library_name = simulator_name` at line 143-144).

**Step 2: Verify**

Run: `python -c "from cocotb_tools.config import lib_name; print(lib_name('vpi', 'ryusim'))"`
Expected: `libcocotbvpi_ryusim.so`

**Step 3: Commit**

```bash
git add src/cocotb_tools/config.py
git commit -m "feat: register ryusim in cocotb_tools.config lib_name()"
```

---

### Task 3: Add VPI library build extension

**Files:**
- Modify: `cocotb_build_libs.py:764-766` (before `return ext` in `get_ext()`)

**Step 1: Add RyuSim VPI extension**

Insert before `return ext` at line 766, after the DSim block:

```python

    #
    # RyuSim
    #
    if os.name == "posix":
        logger.info("Compiling libraries for RyuSim")
        ryusim_vpi_ext = _get_vpi_lib_ext(
            include_dirs=include_dirs,
            share_lib_dir=share_lib_dir,
            sim_define="RYUSIM",
        )
        ext.append(ryusim_vpi_ext)
```

Linux-only (`posix`) like DSim — no Windows support initially.

**Step 2: Verify the build extension is registered**

Run: `python -c "from cocotb_build_libs import get_ext; exts = get_ext(); print([e.name for e in exts if 'ryusim' in e.name])"`
Expected: `['cocotb/libs/libcocotbvpi_ryusim']`

**Step 3: Commit**

```bash
git add cocotb_build_libs.py
git commit -m "feat: add RyuSim VPI library build extension"
```

---

### Task 4: Create Makefile.ryusim

**Files:**
- Create: `src/cocotb_tools/makefiles/simulators/Makefile.ryusim`

**Step 1: Create the Makefile**

Model after `Makefile.dsim` (simplest recent addition). Key differences: RyuSim uses `ryusim compile` for build, then runs the output `.so` with `LD_PRELOAD` for the VPI shim.

```makefile
# Copyright cocotb contributors
# Licensed under the Revised BSD License, see LICENSE for details.
# SPDX-License-Identifier: BSD-3-Clause

TOPLEVEL_LANG ?= verilog

ifneq ($(or $(filter-out $(TOPLEVEL_LANG),verilog),$(VHDL_SOURCES)),)

$(COCOTB_RESULTS_FILE):
	@echo "Skipping simulation as only Verilog is supported on simulator=$(SIM)"
debug: $(COCOTB_RESULTS_FILE)

else

CMD_BIN := ryusim

ifdef RYUSIM_BIN_DIR
    CMD := $(shell :; command -v $(RYUSIM_BIN_DIR)/$(CMD_BIN) 2>/dev/null)
else
    CMD := $(shell :; command -v $(CMD_BIN) 2>/dev/null)
endif

ifeq (, $(CMD))
    $(error Unable to locate command >$(CMD_BIN)<)
endif

# RyuSim root (for locating libryusim_vpi.so)
RYUSIM_ROOT ?= $(shell dirname $(shell dirname $(CMD)))
RYUSIM_VPI_LIB ?= $(RYUSIM_ROOT)/lib/libryusim_vpi.so

# cocotb library paths
COCOTB_LIB_DIR := $(shell $(PYTHON_BIN) -m cocotb_tools.config --lib-dir)
COCOTB_VPI_LIB := $(shell $(PYTHON_BIN) -m cocotb_tools.config --lib-name-path vpi ryusim)

# Build output
SIM_MODEL := $(SIM_BUILD)/lib$(call deprecate,TOPLEVEL,COCOTB_TOPLEVEL).so

ifdef COCOTB_TOPLEVEL
    TOPMODULE_ARG := --top $(COCOTB_TOPLEVEL)
else
    TOPMODULE_ARG :=
endif

ifeq ($(WAVES), 1)
    COMPILE_ARGS += --trace-vcd
endif

# Compilation phase
$(SIM_MODEL): $(VERILOG_SOURCES) $(CUSTOM_COMPILE_DEPS) | $(SIM_BUILD)
	$(CMD) compile $(TOPMODULE_ARG) --Mdir $(SIM_BUILD) $(COMPILE_ARGS) $(EXTRA_ARGS) $(VERILOG_SOURCES)

# Execution phase
$(COCOTB_RESULTS_FILE): $(SIM_MODEL) $(CUSTOM_SIM_DEPS)
	$(RM) $(COCOTB_RESULTS_FILE)

	COCOTB_TEST_MODULES=$(call deprecate,MODULE,COCOTB_TEST_MODULES) \
	COCOTB_TESTCASE=$(call deprecate,TESTCASE,COCOTB_TESTCASE) \
	COCOTB_TEST_FILTER=$(COCOTB_TEST_FILTER) \
	COCOTB_TOPLEVEL=$(call deprecate,TOPLEVEL,COCOTB_TOPLEVEL) \
	GPI_EXTRA=$(GPI_EXTRA) \
	TOPLEVEL_LANG=$(TOPLEVEL_LANG) \
	LD_LIBRARY_PATH=$(COCOTB_LIB_DIR):$(LD_LIBRARY_PATH) \
	LD_PRELOAD=$(RYUSIM_VPI_LIB) \
	$(SIM_CMD_PREFIX) $(SIM_MODEL) $(SIM_ARGS) $(EXTRA_ARGS) $(call deprecate,PLUSARGS,COCOTB_PLUSARGS) $(SIM_CMD_SUFFIX)

	$(call check_results)

debug: $(SIM_MODEL) $(CUSTOM_SIM_DEPS)
	$(RM) $(COCOTB_RESULTS_FILE)

	COCOTB_TEST_MODULES=$(call deprecate,MODULE,COCOTB_TEST_MODULES) \
	COCOTB_TESTCASE=$(call deprecate,TESTCASE,COCOTB_TESTCASE) \
	COCOTB_TEST_FILTER=$(COCOTB_TEST_FILTER) \
	COCOTB_TOPLEVEL=$(call deprecate,TOPLEVEL,COCOTB_TOPLEVEL) \
	GPI_EXTRA=$(GPI_EXTRA) \
	TOPLEVEL_LANG=$(TOPLEVEL_LANG) \
	LD_LIBRARY_PATH=$(COCOTB_LIB_DIR):$(LD_LIBRARY_PATH) \
	LD_PRELOAD=$(RYUSIM_VPI_LIB) \
	$(SIM_CMD_PREFIX) gdb --args $(SIM_MODEL) $(SIM_ARGS) $(EXTRA_ARGS) $(call deprecate,PLUSARGS,COCOTB_PLUSARGS) $(SIM_CMD_SUFFIX)

	$(call check_results)

endif
```

**Step 2: Verify file is discoverable**

Run: `ls src/cocotb_tools/makefiles/simulators/Makefile.ryusim`
Expected: file exists

**Step 3: Commit**

```bash
git add src/cocotb_tools/makefiles/simulators/Makefile.ryusim
git commit -m "feat: add Makefile.ryusim for Makefile-based cocotb flows"
```

---

### Task 5: Add RyuSim Runner class

**Files:**
- Modify: `src/cocotb_tools/runner.py:2065-2096` (add class before `get_runner`, register in dict)

**Step 1: Add the RyuSim class**

Insert before `get_runner()` (before line 2067). This is the core integration — modeled after Dsim (Verilog-only, VPI-only) but with RyuSim's compile-to-shared-library model and LD_PRELOAD VPI loading.

```python
class RyuSim(Runner):
    """Implementation of :class:`Runner` for RyuSim.

    .. admonition:: Simulator-specific Usage

       * ``hdl_toplevel`` argument to :meth:`.build` is *required*.
       * Only supports Verilog/SystemVerilog (no VHDL).
       * Does not support the ``pre_cmd`` argument to :meth:`.test`.
    """

    supported_gpi_interfaces = {"verilog": ["vpi"]}

    def _simulator_in_path(self) -> None:
        if shutil.which("ryusim") is None:
            raise SystemExit("ERROR: ryusim executable not found!")

    def _get_include_options(self, includes: Sequence[PathLike]) -> _Command:
        return [f"-I{include}" for include in includes]

    def _get_define_options(self, defines: Mapping[str, object]) -> _Command:
        return [
            f"-D{name}={_as_sv_literal(value)}" for name, value in defines.items()
        ]

    def _get_parameter_options(self, parameters: Mapping[str, object]) -> _Command:
        return [
            f"-P{name}={_as_sv_literal(value)}"
            for name, value in parameters.items()
        ]

    @property
    def sim_file(self) -> Path:
        return self.build_dir / f"lib{self.hdl_toplevel}.so"

    def _use_external_viewer(self) -> bool:
        return True

    def _waves_file(self) -> str | None:
        return f"{self.hdl_toplevel}.vcd"

    def _ryusim_root(self) -> Path:
        """Locate RyuSim installation root from binary path."""
        ryusim_bin = shutil.which("ryusim")
        assert ryusim_bin is not None
        return Path(ryusim_bin).resolve().parent.parent

    def _ryusim_vpi_lib(self) -> Path:
        """Locate RyuSim's VPI shim library."""
        env_path = os.environ.get("RYUSIM_VPI_LIB")
        if env_path is not None:
            return Path(env_path)
        return self._ryusim_root() / "lib" / "libryusim_vpi.so"

    def _set_env_test(self) -> None:
        super()._set_env_test()
        # RyuSim loads cocotb VPI via LD_PRELOAD of its VPI shim
        vpi_lib = self._ryusim_vpi_lib()
        existing_preload = self.env.get("LD_PRELOAD", "")
        self.env["LD_PRELOAD"] = (
            f"{vpi_lib}:{existing_preload}" if existing_preload else str(vpi_lib)
        )
        # Ensure cocotb libs are on the library path
        lib_dir = str(cocotb_tools.config.libs_dir)
        existing_ld_path = self.env.get("LD_LIBRARY_PATH", "")
        self.env["LD_LIBRARY_PATH"] = (
            f"{lib_dir}:{existing_ld_path}" if existing_ld_path else lib_dir
        )

    def _build_command(self) -> list[_Command]:
        if self.hdl_toplevel is None:
            raise ValueError(
                "hdl_toplevel argument is required for all RyuSim builds"
            )

        sources = self._sources + self._verilog_sources

        for source in sources:
            if source.tag is not Verilog:
                raise ValueError(
                    f"{type(self).__qualname__} only supports Verilog. "
                    f"{str(source.value)!r} cannot be compiled."
                )

        for arg in self._build_args:
            if arg.tag not in (Verilog, None):
                raise ValueError(
                    f"{type(self).__qualname__} only supports Verilog. "
                    f"build_args {arg.value!r} cannot be applied."
                )

        build_args = [arg.value for arg in self._build_args]
        if self.waves:
            build_args.append("--trace-vcd")

        cmds: list[_Command] = []
        if (
            outdated(self.sim_file, (source.value for source in sources))
            or self.always
        ):
            cmds = [
                [
                    "ryusim",
                    "compile",
                    "--top",
                    self.hdl_toplevel,
                    "--Mdir",
                    str(self.build_dir),
                ]
                + self._get_define_options(self.defines)
                + self._get_include_options(self.includes)
                + self._get_parameter_options(self.parameters)
                + build_args
                + [str(source_file.value) for source_file in sources]
            ]
        else:
            self.log.warning("Skipping compilation of %s", self.sim_file)

        return cmds

    def _test_command(self) -> list[_Command]:
        if self.pre_cmd is not None:
            raise RuntimeError("pre_cmd is not implemented for RyuSim.")

        return [
            [
                str(self.sim_file),
                *self.test_args,
                *self.plusargs,
            ]
        ]
```

**Step 2: Register in get_runner()**

Add `"ryusim": RyuSim,` to the `supported_sims` dict at line 2087 (after `"dsim": Dsim,`):

```python
    supported_sims: dict[str, type[Runner]] = {
        "icarus": Icarus,
        "questa": Questa,
        "ghdl": Ghdl,
        "riviera": Riviera,
        "activehdl": ActiveHDL,
        "verilator": Verilator,
        "xcelium": Xcelium,
        "nvc": Nvc,
        "vcs": Vcs,
        "dsim": Dsim,
        "ryusim": RyuSim,
        # TODO: "activehdl": ActiveHdl,
    }
```

**Step 3: Verify the runner instantiates**

Run: `python -c "from cocotb_tools.runner import get_runner; r = get_runner('ryusim'); print(type(r).__name__, r.supported_gpi_interfaces)"`
Expected: `RyuSim {'verilog': ['vpi']}`

**Step 4: Commit**

```bash
git add src/cocotb_tools/runner.py
git commit -m "feat: add RyuSim runner class and register in get_runner()"
```

---

### Task 6: Verify full integration

**Step 1: Build cocotb from source to compile VPI libraries**

Run: `pip install -e .`
Expected: build succeeds, includes `Compiling libraries for RyuSim` in output

**Step 2: Verify the VPI library was built**

Run: `ls src/cocotb/libs/libcocotbvpi_ryusim*`
Expected: `libcocotbvpi_ryusim.so` exists

**Step 3: Verify config returns correct paths**

Run: `python -m cocotb_tools.config --lib-name vpi ryusim`
Expected: `libcocotbvpi_ryusim.so`

Run: `python -m cocotb_tools.config --lib-name-path vpi ryusim`
Expected: absolute path to `libcocotbvpi_ryusim.so`

**Step 4: Run pre-commit checks**

Run: `pre-commit run --all-files`
Expected: all checks pass (ruff, mypy, etc.)

**Step 5: Run simulator-agnostic tests**

Run: `nox -s dev_test_nosim`
Expected: existing tests still pass — no regressions

**Step 6: Final commit (if any fixups needed)**

```bash
git add -A
git commit -m "fix: address linting/type-check issues from RyuSim integration"
```

---

## File Summary

| File | Action | Lines |
|------|--------|-------|
| `src/cocotb_tools/sim_versions.py` | Append `RyusimVersion` | after L142 |
| `src/cocotb_tools/config.py` | Add `"ryusim"` to list | L131 |
| `cocotb_build_libs.py` | Add VPI ext in `get_ext()` | before L766 |
| `src/cocotb_tools/makefiles/simulators/Makefile.ryusim` | Create new | — |
| `src/cocotb_tools/runner.py` | Add `RyuSim` class + register | before L2067, L2087 |
| `.gitignore` | Add `.docs/` | already done |
