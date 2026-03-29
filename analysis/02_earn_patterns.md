# Earn/Vault Patterns Analysis

Extracted from 215 earn-related PRs and current codebase analysis.

---

## 1. Component Patterns

### Page Structure

The earn feature follows a three-level page hierarchy:

```
/earn/              -> Home (all strategies table + wizard)
/earn/portfolio/    -> Portfolio (user positions + financial data + TVL graph)
/earn/vaults/:chainId/:strategyId -> Strategy detail (strategy-guardian-detail page)
```

Each page uses a `Frame` component that handles route registration and layout:

```tsx
// Frame pattern: register route + wrap children in StyledNonFormContainer
const EarnFrame = () => {
  const dispatch = useAppDispatch();
  React.useEffect(() => { dispatch(changeRoute(EARN_ROUTE)); }, []);

  return (
    <StyledNonFormContainer flexDirection="column" flexWrap="nowrap">
      <ContainerBox flexDirection="column" gap={32}>
        {/* page content */}
      </ContainerBox>
    </StyledNonFormContainer>
  );
};
```

### Styled Components Convention

Components use `styled-components` with `.attrs()` for default props and template literal syntax for theme access:

```tsx
const StyledCard = styled(Card).attrs({ variant: 'outlined' })<{ $isPromoted?: boolean }>`
  ${({ theme: { palette, spacing }, $isPromoted }) => `
    padding: ${spacing(6)};
    box-shadow: ${colors[palette.mode].dropShadow.dropShadow300};
    ${$isPromoted && `
      outline-color: ${colors[palette.mode].semantic.success.darker};
    `}
  `}
`;
```

Key conventions:
- Transient props prefixed with `$` (e.g. `$isPromoted`, `$isPortfolio`, `$isMigration`)
- Theme colors always accessed via `colors[palette.mode]` helper from `ui-library`
- Spacing always through `theme.spacing()`
- `BackgroundPaper` with `variant="outlined"` as container wrapper

### Strategy Detail Page (`strategy-guardian-detail`)

The detail page is split into two main areas:
- `vault-data/` - Public vault info (header, data cards, guardian, rewards, about)
- `investment-data/` - User's position data (balances, expected returns, financial overview, wallet breakdown)
- `strategy-management/` - Deposit/withdraw forms and transaction flows

Each sub-area follows the same pattern: a top-level component that composes smaller data-display components.

### Table System

The strategies table is the central UI component, reused across three variants via `StrategiesTableVariants`:

```tsx
export enum StrategiesTableVariants {
  ALL_STRATEGIES = 'allStrategies',     // Home page
  USER_STRATEGIES = 'userStrategies',   // Portfolio page
  MIGRATION_OPTIONS = 'migrationOptions', // One-click migration
}
```

The table uses a column config pattern:

```tsx
export type StrategyColumnConfig<T extends StrategiesTableVariants> = {
  key: StrategyColumnKeys;
  label: ReactNode;
  renderCell: (data: TableStrategy<T>, showBalances?: boolean) => ReactNode;
  getOrderValue?: (data: TableStrategy<T>) => number | string;
  hiddenProps: HiddenProps;
};
```

This allows different column sets for different table variants while sharing the table rendering logic.

### Card Components

Strategy cards (`strategy-card-item`) follow a consistent structure:
- Token icon with network indicator
- Asset symbol + reward tokens
- Farm name
- DataCards component for metrics
- "View Vault" link using react-router `Link`

---

## 2. Data Flow

### Service Layer Architecture

The earn feature uses a service-based architecture with `EventsManager` as the base class:

```
EarnService extends EventsManager<EarnServiceData>
  -> SdkService (SDK calls)
  -> AccountService (wallet management)
  -> ProviderService (blockchain interaction)
  -> ContractService (contract addresses)
  -> MeanApiService (API calls)
```

`EarnServiceData` holds the complete earn state:

```tsx
export interface EarnServiceData {
  allStrategies: SavedSdkStrategy[];
  hasFetchedAllStrategies: boolean;
  strategiesParameters: SummarizedSdkStrategyParameters;
  earnPositionsParameters: SummarizedSdkStrategyParameters;
  hasFetchedUserStrategies: boolean;
  userStrategies: SavedSdkEarnPosition[];
}
```

### Data Fetching Pattern

