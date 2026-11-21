---
title: 'Formal Verification: What It Is, Why AI Makes It Practical, and What a Proof Does Not Tell You'
date: 2026-11-21
draft: false
categories: ['notes']
tags: ['formal-verification', 'lean', 'verus', 'kani', 'tla+', 'llm']
summary: 'Why tests miss bugs, how AI makes proofs cheap, how Lean, Verus, TLC and Kani find bugs (down to the SAT clauses), what I saw when an agent used five tools on one spec, and why a proof is only as good as its spec.'
ShowToc: true
---

This post is the written version of a meetup talk for software developers. It has five parts:

1. The problem: tests check some inputs, and bugs hide in the rest.
2. Why now: AI makes proofs much cheaper.
3. What formal verification is: three families of methods, with examples in Lean, Verus, TLA+ and Kani. For some examples, I go all the way down to what the SAT or SMT solver sees.
4. My experiment: one spec, five tools.
5. Limits: what a proof does not tell you.

## AI now helps with formal verification

Two results from fall 2026 show that AI is changing how practical formal verification is.

**Boris Cherny: bugs from a model** ([post on X, Sep 22, 2026](https://x.com/bcherny/status/2102543349102338309)). Boris Cherny opened 16 PRs that fix bugs and race conditions, after a few short prompts to Claude (Opus 5.5). He had Claude model the Claude Agent SDK in **Lean**, and sometimes in **TLA+**, to check data flow, concurrency and state. He does not know either language well, so Claude wrote the models. A human reviewer would probably miss the bugs that the models found.

**Anthropic: proof at scale** ([Anthropic, Oct 2026](https://www.anthropic.com/research/formal-verification)). This work proved **239 entry points** in **123 open-source libraries** safe for **all inputs under 64 MB**, not for a sample. It also found hundreds of bugs on the way. The result is not "many tests saw no crash". The result is that no input under 64 MB can make these entry points crash, and a small program checked every step of the argument.

## 1. The problem

### Tests check some inputs; bugs hide in the rest

The first example is a 32-bit `abs` function. All of its tests pass:

```c
int abs(int x) {
    return x < 0 ? -x : x;
}

assert abs(-5) == 5;   // pass
assert abs(0)  == 0;   // pass
assert abs(7)  == 7;   // pass

abs(-2147483648) == -2147483648   // negative!
```

In 32-bit two's complement, −2,147,483,648 has no positive twin, so `-x` wraps around to the same negative number. Only **1** of the **4.3 billion** possible inputs fails. Thus one million random tests find this bug with a chance of only about **0.023%**.

The second example is the binary search in the JDK. This bug [hid for about 9 years](https://research.google/blog/extra-extra-read-all-about-it-nearly-all-binary-searches-and-mergesorts-are-broken/) (Joshua Bloch, Google Research blog, 2006):

```java
int low = 0, high = a.length - 1;
while (low <= high) {
    int mid = (low + high) / 2;  // can overflow
    if      (a[mid] < key) low  = mid + 1;
    else if (a[mid] > key) high = mid - 1;
    else return mid;
}
// fix: int mid = low + (high - low) / 2;
```

As math, `(low + high) / 2` is correct. But when `low + high` is more than 2,147,483,647, the sum wraps to a negative number, and `a[mid]` crashes. The bug needs an array of about $2^{30}$ elements (more than 4 GB). Test machines did not have that much memory, so the bug stayed in the JDK until 2006.

A proof is different from a test, because a proof reasons about **all** inputs at once. A proof finds both of these bugs directly, without a 4 GB array.

### Concurrency bugs are hard to test

Concurrency adds a second problem: the bug depends on the order of events. In the next example, two threads both run `t := x; x := t + 1`. The counter starts at 0, so we expect 2 at the end. But one order of steps gives a different result:

| Step | Thread 1 | Thread 2 | x |
|---|---|---|---|
| 1 | read: t₁ = 0 | | 0 |
| 2 | | read: t₂ = 0 | 0 |
| 3 | write: x = t₁ + 1 | | 1 |
| 4 | | write: x = t₂ + 1 | **1** |

**One update is lost.** There is no error, no exception and no log line.

Real systems have the same type of bug, but with much longer traces. AWS [modeled the replication and fault-tolerance design of DynamoDB in TLA+](https://cacm.acm.org/magazines/2015/4/184701-how-amazon-web-services-uses-formal-methods/fulltext) (Newcombe et al., CACM 2015). The model checker found a bug that could **lose data**. The bug needed one exact order of events, with failures and recoveries interleaved with normal work. The shortest trace had **35 steps**. The bug had passed design reviews, code reviews and testing, including fault-injection tests that reordered messages.

## 2. Why now: AI changes the cost

### Proofs used to cost too much

If proofs find such bugs, why do few teams use them? The answer is the cost. [seL4](https://trustworthy.systems/projects/OLD/l4.verified/numbers) is an OS microkernel whose C code is proved to do exactly what its spec says:

| | Lines |
|---|---|
| Verified C code | ~7,500 |
| Isabelle proof | ~200,000 |

The proof is about 27 times as long as the code. Design, implementation and proof together took about **25 person-years**. Because of this cost, proofs were used for decades only where a bug is very expensive: kernels, compilers, crypto and aircraft.

### The AI writes the proof; the checker keeps it honest

AI changes this cost, because an AI agent can write the long proof. The agent is fast but sometimes wrong. That is not a problem, because the agent works in a loop with a proof checker:

1. The **AI agent** writes a proof attempt.
2. The **proof checker** checks every step. The checker is a small, strict program that people have reviewed for years.
3. If the checker accepts the proof, the proof is valid for all inputs.
4. If the checker rejects the proof, it gives an **error message** with the exact goal that it could not close. The agent tries again with this error.

So we do not need to trust the AI. We only need to trust the checker.

There is one important limit. The checker confirms that the proof matches the spec. It cannot tell you whether the spec (the rules you wrote down) says what you want. Part 5 comes back to this.

## 3. What is formal verification

### Three approaches

Formal verification is not one method. It has three families, and the first two have two types each:

| Family | Type | How it works | You provide | Guarantee | Languages (tools) |
|---|---|---|---|---|---|
| **1. Deductive proof** | **1a** Kernel-checked proof | A small kernel checks every step, also steps found by tactics or solvers | The proof, or its main steps | All executions; trust only the kernel | **Lean 4**, **Bend**, Rocq, Isabelle/HOL |
| | **1b** Verification conditions + SMT | Annotations become logic formulas; an SMT solver proves them | Pre/postconditions, invariants | All executions; also trust the solver | F*, Dafny, **Rust (Verus)**, SPARK (GNATprove), TLA+ (TLAPS), P (P Verifier) |
| **2. State-space search** | **2a** Explicit enumeration | Visits reachable states one by one | A model + a finite configuration | Complete for that configuration; counterexample | **TLA+ (TLC)**, P (default checker) |
| | **2b** Symbolic search | SAT, SMT or BDDs handle many states at once | A model or code + a bound | Complete within the bound; counterexample | Alloy, TLA+ (Apalache), P (PSym), Isabelle/HOL (Nitpick), **Rust (Kani)** |
| **3. Abstract interpretation** | | Computes a larger set that contains every execution | Usually nothing | Sound; may give false alarms | SPARK (flow analysis) |

The tools in bold are the ones I use in this post and in my experiment. The table uses three terms from logic tools:

- **SAT solver**: decides if a true/false formula can be true.
- **SMT solver** (Satisfiability Modulo Theories, for example Z3): a SAT solver plus rules for numbers, arrays and bit vectors.
- **BDD** (binary decision diagram): a compact graph for a true/false function, so one BDD can stand for many states.

### Deductive proof: kernel-checked vs. verification conditions + SMT

The two types of deductive proof differ in what you write and what you trust.

**1a, kernel-checked proof.** You (or an AI) write the spec and the proof, or the main steps of the proof. Tactics fill in the small steps. Tactics are helpers, and you do not need to trust them, because a small kernel then checks every step. The result is "proved", or an open goal.

**1b, verification conditions + SMT.** You (or an AI) write code with annotations such as `ensures`, `invariant` and `decreases`. A VC (verification condition) generator turns the code and the annotations into logic formulas. Then an SMT solver proves each formula automatically. The result is "proved", "could not prove", or a timeout.

A **verification condition** is a formula that is true only if the code meets its annotation. For example, `x = x + 1` with `requires x ≥ 0` and `ensures x > 0` gives the VC `x ≥ 0 ⟹ x + 1 > 0`.

| | 1a · kernel-checked | 1b · VCs + SMT |
|---|---|---|
| **You write** | The proof, or its main steps | Only annotations: pre/postconditions, loop invariants |
| **You trust** | One small kernel | The VC generator and the SMT solver (both large) |
| **When it fails** | The open goal: exactly what is left to prove | "Could not prove", or a timeout; often not clear why |

### Kernel-checked: binary search in Lean

The first example of type 1a is a binary search in Lean. The pseudo-code below finds `x` in the range `[lo, hi)` of the array `a`:

```text
search(a, x, lo, hi):
  if lo < hi:
    mid ← lo + (hi - lo) / 2
    if mid < size of a:
      if a[mid] = x: return mid
      if a[mid] < x: return search(a, x, mid + 1, hi)  // drop left
      else:          return search(a, x, lo, mid)      // drop right
    else: return "none"
  else: return "none"
```

The search keeps a range where `x` can still be. It looks at the middle element and drops half of the range. The test `mid < size of a` is a safety check before the code reads `a[mid]`.

The spec is two theorems. Each theorem says what must be true:

- **`search_some`** (a found index is right): if `search(a, x, lo, hi)` returns an index `i`, then `i` is a valid index of `a` and `a[i] = x`. This holds for every array, sorted or not.
- **`search_none`** ("not found" is right): if `a` is sorted, `hi ≤ size of a`, and `search(a, x, lo, hi)` returns "none", then no `i` with `lo ≤ i < hi` has `a[i] = x`.

### The proof of `search_none`

The proof has one case for each way that `search` can end. When `search` calls itself on a smaller range, the proof may use the theorem for that smaller call. This is **induction**, and the assumption about the smaller call is the **induction hypothesis**. These are the five cases:

1. `return mid`: the result is not "none", so there is nothing to prove.
2. `a[mid] < x`, and the recursive call returns "none": proof by contradiction (see below).
3. `a[mid] > x`, and the recursive call returns "none": the mirror of case 2.
4. `mid ≥ size of a`: this cannot happen, because `mid < hi ≤ size of a`.
5. `lo ≥ hi`: the range is empty, so there is no `i` to check.

Case 2 is the interesting one. We know that `a[mid] < x` and that `search(a, x, mid+1, hi)` returned "none". We must show that no `i` with `lo ≤ i < hi` has `a[i] = x`. The proof is by contradiction: take such an `i`, assume `a[i] = x`, and show that this is impossible. There are two sub-cases:

- **If `i ≥ mid + 1`** (the right part): the induction hypothesis on `search(a, x, mid+1, hi)` says `a[i] ≠ x`. This is a contradiction.
- **If `i ≤ mid`** (the left part): `a` is sorted, so `a[i] ≤ a[mid]`. Because `a[mid] < x`, we get `a[i] < x`. This is a contradiction with `a[i] = x`.

The kernel checks every step of all five cases. So `search_none` holds for **every** sorted array and every `x`, not only for tested ones.

### Lean finds the abs overflow with a theorem

Next, back to the `abs` example from Part 1, now in Lean (`abs-naive/Abs.lean`, `AbsSpec.lean`). The first version is a direct copy of the C code, plus the same tests:

```lean
def abs (x : Int32) : Int32 :=
  if x < 0 then -x else x

#eval abs (-5)           -- 5
#eval abs 0              -- 0
#eval abs 7              -- 7
#eval abs (-2147483648)  -- -2147483648
```

Lean accepts this code with no error and no warning. The reason is that for a `def`, Lean only checks that the types match and that the function terminates. In Lean, `Int32` negation is **defined to wrap**, like the CPU: `-(-2147483648) = -2147483648`. So `-x` is never an error. Also, nothing in this file says what the result should be. The `#eval` lines are only tests, and they miss the bug like the tests in Part 1.

The second version adds one theorem. `abs_nonneg` says that the result is 0 or more for every `x`. This is the property we had in mind, but never wrote down:

```lean
def abs (x : Int32) : Int32 :=
  if x < 0 then -x else x

theorem abs_nonneg (x : Int32)
    -- (h : x ≠ Int32.minValue)
    : 0 ≤ abs x := by
  unfold abs
  split <;> bv_decide
```

Now Lean finds the bug:

```text
error: The prover found a counterexample
x = -2147483648
```

The proof has three steps. `unfold abs` replaces `abs x` with its body. `split` makes one goal per branch of the `if`, for example "if `x < 0`, then `0 ≤ -x`". Then `bv_decide` handles each goal:

1. `x` becomes 32 Boolean variables, and `-x` and `<` become logic circuits.
2. A SAT solver looks for bits where `x < 0` but `-x < 0`.
3. If no such bits exist, the goal is proved. Lean checks the solver's certificate. If the solver finds bits, they are `1000…0`, which Lean prints as `x = -2147483648`.

The commented-out line is the fix. With `h : x ≠ Int32.minValue`, the only bad bit pattern is excluded, and the theorem is proved. When the solver says "unsat", it also writes a certificate (LRAT). An LRAT checker, which is written and proved correct in Lean, checks that certificate. So we do not have to trust the SAT solver. (The checker runs as compiled code through `Lean.ofReduceBool`, so we do trust Lean's compiler, not only the kernel.)

The next three sections show these steps with 3 bits instead of 32.

### bv_decide turns the goal into a circuit

In the `x < 0` branch, the goal is:

```text
x < 0  →  0 ≤ -x
```

`bv_decide` wants to prove this goal for every `x`. So it asks the SAT solver the opposite question: is there an `x` that breaks the goal?

```text
x < 0  ∧  -x < 0      -- an x that breaks the goal
```

If no such `x` exists, the goal is proved. If one exists, it is a counterexample. I show the steps with 3 bits (values −4 to 3). `Int32` has 32 bits, and the steps are the same.

{{< figure src="bv-circuit.svg" alt="Circuit for -x = NOT x + 1 with 3 bits, laid out like column addition" caption="The circuit for `-x` with 3 bits. The red boxes are the two goals." >}}

The circuit is drawn like addition on paper. Each bit of `x` is one true/false value, and each column is one bit. `-x` is `NOT x` plus 1. So each column first has a NOT gate, then one adder cell. The cell adds two bits: XOR gives the sum bit, and AND gives the carry bit. Bit 0 adds the 1, bit 1 adds the carry `c1`, and bit 2 adds the carry `c2`. The last carry `c3` does not fit, so the circuit drops it. This is where the overflow happens.

"Less than 0" needs no circuit, because for signed numbers it is just the top bit (the sign bit). So the two goals are `x2 = 1` and `r2 = 1`.

The same circuit as equations, one line per column (⊕ = XOR, ∧ = AND):

```text
        sum               carry
bit 0:  r0 = ¬x0 ⊕ 1      c1 = ¬x0
bit 1:  r1 = ¬x1 ⊕ c1     c2 = ¬x1 ∧ c1
bit 2:  r2 = ¬x2 ⊕ c2     (dropped)
goals:  x2 = 1            r2 = 1
```

In bit 0, the carry is `¬x0 ∧ 1`, which is just `¬x0`.

The solver does not try the 8 values one by one. Instead, it starts from the goals and follows what is forced. `x2 = 1` makes `¬x2 = 0`, so `r2` equals `c2`, and the goal `r2 = 1` forces `c2 = 1`. An AND is 1 only when both inputs are 1, so `x1 = 0` and `c1 = 1`. Because `c1 = ¬x0`, we get `x0 = 0`. No guess was needed. This method is called **unit propagation**.

The result is `x = 100`, which is −4, the smallest 3-bit number. Its negation +4 does not fit in 3 bits, so it wraps back to −4. This is the same bug as −2147483648 for `Int32`.

### From a gate to clauses

A SAT solver does not read circuits. It reads a list of **clauses**. A clause is an OR of values (`¬` = NOT), and all clauses must be true at once. This form is called CNF.

To get the clauses, first give every wire a name, so that each gate is an equation `y = f(a, b)`. The equation holds when two rules hold: **if `y`, then `f`**, and **if `f`, then `y`**. Each rule becomes one clause, because "if P, then Q" is the same as `¬P ∨ Q`. The three gate types give these clauses:

**NOT, `c1 = ¬x0`**: 2 clauses. Together they say that `c1` and `x0` never have the same value.

```text
if c1, then ¬x0   →  (¬c1 ∨ ¬x0)
if ¬x0, then c1   →  ( x0 ∨  c1)
```

**AND, `c2 = ¬x1 ∧ c1`**: 3 clauses. If `c2` is 1, both inputs must be 1, which gives two clauses. If both inputs are 1, `c2` must be 1. De Morgan's rule, `¬(a ∧ b) = ¬a ∨ ¬b`, turns this third rule into a clause.

```text
if c2, then ¬x1         →  (¬c2 ∨ ¬x1)
if c2, then c1          →  (¬c2 ∨  c1)
if ¬x1 ∧ c1, then c2    →  ¬(¬x1 ∧ c1) ∨ c2
                        →  ( x1 ∨ ¬c1 ∨ c2)
```

**XOR, `r2 = ¬x2 ⊕ c2`**: 4 clauses. For XOR it is easier to go row by row through the truth table. There are four input rows, and for each row, one clause forbids the wrong output. For example, row 1 says: "if `x2 = 0` and `c2 = 0`, then `r2` is not 0."

```text
x2 c2 → r2   clause
 0  0 → 1    ( x2 ∨  c2 ∨  r2)
 0  1 → 0    ( x2 ∨ ¬c2 ∨ ¬r2)
 1  0 → 0    (¬x2 ∨  c2 ∨ ¬r2)
 1  1 → 1    (¬x2 ∨ ¬c2 ∨  r2)
```

Then all gate clauses and the two goals, `(x2)` and `(r2)`, are joined by AND. This gives **17 clauses over 8 values**: 3 for `x`, 2 carries and 3 for `-x`. Each gate adds only 2 to 4 clauses, so the formula grows in step with the circuit. Without the wire names, one big formula for the circuit can grow exponentially. This method is the **Tseitin encoding**.

(`r0` and `r1` are not needed for the goals, and a real tool would drop their 6 clauses. I keep them so that the circuit is complete. Also, the real `bv_decide` first rewrites the circuit with only AND and NOT gates, an and-inverter graph, and simplifies it. So its clauses differ in detail, but the idea is the same.)

### CaDiCaL finds the bits; Lean prints them

The last step is a real run. I used CaDiCaL 2.1.2 from the Lean v4.35.0 toolchain, with the flags from Lean's `Lean/Meta/Tactic/BVDecide/External.lean`. The solver reads the 17 clauses as a DIMACS file. DIMACS is the standard input format for SAT solvers:

```text
p cnf 8 17      8 values, 17 clauses
4 1 0           c1 = ¬x0
-4 -1 0
-5 -2 0         c2 = ¬x1 ∧ c1
-5 4 0
2 -4 5 0
-6 1 0          r0 = x0
6 -1 0
-7 -2 4 0       r1 = ¬x1 ⊕ c1
-7 2 -4 0
7 2 4 0
7 -2 -4 0
-8 -3 5 0       r2 = ¬x2 ⊕ c2
-8 3 -5 0
8 3 5 0
8 -3 -5 0
3 0             goal x < 0
8 0             goal -x < 0
```

Each value has a number: 1 `x0`, 2 `x1`, 3 `x2`, 4 `c1`, 5 `c2`, 6 `r0`, 7 `r1`, 8 `r2`. A positive number is the value, a negative number is its NOT, and `0` ends a clause. The labels on the right are mine; they are not in the file.

Lean calls the solver like this. `--lrat` asks for a certificate if the answer is unsat. `--unsat` tunes the solver for proving, because that is the usual case.

```bash
cadical problem.cnf proof.lrat --lrat \
  --binary=… --quiet --shrink=0 --unsat
```

The output is:

```text
s SATISFIABLE
v -1 -2 3 4 5 -6 -7 8 0
```

"s SATISFIABLE" means that the solver found values that make all 17 clauses true. The `v` line lists them: + is 1, and − is 0.

```text
-1 → x0 = 0     4 → c1 = 1     -6 → r0 = 0
-2 → x1 = 0     5 → c2 = 1     -7 → r1 = 0
 3 → x2 = 1                     8 → r2 = 1
```

So `x = x2 x1 x0 = 100` = **−4**. These are the same values that we found by hand from the circuit. Lean turns the bits back into a number and prints the counterexample:

```text
error: The prover found a counterexample, …
x = 4#3
```

`4#3` is the 3-bit value `100`, which is −4 as a signed number. With `Int32`, the same steps give `x = -2147483648`.

If we add the hypothesis `x ≠ 4#3`, the solver answers "s UNSATISFIABLE" instead and writes an LRAT certificate. The certificate is a list of steps, and each step is a small consequence of earlier clauses. Lean does not trust CaDiCaL: a checker that is proved correct in Lean replays the certificate. I ran this too, and Lean accepts the proof.

### Verus finds the abs overflow with no spec

Verus (type 1b) finds the same bug in a different way. I wrote the `abs` from Part 1 in Verus exactly like the C code, with no `requires` and no `ensures` (`abs-naive/abs.rs`):

```rust
fn abs(x: i32) -> i32     // (1)
//  requires x != i32::MIN,
{
    if x < 0 {            // (2)
        -x                // (3)
    } else { x }
}
```

Verus still rejects this code. First, Verus turns the code into math:

1. `x: i32` becomes the fact **−2,147,483,648 ≤ x < 2,147,483,648**.
2. `if x < 0` becomes the fact **x < 0** inside the branch.
3. `-x` becomes the hidden goal **−x fits in an i32**. Verus adds such a goal to every addition, subtraction, negation and multiplication by itself. Nobody writes this goal.

In the math, integers are exact: they never wrap around.

Then Verus asks Z3 the opposite question: is there an `x` where facts 1 and 2 hold, but goal 3 fails? Z3 does not try values. Instead, it reasons:

- Goal 3 fails if −x is too small **or** too big.
- Fact 2 says x < 0, so −x > 0. Only "too big" is left: **x ≤ −2,147,483,648**.
- Fact 1 says **x ≥ −2,147,483,648**.
- Both meet at one value: **x = −2,147,483,648**. So the answer is **sat**: a bad input exists.

Verus knows which goal broke, so it points at the `-x` in the source. This is the real Verus output:

```text
error: possible arithmetic underflow/overflow
 --> abs.rs:7:16
  |
7 |     if x < 0 { -x } else { x }
  |                ^^
```

The commented-out line is the fix. With `requires x != i32::MIN`, Z3 gets one more fact, the only bad value is gone, and Verus reports "verified". But now every caller must prove that its argument is not `i32::MIN`.

This is the difference from Lean. In Lean, `Int32` negation wraps by definition, and Lean checks only the theorems that you write. With no theorem, Lean accepts the same code and returns −2147483648. Verus finds the bug with no spec, because it adds a "no overflow" goal to every arithmetic operation.

### Verification conditions + SMT: binary search in Verus

The next example is the binary search from my experiment, proved with Verus (`rust/verus/src/binary_search.rs`). Only two parts are not normal Rust: the spec and the loop invariant.

```rust
pub fn search(a: &[u32], x: u32) -> (r: Option<usize>)
  // ---- Spec: what search promises ----
  ensures
    // P1: a found index is right
    r matches Some(i) ==> i < a.len() && a@[i as int] == x,
    // P2: "not found" is right, when a is sorted
    sorted(a@) && r is None ==>
        forall|i: int| 0 <= i < a.len() ==> a@[i] != x,
{
  let mut lo: usize = 0;
  let mut hi: usize = a.len();
  while lo < hi
    // ---- Loop invariant: true at every pass ----
    invariant
      lo <= hi <= a.len(),
      sorted(a@) ==> forall|i: int| 0 <= i < lo ==> a@[i] < x,
      sorted(a@) ==> forall|i: int| hi <= i < a.len() ==> a@[i] > x,
    decreases hi - lo,
  {
    let mid = lo + (hi - lo) / 2;
    if a[mid] == x { return Some(mid); }
    else if a[mid] < x { lo = mid + 1; } else { hi = mid; }
  }
  None
}
```

The `ensures` clause is the spec, which is what `search` promises to its caller. P1: if it returns `Some(i)`, then `i` is a valid index and `a[i]` is `x`. P2: if it returns `None` and the array is sorted, then no element is `x`.

The loop invariant is a fact that is true each time a pass of the loop starts. Everything left of `lo` is less than `x`, and everything from `hi` on is greater than `x`. So if `x` is in the array, it can only be between `lo` and `hi`. The invariant is a proof hint. Verus cannot guess it, so the programmer (or an AI) writes it. Also, `decreases hi - lo` says that `hi - lo` gets smaller on every pass, so the loop stops.

Verus does not run the loop. Instead, it turns the code into questions:

- **Start**: is the invariant true for `lo = 0`, `hi = n`?
- **One pass**: if the invariant is true before a pass, is it true after the pass?
- **Exit**: when `lo = hi`, is `None` right?
- **Safety**: is `a[mid]` in bounds, is there no overflow, and does `hi - lo` shrink?

Z3 answers each question in the "find a counterexample" form. Here, it finds none: 17 questions, in about 10 ms.

### What Z3 gets: five facts and three goals

This section shows one of these questions exactly. It is the "One pass" question for one branch of the loop body: `a[mid] < x`, so `lo = mid + 1`. Z3 gets two lists: facts and goals.

A formula has no "assign". So the value of `lo` after the pass gets a new name, `lo′`.

**Facts** (true before the pass, or true because of the code). I write `n` for `a.len()`.

| | Fact | Source |
|---|---|---|
| F1 | `0 ≤ lo < hi ≤ n` | the first invariant line + the loop condition |
| F2 | `mid = lo + (hi − lo) / 2` | the `let` line |
| F3 | `a[mid] < x` | the test of this branch |
| F4 | `lo′ = mid + 1` | the new value of `lo` |
| F5 | `for every i ≤ j: a[i] ≤ a[j]` | `a` is sorted |

Verus also sends invariant lines 2 and 3 from before the pass. For this branch, Z3 does not need them, because the sorted fact alone covers every index up to `mid`. I checked this in Z3: without them, the answer is still unsat.

**Goals** (must be true after the pass):

| | Goal | Source |
|---|---|---|
| G1 | `lo′ ≤ hi` | invariant line 1 still holds |
| G2 | `for every i < lo′: a[i] < x` | invariant line 2 still holds |
| G3 | `hi − lo′ < hi − lo` | `decreases`: the loop shrinks |

Invariant line 3 needs no goal here, because this branch does not change `hi`.

**The question for Z3** is: can F1 to F5 all be true while G1, G2 or G3 is false? If Z3 answers **unsat**, no such values exist, so the goals always hold. Nothing in these lists is a number: `a`, `n`, `lo`, `hi` and `x` are unknown values. Thus one answer covers every array. Verus checks one pass, but for all inputs at once. Together with the Start check, this is induction: the invariant is true before the first pass, and each pass keeps it true, so it is true after any number of passes. Z3 4.16.0 gives unsat for these lists.

(This is simplified. The real query also holds the other two branches, the facts are in Z3's own input language, and the invariant lines have the "if `a` is sorted" guard in front. The meaning is the same.)

### Z3: a logic part and a math part

How does Z3 find such an answer? Z3 has two parts that take turns:

- **The logic part** (a SAT solver) sees each fact only as a switch: on or off. It finds a setting that makes every "and / or" rule true. It does not know what the facts mean.
- **The math part** takes the facts that are on and checks if real numbers can make them all true. If not, it adds a new rule that forbids this setting.
- **They repeat** until one of two endings. Either the math part finds numbers (**sat**: here is a solution), or no setting is left (**unsat**: no solution exists).

For example: are there whole numbers x, y ≥ 0 with

$$(x > 5 \text{ or } y > 5) \text{ and } x + y < 4 \;?$$

The switches are A = "x > 5", B = "y > 5" and C = "x + y < 4". The logic part sees only **(A or B) and C**. C is joined with "and", so C is always on.

| Round | Logic part: on | Math part checks | New rule |
|---|---|---|---|
| 1 | A, C | x > 5, but x + y < 4 ✗ | not (A and C) |
| 2 | B, C | y > 5, but x + y < 4 ✗ | not (B and C) |
| 3 | No setting is left → **unsat**: no such numbers | | |

If we change the last part to **x + y < 10**, then in round 1 the math part finds `x = 6, y = 0`, and the answer is **sat**. Both answers are real Z3 4.16.0 output.

### State-space search: explicit enumeration

The second family does not prove. Instead, it searches. The first type, explicit enumeration, visits the states one by one. The example is the lost-update counter from Part 1, as a TLA+ model (`tla/lost-update-counter/Counter.tla`). Each of the two threads runs:

```text
t := x       // read
x := t + 1   // write
```

The property is: when both threads are done, `x = 2`. The checker, TLC, does three things:

1. Start from the first state.
2. Make every next state. Skip states that it has seen before.
3. Check each state. Stop at the first failure.

{{< figure src="state-graph.svg" alt="State graph of the two-thread counter" caption="The state graph of the counter. \"write 1\" means that the thread will set `x` to 1." >}}

The red path is the counterexample: T1 reads 0, T2 reads 0, T1 writes 1, T2 writes 1, so `x = 1`.

### State-space search: symbolic search

The second type of search handles many states at once. Kani does this for Rust code (`rust/kani/src/binary_search.rs`, shortened):

```rust
#[kani::proof]
#[kani::unwind(10)]   // the bound
fn p1_found_is_right() {
  // any 8 values, any length 0..=8
  let arr: [u32; 8] = kani::any();
  let a = any_array(&arr);
  let x: u32 = kani::any();
  if let Some(i) = search(a, x) {
    assert!(i < a.len() && a[i] == x);
  }
}
```

Kani works in four steps:

1. **Unroll the loops.** Kani copies each loop body up to the bound. Then the code has no loops left.
2. **Turn numbers into bits.** A `u32` becomes 32 true/false values. Each `+`, `<`, … becomes a logic circuit, like the one in a CPU.
3. **Ask one question.** All of it becomes one big SAT formula: "Is there an input that makes the `assert` false?"
4. **Read the answer.** **sat** gives a concrete bad input. **unsat** means no bug, up to the bound. A bound that is too small gives a failure, not "verified".

The next sections show these steps one by one.

### Kani step 0: the loop

This is `search`, as written:

```rust
let mut lo = 0;
let mut hi = a.len();
while lo < hi {
    let mid = lo + (hi - lo) / 2;
    if a[mid] == x      { return Some(mid); }
    else if a[mid] < x  { lo = mid + 1; }
    else                { hi = mid; }
}
None
```

The number of loop passes depends on the input. But a SAT formula has a fixed size, so it cannot hold a loop. Kani's plan is to remove the loop first, then the variables, then the numbers. At the end, only true/false values stay.

To keep the code short, I unroll only 2 passes here. `unwind(N)` allows N − 1 passes, so 2 passes is `unwind(3)`. The harness above, with `unwind(10)`, unrolls 9 passes.

### Kani step 1: unroll the loop

```rust
let mut lo = 0;
let mut hi = a.len();
if lo < hi {                                // copy 1
    let mid = lo + (hi - lo) / 2;
    if a[mid] == x      { return Some(mid); }
    else if a[mid] < x  { lo = mid + 1; }
    else                { hi = mid; }
    if lo < hi {                            // copy 2
        let mid = lo + (hi - lo) / 2;
        if a[mid] == x      { return Some(mid); }
        else if a[mid] < x  { lo = mid + 1; }
        else                { hi = mid; }
        assert!(!(lo < hi));                // unwinding assertion
    }
}
None
```

Each `if` is one copy of the loop body. The code now runs from top to bottom, and it never jumps back.

The **unwinding assertion** says "the loop does not need a third pass". If an input can make it false, the bound is too small, and Kani reports a failure. An array of 8 elements can need 4 passes, so 2 passes are too few. Here, the failure is the correct answer.

### Kani step 2: name each value

A formula cannot change a variable. So each new value gets a new name: `lo0`, `lo1`, `lo2`. Also, each `if` becomes a choice: `c ? p : q` means "p if c is true, else q".

```text
lo0  = 0
hi0  = len
g1   = lo0 < hi0                          // does pass 1 run?
mid1 = lo0 + (hi0 - lo0) / 2
eq1  = a[mid1] == x
lt1  = a[mid1] <  x
lo1  = (g1 && !eq1 &&  lt1) ? mid1 + 1 : lo0
hi1  = (g1 && !eq1 && !lt1) ? mid1     : hi0
g2   = g1 && !eq1 && lo1 < hi1            // does pass 2 run?
mid2 = lo1 + (hi1 - lo1) / 2
eq2  = a[mid2] == x
lt2  = a[mid2] <  x
lo2  = (g2 && !eq2 &&  lt2) ? mid2 + 1 : lo1
hi2  = (g2 && !eq2 && !lt2) ? mid2     : hi1
result    = (g1 && eq1) ? Some(mid1) : (g2 && eq2) ? Some(mid2) : None
unwind_ok = !(g2 && !eq2 && lo2 < hi2)
```

`g1` and `g2` are guards. They say whether pass 1 and pass 2 run. They also replace the early `return`. The result is a list of equations with no branches and no jumps. (This form is called static single assignment.)

### Kani step 3: turn into bits

Each name becomes a row of true/false values: 64 for a `usize` and 32 for a `u32`. Each operation becomes gates (AND, OR, XOR, NOT), like the circuit in a CPU. Below are three equations as gates, with 4 bits to save space:

```text
// lo1 = mid1 + 1 → an adder. mid1 = m3 m2 m1 m0, sum = s3 s2 s1 s0
s0 = !m0
s1 = m1 XOR m0
s2 = m2 XOR (m1 AND m0)
s3 = m3 XOR (m2 AND m1 AND m0)
overflow  = m3 AND m2 AND m1 AND m0          // must be false
// g1 = lo0 < hi0 → a comparator, from the top bit down
g1 =    (!l3 AND h3)
     OR (l3 = h3) AND (!l2 AND h2)
     OR (l3 = h3) AND (l2 = h2) AND (!l1 AND h1)
     OR (l3 = h3) AND (l2 = h2) AND (l1 = h1) AND (!l0 AND h0)
// a[mid1] → a selector, for each of the 32 bits
a[mid1]   = (mid1 = 0) ? arr0 : (mid1 = 1) ? arr1 : … : arr7
in_bounds = mid1 < len                       // must be true
```

Two checks come for free. Kani adds "no overflow" for each `+` and "index in bounds" for each `a[…]`, with no `assert` from us.

### Kani step 4: ask one question

Everything is joined into one formula for the SAT solver:

```text
    all the gates from step 3
AND NOT (     the assert in the harness
          AND unwind_ok
          AND in_bounds
          AND no overflow )
```

In words: "Is there an input that follows the code and breaks a check?" Only true/false values and gates are left. The solver looks for values that make the whole formula true. If the answer is **sat**, the solver gives the bits of `arr`, `len` and `x`, and Kani prints them as a counterexample. If the answer is **unsat**, no input breaks a check, up to the bound.

I ran Kani on `p1_found_is_right` with three bounds and read the size that the solver reports (Kani 0.68.0 with CBMC 6.11.0, after its own simplification):

| Loop passes | Attribute | True/false values | Clauses |
|---|---|---:|---:|
| 2 (the example above) | `unwind(3)` | 10,059 | 31,074 |
| 3 | `unwind(4)` | 12,011 | 39,626 |
| 9 (the real harness) | `unwind(10)` | 23,723 | 91,673 |

Only 352 of the values are inputs: 256 for `arr`, 64 for `len` and 32 for `x`. The others are outputs of gates. The size grows in a straight line with the number of passes: each added pass costs about 1,950 values and 8,550 clauses.

(This is simplified. Kani first compiles Rust to an intermediate form, and CBMC does these steps on that form. The real circuits use 64 and 32 bits, and the formula is changed to CNF before the solver gets it.)

### Abstract interpretation

The third family runs fully automatically. Look at this code:

```text
if c { x = 3 } else { x = 7 }
y = 10 / x         // safe?
z = 10 / (x - 5)   // safe?
```

The tool does not track the exact set of values. It tracks a range that surely holds every real value:

| | Real values | Tracked range | Result |
|---|---|---|---|
| `x` | {3, 7} | [3, 7] | |
| `10 / x` | x is never 0 | no 0 in [3, 7] | ✓ proved safe |
| `10 / (x−5)` | x − 5 ∈ {−2, 2} | 0 is in [−2, 2] | ⚠ false alarm |

- The tool computes a **bigger set** that surely holds every real value.
- The method is **sound**: no error in the bigger set means no error in any real run.
- **The price**: the bigger set has values that never happen, so some warnings are false alarms.
- **You provide**: usually nothing. It runs fully automatically, also on large code.

Tools of this type are SPARK flow analysis (Ada), and Astrée and Frama-C (C).

### Three families, three trade-offs

| Type | Covers | You write | You trust | Trade-off | Example |
|---|---|---|---|---|---|
| **1a** Kernel-checked proof | All inputs, all sizes | The proof (or an AI does) | A small proof kernel | Full coverage, but you must write the proof | Lean |
| **1b** VCs + SMT | All inputs, all sizes | Annotations, loop invariants | The VC generator and the SMT solver | Full coverage, but you must write the invariants | Verus |
| **2a** Explicit enumeration | Only the finite model | A model + its size | The checker, and that the model matches the code | Little work, but only a small model | TLA+ / TLC |
| **2b** Symbolic search | Up to the bound | A bound (+ a model or code) | The translation to SAT, and the SAT solver | Little work, but only up to a bound | Kani |
| **3** Abstract interpretation | All runs, approximated | Usually nothing | The analyzer | No work, but false alarms | — |

Many real projects use more than one method: a search finds bugs fast, and then a proof covers the parts that must hold for every size.

## 4. My experiment: one spec, five tools

### Setup

I wanted to see how these tools compare when an AI agent does the work. So I gave the same specs to five tools. The code, specs and numbers are in `formal-verification/experiments/comparison/`. This evidence is empirical and comes from small algorithms. For each algorithm, the work had four steps:

1. **Human**: an English spec, for example "If search returns *i*, then *i* is a valid index and *a*[*i*] = *x*."
2. **Agent**: the code, in each language.
3. **Agent**: the proof, with one formal statement per English sentence.
4. **Agent**: mutants, which are copies of the code with one planted bug. Each mutant must fail.

There were 6 algorithms: max, binary search, insertion sort, merge sort, ring buffer and a lost-update counter. An AI agent (Claude) did the work from 2026-10-24 to 10-27, on one machine. These are the five tools:

| Tool | Kind |
|---|---|
| **Lean 4** | theorem prover: code + proofs in one language |
| **Bend 2** | type checker = proof checker |
| **Rust + Verus** | SMT-based verifier: `ensures` + invariants |
| **Rust + Kani** | bounded model checker: all inputs up to a size |
| **TLA+ / TLC** | explicit-state model checker (counter only) |

### Spec and proof size

The table gives non-blank, non-comment lines as "spec + proof". For Verus, the second number is hints, not a proof. The spec has a different form in each tool: theorems in Lean, laws in Bend, `ensures` in Verus and harnesses in Kani. The data is in `results/metrics.csv`.

| Algorithm | Lean | Bend | Verus | Kani |
|---|---:|---:|---:|---:|
| max | 3 + 35 | 40 + 129 | 4 + 7 | 31 + 0 |
| binary search | 6 + 27 | 50 + 416 | 6 + 5 | 37 + 0 |
| insertion sort | 2 + 36 | 43 + 110 | 6 + 29 | 28 + 0 |
| merge sort | 2 + 47 | 43 + 210 | 12 + 38 | out of memory |
| ring buffer | 18 + 69 | 90 + 505 | 44 + 7 | out of memory |

Why are the sizes so different? Lean's tactics (`omega`, `simp`) close most steps automatically. Verus needs only hints (invariants, `decreases`, `assert`), because its SMT solver finds the proof. Bend has no tactics and no arithmetic library yet, so every index fact is proved by hand.

Kani has **0 lines of proof or hints**, because it does not prove. Instead, it checks every input up to a size. Its cost is memory: Kani ran out of memory on merge sort and on the ring buffer.

## 5. Limits: what a proof does not tell you

### A proof is only as good as its spec

A formal proof has three steps ([Runtime Verification, "The proof is in the spec"](https://runtimeverification.com/blog/the-proof-is-in-the-spec)):

1. **The idea**, in normal words. For example: "this program always sorts the list."
2. **The formal version (the spec)**. A human or an LLM translates the idea to a strict computer language (Lean, Verus, TLA+). **The mistakes happen here.**
3. **The checker**. A program checks the formal version and says "proved" or "not proved". It checks only step 2, not step 1.

The checker is very reliable, but it sees only the spec. It cannot know what you meant. So the weak point is the translation from step 1 to step 2, and today an LLM often does that translation. If the translation is wrong, the checker can say **"proved"** about the **wrong thing**.

### Example: a wrong calculation came out as "proved"

[FaithformBench (Cornish et al., 2026)](https://arxiv.org/abs/2608.10916) tests whether a translator keeps a mistake visible. The authors take a correct reasoning step and make it wrong on purpose. A faithful translator must then give a false Lean statement, which Lean cannot prove. Figure 1 of the paper has this example with Kimina, an open model that translates math into Lean:

```lean
-- input step, wrong on purpose: "3^x = 5, so 3^(x+2) = 51"   (the right value is 45)
-- faithful: x is real, so the claim is false and Lean cannot prove it
theorem step (x : ℝ) (h : (3 : ℝ) ^ x = 5) : (3 : ℝ) ^ (x + 2) = 51
-- what the LLM (Kimina) wrote: x is a natural number, so h is impossible
theorem step (x : ℕ) (h : 3 ^ x = 5) : 3 ^ (x + 2) = 51   -- Lean proves it
```

The input step is wrong, because $3^{x+2} = 9 \cdot 3^x = 45$, not 51. Kimina changed `x` from a real number to a natural number. No natural number gives $3^x = 5$, so the hypothesis `h` can never hold, and anything follows from it. Thus Lean proves the wrong step. The checker says "proved", but the wrong step is still wrong. (For the correct input, 45, Kimina kept `x` real.)

The paper calls this **silent correction**. The authors found it in all the fine-tuned formalizers that they tested (Goedel, Herald, Kimina and StepFun). It was worst in the ones that were best at formalizing correct statements: Goedel was best on correct steps and corrected wrong steps most often. General LLMs (Claude Opus 4.7, GPT 5.2, Gemini 3.1 Pro, Qwen Plus) did it less.

So AI makes proofs cheap, and the checker makes sure that the proof is valid. But a person still has to read the spec and confirm that it says what they mean.
