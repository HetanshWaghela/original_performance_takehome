---
name: Perf Takehome From Zero
overview: "A beginner-friendly walkthrough: what the problem does in plain English, how memory and a “fake CPU” work, what a cycle is, then the tree, the hash, the machine’s rules, and how the code produces instructions. Each idea is explained simply with analogies and small examples."
todos: []
---

# Understanding the Performance Take-Home — From the Very Beginning

You’re going to learn this one small idea at a time. Each section builds on the one before. If something is fuzzy, stop and re-read that part before going on.

---

## Step 1: What is “memory” in this project?

**Idea:** The machine has a **big list of numbers**. Each place in the list has a **position number** (0, 1, 2, 3, …). That’s it.

- Like a **row of lockers**: locker 0, locker 1, locker 2, …  
- Each locker holds one integer (a whole number).  
- To “read” from memory: “What’s in locker 5?” → you get the number stored at position 5.  
- To “write” to memory: “Put 42 in locker 5.” → the number at position 5 becomes 42.

So when we say “memory” or `mem`, we mean: **a Python list of integers**. `mem[7]` is “the number stored at position 7.”

---

## Step 2: What is a “program” and a “cycle”?

**Idea:** The machine runs a **list of instructions** one by one. Each instruction is a **bundle of small tasks**. Running **one** instruction takes **one cycle**.

- **Cycle** = one “tick” of the clock. The machine does one instruction per tick.  
- **Program** = the full list of instructions.  
- **Faster program** = **fewer cycles** to get the right answer.

So: **your job is to change the program so it needs fewer cycles** (fewer instructions, or more work per instruction — we’ll see that later).

---

## Step 3: What does this program actually *do*? (The story)

**In plain English:**

We have:

1. **A tree of numbers**  

   - Imagine a family tree, but each person (node) has a number.  
   - The tree is **binary**: each node has at most 2 children (left and right).  
   - The tree is **full**: every level is completely filled.  
   - All these numbers are stored in a **flat list** in memory (we’ll see how in Step 5).

2. **A batch of “walkers”**  

   - Each walker has:  
     - a **position** (which node they’re on: 0 = root),  
     - a **value** (an integer that gets updated as they move).  
   - We have `batch_size` walkers (e.g. 256).

3. **What happens each “round” for each walker**  

   - Read the number at the current node.  
   - **Mix** the walker’s value with that number (XOR, then a hash — we’ll simplify in a moment).  
   - Look at the **new** value:  
     - if it’s **even** → go **left** (one formula),  
     - if it’s **odd** → go **right** (another formula).  
   - If you’d go past the bottom of the tree, you **wrap** back to the root (position 0).  
   - **Save** the new position and new value for that walker.

We do many **rounds** (e.g. 16). After all rounds, the **values** of all walkers are the “answer” we care about. The machine’s memory must match what this description produces.

**Why it matters:** The *correct* behavior is fully defined by the reference (the Python functions `reference_kernel` / `reference_kernel2`). Your job is to **implement that same behavior** on a **weird simulated machine**, and then **make it use fewer cycles**.

---

## Step 4: The tree in a flat list (no pointers, no real “tree” in memory)

The tree is **not** stored with pointers or node objects. It’s a **list**. The **index** in the list tells you who is whose child.

**Rules:**

- Root is at **index 0**.  
- For any node at index **i**:  
  - **left child** is at index **2*i + 1**  
  - **right child** is at index **2*i + 2**

**Example (height 1 → 3 nodes):**

- Index 0: root  
- Index 1: left child of 0  
- Index 2: right child of 0  

So the “tree” is just: `[value_at_root, value_at_1, value_at_2]`.

**Example (height 2 → 7 nodes):**

```
         [0]   root
        /   \
      [1]   [2]
      / \   / \
    [3][4][5][6]
```

- 0’s children: 1 (left), 2 (right)  
- 1’s children: 3 (left), 4 (right)  
- 2’s children: 5 (left), 6 (right)  
- 3,4,5,6 are leaves (no children in this small tree).

Total nodes = 2^(height+1) − 1. For height 2: 2^3 − 1 = 7.

**Important:**

- “Go left” from node `i` means: new index = **2*i + 1**.  
- “Go right” means: new index = **2*i + 2**.  
- The “wrap” rule: if that new index is **≥ number of nodes**, set it to **0**.

---

## Step 5: Where is everything in memory? (The layout)

The machine gets **one big `mem` array**. The first few positions are **header**: they tell the program the sizes and **where** the real data starts.

**Layout (simplified):**

| Position | What it is |

|----------|------------|

| 0 | number of rounds |

| 1 | number of nodes in the tree |

| 2 | batch size (number of walkers) |

