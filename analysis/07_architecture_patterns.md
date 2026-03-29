# Architecture Patterns - Web3 Frontend

## 1. Project Structure

### Monorepo with Turborepo

The project is a Yarn workspaces monorepo managed by Turborepo.

```
web3-frontend/
  apps/
    root/                    # Main application (webpack, React 18)
  packages/
    common-types/            # Shared TypeScript types (built with tsup)
    ui-library/              # Shared UI components, theme, icons
    eslint-config-custom/    # Shared ESLint config
    tsconfig/                # Shared TypeScript config
  lang/                      # i18n source files (en, es, tr)
  package.json               # Root: turbo scripts, @balmy/sdk dependency
  turbo.json                 # Task pipeline config
```

Root `package.json` holds `@balmy/sdk` as the single production dependency at the monorepo level. All other deps live in `apps/root/package.json`.

### Application Directory Layout (`apps/root/src/`)

```
src/
  index.tsx                  # Bootstrap: creates Web3Service, Redux store, renders App
  frame/                     # App shell: Router, providers, polling handlers
    components/
      navigation/            # Top-level nav
      promises-initializer/  # Runs initial data fetches on mount
      background-grid/       # Visual background
      referred-by-handler/   # Referral tracking
    polling-handlers/        # Headless components that poll for updates
      earn-positions-updater/
      tier-updater/
  pages/                     # Route-level components (lazy loaded)
    home/                    # Dashboard
    dca/                     # DCA position creation and list
    earn/                    # Earn home + portfolio
    aggregator/              # Token swap
    transfer/                # Token transfer
    history/                 # Transaction history
    position-detail/         # Individual DCA position
    strategy-guardian-detail/# Earn strategy details
    token-profile/           # Token info page
    tier-view/               # User tier info
  services/                  # Business logic layer (class-based)
  hooks/                     # React hooks (100+ files)
    earn/                    # Earn-specific hooks
    tiers/                   # Tier-specific hooks
    state.ts                 # Re-exports useAppDispatch, useAppSelector
  state/                     # Redux store slices
  common/
    components/              # Shared React components
    utils/                   # Utility functions
    mocks/                   # Mock data (tokens, etc.)
  constants/                 # App-wide constants
  types/                     # App-level type re-exports
  abis/                      # Contract ABIs
  assets/                    # Static assets
  lang/                      # Compiled i18n messages
```

### Module Boundaries

Path aliases defined in `webpack.common.js`:
- `@common` -> `src/common`
- `@constants` -> `src/constants`
- `@hooks` -> `src/hooks`
- `@services` -> `src/services`
- `@state` -> `src/state`
- `@pages` -> `src/pages`
- `@types` -> `src/types`
- `@abis` -> `src/abis`
- `@assets` -> `src/assets`
- `@frame` -> `src/frame`
- `@lang` -> `src/lang`
- `@fonts` -> `src/fonts`

Cross-package references: `apps/root` imports from `common-types` (shared types) and `ui-library` (shared UI). These are resolved via workspace `*` version references.

---

## 2. State Management

### Redux Toolkit (Primary Global State)

The store is configured in `apps/root/src/state/index.ts` using `@reduxjs/toolkit`'s `configureStore`.

**Store slices (16 total):**
- `transactions` - Pending/completed transaction tracking
- `initializer` - App initialization flags
- `tokenLists` - Token metadata from external lists + custom tokens
- `createPosition` - DCA position creation form state
- `aggregator` - Swap form state (from/to/values/selected route)
- `config` - User preferences (theme, locale, show balances, etc.)
- `tabs` - UI tab selections
- `positionPermissions` - DCA position permissions
- `modifyRateSettings` - DCA rate modification form
- `error` - Global error state
- `positionDetails` - Position detail page state
- `aggregatorSettings` - Swap settings (slippage, gas, timeout, etc.)
- `transfer` - Transfer form state
- `balances` - Token balances by chain/wallet
- `strategiesFilters` - Earn strategy filter state
- `earnManagement` - Earn management form state

