# Swap & Aggregator Patterns Analysis

Source: 133 swap-related PRs + current codebase at `apps/root/src/pages/aggregator/swap-container/`

---

## 1. Component Architecture

### Page Layout

The swap page follows a top-level container pattern with three sections:

```
SwapContainer (index.tsx)
  -> Title + Description
  -> Swap (main form component)
  -> AggregatorLanding (promotional/info section)
  -> AggregatorFAQ (accordion FAQ)
```

`SwapContainer` is the data orchestration layer. It:
- Reads URL params (`from`, `to`, `chainId`) via `useParams`
- Resolves tokens with `useToken` (supports address and symbol lookup)
- Handles custom token imports via `useAddCustomTokenToList` for pasted addresses
- Calls `useSwapOptions` hook with all parameters to fetch quotes
- Auto-selects first quote when results arrive, clears selection while loading
- Wraps everything in `React.memo` for performance

### Swap Component (Main Form)

`Swap` component at `swap-container/components/swap/index.tsx` (~1000+ lines) is the central orchestrator. It manages:
- Modal states (picker, settings, confirmation, transaction steps)
- Token approval logic
- Permit2 signing flow
- Actual swap execution
- Safe app batch approve+swap
- Post-swap transaction tracking and confirmation display

State management mixes Redux (via `useAggregatorState`) for cross-component form values with local `useState` for UI concerns (modals, transaction steps, current quote status).

### Sub-component Breakdown

```
Swap
  -> SwapFirstStep (step1/) - The visible form
    -> FormWalletSelector - Wallet picker
    -> SwapNetworkSelector - Chain selector
    -> TokenPickerWithAmount (x2) - From/To token+amount inputs
    -> ToggleButton - Swap direction (from<->to)
    -> QuoteSelection - Best route display + diff
      -> QuotePicker - Source selector dropdown
      -> QuoteRefresher - Auto-refresh timer (60s)
    -> QuoteSimulation - Blowfish/provider tx simulation
    -> QuoteData - Tx cost + min received / max spent
    -> SwapButton - Context-aware CTA
    -> TransferTo - Optional recipient address
  -> SwapSettings - Slide-up advanced settings panel
  -> TokenPicker - Full modal token search/select
  -> TransactionSteps - Multi-step flow UI
  -> TransactionConfirmation - Post-swap receipt
```

### Styled Components Pattern

All components use `styled-components` with theme access. Common pattern:

```tsx
const StyledBackgroundPaper = styled(BackgroundPaper)<{ $showsCog: boolean }>`
  position: relative;
  overflow: hidden;
  ${({ theme: { space }, $showsCog }) => `
    padding: ${space.s07} ${space.s05} ${space.s06};
    ${!$showsCog && `padding-top: ${space.s06};`}
  `}
`;
```

Transient props use `$` prefix to avoid passing to DOM. Theme tokens come from the `ui-library` package.

---

## 2. State Management

### Two Redux Slices

**`state/aggregator/`** - Form state:

```ts
interface AggregatorState {
  fromValue: string;     // Raw input string
  toValue: string;
  from: Token | null;
  to: Token | null;
  isBuyOrder: boolean;   // Toggled by which field user types in
  selectedRoute: SwapOptionWithFailure | null;
  transferTo: null | string;
  network: number;       // ChainId for aggregator
  cleared: boolean;      // Prevents auto-selection after manual clear
}
```

Actions: `setFromValue`, `setToValue`, `setFrom`, `setTo`, `toggleFromTo`, `setSelectedRoute`, `setTransferTo`, `setAggregatorChainId`, `resetForm`, `setCleared`.

Key behavior in `setFromValue`/`setToValue`: takes `{ value, updateMode? }`. When `updateMode` is true, it toggles `isBuyOrder`. This is how the UI knows if user is entering a sell or buy amount.

On `toggleFromTo`: swaps both tokens AND values, flips `isBuyOrder`.

On `setAggregatorChainId`: resets tokens (keeping protocol token if selected), clears values, route, and transferTo.