| 3 | tree height (we can derive nodes from this) |

| 4 | **pointer** to the tree’s values ( = the index in `mem` where `tree.values[0]` is) |

| 5 | **pointer** to the walkers’ **positions** (indices) |

| 6 | **pointer** to the walkers’ **values** |

“Pointer” here = “the `mem` index where that block starts.”

- **Tree values:** from `mem[4] `we get a number, call it `P`. The tree is `mem[P]`, `mem[P+1]`, … `mem[P + n_nodes - 1]`.  
- **Walker positions:** from `mem[5] `we get `Q`. The positions are `mem[Q]`, `mem[Q+1]`, … `mem[Q + batch_size - 1]`.  
- **Walker values:** from `mem[6] `we get `R`. The values are `mem[R]`, `mem[R+1]`, … `mem[R + batch_size - 1]`.

So: **header in 0–6**, then the actual arrays start at the stored pointers. The program reads 0–6 first, then uses those pointers to find the tree and the two arrays.

---

## Step 6: The hash (mixing numbers)

The algorithm uses `val = myhash(val ^ node_val)`. For understanding:

- **^** is XOR (bitwise).  
- **myhash** is a fixed recipe that **mixes** a 32‑bit number so it looks more “random.”  
- It does **6 steps** (stages). Each stage: combine the current number with some constants using +, ^, <<, >>; the result is taken mod 2^32 so it stays 32‑bit.

You don’t need to memorize the formula. You only need to know:

- There is a function `myhash(x)` in Python that is the **truth**.  
- The machine has no “call myhash” instruction. So we must **implement** those 6 stages using the machine’s **arithmetic** and **logic** instructions (add, xor, shift, etc.).  
- `build_hash` in the code is what emits those machine instructions for one `myhash` call.

---

## Step 7: Scratch vs main memory

The machine has **two** places to hold numbers:

1. **Main memory (`mem`)**  

   - Big. Holds the tree, the batch’s positions and values, and extra space.  
   - This is what we described in Step 5.  
   - **Load** = copy from `mem` into the machine’s “working” area.  
   - **Store** = copy from the “working” area into `mem`.

2. **Scratch (like registers)**  

   - A **separate** array of 1536 integers (in this setup).  
   - Think of it as the **desk** the CPU uses while it works:  
     - Before an operation: values must be **in scratch** (either already there or loaded from `mem`).  
     - After an operation: the result is written **to scratch** (at a chosen “address” = index).  
   - To change `mem`, you eventually have to **store** from scratch to `mem`.

So:

- We **load** from `mem` **into scratch**.  
- We do **math and logic** in scratch (e.g. add, xor, multiply).  
- We **store** from scratch **into `mem`** when we want to update the batch’s positions or values.

**Address:**

- In `mem`, “address” = index in the list (0, 1, 2, …).  
- In scratch, “address” = index in the scratch array (0, 1, 2, … up to 1535).  
- A **load** or **store** instruction often uses an address that is **stored in scratch**. Example: “load into scratch[10] from mem[scratch[8]]” — so scratch[8] holds the `mem` index to read from.

---

## Step 8: What is an “engine” and a “slot”?

The simulated CPU is **not** like “one thing does one add per cycle.” It has several **engines** that can each do **several** small operations in **one cycle**.

**Engine** = a type of work:

- **alu** — arithmetic and logic: add, subtract, multiply, xor, shifts, compare, etc.  
- **load** — bring a value from `mem` into scratch (or load a constant into scratch).  
- **store** — write from scratch to `mem`.  
- **flow** — control: things like “if this is 0, pick A, else pick B,” or pause, or jump.  
- **valu** — vector (SIMD) math; we’ll get to that later.  
- **debug** — only for checking; the real grading ignores it.

**Slot** = one single task for one engine. For example:

- One **alu** slot: “add the numbers in scratch[3] and scratch[5], put the result in scratch[7].”  
- One **load** slot: “load from mem[scratch[12]] into scratch[9].”

**One instruction** = one cycle = **one bundle** of slots. The bundle can have:

- several **alu** slots,  
- up to 2 **load** slots,  
- up to 2 **store** slots,  
- up to 1 **flow** slot,  
- etc.

**Limits (how many slots per engine per instruction):**

- alu: 12  
- load: 2  
- store: 2  
- flow: 1  
- valu: 6 (we’ll use this when we talk about vectors)

So in **one cycle** you could, for example, do **many** adds and a couple of loads and a couple of stores, all in parallel. That’s the key to going faster: **put more work into each instruction** instead of using one slot per instruction.

---

## Step 9: What does one “instruction” look like in code?

In Python, one instruction is a **dict** (dictionary):

