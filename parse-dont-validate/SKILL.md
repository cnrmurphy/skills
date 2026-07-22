---
name: parse-dont-validate
description: Apply the "parse, don't validate" type-driven design principle when writing, reviewing, or refactoring code that handles external or untrusted data. Use this skill whenever code touches a trust boundary (API request handlers, env vars, config files, database reads, file parsing, LLM/tool output, user input), whenever the user mentions validation, input checking, defensive programming, branded types, or invariants, and whenever you notice repeated defensive checks (length/null/format checks), non-null assertions, casts to silence "impossible" cases, or validate-style functions returning void/boolean. Also use it when asked to review code for robustness or to clean up scattered validation logic.
---

# Parse, don't validate

Convert data to a precise domain type **once**, at the trust boundary. Downstream
code receives the precise type and never re-checks it. A check that returns
`void`/`boolean` throws away what it learned; a check that returns a refined type
preserves it in the type system, where the compiler enforces it for everyone.

Why this matters: in validation-based code, no signature records that a check
happened, so every function defensively re-checks — producing scattered checks,
untested "impossible" else-branches, and latent bugs when an upstream check is
removed. Parsing turns "remember to check" into "can't compile without checking."

## Core rules

1. **Checks return the refined type, never void/boolean.**
   `parseEmail(s: string): Email | null` — not `validateEmail(s: string): boolean`.
   If a check's result can be ignored, it will eventually be forgotten.

2. **Refined types get exactly one door.** Construct branded/wrapper types only
   through a single smart constructor that performs the check (private
   constructor, or a `parse*`/`from*` factory in the type's own module). Never
   cast into a refined type outside that module. In structurally-typed languages
   (TypeScript), add a private field or brand so outside objects cannot
   impersonate the type.

3. **Inner functions demand proof via their signature.** A function requiring an
   invariant (non-empty, positive, sorted, authenticated) takes the refined type
   as its parameter. Do not accept the loose type and check inside.

4. **Prefer types that make illegal states unrepresentable over checks that
   detect them.**
   - Duplicate keys forbidden → take a `Map`, not a list of pairs
   - Mutually exclusive fields → discriminated union, not optional fields + flag
   - At least one element → non-empty type (`[T, ...T[]]`), not array + length check
   - Value duplicated in two places → single source of truth, derive the other

5. **All parse failures surface at the boundary, before any state changes.**
   Never partially process input and discover invalidity midway ("shotgun
   parsing") — that leaves the system in a half-mutated state that is hard to
   roll back. Parse fully, then act.

## Smells — stop and refactor if about to write:

- a re-check (`length === 0`, null check, format check) on a value already
  checked upstream
- `value!`, `as T`, `.unwrap()`, or `?? default` to silence a case that
  "can't happen"
- a comment like `// safe: validated earlier` or `error("impossible")`
- a `validate*`/`check*`/`assert*` function returning `void` or `boolean`
- the same field-check appearing in more than one function

Each smell means an invariant exists but lives outside the type system. Fix:
move the check into a parser at the boundary and thread the refined type through
the call chain. Change the innermost function's signature first, then let
compiler errors walk you up to the boundary where the parse belongs.

## Use the codebase's existing tools

Before hand-rolling a wrapper type, check what the project already uses and
prefer it:
- **Zod** → `.brand()`, `.nonempty()`, refinements; parse with `safeParse` at boundaries
- **Effect** → `Schema` for boundary decoding, `Schema.brand`, `Array.NonEmptyArray`
- **io-ts / runtypes / valibot / typia** → their codec/refinement equivalents
- **Rust** → newtype structs with private fields + `TryFrom`; this is idiomatic
- **Python** → Pydantic models at boundaries; `NewType` + factory functions internally
- **Java/Kotlin** → value classes / records with private constructors + static factories

Match the established pattern; do not introduce a second branding mechanism into
a codebase that has one.

## Scope — when NOT to apply

- **Values checked and consumed within a few lines**: a local check is fine;
  wrapping it is ceremony. Refined types pay off for domain concepts that
  travel across multiple functions (IDs, emails, money, quantities, parsed
  config, auth state).
- **Legacy code, small diffs**: introduce the refined type at the seam being
  edited rather than refactoring the whole call chain. If the full refactor is
  out of scope, keep the local check and document the invariant at the check
  site.
- **Do not mass-refactor unprompted.** When writing new code, apply the pattern.
  When editing existing code, apply it to the code being touched and *mention*
  wider opportunities to the user rather than rewriting files they didn't ask
  about.
- **Serialization boundaries**: refined types may need explicit encode/decode at
  JSON/DB edges; keep that logic next to the smart constructor.

## Example (TypeScript)

```typescript
// BEFORE — validation: knowledge thrown away, callers re-check
function validateDirs(dirs: string[]): void {
  if (dirs.length === 0) throw new Error("empty");
}

// AFTER — parsing: knowledge kept in the type
type NonEmpty<T> = readonly [T, ...T[]];

function parseNonEmpty<T>(arr: readonly T[]): NonEmpty<T> | null {
  return arr.length > 0 ? (arr as NonEmpty<T>) : null;
}

function getConfigDirs(): NonEmpty<string> {
  const parsed = parseNonEmpty(env.CONFIG_DIRS.split(","));
  if (!parsed) throw new Error("CONFIG_DIRS cannot be empty");
  return parsed;
}

// Downstream demands proof via its signature — cannot be called unsafely,
// and if the upstream check is ever removed, this fails to COMPILE.
function initializeCache(dirs: NonEmpty<string>): void {
  const primary = dirs[0]; // type: string, not string | undefined
}
```

The same shape applies to any invariant: check once in a parser, return a type
that carries the proof, require that type downstream.
