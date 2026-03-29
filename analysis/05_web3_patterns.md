# Web3 Patterns - Web3 Frontend

Extracted from PR diffs (wallet, chains, tokens, transactions areas) and current codebase source.

---

## 1. Wallet Connection

### Library Stack
- **RainbowKit** (`@rainbow-me/rainbowkit`) for the connection modal UI
- **Wagmi** for React hooks and wallet client management
- **Viem** (`WalletClient`, `PublicClient`) as the underlying Ethereum library (replaced ethers.js)

### Connection Flow
1. User clicks connect -> `useOpenConnectModal` hook is called
2. Hook calls `walletClientService.disconnect()` first (clears previous state)
3. Opens RainbowKit's `openConnectModal()` from `useConnectModal()`
4. After wallet connects, `WalletClientsService` detects the new connection via wagmi's connection events
5. `WalletClientsService.updateProviders()` maps connections to `AvailableProvider` records keyed by address
6. `AccountService.updateWallets()` is called with the provider map and the affected wallet address
7. Based on `WalletActionType`, either `logInUser()` or `linkWallet()` is triggered

### Wallet Action Types
```typescript
enum WalletActionType {
  link = 'link',        // Add wallet to existing account
  connect = 'connect',  // Log in / create account
  none = 'none',
  reconnect = 'reconnect', // Re-establish connection to correct wallet
}
```

### Login Signature
- On first login, `AccountService` prompts a `signMessage` (not a transaction, no gas)
- The message includes Terms of Use and Privacy Policy links
- Signature is stored in localStorage under `wallet_auth_signature`
- Signature version is tracked (`LATEST_SIGNATURE_VERSION = '1.0.2'`) to force re-signing when the message changes
- A 100ms timeout is added before `signMessage` to work around a Rabby wallet race condition

### Multi-Wallet Support
- Users can link multiple wallets to a single account
- `AccountService` maintains a `User` with an array of `Wallet` objects
- Each wallet has: `address`, `status` (WalletStatus), `type` (WalletType), `walletClient`, `providerInfo`, `isAuth`
- Active wallet is tracked separately via `activeWallet` (an Address)
- `setActiveWallet()` validates the wallet exists in the user's wallet list before setting it
- Wallets are sorted: labeled/ENS wallets first (alphabetically), then by address

### Gnosis Safe Support
- `useLoadedAsSafeApp` hook detects if app is running inside Gnosis Safe
- When loaded as Safe app, chain switching is disabled (Safe controls the chain)
- Block watching uses polling (`setInterval` every 10s) instead of websocket subscriptions
- Safe transaction hashes are resolved to real tx hashes via `safeService`

### Key Files
- `/apps/root/src/hooks/useOpenConnectModal.ts` - Connection modal logic
- `/apps/root/src/services/accountService.ts` - User/wallet state management
- `/apps/root/src/services/walletClientsService.ts` - Wagmi connection tracking
- `/apps/root/src/services/providerService.ts` - Signer/provider resolution

---

## 2. Chain Management

### Multi-Chain Architecture
- Chains come from `@balmy/sdk` (`Chains` object, `getAllChains()`)
- Local `NETWORKS` record maps chainId to `NetworkStruct` with name, RPC, nativeCurrency, wToken, explorer
- Separate supported chain lists per feature:
  - `SUPPORTED_NETWORKS_DCA` - Chains where DCA is available
  - `SUPPORTED_NETWORKS_EARN` - Chains where Earn vaults exist
  - `AGGREGATOR_SUPPORTED_CHAINS` - Chains for swap aggregator
- Chains can be disabled by removing from these arrays (e.g., Mantle was disabled)
- Testnets are tracked in `TESTNETS` array and filtered out in production contexts

### Chain Switching
```typescript
// WalletService.changeNetwork() flow:
async changeNetwork(newChainId, address?, callbackBeforeReload?) {
  const currentNetwork = await providerService.getNetwork(address);
  if (currentNetwork?.chainId !== newChainId) {
    await providerService.attempToAutomaticallyChangeNetwork(newChainId, ...);
  } else if (callbackBeforeReload) {
    callbackBeforeReload(); // Important: callback runs even when already on correct chain
  }
}
```

