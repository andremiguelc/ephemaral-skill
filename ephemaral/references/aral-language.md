# Aral Language Reference

*Version: 2026-04-23*

Aral is a constraint language for describing what must always be true about data. Each construct below is formally verified — its compilation has been proved correct.

## A note on the examples

Throughout this file, examples use `<type>` and `<field>` as placeholders in angle brackets. **These are not literal keywords** — substitute them with the actual names in your code.

- `<type>` stands for the lowercase of the type your invariant binds to. For a type called `PagePaginationInformation`, write `pagepaginationinformation`. For `Order`, write `order`. Matching is case-insensitive.
- `<field>` stands for a field name on that type.

**Do not write `<root>.<field>` literally.** `<root>` is a placeholder, not a literal prefix — substitute the lowercased type name. Writing a literal `<root>.total` against any real type parses but matches nothing, and the pipeline silently skips the invariant.

## Primitives

### Field reference

A dot-separated path to a field on a typed object.

```
<type>.<field>
```

The identifier before the dot is the type binding (matched case-insensitively to the function's input type). The field is a field on that type. The type prefix is stripped during compilation — only the field name matters for verification.

### Numeric literal

Integer values.

```
0    100    -5
```

### Comparison operators

```
==   !=   >   <   >=   <=
```

## Scoping rules

There are exactly two scopes in the language:

1. **Record scope** — `<type>.<field>` references. Available everywhere. These are scalar fields on the record being constrained.
2. **Item scope** — bare field names inside `sum()` and `each()` bodies. These reference fields on the current collection item. Only available inside collection body expressions.

**The two scopes do not mix inside a collection body.** A `sum` or `each` body can only reference item fields (bare names). It cannot reference parent record fields (`<type>.<field>`). This is a hard constraint of the language — the body is evaluated per-item in isolation.

To constrain relationships between item fields and record-level fields, use separate invariants: one `each`/`sum` for the per-item property, and a separate scalar invariant for the record-level relationship.

## Constructs

### Field comparison

Compare an expression to another expression.

```
invariant non_negative:
  <type>.<fieldA> >= 0

invariant fields_equal:
  <type>.<fieldA> == <type>.<fieldB>
```

Both sides can be field references, literals, or arithmetic expressions.

### Arithmetic

Binary operations between expressions: `+`, `-`, `*`, `/`.

```
invariant difference_non_negative:
  <type>.<fieldA> - <type>.<fieldB> >= 0

invariant product_non_negative:
  <type>.<fieldA> * <type>.<fieldB> >= 0
```

Division is total: dividing by zero equals zero, and the domain has no NaN or Infinity — so a real division-by-zero bug produces `0` in the model, not a violation. The tool does **not** currently flag divisor-can-be-zero, so it won't catch a NaN-shaped bug on its own. Hunt around it: pick an invariant whose violation is a *finite* value that sits next to the zero divisor (a negative result, a count reaching zero), and the finite counterexample will point you at the same site.

Multi-term arithmetic with correct precedence is supported. `*`/`/` bind tighter than `+`/`-`, and expressions are left-associative.

```
invariant sum_of_parts:
  <type>.<whole> == <type>.<partA> + <type>.<partB> - <type>.<adjustment>

invariant mixed_precedence:
  <type>.<result> == <type>.<base> * <type>.<rate> + <type>.<offset>
```

### Boolean connectives

Combine comparisons with `and` or `or`.

```
invariant bounded:
  <type>.<value> >= 0 and <type>.<value> <= 1000

invariant at_least_one_positive:
  <type>.<fieldA> > 0 or <type>.<fieldB> > 0
```

One connective per expression — `a and b or c` is not supported (ambiguous without precedence). Write separate invariants instead.

### Rounding

Rounding modes are used in function bodies (not in invariant expressions). Three modes:

| Mode | Meaning |
|------|---------|
| `floor` | Round toward negative infinity |
| `ceil` | Round toward positive infinity |
| `halfUp` | Round half toward positive infinity (standard rounding) |

The verifier understands rounding — it can prove that rounding preserves non-negativity, for example.

### Field domains

A field domain declares what *kind* of number a field holds. It is a plain
invariant body — it uses the ordinary `invariant <name>:` form and sits alongside
your other rules. Today there is exactly one domain, `integer`.

```
# Item count is always a whole number
invariant count_is_integer:
  <type>.<field> is integer
```

Like any invariant, `is integer` is **assumed on the input and proved on the
output**. It tells the solver "never propose a fractional value for this field,
and the function must keep it whole." It applies to **top-level fields only**
(`order.count` ✓, not `order.line.count`).

This is distinct from `round` above. Rounding is a *computation* in a function
body that produces a whole number; `is integer` is a *constraint* in an invariant
declaring that a field only ever holds whole numbers.

**Why it matters.** Without it, the solver is free to choose fractional inputs
that can't occur at runtime, producing false counterexamples. Suppose `count`
must stay `>= 1` and a function decrements it only when `count > 1`. The solver
can pick `count = 3/2`, decrement to `1/2`, and report a bogus violation of
`>= 1`. Adding `count is integer` rules that out and the function VERIFIES.
Conversely, a function that *halves* `count` genuinely breaks integrality —
halving `7` gives `7/2` — and `is integer` correctly catches it as a real
counterexample.

**No IR node.** `is integer` lives entirely on the `.aral` side. It desugars
during compilation to `field == floor(field)`, reusing the already-proved round
and comparison nodes. You never write it in `.aral-fn.json` — there is no
`isInteger` shape in the IR schema. It's a rule you write, not a function
construct you model.

### Optional fields

When a function declares fields as optional (via `optionalFields` in the JSON), invariants referencing those fields are automatically guarded by presence. Write invariants as if the field always exists — the pipeline adds the guard.

For example, if `<optionalField>` is optional:
```
invariant optional_non_negative:
  <type>.<optionalField> >= 0
```

This is interpreted as: "when adjustment is present, it must be non-negative." No special syntax needed.

### sum — Collection Aggregation

Sum a per-item expression across all items in a collection.

```
sum(<type>.<collection>, <per-item expression>)
```

```
invariant total_matches_items:
  <type>.<total> == sum(<type>.<items>, <itemField>)

invariant total_with_factor:
  <type>.<total> == sum(<type>.<items>, <unitValue> * <quantity>) + <type>.<offset>
```

`sum(<type>.<items>, <itemField>)` reads as: "sum the `<itemField>` field across all `<items>`." Field names after the comma are implicitly item-scoped — no arrows, lambdas, or path repetition needed.

Sum can appear inside larger arithmetic expressions — it returns a numeric value like any other expression.

**Constraints:**
- Body is item-scoped only — bare field names reference item fields (see Scoping rules above)
- Per-item expression supports field references and arithmetic (`<unitValue> * <quantity>` ✓, conditionals ✗)
- No nested sums (`sum(<items>, sum(<subitems>, x))` ✗)
- Collection must be a simple field reference (`<type>.<items>` ✓, not `<type>.<nested>.<items>`)

**How it works with functions:** Two scenarios:
- **Pass-through:** The function modifies scalar fields but doesn't touch the collection. The invariant catches broken scalar-collection relationships. The pipeline reuses input collection data for the output side.
- **Compute from collection:** The tracked field is assigned a total reduced over the items. You write a `sum` expression as that field's assigned value in the IR (a `reduce` in the source becomes a `sum` node). The verifier checks that the assigned value matches the invariant's sum.

### each — Universal Quantifier over Collections

Assert that a boolean predicate holds for every item in a collection.

```
each(<type>.<collection>, <per-item predicate>)
```

```
invariant items_positive:
  each(<type>.<items>, <itemField> > 0)

invariant items_bounded:
  each(<type>.<items>, <itemField> > 0 and <itemField> <= 1000)
```

`each(<type>.<items>, <itemField> > 0)` reads as: "for every item in `<items>`, `<itemField>` is greater than zero." Field names after the comma are item-scoped, just like in `sum`.

The body is a boolean predicate (comparison, `and`/`or`, or nested `each`), not a numeric expression. Think of `each` as the boolean analog of `sum`:

| | `sum` | `each` |
|---|---|---|
| Returns | number | true/false |
| Body | arithmetic expression | boolean predicate |
| Combines items with | `+` | `and` |
| Empty collection | `0` | `true` |

**Constraints:**
- Body is item-scoped only — bare field names reference item fields (see Scoping rules above)
- Per-item predicate supports comparisons and `and`/`or` connectives
- Collection must be a simple field reference (`<type>.<items>` ✓)
- `any` is expressible as `not each(coll, not P)` — no separate keyword needed
- `count` is expressible as `sum(coll, 1)` — no separate keyword needed

**How it works with functions:** Same as `sum` — pass-through collections reuse input accessors. The function doesn't need to touch collection items for the invariant to be checked.

## What the language cannot express

These are deliberate scope boundaries:

- **No variables or assignments** — invariants describe data, not computation
- **No loops** — `sum` and `each` handle collection operations; `count` = `sum(coll, 1)`, `any` = `not each(coll, not P)`
- **No function calls** — invariants reference types, not code
- **No string operations** — beyond `==` and `!=`
- **No cross-entity constraints** — invariants describe one record at a time
- **No temporal properties** — no time-dependent constraints
- **No explicit presence guards in `.aral` syntax** — optional field handling works at the pipeline level via `optionalFields`. Writing `if <type> has <field>: ...` directly in `.aral` expressions is not supported.
- **No parent field references inside collection bodies** — `sum` and `each` bodies can only use bare item field names, not `<type>.<field>`. To compare item fields against record-level values, the record-level constraint must be a separate invariant outside the collection body.
- **No `filter` or conditional aggregation** — `sum(coll, ite(pred, 1, 0))` is expressible in the IR but not yet in `.aral` syntax

## Common mistakes

These patterns look reasonable but will be silently skipped by the pipeline — the rule is checked against nothing, so no counterexample can ever be found and the run still says MODEL CHECKED. Nothing flags them, so check for these yourself before reading a clear result as meaningful.

**Using a placeholder as the literal prefix:**

```
# WRONG — <type> is a placeholder, not a literal identifier to copy into your .aral
invariant non_negative:
  <type>.<field> >= 0

# ALSO WRONG — <root> is a placeholder too, never a literal prefix
invariant non_negative:
  <root>.<field> >= 0
```

Both `<type>` and `<root>` are placeholders meaning "the root of the expression." Substitute them with the lowercase of your actual type name. The verifier accepts any identifier as the prefix, so a placeholder parses but matches no real type, and the pipeline silently skips the invariant.

**Referencing parent fields inside a collection body:**
```
# WRONG — <type>.<threshold> is a parent field, not an item field
invariant all_above_threshold:
  each(<type>.<items>, <itemField> >= <type>.<threshold>)

# WRONG — same issue with sum
invariant weighted_total:
  <type>.<result> == sum(<type>.<items>, <amount> * <type>.<rate>)
```

The body of `sum` and `each` can only reference item fields (bare names). There is currently no way to compare an item field against a record-level field inside a collection body. Decompose into separate invariants where possible. Note also that a few things the `.aral` rule syntax cannot express *are* expressible directly in the IR — for example conditional aggregation, `sum(coll, ite(pred, 1, 0))`. Since you author the IR by hand here, you can reach for those directly; see `ir-authoring.md`.

**Mixing `and`/`or` without collection context:**
```
# WRONG — ambiguous: is this (a and b) or c? or a and (b or c)?
invariant three_way:
  <type>.<a> > 0 and <type>.<b> > 0 or <type>.<c> > 0
```

Use one connective per invariant, or split into separate invariants.
