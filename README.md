# pyhls — a tiny Python → HLS C++ transpiler for the Pynq-Z2

`pyhls` takes a **small, well-defined subset of Python**, transpiles it into
**synthesizable HLS C++**, builds it into an FPGA bitstream, and runs it on a
**Pynq-Z2** board. You write a plain Python function; the tool turns it into
hardware you can call from the board's ARM processor.

This is **not** a general Python-to-hardware compiler (that is a research-grade
problem). It is a deliberately narrow, *honest* transpiler: it handles a clearly
stated grammar and rejects everything else with a clear error instead of
emitting broken C++. The narrow scope is the point — it keeps the translation
provably correct within its boundaries.

---

## What each file does

| File | Runs where | Job |
|------|-----------|-----|
| `transpiler.py` | any machine | Walks a Python function's syntax tree and emits HLS C++. **The core of the project.** |
| `compiler.py`   | FPGA build machine | Scripts Vitis HLS + Vivado to turn that C++ into `.bit` + `.hwh`. |
| `deploy.py`     | on the Pynq-Z2 | Loads the bitstream and calls the block over AXI-Lite. |
| `examples/demo.py` | any machine | Transpiles three different functions to show it's a real tool. |

## The workflow

```
   You write a simple Python function
              │
              ▼
   transpiler.py   ── walks the AST, emits HLS C++ (+ AXI-Lite pragmas)
              │
              ▼
   compiler.py     ── Vitis HLS (C++ → RTL → IP) → Vivado (IP → bitstream)
              │            produces  name.bit  +  name.hwh
              ▼
   [ copy name.bit + name.hwh onto the board ]
              │
              ▼
   deploy.py       ── loads the overlay, run(inputs) → output
```

---

## The supported subset (Tier 1)

**Works:** one top-level function; scalar arguments; local variables;
`=` and `+= -= *= /=`; arithmetic `+ - * /` and unary minus; comparisons
`< <= > >= == !=`; `and / or / not`; `if / elif / else`;
`for i in range(...)` with **constant** integer bounds; `return`; numeric and
boolean literals; `pass`; a function docstring.

**Rejected (with a clear message):** `while` loops; lists / dicts / tuples /
strings; any function call except `range()`; multiple or nested `def`s;
recursion; imports, classes, decorators; tuple unpacking / multiple targets;
f-strings, comprehensions, lambdas; `** // %` and bitwise operators; chained
comparisons (`a < b < c`); `range()` with variable bounds.

### Type model
Every argument and variable is one fixed-point type, `ap_fixed<16,6>` by default
(16 total bits, 6 integer bits). Loop counters are `int`. The return type
defaults to the same fixed-point type; pass `return_type="int"` for functions
that return an integer decision (e.g. a 0/1 signal). No type inference is
attempted — that is out of scope by design.

### Example
```python
from transpiler import transpile

src = """
def signal(price, fast, slow, band):
    spread = fast - slow
    if spread > band and price > slow:
        return 1
    else:
        return 0
"""
cpp, header = transpile(src, return_type="int")
print(cpp)
```

---

## How to run it, step by step

### Step 0 — Try it with no hardware (any laptop)
```bash
cd pyhls/examples
python demo.py
```
This transpiles three functions and dry-run-compiles one, writing
`build/signal.cpp`, `build/signal.h`, `build/run_hls.tcl`, and
`build/run_vivado.tcl`. Open them and read the generated C++. This confirms the
transpiler half works. No FPGA tools needed.

### Step 1 — Build the bitstream (FPGA machine: Vitis HLS + Vivado on PATH)
```python
from transpiler import transpile
from compiler import Compiler

src = open("my_function.py").read()
cpp, header = transpile(src, return_type="int")

c = Compiler(name="signal", part="xc7z020clg400-1", clock_ns=10.0)
c.build(cpp, header, dry_run=False)   # runs Vitis HLS then Vivado (10–30+ min)
```
Vitis HLS turns the C++ into RTL and packages it as IP; Vivado builds the block
design and the bitstream. Output: `build/signal.bit` and `build/signal.hwh`.

### Step 2 — The Vivado block design (do it BY HAND the first time)
The auto-generated Vivado TCL in `compiler.py` handles the common case, but
exact paths and IP version strings vary between Vivado releases, so the first
time it's safest to build the block design in the GUI, confirm it works, then
export a known-good TCL. Do this:

1. **Create a block design** and add the **ZYNQ7 Processing System**.
2. Click **Run Block Automation** → apply the **Pynq-Z2 board preset** (this
   sets up DDR and the fixed I/O correctly). Make sure a general-purpose AXI
   master (`M_AXI_GP0`) is enabled — that's how the PS talks to your block.
3. Add your HLS IP (it appears in the IP catalog after HLS export; the repo path
   is `signal_hls/solution1/impl/ip`).
4. Click **Run Connection Automation** → let Vivado wire the PS master to your
   block's `s_axi_CONTROL`, plus clock and reset. **Check that the clock and
   reset actually connect** — an unconnected reset is the most common silent
   failure.
5. **Validate Design** (F6). Fix any critical warnings.
6. **Create HDL Wrapper** (right-click the `.bd`), set it as top.
7. **Generate Bitstream.**
8. `File → Export → Export Block Design` to save a clean TCL. Paste that into
   `_vivado_tcl()` in `compiler.py`, replacing the auto-generated body. From
   then on, builds are fully automated.

**Things to double-check in the block design:**
- board preset is **Pynq-Z2** (not plain Zynq) — wrong preset → DDR/timing issues
- the AXI-Lite interface on your block is named `s_axi_CONTROL`
- clock and reset are connected to your block (not left floating)
- address is assigned to your block (`assign_bd_address`) — otherwise the PS
  can't reach its registers

### Step 3 — Find the register offsets
After HLS runs, open the generated driver header:
```
signal_hls/solution1/impl/misc/drivers/.../xsignal_hw.h
```
It lists the exact AXI-Lite offset of every argument and of the return value.
You'll plug those into `deploy.py`. (The control register at `0x00` is standard:
bit0 = start, bit1 = done.)

### Step 4 — Run it on the board
Connect the Pynq-Z2 over Ethernet exactly as you already do (browse to its
address, open Jupyter running *on the board*). Copy `signal.bit`, `signal.hwh`,
and `deploy.py` onto the board (same folder; the `.hwh` must share the `.bit`'s
base name). Then, in a notebook cell on the board:
```python
from deploy import FPGAFunction

fn = FPGAFunction(
    "signal.bit",
    arg_offsets={"price": 0x10, "fast": 0x18, "slow": 0x20, "band": 0x28},  # from xsignal_hw.h
)
print(fn.run(price=1.5, fast=1.4, slow=1.2, band=0.05))   # -> the decision
```
Use `run()` for an integer decision, `run_fixed()` for a computed fixed-point
value.

---

## Measuring the latency (the number that sells it)
The honest FPGA latency comes from the **Vitis HLS synthesis report** (latency in
clock cycles × clock period), *not* from timing `run()` in Python — that Python
round trip is dominated by AXI-Lite bus + PYNQ software overhead, not by the
computation. Report the HLS core latency against the same function timed on CPU
for a fair comparison, and be ready to explain the bus-overhead caveat.

## Known limitations / honest caveats
- Tier-1 subset only (see above); by design.
- Vivado block-design TCL is release-sensitive; verify once by hand, then export.
- Register offsets must be read from the generated header, not assumed.
- One fixed-point type for everything; no per-variable precision or type inference.
- Function-body values are fixed-point; only loop counters are integers.
