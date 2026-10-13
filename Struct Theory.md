# Struct Theory v0.1

## 0. Conventions

- **Names** are strings used as labels for addressing. They are not part of a struct's value, and equality ignores them (A2). `a ++ b` is concatenation.
- **Positions** are 1, 2, 3, … for finite structs. For infinite structs, positions come from a well-ordered index, so a struct's fields always form a well-ordered sequence. 
- A struct is written `⟨n₁: t₁, n₂: t₂, …⟩`. `Field(s, i, n, t)` means "s has, at position i, a field named n with value t."
- **Structs are values, not references.** Storing a struct somewhere stores a copy. This is why Foundation can never be broken by an operation.
- **Truth:** `0` is false, and anything else is true. The canonical true value is `1`.
- **Numerals:** `0 = ⟨⟩`, and `S(n) = ⟨p: n⟩`.

## 1. Primitives (undefined terms)

- **P1.** Struct
- **P2.** Name
- **P3.** `Field(s, i, n, t)`, as above
- **P4.** `Op(s)`: a distinguished kind of struct, the operation structs (section 5)
- **P5.** String (used only by Prose)

## 2. Axioms

**A1. Construction.** For every well-ordered index I, every one-to-one name assignment `n: I → Name`, and every value assignment `t: I → Struct`, there exists a struct s with `Field(s, i, n(i), t(i))` for every i in I and no other fields. I may be empty or infinite.

**A2. Extensionality and Precise Equality.** Two structs are *identical* (the same object) if and only if they have the same fields: for every position i, name n and struct v, `Field(s,i,n,v) ⇔ Field(t,i,n,v)`. Two structs are *precisely equal*, written `s = t`, if and only if they have the same number of fields and, at every position i, the values `val(s,i)` and `val(t,i)` are precisely equal. Names are not compared, and order is. The definition is recursive, which A3 makes legitimate. Identical structs are always equal, but not the reverse: `⟨a: 0, b: 0⟩ = ⟨b: 0, a: 0⟩`, while `⟨a: 0, b: ⟨c: 0⟩⟩ ≠ ⟨a: ⟨c: 0⟩, b: 0⟩`.

**A3. Foundation (Containment).** There is no infinite sequence `s₀, s₁, s₂, …` in which each `s_{k+1}` is a field value of `s_k`. This forbids self-containment at any depth, and forbids infinitely deep structs.

**A4. Infinity.** There exists a struct ω whose k-th field (k = 0, 1, 2, …, at position k+1) is named by the decimal string of k and has the numeral k as its value.

**A5. Naturality.** A struct has a whole-number cardinality if and only if it is finite (definitions in section 3). For a finite struct, that cardinality is a numeral. An infinite struct has no whole-number cardinality.

**A6. Coordination.** Every occurrence of a struct inside another has a coordinate: a finite tuple of positions giving the route from the root to it. `resolve(s, ()) = s`, and `resolve(s, (i, rest)) = resolve(v, rest)`, where v is the value at position i of s.

**A7. Scope.** A problem is a struct **S\[u\]** (written U), the *scope root*. Every struct under consideration is an occurrence inside U. The root is fixed for the whole problem. Only the contents of U may change, and only by the write operations in section 5 (Assign, Del).

## 3. Definitions (these assert nothing)

- **Zero:** `0 = ⟨⟩`. It exists by A1 with an empty index and is unique by A2. Every struct with no fields is 0.
- **Value of a field:** `val(s, i)` is the value at position i. `name(s, i)` is the name there.
- **Occurrence and dom:** `dom(s)` is the set of coordinates that resolve inside s, including `()`.
- **Level (depth of an occurrence):** a coordinate γ has depth `|γ| − 1`. So `()` is the root `S[u]`, depth 0 is the direct fields of U (coordinates of length 1), and so on. `S[n]` is the set of occurrences at depth n.
- **Height of a struct:** `h(0) = 0`, and `h(s) = 1 + sup of h(values of s)`. Heights are ordinals, well defined by A3. **Depth Equality** means equal height. 
- **Finite:** s is finite if its direct fields match one-to-one with the numbers below some numeral, and every field value is finite (recursion by A3). Otherwise infinite. Numbers come from structs, so this is not circular.
- **Cardinality:** for finite s, `c(0) = 0` and `c(s) = sum over fields of (1 + c(value))`. It counts fields at all depths, and the numeral n has `c = n`.  for infinite structs, cardinality is not defined here.
- **Layer n of s:** `Lₙ(s)` is the set of all coordinates γ in s with `|γ| = n + 1`. Names play no part, so it records only the shape of level n.
- **Same-size / same-height / same-layer:** equal cardinality / equal height / equal `Lₙ`. These are weaker than `=`, and I suggest not calling them "equal" in prose.
- **Last field, `S[-1]`:** the field with the greatest position, if one exists. It is undefined for infinite structs with no last position.

## 4. The machine

The state is `σ = (U, pc, skip, status)`:

- **U**: the scope root
- **pc**: a coordinate in U, the next operation to run
- **skip**: true or false, set by `If`
- **status**: running, halted, or failed

