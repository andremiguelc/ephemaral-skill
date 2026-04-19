---
name: ephemaral
description: Verify that functions preserve business rules using formal verification. Use this skill whenever the user mentions invariants, verification, business rules, .aral files, writing constraints, or wants to check that a function can never produce invalid output. Also trigger when the user asks to "verify", "prove", or "check" a function against rules.
license: MIT
compatibility: Requires ephemaral binary (Lean 4) and Z3 solver
metadata:
  author: andremiguelc
  version: "2026-04-08"
  repository: "https://github.com/andremiguelc/ephemaral-skills"
---

# ephemaral — Formal Verification for Business Rules

ephemaral checks whether a function can ever produce output that violates business rules. It does this exhaustively — not by testing with sample inputs, but by mathematically proving the function is safe for all possible inputs. When it finds a violation, it shows you the exact values that cause it.

The tool takes two inputs:
- A function representation (`.aral-fn.json`) — what the function computes. Use the `ephemaral-parser` skill to produce this.
- One or more invariant files (`.aral`) — what must always be true about the data.

The invariants are the primary artifact. Functions come and go; invariants persist.

## The .ephemaral directory

All verification artifacts live in a `.ephemaral/` directory at the root of the user's project:

```
.ephemaral/
├── bin/
│   └── ephemaral               # binary (gitignored)
├── parsed/                     # .aral-fn.json files (gitignored, regenerable)
└── rules/                      # .aral invariant files (checked in, one per type)
    └── <type>.aral
```