1. `EarnService.fetchAllStrategies()` fetches from SDK, stamps `lastUpdatedAt`, builds parameters
2. `EarnService.fetchUserStrategies()` fetches positions for all connected wallets
3. Each fetch uses `batchUpdateStrategies()` / `batchUpdateUserStrategies()` for efficient updates
4. Conditional fetching via `needsToUpdateStrategy()` checks timestamp staleness against `IntervalSetActions.strategyUpdate`

### Hook Layer

Hooks bridge services to React via `useServiceEvents`:

```tsx
// Pattern: hook calls service method through useServiceEvents
const allStrategies = useServiceEvents<EarnServiceData, EarnService, 'getAllStrategies'>(
  earnService,
  'getAllStrategies'
);
```

Hook hierarchy:
- `useEarnService()` - Gets the EarnService instance from web3Service
- `useAllStrategies()` - Fetches + parses all strategies
- `useEarnPositions()` - Fetches + parses user positions
- `useFilteredStrategies()` - Applies filters, search, sorting, tier-level filtering
- `useStrategyDetails()` - Single strategy detail
- `useSuggestedStrategies()` - Wizard suggestions
- `useHasFetchedAllStrategies()` / `useHasFetchedUserStrategies()` - Loading state

### Parsing Layer (`common/utils/earn/parsing.tsx`)

Raw SDK data is transformed through parsing functions before hitting the UI:

1. `parseAllStrategies()` - Converts `SavedSdkStrategy[]` to `Strategy[]`, resolves tokens from tokenList
2. `parseUserStrategies()` - Converts `SavedSdkEarnPosition[]` to `EarnPosition[]`, maps history/balances
3. `parseUserStrategiesFinancialData()` - Aggregates user position data into financial metrics

Token resolution pattern:
```tsx
export const sdkStrategyTokenToToken = (
  sdkToken: SdkStrategyToken,
  tokenKey: TokenListId,
  tokenList: TokenList,
  chainId?: number
): Token => {
  const token = tokenList[tokenKey] || toToken({ ...sdkToken, chainId });
  return { ...token, price: sdkToken.price };
};
```

TokenListId format: `${chainId}-${tokenAddress}` (e.g. `10-0x...`)

### Strategy Parameters

Strategies are summarized into filter parameters:
```tsx
SummarizedSdkStrategyParameters = {
  protocols: string[];
  guardians: Record<string, Guardian>;
  tokens: { assets: Record<TokenListId, Token>; rewards: Record<TokenListId, Token> };
  networks: Record<ChainId, NetworkStruct>;
  yieldTypes: StrategyYieldType[];
}
```

This is computed once during `fetchAllStrategies()` and stored for filter UI.

---

## 3. State Management

### Redux Toolkit Slices

Two Redux slices manage earn-specific state:

**`earn-management`** - Form state for deposit/withdraw operations:
```tsx
interface EarnManagementState {
  asset?: Token;
  depositAmount?: string;
  depositAssetAmount?: string;
  depositAsset?: Token;
  withdrawAmount?: string;
  withdrawRewards: boolean;
  chainId?: number;
  triggerSteps?: boolean;
}
```

Actions: `setAsset`, `setDepositAmount`, `setWithdrawAmount`, `setWithdrawRewards`, `resetEarnForm`, `fullyResetEarnForm`, `setOneClickMigrationSettings`, `setTriggerSteps`

**`strategies-filters`** - Table filtering/sorting state per variant:
```tsx
type StrategiesFiltersState = Record<StrategiesTableVariants, {
  assets: Token[];
  networks: ChainId[];
  rewards: Token[];
  protocols: string[];
  yieldTypes: StrategyYieldType[];
  guardians: GuardianId[];
  search: string;
  orderBy: { column: StrategyColumnKeys; order: ColumnOrder };
  secondaryOrderBy?: { column: StrategyColumnKeys; order: ColumnOrder };
  tertiaryOrderBy?: { column: StrategyColumnKeys; order: ColumnOrder };
  quarterOrderBy?: { column: StrategyColumnKeys; order: ColumnOrder };
}>;
```

Sorting supports up to 4 levels of ordering (primary, secondary, tertiary, quarter).

### Service-Level State (EventsManager)

The `EarnService` extends `EventsManager<EarnServiceData>`, which provides a reactive state model outside Redux. Data flows via getters/setters that trigger events consumed by `useServiceEvents` hooks:

