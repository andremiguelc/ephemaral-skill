# IR Authoring Reference — writing `.aral-fn.json` by hand

*Version: 2026-07-06*

This guide teaches you to turn source code into the `.aral-fn.json` IR that the
verifier consumes. You read the source and write the JSON. The exact schema you
must satisfy is bundled next to this file as `aral-fn.schema.json`; treat it as
ground truth and validate against it.

The IR sits **exactly at the proof boundary**. Everything before it (you, the
source, this translation) is unproved. Everything after it (compilation to
SMT-LIB, the Z3 query) is proved correct in Lean 4. So the whole burden on you
is a single thing: **model the assignment faithfully.** The verifier will do the
rest, but it can only be as right as your IR.

**Read this before any example below.** Every code snippet and JSON block in
this guide is an **illustrative example**, never a literal to copy. They use a
concrete scenario — a `Record`/`adjustValue` function with names like `value`
and `delta`, or a `fee`/`amount` billing case — purely so you can see complete,
runnable JSON. In your own work, *all* of those names come from the user's actual
code; substitute them. The instruction is always in the prose; the block only
shows the shape it produces. (The `.aral` rules reference does the same job with
`<type>`/`<field>` angle-bracket placeholders; this guide uses concrete names
instead so the JSON is complete, but they are examples just the same.)

## The one idea: model the assignment zone

Read this before anything else, because it shrinks the job dramatically.

You model only the **assignment zone** — the code that gives each field your
`.aral` rules track its value, together with the guards around that assignment.
You are modeling around one field's assignment, nothing wider.

This works because of the IR's **spread-and-override contract**. Any field you do
*not* list in `assigns` is assumed to travel from input to output unchanged — the
`...record` spread. So you only ever describe the fields the rules track. If the
code returns `{ ...order, total: newTotal }` and your rule is about `total`, you
model how `total` is assigned and ignore every other field.

The freeing consequence: the surrounding code can be unmodelable and it still
doesn't matter. Logging, mutating untracked state, database calls, throwing —
none of it reaches the IR, as long as the code that assigns the *tracked field*
is pure arithmetic and guards. You are verifying one field's assignment, not the
plumbing around it.

Procedure: for each tracked field, find where it is assigned, capture that
expression, and stop.

## Top-level shape

An IR is one JSON object describing how the code assigns its tracked fields.
Required keys first:

```json
{
  "name": "adjustValue",
  "inputType": "Record",
  "inputFields": ["value"],
  "params": ["delta"],
  "assigns": [ /* one FieldAssign per tracked field you set */ ]
}
```

- **`name`** — the function name, used only in diagnostics.
- **`inputType`** — the type being transformed. Matched case-insensitively to your
  `.aral` rule prefixes: `"Record"` matches rules written `record.value >= 0`.
  Getting this wrong (or mismatching the rule prefix) makes rules silently skip.
- **`inputFields`** — every field on the input type that verification touches:
  fields you assign, fields you read inside an expression (including inside an
  `ite` condition), and fields any applicable invariant mentions. Fields not
  listed are assumed unchanged by spread.
- **`params`** — extra inputs beyond the object (e.g. `delta`, `rate`). These are
  free variables: Z3 will try every possible value unless a precondition bounds
  them.
- **`assigns`** — the heart of the IR: which fields change, and to what. Each entry
  is `{ "fieldName": "...", "value": <Expr> }`. Fields not listed are preserved.

Three optional keys, each covered in its own section below: `typedParams`,
`optionalFields`, `paramPreconditions`.

## Expressions (`Expr`) — the value side

An `Expr` computes a number. There are seven shapes. Each is a JSON object with
exactly one key naming its variant.

**`lit`** — a numeric literal. Integers only. Convention: `1` = true, `0` = false
for boolean-ish fields.
```json
{ "lit": 0 }
```

