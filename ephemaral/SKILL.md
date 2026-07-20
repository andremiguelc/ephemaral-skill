---
name: ephemaral
description: Hunt for counterexamples — inputs that make a function break a business rule — by authoring the `.aral-fn.json` IR directly from the source and searching it exhaustively. Use this skill whenever the user mentions invariants, verification, business rules, .aral files, writing constraints, or wants to check whether a function can produce invalid output. Also trigger when the user asks to "verify", "prove", "check", or "find counterexamples for" a function against rules.
license: MIT
compatibility: Requires ephemaral binary v0.1.5 (Lean 4) and Z3 solver. You read the source and author the `.aral-fn.json` IR directly, writing any per-parameter preconditions yourself.
metadata:
  author: andremiguelc
  version: "2026-07-20"
  repository: "https://github.com/andremiguelc/ephemaral-skill"
---

# ephemaral v0.1.5 — Counterexample Finder for Business Rules

ephemaral hunts for **counterexamples**: concrete inputs that make a function produce output which violates a business rule. It searches exhaustively over the whole input space — not by sampling test cases — so when it surfaces a counterexample, it hands you the exact values that trigger the violation. When it finds none, that's a weak signal ("no counterexample within the model"), not a proof of correctness.

**Your goal while modeling is to find counterexamples**, and that shapes how you work. Start from the types and fields your `.aral` invariants name, find where the code assigns those fields, and model that assignment zone with an eye for where the value could break the rule. You read the source and write the `.aral-fn.json` by hand; the verifier validates it against a fixed schema, and everything downstream is proved correct in Lean 4. A found counterexample is cheap to confirm — you can run the real function on the witness — so a wrong model can't fool you there (see "Reading results").

The tool takes two inputs:
- A function representation (`.aral-fn.json`) — how the fields your rules track get their values. You author this by hand from the source (see "Writing .aral-fn.json yourself" below).
- One or more invariant files (`.aral`) — what must always be true about the data.

The invariants are the primary artifact. Functions come and go; invariants persist.