```tsx
set allStrategies(allStrategies) {
  this.serviceData = { ...this.serviceData, allStrategies };
}
```

This pattern keeps data-intensive state (strategies, positions) outside Redux for performance, while UI-specific state (form inputs, filters) stays in Redux.

---

## 4. Token/Amount Handling

### AmountsOfToken Pattern

All token amounts use a triple representation:

```tsx
type AmountsOfToken = {
  amount: bigint;           // Raw on-chain amount
  amountInUnits: string;    // Human-readable (formatted with decimals)
  amountInUSD: string;      // USD value as string
};
```

### Price Calculations

```tsx
// Convert token price to BigInt for precise math
parseNumberUsdPriceToBigInt(token.price)

// Calculate USD value from raw amount
parseUsdPrice(token, rawAmount, priceBigInt)

// Format for display
formatCurrencyAmount({ amount, token, intl })
```

### APY Handling

APY is stored as a percentage (e.g. 8 = 8%). Calculations must divide by 100:

```tsx
// Correct
(Number(assetBalance?.amount.amountInUSD) || 0) * period.annualRatio * (position.strategy.farm.apy / 100)

// Bug that was fixed (PR #1011)
// Wrong: used raw apy without /100
(Number(assetBalance?.amount.amountInUSD) || 0) * period.annualRatio * position.strategy.farm.apy
```

Total APY combines base APY and rewards APY:
```tsx
const totalApy = position.strategy.farm.apy + (position.strategy.farm.rewards?.apy || 0);
```

### Expected Returns Calculation

```tsx
const STRATEGY_RETURN_PERIODS = [
  { period: 'day',   annualRatio: 1/365 },
  { period: 'week',  annualRatio: 1/52 },
  { period: 'month', annualRatio: 1/12 },
  { period: 'year',  annualRatio: 1 },
];

// For each period:
expectedReturn = balanceUSD * annualRatio * (totalApy / 100)
```

### Fee Calculations

```tsx
export function calculateEarnFeeBigIntAmount({ strategy, feeType, assetAmount }) {
  const feePercentage = strategy?.guardian?.fees.find((fee) => fee.type === feeType)?.percentage;
  if (!feePercentage || isUndefined(assetAmount)) return undefined;
  const feePercentageBigInt = BigInt(Math.round(feePercentage * 100));
  return (assetAmount * feePercentageBigInt) / 100000n;
}
```

### Protocol Token / Wrapped Token Handling

A recurring pattern handles native protocol tokens vs their wrapped counterparts:

```tsx
const protocolToken = getProtocolToken(strategy.farm.chainId);
const wrappedProtocolToken = getWrappedProtocolToken(strategy.farm.chainId);

// When withdrawing: find by protocol token address when asset is wrapped
const addressToFind = isSameAddress(w.token.address, wrappedProtocolToken.address)
  ? protocolToken.address
  : w.token.address;
```

---

## 5. Transaction Flows

### Multi-Step Transaction Pattern

Deposit and withdraw follow a multi-step pattern with these possible steps:

**Deposit steps:**
1. Sign ToS (Terms of Service) - only for new positions
2. Token approval (ERC20 approve)
3. Companion signature (for wrapped protocol tokens on increase)
4. Deposit transaction

**Withdraw steps:**
1. Companion signature (for wrapped protocol tokens)
2. Withdraw transaction

Steps are built dynamically via `buildSteps()`:

```tsx
const buildSteps = (isApproved?: boolean) => {
  const newSteps: TransactionStep[] = [];

  if (!isIncrease && strategy?.tos) {
    newSteps.push({ hash: '', onAction: onSignEarnToS, type: SIGN_TOS_ACTION, ... });
  }
  if (!isApproved && asset?.address !== PROTOCOL_TOKEN_ADDRESS) {
    newSteps.push({ hash: '', onAction: onApproveToken, type: APPROVE_ACTION, ... });
  }
  if (requiresCompanionSignature) {
    newSteps.push({ hash: '', onAction: onSignCompanionApproval, type: SIGN_COMPANION_ACTION, ... });
  }
  newSteps.push({ hash: '', onAction: onDeposit, type: DEPOSIT_ACTION, ... });

  return newSteps;
};
```

### Transaction Type Data