**`field`** — a reference to an input field or a parameter. A bare `name`
resolves against `inputFields` or `params`. Add a `qualifier` to read a field on
a typed parameter (e.g. `config.rate`).
```json
{ "field": { "name": "value" } }
{ "field": { "qualifier": "config", "name": "rate" } }
```

**`arith`** — a binary arithmetic op: `add`, `sub`, `mul`, `div`. Nest to build
bigger expressions. Division is **total**: `a / 0 = 0` (no precondition needed).
Note the divisor-can-be-zero case is **not** flagged today, and the domain has no
NaN/Infinity — so a div-by-zero bug shows up as `0`, not a violation. To catch one,
target a finite-valued symptom next to it rather than the NaN itself.
```json
{ "arith": { "op": "sub",
             "left":  { "field": { "name": "value" } },
             "right": { "field": { "name": "delta" } } } }
```

**`ite`** — if-then-else. `cond` is a `BoolExpr`; `then` and `else` are `Expr`s.
This is where a value-selecting guard lands (see "Guards").
```json
{ "ite": { "cond":  <BoolExpr>,
           "then":  <Expr>,
           "else":  <Expr> } }
```

**`round`** — apply a rounding mode to an expression. Modes: `floor` (toward −∞),
`ceil` (toward +∞), `half_up` (standard rounding). Use this when the source
computes an integer result via rounding.
```json
{ "round": { "expr": { "field": { "name": "total" } }, "mode": "floor" } }
```

**`sum`** — sum a per-item expression over a collection. `collection` is a plain
field name on the input type. `body` is a per-item `Expr` whose field references
are **item-scoped** (bare item field names, not `inputType`-prefixed).
```json
{ "sum": { "collection": "lineItems",
           "body": { "arith": { "op": "mul",
                                "left":  { "field": { "name": "price" } },
                                "right": { "field": { "name": "quantity" } } } } } }
```

(That seventh shape — the qualified `field` ref — is the same `field` variant with
a `qualifier`, listed above.)

## Booleans (`BoolExpr`) — the condition side

A `BoolExpr` is true or false. It appears as an `ite` condition and inside
`paramPreconditions`. Five shapes:

**`cmp`** — compare two `Expr`s: `eq`, `neq`, `gt`, `lt`, `gte`, `lte`.
```json
{ "cmp": { "op": "gt",
           "left":  { "field": { "name": "delta" } },
           "right": { "lit": 0 } } }
```

**`logic`** — combine two `BoolExpr`s with `and` or `or`.
```json
{ "logic": { "op": "and", "left": <BoolExpr>, "right": <BoolExpr> } }
```

**`not`** — negate a `BoolExpr`.
```json
{ "not": <BoolExpr> }
```

**`isPresent`** — true when an optional field has a value. The `name` must be a
field listed in `optionalFields` (or a qualified ref to a typed-parameter field).
```json
{ "isPresent": { "name": "bonus" } }
```

**`each`** — universal quantifier: true when `body` holds for every item in a
collection. Like `sum`, `collection` is a plain input-type field name and `body`
is item-scoped.
```json
{ "each": { "collection": "lineItems",
            "body": { "cmp": { "op": "gt",
                               "left":  { "field": { "name": "quantity" } },
                               "right": { "lit": 0 } } } } }
```

## Guards land in two different places

When you read the assignment zone you will find guards. Which kind decides where
it goes in the IR — this distinction is most of the skill.

A **value-selecting guard** chooses *what* value is assigned. A ternary, or an
`if/else` that assigns the field differently per branch. It becomes an `ite`
inside the assigned value:

```ts
value: delta > 0 ? record.value - delta : record.value
```
```json
{ "ite": {
    "cond": { "cmp": { "op": "gt", "left": { "field": { "name": "delta" } }, "right": { "lit": 0 } } },
    "then": { "arith": { "op": "sub", "left": { "field": { "name": "value" } }, "right": { "field": { "name": "delta" } } } },
    "else": { "field": { "name": "value" } } } }
```

