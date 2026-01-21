# Code Reference: Function-by-Function Guide

This document explains every function, class, and important constant in `problem.py` and `perf_takehome.py` in a clean, structured way.

---

## Table of Contents

1. [problem.py – Types and Constants](#problempy--types-and-constants)
2. [problem.py – Data Structures](#problempy--data-structures)
3. [problem.py – Helper and Hash Functions](#problempy--helper-and-hash-functions)
4. [problem.py – Machine Simulator](#problempy--machine-simulator)
5. [problem.py – Reference Kernels and Memory](#problempy--reference-kernels-and-memory)
6. [perf_takehome.py – KernelBuilder](#perf_takehomepy--kernelbuilder)
7. [perf_takehome.py – Tests and Runner](#perf_takehomepy--tests-and-runner)

---

## problem.py – Types and Constants

### `Engine`

```python
Engine = Literal["alu", "load", "store", "flow"]
```

**What it is:** A type hint. An engine is one of the four functional units that can run in a single cycle: `alu` (arithmetic/logic), `load`, `store`, or `flow`. The simulator also supports `valu` and `debug`, but `Engine` in the type hint only lists the four used in `perf_takehome`.

---

### `Instruction`

```python
Instruction = dict[Engine, list[tuple]]
```

**What it is:** One machine instruction = one cycle. It is a dictionary:

- **Keys:** engine names (`"alu"`, `"load"`, `"store"`, `"flow"`, etc.)
- **Values:** list of **slots** (tuples) for that engine.

Example: `{"alu": [("+", 10, 8, 9)], "load": [("load", 5, 12)]}` means in one cycle: one add and one load.

---

### `CoreState` (enum)

```python
class CoreState(Enum):
    RUNNING = 1
    PAUSED = 2
    STOPPED = 3
```

**What it is:** The state of a core.

- **RUNNING:** Executing instructions.
- **PAUSED:** Stopped at a `pause` until `run()` resumes (used to sync with `yield` in the reference).
- **STOPPED:** Program finished (past end of program or `halt`).

---

### `SLOT_LIMITS`

```python
SLOT_LIMITS = {
    "alu": 12,
    "valu": 6,
    "load": 2,
    "store": 2,
    "flow": 1,
    "debug": 64,
}
```

**What it is:** Maximum number of slots per engine **per instruction (per cycle)**. For example, at most 12 `alu` slots, 2 `load` slots, 2 `store` slots, and 1 `flow` slot in a single instruction.

---

### `VLEN`

```python
VLEN = 8
```

**What it is:** Vector length. Vector ops (`vload`, `vstore`, `valu`, `vselect`) work on 8 elements at a time. A vector in scratch occupies 8 consecutive cells starting at some base address.

---

### `N_CORES`

```python
N_CORES = 1
```

**What it is:** Number of cores. This codebase only uses one core.

---

### `SCRATCH_SIZE`

```python
SCRATCH_SIZE = 1536
```

**What it is:** Size of each core’s scratch space (number of 32‑bit words). Scratch is used like registers and temporary storage.

---

### `BASE_ADDR_TID`

```python
BASE_ADDR_TID = 100000
```

**What it is:** Base value used for trace “thread” IDs when logging scratch addresses to `trace.json`. Used only for debugging/visualization.

---

## problem.py – Data Structures

### `Core` (dataclass)

```python
@dataclass
class Core:
    id: int
    scratch: list[int]
    trace_buf: list[int]
    pc: int = 0
    state: CoreState = CoreState.RUNNING
```

**What it is:** One CPU core in the simulated machine.

| Field       | Meaning                                                                 |
|------------|-------------------------------------------------------------------------|
| `id`       | Core index (0, 1, …).                                                   |
| `scratch`  | 1536-word scratch space. Holds operands and results for computations.   |
| `trace_buf`| List used by `trace_write` to record values; for debugging.             |
| `pc`       | Program counter: index of the next instruction in `program`.            |
| `state`    | `CoreState`: RUNNING, PAUSED, or STOPPED.                               |

---

### `DebugInfo` (dataclass)

```python
@dataclass
class DebugInfo:
    scratch_map: dict[int, (str, int)]
```

**What it is:** Debug metadata for the Machine.

- **Keys:** scratch addresses (integers).
- **Values:** `(name, length)` — logical name and length of the variable at that address.

Used so the simulator and trace can show names (e.g. `tmp1`, `n_nodes`) instead of raw addresses.

---

## problem.py – Helper and Hash Functions

### `cdiv(a, b)`

```python
def cdiv(a, b):
    return (a + b - 1) // b
```

**What it does:** Ceiling division. `cdiv(a, b)` is the smallest integer `q` such that `q * b >= a`. Used by the `alu` engine for the `"cdiv"` op.

---

### `HASH_STAGES`

```python
HASH_STAGES = [
    ("+", 0x7ED55D16, "+", "<<", 12),
    ("^", 0xC761C23C, "^", ">>", 19),
    ...
]
```

**What it is:** Defines the 6 stages of `myhash`. Each tuple is `(op1, val1, op2, op3, val3)`. One stage computes:

`a = (a op1 val1) op2 (a op3 val3)`  (all mod 2^32)

So `op1`/`val1` and `op3`/`val3` are the two “sides,” and `op2` combines them. The last number is the shift amount for `<<` or `>>`.

---

### `myhash(a: int) -> int`

```python
def myhash(a: int) -> int:
```

**What it does:** 32‑bit hash used in the algorithm. For each stage in `HASH_STAGES` it does:

`a = ((a op1 val1) op2 (a op3 val3)) % 2**32`

with `op1`/`op3` in `+`, `^`, `<<`, `>>`. Returns the final `a`. This is the reference that any machine implementation must match.

---

### `myhash_traced(a, trace, round, batch_i) -> int`

**What it does:** Same as `myhash`, but after each stage it does:

`trace[(round, batch_i, "hash_stage", stage_index)] = a`

Used by `reference_kernel2` so `value_trace` can be compared to `debug` `compare` in the kernel.

---

## problem.py – Machine Simulator

### `Machine.__init__(mem_dump, program, debug_info, n_cores=1, scratch_size=SCRATCH_SIZE, trace=False, value_trace={})`

**What it does:** Builds the machine.

| Argument       | Role                                                                 |
|----------------|----------------------------------------------------------------------|
| `mem_dump`     | Initial main memory (list of ints). Copied; original is not changed. |
| `program`      | List of `Instruction` (the program to run).                          |
| `debug_info`   | `DebugInfo` for scratch names and lengths.                           |
| `n_cores`      | Number of cores.                                                     |
| `scratch_size` | Words of scratch per core.                                           |
| `trace`        | If True, write `trace.json` for Perfetto/Chrome.                     |
| `value_trace`  | Dict filled by `reference_kernel2`; used by `debug` `compare`.       |

It creates `cores`, copies `mem_dump` into `self.mem`, sets `cycle=0`, and if `trace` is True calls `setup_trace()`.

---

### `Machine.rewrite_instr(instr)`

**What it does:** Takes one `Instruction` and, for each slot, replaces scratch **addresses** with the corresponding **names** from `scratch_map` where possible. Used for human‑readable printing (`print_step`), not for execution.

---

### `Machine.print_step(instr, core)`

**What it does:** Prints the current scratch map (by name) and the instruction with named operands. Used when `machine.prints` is True for debugging.

---

### `Machine.scratch_map(core)`

**What it does:** Returns a dict `{name: core.scratch[addr : addr+length]}` for each `(addr, (name, length))` in `debug_info.scratch_map`. Lets you inspect scratch by variable name.

---

### `Machine.rewrite_slot(slot)`

**What it does:** For one slot tuple, replaces each element that is a scratch address in `scratch_map` with its name; leaves others (e.g. opcodes, immediates) as is. Helper for `rewrite_instr`.

---

### `Machine.setup_trace()`

**What it does:** Opens `trace.json` and writes the Trace Event Format header and metadata: process/thread names for each engine slot and for scratch variables. Used when `trace=True` in `__init__`.

---

### `Machine.run()`

**What it does:** Main run loop.

1. For any PAUSED core, set state to RUNNING.
2. While any core is RUNNING:
   - For each RUNNING core: if `pc >= len(program)`, set state to STOPPED and skip.
   - Otherwise, fetch `program[pc]`, increment `pc`, call `step(instr, core)`.
   - If the instruction has any engine other than `debug`, set `has_non_debug = True`.
3. After processing all cores for this “tick,” if `has_non_debug` then `self.cycle += 1`.

So one logical “tick” can include one instruction from each core; `cycle` increments when at least one core did non‑debug work.

---

### `Machine.alu(core, op, dest, a1, a2)`

**What it does:** One scalar ALU op. Reads `a1 = core.scratch[a1]`, `a2 = core.scratch[a2]`, computes `res = a1 op a2`, then `res = res % 2**32`, and records `self.scratch_write[dest] = res` (applied at end of `step`).

**Supported `op`:** `+`, `-`, `*`, `//`, `cdiv`, `^`, `&`, `|`, `<<`, `>>`, `%`, `<`, `==`. For `<` and `==`, the result is 0 or 1.

---

### `Machine.valu(core, *slot)`

**What it does:** Vector (SIMD) ops. Each op works on `VLEN` (8) consecutive scratch cells.

| Slot form                 | Effect                                                                 |
|---------------------------|------------------------------------------------------------------------|
| `("vbroadcast", dest, src)` | `scratch[dest+j] = scratch[src]` for `j in 0..7`.                    |
| `("multiply_add", dest, a, b, c)` | `scratch[dest+j] = (scratch[a+j]*scratch[b+j] + scratch[c+j]) % 2**32`. |
| `(op, dest, a1, a2)`      | For each `j`: `scratch[dest+j] = scratch[a1+j] op scratch[a2+j]` (via `alu`). |

All writes go to `scratch_write` and are applied at end of `step`.

---

### `Machine.load(core, *slot)`

**What it does:** Load (or constant) into scratch.

| Slot form                 | Effect                                                                 |
|---------------------------|------------------------------------------------------------------------|
| `("load", dest, addr)`    | `scratch[dest] = mem[scratch[addr]]`.                                 |
| `("load_offset", dest, addr, offset)` | `scratch[dest+offset] = mem[scratch[addr+offset]]`.              |
| `("vload", dest, addr)`   | `scratch[dest+j] = mem[scratch[addr]+j]` for `j in 0..7`. `addr` is a scalar in scratch. |
| `("const", dest, val)`    | `scratch[dest] = val % 2**32`.                                        |

---

### `Machine.store(core, *slot)`

**What it does:** Store from scratch to main memory.

| Slot form                | Effect                                                                 |
|--------------------------|------------------------------------------------------------------------|
| `("store", addr, src)`   | `mem[scratch[addr]] = scratch[src]`.                                  |
| `("vstore", addr, src)`  | `mem[scratch[addr]+j] = scratch[src+j]` for `j in 0..7`.              |

Writes go to `mem_write` and are applied at end of `step`.

---

### `Machine.flow(core, *slot)`

**What it does:** Control flow and select.

| Slot form                  | Effect                                                                 |
|----------------------------|------------------------------------------------------------------------|
| `("select", dest, cond, a, b)` | `scratch[dest] = scratch[a] if scratch[cond]!=0 else scratch[b]`.  |
| `("add_imm", dest, a, imm)`    | `scratch[dest] = (scratch[a] + imm) % 2**32`.                    |
| `("vselect", dest, cond, a, b)`| Per lane: `scratch[dest+j] = scratch[a+j] if scratch[cond+j]!=0 else scratch[b+j]`. |
| `("halt",)`                | `core.state = STOPPED`.                                                |
| `("pause",)`               | If `enable_pause`: `core.state = PAUSED`.                              |
| `("trace_write", val)`     | `core.trace_buf.append(scratch[val])`.                                 |
| `("cond_jump", cond, addr)`   | If `scratch[cond]!=0`: `core.pc = addr`.                           |
| `("cond_jump_rel", cond, offset)` | If `scratch[cond]!=0`: `core.pc += offset`.                     |
| `("jump", addr)`           | `core.pc = addr`.                                                      |
| `("jump_indirect", addr)`  | `core.pc = scratch[addr]`.                                             |
| `("coreid", dest)`         | `scratch[dest] = core.id`.                                             |

---

### `Machine.trace_post_step(instr, core)`

**What it does:** After `step`, if tracing, appends events to `trace.json` for scratch cells that were written (using `scratch_map` names). Used for Perfetto-style visualization.

---

### `Machine.trace_slot(core, slot, name, i)`

**What it does:** Writes one trace event for a single slot execution (engine `name`, slot index `i`) at the current `cycle`. Used during `step` when `trace` is set.

---

### `Machine.step(instr, core)`

**What it does:** Executes one instruction for one core.

1. `scratch_write = {}`, `mem_write = {}`.
2. For each `(name, slots)` in `instr`:
   - **debug:** If `enable_debug`, run `compare` / `vcompare` against `value_trace`; no engine execution. Skip the rest for `debug`.
   - For other engines: check `len(slots) <= SLOT_LIMITS[name]`, then for each slot call the right engine function (`alu`, `valu`, `load`, `store`, `flow`). Writes go to `scratch_write` / `mem_write`.
3. Apply `scratch_write` to `core.scratch` and `mem_write` to `self.mem`.
4. If tracing, call `trace_post_step`.
5. Delete `scratch_write` and `mem_write`.

So all reads in a `step` see the state from the start of the step; all writes take effect at the end.

---

### `Machine.__del__()`

**What it does:** If a trace file was opened, writes the closing `]` and closes it.

---

## problem.py – Reference Kernels and Memory

### `Tree` (dataclass)

```python
@dataclass
class Tree:
    height: int
    values: list[int]
```

**What it is:** Implicit perfect binary tree. `values` is a flat array: root at 0, left child of `i` at `2*i+1`, right at `2*i+2`. Number of nodes = `2^(height+1) - 1`.

---

### `Tree.generate(height)`

**What it does:** Creates a `Tree` of given `height` with `2^(height+1)-1` random 32‑bit values in `values`.

---

### `Input` (dataclass)

```python
@dataclass
class Input:
    indices: list[int]
    values: list[int]
    rounds: int
```

**What it is:** One batch of walkers: `indices[i]` = node index for walker `i`, `values[i]` = its value. `rounds` is how many rounds to run.

---

### `Input.generate(forest, batch_size, rounds)`

**What it does:** Builds an `Input` with `batch_size` walkers, all starting at index 0 and with random `values`, and the given `rounds`.

---

### `reference_kernel(t: Tree, inp: Input)`

**What it does:** Reference implementation in plain Python. For each round and each batch index `i`:

1. `idx = inp.indices[i]`, `val = inp.values[i]`
2. `val = myhash(val ^ t.values[idx])`
3. `idx = 2*idx + (1 if val%2==0 else 2)`
4. `idx = 0 if idx >= len(t.values) else idx`
5. `inp.values[i] = val`, `inp.indices[i] = idx`

It mutates `inp.indices` and `inp.values` in place.

---

### `build_mem_image(t: Tree, inp: Input) -> list[int]`

**What it does:** Builds the flat memory image the machine and `reference_kernel2` use.

**Layout:**

- `mem[0]` = rounds  
- `mem[1]` = `len(t.values)` (n_nodes)  
- `mem[2]` = `len(inp.indices)` (batch_size)  
- `mem[3]` = `t.height`  
- `mem[4]` = `forest_values_p` (start of tree values)  
- `mem[5]` = `inp_indices_p` (start of indices)  
- `mem[6]` = `inp_values_p` (start of values)  
- `mem[7]` = start index of the “extra” region  

Then: tree values, then indices, then values, then extra room. The three pointers are chosen so the blocks are contiguous in that order.

---

### `reference_kernel2(mem, trace={})`

**What it does:** Same algorithm as `reference_kernel`, but on the flat `mem` layout. Reads `rounds`, `n_nodes`, `batch_size`, and the three pointers from `mem[0..6]`, then for each round and each `i` updates `mem[inp_indices_p+i]` and `mem[inp_values_p+i]`. Uses `myhash_traced` and fills `trace` for `(round, i, "idx")`, `("val")`, `("node_val")`, `("hashed_val")`, `("next_idx")`, `("wrapped_idx")`, and each `("hash_stage", hi)`.

**Yields:** `mem` once at the start (before any rounds) and once at the end (after all rounds). Those yields align with `pause` in the kernel for `do_kernel_test`.

---

## perf_takehome.py – KernelBuilder

### `KernelBuilder.__init__()`

**What it does:** Resets builder state:

- `instrs`: list of instructions (the program).
- `scratch`: `{name: scratch_address}`.
- `scratch_debug`: `{address: (name, length)}` for `DebugInfo`.
- `scratch_ptr`: next free scratch index.
- `const_map`: `{value: scratch_address}` for constants already materialized.

---

### `KernelBuilder.debug_info()`

**What it does:** Returns `DebugInfo(scratch_map=self.scratch_debug)` so the Machine can map scratch addresses to names.

---

### `KernelBuilder.build(slots, vliw=False)`

**What it does:** Turns a list of `(engine, slot)` into a list of instructions. The baseline implementation does **one slot per instruction**:

```python
for engine, slot in slots:
    instrs.append({engine: [slot]})
```

So it does not pack multiple slots into one instruction. `vliw` is ignored. This is the main place to improve for VLIW packing.

---

### `KernelBuilder.add(engine, slot)`

**What it does:** Appends one instruction that contains only that one slot: `self.instrs.append({engine: [slot]})`.

---

### `KernelBuilder.alloc_scratch(name=None, length=1)`

**What it does:** Reserves `length` consecutive scratch words starting at `scratch_ptr`. If `name` is given, records `scratch[name]=addr` and `scratch_debug[addr]=(name, length)`. Increments `scratch_ptr` by `length`, asserts `<= SCRATCH_SIZE`, and returns the starting address.

---

### `KernelBuilder.scratch_const(val, name=None)`

**What it does:** Ensures the constant `val` exists in scratch. If `val` is not in `const_map`, it allocates scratch, emits `("load", ("const", addr, val))`, and stores `const_map[val]=addr`. Returns that scratch address. Reuses the same address if `val` was already requested.

---

### `KernelBuilder.build_hash(val_hash_addr, tmp1, tmp2, round, i)`

**What it does:** Produces a list of `(engine, slot)` that implement `myhash` on the value in scratch at `val_hash_addr`, using `tmp1` and `tmp2` as temporaries. For each `HASH_STAGES` row `(op1, val1, op2, op3, val3)` it emits:

1. `("alu", (op1, tmp1, val_hash_addr, scratch_const(val1)))`  → `tmp1 = val_hash_addr op1 val1`
2. `("alu", (op3, tmp2, val_hash_addr, scratch_const(val3)))`  → `tmp2 = val_hash_addr op3 val3`
3. `("alu", (op2, val_hash_addr, tmp1, tmp2))`                 → `val_hash_addr = tmp1 op2 tmp2`
4. `("debug", ("compare", val_hash_addr, (round, i, "hash_stage", hi)))`

So the value is updated in place in `val_hash_addr`. The `debug` slots are for checking against `value_trace` when `enable_debug` is True.

---

### `KernelBuilder.build_kernel(forest_height, n_nodes, batch_size, rounds)`

**What it does:** Builds the full scalar kernel program (what you are meant to optimize).

1. **Temporaries:** `alloc_scratch` for `tmp1`, `tmp2`, `tmp3`, and for the 7 init vars: `rounds`, `n_nodes`, `batch_size`, `forest_height`, `forest_values_p`, `inp_indices_p`, `inp_values_p`.

2. **Init:** For `i in 0..6`, emits:
   - `("load", ("const", tmp1, i))`  
   - `("load", ("load", self.scratch[init_vars[i]], tmp1))`  
   So it loads `mem[i]` into the corresponding init var. (Note: the second `"load"` uses `tmp1` as the address, which was just set to `i`; so this is `mem[i]` → scratch.)

3. **Constants:** `scratch_const` for 0, 1, 2.

4. **Pause and debug:** `("flow", ("pause",))` and `("debug", ("comment", "Starting loop"))` to match the first `yield` of `reference_kernel2`.

5. **Body:** Allocates `tmp_idx`, `tmp_val`, `tmp_node_val`, `tmp_addr`. For each `(round, i)` it appends slots that:
   - Compute `inp_indices_p + i` into `tmp_addr`, load `mem[tmp_addr]` → `tmp_idx`; debug compare for `(round,i,"idx")`.
   - Same for `inp_values_p + i` → `tmp_val`; debug compare for `(round,i,"val")`.
   - Compute `forest_values_p + tmp_idx` into `tmp_addr`, load `mem[tmp_addr]` → `tmp_node_val`; debug compare for `(round,i,"node_val")`.
   - `tmp_val ^= tmp_node_val`, then `build_hash(tmp_val, tmp1, tmp2, round, i)`; debug compare for `(round,i,"hashed_val")`.
   - `tmp1 = tmp_val % 2`, `tmp1 = (tmp1 == 0)`, `select(tmp3, tmp1, 1, 2)`, `tmp_idx = tmp_idx*2 + tmp3`; debug for `(round,i,"next_idx")`.
   - `tmp1 = (tmp_idx < n_nodes)`, `select(tmp_idx, tmp1, tmp_idx, 0)`; debug for `(round,i,"wrapped_idx")`.
   - Store `tmp_idx` at `inp_indices_p + i` and `tmp_val` at `inp_values_p + i`.

6. **Build and append:** `body_instrs = self.build(body)` then `self.instrs.extend(body_instrs)`.

7. **Final pause:** `self.instrs.append({"flow": [("pause",)]})` to match the final `yield` of `reference_kernel2`.

---

## perf_takehome.py – Tests and Runner

### `BASELINE`

```python
BASELINE = 147734
```

**What it is:** The cycle count of the unoptimized kernel for the default test. Used to compute “speedup over baseline.”

---

### `do_kernel_test(forest_height, rounds, batch_size, seed=123, trace=False, prints=False)`

**What it does:** Runs one full test and checks correctness.

1. Set `random.seed(seed)`, build `Tree` and `Input`, build `mem = build_mem_image(...)`.
2. Create `KernelBuilder`, call `build_kernel(forest.height, n_nodes, batch_size, rounds)`.
3. Create `value_trace = {}` and `Machine(mem, kb.instrs, kb.debug_info(), value_trace=value_trace, trace=trace)`.
4. Iterate over `reference_kernel2(mem, value_trace)`. Each iteration:
   - Call `machine.run()` (runs until the next `pause` or end; `value_trace` is filled by the reference before the first run, and `mem` is updated by the reference after each round).
   - After each `run`, compare `machine.mem[inp_values_p : inp_values_p + len(inp.values)]` to `ref_mem[inp_values_p : ...]`. They must match.
5. Print `machine.cycle` and speedup vs `BASELINE`, and return `machine.cycle`.

---

### `Tests.test_ref_kernels`

**What it does:** Checks that `reference_kernel` and `reference_kernel2` produce the same indices and values. It runs `reference_kernel` on `(Tree, Input)`, runs `reference_kernel2` on `mem` (which `build_mem_image` had populated from the same `Tree` and `Input`), and asserts `inp.indices`/`inp.values` match the corresponding slices of `mem`.

---

### `Tests.test_kernel_trace`

**What it does:** Calls `do_kernel_test(10, 16, 256, trace=True, prints=False)` to produce `trace.json` for visualization.

---

### `Tests.test_kernel_cycles`

**What it does:** Calls `do_kernel_test(10, 16, 256)` and reports cycle count and speedup. This is the main performance test in this file.

---

## Quick Reference: Instruction Slot Formats

### ALU

- `(op, dest, a1, a2)` → `scratch[dest] = (scratch[a1] op scratch[a2]) % 2**32`  
  `op` in: `+`, `-`, `*`, `//`, `cdiv`, `^`, `&`, `|`, `<<`, `>>`, `%`, `<`, `==`

### Load

- `("load", dest, addr)` → `scratch[dest] = mem[scratch[addr]]`
- `("load_offset", dest, addr, offset)` → `scratch[dest+offset] = mem[scratch[addr+offset]]`
- `("vload", dest, addr)` → `scratch[dest..dest+7] = mem[scratch[addr]..scratch[addr]+7]`
- `("const", dest, val)` → `scratch[dest] = val % 2**32`

### Store

- `("store", addr, src)` → `mem[scratch[addr]] = scratch[src]`
- `("vstore", addr, src)` → `mem[scratch[addr]+j] = scratch[src+j]` for j in 0..7

### Flow

- `("select", dest, cond, a, b)` → `scratch[dest] = scratch[a] if scratch[cond]!=0 else scratch[b]`
- `("pause",)` → core PAUSED
- `("cond_jump", cond, addr)`, `("cond_jump_rel", cond, offset)`, `("jump", addr)`, `("jump_indirect", addr)`

### Debug (in `step`, only when `enable_debug`)

- `("compare", loc, key)` → assert `scratch[loc] == value_trace[key]`
- `("vcompare", loc, keys)` → assert `scratch[loc:loc+8] == [value_trace[k] for k in keys]`

---

*End of Code Reference*