**Typed hooks:**
```typescript
// apps/root/src/state/hooks.ts
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

**Async thunks** use a typed wrapper:
```typescript
// apps/root/src/state/createAppAsyncThunk.ts
export const createAppAsyncThunk = createAsyncThunk.withTypes<{
  extra: ExtraArgument;
  state: RootState;
  dispatch: AppDispatch;
}>();
```

The `ExtraArgument` passed to thunks contains `{ web3Service, axiosClient }`, giving thunks access to all services.

**Redux reducers** use `createReducer` with the builder callback pattern (not `createSlice`). Actions are defined separately in `actions.ts` files using `createAction` and `createAppAsyncThunk`.

### Persistence

Custom localStorage persistence (not redux-persist). Implemented in `state/utils/persistor.js` with a `save` middleware and `load` preloaded state function.

Persisted state paths:
- `transactions`
- `aggregatorSettings`
- `tokenLists.customTokens`
- `config.selectedLocale`
- `config.theme`
- `config.showSmallBalances`
- `config.showBalances`
- `config.switchActiveWalletOnConnection`

A **listener middleware** watches for specific "saved actions" and pushes config updates to the backend via `accountService.updateUserConfig()`.

**Storage versioning:** On app load, `checkStorageValidity()` compares version constants against localStorage. When versions mismatch, it selectively clears storage while preserving transaction data.

### Service-Level State (EventsManager Pattern)

Services that hold their own state extend `EventsManager<T>`:

```typescript
export class EventsManager<T> {
  private _serviceData: T;
  private eventCallbacks: Record<string, EventCallback>;

  set serviceData(newServiceData: T) {
    this._serviceData = newServiceData;
    Object.values(this.eventCallbacks).forEach((callback) => callback());
  }

  setCallback(id: string, callback: EventCallback): void { ... }
  removeCallback(callbackId: string): void { ... }
}
```

React components subscribe via the `useServiceEvents` hook:

```typescript
function useServiceEvents<ServiceData, ServiceType extends EventsManager<ServiceData>, Key extends keyof ServiceType>(
  service: ServiceType,
  property: Key
) {
  const [serviceData, setServiceData] = useState(getServicePropertyData());
  useEffect(() => {
    const callbackId = uuidv4();
    service.setCallback(callbackId, () => setServiceData(getServicePropertyData()));
    return () => service.removeCallback(callbackId);
  }, []);
  return serviceData;
}
```

This is a pub/sub system: services hold state, notify subscribers on change, and React components re-render via `useState` updates. Used by: `AccountService`, `LabelService`, `ContactListService`, `PairService`, `EarnService`, `PositionService`, and others.

### React Context

Used sparingly for cross-cutting concerns:
- `WalletContext` - Provides `web3Service` and `axiosClient` to the entire app
- `LanguageContext` - Current language and change handler
- `TransactionModalProvider` - Transaction confirmation modal state

### React Query (QueryClient)

Present but **intentionally neutered** for RainbowKit compatibility:

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      enabled: false,
      refetchOnWindowFocus: false,
      retryOnMount: false,
      staleTime: Infinity,
      retry: false,
    },
  },
});
```

The app does NOT use React Query for its own data fetching. This QueryClient exists solely because `@rainbow-me/rainbowkit` requires a `QueryClientProvider`.

---

## 3. Data Fetching Patterns

### SDK-Based Data Access

The primary data source is `@balmy/sdk`, initialized in `SdkService`. The SDK handles:
- Quote fetching from multiple DEX aggregators (via batch API proxied through MEAN_API_URL)
- DCA position operations
- Earn vault operations
- Price data (via cached DefiLlama source)
- Token balances (multicall)
- Permit2 operations

