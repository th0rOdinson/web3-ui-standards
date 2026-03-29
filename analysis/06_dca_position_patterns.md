# DCA, Position, Transfer & Migration Patterns

## 1. DCA Creation Flow

### State Management (Redux Toolkit)
The DCA creation form uses a dedicated Redux slice at `apps/root/src/state/create-position/`.

**State shape** (`CreatePositionState`):
```
fromValue: string          // total deposit amount
rate: string               // per-swap rate (computed: fromValue / frequencyValue)
frequencyType: bigint      // swap interval (ONE_DAY, etc.)
frequencyValue: string     // number of swaps (default "7")
from: Token | null
to: Token | null
fromYield: PositionYieldOption | null
toYield: PositionYieldOption | null
userHasChangedYieldOption: boolean
chainId: number
modeType: ModeTypesIds     // FULL_DEPOSIT_TYPE or RATE_TYPE
```

**Actions**: `setFromValue`, `setFrom`, `setTo`, `setFrequencyType`, `setFrequencyValue`, `setFromYield`, `setToYield`, `setRate`, `setModeType`, `setDCAChainId`, `resetDcaForm`.

**Chain change resets**: When `setDCAChainId` fires, it clears `fromValue`, resets frequency to defaults (daily/7), clears yields, and resets `from` to the chain's protocol token if the current from-token is on a different chain.

### Component Architecture
- `SwapContainer` (`create-position/index.tsx`): Top-level container. Reads URL params (`from`, `to`, `chainId`) to pre-populate tokens. Provides network handling.
- `Swap` (`create-position/components/swap/index.tsx`): Main form orchestrator with multi-step transaction flow.
- `SwapFirstStep` (`step1/`): Token selection, frequency configuration.
- `DcaButton` (`dca-button/index.tsx`): Smart button with 10+ conditional states.
- `TokenPicker` / `TokenSelector`: Token selection modals.
- `FrequencySelector`: Interval/count picker.
- `YieldSelector` (`step2/`): Optional yield configuration.
- `DcaRecapData`: Summary before confirmation.
- `DcaLanding`: Marketing/info section below the form.

### Multi-step Transaction Pattern
The Swap component manages a `transactionsToExecute: TransactionStep[]` array and renders `TransactionSteps`. Steps can include:
1. `TRANSACTION_ACTION_APPROVE_TOKEN_SIGN_DCA` - Permit2 signature for token approval
2. `TRANSACTION_ACTION_APPROVE_TOKEN` - ERC20 approve fallback
3. `TRANSACTION_ACTION_CREATE_POSITION` - The actual position creation

Safe App flow (`SafeApproveAndStartPositionButton`) bundles approve + create into a single Safe transaction batch.

### Validation Rules
- **Pair support**: `useCanSupportPair` checks if the pair exists on the protocol.
- **Minimum USD rate**: Per-chain minimum via `MINIMUM_USD_RATE_FOR_DEPOSIT` (e.g., $1 on Polygon, $5 on mainnet). Positions below threshold are blocked.
- **Max swaps**: `frequencyValue` cannot exceed `MAX_UINT_32` (4294967295).
- **Whale mode**: Certain high-frequency intervals require minimum total USD values via `WHALE_MINIMUM_VALUES`.
- **Yield eligibility**: Requires minimum USD rate per swap (`MINIMUM_USD_RATE_FOR_YIELD`) and at least 10 days duration.
- **Token blacklists**: `DCA_TOKEN_BLACKLIST` prevents creation with blacklisted tokens. Separate `TOKEN_BLACKLIST` for aggregator.
- **Pair blacklist**: `DCA_PAIR_BLACKLIST` blocks specific trading pairs.

### Button State Priority
```
1. No wallet -> "Connect wallet"
2. Wallet disconnected -> "Switch to {wallet}'s Wallet"
3. Wrong network -> "Change network to {network}"
4. Network not supported -> "We do not support this chain yet"
5. Loading pair support -> Loading spinner
6. Insufficient funds -> "Insufficient funds"
7. Pair not supported -> "We do not support this pair"
8. Below minimum USD -> Shows minimum required
9. Protocol token (no approve needed) -> "Create position"
10. Non-protocol token -> "Authorize and create position"
11. Safe app -> "Authorize {from} and create position" (batched)
```