Each earn transaction type has a typed data structure:

```tsx
interface EarnCreateTypeData {
  type: TransactionTypes.earnCreate;
  typeData: { asset: Token; assetAmount: string; strategyId: StrategyId; };
}

interface EarnIncreaseTypeData {
  type: TransactionTypes.earnIncrease;
  typeData: { asset: Token; assetAmount: string; positionId: SdkEarnPositionId; strategyId: StrategyId; signedPermit: boolean; };
}

interface EarnWithdrawTypeData {
  type: TransactionTypes.earnWithdraw;
  typeData: { positionId: SdkEarnPositionId; strategyId: StrategyId; signedPermit: boolean; withdrawn: { token: Token; amount: string; }[]; };
}

interface EarnClaimDelayedWithdrawTypeData {
  type: TransactionTypes.earnClaimDelayedWithdraw;
  typeData: { strategyId: StrategyId; positionId: SdkEarnPositionId; claim: Token; withdrawn: string; };
}
```

### Transaction Modal Pattern

All earn transactions use the same modal flow:

```tsx
const [, setModalLoading, setModalError, setModalClosed] = useTransactionModal();

// 1. Show loading
setModalLoading({ content: <Typography>...</Typography> });

// 2. Track event
trackEvent('Earn - Deposit submitting', { token, chainId });

// 3. Execute
const result = await earnService.depositOrCreate(...);

// 4. Track success
trackEvent('Earn - Deposit submitted', { strategy, amount, amountInUsd });

// 5. Add to transaction list
addTransaction(result, typeData);

// 6. Close modal
setModalClosed({ content: '' });
```

### Virtual Events / Optimistic Updates

After a transaction is submitted, the EarnService applies "virtual events" to local state before the API catches up:

```tsx
// In handleTransaction():
switch (transactionType) {
  case TransactionTypes.earnCreate:
    // Create a new position locally from tx data
    const newUserStrategy = getNewEarnPositionFromTxTypeData({ ... });
    userStrategies.push({ ...newUserStrategy, pendingTransaction: transaction.hash });
    break;
  case TransactionTypes.earnIncrease:
    // Update existing position balance locally
    modifiedStrategy.balances = updatedBalances;
    modifiedStrategy.pendingTransaction = '';
    break;
}
```

The service also reconciles local virtual events with API data using `applyVirtualEventsToBalances()` and `applyNewerEventsToHistoricalBalances()`.

### Permission Management

Companion contract permissions are tracked per position:

```tsx
permissions: companionAddress ? { [companionAddress]: [EarnPermission.INCREASE] } : {}
```

After signing a permit, permissions are added locally:
```tsx
if (signedPermit) {
  const companionAddress = this.contractService.getEarnCompanionAddress(transaction.chainId);
  modifiedStrategy.permissions[companionAddress] = [EarnPermission.INCREASE];
}
```

---

## 6. Error Handling

### Custom Error Names

Earn-specific errors use a custom error naming system:

```tsx
export enum CustomTransactionErrorNames {
  ApiSignatureForEarnCreateTransaction = 'ApiSignatureForEarnCreateTransaction',
}

enum CustomTransactionErrorCodes {
  API_SIGNATURE_FOR_EARN_CREATE_TRANSACTION = 900000,
}
```

### Error Flow

The API signature error handler wraps the API call in try/catch and re-names the error:

```tsx
try {
  return await this.meanApiService.getEarnStrategySignature({ ... });
} catch (e) {
  e.name = CustomTransactionErrorNames.ApiSignatureForEarnCreateTransaction;
  throw e;
}
```

This is then caught by `getTransactionErrorCode()` in the transaction modal:

```tsx
case CustomTransactionErrorNames.ApiSignatureForEarnCreateTransaction as string:
  return CustomTransactionErrorCodes.API_SIGNATURE_FOR_EARN_CREATE_TRANSACTION;
```

And displayed with a user-friendly message:

```tsx
[CustomTransactionErrorCodes.API_SIGNATURE_FOR_EARN_CREATE_TRANSACTION]: (
  <FormattedMessage
    defaultMessage="There was an error generating the transaction data, if you have just upgraded your tier, please try again in a few minutes"
  />
)
```

### Error Tracking Pattern