SDK configuration includes:
- **Provider fallback**: HTTP RPC sources with public RPC load-balancing
- **Price caching**: 1-minute cache with 5-minute stale fallback, 20,000 entry max
- **Quote routing**: Overridable source list pattern where quotes go through the project's API (`batch-api` type)

### REST API via Axios

`MeanApiService` makes REST calls to `MEAN_API_URL` for DCA-specific actions (swap-and-deposit, withdraw-and-swap, terminate-and-swap, etc.).

**Axios caching** via `axios-cache-adapter`:
- 15-minute default max age
- Excluded from cache: PUT, PATCH, DELETE methods, plus paths matching `accounts`, `balances`, `history`

### Async Thunks for Redux State

Data fetching that updates Redux goes through `createAppAsyncThunk`:

```typescript
export const fetchWalletBalancesForChain = createAppAsyncThunk<
  { chainId: number; tokenBalances: TokenBalancesAndPrices; walletAddress: string },
  { tokenList: TokenList; chainId: number; walletAddress: string }
>('balances/fetchWalletBalancesForChain', async ({ tokenList, chainId, walletAddress }, { extra: { web3Service } }) => {
  const sdkService = web3Service.getSdkService();
  // ...fetch and return
});
```

Thunks access services through `extra.web3Service`, which is the central service container.

### Polling

Polling is implemented through **headless updater components** rendered as part of `PollingHandlers`:
- `BalancesUpdater` - Polls balance data
- `TransactionUpdater` - Checks pending transaction status via `useInterval`
- `NetworkUpdater` - Watches network changes
- `EarnPositionsUpdater` - Polls earn position data
- `TierUpdater` - Polls tier information

The `useInterval` hook wraps `setInterval` with a ref-based callback pattern.

### Initialization

`PromisesInitializer` runs on mount after token lists load:
- Fetches initial balances across all chains
- Loads contacts, labels, positions
- Processes confirmed transactions
- Uses `timeoutPromise` from `@balmy/sdk` for request timeouts
- Shows snackbar errors for API failures with a refresh button

---

## 4. Hook Patterns

### Naming Conventions

100+ hooks in `apps/root/src/hooks/`:

- **Service accessor hooks**: `useWeb3Service`, `useSdkService`, `useAggregatorService`, `usePositionService`, `usePriceService`, etc. These access service instances via `WalletContext`.
- **Data hooks**: `useToken`, `useTokenList`, `useCurrentPositions`, `useYieldOptions`, `useNetWorth`, `useWallets`, `useActiveWallet`
- **State selector hooks**: `useThemeMode`, `useSelectedLocale`, `useShowBalances` (thin wrappers around `useAppSelector`)
- **Action hooks**: `useAnalytics`, `useEditLabel`, `useChangeLanguage`, `usePushToHistory`, `useAddCustomTokenToList`
- **UI hooks**: `useCurrentBreakpoint`, `useCountingAnimation`, `useInfiniteLoading`, `useDebounce`, `useTimeout`, `usePrevious`
- **Domain hooks in subdirectories**: `hooks/earn/useEarnPositions`, `hooks/tiers/useTierService`

### Service Accessor Pattern

All service hooks follow the same pattern, going through `WalletContext`:

```typescript
function useWeb3Service() {
  const context = React.useContext(WalletContext);
  return context.web3Service;
}

function useSdkService() {
  const web3Service = useWeb3Service();
  return web3Service.getSdkService();
}
```

### Hook Composition

The `useAnalytics` hook wraps every analytics method in `useCallback` and returns a memoized object:

```typescript
function useAnalytics() {
  const analyticsService = useAnalyticsService();
  const trackEvent = useCallback((action, extraData) => {
    try { void analyticsService.trackEvent(action, extraData); } catch {}
  }, [analyticsService]);
  // ...12 more useCallback wrappers
  return useMemo(() => ({ trackEvent, ... }), [...]);
}
```

The `useToken` hook composes `useTokenList`, `useAppDispatch`, and `useAllBalances`, with multiple `useMemo` layers for token resolution (by ID, address, symbol).