### Deposit Parameter Building
`positionService.buildDepositParams()`:
- Calculates `weiValue` from `fromValue` with token decimals
- Assigns companion permissions based on yield and protocol token usage:
  - `from` has yield OR is protocol token -> grants INCREASE, REDUCE
  - `to` has yield OR is protocol token -> grants WITHDRAW
  - Any yield or protocol token -> grants TERMINATE
- Returns structured params for SDK call

### Permit2 Integration
When wallet supports signing (`useSupportsSigning`), the flow uses Permit2:
- Gets nonce from `permit2Service`
- Signs EIP-712 permit message
- Passes signature data to `buildDepositTx` which includes it in the SDK call as `permitData`

## 2. Position Display

### Position List Page
**File**: `apps/root/src/pages/dca/positions/`

**Architecture**:
- `Positions` (`positions/index.tsx`): Tab container with "Active" and "History" tabs. Shows dashboard only when positions exist.
- `CurrentPositions`: Lists active positions with action handlers.
- `History`: Lists past/terminated positions.
- `PositionCard`: Individual position card display.

**Key display elements per card**:
- Token pair with icons
- Remaining liquidity and swapped amounts
- Swap frequency and remaining count
- Position warnings (frozen tokens, hacks, deprecated networks)
- Pending transaction indicator
- Action buttons (withdraw, modify, terminate)

### Position Detail Page
**File**: `apps/root/src/pages/position-detail/`

**Components**:
- `PositionDetailFrame` (`frame/index.tsx`): Page container, fetches position data.
- `SummaryContainer` with `PositionData`: Shows from/to tokens, rates, swapped amounts, average buy price, total gas saved, remaining info.
- `PositionSummaryControls`: Action buttons (withdraw, modify, terminate, transfer, view NFT, download CSV).
- `GraphContainer`: Multiple graph views (swap price, average price, gas saved, profit/loss).
- `PermissionsContainer`: Operator permission management.
- `Timeline` (swaps history): Chronological list of all position events.

### Average Buy Price Calculation
Iterates over swapped actions, builds a `ratioSum` keyed by token address, then divides by swap count:
```typescript
for (const { tokenA, tokenB, ratioAToB, ratioBToA } of swappedActions) {
  ratioSum[tokenA.address] = (ratioSum[tokenA.address] ?? 0n) + ratioAToB;
  ratioSum[tokenB.address] = (ratioSum[tokenB.address] ?? 0n) + ratioBToA;
}
averageBuyPrice = ratioSum[tokenFromAverage.address] / BigInt(swappedActions.length);
```

### Position Warnings
Conditional alert banners shown for:
- AAVE frozen tokens (Euler hack impact)
- Sonne frozen tokens (Sonne hack)
- Yearn/Sonne frozen tokens (cascading impact)
- agEUR tokens (Angle protocol paused)
- jEUR tokens (similar issues)
- Base Goerli deprecation
- Double-yield positions on Ethereum mainnet

The `PositionWarning` component checks `AAVE_FROZEN_TOKENS`, `SONNE_FROZEN_TOKENS`, `YEARN_SONNE_FROZEN_TOKENS` arrays against position yield token addresses.

### CreatePositionBox
When viewing the position list, a dashed-border CTA card is injected in the grid promoting "Create new position". Uses `changeRoute` + `pushToHistory` to navigate.

### Empty Positions State
Custom `EmptyPositions` component with a styled dashed card and a "Create new position" button linking to the DCA create route.

## 3. Position Management (Modify, Withdraw, Terminate)

### Modify Rate and Swaps
`positionService.modifyRateAndSwaps()` and `modifyRateAndSwapsSafe()`:
- Takes position, new rate, new swap count, and yield options
- Handles companion permissions for protocol token and yield scenarios
- Supports Permit2 signatures for token approvals when increasing position
- Safe version batches approve + modify in one transaction