```tsx
} catch (e) {
  if (shouldTrackError(e as Error)) {
    trackEvent('Earn - Deposit error', { fromSteps: !!transactionsToExecute?.length });
    void errorService.logError('Error depositing', JSON.stringify(e), {
      token: asset?.symbol,
      strategy: strategy.id,
    });
  }
  setModalError({
    content: <FormattedMessage description="..." defaultMessage="Error depositing" />,
    error: e as Error,
  });
}
```

### Transaction Error Display

The transaction modal has a two-tier error display:
1. Known errors: Mapped via `TRANSACTION_ERRORS[errorCode]` to user-friendly FormattedMessage
2. Unknown errors: Show raw `error.message` as fallback (fixed in PR #1276)

---

## 7. Repeated Patterns

### i18n with FormattedMessage

Every user-facing string uses `react-intl` FormattedMessage with `description` and `defaultMessage`:

```tsx
<FormattedMessage
  description="earn.strategy-details.vault-about.rewards"
  defaultMessage="Rewards"
/>
```

Description format: `area.component.element` (e.g. `earn.strategy-card.button`, `earn.all-strategies-table.column.rewards`)

For programmatic use, `defineMessage()` is used with `intl.formatMessage()`.

### Loading / Skeleton Pattern

Components handle loading states by checking if the strategy object is undefined:

```tsx
const isLoading = !strategy;
// Then:
{isLoading ? <Skeleton variant="text" width="6ch" /> : <ActualContent />}
```

Tables have dedicated skeleton components that render the same number of rows as `ROWS_PER_PAGE`.

### Color Access Pattern

Colors are always accessed through the `colors` helper from `ui-library`:

```tsx
color={({ palette: { mode } }) => colors[mode].typography.typo1}
// or
color={({ palette }) => colors[palette.mode].typography.typo2}
```

### Gradient Theme Extension

The MUI theme is extended with custom gradients for earn-specific UI:

```tsx
gradient: {
  earnWizard: 'linear-gradient(90deg, #270A50 0%, #16062D 100%)',
  rewards: 'linear-gradient(180deg, #473267 0%, #47326700 100%)',
  newsBanner: 'linear-gradient(286deg, #7A1BFF 1.4%, #0EECC1 113.11%)',
}
```

### Token Comparison

Token comparison always uses utility functions, never direct address comparison:

```tsx
isSameToken(balance.token, strategy.asset)
isSameAddress(w.token.address, wrappedProtocolToken.address)
getIsSameOrTokenEquivalent(filterToken, strategyReward)
```

### Analytics Tracking

Event tracking follows a consistent naming pattern:

```tsx
trackEvent('Earn - Deposit submitting', { token, chainId });
trackEvent('Earn - Deposit submitted', { strategy, amount, amountInUsd, isDeposit, isIncrease });
trackEvent('Earn - Sign ToS error', { fromSteps });
trackEvent('Earn - Sign companion error', { fromSteps });
trackEvent('Earn - Claim delayed withdraw submitted', { token, strategy, amountUsd });
```

Naming convention was normalized from `EARN - X` to `Earn - X` (PR #1222).

### CTA Button Pattern

CTA buttons follow a priority-based rendering pattern:

```tsx
let ButtonToShow = DefaultButton;

if (!activeWallet) {
  ButtonToShow = NoWalletButton;
} else if (activeWallet.status === WalletStatus.disconnected) {
  ButtonToShow = ReconnectWalletButton;
} else if (!isOnCorrectNetwork && !activeWallet?.canAutomaticallyChangeNetwork) {
  ButtonToShow = IncorrectNetworkButton;
} else if (cantFund) {
  ButtonToShow = NoFundsButton;
} else if (needsApproval || needsSignature) {
  ButtonToShow = ProceedButton;
} else {
  ButtonToShow = ActualActionButton;
}

return ButtonToShow;
```

### Wallet Selector in Forms

```tsx
<FormWalletSelector
  filter={(wallet) =>
    strategy?.userPositions?.some((pos) => pos.owner === wallet.address) || false
  }
  overrideUsdBalances={usdBalances}
/>
```

---

## 8. Evolution

### Phase 1: Initial Scaffold (PR #828, May 2024)

- Created `EarnService` with mock data (`ApiStrategy` type)
- Simple service: `allStrategies` + `isLoadingAllStrategies`
- `parseAllStrategies()` maps API data to display types
- `SdkService.getAllStrategies()` returned hardcoded mocks with `setTimeout`
- Service registered on `Web3Service` via `getEarnService()`

### Phase 2: Table + Data Display (PR #833, May 2024)

- Built `AllStrategiesTable` component with pagination
- Added `ApiEarnToken` type (tokens became objects with price, decimals, not just addresses)
- Added `StrategyRiskLevel`, safety icons
- Token resolution via tokenList with fallback to `toToken()`
- Yield type formatting moved to `defineMessage()` pattern

### Phase 3: SDK Integration (PRs #919-#978, Jul-Sep 2024)

- Replaced mock data with real `@balmy/sdk` (aliased as `@koaj/earn-sdk`)
- SDK version bumped multiple times (0.0.25 -> 0.0.26, etc.)
- Added user positions, historical balances, financial data
- Portfolio page with TVL graph using Recharts
- `parseUserStrategiesFinancialData()` added for earnings calculations
- Expected returns component with period-based projections

### Phase 4: Transaction Support (PRs #1011-#1045, Sep-Oct 2024)

- APY bug fix (divide by 100)
- Companion signatures for deposit/withdraw flows
- Multi-step transaction system
- Virtual events for optimistic UI updates
- Fee calculations extracted to utility functions

### Phase 5: Permission System + Error Handling (PRs #1093-#1178, Oct-Nov 2024)

- Signature-based permissions for companion contracts
- `signedPermit` tracking in transaction type data
- Custom error handling for API signature failures
- Promoted strategies with visual indicators
- Farm description localization via `defineMessage()` maps
- Virtual permission tracking (companion permissions set locally after signing)

### Phase 6: UI Polish + Rewards (PRs #1174-#1297, Nov 2024-Jan 2025)

- Rewards display component with gradient background and RainCoins SVG asset
- Balance/rewards info split for strategies with/without rewards
- Rewards pills in portfolio table
- `formatListMessage()` utility for "X, Y and Z" formatting
- Multiple sorting levels (up to 4)
- Tier-based strategy filtering

### Phase 7: Claim Rewards + Migration (PRs #1324-#1350, Feb-Mar 2025)

- Withdraw rewards card with dedicated CTA
- One-click migration from external vaults
- Network auto-switching for wallets that support it (non-WalletConnect)
- `canAutomaticallyChangeNetwork` on wallet display objects
- Moved form components from `WithdrawForm` into `WithdrawTransactionManager` for better composition

### Key Architectural Decisions Throughout

1. **Service-based state over Redux for data**: Large datasets (strategies, positions) kept in EventsManager, not Redux
2. **Redux for UI state only**: Form inputs, filters, sorting preferences
3. **Parsing layer as bridge**: SDK types never hit components directly; always parsed first
4. **Type-driven table**: Column configs with generic `StrategiesTableVariants` parameter
5. **SDK aliasing**: Used `@balmy/sdk` alias pointing to `@koaj/earn-sdk` during development
6. **Progressive enrichment**: Strategies start minimal and get detailed data on demand (`hasFetchedHistoricalData`)
7. **Virtual events**: Optimistic updates bridge the gap between tx submission and API indexing

---

## Key File Paths

- Service: `apps/root/src/services/earnService.ts`
- Parsing utilities: `apps/root/src/common/utils/earn/parsing.tsx`
- Sort utilities: `apps/root/src/common/utils/earn/sort.ts`
- Search utilities: `apps/root/src/common/utils/earn/search.ts`
- Hooks: `apps/root/src/hooks/earn/` (13 hooks)
- Constants: `apps/root/src/constants/earn.ts`
- Mocks: `apps/root/src/common/mocks/earn.ts`
- Redux - form state: `apps/root/src/state/earn-management/`
- Redux - filters: `apps/root/src/state/strategies-filters/`
- Pages:
  - Home: `apps/root/src/pages/earn/home/`
  - Portfolio: `apps/root/src/pages/earn/portfolio/`
  - Strategy detail: `apps/root/src/pages/strategy-guardian-detail/`
- Shared table: `apps/root/src/pages/earn/components/strategies-table/`
- Strategy card: `apps/root/src/pages/earn/components/strategy-card-item/`
- Types: `packages/common-types/src/earn.ts`
- Tests: `apps/root/src/services/earnService.spec.ts`