---

## 5. Type System

### Type Organization

- `packages/common-types/src/` - Barrel-exports from domain-specific files:
  - `tokens.ts`, `positions.ts`, `pairs.ts`, `responses.ts`, `transactions.ts`, `contracts.ts`, `yield.ts`, `aggregator.ts`, `sdk.ts`, `campaigns.ts`, `account.ts`, `contactList.ts`, `accountLabels.ts`, `accountHistory.ts`, `providerInfo.ts`, `earn.ts`, `tiers.ts`
- `apps/root/src/types/index.ts` - Re-exports everything from `common-types` plus `./services`
- `apps/root/src/types/services.ts` - Service-specific types

### Key Type Patterns

**Branded/nominal types:**
```typescript
export type ChainId = number;
export type Address = `0x${string}`;
export type TokenAddress = string;
export type AmountOfToken = string;
export type Timestamp = number;
export type TokenListId = `${number}-${string}`;  // "chainId-address"
```

**Discriminated union types** for transaction actions:
```typescript
export type TransactionActionType =
  | TransactionActionApproveTokenType
  | TransactionActionSwapType
  | TransactionActionCreatePositionType
  | TransactionActionEarnDepositType
  // ... 10+ more types
```

**Service data generics** via `EventsManager<T>` where `T` defines the shape of each service's internal state.

**Redux state types** are inferred from the store:
```typescript
export type StoreType = ReturnType<typeof createStore>;
export type RootState = ReturnType<StoreType['getState']>;
export type AppDispatch = StoreType['dispatch'];
```

### SDK Types

Types from `@balmy/sdk` are used directly: `EstimatedQuoteRequest`, `QuoteResponse`, `SourceId`, `SOURCES_METADATA`, `PermitData`, `QuoteTransaction`, `DCAPermission`, `EarnPermission`, etc.

---

## 6. Error Handling

### ErrorBoundary (Class Component)

`apps/root/src/common/components/error-boundary/indext.tsx` (note: filename typo is intentional in codebase).

- Class component wrapping all routed content
- Connected to Redux (`error` slice) via `connect()`
- Uses `getDerivedStateFromError` and `componentDidCatch`
- Logs errors to `ErrorService`
- Displays user-facing error UI with:
  - Error name and message
  - Discord link for reporting
  - "Copy Error Log" button (copies JSON with error context including page/route info)
  - "Reload Page" button
- Route-aware error context: extracts page name + params from URL for error reporting

### Redux Error State

Global error state in `state/error/reducer.ts` with `setError` action. The ErrorBoundary reads from both its own component state and the Redux error state.

### Service-Level Error Handling

- `ErrorService` logs errors to backend
- `try/catch` blocks in analytics calls silently swallow errors (fire-and-forget pattern)
- `timeoutPromise` wraps slow API calls with configurable timeouts (e.g., `'30s'` for legacy subgraph queries, `'5s'` default for quotes)
- `PromisesInitializer` handles API errors with snackbar notifications keyed to `ApiErrorKeys`

### Transaction Error Handling

- Transaction receipts handle missing fields defensively: `(receipt.gasUsed || 0).toString()`
- Transaction updater retries with `getMaxTransactionRetries()` configuration

---

## 7. Routing

### React Router v6

Configured in `apps/root/src/frame/index.tsx` with `BrowserRouter`.

**Route structure:**
```
/                           -> Home (dashboard)
/home, /dashboard           -> Home
/swap/:chainId?/:from?/:to? -> Aggregator (swap)
/invest/create/:chainId?/:from?/:to? -> DCA creation
/invest/positions            -> DCA position list
/invest/positions/:positionId -> DCA position detail
/invest/positions/:chainId/:positionVersion/:positionId -> DCA position detail (legacy)
/earn                        -> Earn discovery
/earn/:assetTokenId?/:rewardTokenId? -> Earn filtered
/earn/portfolio              -> Earn portfolio
/earn/vaults/:chainId/:strategyGuardianId -> Strategy detail
/transfer/:chainId?/:token?/:recipient? -> Transfer
/history                     -> Transaction history
/token/:tokenListId          -> Token profile
/tier-view                   -> Tier information
*                            -> Home (fallback)
```