- **Keys** = engine names: `"alu"`, `"load"`, `"store"`, `"flow"`, etc.  
- **Value** for each key = a **list of slots**. Each slot is a tuple: `(operation, arg1, arg2, ...)`.

**Example (fake, to show shape):**

```python
{
  "alu": [ ("+", dest, a, b), ("-", dest2, x, y) ],   # 2 alu slots
  "load": [ ("load", dest, addr) ]                     # 1 load slot
}
```

Meaning in that one cycle:

- alu: (1) `scratch[dest] = scratch[a] + scratch[b]`, (2) `scratch[dest2] = scratch[x] - scratch[y]`  
- load: `scratch[dest] = mem[scratch[addr]]`

The exact format for each op (what is “dest”, “a”, “b”, etc.) is defined in `problem.py` in the `Machine` class (the `alu`, `load`, `store`, `flow` functions). For now, the idea is: **one dict = one instruction = one cycle**, and it can contain several slots for several engines.

---

## Step 10: Reads and writes in the same cycle

**Rule:** In one cycle, **all reads happen using the old values**. All **writes** are collected and applied **only at the end** of the cycle.

So:

- If slot 1 does `scratch[5] = scratch[3] + scratch[4]` and slot 2 does `scratch[6] = scratch[5] * 2`, then slot 2 **still sees the old** `scratch[5]` from before this cycle. Your result in `scratch[5]` appears “after” the cycle.  
- Same for `mem`: a load always sees the `mem` from the start of the cycle; stores apply at the end.

That avoids “which order do we run the slots in?” — you can imagine they all run in parallel; the only rule is: **reads = old state, writes = at end of cycle**.

---

## Step 11: What is KernelBuilder and “build”?

**KernelBuilder** (in `perf_takehome.py`) is the thing that **creates the program** (the list of instructions) that the machine will run.

- **`alloc_scratch(name, length=1)`** — Reserve `length` scratch positions, give them a name for debugging. It returns the starting index (address) in scratch.  
- **`scratch_const(val)`** — “I need the constant `val` in scratch.” It要么 reuses an existing slot that already has `val`, or allocates one and emits a **load-constant** instruction to put `val` there. Returns the scratch address.  
- **`add(engine, slot)`** — Append **one instruction** that contains **only that one slot**. So `add("alu", ("+", d, a, b))` adds an instruction like `{"alu": [("+", d, a, b)]}`.

**`build(slots)`** (used for the main loop body):

- `slots` is a list of `(engine, slot)` pairs.  
- The **baseline** `build` does: for each `(engine, slot)`, create an instruction `{engine: [slot]}` and add it. So **one slot per instruction** = one cycle per small operation. That’s why the baseline is slow: it hardly uses the fact that we can do 12 alu + 2 load + 2 store in one cycle.

---

## Step 12: What does `build_kernel` do (in words)?

It **builds the full program** that implements the same algorithm as `reference_kernel2`. High level:

1. **Setup**  

   - Allocate scratch for temporaries (tmp1, tmp2, tmp3, etc.) and for the 7 “init” variables (rounds, n_nodes, batch_size, forest_height, and the 3 pointers).  
   - Emit instructions to **load** the 7 header values from `mem` into those scratch locations (using `mem[0]`…`mem[6]`).  
   - Add a **pause** (and a debug comment) to match the reference’s first `yield`.

2. **Main work (the big loop)**  

   - For each `(round, i)` we need to do one “step” for walker `i` in round `round`.  
   - The code **unrolls** this: it doesn’t emit a “loop” in the machine; it emits a **long sequence** of instructions, one step per `(round, i)`.  
   - For each step it appends slots that:  
     - Compute `inp_indices_p + i` and load `idx` from `mem[inp_indices_p + i]`.  
     - Compute `inp_values_p + i` and load `val` from `mem[inp_values_p + i]`.  
     - Compute `forest_values_p + idx` and load `node_val` from `mem[forest_values_p + idx]`.  
     - `val = val ^ node_val`, then `build_hash` to do `val = myhash(val)`.  
     - `val % 2`, compare to 0, then **select** 1 or 2; then `idx = 2*idx + that`, and **wrap** if `idx >= n_nodes`.  
     - Store `idx` and `val` back to `mem[inp_indices_p + i]` and `mem[inp_values_p + i]`.  

All those slots are passed to **`build`**, which in the baseline turns each into its **own** instruction. So we get **one cycle per slot** — lots of cycles.

3. **End**  

   - One more **pause** to match the reference’s last `yield`.

---

## Step 13: The reference and how we check “right”