On `resetForm`: preserves `from`, `to`, and `network` but clears everything else. Sets `cleared: true`.

**`state/aggregator-settings/`** - Persisted user preferences:

```ts
interface AggregatorSettingsState {
  gasSpeed: GasKeys;              // 'standard' | 'fast' | 'instant'
  slippage: string;               // "0.3" - stored as string for input binding
  disabledDexes: string[];        // Source IDs to exclude
  showTransactionCost: boolean;
  confettiParticleCount: number;  // Post-swap celebration
  sorting: SwapSortOptions;       // Sort criteria for quotes
  isPermit2Enabled: boolean;      // Universal approvals toggle
  sourceTimeout: TimeoutKey;      // How long to wait for each source
}
```

Defaults: slippage 0.3%, gas speed "fast", sorting by "most-swapped" (most received tokens), Permit2 enabled, source timeout "balance" (3.5s).

Settings are hydrated from persistence on load via `hydrateAggregatorSettings` action.

---

## 3. Quote Handling

### Fetching Quotes - `useSwapOptions` Hook

Located at `hooks/useSwapOptions.ts`. Key patterns:

- Takes all swap parameters (from, to, chainId, value, isBuyOrder, sorting, transferTo, slippage, gasSpeed, disabledDexes, isPermit2Enabled, sourceTimeout)
- Uses **debounced calls** to avoid excessive API requests
- Tracks previous values of all params; re-fetches when any relevant param changes
- Skips when pending transactions exist (waits for confirmation)
- Returns `[swapOptions, isLoading, error, fetchOptions]` tuple

Quote flow:
1. `aggregatorService.getSwapOptions()` returns a `{ resultsPromise, results, quotesToSimulate }` structure
2. Individual source promises resolve independently - results array grows incrementally
3. When Permit2 is enabled and token is not protocol token, quotes that need simulation go through `simulationService.simulateQuotes()`
4. Simulated quotes replace initial estimates
5. Results are sorted by the user's chosen sorting criteria

### Quote Types

The SDK returns `EstimatedQuoteResponse` (no tx data) or `QuoteResponseWithTx` (with executable tx). These are converted via `quoteResponseToSwapOption()` utility in `common/utils/quotes.ts` to the app's `SwapOption` type with BigInt amounts.

### Quote Sorting

Three sort modes defined in `constants/aggregator.ts`:
- `SORT_MOST_RETURN` ("most-swapped") - Most received tokens for sell orders, least spent for buy orders
- `SORT_MOST_PROFIT` ("most-swapped-accounting-for-gas") - Best return minus gas cost
- `SORT_LEAST_GAS` ("least-gas") - Lowest gas cost

### Quote Comparison - `getBetterBy` / `getWorseBy`

These functions in `common/utils/quotes.ts` calculate percentage differences between quotes:
- For MOST_RETURN: `((best.buyAmount - second.buyAmount) * 10^18 * 100) / second.buyAmount`
- For MOST_PROFIT: calculates `boughtUSD - soldUSD - gasCostUSD` for each, takes difference
- For LEAST_GAS: `((second.gasCost - best.gasCost) * 10^18 * 100) / second.gasCost`

Exponential number parsing is required when profit differences are very small (fixed with `parseExponentialNumberToString`).

### Auto-Refresh

`QuoteRefresher` component implements a 60-second countdown timer. When it reaches 0, quotes are re-fetched. Also handles browser tab visibility changes - if the tab was hidden for 60+ seconds, refreshes immediately on return.

---

## 4. Aggregator Integration

### Multi-Source Architecture

The app queries multiple DEX aggregators simultaneously through the `@balmy/sdk` (formerly `@mean-finance/sdk`). Sources include: 1inch, Uniswap, 0x, OKX DEX, Rango, Changelly, DODO, Squid, Magpie, and others.

### Source Configuration