**Ephemaral is an oracle, not a verdict.** When the verifier rejects an invariant, it's saying "the code and the rule disagree on this input." It is *not* saying which side is wrong. The discipline is to interrogate the disagreement before deciding. Sometimes the rule is a guess that doesn't survive contact with real semantics; retire it. Sometimes the code has a defect the rule correctly catches; fix the code. Sometimes the rule is right and the bug is upstream of where the verifier points; trace it. Defaulting either way without inspecting both is how false confidence creeps in — a MODEL CHECKED result stops meaning anything, and counterexamples stop being read carefully.

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
- **parsed/** — `.aral-fn.json` files you author by hand from the source. Gitignore this (regenerable — just re-read the source and rewrite the IR).
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

## Writing .aral-fn.json yourself

You read the source and write the IR by hand. The full node-by-node reference — every expression and boolean shape, the source→IR recipes, and a worked example — lives in `references/ir-authoring.md`, and the exact schema you must validate against is bundled at `references/aral-fn.schema.json`. Read the guide before writing your first IR. What follows is the mental model that keeps this honest.

### Model the assignment zone — and hunt while you do it

This is the single most important idea. Your invariants name a type and some fields; your job is to find every place the code assigns those fields and model that slice — the **assignment zone** — because that's the only place a counterexample can be born. You model around one field's assignment, not the surrounding code and not the return object as a whole.

Work from the rule outward, field by field: take a field the invariant constrains, locate where the code sets or modifies it, and ask *"what input could make this assignment break the rule?"* That question is the point — you're not transcribing the function neutrally, you're looking for where its value could go out of bounds.

Here is why the narrow slice is enough. The IR follows a spread-and-override contract: any field you do **not** list in `assigns` is assumed to pass from input to output unchanged (the `...record` spread). So you only describe the fields the rules track. The surrounding code can log, mutate untracked state, call out to services, or do anything else — none of it reaches the IR, as long as the code that assigns the tracked field is pure arithmetic and guards.

To write an IR: for each tracked field, find where it is assigned, capture that expression, and stop. If the assigned value never depends on a field or parameter, you don't need to mention it.

### Guards land in two different places

When you read the assignment zone, you will find guards. Which kind decides where they go in the IR:

- **A value-selecting guard** chooses *what* value is assigned — a ternary (`x > 0 ? a : b`), or an `if/else` that assigns the field differently on each branch. This becomes an `ite` node inside the field's assigned value.
- **An entry guard** gates *whether* the assignment runs at all — an early `return`/`throw`, or an `assert(amount >= 0)` before the assignment. This becomes a `paramPreconditions` entry: a predicate the verifier may assume about that parameter. You write it directly from the guard you read in the assignment zone.

Getting this split right is most of the skill. A missing entry guard shows up as an UNCONSTRAINED_PARAMETER counterexample (see "Reading results"), which is your cue that the assignment zone had a precondition you didn't capture.

### Calls you can't model become named ad-hoc params

Sometimes the assigned value flows through a call you can't see into — `parseFloat(...)`, an external API, a library helper. Don't guess its behavior. Introduce a parameter standing in for its output, named after the real thing with a short `ext_` marker so it's clearly built ad hoc: a `parseFloat(raw)` feeding a `price` field becomes a param like `ext_price` (or `ext_parseFloat`). Then handle it by whether the source *guards* that output:

- **Guarded** — if the code checks the result before using it (`if (x < 0) throw`, a clamp, an assert), pull that guard into the IR: a value-selecting clamp becomes an `ite`, an entry check becomes a `paramPreconditions` entry on the `ext_` param. Now the search can't pick values the real code would have rejected.
- **Unguarded** — if the code trusts whatever comes back, leave the `ext_` param free and say so to the human: this is a trust boundary, and any counterexample that leans on it depends on that unguarded value. The verifier surfaces it as `UNCONSTRAINED_PARAMETER` (see "Reading results").

Rule of thumb: every unmodeled output gets a named `ext_` param; if it's guarded the guard travels with it, and if it's not, the human hears about it.

### Before you read a `MODEL CHECKED` — check the prefix

One silent trap: every `.aral` invariant's prefix must equal the lowercased `inputType` in your IR. If the IR says `"inputType": "Order"`, the rule must read `order.total >= 0`. Watch out for placeholders — `<root>`, `<type>`, and `<field>` are stand-ins used in the reference docs, never literals to copy. A mismatch (writing a literal `<root>.total` or `<type>.total`, or using the wrong type name) is **silently skipped** — the rule is checked against nothing, so no counterexample can ever be found and the run still prints `MODEL CHECKED`. That's a false "nothing found." Nothing warns you here, so make this a habit: confirm every rule prefix matches the IR's `inputType` before you read a clear result as meaningful.

## Running verification

```bash
# Verify a function against ALL invariants — glob everything
.ephemaral/bin/ephemaral .ephemaral/parsed/<function>.aral-fn.json .ephemaral/rules/*.aral

# Compile invariants to SMT-LIB (inspect what the compiler produces)
.ephemaral/bin/ephemaral .ephemaral/rules/<type>.aral [more-rules.aral ...]
```

Output: either `MODEL CHECKED` — the search found no counterexample *within this model*, a weak signal rather than a proof — or `COUNTEREXAMPLE FOUND`, with the exact values and a diagnosis.

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

1. **Pick one invariant and find where its fields get assigned** — usually several functions or sites, not just one. Model the assignment zone at one site (see "Writing .aral-fn.json yourself"), looking for where the value could break the rule.
2. **Run the search**
3. **Read the result:**
   - MODEL CHECKED → nothing broke *within this model*; move to the next site that assigns the field, or add the next invariant, and keep hunting.
   - COUNTEREXAMPLE FOUND → the tool hands you the exact values; this is the find you were after (see "Reading results" below).
4. **Repeat** — across every site that assigns the tracked fields; each run either turns up a counterexample or tells you where to look next.

Each counterexample points at the next thing. The tool drives the conversation.

## Writing .aral files

### File format

```
# Human-readable explanation of the rule
invariant rule_name:
  <type>.<field> >= 0
```

- Lines starting with `#` are comments — for humans, ignored by the tool
- Names use `snake_case`
- The expression is indented under the declaration
- Separate invariants with blank lines

### How invariants bind to code

In every example here, `<type>` and `<field>` are placeholders — **never copy them literally** into your `.aral` file. Substitute the actual names:

- `<type>` → the lowercase of the function's input/output type name. For a type called `PagePaginationInformation`, write `pagepaginationinformation`. For `Order`, write `order`. Matching is case-insensitive.
- `<field>` → a field on that type.

It is a type binding, not a fixed keyword. The identifier before the dot changes per invariant depending on which type you're constraining.

**Never write `<root>.<field>` or `<type>.<field>` literally.** `<root>` and `<type>` are placeholders meaning "the root of the expression" — substitute the lowercased type name. Writing a literal `<root>.total` (or `<type>.total`) against a real type parses but matches no type, so the pipeline silently skips the invariant.

The tool checks:

> If the input satisfies all invariants, does the output also satisfy them?

Field names in invariants must match field names on the type.

### File organization

Place `.aral` files in `.ephemaral/rules/`. Organize by type or business concern. Name files to reflect what they protect. Multiple `.aral` files can coexist and be used independently with different functions.

### The Aral language

Read `references/aral-language.md` inside this skill folder for the full construct reference — supported syntax, operators, field domains (like `is integer`), and examples.

For the **function side** — how to turn source into `.aral-fn.json` — read `references/ir-authoring.md`, and validate your JSON against the bundled `references/aral-fn.schema.json`.

## Reading results

### MODEL CHECKED

Read this as **"the search found no input that breaks these rules, within the model you gave it."** It is a weak signal, not a proof: it's only as good as how faithfully your IR models the assignment, and it says nothing about corners the model didn't capture. When typed parameters are involved, the output also lists which parameter rules it assumed. Don't report it as a guarantee — report it as "nothing found yet," and move on to the next site or invariant.

### COUNTEREXAMPLE FOUND

The tool shows the exact values that cause a violation, then classifies the issue. A counterexample is information — it maps where safety lives and who owns it.

**Three diagnostic categories:**

#### UNCONSTRAINED_PARAMETER

A parameter has no rules limiting its values. The tool found extreme values that break output rules. The counterexample is a genuine trust boundary — the function accepts unvalidated input. Three ways to remove it:

- **Primitive parameter, in-function constraint**: the function already constrains the parameter locally (an early `return`/`throw` on bad input, or an `assert`). Capture that entry guard as a `paramPreconditions` entry in the IR — write the predicate by hand from the guard you read in the assignment zone. The verifier asserts it before postcondition checking, dropping the parameter out of the unconstrained set. If you see this diagnostic and the source *does* guard the parameter, that's your signal you missed the guard when writing the IR.
- **Primitive parameter, caller discipline**: accept that callers must validate. The verifier still flags the trust boundary, but you've made an explicit decision to push safety upstream.
- **Typed parameter**: write an `.aral` file for the parameter's type and pass it to the verifier. It applies those rules as preconditions on the typed parameter automatically.

##### Where preconditions come from

The verifier merges preconditions from two sources before checking the function:

1. **`.aral` typed-parameter invariants.** Any `.aral` file whose root prefix matches a typed parameter's type is applied as a precondition on that parameter.
2. **Hand-written per-function preconditions.** When the IR's `paramPreconditions` field is non-empty — because you read an entry guard in the assignment zone and wrote the predicate into the IR — each predicate is asserted as a precondition on the named parameter.

Both sources feed the same downstream check. A parameter is "constrained" when at least one source provides a predicate that mentions it. The verifier doesn't care which source produced the predicate — only that one exists.

#### INVARIANT_GAP

A field used in the computation has no rule bounding it. Since the field can be anything, the function can produce output that breaks other rules.

What to do: add a rule for the unbounded field, or add a rule expressing the relationship between fields.

#### RULE_CONFLICT

All parameters and fields are constrained, but the function's logic conflicts with the rules.

What to do: either the function has a bug, or the rules don't reflect the actual business intent. Compare the input/output values. This category is neutral — early on, rules may be conjectures that need adjusting.

## What to do with a counterexample

A counterexample is the thing you were hunting for, but it's a *lead*, not yet a confirmed bug — it was found against your model of the code, not the code itself. There's a useful asymmetry here: a counterexample is cheap to check, while a MODEL CHECKED result leans entirely on your IR being faithful.

So when the verifier hands you a counterexample, offer the human the next move rather than deciding for them:

- **Confirm it** — run the *actual* source function on the witness values. If the real function reproduces the violation, the bug is real, and you proved it by running the code, not your model. If it does *not* reproduce, the model was looser than the code (often an unmodeled call left too free, or a missing guard) — tighten the IR and search again.
- **Capture it** — turn the witness into a regression test so the case stays pinned down.

Present both, with the witness values, and let the human choose. Neither is mandatory — but a counterexample nobody checks is just a lead left on the table.

If you ever want to gauge how much to trust a MODEL CHECKED result, the honest check is a differential one: evaluate the tracked field two ways — real function and IR — on a batch of sample inputs and see whether they agree. It's optional here, because this workflow never rests on "no counterexample" meaning correctness.

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