- `ProviderService.changeNetwork()` calls `wallet_switchEthereumChain` RPC
- If chain doesn't exist (error code 4902), falls back to `wallet_addEthereumChain` with chain params
- `NON_AUTOMATIC_CHAIN_CHANGING_WALLETS` - Some wallets don't support automatic chain switching
- `CHAIN_CHANGING_WALLETS_WITH_REFRESH` - Some wallets require page reload after chain change
- When not connected to a supported network, defaults to Optimism

### Chain-Specific Logic
- `TRANSACTION_RETRIES_PER_NETWORK` - Different retry counts per chain (Rootstock gets 90 retries vs default 10)
- Explorer URLs are chain-specific, built from SDK chain data
- Token lists are organized by chain, with chain-specific parsers (e.g., 1inch Gnosis list filters out RealToken entries)

### RPC Configuration
```typescript
// SDK provider config uses fallback strategy:
provider: {
  source: {
    type: 'fallback',  // Changed from 'prioritized' to 'fallback'
    sources: [
      { type: 'http', supportedChains: [Chains.ROOTSTOCK.chainId], url: '...' },
      { type: 'public-rpcs', config: { type: 'load-balance', maxAttempts: 2, maxConcurrent: 2 } },
    ],
  },
}
```

### Key Files
- `/apps/root/src/constants/addresses.ts` - Chain lists, contract addresses per chain
- `/apps/root/src/constants/aggregator.ts` - Aggregator chain whitelist
- `/apps/root/src/services/providerService.ts` - Chain switching, network addition

---

## 3. Token Handling

### Token Type System
```typescript
enum TokenType {
  BASE = 'BASE',                          // Native token (ETH, MATIC, etc.)
  NATIVE = 'NATIVE',                      // Alias for native
  ERC20_TOKEN = 'ERC20_TOKEN',
  WRAPPED_PROTOCOL_TOKEN = 'WRAPPED_PROTOCOL_TOKEN',
  YIELD_BEARING_SHARE = 'YIELD_BEARING_SHARE',
  ERC721_TOKEN = 'ERC721_TOKEN',
}
```

### Protocol Token (Native) Pattern
- All native tokens use a sentinel address: `PROTOCOL_TOKEN_ADDRESS = '0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee'`
- A companion address variant exists: `ETH_COMPANION_ADDRESS = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE'` (mixed case)
- Each chain has its own native token factory function (ETH, MATIC, BNB, AVAX, RBTC, etc.)
- `getProtocolToken(chainId)` returns the correct native token for any chain
- Falls back to searching the SDK network data, then mainnet ETH as last resort

### Wrapped Protocol Token
- Each chain maps to its wrapped native token (WETH, WMATIC, WXDAI, etc.)
- `getWrappedProtocolToken(chainId)` returns the wrapped version
- Native and wrapped are treated as equivalent in DCA pair matching

### Token Lists
- Multiple external token list sources (1inch, CowSwap, SushiSwap, etc.) organized by chain
- Custom parsers per list (some return arrays, some return records, some need filtering)
- `custom-tokens` list stores user-added tokens
- Token blacklist for low-liquidity or problematic tokens
- Tokens not found in lists are fetched on-chain via `getCustomToken()` using multicall (balanceOf, decimals, name, symbol)

### Token Factory (`toToken`)
```typescript
export const toToken = ({ address, decimals, chainId, ... }) => ({
  decimals: decimals || 18,
  chainId: chainId || 1,
  address: (address?.toLowerCase() || '0x') as Address,  // Always lowercase
  name: name || '',
  symbol: symbol || '',
  type: type || TokenType.BASE,
  underlyingTokens: underlyingTokens || [],
  chainAddresses: chainAddresses || [],
});
```