An **entry guard** gates *whether* the assignment runs at all. An early
`return`/`throw`, or an `assert(delta >= 0)` before the assignment. It becomes a
`paramPreconditions` entry — a fact the verifier may assume about that parameter:

```ts
if (delta < 0) throw new Error("delta must be non-negative");
// ... then assign using delta
```
```json
"paramPreconditions": [
  { "name": "delta",
    "predicates": [
      { "cmp": { "op": "gte", "left": { "field": { "name": "delta" } }, "right": { "lit": 0 } } }
    ] } ]
```

If you miss an entry guard, you'll see an UNCONSTRAINED_PARAMETER counterexample:
Z3 found an extreme value the real code would have rejected. That's your cue to
go back and capture the guard.

## Source → IR recipes

| Source construct | IR node |
|---|---|
| `a ? b : c` (ternary) | `ite` with `cond` = the test |
| `if (t) x = b; else x = c;` (value-selecting) | `ite` for `x` |
| early `return`/`throw` / `assert` on a param | `paramPreconditions` entry |
| `a + b * c` | nested `arith`; `mul`/`div` bind tighter than `add`/`sub` |
| `Math.floor(x)` / `Math.ceil` / rounding | `round` with the matching `mode` |
| `items.reduce((s, i) => s + i.amount, 0)` | `sum` over `items`, item-scoped `body` |
| `items.every(i => i.qty > 0)` | `each` over `items`, item-scoped `body` |
| `config.rate` (field on a typed param) | `field` with `qualifier: "config"` + `typedParams` entry |
| nullable field access | list in `optionalFields`; guard with `isPresent` |
| `helper(x)` call feeding the assigned value | new `params` entry (open variable); bound it with `paramPreconditions` if you know its output range (see "Function calls in the assignment zone") |

Precedence matters when you flatten infix math into trees. `a + b * c` is
`add(a, mul(b, c))`, not `mul(add(a, b), c)`. Build the tighter-binding operator
as the inner node.

## Typed parameters and optional fields

**`typedParams`** lets a parameter carry a type, so `.aral` rules for that type
apply as *preconditions* on it. Declare the parameter in both `params` and
`typedParams`, then reference its fields with a `qualifier`:

```json
"params": ["config"],
"typedParams": [ { "name": "config", "type": "DiscountConfig" } ]
```
Now a rule file with prefix `discountconfig` (e.g. `discountconfig.rate >= 0`) is
assumed true about `config` on every run.

**`optionalFields`** lists fields that may be absent. For each, the pipeline
creates a presence flag and auto-guards any invariant mentioning the field — you
write invariants as if the field always exists. Use `isPresent` in the IR only
when the *function's own logic* branches on presence.

## Function calls in the assignment zone

> Every code snippet and JSON block in this section is an **illustrative
> example**. Names like `fee`, `amount`, and `computeFee` stand in for whatever
> appears in the user's real code — do not copy them literally. The *rule* is in
> the prose; the blocks only show its shape.

Sometimes the value assigned to a tracked field comes from a call to another
function. The IR has no node for a call, and you should not try to inline
arbitrary code. Instead, treat the **result of the call as an open variable**:
add a name to `params` and use it in the assigned expression, exactly as if it
were an ordinary parameter. The call's body becomes invisible; only its output
value enters the IR.

Name that param after the real thing with a short `ext_` marker, so anyone reading
the IR can see it was built ad hoc to stand in for an unmodeled call. For example,
if the real code assigned `fee = computeFee(amount)`, you would add a param like
`ext_fee` (or `ext_computeFee`) and reference it where `fee` is used. The `fee` and
`amount` names here are illustrative — use whatever the user's code uses, with the
`ext_` marker in front of the stand-in.

That leaves one question — what values can that output take? There are two cases,
and which one you're in decides how trustworthy the result is.