- **`reference_kernel`** — Pure Python: takes `Tree` and `Input`, updates `inp.indices` and `inp.values` in place. Easy to read.  
- **`reference_kernel2`** — Same algorithm, but on the **flat `mem`** layout. It reads `mem[0..6]`, then reads/writes the arrays at the pointers. It **yields** `mem` at the start and after all rounds; those yields are used for debugging and to align with **pause** in your program.  
- **Correctness:** We run your program on a **copy** of `mem`, run `reference_kernel2` on the **original** `mem`, and compare `mem[inp_values_p : inp_values_p + batch_size]` at the end. They must be equal. So: **same final walker values**.

---

## Step 14: Why is the baseline slow and how do we speed it up?

**Why it’s slow:**

- `build` does **one (engine, slot) → one instruction**.  
- So each add, each load, each store, each flow op = **one cycle**.  
- Many steps × many ops per step = huge cycle count (e.g. 147734 for the default test).

**How to speed up (conceptually):**

1. **Packing (use the slot limits)**  

   - In one instruction we are **allowed** up to 12 alu, 2 load, 2 store, 1 flow.  
   - We should **combine** many of the small ops from the **same** step (or from different steps that don’t depend on each other) into **one** instruction, until we hit those limits.  
   - That alone can cut cycles a lot: we do more work per cycle.

2. **Vectorization (SIMD)**  

   - **Vector** = 8 numbers in a row in scratch (VLEN=8).  
   - **vload** = load 8 consecutive `mem` cells into 8 consecutive scratch cells.  
   - **vstore** = store 8 consecutive scratch cells into 8 consecutive `mem` cells.  
   - **valu** = do the same operation on 8 pairs of numbers (e.g. 8 adds in one slot).  
   - If we can treat **8 walkers at a time** (e.g. `i`, `i+1`, … `i+7`) and do their loads, hashes, and stores using **vector** ops, we do **8 steps per vector instruction** instead of 8 scalar instructions.  
   - The hard part: each walker has its **own** `idx`, so `forest_values_p + idx` is different for each. The machine has no “gather” from 8 different addresses in one slot; we have to design how to do those 8 loads (e.g. with several load slots or with different tricks).

3. **Smarter structure**  

   - The baseline **fully unrolls** the (round, batch) loop into one giant sequence. We could instead emit an **inner loop** using **jump** / **cond_jump** so the program is shorter (fewer instructions), as long as we don’t add so much loop overhead that we lose.

---

## Step 15: Order to learn the code

1. Read **`reference_kernel`** in `problem.py`: one round, one `i`. See: read idx/val, read node, hash, branch, wrap, write.  
2. Read **`build_mem_image`**: see how `mem[0..6]` and the three blocks are filled.  
3. Skim **`reference_kernel2`**: same steps, but on `mem` and with `yield`.  
4. In **`Machine`** in `problem.py`: look at **`step`** — it loops over engines and slots and calls `alu`, `load`, `store`, `flow`. Then open **`alu`**, **`load`**, **`store`**, **`flow`** and see the tuple formats: `(op, dest, a, b)` etc.  
5. In **`KernelBuilder`**: **`alloc_scratch`**, **`scratch_const`**, **`add`**, **`build`**.  
6. In **`build_kernel`**: follow **one** `(round, i)` — from the first `body.append` for that `i` to the last **store**. Map each append to: “this is an alu add,” “this is a load,” “this is a flow select,” etc.  
7. In **`build_hash`**: see how one `HASH_STAGES` row becomes several **alu** slots and a **debug** slot.

---

## Tiny picture of the flow

```
mem (header + tree + indices + values)
         |
         |  load pointers and then load/store
         v
   [Scratch: 1536 words]  <-- alu, flow work here
         |
         |  store
         v
mem (we only care that indices/values are updated correctly for the next round
     and that final values match the reference)
```

---

## Words to remember

- **memory (`mem`)** = big list of integers; index = address.  
- **scratch** = small “desk” of 1536 integers; we do math here and load/store to `mem`.  
- **cycle** = one tick; one instruction.  
- **instruction** = one dict: `{ "alu": [...], "load": [...], "store": [...] }` — many slots in one cycle.  
- **engine** = kind of work (alu, load, store, flow, valu).  
- **slot** = one small op (one add, one load, one store, etc.).  
- **KernelBuilder** = builds the list of instructions; `build` in the baseline turns one slot into one instruction.  
- **Tree** = full binary tree stored in a list; children of `i` at `2*i+1` and `2*i+2`.  
- **myhash** = 6-stage mix; implemented with many alu slots in `build_hash`.  
- **Faster** = fewer cycles = more slots per instruction (packing) and/or 8-at-a-time vectors (SIMD).

If you want to go deeper on **one** of these (e.g. “explain only the tree” or “explain only scratch vs mem”), say which number and we can zoom in there next.