**Legacy route redirects:** `RedirectOldRoute` component handles old URL patterns (e.g., `/positions/*` -> `/invest/positions/*`, `/create/*` -> `/invest/create/*`).

**Route constants** defined in `apps/root/src/constants/routes.tsx` with i18n labels and icons.

### Code Splitting

All page components are lazy-loaded:
```typescript
const Home = lazy(() => import('@pages/home'));
const DCA = lazy(() => import('@pages/dca'));
// ... etc
```

Wrapped in `<Suspense fallback={<CenteredLoadingIndicator />}>`.

---

## 8. Dependency Patterns

### Key Libraries

| Library | Purpose |
|---------|---------|
| `@balmy/sdk` | Core SDK for DCA, earn, quotes, prices, balances |
| `@reduxjs/toolkit` | State management |
| `react-redux` | Redux React bindings |
| `wagmi` + `viem` | Wallet connection, chain interaction, address/tx utilities |
| `@rainbow-me/rainbowkit` | Wallet connection UI |
| `@tanstack/react-query` | Required by RainbowKit (not used for app data) |
| `react-router-dom` v6 | Client-side routing |
| `react-intl` / `@formatjs` | i18n (ICU message format) |
| `styled-components` | Component styling (with TypeScript plugin) |
| `ui-library` (internal) | Wraps MUI with custom theme, shared components |
| `axios` + `axios-cache-adapter` | HTTP client with caching |
| `mixpanel-browser` | Analytics event tracking |
| `@hotjar/browser` | Session recording and heatmaps |
| `lodash` | Utility functions (tree-shakeable imports like `lodash/find`) |
| `luxon` | Date/time handling |
| `recharts` | Charts |
| `react-window` + `react-virtualized-auto-sizer` | Virtualized lists (token picker) |
| `react-virtuoso` | Another virtualized list component |
| `canvas-confetti` | Celebration animations |
| `@gnosis.pm/safe-apps-sdk` / `@safe-global/safe-apps-sdk` | Gnosis Safe integration |
| `ethers` (legacy) -> `viem` (current) | BigNumber migration happened over time |

### Service Architecture (Dependency Injection)

`Web3Service` is the **central service container**, instantiated once at bootstrap. It creates all other services with explicit constructor injection:

```
Web3Service
  -> SafeService
  -> MeanApiService (axiosClient)
  -> WalletClientsService (web3Service)
  -> TierService (web3Service, meanApiService)
  -> AccountService (web3Service, meanApiService, walletClientsService, tierService)
  -> SdkService (axiosClient)
  -> ProviderService (accountService, sdkService, walletClientsService)
  -> ContractService (providerService)
  -> EarnService (sdkService, accountService, providerService, contractService, meanApiService)
  -> LabelService (meanApiService, accountService, providerService, contractService)
  -> WalletService (contractService, providerService)
  -> ContactListService (accountService, ...)
  -> PriceService (sdkService, ...)
  -> PositionService (...)
  -> AggregatorService (...)
  -> TransactionService (...)
  -> AnalyticsService (providerService, accountService)
  -> SimulationService (...)
  -> Permit2Service (...)
  -> CampaignService (...)
  -> ErrorService (...)
```

Services are class-based singletons living for the app's lifetime. They're accessed in React through Context (`WalletContext` provides `web3Service`) and getter hooks like `useWeb3Service()`.

---

## 9. Build & Configuration

### Build Tool: Webpack 5

- `webpack.common.js` - Shared config (entry, resolve, loaders, plugins)
- `webpack.dev.js` - Dev server config
- `webpack.prod.js` - Production config with TerserPlugin

