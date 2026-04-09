---
name: ephemaral-parser
description: Extract expressions from source code for ephemaral verification. Use this skill when the user wants to extract, parse, or convert code into .aral-fn.json for verification, or when preparing expressions for ephemaral. Also trigger when dealing with UNCONSTRAINED_PARAMETER diagnostics, coverage reports, or when assessing what can be extracted from a codebase.
license: MIT
metadata:
  author: andremiguelc
  version: "2026-04-08"
  repository: "https://github.com/andremiguelc/ephemaral-skills"
---

# ephemaral-parser — Expression-Level Extraction

Extracts expression trees from source code into `.aral-fn.json` files that ephemaral can verify. The `.aral` file drives extraction — it names types and fields, and the parser searches the codebase for assignments to those fields.

## How it works

1. Write `.aral` invariant files declaring types and fields to verify
2. Run the language parser against the codebase — it finds every assignment to those fields
3. For each assignment site, the parser extracts the expression tree into `.aral-fn.json`
4. Feed each `.aral-fn.json` + its `.aral` rules to ephemaral for verification

The pipeline is fully deterministic. The parser scans, extracts, and writes `.aral-fn.json` files automatically.

## The extraction spectrum

| Level | Meaning | What happens |
|---|---|---|
| **Fully extracted** | Every sub-expression maps to an Aral-fn IR variant | Full verification — deterministic and airtight |
| **Partially extracted** | Some sub-expressions are opaque | Opaque parts become UNCONSTRAINED_PARAMETER — verification runs with them as free variables |
| **No sites found** | No assignments to the declared fields found | Reported in coverage — check type names and field names match |

When the result is VERIFIED, it's a mathematical proof via Z3. The only softness is in extraction — and extraction is honest about what it couldn't parse.

## Coverage reporting

The parser prints a test-runner style report: per-field results with pass/gap markers, a summary line with counts, and the number of files scanned. When 0 sites are found, check the file count — a low number usually means the tsconfig doesn't cover the files you're targeting.

**In monorepos:** Different tsconfig files see different files — there's no single root tsconfig that sees everything. For each `.aral` file, run the parser against every relevant tsconfig to get full coverage. Deduplicate the results by file+line (different tsconfigs may find the same site). If any tsconfig run shows 0 sites with a low file count, that tsconfig simply doesn't cover the target code — move on to the next one.

## `.aral` file organization

One file per type in `.ephemaral/rules/`. Each `.aral` file contains everything that must always be true about that type. The parser uses it to know which types and fields to search for.

## Available language parsers

| Language | Parser | Reference |
|---|---|---|
| TypeScript | [ts-to-ephemaral](https://github.com/andremiguelc/ts-to-ephemaral) | `references/typescript.md` |

## Reference

- `references/aral-fn-schema.md` — the Aral-fn JSON schema (language-agnostic)
- `references/typescript.md` — TypeScript parser setup and expression mapping