In `services/sdkService.ts`, the SDK is configured with:
- API endpoint override: individual quote requests per source (changed from batch to single in PR #684)
- Custom configs per source (e.g., Squid requires `integratorId: 'meanfinance-api'`)
- Source list type: `'overridable-source-list'` allowing API and default sources

### Disabled Sources

`disabledDexes` array in settings. Currently `['portals-fi', 'bebop']` by default. Magpie was temporarily disabled (PR #1341) then re-enabled (PR #1361). Sources can be toggled individually in SwapSettings via checkboxes.

### Source Timeout

Configurable per-source timeout with chain-specific overrides. Rootstock gets longer timeouts (2.8x multiplier). Values: instant (1s), rapid (2s), balance (3.5s), patient (5s).

### Supported Chains

Moved from a blacklist (`REMOVED_AGG_CHAINS`) to a whitelist (`AGGREGATOR_SUPPORTED_CHAINS`) in PR #719. Current list is filtered against SDK-supported chains at build time:

```ts
export const AGGREGATOR_SUPPORTED_CHAINS: ChainId[] = [
  Chains.ARBITRUM.chainId, Chains.AVALANCHE.chainId, Chains.BASE.chainId,
  Chains.BNB_CHAIN.chainId, /* ...18 more chains... */
].filter((chainId) => aggSupportedChains.includes(chainId));
```

### Curated Token List

The aggregator token picker uses `curateList: true` which filters tokens to only those from a curated list (`Mean-Finance/token-lister`). This prevents showing obscure/scam tokens. Custom tokens can still be imported by pasting addresses.

---

## 5. Slippage & Settings

### Slippage Input

`SlippageInput` component provides:
- Predefined buttons: 0.1%, 0.3%, 1%
- Custom text input with regex validation: `^((100)|(\d{1,2}(\.\d{0,2})?))%?$`
- On blur, reverts to default (0.3%) if input is NaN
- Tracks user-initiated vs programmatic changes via `setByUser` flag

### Zero Slippage Handling

PR #1291 fixed a bug where 0% slippage was treated as undefined. Changed from `slippagePercentage && !isNaN(slippagePercentage)` to `!isNil(slippagePercentage) && !isNaN(slippagePercentage)` using lodash's `isNil`.

### Settings Panel

`SwapSettings` uses a `Slide` animation (direction="up"). Contains:
1. Slippage - Accordion with SlippageInput
2. Gas speed - Standard/Fast/Instant buttons
3. Source waiting time - Instant/Rapid/Balance/Patient buttons
4. Enabled sources - OptionsMenu with checkboxes for each DEX
5. Quote sorting - OptionsMenu with sort options
6. Universal Approval (Permit2) - Toggle switch
7. Restore defaults button

All setting changes are tracked via analytics (`trackSlippageChanged`, `trackGasSpeedChanged`, etc.).

---

## 6. Token Selection

### AggregatorTokenPicker

Wrapper around the `TokenPicker` from `ui-library`. Key behaviors:
- Fetches token list with `useTokenList({ chainId, curateList: true })`
- Includes custom tokens from `useCustomTokens`
- Shows wallet balances via `useWalletBalances`
- Supports custom token import: `onFetchCustomToken` calls `addCustomTokenToList(address, chainId)`
- Provides protocol token and wrapped protocol token for special handling
- Tokens are parsed via `parseTokensForPicker` which merges balance data

### Token Resolution from URL

`SwapContainer` supports deep-linking with `/swap/:chainId/:from/:to`. Token resolution:
- Uses `useToken` hook with `checkForSymbol: true` and `filterForDca: false`
- If token address is valid but not in lists, triggers custom token import
- Falls back to protocol token for `from` if nothing is selected

### Token Toggle

`ToggleButton` dispatches `setSelectedRoute(null)` then `toggleFromTo()`. Updates URL history with swapped addresses. Disabled during loading.

---

## 7. Price Display

### USD Values

Price display follows a fallback chain:
1. Quote's `amountInUSD` from the aggregator source
2. Fetched USD price via `useUsdPrice` hook or `usePortfolioPrices` from balance state
3. Calculated via `parseUsdPrice(token, amount, pricePerUnit)`

### Price Impact

Calculated in `SwapFirstStep`:
```ts
const priceImpact = (
  Math.round(((Number(toUsdValueToUse) - Number(fromUsdValueToUse)) / Number(fromUsdValueToUse)) * 10000) / 100
).toFixed(2);
```

Only shown when both USD values are available. Passed to the "You receive" `TokenPickerWithAmount` as `priceImpact` prop.

### Missing Price Warning

When USD price cannot be determined for either token, an `Alert` (warning severity) is shown:
> "We couldn't calculate the price for {from}{and}{to}, which means we cannot estimate the price impact. Please be cautious and trade at your own risk."

### QuoteData Component

Displays below the quote selection area:
- **Transaction cost**: `$X.XX (Y.YY ETH)` - gas cost in USD and native token
- **Minimum received** (sell orders): Based on slippage-adjusted `minBuyAmount`
- **Maximum spent** (buy orders): Based on slippage-adjusted `maxSellAmount`

Each has a tooltip explaining the slippage relationship.

---

## 8. Transaction Submission

### Three Swap Paths

1. **Direct Swap** (`handleSwap`): Standard EOA swap. Requires tx data on the selected route.
2. **Multi-Step Swap** (`handleMultiSteps`): Approve -> (Wait for Simulation) -> Sign Permit2 -> Swap. Used when Permit2 is enabled or approval is needed.
3. **Safe Approve and Swap** (`handleSafeApproveAndSwap`): Gnosis Safe batch transaction combining approval and swap.

### Direct Swap Flow

```
1. Validate: from, to, selectedRoute, selectedRoute.tx, totalAmountToApprove
2. Show loading modal with "Swapping X FROM for Y TO"
3. Capture balance before (if protocol token involved)
4. Set maxSellAmount on swap option (for buy orders, uses max across all quotes)
5. Call aggregatorService.swap(parsedSwapOption)
6. Track analytics (swap source, amounts, USD values)
7. Add transaction to Redux with SwapTypeData
8. Show confirmation screen, scroll to top
9. Update transaction steps if in multi-step flow
```

### SwapTypeData Structure

```ts
interface SwapTypeData {
  type: TransactionTypes.swap;
  typeData: {
    from: Token;
    to: Token;
    amountFrom: bigint;
    amountTo: bigint;
    amountFromUsd: number;
    amountToUsd: number;
    balanceBefore: string | null;
    transferTo?: string | null;
    swapContract: string;
    orderType: 'buy' | 'sell';  // Was 'type' before PR #1122
  };
}
```

### Wrap/Unwrap Simplification

PR #1122 removed separate `TransactionTypes.wrap` and `TransactionTypes.unwrap`. All swaps (including wrapping ETH->WETH and unwrapping) now use `TransactionTypes.swap`. The swap button still shows "Swap" for wrap/unwrap but the approve+swap Safe button shows just "Swap" when wrapping (since wrapping doesn't need approval).

### Post-Swap Confirmation

`TransactionConfirmation` component:
- Tracks transaction pending state via `useIsTransactionPending`
- On confirmation, calculates actual amounts sent/received:
  - ERC20: parses transfer events from receipt logs via `aggregatorService.findTransferValue`
  - Protocol token: compares `balanceBefore` with `balanceAfter`, subtracting gas used
- Shows balance changes with OUTGOING (from) and INCOMING (to) indicators
- Displays "Transferred to: 0xabc...xyz" if `transferTo` was set
- Order: outgoing shown first, then incoming (fixed in PR #1146)

---

## 9. Approval Flows

### Approval Check

```ts
const isApproved =
  !from || !selectedRoute ||
  (from && selectedRoute && (
    (allowance.allowance &&
     allowance.token.address === from.address &&
     parseUnits(allowance.allowance, from.decimals) >= selectedRoute.maxSellAmount.amount) ||
    from.address === PROTOCOL_TOKEN_ADDRESS  // Native token needs no approval
  ));
```

Uses `useSpecificAllowance` hook with target address depending on Permit2 status:
- Permit2 enabled: `PERMIT_2_ADDRESS[chainId]`
- Permit2 disabled: `selectedRoute.swapper.allowanceTarget`

### Multi-Step Transaction Flow

When approval is needed, `handleMultiSteps` builds a step array:

```ts
const steps = [
  { type: TRANSACTION_ACTION_APPROVE_TOKEN, ... },           // ERC20 approve
  { type: TRANSACTION_ACTION_WAIT_FOR_SIMULATION, ... },     // Tx simulation
  { type: TRANSACTION_ACTION_APPROVE_TOKEN_SIGN_SWAP, ... }, // Permit2 sign (if enabled)
  { type: TRANSACTION_ACTION_SWAP, ... },                    // Execute swap
];
```

Each step has: `hash`, `chainId`, `done`, `failed`, `checkForPending`, `extraData`.

The `TransactionSteps` component renders this as a stepper UI where each step is executed sequentially.

### Permit2 Flow

When Permit2 is enabled:
1. User approves token to Permit2 contract (one-time, shared across DEXes)
2. User signs a Permit2 message (gasless signature)
3. After signing, quotes are re-simulated with the signature
4. Post-simulation can detect:
   - `QuoteStatus.OriginalFailed` - Selected route failed, using best alternative
   - `QuoteStatus.BetterQuote` - A better route was found during simulation
   - `QuoteStatus.AllFailed` - No routes work
5. Final swap executes with the signature attached

### Exact vs Unlimited Approval

The `handleApproveToken` callback accepts an optional `amount` parameter:
- If provided: `TransactionTypes.approveTokenExact` with the specific amount
- If undefined: `TransactionTypes.approveToken` (unlimited approval)

---

## 10. Error States

### SwapButton Decision Tree

The button renders different states based on priority:

```
1. No wallet connected        -> "Connect wallet"
2. Wallet disconnected        -> "Switch to {wallet}'s Wallet"
3. Wrong network              -> "Change network to {network}"
4. Insufficient balance       -> "Insufficient funds" (disabled)
5. Needs approval (Safe app)  -> "Authorize {from} and swap"
6. Needs approval (Permit2)   -> "Continue to Swap" (opens multi-step)
7. Ready to swap              -> "Swap"
```

The `shouldDisableApproveButton` check covers: no from, no to, no fromValue, cantFund, no balance, no selectedRoute, allowance errors, zero amount, or loading.

The `shouldDisableButton` adds: not approved, no tx data, or transaction will fail (from simulation).

### Quote Errors

`QuoteSelection` component handles the "all routes failed" case:
```tsx
if (!quotes.length && !isLoading && !!swapOptionsError) {
  // Shows error icon + "Try to get a route again" button
}
```

### Quote Status Notifications

The `QuoteStatus` enum tracks simulation results:
- `None` - Default
- `AllFailed` - All simulated quotes failed
- `OriginalFailed` - Selected source failed, switched to alternative
- `BetterQuote` - A better quote was found during Permit2 simulation

### Error Categorization

`categorizeError` in `common/utils/quotes.ts` maps error messages to typed enum values:
- `QuoteErrors.TIMEOUT` - Source timed out
- `QuoteErrors.REFERRAL_CODE` - Invalid referral code
- `QuoteErrors.BIGINT_CONVERSION` - BigInt parsing failure
- `QuoteErrors.NETWORK_REQUEST` - Network failure
- `QuoteErrors.UNKNOWN` - Catch-all

### Transaction Simulation

Two simulation paths:
1. **Blowfish** (on supported chains, e.g., Ethereum): Full simulation showing expected state changes (token transfers, approvals)
2. **Provider simulation** (fallback, or forced when `transferTo` is set): Simple gas estimation, shows "Transaction will be successful"

When `transferTo` is set, Blowfish simulation is bypassed in favor of provider simulation (PR #288).

### Error Logging

All errors go through `errorService.logError` with structured metadata:
```ts
errorService.logError('Error swapping', JSON.stringify(e), {
  swapper: selectedRoute.swapper.id,
  chainId: currentNetwork.chainId,
  from: selectedRoute.sellToken.address,
  to: selectedRoute.buyToken.address,
  buyAmount: selectedRoute.buyAmount.amountInUnits,
  sellAmount: selectedRoute.sellAmount.amountInUnits,
  type: selectedRoute.type,
});
```

User-cancelled transactions (detected via `shouldTrackError`) close the modal silently without logging.

---

## 11. Transfer To (Swap and Send)

Swap supports sending output tokens to a different address:
- `transferTo` stored in aggregator Redux state
- Opened via contact list modal (`ContactListActiveModal.CONTACT_LIST`)
- `TransferTo` component shows the destination address with controls
- When set, it's passed through the entire quote/simulation/swap pipeline
- PR #1360 added ENS resolution support
- Balance checks for protocol tokens use the recipient's address when `transferTo` is set
- Provider simulation is forced (not Blowfish) when `transferTo` is set

---

## 12. Analytics

Every significant user action is tracked:
- Token/amount changes
- Quote selection changes
- Settings modifications (each setting has a dedicated tracker)
- Approval submissions/completions/errors
- Swap submissions/completions/errors
- Quote picker open/close
- Network changes
- Refresh quotes
- Toggle from/to

The `trackSwap` call captures chainId, tokens, and USD value for conversion tracking.

---

## 13. Network Handling

### Chain Selection

`SwapNetworkSelector` filters available chains against `AGGREGATOR_SUPPORTED_CHAINS`. Changing network:
1. Dispatches `setAggregatorChainId(chainId)` which resets tokens and values
2. Updates URL via `replaceHistory`
3. Does NOT call `walletService.changeNetwork` - that happens only from SwapButton when the wallet is on wrong chain

### Network Mismatch

When wallet network differs from selected aggregator network:
- Quotes still fetch (using selected network)
- SwapButton shows "Change network to {network}" instead of swap

### URL Routing

Pattern: `/swap/:chainId/:from/:to`
- `identifyNetwork(supportedNetworks, chainId)` resolves chain from param
- Falls back through: URL param -> wallet network -> selected network -> mainnet

---

## 14. Key File Paths

- Container: `apps/root/src/pages/aggregator/swap-container/index.tsx`
- Swap form: `apps/root/src/pages/aggregator/swap-container/components/swap/index.tsx`
- Step 1 layout: `apps/root/src/pages/aggregator/swap-container/components/step1/index.tsx`
- Swap button: `apps/root/src/pages/aggregator/swap-container/components/swap-button/index.tsx`
- Settings: `apps/root/src/pages/aggregator/swap-container/components/swap-settings/index.tsx`
- Slippage: `apps/root/src/pages/aggregator/swap-container/components/swap-settings/components/slippage-input/index.tsx`
- Token picker: `apps/root/src/pages/aggregator/swap-container/components/token-picker/index.tsx`
- Quote selection: `apps/root/src/pages/aggregator/swap-container/components/quote-selection/index.tsx`
- Quote picker: `apps/root/src/pages/aggregator/swap-container/components/quote-picker/index.tsx`
- Quote data: `apps/root/src/pages/aggregator/swap-container/components/quote-data/index.tsx`
- Quote refresher: `apps/root/src/pages/aggregator/swap-container/components/quote-refresher/index.tsx`
- Quote simulation: `apps/root/src/pages/aggregator/swap-container/components/quote-simulation/index.tsx`
- Toggle button: `apps/root/src/pages/aggregator/swap-container/components/toggle-button/index.tsx`
- Transfer to: `apps/root/src/pages/aggregator/swap-container/components/transfer-to/index.tsx`
- Constants: `apps/root/src/constants/aggregator.ts`
- Quote utils: `apps/root/src/common/utils/quotes.ts`
- Aggregator state: `apps/root/src/state/aggregator/reducer.ts`
- Settings state: `apps/root/src/state/aggregator-settings/reducer.ts`
- useSwapOptions hook: `apps/root/src/hooks/useSwapOptions.ts`