**Loaders:**
- `ts-loader` with `@formatjs/ts-transformer` for i18n extraction + `typescript-plugin-styled-components`
- `babel-loader` for JS
- `@svgr/webpack` for SVG-as-components
- `url-loader` for fonts (inline < 50KB)
- `file-loader` for images

**Plugins:**
- `HtmlWebpackPlugin` - HTML template
- `CopyPlugin` - Static metadata files
- `DefinePlugin` - Environment variable injection
- `WebpackBar` - Build progress
- `ForkTsCheckerWebpackPlugin` - Parallel type checking (4GB memory limit)
- `webpack.ProvidePlugin` - Buffer and process polyfills

**Node.js polyfills** for browser: stream, crypto, assert, http, https, os, url.

### Environment Variables

Injected at build time via `webpack.DefinePlugin`:
- `ETHPLORER_KEY` - Ethplorer API key
- `ETHERSCAN_API` - Etherscan API key
- `MIXPANEL_TOKEN` - Analytics
- `WC_PROJECT_ID` - WalletConnect project ID
- `MEAN_API_URL` - Backend API base URL
- `TOKEN_LIST_URL` - Token list source
- `ENABLED_TRANSLATIONS` - Available language codes
- `HOTJAR_PAGE_ID` - Hotjar tracking

### Feature Flags

Minimal: `apps/root/src/constants/flags.ts` exports only `FAIL_ON_ERROR = false`.

Feature toggling is done more through configuration constants (e.g., `DCA_TOKEN_BLACKLIST`, `DISABLED_YIELD_WITHDRAWS`, source lists in `SdkService`) rather than a feature flag system.

### i18n

- `react-intl` with `@formatjs` tooling
- Source strings extracted from TSX via `formatjs extract`
- Translations managed through Localazy (`localazy.json`)
- Languages: English, Spanish, Turkish
- Message IDs are content-hash based: `[sha512:contenthash:base64:6]`

### Testing

- Jest with `jest-environment-jsdom`
- `ts-jest` for TypeScript
- `@inrupt/jest-jsdom-polyfills` for browser API polyfills
- Service test files co-located: `*.spec.ts` next to source files
- `apps/root/tests/` directory for integration tests

---

## 10. Analytics Integration

### Mixpanel + Hotjar

`AnalyticsService` initializes both on construction:

```typescript
constructor(providerService, accountService) {
  this.mixpanel = MixpanelLibrary.init(process.env.MIXPANEL_TOKEN, { api_host: MEAN_PROXY_PANEL_URL }, ' ');
  this.mixpanel.set_config({ persistence: 'localStorage', ignore_dnt: true });
  Hotjar.init(Number(process.env.HOTJAR_PAGE_ID), 6);
}
```

Mixpanel events are proxied through `MEAN_PROXY_PANEL_URL` (first-party proxy to avoid ad blockers).

### Event Tracking Pattern

Every tracked event includes context:
- `chainId` and `chainName` from the current network
- `distinct_id` from the user account
- `activeWallet` address

The `useAnalytics` hook exposes typed, memoized tracking functions:
- `trackEvent(action, extraData)` - Generic events
- `trackSwap(...)` - Swap-specific with token details, amounts, source
- `trackDcaCreatePosition(...)` - DCA creation with frequencies, yield options
- `trackEarnDeposit(...)` / `trackEarnWithdraw(...)` - Earn operations
- `trackTransfer(...)` - Transfer events
- `trackPositionModified(...)` / `trackPositionTerminated(...)` - Position lifecycle
- `trackSlippageChanged(...)`, `trackGasSpeedChanged(...)`, etc. - Settings changes

All analytics calls are fire-and-forget with `try/catch` swallowing errors.

### User Identification

- `identifyUser(userId)` called on login, identifies in both Mixpanel and Hotjar
- Error boundary captures page context for error tracking (parses route to determine which feature page errored)
- `PromisesInitializer` tracks API failures: `trackEvent('Api initial request failed', { requestType, timeouted })`

