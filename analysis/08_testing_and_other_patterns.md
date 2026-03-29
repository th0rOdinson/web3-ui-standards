# Testing, Notifications, and Other Patterns Analysis

Analysis of testing, notifications, uncategorized PRs, and project evolution patterns from the Web3 Frontend (1,385 PRs, May 2021 - July 2025).

---

## 1. Testing Patterns

### Framework and Tooling

- **Test runner**: Jest (integrated via Turborepo with `yarn test` / `turbo run test`)
- **Test file convention**: `*.spec.ts` files co-located with the service files they test (e.g., `src/services/errorService.spec.ts`, `src/services/safeService.spec.ts`)
- **CI integration**: Tests added to the GitHub Actions `check.yml` workflow in PR #482 (Nov 2023), relatively late in the project's life

### Test Structure

Tests follow a consistent pattern across all service test files:

```typescript
// 1. Import createMockInstance utility
import { createMockInstance } from '@common/utils/tests';

// 2. jest.mock() the dependencies
jest.mock('./meanApiService');

// 3. Create mocked constructor references
const MockedService = jest.mocked(ServiceClass, { shallow: true });

// 4. describe/beforeEach/afterEach/test blocks
describe('Service Name', () => {
  let service: ServiceUnderTest;
  let dependency: jest.MockedObject<DependencyClass>;

  beforeEach(() => {
    dependency = createMockInstance(MockedConstructor);
    service = new ServiceUnderTest(dependency as unknown as DependencyClass);
  });

  afterEach(() => {
    jest.resetAllMocks();
    jest.restoreAllMocks();
  });
});
```

### Mocking Patterns

- **`createMockInstance`**: Custom utility at `@common/utils/tests` that creates mock instances from jest-mocked constructors. This is the primary mocking pattern.
- **`jest.mocked(Class, { shallow: true })`**: Used to get typed mock references from `jest.mock()`'d modules
- **`jest.useFakeTimers()`**: Used for time-dependent tests (e.g., Safe SDK timeout detection)
- **`jest.spyOn`**: Used for specific method spying (e.g., `jest.spyOn(uuid, 'v4')`, `jest.spyOn(window, 'parent', 'get')`)
- **Type casting**: `dependency as unknown as DependencyClass` pattern for passing mocked objects to constructors

### What Gets Tested

Testing is focused almost exclusively on **service-layer unit tests**:

- `errorService.spec.ts` - API error logging
- `eventService.spec.ts` - Analytics event tracking, identifier generation
- `safeService.spec.ts` - Gnosis Safe SDK integration (tx hash resolution, iframe detection, multi-tx submission)
- `simulationService.spec.ts` (PR #402)
- `yieldService.spec.ts` (PR #406)
- `labelService.spec.ts` - Contact label CRUD
- `accountService.spec.ts` - Wallet authentication, signature messages
- `priceService.spec.ts` (PR #454) - USD price fetching, historic prices

No evidence of:
- Component-level testing (no React Testing Library or Enzyme)
- E2E testing (no Cypress or Playwright test files)
- Snapshot testing
- Integration testing

### Testing Volume

Only **27 PRs** out of 1,385 are categorized as testing-related. Most of the "testing" PRs are actually about:
- Removing test networks from supported chains (PRs #141, #319)
- Removing mock data (PR #1067)
- Updating mock tokens (PR #721)
- Adding CI test steps (PR #482)

Actual test file creation is concentrated in a burst around May-June 2023 (PRs #402, #406, #408, #409, #410) suggesting a deliberate testing initiative that didn't sustain momentum.

---

## 2. Notification Patterns

Only **5 PRs** are categorized under notifications, making this one of the smallest categories.

### Snackbar/Toast System

- Uses **notistack** library (`snackbar.enqueueSnackbar()`)
- Snackbar variants: `'info'`, `'success'`, `'error'`, `'warning'`
- Messages are internationalized via `react-intl` (`intl.formatMessage`)

Example pattern from PR #1005:
```typescript
snackbar.enqueueSnackbar({
  variant: 'info',
  message: intl.formatMessage(
    defineMessage({ ... }),
    { remainingClicks: menuClicksDiff }
  ),
});
```

### Transaction Notification System

The project has a multi-layered notification system for blockchain transactions:

1. **Transaction Modal** (`useTransactionModal`): Three-state modal system:
   - `setModalLoading` - Shows pending state with descriptive message
   - `setModalSuccess` - Shows tx hash and success message
   - `setModalError` - Shows error code and message

2. **Transaction Message Builders** (hooks):
   - `useBuildTransactionMessage` - Post-confirmation messages per transaction type
   - `useBuildTransactionDetail` - Detailed transaction descriptions
   - `useBuildRejectedTransactionMessage` - Rejection/failure messages
   - Each uses a large `switch` statement over `TransactionTypes` enum

3. **Quote Status Notification** (`QuoteStatusNotification` component):
   - Tracks swap quote status with `QuoteStatus` enum (`None`, loading, success, etc.)
   - Shows "better quote available" notifications during swap flow
   - Reset on form clear (PR #1325)

4. **Transaction Receipt** system:
   - `useTransactionReceipt` hook for checking on-chain results
   - Differentiates between transaction event types (ERC20_APPROVAL, EARN_WITHDRAW, etc.)
   - Special handling for unlimited approvals vs specific amounts (PR #898)
   - Delayed vs immediate withdrawal distinction (PR #1168)

### Companion Approval Alert

PR #39 introduced a persistent alert pattern for position permissions:
- Uses MUI `Alert` component with `severity="warning"` and `variant="outlined"`
- Shows when the Hub Companion contract lacks approval to modify a position
- Includes external link to the contract on block explorer

---

## 3. Common Patterns from "Other" PRs

The "other" category contains **400 PRs** (29% of total), the largest single category.

### Sub-patterns within "Other"

#### Deployment and Infrastructure
- **AWS S3 + CloudFront**: Three deployment targets (staging, playground, production)
- **Deployment workflow fix** (PR #463): Changed from selective file sync (only `.js`, `.js.map`, `index.html`) to full sync with `--include "*"` and full CloudFront invalidation with `"/*"`
- Three separate GitHub Actions workflows: `deployToStaging.yml`, `deployToPlayground.yml`, `deployMainToProduction.yml`

#### Token/Protocol Blacklisting
- `TOKEN_BLACKLIST` array in `config/constants/common.ts` with inline comments explaining why each token is blacklisted
- Separate blacklists for DCA and Aggregator (PR #399)
- Tokens blacklisted for reasons like: liquidity moved, protocol no longer supported, testing not complete (Beefy wrappers in PR #273)
- Yield options commented out with `// TODO: Remove this once we check beefy works correctly`

#### News Banner System
- Reusable `NewsBanner` component in `src/common/components/news-banner/`
- Uses `StyledBannerContainer` with `ContainerBox` for layout
- Frequently updated to promote campaigns (Galxe quests, LM programs) - PRs #1363, #1364, #1374
- Short-lived, often added and removed in quick succession

#### SDK Updates
- Frequent `@balmy/sdk` version bumps (previously `@mean-finance/sdk`)
- Version jumps tracked in `package.json` at root level (monorepo dependency)
- SDK updates often paired with new chain support or aggregator source changes
- Branch naming for SDK updates: `update-sdk-*` or `magpie-rebrand`

#### Earn-Related Fixes
- APY calculation fixes including rewards APY (PR #1296)
- Strategy filtering by tier level (PR #1296)
- Graph formatting improvements (compact number notation)
- Protocol token address resolution for wrapped tokens during withdrawals

---

## 4. PR Description Conventions

### Early PRs (2021-2022)
- Descriptions are typically `null` or empty
- Titles carry all the information
- No structured format

### Mid-period PRs (2023)
- Descriptions become more common but remain free-form
- Technical context appears: "We are now...", "With this change..."
- Some include screenshot embeds via GitHub's image upload

### Late-period PRs (2024-2025)
- More structured descriptions emerge, especially from `Reviewer B`:
  - `## Context` section explaining the problem
  - `## Demo` section with screenshots or video recordings
  - Inline image embeds for UI changes
- `Reviewer A` PRs remain terse, often just screenshots
- `Reviewer E`/`Reviewer C` PRs explain the "why" well: "We are now updating the SDK to version X, which has Y"

### PR Description Patterns by Author

- **Reviewer A**: Minimal or no description, relies on PR title. Occasional screenshot.
- **Reviewer B**: Most detailed descriptions. Uses Context/Demo sections, includes before/after screenshots, explains the user flow affected.
- **Reviewer E**: Explains what changed and why in plain language. Often numbered lists.
- **Reviewer C**: Short but contextual ("Tested on my local env, and working sweet")
- **dependabot[bot]**: Auto-generated with full changelog, release notes, and compatibility scores
- **Reviewer D**: Good descriptions with screenshots for UI changes, technical details for logic changes

### Title Conventions

Format: `type(scope): description`

- `feat(earn): ...` - New feature in earn area
- `fix(swap): ...` - Bug fix in swap
- `chore(deps): ...` - Dependency updates
- `refactor(aggregator): ...` - Code restructuring
- `test(safeservice): ...` - Test additions
- Some PRs use Jira-style ticket IDs as branch names: `BLY-3847`, `MF-779`, `MEA-85`

Scope is typically a component/service name in lowercase: `earn`, `swap`, `dca`, `transfer`, `aggregator`, `ui-library`, `nav`, etc.

---

## 5. Feature Flag Patterns

### Token/Chain Enable/Disable

The primary "feature flag" mechanism is configuration-based, not a runtime flag system:

- **Chain support**: Arrays like `SUPPORTED_NETWORKS`, `AGGREGATOR_SUPPORTED_CHAINS`, `SUPPORTED_NETWORKS_DCA` in constants files
- Adding a chain = adding its chainId to the array + setting up PERMIT_2 and MEAN_PERMIT_2 addresses
- Removing a chain = removing from arrays (PR #1369: removed Fantom and Polygon zkEVM)

- **Aggregator source toggle**: `DISABLED_DEXES` array (e.g., PR #1341 disabled Magpie, PR #1361 re-enabled it)
- **Token blacklists**: `TOKEN_BLACKLIST` array with per-token disable

### Tier-Based Access Control

For the Earn feature, a tier system gates strategy visibility:
- Strategies have a `needsTier` property (values 0-3)
- User tier is determined by `tierService`
- Filtering logic: `(strategy.needsTier ?? 0) >= tierToUseToFilter`
- Override: tiers 2 and 3 always shown regardless of user tier (PR #1296)

### Early Access / Invite Codes

- `earlyAccessEnabled` flag on `AccountService`
- Invite code claiming via `meanApiService.claimEarnInviteCode()`
- Used for gating Earn features before public launch

### Secret Menu

- Click counter pattern (`secretMenuClicks`) requiring `SECRET_MENU_CLICKS` clicks to activate
- Threshold notification at `menuClicksDiff < 3` (PR #1005)
- No runtime feature flag service (no LaunchDarkly, no Unleash, no Firebase Remote Config)

### Downtime Modal

- Quick deploy/revert pattern for maintenance windows (PR #1337 added, PR #1338 reverted)
- Direct code change, not a flag

---

## 6. Release Patterns

### Release Process

Releases follow a `staging -> main` merge pattern:

1. Feature branches target `staging` as base
2. When ready to release, a "Release" PR merges `staging` into `main`
3. Main branch triggers production deployment via `deployMainToProduction.yml`

### Release PR Characteristics

- Title: `"Release"`, `"Release!"`, or `"Release: <milestone>"` (e.g., "Release: Earn Public Launch")
- Body: Always `null` - no release notes in the PR
- Author: Either `Reviewer A` or `Reviewer B`
- These PRs often contain multiple accumulated features

### Release Frequency

Looking at release PRs merging `staging -> main`:
- **2023**: Irregular, weeks between releases
- **2024 Q3-Q4**: More frequent, roughly weekly during earn feature development
- **2025**: Regular cadence of 1-3 weeks between releases
- Final PRs (July 2025): "Sunset balmy" - project shutdown

### Base Branch Evolution

- **2021**: `master`
- **2021-2022**: `v2-stable` briefly, then back to `master`
- **2022-2023**: Transition to `main`
- **2023-2025**: `staging` as development branch, `main` as production
- Some PRs target intermediate branches (e.g., `scope-steps-components`, `scope-aggregator-components`) for large feature branches

---

## 7. Dependency Update Patterns

### Dependabot

- Active for security patches on yarn.lock dependencies
- Auto-generated PRs with standard format: changelog, release notes, compatibility scores
- Examples: `json5` 1.0.1 -> 1.0.2 (prototype pollution fix), `decode-uri-component` 0.2.0 -> 0.2.2, `cookiejar` 2.1.3 -> 2.1.4
- These are typically merged in batches (PRs #294, #295, #296 all merged same day)
- No evidence of Renovate or other automated dependency tools

### SDK Updates

The most frequent manual dependency update is the the project SDK (`@balmy/sdk`, formerly `@mean-finance/sdk`):

- Updated via PRs from `Reviewer E`, `Reviewer C`, or `Reviewer A`
- Often paired with feature changes that require the new SDK version
- Naming: `chore(*): update sdk`, `chore: update sdk to 0.0.36`, `feat(*): upgrade sdk`
- SDK is declared at root `package.json` in the monorepo, not in individual packages
- Version progression visible: 0.0.36 -> 0.0.151 -> 0.3.x -> 0.4.x

### UI Library

The project has an internal `ui-library` package within the monorepo (`packages/ui-library/`), which wraps Material UI components with custom theming.

---

## 8. Security Patterns

### Token Approval Safety

- **Specific vs unlimited approval**: Split button pattern allowing users to choose between approving exact amount or max (PR #355)
- Default changed to **specific approval** for aggregator (more conservative)
- Unlimited approval detection: `receipt.data.amount.amount >= totalSupplyThreshold(token.decimals)` (PR #898)
- `totalSupplyThreshold` utility in `@common/utils/parsing`

### Wallet Authentication

- Sign-in message includes Terms of Use and Privacy Policy links (PR #1094)
- Signature version tracking: `LATEST_SIGNATURE_VERSION` constant, bumped on message changes
- Local storage keys for signature persistence: `WALLET_SIGNATURE_KEY`, `LATEST_SIGNATURE_VERSION_KEY`
- Auth header format: `WALLET signature="${signature.message}", signer="${signature.signer}"`

### Input Validation

- Token address validation when importing custom tokens (PR #451): shows "invalid address" or "no token found" errors
- Case-insensitive address comparison throughout: `.toLowerCase()` before comparing addresses
- `isSameAddress()` utility for address comparison

### Safe Wallet Handling

- Safe App SDK integration with timeout protection (3-second timeout for `isSafeApp()` check)
- Value sanitization for Safe transactions: handles string, BigNumber, and number types, defaults to "0"

### Token Blacklisting

- Address-based blacklisting to prevent users from interacting with deprecated or dangerous tokens
- Inline comments documenting the reason for each blacklisted address
- Separate lists for DCA and aggregator contexts

### Rate Limiting Protection

- 1-minute cache for price fetches to avoid Coingecko rate limiting (PR #393)
- Cache holds up to 20 token prices

### Permit2 Addresses

- Per-chain `PERMIT_2_ADDRESS` and `MEAN_PERMIT_2_ADDRESS` mappings
- All use the canonical Uniswap Permit2 deployment: `0x000000000022D473030F116dDEE9F6B43aC78BA3`

---

## 9. Project Temporal Evolution

### Phase 1: Foundation (May 2021 - Dec 2021) - PRs #1-30

- Basic DCA functionality on Optimism and Polygon
- Material UI for components
- Simple structure, single `src/` directory
- Branch from `master`
- Authors: `Reviewer G`, `Reviewer F`
- Minimal PR descriptions, no screenshots

### Phase 2: Feature Expansion (2022) - PRs #30-170

- Yield while DCA feature
- Multi-chain support expansion
- Position details, transfer, terminate modals
- New authors join: `Reviewer A`, `Reviewer D`, `Reviewer E`
- Still monolithic structure

### Phase 3: Aggregator Era (Early-Mid 2023) - PRs #170-500

- Swap aggregator becomes a major product area
- Monorepo restructuring with Turborepo
- `ui-library` package extracted
- Testing initiative (May 2023, 6 test PRs in one month)
- `Reviewer B` joins as primary frontend developer
- CI improvements (test step added)
- Transition from ethers.js BigNumber to native BigInt

### Phase 4: Design System Overhaul (Late 2023 - Mid 2024) - PRs #500-800

- Massive UI redesign: new home page, navigation, portfolio breakdown
- Component migration from MUI to custom `ui-library`
- Transfer feature built
- Token picker, token input redesigned
- Contact list, wallet menu improvements
- PR descriptions become more detailed with screenshots

### Phase 5: Earn Product (Mid 2024 - Early 2025) - PRs #800-1350

- Earn/yield vaults become the dominant feature area
- 200+ earn-related PRs
- Strategy wizard, vault management, delayed withdrawals
- Tier system, invite codes, referral links
- One-click migration between vaults
- `Reviewer B` becomes the primary contributor
- Most active development period

### Phase 6: Wind-Down (Mid 2025) - PRs #1350-1385

- SDK updates, minor fixes
- Chain additions/removals (Sonic added, Fantom/PolygonZkEVM removed, Rootstock disabled)
- Removal of Mixpanel analytics (PR #1381)
- "Sunset balmy" (PR #1384) - project shutdown
- Final PR: July 2, 2025

### Category Attention Over Time

| Category | Early (2021-22) | Mid (2023) | Late (2024-25) |
|----------|-----------------|------------|-----------------|
| DCA/Position | High | Medium | Low |
| Swap/Aggregator | None | High | Medium |
| Earn | None | None | Very High |
| UI Components | Medium | High | Medium |
| Chains | Medium | High | Medium |
| Testing | Low | Medium | Low |
| Tokens | Medium | Medium | Medium |
| Wallet | Low | Medium | Low |
| Transfer | None | Low | Low |
| Migration | None | None | Medium |
| Analytics | None | Medium | Medium |

The project shows a clear product evolution: DCA-only -> DCA + Aggregator -> DCA + Aggregator + Earn, with the Earn product consuming the majority of development effort in its final year.

### Author Contribution Shift

- **2021-2022**: `Reviewer G` (co-founder), `Reviewer F`, `Reviewer E`
- **2023**: `Reviewer A` (most prolific), `Reviewer B` (joins mid-year), `Reviewer E`, `Reviewer C`
- **2024-2025**: `Reviewer B` and `Reviewer A` share most work, with `Reviewer B` taking the lead on earn features
- `Reviewer D` contributed during 2022-2023 period (wallet integration, UI improvements)
- `Reviewer I` made a single contribution (Connext services, PR #412)