- **bin/** — the ephemaral binary. Gitignore this.
- **parsed/** — `.aral-fn.json` files produced by parsers. Gitignore this (regenerable from source).
- **rules/** — `.aral` invariant files. Check these in — they're the business rules.

Create it if it doesn't exist: `mkdir -p .ephemaral/{bin,parsed,rules}`

## Setup

Before running verification, locate the ephemaral binary. Check in this order:

1. **Project-local**: `.ephemaral/bin/ephemaral` — preferred location
2. **System PATH**: `which ephemaral`
3. **Contributor build**: `proofs/.lake/build/bin/ephemaral` (only when working inside the ephemaral repo itself)

**If not found**, the user can either:
- **Download a pre-built binary** from [GitHub Releases](https://github.com/andremiguelc/ephemaral/releases/latest) into `.ephemaral/bin/` and `chmod +x` it
- **Build from source**: `git clone https://github.com/andremiguelc/ephemaral.git && cd ephemaral/proofs && lake build ephemaral` (requires [elan](https://github.com/leanprover/elan) and Z3)

**Check for Z3**: run `which z3`. If not found, guide the user to install it (`brew install z3` / `apt install z3` / [GitHub releases](https://github.com/Z3Prover/z3/releases)).

### Producing .aral-fn.json — ALWAYS use the parser

**Before running verification, invoke the `ephemaral-parser` skill.** The parser produces `.aral-fn.json` files from source code automatically. There is no hand-crafting JSON — ever. The `.aral-fn.json` is an internal format between the parser and verifier, not a user-facing artifact. If the parser can't extract an expression, that's a parser limitation to surface — not something to work around manually.

## Running verification

```bash
# Verify a function against ALL invariants — glob everything
.ephemaral/bin/ephemaral .ephemaral/parsed/<function>.aral-fn.json .ephemaral/rules/*.aral

# Compile invariants to SMT-LIB (inspect what the compiler produces)
.ephemaral/bin/ephemaral .ephemaral/rules/<type>.aral [more-rules.aral ...]
```

Output: either `VERIFIED` (the function is safe for all valid inputs) or `COUNTEREXAMPLE FOUND` (with exact values and a diagnosis).

**Always pass all `.aral` files** — use `.ephemaral/rules/*.aral`. The pipeline routes each file by type automatically. Irrelevant `.aral` files are silently ignored — no performance cost, no need to figure out which ones match:
- Files matching the function's **input type** → applied as pre+post conditions (the function must preserve these)
- Files matching a **typed parameter's type** → applied as preconditions on that parameter (assumed true on input)
- Files matching **no type in the function** → silently skipped

## Surfacing invariants

The hardest part of verification isn't running the tool — it's knowing what rules to write. Invariants enter the system from four directions:

### From type definitions

Given a type with numeric fields, ask: what must be true about every instance of this type, in every context, at all times? Look for:
- **Non-negativity bounds** — fields that represent things which can't go below zero
- **Relationships between fields** — one field computed from others (a whole equals the sum of its parts)
- **Collection totals** — a scalar field equals the sum of items in a collection
- **Per-item constraints** — every item in a collection satisfies a predicate (e.g., all quantities positive, all values within range)
- **Ordering constraints** — one field must always be >= another
- **Bounded ranges** — fields that must stay within a fixed range

Propose candidates, but apply the universal truth test: if you can imagine a legitimate scenario where this rule doesn't hold for this type, it's not an invariant — it's a context-dependent constraint. The space of per-record, universally-true invariants is small. That's expected and healthy.

### From verification results

Every counterexample points at what to investigate next. An UNCONSTRAINED_PARAMETER tells you which parameter needs bounding. An INVARIANT_GAP names the field without coverage. The tool discovers constraints you didn't know you needed.

### From incidents

When something breaks in production, formalize the business rule that was violated as an invariant. This turns a one-time fix into a permanent guarantee.

### From existing validation code

Guard clauses, input checks, and validation logic in the codebase already encode invariants implicitly. Extract and formalize them.

Start with one or two candidates per type. Don't enumerate upfront — let the tool guide you.

## The discovery loop

This is the core workflow. It works better than trying to get all invariants right before running anything.

1. **Pick one function and one invariant** — the simplest of each
2. **Run verification**
3. **Read the result:**
   - VERIFIED → add the next invariant and re-run
   - COUNTEREXAMPLE → it tells you what to investigate next (see "Reading results" below)
4. **Repeat** — each run either confirms a guarantee or reveals the next question

Each counterexample points at the next thing. The tool drives the conversation.

## Writing .aral files

### File format

```
# Human-readable explanation of the rule
invariant rule_name:
  order.total >= 0
```

- Lines starting with `#` are comments — for humans, ignored by the tool
- Names use `snake_case`
- The expression is indented under the declaration
- Separate invariants with blank lines

The identifier before the dot (`order` above) names the type the invariant binds
to. Use the lowercase of your actual type — `order` for `Order`, `payment` for
`Payment`, and so on. It is a type binding, not a fixed keyword.

### How invariants bind to code

The identifier before the dot is the type binding — in `order.total`, `order`
matches a type called `Order` case-insensitively. The tool checks:

> If the input satisfies all invariants, does the output also satisfy them?

Field names in invariants must match field names on the type.

### File organization

Place `.aral` files in `.ephemaral/rules/`. Organize by type or business concern. Name files to reflect what they protect. Multiple `.aral` files can coexist and be used independently with different functions.

### The Aral language

Read `references/aral-language.md` inside this skill folder for the full construct reference — supported syntax, operators, and examples.

## Reading results

### VERIFIED

The function is mathematically guaranteed to preserve the listed rules for all valid inputs. When typed parameters are involved, VERIFIED states its assumptions — which parameter rules were assumed. The guarantee is conditional: if a caller passes an invalid parameter, it doesn't hold.

### COUNTEREXAMPLE FOUND

The tool shows the exact values that cause a violation, then classifies the issue. A counterexample is information — it maps where safety lives and who owns it.

**Three diagnostic categories:**

#### UNCONSTRAINED_PARAMETER

A parameter has no rules limiting its values. The tool found extreme values that break output rules. What this means depends on the parameter's name:

**Real function parameter** (no `__ext_` prefix — name matches an actual function argument):
A genuine trust boundary. The function accepts unvalidated input and the counterexample is actionable.
- **Primitive parameter**: add a guard inside the function (reject/clamp), or accept that callers must validate.
- **Typed parameter**: write an `.aral` file for the parameter's type and pass it to the verifier. It applies those rules as preconditions.

**Parser extraction gap** (`__ext_` prefix — e.g., `__ext_functionName_0`):
The parser couldn't trace this sub-expression. The counterexample is against the model's incompleteness, not the function's logic. The readable name after `__ext_` identifies what couldn't be resolved. Worth noting as a coverage boundary, but not a code bug. Real bugs surface as RULE_CONFLICT with fully extracted models.

#### INVARIANT_GAP

A field used in the computation has no rule bounding it. Since the field can be anything, the function can produce output that breaks other rules.

What to do: add a rule for the unbounded field, or add a rule expressing the relationship between fields.

#### RULE_CONFLICT

All parameters and fields are constrained, but the function's logic conflicts with the rules.

What to do: either the function has a bug, or the rules don't reflect the actual business intent. Compare the input/output values. This category is neutral — early on, rules may be conjectures that need adjusting.

## What counterexamples teach

Beyond the immediate fix, counterexamples reveal deeper things:

### The invariant set might be incomplete

When a function fails because a field is unbounded, the issue isn't always in the function — it might be in the invariant set. A field used in a computation but uncovered by any rule is a gap in the specifications. The response might be to add an invariant, not to change the code.

### A passing "buggy" function signals a type boundary

When a function with a known semantic bug VERIFIES, the invariant space for that type is too broad. The type conflates distinct concepts. The fix is to split the type — not to add conditional invariants. Each split type gets its own `.aral` file with the stronger guarantees that hold in that specific context.

### Guard clauses are load-bearing — or not

Verifying a function with and without its guard clause shows whether the guard is safety-critical. If both versions verify, the guard is defensive but not required for invariant preservation. If only the guarded version verifies, the guard is load-bearing.

### Parameters reveal trust boundaries

UNCONSTRAINED_PARAMETER isn't a tool limitation — it's the tool telling you "someone must own the safety of this value." The counterexample forces an explicit decision: does the function guard against bad inputs, or does the caller guarantee them?

## Maintaining invariants

### When to add

After the discovery loop surfaces a gap. After an incident. When a new type enters the system. Always propose to the user — don't add silently.

### When to remove or weaken

If an invariant blocks legitimate uses across multiple contexts, it might be too strict. The universal truth test: if there's a real, valid scenario where this rule doesn't hold, it's not an invariant. Remove or weaken it.

### When to split types

When you find yourself wanting conditional invariants ("this rule holds only in context X"), that's a signal the type serves multiple roles. Split into distinct types, each with its own `.aral` file. The tool's verification becomes stronger because each type has a tighter invariant set.

## Optional field handling

When a function's `.aral-fn.json` includes `optionalFields`, the pipeline handles nullable fields automatically:

- Each optional field gets a presence flag constrained to 0 or 1
- Invariants referencing optional fields are auto-guarded: the invariant only applies when the field is present
- No special `.aral` syntax needed — write invariants as if the field always exists, the pipeline adds the presence guard

In counterexample output, presence flags are annotated with plain language:
```
has-field = 0 (field was not provided)
```

This means the counterexample was found in a scenario where the optional field was absent.