### Token Equivalence
- `isSameToken(a, b)` - Same address AND chainId
- `getIsSameOrTokenEquivalent(a, b)` - Same token or cross-chain equivalent via `chainAddresses`
- `chainAddresses` array maps a token across chains (e.g., USDC on Ethereum = USDC on Polygon)
- `findTokenAnyMatch()` searches by tokenId, address, or symbol

### Empty Token Factories
Helper functions for UI contexts where a full token isn't available:
- `emptyTokenWithAddress(address)` - Token stub with just an address
- `emptyTokenWithLogoURI(logoURI)` - Token stub with just a logo
- `emptyTokenWithSymbol(symbol)` - Token stub with just a symbol
- `emptyTokenWithDecimals(decimals)` - Token stub with custom decimals

### Key Files
- `/apps/root/src/common/mocks/tokens.ts` - Protocol/wrapped token definitions
- `/apps/root/src/common/utils/currency.ts` - Token factory, comparison utils
- `/apps/root/src/services/walletService.ts` - getCustomToken, getBalance

---

## 4. Amount / BigNumber Handling

### Migration: ethers BigNumber -> viem bigint
The codebase migrated from ethers.js `BigNumber` to native `bigint` (viem). This is visible in the PR history where `BigNumber.from()` was replaced with `BigInt()` / `0n` literals.

### Core Patterns

**Parsing user input to on-chain amounts:**
```typescript
import { parseUnits } from 'viem';
const amount = parseUnits(userInput, token.decimals);  // string -> bigint
```

**Formatting on-chain amounts for display:**
```typescript
import { formatUnits } from 'viem';
const display = formatUnits(amount, token.decimals);  // bigint -> string
```

**Significant digits display:**
```typescript
// Uses decimal.js-light + JSBI for precision
formatCurrencyAmount({ amount, token, sigFigs: 4, maxDecimals: 3, intl })
// Returns '<0.001' when amount is below threshold
// Returns '0' for zero
// Supports locale-aware formatting with intl
```

**USD price calculation:**
```typescript
export const parseUsdPrice = (from, amount, usdPrice) => {
  const multiplied = amount * usdPrice;  // bigint * bigint
  return parseFloat(Number(formatUnits(multiplied, from.decimals + 18)).toFixed(2));
  // usdPrice is stored with 18 decimals precision
};
```

**USD price stored as bigint with 18 decimals:**
```typescript
parseNumberUsdPriceToBigInt(usdPrice: number) => parseUnits(usdPrice.toFixed(18), 18)
parseBaseUsdPriceToNumber(usdPrice: bigint) => parseFloat(formatUnits(usdPrice, 18))
```

### Input Validation
```typescript
export const amountValidator = ({ nextValue, decimals, onChange }) => {
  const newNextValue = nextValue.replace(/,/g, '.');  // Normalize comma to dot
  const inputRegex = RegExp(`^\\d*(?:\\\\[.])?\\d{0,${decimals}}$`);
  if (inputRegex.test(newNextValue.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'))) {
    onChange(newNextValue.startsWith('.') ? `0${newNextValue}` : newNextValue || '');
  }
};
```