### Withdraw Swapped Funds
`positionService.withdraw(position, useProtocolToken)`:
- If `useProtocolToken` is true and position `to` is wrapped protocol token, companion converts to native
- Requires companion WITHDRAW permission check first via `companionHasPermission()`
- Shows modal flow: loading -> success with hash -> error handling

### Withdraw Remaining Funds
Similar to withdraw but pulls from `remainingLiquidity` instead of `toWithdraw`. Requires REDUCE permission for companion.

### Terminate Position
`positionService.terminate(position, useProtocolToken)`:
- Returns both remaining liquidity and swapped funds
- Handles protocol token conversion via companion
- `terminateSafe()` for Safe app contexts
- `terminateManyRaw()` for batch termination of multiple positions

**Terminate modal** shows what user will receive back (from token remaining + to token swapped). Handles special case of transferred positions using wrapped tokens.

### Version Gating
- `VERSIONS_ALLOWED_MODIFY`: Only latest version positions can be modified.
- `showExtendedFunctions`: Requires `LATEST_VERSION`, pair not blacklisted, tokens still in DCA token list, and frequency still enabled.

### Disabled States
- Withdraw disabled when yield token is in `DISABLED_YIELD_WITHDRAWS`
- Terminate disabled when either from or to yield tokens are in disabled list
- All actions disabled during pending transactions
- All actions disabled when wallet is on wrong network

## 4. Rate / Frequency Handling

### Swap Intervals
Defined as constants (`ONE_DAY`, etc.) mapped to bigint values. Frequencies controlled by `shouldEnableFrequency()` which checks per-token and per-chain configuration.

### Mode Types
Two input modes via `ModeTypesIds`:
1. `FULL_DEPOSIT_TYPE` - User enters total amount, rate is calculated as `fromValue / frequencyValue`
2. `RATE_TYPE` - User enters per-swap rate, total is `rate * frequencyValue`

### Rate Calculation
In the Swap component:
```typescript
setRate(
  fromValue && parseUnits(fromValue, from.decimals) > 0n && frequencyValue && BigInt(frequencyValue) > 0n
    ? formatUnits(parseUnits(fromValue, from.decimals) / BigInt(frequencyValue), from.decimals)
    : '0'
);
```

### Frequency Display
`STRING_SWAP_INTERVALS` maps interval values to human-readable strings with singular/plural/adverb forms for i18n. Used in position cards and detail views.

### Default Values
- Default frequency: `ONE_DAY` (daily)
- Default swap count: `"7"` (one week)
- Resets to these defaults on chain change

## 5. History / Logs

### Position History Model
Each position has a `history: DCAPositionAction[]` array. Action types:
- `SWAPPED` - Automated swap executed
- `CREATED` - Position created
- `MODIFIED` - Rate/swaps changed
- `WITHDREW` - User withdrew swapped funds
- `TERMINATED` - Position closed
- `PERMISSIONS_MODIFIED` - Operator permissions changed
- `TRANSFERED` - Position transferred to another address

### Timeline Component
`apps/root/src/pages/position-detail/components/summary-container/components/swaps/components/timeline/index.tsx`

Uses a `MESSAGE_MAP` keyed by action type to render each history event with:
- Formatted amounts and token symbols
- Timestamps
- Transaction links (etherscan)
- Operator addresses for permission changes

Filters available: all events, swaps only, modifications only, etc.

### CSV Export
`PositionSummaryControls` fetches CSV data via `positionService.fetchPositionSwapsForCSV(position)`, creates a blob URL, and renders a hidden download link. CSV fetch is gated by `show` prop to avoid unnecessary API calls.

### Graph History
The detail page includes multiple graph types:
- **Swap Price Graph**: Price at each swap execution
- **Average Price Graph**: Running average buy price
- **Gas Saved Graph**: Cumulative gas savings vs individual swaps
- **Profit/Loss Graph**: P&L over time

