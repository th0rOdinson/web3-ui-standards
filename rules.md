# Web3 UI Development Standards

Standards based on years of building production DeFi applications. These rules represent patterns learned from real-world web3 frontend development, covering the mistakes that keep happening and the conventions that make code maintainable at scale.

## How to use

Drop this file into the root of your web3 frontend project. Claude Code loads it automatically on every session, giving you senior-level code review and guidance as you develop.

---

## Code Quality

### Clean Code
- Never leave commented-out code. Remove it or keep it active.
- Remove all `console.log` before merging.
- Fix all typos in code, copy, and variable names before merge.
- Remove unused dependencies, imports, and dead code.
- User-facing messages must end with a period and have correct grammar.

### JavaScript/TypeScript Style
- Prefer `?.` (optional chaining) and `??` (nullish coalescing) over manual null checks and `&&` chains.
- Return boolean expressions directly: `return x > 5` not `if (x > 5) return true; return false;`.
- Prefer TypeScript enums over string union types for fixed sets. Use ALL_CAPS for enum entries.
- Use object destructuring for state and function return values.
- Use `reduce<Type>()` generic parameter instead of `as Type` assertion at the end.
- Don't use `async` on functions that don't `await` anything.
- Remove `break` after `return` in switch cases.
- Use `.catch(handleError)` shorthand, not `.catch((error) => handleError(error))`.
- Use `eslint-disable-next-line` for single-line overrides, not `eslint-disable`/`eslint-enable` blocks.
- String literals in JSX: use `maxWidth="16ch"` not `maxWidth={'16ch'}`.
- Use existing constants (e.g. `LATEST_VERSION`) instead of computing from arrays.

### Patterns
- Abstract repeated logic into utility functions. If you see the same calculation or check twice, extract it.
- Prefer `map` + `lodash/compact` over `reduce` for filter-and-transform operations.
- Use `some()` instead of `forEach` + throw for validation checks.
- Query only fields you actually use from APIs/GraphQL.
- When adding items to blacklists, include a comment explaining why.
- Use `toLocaleString` / `DateTime.DATE_MED` for date formatting, not hardcoded format strings.

### Naming
- Names must be descriptive and accurate. Boolean functions should be named as questions (e.g. `doesCompanionNeedWithdrawPermission`).
- Props should be minimal and focused. Don't pass entire objects when only one field is needed.

---

## Web3 Patterns

### Address Handling
- **Never compare addresses with `===`.** Use an `isSameAddress` utility that handles case normalization. Ethereum addresses are case-insensitive.
- All addresses should be lowercased throughout the codebase.
- Token address validation: check format with `/^0x[A-Fa-f0-9]{40}$/` before custom token lookups.
- ENS names should be resolved lazily with caching.

### Token Amounts and Financial Math
- **Use `bigint` for all token balances and financial calculations.** Never use `parseFloat` or regular JS numbers for amounts.
- Use `parseUnits`/`formatUnits` (from viem or ethers) for converting between human-readable and on-chain amounts.
- Use a `formatCurrencyAmount` utility for displaying token amounts, never `.toFixed()`.
- USD prices should be stored as `bigint` with 18 decimals precision to avoid floating-point loss.
- APY is stored as a percentage (8 means 8%). Calculations must divide by 100.
- Fee calculations use BigInt math: `(amount * feeBigInt) / 100000n`.
- Use a consistent amount representation: `{ amount: bigint, amountInUnits: string, amountInUSD: string }`.

### Token Handling
- Native tokens use a sentinel address (`0xeeee...eeee`). Use a `getProtocolToken(chainId)` helper.
- Handle native/wrapped token duality: users interact with ETH but contracts use WETH.
- Token comparison must check BOTH address AND chainId. Same address on different chains is a different token.
- Use a `TokenListId` format of `${chainId}-${tokenAddress}` for unique identification across chains.
- Curated token lists filter the picker to vetted tokens. Custom tokens require explicit import by pasting address.
- Token blacklists prevent interaction with deprecated or dangerous tokens.

### Transaction Lifecycle
- Multi-step flows: build steps dynamically based on current state (approve -> sign -> execute).
- Gas estimation should include a ~30% buffer: `gasLimit = (gasUsed * 130n) / 100n`.
- User-cancelled transactions (error code 4001, ACTION_REJECTED) are NOT errors. Don't track them.
- Transaction retry counts should be chain-specific (some chains need more retries due to longer confirmation times).
- Each transaction type should have a typed data structure for post-transaction state updates.
- Optimistic updates bridge the gap between tx submission and indexer catching up. Place revertible operations inside the try block.
- Transaction modals use a 4-state pattern: success, loading, error, closed.

### Multi-Chain
- When working with position/asset data, always use the item's own `chainId`, never the wallet's current network. The wallet might be on Ethereum while the position lives on Arbitrum. Using the wallet's network breaks token resolution, explorer links, and contract calls.
- Multi-chain fetches: use `Promise.all` with per-promise `.catch()` so one chain's failure doesn't block others.
- Per-chain configuration (RPCs, contract addresses, supported features) should be explicit and typed.
- Block-based sorting is preferred over timestamp sorting for on-chain history.

### Approvals
- Default to specific (exact amount) token approvals over unlimited for security.
- Support Permit2 as a gasless approval alternative where available.
- Use `encodeFunctionData` over `populateTransaction` for building transaction data (synchronous, simpler).
- Pin all dependency versions exactly (no `^` or `~` prefix). Unexpected updates cause hard-to-debug issues, especially with pre-1.0 packages where patch can mean breaking changes.

