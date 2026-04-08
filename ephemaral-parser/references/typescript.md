# TypeScript Parser — ts-to-ephemaral

Expression-level extraction for TypeScript codebases using the TS compiler API.

## Prerequisites

- `node_modules` installed in the target project (TS compiler API needs it for type resolution)
- A `tsconfig.json` for the target project
- The [ts-to-ephemaral](https://github.com/andremiguelc/ts-to-ephemaral) parser installed

## Running extraction

```bash
npx tsx src/extract.ts .ephemaral/rules/<type>.aral --tsconfig path/to/tsconfig.json
```

The parser reads the `.aral` file to identify the target type and fields, scans every source file in the TS program, and writes `.aral-fn.json` files to `.ephemaral/parsed/<type>/`.

## What the parser needs

- **`.aral` file**: declares the type and fields to search for. The parser uses this to know what to look for.
- **`tsconfig.json`**: tells the TS compiler API which files to include and how to resolve types. In monorepos, different tsconfigs cover different packages — choose the one closest to the code you want to analyze.
- **Installed dependencies**: the TS compiler API resolves types through `node_modules`.

## What the parser finds

The parser looks for three assignment patterns:

1. **Object literal properties** — `{ fieldName: expr }` where the contextual type matches the target
2. **Direct property assignments** — `result.fieldName = expr` where the object's type matches
3. **ORM/Prisma patterns** — `orm.model.create/update/upsert({ data: { fieldName: expr } })`

Type matching is case-insensitive (`record` in `.aral` matches `Record` in TS).

## Reading the output

The parser prints a report with per-field results and a summary:

```
  Type (type.aral)

    field1
      ✓ module-service.ts:42 — containerFn
      ⚠ module-handler.ts:18 — otherFn  (__ext_functionName_0)

    field2
      ✓ module-service.ts:44 — containerFn

  2 passed, 1 with gaps
  Scanned 342 files via tsconfig.json
```

- **✓** — fully extracted, ready for verification
- **⚠** — some sub-expressions couldn't be extracted (named in the gap marker)
- **Scanned N files** — if this is low and 0 sites were found, the tsconfig probably doesn't cover the right files
