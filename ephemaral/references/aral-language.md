# Aral Language Reference

*Version: 2026-03-26*

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
  order.total == sum(order.lineItems, subtotal)

invariant total_with_tax:
  order.total == sum(order.lineItems, price * quantity) + order.tax
```

`sum(order.lineItems, subtotal)` reads as: "sum the `subtotal` field across all `lineItems`." Field names after the comma are implicitly item-scoped — no arrows, lambdas, or path repetition needed.

Sum can appear inside larger arithmetic expressions — it returns a numeric value like any other expression.

**Constraints:**
- Per-item expression supports field references and arithmetic (`price * quantity` ✓, conditionals ✗)
- All fields after the comma are item fields (no parent fields like `order.taxRate`)
- No nested sums (`sum(items, sum(subitems, x))` ✗)
- Collection must be a simple field reference (`order.lineItems` ✓, not `order.items.subitems`)

**How it works with functions:** Two scenarios:
- **Pass-through:** The function modifies scalar fields (like `total`) but doesn't touch the collection. The invariant catches broken scalar-collection relationships. The pipeline reuses input collection data for the output side.
- **Compute from collection:** The function computes `total = items.reduce((acc, i) => acc + i.subtotal, 0)`. The parser emits a `sum` expression in the function JSON. The verifier checks that the computed value matches the invariant's sum.

## What the language cannot express

These are deliberate scope boundaries:

- **No variables or assignments** — invariants describe data, not computation
- **No loops** — `sum` handles aggregation; `each`, `count`, `filter` are not yet supported
- **No function calls** — invariants reference types, not code
- **No string operations** — beyond `==` and `!=`
- **No cross-entity constraints** — invariants describe one record at a time
- **No temporal properties** — no time-dependent constraints
- **No explicit presence guards in `.aral` syntax** — optional field handling works at the pipeline level via `optionalFields`. Writing `if root has field: ...` directly in `.aral` expressions is not supported.
- **No collection operations beyond `sum`** — `each`, `count`, `filter` are not yet supported
