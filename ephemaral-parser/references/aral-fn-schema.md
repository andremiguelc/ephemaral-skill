# Aral-fn JSON Schema Reference

The `.aral-fn.json` format describes a function's behavior for ephemaral verification. This is the contract between any parser (or hand-crafted JSON) and the verifier.

## Top-level fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Function name (for diagnostics) |
| `inputType` | string | yes | The type being transformed (matched against `.aral` type prefixes) |
| `inputFields` | string[] | yes | Fields on the type relevant to verification |
| `params` | string[] | yes | Extra parameters beyond the input object (free variables) |
| `assigns` | FieldAssign[] | yes | Which fields change and to what expression |
| `typedParams` | Array<{name, type}> | no | Parameters with non-primitive types (enables parameter preconditions) |
| `optionalFields` | string[] | no | Fields that may be absent. For each, the pipeline generates a presence flag (`has-{field}`) constrained to 0 or 1. Invariants on optional fields are auto-guarded by presence. |

## FieldAssign

| Field | Type | Description |
|-------|------|-------------|
| `fieldName` | string | Which output field is being set |
| `value` | Expr | Expression computing the new value |

## Expr (expression tree)

Expressions are recursive. Seven variants:

| Variant | JSON shape | Description |
|---------|-----------|-------------|
| Literal | `{"lit": 42}` | Numeric constant |
| Field (simple) | `{"field": {"name": "total"}}` | Field on the input type or a scalar parameter |
| Field (qualified) | `{"field": {"qualifier": "param", "name": "field"}}` | Field on a typed parameter |
| Arithmetic | `{"arith": {"op": "add", "left": <Expr>, "right": <Expr>}}` | Operators: `add`, `sub`, `mul`, `div` |
| If-then-else | `{"ite": {"cond": <BoolExpr>, "then": <Expr>, "else": <Expr>}}` | Conditional |
| Rounding | `{"round": {"expr": <Expr>, "mode": "floor"}}` | Modes: `floor`, `ceil`, `half_up` |
| Sum | `{"sum": {"collection": "lineItems", "body": <Expr>}}` | Sum over collection items. Body is per-item. |

## BoolExpr (boolean expression tree)

Used as conditions in `ite`. Five variants:

| Variant | JSON shape | Description |
|---------|-----------|-------------|
| Comparison | `{"cmp": {"op": "gte", "left": <Expr>, "right": <Expr>}}` | Operators: `eq`, `neq`, `gt`, `lt`, `gte`, `lte` |
| Logic | `{"logic": {"op": "and", "left": <BoolExpr>, "right": <BoolExpr>}}` | Operators: `and`, `or` |
| Negation | `{"not": <BoolExpr>}` | Boolean not |
| Presence | `{"isPresent": {"name": "field"}}` | True if the optional field has a value. Supports qualifier: `{"isPresent": {"qualifier": "param", "name": "field"}}` |
| Each | `{"each": {"collection": "items", "body": <BoolExpr>}}` | Universal quantifier — body must hold for every item. Body is a per-item BoolExpr; field refs inside are item-scoped. |

## How fields resolve

- **Simple field** `{"field": {"name": "balance"}}` — if the name is in `inputFields`, it refers to the input value. Otherwise, it refers to a parameter.
- **Qualified field** `{"field": {"qualifier": "param", "name": "field"}}` — a field on a typed parameter. The verifier joins them (`param-field`) as an SMT constant.

## typedParams

When a parameter has a non-primitive type, include it in `typedParams`:

```json
"typedParams": [{"name": "paramName", "type": "TypeName"}]
```

The parameter name must also appear in `params`. The verifier matches the type name against `.aral` file type prefixes to apply invariants as preconditions on that parameter.

## optionalFields

When a field may be absent, include it in `optionalFields`:

```json
"optionalFields": ["fieldA", "fieldB"]
```

For each optional field `f`, the pipeline:
1. Generates a presence flag `has-f` constrained to 0 or 1
2. Guards any invariant referencing `f` with `(=> has-f ...)` — meaning the invariant only applies when the field is present
3. Annotates counterexample output: `has-f = 0 (f was not provided)`

No special syntax is needed in `.aral` files — write invariants as if the field always exists, the pipeline handles presence automatically.

## Division semantics

Division is total: `a / 0 = 0`. The verifier surfaces "your divisor can be zero" as a diagnostic rather than requiring a precondition.

## What fields to include in inputFields

Include all fields that are:
- Assigned in `assigns`
- Referenced in assignment expressions (including inside `ite` conditions)
- Referenced by applicable invariants

Fields not in `assigns` are preserved by spread — the verifier knows the output equals the input for those fields.

## Collections and sum

The `sum` expression represents a function computing a total from a collection:

```json
{"sum": {"collection": "items", "body": {"field": {"name": "amount"}}}}
```

- `collection` is a plain string — the collection field name on the input type
- `body` is a per-item Expr — field refs inside are item-scoped (e.g., `amount`, `unitValue`)
- Per-item body supports field refs, arithmetic, and scalar `ite` conditionals — e.g. `{"sum": {"collection": "items", "body": {"ite": {"cond": <BoolExpr>, "then": <Expr>, "else": <Expr>}}}}` for `SUM(CASE WHEN ...)`-style aggregation. No nested `sum`/`each` inside the body, no parent field refs.

For **pass-through collections** (function doesn't modify the collection, just reads from it or touches scalar fields), no special JSON is needed — just declare the scalar fields in `inputFields` and `assigns`. The pipeline detects collections from the `.aral` invariant and generates the appropriate SMT encoding.