**Operands.** An operation struct has the form `⟨op: Name, 1: r₁, 2: r₂, …⟩`. Each operand r is a reference, one of:

- `Lit(s)`: the struct s itself
- `Var(n)`: the value of the depth-0 field of U named n (always unique by A1). It fails if there is none.
- `Coord(γ)`: the value at coordinate γ in U. It fails if γ is not in dom(U).

`Val(r)` is the struct a reference denotes. 

**Failure.** An operation that fails sets `status := failed` and changes nothing else.

**Writing.** `Assign(d, v)`: if U has a depth-0 field named d, replace its value with v (position unchanged). Otherwise append the field `d: v` after the last field. Coordinates of existing fields never move. This is the only way values enter U besides Del. 

**Execution order.** For a struct s, let P be the positions whose value is an operation struct, and N the other positions, each in order. Then:

```
E(s) = [ (i) for i in P ]  ++  concat over i in N of [ (i)·γ for γ in E(val(s,i)) ]
```

So a struct's own operations come first, then each remaining field's subtree in turn. This is a depth-first traversal that never descends into an operation's operands. `next(γ)` is the successor of γ in `E(U)`, or none.

**Entering a target.** `target(γ)` is the first coordinate in `E(U)` that equals γ or extends it. If there is none, the machine halts.

## 5. Operations

The result of every value-producing operation is written with `Assign` to the named depth-0 variable after `→`.

**Data**

- `Assign<d, v>`: `Assign(d, Val(v))`.
- `Del<γ>`: replace the value at γ by 0 (field stays, so coordinates never shift). Fails if γ is empty or not in dom(U).
- `Wrap<x, n> → d`: `d := ⟨n: Val(x)⟩`. With `n = p` this is successor. Any other name gives an equal result, since names are ignored.
- `Combine<x, y> → d`: the fields of x in order, then the fields of y in order, where each field name of y is renamed by appending `_1` repeatedly until it matches no name already in the result. Not commutative.
- `Break<s, γ> → d₁, d₂`: `d₁` is the value at γ in `Val(s)`, and `d₂` is `Val(s)` with that value replaced by 0. Fails unless γ is nonempty and in dom(Val(s)). 
- `Elim<x, y> → d`: walk `A = Val(x)` from the root. At each coordinate γ that is also in `B = Val(y)`: if the values at γ are equal, replace A's value there with 0 and do not look inside it, otherwise continue into the children. Coordinates are stable.

**Comparison** (all return `1` or `0`)

- `PEqual<x, y> → d`: 1 if `Val(x) = Val(y)` by A2.
- `DPEqual<x, y, n> → d`: 1 if `Lₙ(Val(x)) = Lₙ(Val(y))`. This compares the shape of the structs at level n (which positions exist). It ignores names, and everything above and below level n. 
- `CEqual<x, y> → d`: 1 if both are finite with equal cardinality, 0 if both finite and unequal. Fails if either is infinite. 
- `DEqual<x, y> → d`: 1 if the heights are equal.

**Control**

- `If<n>`: the next operation runs only if `Val(n) ≠ 0`. Formally, if `Val(n) = 0` then `skip := true`.
- `Jump<x>`: count the fields named x anywhere in U. If the count is not exactly 1, fail. Otherwise `pc := target(γ)` for that field's coordinate.
- `LJump<x>`: as Jump, but only among occurrences strictly inside the struct enclosing the current operation (coordinate `pc` with its last component removed). Since siblings have distinct names by A1, "local" must mean the whole subtree.
- `Goto<γ>`: `pc := target(γ)` if γ is in dom(U), else fail. Coordinates are stable under Assign and Del, but Combine can shift them if you store results over code.
- `Halt`: `status := halted`.

## 6. Execution axiom

**A8. Execution.** The initial state has `pc` the first element of `E(U)` (or halted if empty), `skip = false`, `status = running`. While running, with γ = pc and o the operation at γ:

1. If `skip`: set `skip := false`, `pc := next(γ)`. Operation o does not run.
2. Otherwise run o as defined in section 5. Unless it changed pc itself (a jump) or the status, set `pc := next(γ)`. If next is none, set `status := halted`.

Execution is deterministic: each state has exactly one successor. Code is data: Assign can store an operation struct into U, so programs can modify themselves.

## 7. Prose (optional extension, see section 9)

**A9. Prose.** A *prose operation* is a struct `⟨says: String, grounding: G⟩` where G is either a **reduction** (a program of core operations or other valid prose operations) or **PST** (problem solving text) (a struct in the grammar below). It is valid only if its grounding is valid, and the grounding may not refer to the operation it grounds, directly or through other prose operations (checked by A3). Running a prose operation means running its grounding. A prose description with no grounding is a comment and has no semantics.

**PST grammar (v0).** `⟨cond: ref, effect: program, mode: once-per-run | none⟩`. Validity is a purely syntactic check, and each connective translates to core operations by a fixed rule:

- **`then`** (sequence): concatenation of programs in order.
- **`Condition:`** `T(cond: c, effect: E)`:

  ```
  PEqual<c, 0> → t ; If<t> ; Jump<L_end> ; E ; L_end
  ```

  This skips E when c is false.
