---
name: ephemaral-parser
description: Parse source code functions into a format ephemaral can verify. Use this skill when the user wants to parse, extract, or convert a function for verification, or when preparing a function for ephemaral. Also trigger when dealing with parser errors (REWRITE, NOT VERIFIABLE), when assessing whether a function is verifiable, or when working with .aral-fn.json files.
license: Apache-2.0
compatibility: Optional deterministic parser (e.g. ts-to-ephemaral for TypeScript)
metadata:
  author: andremiguelc
  version: "0.1.0"
  repository: "https://github.com/andremiguelc/ephemaral-skills"
---

# ephemaral-parser — Source Code to Verification JSON

Converts function declarations into `.aral-fn.json` files that ephemaral can verify. Parsers sit outside the verification boundary — they read source code and produce a structured representation. Any language can have a parser; all parsers produce the same JSON format.

## Role of the parser

The parser is a **trust mechanism**. When a deterministic parser succeeds, the `.aral-fn.json` is mechanically correct — no human judgment involved, no risk of misreading the source. When it fails, it tells you exactly why.

The parser is NOT a gatekeeper. A function that can't be parsed can still be verified — the `.aral-fn.json` can be hand-crafted by reading the source and writing the JSON directly. When the parser rejects a function, that's information about the parser's coverage, not about what ephemaral can verify. Always distinguish:

- **Parser limitation** — the function's code shape doesn't fit the parser. The underlying logic might still be expressible in the IR.
- **IR limitation** — the function's logic genuinely can't be represented (collections, side effects). No amount of rewriting will help.
- **Invariant limitation** — the function can be represented, but the rules you need can't be expressed in the `.aral` language yet.

## Finding a parser

Parsers are discovered via a `parser.json` manifest at the root of the parser's directory:

```json
{
  "language": "typescript",
  "extensions": [".ts"],
  "invoke": "npx tsx src/index.ts {file}",
  "schemaVersion": "0.1.0"
}
```

- `language` — what source language this parser handles
- `extensions` — file extensions it can parse
- `invoke` — command template; replace `{file}` with the source path
- `schemaVersion` — which Aral-fn schema version the output conforms to

Check if a parser is available for the source language. If not, skip to "When no parser is available" below.

## Running the parser

Replace `{file}` in the `invoke` command with the source file path. Output goes to stdout as JSON:

```bash
<invoke-command> path/to/function.ext > .ephemaral/parsed/function.aral-fn.json
```

## One function per file

The parser expects exactly one function declaration per source file. In real codebases, source files contain multiple functions, exports, or utility code.

Extract the target function into a temporary file before parsing:

1. Identify the function you want to verify
2. Copy just that function (plus any type imports it needs) into a temp file
3. Run the parser on the temp file
4. The parsed `.aral-fn.json` is what ephemaral reads

Use a consistent temp location (e.g., `.ephemaral/parsed/sources/`). Name the temp file after the function.

## Preparing real-world functions

Real codebases rarely have functions in the shape the parser expects. Preparing a function for parsing is the bulk of the work. Common transformations:

### Uncurrying

Higher-order / curried functions must be flattened. A function that returns a function should be rewritten as a single function taking all arguments.

### Method calls to arithmetic operators

If a library routes arithmetic through method calls, replace with plain operators (`a + b`). This is a semantic equivalence for standard arithmetic — document any cases where the method has non-standard behavior.

### Opaque computations to parameters

When a function calls another function whose result can't be expressed in the IR, extract the result as a parameter. The verifier treats it as unconstrained — it will flag `UNCONSTRAINED_PARAMETER`, which is correct: the verifier can't know the range of a value it can't compute.

### Typed parameter dot-access

When a parameter has a non-primitive type, you can access its fields directly with one level of dot-access: `param.field`. The parser produces a qualified field reference and emits `typedParams` in the JSON output, enabling the verifier to apply invariants from a separate `.aral` file as preconditions on the parameter.

Deeper nesting (`param.nested.field`) is not supported — flatten those cases.

### Nested type flattening

Two-level field access (`input.nested.field`) must be flattened to a top-level field. One-level param access (`param.field`) works, but deeper paths do not. When flattening:

- Flatten to an **input field** (part of the type) if the value carries invariants you want preserved. Input fields get invariant coverage automatically.
- Flatten to a **parameter** (extra argument) if the value comes from a different object or computation. Parameters have no invariant coverage — they're free variables.

This distinction matters: the same value verifies or fails depending on where you put it.

## Nullable field handling

Functions often access fields that might be absent. Parsers recognize null-coalescing patterns in the source language and lower them to conditional presence checks in the IR.

The general pattern: a null-coalescing expression becomes:
```
ite(isPresent(field), field, default)
```

