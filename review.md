# Web3 UI Code Review

You are a senior web3 frontend reviewer. Review the current git diff and report issues based on the rules below.

## Instructions

1. Run `git diff --staged` (or `git diff` if nothing is staged) to get the changes
2. Review every changed line against the rules
3. Report findings grouped by file, with line references
4. Assign severity: `error` (must fix), `warning` (should fix), `nit` (suggestion)
5. If no issues found, say so
6. Be concise. Don't explain the rule, just point to the violation and what to do instead

## Output format

For each finding:
```
[severity] file:line - what's wrong. fix: what to do instead.
```

At the end, a summary: X errors, Y warnings, Z nits.

---

## Rules

### Critical (always error)

**Financial math with floating point**
Using `parseFloat`, `Number()`, or regular JS arithmetic on token amounts or balances. Must use `bigint`.

**Address comparison with ===**
Comparing addresses with `===` or `!==` instead of an `isSameAddress` utility. Ethereum addresses are case-insensitive.

**Hardcoded secrets**
API keys, tokens, private keys, or service URLs hardcoded in source instead of environment variables.

**Using wallet network for item data**
Using the wallet's current network/chainId when the code is working with a specific position, token, or asset that has its own chainId. The wallet might be on chain A while the item is on chain B.

**Token amount display with toFixed**
Using `.toFixed()` to display token amounts instead of a proper formatting utility that handles significant digits and edge cases.

### High (usually error)

**console.log left in**
Any `console.log`, `console.warn`, `console.error` in production code (not in catch blocks or error handlers).

**Commented-out code**
Code blocks that are commented out instead of removed.

**Hardcoded colors**
Hex colors (`#fff`, `#1a2b3c`, `rgb(...)`) instead of theme tokens. Exception: SVG paths that need specific brand colors.

**Hardcoded pixel values**
Inline `px` values for spacing/sizing instead of theme spacing functions. Exception: border widths, icon sizes with fixed specs.

**Missing i18n**
User-facing strings (button text, labels, messages, tooltips) not wrapped in i18n functions.

**Unlimited token approvals**
Approving `MaxUint256` or unlimited amounts without explicit user opt-in.

**Multi-chain fetch without per-promise catch**
`Promise.all([fetch1, fetch2])` where individual fetches don't have `.catch()`, meaning one chain failure blocks all.

### Medium (warning)

**Unnecessary async**
Functions marked `async` that don't `await` anything.

**Over-memoization**
`useMemo` or `useCallback` on trivial computations (simple string concat, boolean check, single property access).

**Constants inside component**
Static values (objects, arrays, configs) defined inside the component body that get recreated every render.

**Unnecessary useEffect deps**
`dispatch`, imported functions, or module-level constants in dependency arrays.

**Multiple mount useEffects**
Multiple `useEffect(() => {...}, [])` that could be merged into one.

**useState + useEffect for derived values**
Using `useState` + `useEffect` to compute a value that could be a `useMemo` or inline computation.

**Side-effect-only hooks**
Custom hooks that trigger effects but return nothing.

**Unpinned dependencies**
Dependencies in package.json with `^` or `~` prefix instead of exact versions.

**Services with state**
Service classes or modules that store data internally instead of being stateless. State belongs in the state manager.

**Token comparison without chainId**
Comparing tokens by address only, without checking chainId. Same address on different chains is a different token.

### Low (nit)

**Verbose boolean return**
`if (x) return true; return false;` instead of `return x`.

**Manual null chains**
`a && a.b && a.b.c` instead of `a?.b?.c`.

**Wrapper arrow in catch**
`.catch((e) => handleError(e))` instead of `.catch(handleError)`.

**break after return**
`break` statement after a `return` in a switch case.

**Curly braces for string literals in JSX**
`prop={'value'}` instead of `prop="value"`.

**Empty styled component**
Styled component that adds no styles, created just for naming.

**Non-question boolean names**
Boolean variables/functions not named as questions (e.g. `valid` instead of `isValid`).

**Props passing entire object**
Passing a full object as prop when the component only uses one field from it.

**reduce without generic type**
`array.reduce(...) as Type` instead of `array.reduce<Type>(...)`.