History is sorted by `createdAtBlock` (not timestamp) for accurate ordering.

## 6. Transfer Patterns

### Token Transfer (Transfer Page)
**Location**: `apps/root/src/pages/transfer/`

**State** (`TransferState`):
```
network: number
token: Token | null
amount: string
recipient: string
```

**Flow**:
1. User selects network (any supported chain, not just DCA chains)
2. User selects token from their balance (sorted by USD value descending)
3. User enters recipient address
4. Address validation: regex check (`/^0x[A-Fa-f0-9]{40}$/`), cannot be own wallet
5. ENS resolution supported via `useEnsAddress`
6. Fee estimation via `useEstimateTransferFee`
7. Confirm modal -> transaction

**Contact System**: Recipients can be saved as contacts with labels. `ContactListService` manages CRUD with API-backed storage. Loading states show skeleton items.

**Address Validation** (`useValidateAddress` hook):
- Validates hex address format
- Optionally restricts sending to own wallet (`restrictActiveWallet`)
- Shows contact match helper text when recipient matches a saved contact
- All addresses lowercased on input

### Position Transfer (DCA)
**Location**: `apps/root/src/pages/position-detail/components/transfer-position-modal/`

**Flow**:
1. User enters recipient hex address
2. Validated with `validRegex = /^0x[A-Fa-f0-9]{40}$/`
3. Input sanitized and lowercased
4. Calls `positionService.transfer(position, toAddress)`
5. Uses PermissionManager's `transferFrom` function (ERC721 transfer)
6. Transfers the position NFT, all liquidity, and all permissions

**Post-transfer handling in `positionService.handleTransaction`**:
- If recipient is one of user's own wallets: updates `user` field, clears permissions, keeps position in list
- If recipient is unknown address: removes position from local state

**Position Detail Reducer**: On transfer, sets `permissions: []` and updates `user` field.

### Swap Transfer (Aggregator)
The aggregator swap supports a `transferTo` field where swapped output is sent to a different address. This is tracked in `SwapTypeData.transferTo` and used in simulation service to set the correct recipient.

## 7. Migration Patterns

### One-Click Migration (Earn)
**Location**: `apps/root/src/pages/earn/home/components/one-click-migration-modal/`

**Purpose**: Move funds from external DeFi protocols (Aave, Yearn, etc.) into the project's Earn vaults.

**Flow**:
1. `useAvailableDepositTokens` hook scans user balances for tokens that can be migrated
2. Shows migration options modal with strategy selection
3. User picks a guardian/strategy for their funds
4. URL params pass deposit details: `assetToDepositAmount`, `underlyingAmount`, asset addresses
5. `EarnDepositTransactionManager` reads URL params and dispatches `setOneClickMigrationSettings`
6. `setTriggerSteps(true)` triggers the multi-step deposit flow

**State integration**: The earn management reducer has a `triggerSteps` boolean flag and a `setOneClickMigrationSettings` action that atomically sets `asset`, `depositAmount`, `depositAsset`, and `depositAssetAmount`.

**Achievement tracking**: Migration deposits update `AchievementKeys.MIGRATED_VOLUME` in the tier service. The `isMigration` flag on `EarnCreateTypeData` and `EarnIncreaseTypeData` distinguishes migrations from regular deposits.

### Strategy Tier Gating
Strategies with `needsTier > 0` require API signature validation. `earnService.generateCreationData()` packs a ToS signature with an API-issued signature obtained via `meanApiService.getEarnStrategySignature()`.

### Protocol Version Migration
The codebase has evolved through position versions (`POSITION_VERSION_1` through `POSITION_VERSION_4`):
- `OLD_VERSIONS` lists deprecated versions
- `VERSIONS_ALLOWED_MODIFY` gates which versions support modification
- `LATEST_VERSION` is the only version for new positions
- `MigrateYieldModal` and `SuggestMigrateYieldModal` handle yield source upgrades