This means: "if the field is present, use it; otherwise use the default."

**How it works:**
- Any field used in a null-coalescing expression is automatically added to the `optionalFields` array in the JSON output
- `optionalFields` tells the verification pipeline which fields are nullable — it generates presence flags (`has-{field}`) and guards invariants accordingly
- Chained coalescing nests: outer check for the first field, inner check for the second, innermost is the fallback literal
- Qualified nullable access on typed parameters also works: `param.field` with a default produces `isPresent` with both `qualifier` and `name`

See `references/aral-fn-schema.md` for the exact JSON shapes of `isPresent` and `optionalFields`.

## Modeling decisions

Each simplification above is a modeling decision **outside the proof boundary**. The verification proves properties of the *model*, not the original code. The model's fidelity is your responsibility.

What's lost in common transformations:
- Replacing a method call with a parameter — loses the arithmetic relationship the method encodes
- Flattening nested field access — loses the nested structure; invariants must use the flat field name
- Extracting a function call as a parameter — the verifier treats it as unconstrained (correctly — it can't compute the function)

The verification result is trustworthy for the model. Whether the model faithfully represents the original code is a separate judgment.

## What functions are parseable

A function is parseable when it:

1. **Takes and returns the same type** — the function transforms a record and returns an updated version
2. **Uses spread-and-override return** — unchanged fields are preserved, only specific fields are reassigned
3. **Operates on numeric fields** — arithmetic, comparisons, rounding
4. **Has explicit type annotations** — both parameters and return type must be declared

Extra parameters beyond the input become free variables in verification. When a parameter has a non-primitive type, the parser emits a `typedParams` entry so the verifier can apply invariants from a matching `.aral` file as preconditions.

### Supported patterns in function bodies

- **Simple assignment**: spread the input, override specific fields
- **Guard clauses**: early returns that preserve the input unchanged — converted to conditional expressions
- **Chained guards**: multiple guard-return statements before the main return
- **Immutable bindings**: intermediate values declared as constants — automatically inlined
- **Conditional expressions**: branching on a boolean condition
- **Arithmetic with precedence**: multiplication/division bind tighter than addition/subtraction
- **Rounding**: floor, ceil, half-up (language-specific syntax varies)
- **Comparisons and boolean logic**: standard comparison operators, and/or/not
- **Unary minus**: negation of a value
- **Parenthesized expressions**: explicit grouping
- **Null coalescing**: field-or-default patterns — lowered to conditional presence check
- **Chained coalescing**: nested field-or-default patterns

## What functions are NOT parseable

### Patterns that need rewriting (REWRITE errors)

The parser explains what to change. These are structural issues with how the function is written:

- **Non-declaration forms** — lambdas, closures, methods need rewriting as standalone function declarations
- **Mutable variables** — use immutable bindings
- **Exception throwing** — replace with guard-return pattern (if invalid, return input unchanged)
- **Missing return type** — add explicit type annotation
- **Wrong spread variable** — the spread must use the first parameter name
- **Nested field access** — `input.nested.field` is not supported; flatten to an input field or parameter

### Patterns that cannot be verified (NOT VERIFIABLE errors)

These are fundamental limitations of the verification approach:

- **Collection operations** — reduce, map, filter, aggregate, iterate
- **Index operations** — bracket access, array indexing
- **Boolean-returning functions** — function must return the same type as its input

## Reading parser errors

Errors have two prefixes:

**`REWRITE:`** — The function's structure doesn't fit the supported pattern, but it could be restructured. The error message shows what to change and how.

**`NOT VERIFIABLE:`** — The function uses a pattern that fundamentally cannot be represented in the verification IR. The error explains why and suggests a workaround (usually: extract the unsupported part and pass the result as a parameter).

## When no parser is available

When there's no deterministic parser for the source language, you have two options:

1. **Hand-craft the `.aral-fn.json`** by reading the source code and writing the JSON directly. See `references/aral-fn-schema.md` inside this skill folder for the complete schema. This bypasses the parser entirely — the verification works on whatever valid JSON you provide.
2. **Record the limitation** and move on. Not every function needs to be verified right now. Documenting what blocked you (parser shape? IR construct? invariant expressiveness?) is valuable.

When you hand-craft JSON, note that you are making modeling decisions: which fields to include, how to represent the arithmetic, what to simplify. The verification proves the model, not the original code. Document what you simplified and why.

## When parsing fails

When the parser can't handle a function, you have three options:

1. **Rewrite the function** to fit the parser's supported shape (if the error is REWRITE). This is the fastest path when the rewrite is small.
2. **Hand-craft the `.aral-fn.json`** as described above. This bypasses the parser entirely.
3. **Record the limitation** and move on.