### Wallet Integration
- Handle different wallet behaviors: Rabby signs differently than MetaMask, Safe batches transactions.
- When integrating with embedded wallet environments (Safe, iframe dApps), add timeouts to detection checks to prevent indefinite hanging.

---

## React Patterns

### Memoization
- Memoize values and callbacks passed to child components with `useMemo`/`useCallback`. React checks by reference.
- Don't over-memoize simple computations where the overhead exceeds the computation cost.
- Move constant computations (values not depending on props/state) outside the component body entirely.

### Hooks
- Only include truly necessary dependencies in `useEffect`/`useCallback`/`useMemo` arrays. Exclude stable references like `dispatch`, imported functions, or constants.
- Merge multiple mount-only `useEffect`s (same deps) into a single one.
- Custom hooks that only trigger side effects with no return value are discouraged. Prefer hooks that return data.
- Prefer derived/computed values via `useMemo` over `useState` + `useEffect` chains.

### State Management
- All application state lives in the state manager (Zustand, Redux, or similar). Services are stateless.
- If using Redux, use async thunk `.pending`/`.fulfilled`/`.rejected` matchers, not manual loading state actions.
- Don't spread previous state when resetting after errors if you want fields to become undefined.

### UI Patterns
- Show loading indicators only when there is NO existing data. If data exists and is refreshing, show the stale data without a loading indicator.
- Components that conditionally render nothing should return `null`, not `<></>`.
- CTA buttons follow a priority chain: no wallet -> wrong network -> no funds -> needs approval -> ready to execute.
- Lazy-load all page components with `React.lazy()` and `Suspense`.
- Fetch data proactively (during login, on app load) so users don't see loading states when navigating.

---

## Theming and Styling

- Never use hardcoded hex colors. Access colors through theme tokens that support dark/light mode.
- Always use theme spacing functions instead of hardcoded pixel values. Hardcoded widths in px are "extremely discouraged."
- SVG icon assets should not have hardcoded `fill` colors. Remove fills so they inherit from the theme.
- Use the `$` prefix for transient styled-component props (e.g. `$isActive`) to avoid passing them to the DOM.
- Token/protocol SVG assets should be hosted on CDN and referenced via token list, not bundled in the app.
- Font weight as numbers (e.g. 600), not strings.
- Avoid `!important` in CSS. Find proper specificity solutions instead.
- Don't create empty styled components just for naming purposes.

---

## Internationalization (i18n)

- All user-facing strings must use `<FormattedMessage>` (or equivalent) with a unique description/key and a default message.
- The description/key field describes the UI location, not the content: `earn.strategy-card.deposit-button`.
- For programmatic use (placeholders, aria labels), use `intl.formatMessage(defineMessage({ ... }))`.
- Description values must be unique across the codebase since they serve as message IDs.

---

## Architecture

### Service Layer
- **Services must be stateless.** They execute logic and return results, but never store data. All state lives in the state manager (Zustand, Redux, etc.).
- The data flow is: `UI component -> state manager -> service -> returns result -> state manager updates`. Components never call services directly.
- If a service needs data (e.g. token list), it receives it as a parameter or accesses the state manager, not its own internal state.
- SDK types are never used directly in components. A parsing layer transforms SDK data to app types first.

### Code Organization
- Reusable components used across features go in `common/components/`. Feature-specific components stay in their feature directory.
- Path aliases (`@common`, `@hooks`, `@services`, `@state`) prevent fragile relative paths and clarify module boundaries.
- Feature toggling through configuration arrays (chain lists, token blacklists) rather than runtime flag systems.
- Polling logic lives in headless updater components rendered as part of a polling handler root.

### Transaction Architecture
- Multi-step transaction flows (approve -> sign -> execute) are built dynamically via `buildSteps()` based on current approval/permit state.
- Optimistic updates (virtual events) provide immediate UI feedback while waiting for blockchain indexing.
- Each transaction type has a typed data structure for capturing post-transaction state.

---

## Security

- API keys, tokens, and service URLs go in environment variables. Never hardcode in source.
- Default to specific (exact amount) token approvals over unlimited.
- Validate token addresses with regex before performing lookups.
- Token blacklists (separate per product) prevent interaction with deprecated or compromised tokens.

---

## Performance

- Virtualize large lists (token pickers) with `react-window` or similar.
- HTTP response caching (15 min) for mostly-static data, excluding dynamic endpoints (balances, history).
- Price caching with short TTL (1 min), stale fallback (5 min), and LRU eviction.
- All analytics calls are fire-and-forget with swallowed errors. Analytics failures never block the user.
- Quote fetching uses debounced calls to avoid excessive API requests during rapid parameter changes.
- Auto-refresh timers handle browser tab visibility changes. Refresh immediately if tab was hidden too long.
- Balance fetching is wallet-aware: only update the active wallet's data, preserve others.

---

## Analytics

- Event tracking should be verbose with properties. Use explicit string values (`is: 'from'`) instead of booleans.
- Event naming follows `Feature - Action state` pattern (e.g. `Earn - Deposit submitting`).
- Analytics calls are fire-and-forget. Never let tracking failures affect the user experience.

---

## Pre-PR Checklist

Before opening a PR, verify:

- [ ] No `console.log` statements
- [ ] No commented-out code
- [ ] No hardcoded hex colors or pixel values
- [ ] All user-facing strings use i18n
- [ ] All financial math uses `bigint`
- [ ] Address comparisons use `isSameAddress`, not `===`
- [ ] Token amounts displayed with `formatCurrencyAmount`, not `.toFixed()`
- [ ] Multi-chain data uses item's `chainId`, not wallet's network
- [ ] Secrets in env vars, not source code
- [ ] Dependencies pinned to exact versions