### Network Deprecation
When a network is removed from `SUPPORTED_NETWORKS_DCA` (e.g., Base Goerli), existing positions show a deprecation banner but retain withdraw/terminate functionality. New positions cannot be created on deprecated networks.

## 8. Multi-Chain Positions

### Aggregation Strategy
- `SUPPORTED_NETWORKS_DCA` defines the set of chains where DCA operates
- `fetchCurrentPositions()` calls `sdkService.getUsersDcaPositions()` with all wallet addresses, which returns positions grouped by chain
- Results are reduced across `SUPPORTED_NETWORKS_DCA`, merging into a single `PositionKeyBy` object
- Position IDs encode chain: `${chainId}-${tokenId}-v${version}`

### Network-Specific Configuration
Each chain has its own:
- `HUB_ADDRESS`, `COMPANION_ADDRESS`, `PERMISSION_MANAGER_ADDRESS`
- `MINIMUM_USD_RATE_FOR_DEPOSIT`
- `MINIMUM_USD_RATE_FOR_YIELD`
- `MIN_AMOUNT_FOR_MAX_DEDUCTION` and `MAX_DEDUCTION` (native token buffer for gas)
- Subgraph URLs (`MEAN_GRAPHQL_URL`)
- Wrapped protocol token definitions

### Position Detail Cross-Chain
Position detail pages use `position.chainId` (not `useCurrentNetwork()`) for all chain-specific operations. This was a significant refactor (PR #183) fixing bugs where the detail page used the wallet's current network instead of the position's network for:
- Wrapped protocol token resolution
- Etherscan links
- Permission manager addresses
- Graph tooltips

### Adding a New Chain
Pattern from PR #576 (Moonbeam): requires updates to 15+ config files:
1. Network definition in `RAW_NETWORKS`
2. Wrapped token mock
3. Protocol token mapping
4. `SUPPORTED_NETWORKS` and `SUPPORTED_NETWORKS_DCA` arrays
5. Contract addresses (Hub, Companion, PermissionManager, TokenDescriptor, Multicall)
6. Subgraph URL
7. Gas deduction config
8. Minimum deposit rate
9. Yield configuration
10. Apollo client setup

## 9. Permission Management

### Permission Types
Four permissions defined in `PERMISSIONS` constant:
1. `INCREASE` - Add funds to position
2. `REDUCE` - Remove remaining liquidity
3. `WITHDRAW` - Withdraw swapped funds
4. `TERMINATE` - Close position entirely

### Companion Permissions
The companion contract (a helper contract) needs permissions for:
- Protocol token handling (wrapping/unwrapping)
- Yield token management (deposit/withdraw from yield sources)

Permissions are automatically assigned during position creation based on token types:
```typescript
if (yieldFrom || from.address === PROTOCOL_TOKEN_ADDRESS) {
  permissions = [...permissions, PERMISSIONS.INCREASE, PERMISSIONS.REDUCE];
}
if (yieldTo || to.address === PROTOCOL_TOKEN_ADDRESS) {
  permissions = [...permissions, PERMISSIONS.WITHDRAW];
}
if (yieldFrom || yieldTo || from === PROTOCOL_TOKEN || to === PROTOCOL_TOKEN) {
  permissions = [...permissions, PERMISSIONS.TERMINATE];
}
```

### Permission UI
`PermissionsContainer` component:
- Lists all operators with their current permissions
- `usePositionPermissions(position.id)` reads from Redux
- `useModifiedPermissions()` tracks unsaved changes
- `useHasModifiedPermissions()` enables/disables save button
- `AddAddressPermissionModal` for adding new operators
- Checkbox-based per-permission toggling

### Permission Modification Flow
1. User edits permissions in UI (dispatches `addPermission`/`removePermission` actions)
2. Save triggers `positionService.modifyPermissions(position, newPermissions)`
3. Calls `getModifyPermissionsTx()` which encodes PermissionManager contract call
4. Transaction added to pending with `TransactionTypes.modifyPermissions`
5. On confirmation, position detail reducer updates `permissions` array

### Transfer Clears Permissions
When a position is transferred, all existing permissions are cleared (`permissions: []`). This prevents the old owner's operators from accessing the transferred position.

### Permission Check Before Actions
Before operations requiring companion (withdraw protocol token, reduce yield position), the code calls `positionService.companionHasPermission(position, PERMISSIONS.REDUCE)` to verify the companion still has the needed permissions.

## 10. Data Fetching

### PositionService (EventsManager Pattern)
`PositionService` extends `EventsManager<PositionServiceData>`, a custom pub/sub pattern where:
- State is held in `serviceData`
- Setting any property via setter triggers subscriber notifications
- React hooks subscribe via `useServiceEvents(service, 'getterName')`

**Service data**:
```typescript
interface PositionServiceData {
  currentPositions: PositionKeyBy;
  pastPositions: PositionKeyBy;
  hasFetchedCurrentPositions: boolean;
  hasFetchedPastPositions: boolean;
  hasFetchedUserHasPositions: boolean;
  userHasPositions: boolean;
}
```

### Fetch Flow

**Quick check** (`fetchUserHasPositions`):
- Called early to determine if user has any positions
- Calls `meanApiService.getUsersHavePositions()` with all wallet addresses
- Controls whether DCA page shows create form or position list

**Full fetch** (`fetchCurrentPositions`):
- Called when user navigates to positions view
- `sdkService.getUsersDcaPositions()` returns all positions across all DCA chains
- Maps SDK positions to app's Position type, preserving pending transaction state
- Key: `${chainId}-${tokenId}-v${version}`

**Past positions** (`fetchPastPositions`):
- Same fetch call but filters for terminated positions
- Lazy-loaded when user switches to History tab

**Single position** (`getPosition`):
- Direct SDK call for a specific position by ID, chain, and version
- Returns `PositionWithHistory` including full action history
- Used on position detail page

### Position Detail Fetching
`fetchPositionAndTokenPrices` is an async thunk that:
1. Fetches position data from SDK
2. Fetches USD prices for from and to tokens
3. Stores all in `position-details` Redux slice

### Error Handling Pattern
Multi-chain fetches use `Promise.all` with `.catch()` per promise to prevent one chain's failure from blocking others:
```typescript
const results = await Promise.all(
  promises.map((promise) => promise.catch(() => ({ data: null, error: true })))
);
```

### Optimistic Updates
`handleTransaction()` in PositionService processes confirmed transactions to update local state immediately:
- `newPosition`: Adds position to `currentPositions`
- `withdrawPosition`: Zeros out `toWithdraw`
- `terminatePosition`: Removes from `currentPositions`, adds to `pastPositions`
- `modifyRateAndSwapsPosition`: Updates rate, remainingSwaps, remainingLiquidity
- `transferPosition`: Either updates `user` field (if recipient is own wallet) or removes from list
- `modifyPermissions`: Updates permissions array

### Pending Transaction Tracking
Each position has a `pendingTransaction: string` field (transaction hash or empty string). While pending:
- Action buttons are disabled
- "Pending transaction" link is shown
- UI shows optimistic state changes

### Wallet-Aware Fetching
All data fetching is wallet-aware:
- `accountService.getWallets()` returns all connected wallets
- Positions are fetched for ALL wallet addresses (multi-wallet support)
- `isLoggingUser` flag prevents fetching during wallet connection to avoid race conditions
- On logout, `logOutUser()` calls `resetData()` to clear all cached positions

### Subgraph Data
Historical data (pre-SDK migration) was fetched from The Graph subgraphs. Position history sorting uses `createdAtBlock` instead of `createdAtTimestamp` for accuracy. The codebase migrated from hosted service to decentralized subgraphs and eventually to SDK-based fetching.

### Price Data
Token prices fetched via `priceService.getUsdHistoricPrice()` and `getUsdCurrentPrices()`. Price data is used for:
- USD value display in position cards
- Minimum deposit validation
- Average buy price calculation
- Gas savings estimation
- Portfolio value aggregation