---

## 11. Performance Patterns

### Code Splitting & Lazy Loading

All 11 page components use `React.lazy()` with dynamic imports. Wrapped in `Suspense` with a loading spinner fallback.

### Memoization

- `React.useMemo` used in hooks returning derived data (e.g., `useYieldOptions`, `useToken`, `useAnalytics`)
- `React.useCallback` wraps event handlers passed as props
- Rate computation moved from `useState` + `useEffect` to derived `useMemo` values to avoid unnecessary re-renders (PR #209 pattern)

### Virtualized Lists

Token picker uses `react-window` with `react-virtualized-auto-sizer` for rendering large token lists without DOM overhead.

### QueryClient Optimization

RainbowKit's QueryClient is configured with all queries disabled, no refetch on window focus, no retry, and infinite stale time to prevent unnecessary background requests.

### Caching Layers

1. **Axios cache adapter** - 15-minute HTTP response cache (excludes dynamic endpoints)
2. **SDK price cache** - 1-minute with 5-minute stale fallback, 20,000-entry LRU
3. **localStorage persistence** - Redux state survives page reloads
4. **Service-level memoization** - Label service creates new object references only on actual changes (`{ ...this.labels }` spread pattern, PR #1282)

### Polling Control

Updater components use `useInterval` with configurable delays. Balance fetching is chain-aware and wallet-aware, only updating balances for the active wallet while preserving other wallet data.

---

## 12. API Layer

### SDK Integration (`@balmy/sdk`)

The SDK is the primary API abstraction. `SdkService` wraps it and configures:

**Quote Sources** (via the project's batch API proxy):
- rango, changelly, 0x, 1inch, uniswap, portals-fi, dodo, bebop, enso, barter, squid, okx-dex, sovryn, balmy, magpie

**Provider Configuration:**
- Fallback strategy: Rootstock-specific HTTP RPCs -> public RPCs with load-balancing
- Evolution: `prioritized` -> `fallback` provider strategy (PR #984)

**Price Source:**
- Cached DefiLlama (simplified from a prioritized stack of coingecko -> balmy -> defi-llama)
- Cache config: 20,000 max entries (increased from 20 in PR #798)

### REST API (`MeanApiService`)

Direct API calls to `MEAN_API_URL/v1/` for operations the SDK doesn't cover:
- DCA operations: swap-and-deposit, withdraw-and-swap, terminate-and-swap, swap-and-increase, reduce-and-swap, swap-and-migrate
- Account management, contacts, labels
- Transaction simulation (Blowfish)

### Request Patterns

- **Timeout wrapping**: `timeoutPromise(promise, '30s')` for legacy subgraph queries
- **Default quote timeout**: 5 seconds (`quoteTimeout=5s` URL parameter)
- **Batch vs single quotes**: Moved from batch-api to single api, then back to batch-api with `buildTxURI` support
- **Address validation**: Regex check before custom token lookups (`/^0x[A-Fa-f0-9]{40}$/`)
- **Permit2 support**: Conditional flow - if `usePermit2` flag is set, quotes go through `sdk.permit2Service.quotes.estimateAllQuotes`

### Data Flow Summary

```
User Action
  -> React Component
    -> useAnalytics().trackEvent() (fire-and-forget)
    -> dispatch(asyncThunk)
      -> thunk accesses web3Service via extra argument
        -> web3Service.getXxxService().method()
          -> SdkService.sdk.xxxService.method()  (or)
          -> MeanApiService.axiosClient.post()
        -> returns data
      -> reducer updates Redux state
    -> useAppSelector() picks up new state
  -> Component re-renders
```

For service-level state (not in Redux):
```
Service method updates this.serviceData
  -> EventsManager triggers all registered callbacks
    -> useServiceEvents hook's callback calls setState
      -> Component re-renders with new data
```