- **`once per run`**: a variable flag f: guard E with `PEqual<f, 0>`, and set `Assign<f, 1>` at the end.

Because every PST and reduction expands to core operations, Prose adds convenience but no power. This is the conservative-extension theorem (section 8).

(PST is relatively open so long as it "compiles" to core operations without any ambiguities.)

## 8. Theorems to prove

- **T1.** Every occurrence has exactly one coordinate, and every coordinate in dom(s) names exactly one occurrence.
- **T2.** Equal structs have equal height, cardinality (if finite), and layers. Identical structs are always equal, but equal structs can have different field names.
- **T3.** A struct is finite iff it has a numeral as cardinality, and numeral n has cardinality n.
- **T4.** A program with no jump operations and no self-modification terminates. (With self-modification it can fail to, since Assign can append new code at the end.)
- **T5.** Prose is conservative: every program using valid prose operations has an equivalent core program, by induction on grounding depth.
- **T6 (target).** Turing completeness. A Minsky counter machine can be simulated with `Wrap` (increment), `Break` (decrement), `PEqual` (zero test), `If`, `Jump`, `Assign` and `Halt`.
- **T7 (target).** An interpretation of hereditarily finite set theory, with membership defined as "is a field value of."

## 9. Variants and extensions

Struct Theory is a family of systems. A **variant** swaps some axioms for others. An **extension** adds axioms on top of a base. Each entry says what differs from the Core (A1 to A8 in this document, without Prose). Names like "Finite Struct Theory" are working labels. Only the Core and Original Struct Theory are written out in full here, and the rest are sketches.

### 9.1 Original Struct Theory (OST)

- **OST1. Structure.** A struct is an ordered group of named fields written in angle brackets, `⟨n₁: t₁, n₂: t₂, …⟩`. Every field value is a struct.
- **OST2. Containment.** A struct contains only structs, and no struct contains itself at any depth.
- **OST3. Fieldship.** A struct may have any number of fields: none, finitely many, or infinitely many.
- **OST4. Zero.** There exists a struct `0` with no fields. Every struct with no fields is `0`.
- **OST5. Precise Equality.** Two structs are precisely equal if and only if they have the same number of fields and, at every position, the values are precisely equal. Names are ignored ("value-wise"), and order counts.
- **OST6. Cardinality.** The cardinality of a struct is the number of fields it has, counted at all depths.
- **OST7. Naturality.** A finite struct's cardinality is a whole number. An infinite struct has no whole-number cardinality. (A struct is infinite if you can never finish counting its fields.)
- **OST8. Coordination.** Every occurrence of a value inside a struct has a location, given as a tuple of positions from the outside in. The occurrence `y` in your example has the coordinate `(3, 1)`, and `i` has `(1)`.
- **OST9. Depth Precise Equality.** Two structs are precisely equal at depth n if and only if they have the same positions at depth n. This says nothing about any other depth.
- **OST10. Depth Equality.** Two structs are depth-equal if and only if they have the same height (the greatest nesting depth).
- **OST11. Size Equality.** Two structs are size-equal if and only if they have the same cardinality.
- **OST12. Operation.** There exist structs designed as operations, and they may be contained in other structs.
- **OST13. Execution.** To run a struct, first execute the operation fields of that struct in order. Then run each remaining field's value in order, each by this same rule, continuing with the next field once a value's run has finished. Your `S[-1]` (reaching the last field) and "travel to the nearest fork" describe this return to the next sibling.
- **OST-P. Prose (optional).** An operation may be defined by a specialized struct together with a prose description. In this form it is not safe: see "Unrestricted Prose" below.

### 9.2 Core Struct Theory

A1 to A8 in this document. 

### 9.3 Extensions

- **Core + Prose.** Section 7 (A9) and T5. It is optional in the way Choice is optional for ZF. Unlike Choice it adds no power: T5 says every prose operation reduces to core operations, so dropping it costs only convenience.
- **Unrestricted Prose (a bad variant, kept for reference).** The original wording, where any sufficiently detailed prose counts as an operation. It admits self-defeating descriptions (Berry's paradox), so the system is inconsistent and proves everything. It exists here to show why the grounding requirement is in A9.

### 9.4 Variants

- **Finite Struct Theory.** Drop Infinity and allow only finite index sets in Construction. Naturality then holds for every struct and stops being a separate axiom, and Foundation follows from finiteness. The universe is the hereditarily finite structs. T6 (Turing completeness) should still hold, since structs can grow without bound, and T7 is the natural target.
- **Name-Sensitive Struct Theory.** Equality counts names, so `⟨a: 0⟩ ≠ ⟨b: 0⟩`. Precise equality and identity coincide, and numerals depend on the field name chosen for the successor.
- **Bag Struct Theory.** Equality ignores both names and order, so a struct is a multiset of its values. This is the closest thing to ordinary set theory with duplicates allowed. It loses position-based coordinates and the order-based execution rule, so it would need a different way to run programs. Not worked out.

### Notes
- This is written by Claude. I provided the ruleset and Claude formalized it for the purposes of this document.
- PST rules are not very formalized yet.
