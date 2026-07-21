# pyhls — a tiny Python → HLS C++ transpiler for the Pynq-Z2

`pyhls` takes a **small, well-defined subset of Python** and transpiles it into
**synthesizable HLS C++**. This is stage one of a larger pipeline (Python →
HLS C++ → bitstream → Pynq-Z2 deployment); this README covers only the
transpiler itself. Later stages (`compiler.py` for the HLS/Vivado build,
`deploy.py` for running on the board) are in progress and will get their own
documentation as they land.

This is **not** a general Python-to-hardware compiler (that's a research-grade
problem). It's a deliberately narrow, *honest* transpiler: it handles a clearly
stated grammar and rejects everything else with a clear error instead of
emitting broken C++. The narrow scope is the point — it keeps the translation
provably correct within its boundaries.

---

## What's in this repo right now

| File | Runs where | Job |
|------|-----------|-----|
| `transpiler.py` | any machine | Walks a Python function's syntax tree and emits HLS C++. **The core of the project.** |


## The workflow 
1. You write a simple Python function 
2. transpiler.py ── walks the AST, emits HLS C++ (+ AXI-Lite pragmas)
3. name.cpp + name.h ── ready to hand to Vitis HLS

---

## The supported subset (Tier 1)

**Works:** 
- one top-level function 
- scalar arguments
- local variables
- `=` and `+= -= *= /=`
- arithmetic `+ - * /` and unary minus
- comparisons `< <= > >= == !=`
- `and / or / not`
- `if / elif / else`
- `for i in range(...)` with **constant** integer bounds 
- `return`
- numeric and boolean literals 
- `pass` 
- a function docstring

**Rejected :**
- `while` loops
- lists / dicts / tuples / strings
- any function call except `range()`
- multiple or nested `def`s
- recursion; imports, classes, decorators
- tuple unpacking / multiple targets
- f-strings, comprehensions, lambdas
- `** // %` and bitwise operators
- chained comparisons (`a < b < c`)
- `range()` with variable bounds

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

## Known limitations
- Tier-1 subset only (see above); by design, not an oversight.
- One fixed-point type for everything; no per-variable precision or type
  inference.
- Function-body values are fixed-point; only loop counters are integers.
- Fixed-point rounding/overflow mode is left at Vitis HLS defaults
  (truncate/wrap) rather than explicitly set.
- Output is HLS C++ source only; it is not yet run through Vitis HLS or
  synthesized in this stage of the project. Correctness is verified by
  reading the generated code, not by compiling/simulating it.