### Max/Unlimited Values
- `maxUint256` from viem represents unlimited approval
- `totalSupplyThreshold(decimals)` = `(maxUint256 - 1n) / 10n ** BigInt(decimals)` - Used to detect if an approval was "unlimited"
- Protocol tokens always return `maxUint256` allowance (native tokens don't need approval)

---

## 5. Address Handling

### Address Display
```typescript
// Truncation:
export const trimAddress = (address: string, trimSize?: number) =>
  `${address.slice(0, trimSize || 6)}...${address.slice(-(trimSize || 6))}`;
// Default shows 6 chars on each side: "0x1234...abcdef"
```

### ENS Resolution
- `LabelService.fetchEns(address)` resolves ENS names asynchronously
- Results cached in `storedEnsNames` state (keyed by lowercase address)
- ENS names are resolved lazily: only when an address has no stored label and no cached ENS result
- Display priority: stored label > ENS name > trimmed address

### Address Component
The `<Address>` component (`/apps/root/src/common/components/address/index.tsx`):
- Props: `address`, `trimAddress`, `trimSize`, `editable`, `showDetailsOnHover`
- Resolves ENS on mount if no label exists
- `showDetailsOnHover`: shows trimmed address + copy icon on hover
- `editable`: renders an inline edit input for custom labels
- Copy to clipboard with snackbar confirmation

### Address Normalization
- All addresses are **lowercased** throughout the codebase: `address.toLowerCase() as Address`
- `toToken()` lowercases the address
- Wallet comparisons always use `.toLowerCase()`
- Contract addresses from constants are lowercased at access time

### Contacts System
```typescript
export const getDisplayContact = (contact: Contact) => {
  return contact.label?.label || trimAddress(contact.address);
};
```

### ENS for Transfers
- Transfer-to fields accept ENS names (PR #1360)
- ENS resolution happens before transaction submission
- Validation prevents transferring to own address

---

## 6. Transaction Lifecycle

### Transaction Building
1. Service method (e.g., `positionService.deposit()`) builds transaction data
2. `encodeFunctionData()` from viem encodes the contract call
3. `providerService.sendTransaction()` or `sendTransactionWithGasLimit()` sends it
4. Returns `SubmittedTransaction`: `{ hash, from, chainId }`

### Transaction Tracking (Redux)
```typescript
interface TransactionDetails {
  hash: string;
  from: Address;
  chainId: number;
  type: TransactionTypes;
  typeData: TransactionTypeDataOptions;
  addedTime: number;
  receipt?: TransactionReceipt;
  retries: number;
  checking: boolean;
  isCleared: boolean;
}
```

### Transaction Updater (`transactionUpdater.tsx`)
Polling-based system that checks pending transactions:
1. `usePendingTransactions()` returns all non-finalized transactions
2. For each pending tx, calls `getTransactionReceipt(hash, chainId)`
3. On receipt found:
   - If `receipt.status !== 0` (success): dispatch `finalizeTransaction`, show success snackbar, update positions/balances
   - If `receipt.status === 0` (reverted): dispatch `removeTransaction`, show error snackbar, call `handleTransactionRejection`
4. If no receipt:
   - Call `getTransaction(hash)` to check if tx exists on-chain at all
   - If not found: increment retry counter
   - If retries exceed `maxRetries` (chain-specific): treat as failed, remove transaction
5. For Safe transactions: resolve safe tx hash to actual on-chain hash first

### Multi-Wallet Transaction Filtering
Transactions are filtered to show all wallets belonging to the user (not just the active wallet):
```typescript
const wallets = useWallets();
const mappedWallets = getWalletsAddresses(wallets);
pickBy(state[chainId], (tx) => mappedWallets.includes(tx.from.toLowerCase()));
```

### Transaction Types
DCA, aggregator swaps, earn deposits/withdraws, token approvals, NFT transfers, position modifications, etc. Each type has associated `typeData` for post-transaction state updates.

### Key Files
- `/apps/root/src/state/transactions/transactionUpdater.tsx` - Polling and confirmation
- `/apps/root/src/state/transactions/hooks.ts` - Transaction selectors
- `/apps/root/src/state/transactions/reducer.ts` - Transaction state management

---

## 7. Gas Estimation

### SDK-Based Gas Estimation
```typescript
// ProviderService.estimateGas()
async estimateGas(tx: TransactionRequestWithChain): Promise<bigint> {
  return this.sdkService.sdk.gasService.estimateGas({
    chainId: tx.chainId,
    tx: { ...tx, to: tx.to || undefined, type: viemTransactionTypeToSdkStype(tx) },
  });
}
```

### Gas Cost Calculation
```typescript
// Full gas cost (estimation + price):
async getGasCost(tx: TransactionRequestWithChain) {
  const gasEstimation = await sdkService.sdk.gasService.estimateGas({ ... });
  return sdkService.sdk.gasService.calculateGasCost({ chainId, gasEstimation, tx });
}
```

### Gas Buffer
When sending with gas limit, a 30% buffer is added:
```typescript
gasLimit: (gasUsed * 130n) / 100n  // 30% more than estimated
```

### Gas Price
```typescript
async getGasPrice(chainId: number) {
  const provider = this.getProvider(chainId);
  return provider.getGasPrice();
}
```

### Gas Display in Transaction Confirmation
- Gas cost shown in native token with USD equivalent
- Protocol token price fetched from `useRawUsdPrice(protocolToken)`
- Gas USD is calculated from receipt's actual gas used (not estimated)

### Historical Gas API Migration
Early versions used GasNow API, then migrated to txprice.com API, and finally delegated to the SDK's gas service.

---

## 8. Approval Patterns

### Standard ERC20 Approve
```typescript
// WalletService.approveSpecificToken()
async approveSpecificToken(token, addressToApprove, ownerAddress, amount?) {
  const data = encodeFunctionData({
    ...erc20,
    functionName: 'approve',
    args: [addressToApprove, amount || maxUint256],  // Default: unlimited
  });
  return providerService.sendTransaction({ to: erc20.address, data, chainId, from });
}
```

### Permit2 (Signature-Based Approval)
The codebase supports Permit2 as an alternative to traditional approvals:

```typescript
// Permit2Service.getPermit2SignedData()
// 1. Prepare permit data via SDK
const preparedSignature = await sdk.permit2Service.arbitrary.preparePermitData({
  appId: PERMIT_2_WORDS[wordIndex],
  chainId, signerAddress, token, amount,
  signatureValidFor: '1d',
});

// 2. Sign typed data
const rawSignature = await signer.signTypedData({
  domain: preparedSignature.dataToSign.domain,
  types: preparedSignature.dataToSign.types,
  message: preparedSignature.dataToSign.message,
  primaryType: 'PermitTransferFrom',
});

// 3. Parse and fix signature values
const fixedSignature = parseSignatureValues(rawSignature);
```

### Permit2 Variants
- `getPermit2SignedData()` - For arbitrary swap transactions
- `getPermit2DcaSignedData()` - For DCA position creation (uses `dcaService.preparePermitData`)
- `getPermit2EarnSignedData()` - For Earn deposits (uses `earnService.preparePermitData`)
- Each variant uses the same sign flow but different SDK service methods

### Unlimited vs Exact Approval
- User setting `useUnlimitedApproval` controls default behavior
- Some contexts force unlimited approval when `!allowsExactApproval`
- Unlimited detection: `amount >= totalSupplyThreshold(decimals)` where threshold = `(maxUint256 - 1n) / 10n ** BigInt(decimals)`

### Allowance Checking
```typescript
// WalletService.getSpecificAllowance()
// Protocol tokens always return maxUint256 (no approval needed)
// NULL_ADDRESS target also returns maxUint256
// Otherwise reads erc20.allowance(owner, spender)
```

### Allowance Targets
- DCA: HUB or HUB Companion address (depends on yield/protocol token usage)
- Aggregator: Source-specific (varies by DEX aggregator)
- Earn: Earn Companion address
- Permit2: Universal Permit2 contract address, then MeanPermit2 for the actual operation

### Key Files
- `/apps/root/src/services/walletService.ts` - Standard approve/allowance
- `/apps/root/src/services/permit2Service.ts` - Permit2 signatures
- `/apps/root/src/common/components/transaction-steps/index.tsx` - Approval step UI

---

## 9. Contract Interaction

### Library: Viem
All contract interactions use **viem** (not ethers). The migration from ethers to viem is complete.

### Contract Instance Pattern
```typescript
// ContractService manages contract instances
// Read-only vs writable:
type ContractInstanceParams<ReadOnly extends boolean> = {
  chainId: number;
  readOnly: ReadOnly;
} & (ReadOnly extends false ? { wallet: NonNullable<Address> } : {});

// Getting an ERC20 instance:
const erc20 = await contractService.getERC20TokenInstance({
  tokenAddress: address,
  readOnly: true,
  chainId: network.chainId,
});
```

### PublicClient vs WalletClient
- `ProviderService.getProvider(chainId)` returns a `PublicClient` (read-only, from SDK)
- `ProviderService.getSigner(address, chainId)` returns a `WalletClient` (for signing)
- If signer is on wrong chain, automatic chain switch is attempted before returning

### Multicall
Used for batching read calls:
```typescript
const [balanceResult, decimalResult, nameResult, symbolResult] = await provider.multicall({
  contracts: [
    { ...erc20, functionName: 'balanceOf', args: [ownerAddress] },
    { ...erc20, functionName: 'decimals' },
    { ...erc20, functionName: 'name' },
    { ...erc20, functionName: 'symbol' },
  ],
});
```

### SDK Service
Most DeFi operations go through `@balmy/sdk`:
- `sdk.dcaService` - DCA position management
- `sdk.earnService` - Earn vault operations
- `sdk.permit2Service` - Permit2 operations
- `sdk.gasService` - Gas estimation/calculation
- `sdk.priceService` - Token prices
- `sdk.providerService` - Public clients

### Event Log Parsing
```typescript
// TransactionService.parseLog()
// Decodes logs against HUB, HUB Companion, and Earn Vault ABIs
const parsedLog = decodeEventLog({ ...contractInstance, ...log });
```

### Contract Address Management
All contract addresses are stored in `@constants/addresses.ts` as records indexed by chainId and version:
```typescript
HUB_ADDRESS[version][chainId]
COMPANION_ADDRESS[version][chainId]
EARN_VAULT_ADDRESS[chainId]
EARN_COMPANION_ADDRESS[chainId]
PERMIT_2_ADDRESS[chainId]
MEAN_PERMIT_2_ADDRESS[chainId]
```

### Key Files
- `/apps/root/src/services/contractService.ts` - Contract instances, address lookups
- `/apps/root/src/services/sdkService.ts` - SDK wrapper
- `/apps/root/src/services/providerService.ts` - PublicClient/WalletClient management

---

## 10. Price Feeds

### SDK Price Service
```typescript
// PriceService.getUsdCurrentPrices()
async getUsdCurrentPrices(tokens: Token[]) {
  return sdkService.sdk.priceService.getCurrentPrices({
    tokens: tokens.map(token => ({ chainId: token.chainId, token: token.address })),
  });
}
```

### Price Storage
- Prices stored as `bigint` with 18 decimals precision in Redux state
- `parseNumberUsdPriceToBigInt(number)` and `parseBaseUsdPriceToNumber(bigint)` for conversion
- Stored per-chain in `balances` state: `state.balances[chainId].balancesAndPrices[tokenAddress].price`

### Price Hooks
- `useRawUsdPrice(token)` - Returns current USD price for a token
- `usePriceService()` - Access to price service for custom queries

### Historical Prices
```typescript
getUsdHistoricPrice(tokens, date?, chainId?)
// Uses sdk.priceService.getHistoricalPricesInChain()
// Supports multiple time periods: dayAgo, weekAgo, monthAgo, yearAgo
```

### Price for Protocol Tokens
Protocol tokens (native) use a special address mapping for price fetching:
```typescript
getTokenAddressForPriceFetching(chainId, address)
// Maps PROTOCOL_TOKEN_ADDRESS to the chain's wrapped token for price lookups
```

### USD Amount Formatting
```typescript
formatUsdAmount({ amount, intl })
// Converts to 2-decimal representation, uses formatCurrencyAmount internally
```

### Net Worth Calculation
Aggregates across wallet balances, DCA positions, and Earn positions:
```typescript
const totalAssetValue = wallet + dca + earn;
// Each computed separately with chain/wallet/token filtering
// Handles NaN with: Number(value || 0)
```

---

## 11. Chain Data

### Block Watching
```typescript
// ProviderService.onBlock()
onBlock(chainId: number, listener: (block: bigint) => void) {
  const provider = this.getProvider(chainId);
  return provider.watchBlocks({
    onBlock: (block) => listener(block.number),
  });
}

// Safe app fallback: polling every 10 seconds
if (loadedAsSafeApp) {
  return window.setInterval(() => callback(this.getBlockNumber(chainId)), 10000);
}
```

### Block Number Tracking
- Block numbers tracked per chain in transaction state
- Used to determine if DCA indexer has caught up to a transaction's block
- `getBlockTimestamp()` for time-based calculations

### RPC Management
- SDK manages RPC connections with fallback strategy
- Rootstock has dedicated RPC endpoints with API keys
- Public RPCs used as fallback with load-balancing (maxAttempts: 2, maxConcurrent: 2)

### Caching Strategy
```typescript
// Axios cache excludes dynamic endpoints:
exclude: {
  paths: [/.*accounts$/, /.*balances.*/, /.*history.*/],
}
```

### Indexer Tracking
- DCA and Earn indexing blocks tracked via `TransactionService.fetchIndexingBlocks()`
- Used to determine when transaction effects are reflected in indexed data
- Transaction confirmation waits for indexer to process the block before updating DCA positions

---

## 12. Error Handling

### Error Classification
```typescript
// errors.tsx
export const TRANSACTION_ERRORS = {
  4001: 'You rejected the transaction',        // User rejected (MetaMask)
  4100: 'You are not authorized...',            // Unauthorized
  4200: 'Unsupported method',
  4900: 'You are disconnected from the net',    // All chains
  4901: 'You are disconnected from the net',    // Specific chain
  ACTION_REJECTED: 'You rejected the transaction', // ethers-style
  UNPREDICTABLE_GAS_LIMIT: 'Error... report on Discord',
};
```

### Error Tracking Filter
```typescript
// shouldTrackError() - Determines if error should be sent to error tracking
const EXCLUDED_ERROR_CODES = [4001, 'ACTION_REJECTED'];
const EXCLUDED_ERROR_MESSAGES = [
  'User canceled',
  'Failed or Rejected Request',
  'user rejected transaction',
  'user rejected signing',
];
// User-initiated rejections are NOT tracked as errors
```

### Viem Error Handling
```typescript
// getTransactionErrorCode() maps viem errors to error codes
switch (error.name) {
  case 'TransactionExecutionError':
    // Unwraps cause to find UserRejectedRequestError
    return (error.cause as UserRejectedRequestErrorType).code;
  default:
    return UNKNOWN_ERROR_CODE;
}
```

### Error Serialization
```typescript
// deserializeError() - Converts Error objects to plain objects for Redux/logging
export const deserializeError = (err) => {
  const plainObject = {};
  Object.getOwnPropertyNames(err).forEach(key => {
    plainObject[key] = err[key];
  });
  return plainObject;
};
```

### Transaction Failure Patterns
1. **Receipt with status 0**: Transaction reverted on-chain
   - Remove from pending, show error snackbar
   - Call `handleTransactionRejection()` to revert optimistic state updates
2. **Transaction not found after max retries**: Transaction likely dropped from mempool
   - Chain-specific retry limits (Rootstock: 90, default: 10)
   - After exceeding retries: treat as failed
3. **User rejection**: Caught in catch blocks, filtered by `shouldTrackError()`

### Error Display
- `TransactionModal` shows error messages from `TRANSACTION_ERRORS` lookup
- Unknown errors show a generic "report on Discord" message
- Error config tracks whether to show a "report" button (only for non-user-rejected errors)

### Standard Error Handling Pattern (used across all transaction flows)
```typescript
try {
  // Execute transaction
} catch (e) {
  if (shouldTrackError(e as Error)) {
    trackEvent('Operation - Error', { error: deserializeError(e) });
    // Show error modal
  }
}
```

### Key Files
- `/apps/root/src/common/utils/errors.tsx` - Error classification and tracking
- `/apps/root/src/common/components/transaction-modal/index.tsx` - Error display UI
- `/apps/root/src/state/transactions/transactionUpdater.tsx` - Transaction failure handling