**Case 1 — the output is guarded/bounded.** If you can read the called function, or
see the source guard its result (a clamp, an `assert`, an early `throw`), and state
a clear, honest range for what it returns (for example: it never returns negative,
or it always returns something `<= amount`), then carry that into the IR — write
the bounds as `paramPreconditions` on the `ext_` variable, or fold a value-selecting
clamp into an `ite`. Now the verifier assumes only values
the real function could actually produce, and it won't waste your time on
counterexamples that can't happen. Only add a bound you are confident is true; an
over-tight bound hides real bugs, which defeats the point.

The following is an **example** of what such preconditions look like — the names
`fee` and `amount` are illustrative stand-ins:

```json
"params": ["fee"],
"paramPreconditions": [
  { "name": "fee",
    "predicates": [
      { "cmp": { "op": "gte", "left": { "field": { "name": "fee" } }, "right": { "lit": 0 } } },
      { "cmp": { "op": "lte", "left": { "field": { "name": "fee" } }, "right": { "field": { "name": "amount" } } } }
    ] } ]
```

**Case 2 — the output is unguarded.** If the source trusts whatever the call
returns and you can't cleanly characterize it, leave the `ext_` variable
unconstrained. The verifier then treats it as a free value and will try every
possibility — so it may surface a counterexample driven entirely by that call's
result. That is the correct outcome, not a failure: it's the tool telling you this
call is an unowned trust boundary. It shows up as an UNCONSTRAINED_PARAMETER, exactly
like any other unbounded value. **Say so to the human** — any counterexample that
leans on this value depends on the call being unguarded, and the decision to trust
it (or to bound it, Case 1) is theirs to make.

## Worked example — the whole thing end to end

Source:
```ts
function adjustValue(record: Record, delta: number): Record {
  return {
    ...record,
    value: delta > 0 ? record.value - delta : record.value,
  };
}
```

Assignment zone: only `value` is tracked. It's set by one value-selecting guard.
Nothing else in the return needs modeling — `...record` handles the rest.

IR:
```json
{
  "name": "adjustValue",
  "inputType": "Record",
  "inputFields": ["value"],
  "params": ["delta"],
  "assigns": [
    {
      "fieldName": "value",
      "value": {
        "ite": {
          "cond": { "cmp": { "op": "gt", "left": { "field": { "name": "delta" } }, "right": { "lit": 0 } } },
          "then": { "arith": { "op": "sub", "left": { "field": { "name": "value" } }, "right": { "field": { "name": "delta" } } } },
          "else": { "field": { "name": "value" } }
        }
      }
    }
  ]
}
```

Rule (`record.aral`):
```
invariant value_non_negative:
  record.value >= 0
```

Run it and you get **COUNTEREXAMPLE FOUND** at `value = 0, delta = 0.5` → output
`-0.5` — the find you were hunting for. Offer to re-run the real `adjustValue` on
those values to confirm the bug (it reproduces). That confirmation costs zero trust
in your IR — you ran the actual code.

## Gauging a MODEL CHECKED result (optional)

A `MODEL CHECKED` means the search found no counterexample *within your model* — a
weak signal, only as good as your translation, not a proof. If you want to gauge how
much to trust it, run a differential check: evaluate the tracked field both by
running the real function and by evaluating your IR, across a batch of sample inputs,
and confirm they agree. If they diverge, your IR is looser than the code — fix it.
This is optional, because the workflow never rests on "no counterexample" meaning
correctness; the weight is on the counterexamples you find and confirm. See the
SKILL's "What to do with a counterexample" section.

## What the IR cannot express

These are deliberate boundaries — if the assignment zone needs one of these, the
function is outside the verifiable subset and should be refactored, not forced:

- No loops or variables — collection work goes through `sum` / `each`.
- No function calls — inline the computation, or the helper is opaque.
- No mutation — model pure spread-and-override only.
- No nested collections, and collections must be a plain field name (not
  `order.nested.items`).
- Item-scoped bodies can't reach parent record fields — `sum`/`each` bodies see
  only item fields.