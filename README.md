# Compiler Design

GATE CSE · full syllabus · diagrams, rules, formulas, traps

[Phases](#ph)[Lexical](#lex)[Parsing](#par)[SDT](#sdt)[Intermediate](#ic)[Runtime](#rt)[Optimization](#opt)[Codegen](#cg)[Rapid-fire](#q)

## 1. Phases of a compiler


- Front end: lexical, syntax, semantic, ICG. Back end: optimization (machine-dep. part), codegen.
  
- **Pass** = one read of the whole program. **Phase** = logical step. Many phases can fit in one pass.
- Lexical errors: illegal char. Syntax errors: missing token/bracket. Semantic: type mismatch, undeclared var, scope.
- Cross-compiler: host ≠ target. Bootstrapping uses T-diagrams.

## 2. Lexical analysis

- **Token** = (type, attribute). **Lexeme** = actual string. **Pattern** = rule (regex).
- Implemented as DFA from regex: regex → NFA (Thompson) → DFA (subset construction) → minimize.
- **Maximal munch** (longest match); tie → rule listed first (keyword before identifier).
- Lexer strips whitespace and comments, tracks line numbers, builds symbol table entries.
- Lexer cannot detect `if(` unbalanced or type errors; cannot count nesting (needs a PDA).
- Counting tokens: `printf("i=%d",&i);` → `printf ( "i=%d" , & i ) ;` = 8 tokens (string literal is 1). Longest match: `x+++++y` → `x ++ ++ + y`.

**Trap:** Thompson NFA for regex of length n has ≤ 2n states. Subset construction can blow to 2n DFA states.


## 3. Parsing

### Grammar basics

- CFG = (V, T, P, S). Ambiguous: more than one parse tree / leftmost derivation. Ambiguity is undecidable. Fix with precedence + associativity layering.
- Left recursion kills top-down: `A→Aα|β` becomes `A→βA'`, `A'→αA'|ε`. Left factoring: `A→αβ1|αβ2` becomes `A→αA'`, `A'→β1|β2`.
- Top-down = leftmost derivation. Bottom-up = reverse of rightmost derivation.


### Top-down: FIRST, FOLLOW, LL(1)

**FIRST(X)**: terminal → itself. `X→ε` adds ε. `X→Y1Y2…`: add FIRST(Y1)−ε; if Y1 nullable add FIRST(Y2)… ; ε only if all nullable.\
**FOLLOW(A)**: FOLLOW(S) ∋ $. For `B→αAβ`: add FIRST(β)−ε. If β nullable or empty: add FOLLOW(B). FOLLOW never contains ε.\
**LL(1) table**: for `A→α`: M\[A,a\] for a∈FIRST(α); if ε∈FIRST(α), M\[A,b\] for b∈FOLLOW(A).

**Grammar is LL(1) iff** for each `A→α|β`: FIRST(α)∩FIRST(β)=∅; at most one of α,β nullable; if β nullable then FIRST(α)∩FOLLOW(A)=∅. Equivalent: no multiply-defined table entry. LL(1) grammars are never ambiguous or left-recursive.

- Recursive descent = LL(k) hand-coded. Predictive parser = table-driven with explicit stack, no backtracking.

### Bottom-up hierarchy

Placement of LL(1): inside LR(1) but incomparable with SLR/LALR (LL(1) and LR(0) overlap partially). Power: LR(0) \< SLR \< LALR \< CLR. State count: LR(0) = SLR = LALR ≤ CLR.

### LR parser construction

| Parser | Item | Reduce on | Conflicts |
| --- | --- | --- | --- |
| LR(0) | A→α·β | every terminal (whole row) | SR + RR, most |
| SLR(1) | LR(0) items | FOLLOW(A) | fewer |
| CLR(1) | \[A→α·β, a\] lookahead | lookahead set a only | fewest, largest table |
| LALR(1) | merge CLR states with same core | merged lookaheads | never new SR; may add RR |

- Augment grammar with `S'→S`. Closure + goto build the DFA of item sets. ACTION table (terminals): shift/reduce/accept; GOTO table (non-terminals).
- Conflicts: **SR** (shift vs reduce), **RR** (two reductions). Shift/reduce resolved by shifting (yacc) and precedence. RR resolved by earlier rule.
- LR(0): item set with a complete item and any other item is a conflict. Complete-item-only state is safe.
- Handle = substring matching RHS whose reduction is a step of reverse rightmost derivation. Viable prefix = prefix of right-sentential form that does not extend past handle.
- Operator precedence parser: no ε-productions and no two adjacent non-terminals on RHS. Not every unambiguous grammar is LR; every LR(k) is unambiguous.

**Traps:** (1) CLR and LALR have the same ACTION entries for shift; LALR merging cannot create SR conflicts. (2) Number of LALR states = number of LR(0) states. (3) A grammar with ε and left recursion cannot be LL(1). (4) Every regular grammar is LR(1) only if unambiguous.

## 4. Syntax-directed translation

|  | Synthesized | Inherited |
| --- | --- | --- |
| Value from | children (and self) | parent / left siblings |
| Evaluate | bottom-up, fits LR | top-down / mixed |

- **S-attributed**: only synthesized. Evaluated during LR reduction, in post-order.
- **L-attributed**: each inherited attribute of Xi depends on parent's inherited and attributes of X1…Xi-1 only (left to right). Includes S-attributed. Works with LL parsing / DFS.
- Dependency graph must be acyclic. Semantic action placed mid-RHS = SDT scheme. Marker non-terminals (ε) let LR do mid-actions.
- Type checking, type coercion, symbol table scope handling all happen in semantic analysis.

## 5. Intermediate code

Three-address code: `x = y op z`. Example `a = b*-c + b*-c`:

| Quadruple (op,a1,a2,res) | Triple (op,a1,a2) | Indirect triple |
| --- | --- | --- |
| t1=uminus c t2=b\*t1 … | (0) uminus c (1) * b (0) | list of pointers to triples, so reorder is cheap |

- Quadruples: easy to reorder (optimization), more space. Triples: refer by position, reordering hard. Indirect triples fix that.
- Count temps/instructions: n-operator expression ⇒ n TAC statements (one per operator). DAG removes common subexpressions, so fewer nodes. Syntax tree has no sharing.
- Backpatching: fill jump targets later for boolean/control flow in one pass (truelist, falselist, `nextlist`).
- Postfix, syntax tree, DAG, TAC, control flow graph are all IR forms. Arrays: `A[i]` address = base + (i − low)·w (row-major 2D: base + ((i−l1)·n2 + (j−l2))·w).

## 6. Runtime environment

- Activation record per call; stack grows with calls. Static scope = access link chain; dynamic scope = control link.
- Parameter passing: **call by value** (copy), **reference** (address), **copy-restore** (value-result), **by name** (textual substitution, re-evaluated each use). Swap(i, a\[i\]) shows differences.
- Storage: static (compile-time size), stack (recursion), heap (dynamic). Garbage collection/ dangling references live in heap.
- Display array: access non-local in O(1); access link chain: O(depth).

## 7. Code optimization

### Basic blocks and CFG

- **Leaders**: (1) first instruction, (2) target of any jump, (3) instruction right after a jump. Block = leader up to the next leader (exclusive). CFG: nodes = blocks, edges = possible flow.
- Number of blocks = number of leaders. Loops: natural loop via back edge (edge to a dominator). Dominators: d dom n if every path from entry to n passes through d.

### Machine-independent techniques

| Technique | Idea |
| --- | --- |
| Constant folding / propagation | `2*3.14` computed at compile time; replace var by known constant |
| Common subexpression elim. | reuse previously computed value (DAG locally, available-expr globally) |
| Copy propagation | after `x=y` use y in place of x |
| Dead code elimination | remove statements whose result is never live |
| Strength reduction | `x*2 → x<<1`, `i*4` in loop → add 4 |
| Loop-invariant code motion | hoist computation out of loop |
| Induction variable elim. | keep one induction var, derive others |
| Loop unrolling / jamming, inlining | cut overhead, bigger code |
| Peephole | small window of instrs: redundant load/store, jumps over jumps, algebraic simplification |

### Data-flow analysis

| Problem | Direction | Meet | Equation |
| --- | --- | --- | --- |
| Reaching definitions | forward | ∪ (may) | OUT = GEN ∪ (IN − KILL) |
| Available expressions | forward | ∩ (must) | OUT = GEN ∪ (IN − KILL) |
| Live variables | backward | ∪ (may) | IN = USE ∪ (OUT − DEF) |
| Very busy expressions | backward | ∩ (must) | IN = GEN ∪ (OUT − KILL) |

- Iterate to a fixed point. Initialize: ∪-problems start with ∅, ∩-problems start with universal set (except entry/exit).


## 8. Code generation


- Issues: instruction selection, register allocation, evaluation order, addressing modes.
- **Register allocation by graph colouring**: interference graph, k registers ⇒ k-colourable; if not, spill. Minimum registers = chromatic number of interference graph (live ranges overlapping).
- Sethi–Ullman for expression trees: label leaf (left) = 1, right leaf = 0; node with children l1, l2: if l1≠l2 then max(l1,l2), else l1+1. Result = minimum registers with no spill.
- Next-use info computed by backward scan of a basic block. Optimal code generation for a DAG is NP-complete.



## Rapid-fire GATE checklist



- Lexical = regular (DFA). Syntax = CFG (PDA). Semantics (declare-before-use, type match) = beyond CFG (needs context-sensitive checks).
- Top-down uses FIRST/FOLLOW; LR(0) uses items only; SLR uses FOLLOW; CLR uses lookahead.
- Counting questions: states in LR(0) DFA, number of tokens, number of basic blocks, number of TAC statements, number of ε entries in table.
- Ambiguity, left recursion, left factor: LL(1) fails. Ambiguous grammar is never LR.
- Reduce only the handle. Stack content + remaining input = right sentential form.
- Inherited attributes: LL friendly. Synthesized: LR friendly.



**Revision order:** parsing tables (FIRST/FOLLOW, LR items) → data-flow → optimization → SDT → runtime. Parsing alone gives \~60% of the marks in this subject.
