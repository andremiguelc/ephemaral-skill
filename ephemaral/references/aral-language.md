# Aral Language Reference

*Version: 2026-04-04*

Aral is a constraint language for describing what must always be true about data. Each construct below is formally verified — its compilation has been proved correct.

## Primitives

### Field reference

A dot-separated path to a field on a typed object.

```
root.fieldName
```

The root is the type binding (matched case-insensitively to the function's input type). The field is a field on that type. The root is stripped during compilation — only the field name matters for verification.

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

1. **Record scope** — `root.field` references. Available everywhere. These are scalar fields on the record being constrained.
2. **Item scope** — bare field names inside `sum()` and `each()` bodies. These reference fields on the current collection item. Only available inside collection body expressions.

**The two scopes do not mix inside a collection body.** A `sum` or `each` body can only reference item fields (bare names). It cannot reference parent record fields (`root.field`). This is a hard constraint of the language — the body is evaluated per-item in isolation.

To constrain relationships between item fields and record-level fields, use separate invariants: one `each`/`sum` for the per-item property, and a separate scalar invariant for the record-level relationship.

## Constructs

### Field comparison

Compare an expression to another expression.

```
invariant non_negative:
  root.fieldA >= 0

invariant fields_equal:
  root.fieldA == root.fieldB
```

Both sides can be field references, literals, or arithmetic expressions.

### Arithmetic

Binary operations between expressions: `+`, `-`, `*`, `/`.

```
invariant difference_non_negative:
  root.fieldA - root.fieldB >= 0

invariant product_non_negative:
  root.fieldA * root.fieldB >= 0
```

Division is total: dividing by zero equals zero. The tool surfaces "your divisor can be zero" as a diagnostic rather than requiring a precondition.

Multi-term arithmetic with correct precedence is supported. `*`/`/` bind tighter than `+`/`-`, and expressions are left-associative.

```
invariant sum_of_parts:
  root.whole == root.partA + root.partB - root.adjustment

invariant mixed_precedence:
  root.result == root.base * root.rate + root.offset
```

### Boolean connectives

Combine comparisons with `and` or `or`.

```
invariant bounded:
  root.value >= 0 and root.value <= 1000

invariant at_least_one_positive:
  root.fieldA > 0 or root.fieldB > 0
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

### Optional fields

When a function declares fields as optional (via `optionalFields` in the JSON), invariants referencing those fields are automatically guarded by presence. Write invariants as if the field always exists — the pipeline adds the guard.

For example, if `adjustment` is optional:
```
invariant adjustment_non_negative:
  root.adjustment >= 0
```

This is interpreted as: "when adjustment is present, it must be non-negative." No special syntax needed.

### sum — Collection Aggregation

Sum a per-item expression across all items in a collection.

```
sum(<root>.<collection>, <per-item expression>)
```

```
invariant total_matches_items:
  root.total == sum(root.items, amount)

invariant total_with_factor:
  root.total == sum(root.items, unitValue * quantity) + root.offset
```

`sum(root.items, amount)` reads as: "sum the `amount` field across all `items`." Field names after the comma are implicitly item-scoped — no arrows, lambdas, or path repetition needed.

Sum can appear inside larger arithmetic expressions — it returns a numeric value like any other expression.

**Constraints:**
- Body is item-scoped only — bare field names reference item fields (see Scoping rules above)
- Per-item expression supports field references and arithmetic (`unitValue * quantity` ✓, conditionals ✗)
- No nested sums (`sum(items, sum(subitems, x))` ✗)
- Collection must be a simple field reference (`root.items` ✓, not `root.nested.items`)

**How it works with functions:** Two scenarios:
- **Pass-through:** The function modifies scalar fields but doesn't touch the collection. The invariant catches broken scalar-collection relationships. The pipeline reuses input collection data for the output side.
- **Compute from collection:** The function computes a total by reducing over items. The parser emits a `sum` expression in the function JSON. The verifier checks that the computed value matches the invariant's sum.

### each — Universal Quantifier over Collections

Assert that a boolean predicate holds for every item in a collection.

```
each(<root>.<collection>, <per-item predicate>)
```

```
invariant items_positive:
  each(root.items, value > 0)

invariant items_bounded:
  each(root.items, value > 0 and value <= 1000)
```

`each(root.items, value > 0)` reads as: "for every item in `items`, `value` is greater than zero." Field names after the comma are item-scoped, just like in `sum`.

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
- Collection must be a simple field reference (`root.items` ✓)
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
- **No explicit presence guards in `.aral` syntax** — optional field handling works at the pipeline level via `optionalFields`. Writing `if root has field: ...` directly in `.aral` expressions is not supported.
- **No parent field references inside collection bodies** — `sum` and `each` bodies can only use bare item field names, not `root.field`. To compare item fields against record-level values, the record-level constraint must be a separate invariant outside the collection body.
- **No `filter` or conditional aggregation** — `sum(coll, ite(pred, 1, 0))` is expressible in the IR but not yet in `.aral` syntax

## Common mistakes

These patterns look reasonable but will be silently skipped by the parser. Always check for `[skipped]` in output.

**Referencing parent fields inside a collection body:**
```
# WRONG — root.threshold is a parent field, not an item field
invariant all_above_threshold:
  each(root.items, value >= root.threshold)

# WRONG — same issue with sum
invariant weighted_total:
  root.result == sum(root.items, amount * root.rate)
```

The body of `sum` and `each` can only reference item fields (bare names). There is currently no way to compare an item field against a record-level field inside a collection body. Decompose into separate invariants where possible, or hand-craft the `.aral-fn.json` IR directly for cases the `.aral` syntax cannot express.

**Mixing `and`/`or` without collection context:**
```
# WRONG — ambiguous: is this (a and b) or c? or a and (b or c)?
invariant three_way:
  root.a > 0 and root.b > 0 or root.c > 0
```

Use one connective per invariant, or split into separate invariants.
