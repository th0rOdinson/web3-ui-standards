# Code Review Patterns - Web3 Frontend

Exhaustive analysis of every review pattern, convention, and preference extracted from PR review comments across the project's history. Each pattern includes the rule, example PRs, reviewer quotes, and frequency assessment.

---

## 1. Commented-Out Code Must Be Removed

**Rule:** Never leave commented-out code in PRs. Either remove it or keep it active.

**Frequency:** Very high - flagged in 10+ PRs across all reviewers.

| PR | Reviewer | Quote |
|---|---|---|
| #116 | Reviewer D | "Why is it commented?" (on commented-out exports) |
| #131 | Reviewer D | "Comment seems unnecessary" (on `// backgroundColor: ...`) |
| #155 | Reviewer C | "Remove it you scawy anon. KILL IT WITH FIRE" |
| #172 | Reviewer D | "You can probably delete this comment" |
| #254 | Reviewer D | "Some unnecessary comments in this file" |
| #311 | Reviewer D | "No need for the comment" |
| #400 | Reviewer D | "Please remove commented lines" |
| #466 | Reviewer B | "Found you!" (console.log left in) |
| #478 | Reviewer B | "Debugging evidence" (commented console.log) |
| #513 | Reviewer B | "This line should be erased" / "In this file we have some pieces of code commented, I suggest erasing them before merge" |

---

## 2. No console.log in Production Code

**Rule:** Remove all console.log statements before merging.

**Frequency:** Moderate - flagged in 5+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #201 | Reviewer D | "a wild console.log appeared!" |
| #166 | Reviewer D | "typo (signer)" (also found console.log) |
| #466 | Reviewer B | "Found you!" (on `console.log(user, wallets)`) |
| #590 | Reviewer B | Flagged a console.log left in sdkService |

---

## 3. Typo and Spelling Corrections

**Rule:** Fix all typos in code, copy, and variable names before merge.

**Frequency:** Very high - flagged in 15+ PRs.

| PR | Reviewer | Quote/Detail |
|---|---|---|
| #3 | Reviewer F | `setCurentAction` -> `setCurrentAction` |
| #126 | Reviewer D | `"singer" !== "signer"` |
| #160 | Reviewer D | "Please correct to 'received'" (recieved -> received) |
| #166 | Reviewer D | "typo (signer)" |
| #259 | Reviewer E | "It's UNIDX actually" / "You have a typo here: beeon -> been" |
| #267 | Reviewer C | Accent corrections in Spanish translations |
| #348 | Reviewer D | "Please change to 'Unknown'. Typo is also present in other parts of the app." |
| #360 | Reviewer E | "Typo here" (EXECTLY -> EXACTLY) |
| #432 | Reviewer C | Multiple accent corrections in Spanish translations |

---

## 4. Use Optional Chaining and Nullish Coalescing

**Rule:** Prefer `?.` and `??` over manual null checks and `&&` chains.

**Frequency:** High - flagged in 8+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #43 | Reviewer F | Suggested `pair?.positions?.[0]?.createdAtTimestamp || 0` over manual `&&` chain |
| #178 | Reviewer F | Suggested `value.valuePerChain[selectedChain.toString()]?.balanceUSD ?? BigNumber.from('0')` |
| #193 | Reviewer E | Suggested `network?.chainId` over `(network && network.chainId)` |
| #345 | Reviewer D | "Please consider adding optional chaining: `{ name: chainId?.toLowerCase() }`" |

---

## 5. Simplify Boolean Returns

**Rule:** Return boolean expressions directly instead of `if (x) return true; return false;`.

**Frequency:** Moderate - flagged in 1 PR but with 3 instances.

| PR | Reviewer | Quote |
|---|---|---|
| #432 | Reviewer E | Three suggestions to simplify `if (x) { return true; } return false;` to `return x;` |

---

## 6. Use Enums Over Union Types

**Rule:** Prefer TypeScript enums over string union types for fixed sets of values.

**Frequency:** Moderate - flagged in 3+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #432 | Reviewer E | "May I suggest using ALL_CAPS for the enum entries? I think that's the convention" |
| #447 | Reviewer A | "Lets make this an enum, and return the enum on `categorizeError`" |
| #541 | Reviewer A | "Can we make this type an enum?" (for status type) |

---

## 7. Use `isSameAddress` from SDK for Address Comparisons

**Rule:** Never compare addresses with `===` directly. Use the SDK's `isSameAddress` utility that handles case normalization.

**Frequency:** Moderate - flagged in 2 PRs, with a Linear ticket created.

| PR | Reviewer | Quote |
|---|---|---|
| #432 | Reviewer C | "Some other day I'd try to abstract comparing addresses to a utils so you can do the lowercase thingy" |
| #432 | Reviewer E | "The SDK exports a function called `isSameAddress`" |
| #432 | Reviewer A | Created ticket BLY-1092 to change all address comparisons |

---

## 8. Use `FormattedMessage` for All User-Facing Text (i18n)

**Rule:** All user-facing strings must use `FormattedMessage` or `defineMessage`/`formatMessage` for internationalization. Never use raw template literals for UI text.

**Frequency:** High - flagged in 5+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #139 | Reviewer A | "Lets set a FormattedMessage for the title, so we can later translate it" |
| #139 | Reviewer A | "this description should be different since they are used as the uniq ids for the messages" |
| #510 | Reviewer A | "we probably want to use `defineMessage` before `formatMessage`" |
| #131 | Reviewer D | Various copy suggestions for FormattedMessage defaultMessage values |

---

## 9. Proper Sentence Punctuation in Copy

**Rule:** User-facing messages must end with a period and have correct grammar.

**Frequency:** Moderate - flagged in 5+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #84 | Reviewer C | "Sorry for that. Needs the final dot." |
| #131 | Reviewer D | "remove the coma" / "add a period in the end" |
| #131 | Reviewer D | "I think it should be 'We will send you..'" |
| #249 | Reviewer F | Grammar correction: "adding funds are disabled" -> "adding funds is disabled" |
| #310 | Reviewer D | Copy suggestion for FAQ text |

---

## 10. SDK Package Version Pinning

**Rule:** Pin SDK versions exactly (no `^` prefix) because `0.0.x` patch versions in standard versioning can mean breaking changes.

**Frequency:** Moderate - flagged in 3 PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #284 | Reviewer A | "I would like to keep this hardfixed since patch versions on `0.0.x` in standard versioning can mean breaking changes" |
| #434 | Reviewer E | "Let's use exact if possible" |

---

## 11. Abstract Repeated Logic into Utils/Functions

**Rule:** When the same logic appears multiple times, extract it into a utility function or shared component.

**Frequency:** High - flagged in 8+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #201 | Reviewer D | "I've seen this `priceImpact` calc a couple of times. Might be a candidate for a utils fn." |
| #211 | Reviewer D | "we can probably create an abstraction for this check, all the setRate have almost identical code" |
| #236 | Reviewer C | "I'd abstract the validation of 'was it a canceled transaction'" |
| #236 | Reviewer D | "I think it should be abstracted in a fn that receives `e` as argument" |
| #376 | Reviewer D | "What do you think about moving mui theme to it's own file?" |
| #489 | Reviewer A | "Maybe we should move all of this into it's own component?" |

---

## 12. Use `React.useMemo` and `React.useCallback` Appropriately

**Rule:** Memoize values and callbacks that are passed to child components to prevent unnecessary re-renders. But don't over-memoize simple computations.

**Frequency:** High - flagged in 6+ PRs, with disagreements on when it's needed.

| PR | Reviewer | Quote |
|---|---|---|
| #131 | Reviewer D | "I don't see the benefit in memoizing these functions" |
| #131 | Reviewer A | Explained why: "react checks by reference... each time the parent component re-rendered... we were making unnecessary re-renders" |
| #199 | Reviewer A | "We should probably React.useMemo this, so it does not recalculate it on every re-render" |
| #199 | Reviewer D | "the load of useMemo would be worse than the check itself" (disagreement) |
| #489 | Reviewer A | "We could generate this ones outside the component so they are not different on every re-render" |

---

## 13. Avoid Unnecessary useEffect Dependencies

**Rule:** Only include truly necessary dependencies in useEffect/useCallback/useMemo dependency arrays. Don't include stable references like `dispatch`, imported functions, or constants.

**Frequency:** High - flagged in 5+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #489 | Reviewer A | "no need for this since its an import" |
| #489 | Reviewer A | "no need to pass `currentChainId`, `selectedToken` should already have its `chainId` set" |
| #489 | Reviewer A | "no need to pass `validateAddress`" / "no need for this dependencies" |
| #489 | Reviewer A | "no need to add dispatch" |
| #499 | Reviewer A | "no need to set the boolean for typing, ts should pick it up as a boolean right away" |

---

## 14. Use Theme Spacing, Not Hardcoded Pixels

**Rule:** Always use `theme.spacing()` instead of hardcoded pixel values. Widths in px are "extremely discouraged."

**Frequency:** Very high - flagged in 10+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #550 | Reviewer A | "SPACIIIING" (on `max-width: '350px'`) |
| #577 | Reviewer B | "Let's apply theme units" (on `style={{ position: 'absolute', top: '24px', right: '32px' }}`) |
| #579 | Reviewer A | "is this necessary? Widths on px are extremely discouraged" |
| #579 | Reviewer A | "why the maxHeight set on px?" |

---

## 15. Use Typography Variants, Not Custom Font Styles

**Rule:** Use MUI Typography `variant` prop instead of manually setting font sizes, weights, and colors in styled components.

**Frequency:** High - flagged in 5+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #516 | Reviewer A | "we should be using the typography variant prop for this" |
| #525 | Reviewer A | "typography should have its variants defined" |
| #537 | Reviewer A | "into a styled component!" (for inline `style={}` usage) |
| #541 | Reviewer A | "I'm starting to think that we should set this directly on the typography since its being used quite a lot everywhere" |

---

## 16. No Inline `style={}` on MUI Components

**Rule:** MUI components accept `sx` prop. Never use `style={}`. For complex styling, use styled-components.

**Frequency:** Moderate - flagged in 3+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #516 | Reviewer A | "style should not be used on MUI components, they accept an `sx` property" |
| #537 | Reviewer A | "into a styled component!" (for `<div style={{...}}>`) |

---

## 17. No Direct MUI Imports in UI Library

**Rule:** The UI library package should not import directly from `@mui/material`. Use re-exported components.

**Frequency:** Low but strict - flagged in 1 PR emphatically.

| PR | Reviewer | Quote |
|---|---|---|
| #537 | Reviewer A | "no mui importssss" (with monkey emoji) |

---

## 18. Use `formatCurrencyAmount` for Token Amounts

**Rule:** Never use `.toFixed()` for displaying token amounts. Use the project's `formatCurrencyAmount` utility.

**Frequency:** Moderate - flagged in 2+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #508 | Reviewer A | "we should be using the formatCurrencyAmount and not toFixed" |
| #508 | Reviewer A | "lets use bignumber so we dont lose accuracy" |

---

## 19. Use BigNumber/bigint for Financial Calculations

**Rule:** Never use `parseFloat` or regular JavaScript numbers for token balances or financial calculations. Use BigNumber (ethers v5) or bigint (v6+).

**Frequency:** Moderate - flagged in 2+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #508 | Reviewer A | "lets use bignumber so we dont lose accuracy, you can use the `add` function from it" |
| #513 | Reviewer B | "We should check if balance is not undefined" (on bigint arithmetic) |

---

## 20. Secrets and API Keys in Environment Variables

**Rule:** API keys, tokens, and service URLs must go in environment variables/secrets, not hardcoded in source.

**Frequency:** Moderate - flagged in 2 PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #414 | Reviewer A | "This should go into an env file/secret" (on Mixpanel token) |
| #414 | Reviewer A | "This should also go into an env file/secret" (on API host URL) |
| #470 | Reviewer A | "Nope, not yet, lets use an env variable" (on API URL change) |

---

## 21. Prefer `lodash/compact` + `map` Over `reduce`

**Rule:** Instead of using `reduce` for filtering and transforming, use `map` returning null for invalid items, then `compact()` from lodash.

**Frequency:** Low - flagged in 2 PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #447 | Reviewer A | "instead of using a reduce, we could use a map, it returns null when the quote is failed, and returns the quote when it is not failed, and then you can use the `compact` function from lodash" |

---

## 22. Use `some()` Over `forEach` + Throw for Validation

**Rule:** Use array `.some()` for validation checks instead of `forEach` with throw.

**Frequency:** Low - flagged in 1 PR with 2 instances.

| PR | Reviewer | Quote |
|---|---|---|
| #377 | Reviewer E | Suggested `const isOneOnDifferentChain = positions.some((position) => position.chainId !== chainId)` over `forEach` with throw |

---

## 23. Use `encodeFunctionData` Over `populateTransaction`

**Rule:** For building transaction data, prefer `interface.encodeFunctionData()` which returns a string synchronously, over `populateTransaction` which returns a Promise.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #377 | Reviewer E | "If you like, I can show you how to build transaction data without promises" - showed `encodeFunctionData` pattern |

---

## 24. Use Position's `chainId` Over Global Network

**Rule:** When working with position data, use the position's own `chainId` property instead of relying on the user's currently selected network.

**Frequency:** Low but important - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #446 | Reviewer A | "Positions have a `chainId` parameter, what do you think about using that instead of relying on the users selected/current network?" |

---

## 25. Function/Variable Naming Clarity

**Rule:** Names should be descriptive and accurate. Boolean functions should be named as questions. Misleading names must be fixed.

**Frequency:** High - flagged in 5+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #43 | Reviewer F | "Should be call it `oldestActivePosition`? I think it's more accurate" |
| #152 | Reviewer C | "tokenAddressOrSymbol?" (suggesting more accurate name) |
| #208 | Reviewer F | "Can we rename `ignoreBlacklist` to `useBlacklist` or `listenToBlacklist`?" |
| #432 | Reviewer C | "I'd do `doesCompanionNeedWithdrawPermission` or something more verbose" |
| #478 | Reviewer B | "Should change to `useWalletNetwork`" (function named `useUnderlyingAmount` but returns network) |

---

## 26. Avoid Empty Styled Components

**Rule:** Don't create empty styled components just for naming. Only style what needs styling.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #123 | Reviewer D | "What's the reason behind creating an empty styled component?" (on `const StyledCardFooterButton = styled(Button)``) |

---

## 27. Grid Item Must Have `xs` Defined

**Rule:** MUI Grid items should always have at least their `xs` property defined. Containers should not have `xs`.

**Frequency:** Moderate - flagged in 3+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #537 | Reviewer A | "Grid items should have at least their `xs` property defined" |
| #537 | Reviewer A | "Same comment as before `item` grids should always have their `xs` at least defined" |
| #537 | Reviewer A | "container does not need `xs`" |

---

## 28. Grid Cannot Be Both Item and Container

**Rule:** Per MUI recommendation, a Grid component should not be both `item` and `container` simultaneously.

**Frequency:** Moderate - flagged in 2+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #537 | Reviewer A | "sadly a grid cannot be an item AND a container at the same time per MUI recommendation" |
| #541 | Reviewer A | "every child of a container grid needs to be an item" |

---

## 29. Don't Use Grid as a Replacement for Box/div

**Rule:** Use Grid only for responsive layouts. For simple flex containers, use `div` or `ContainerBox`.

**Frequency:** Low but clearly stated.

| PR | Reviewer | Quote |
|---|---|---|
| #541 | Reviewer A | "I feel like you are using the Grid as the new Box, if you just need a display flex property use a div. A Grid should be used when we want to format elements to be reorganized when screen sizes change" |

---

## 30. Avoid `!important` in CSS

**Rule:** Avoid `!important` in CSS. Find proper specificity solutions instead.

**Frequency:** Low - flagged when seen.

| PR | Reviewer | Quote |
|---|---|---|
| #432 | Reviewer C | "good ol' important xd" |
| #579 | Reviewer A | "I love the smell of `!important` in the morning" (sarcastic) |
| #579 | Reviewer A | "those are a lot of &, why do you need those? should only be one" (on `&&&.Mui-selected`) |

---

## 31. Use Theme Colors, Not Hardcoded Hex Values

**Rule:** Always use theme color variables instead of hardcoded hex values. SVG icons should not have hardcoded `fill` colors.

**Frequency:** High - flagged in 5+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #139 | Reviewer A | "Can we use this for default: #238636" |
| #139 | Reviewer A | "lets not change this green" |
| #516 | Reviewer A | "remember to remove the `fills` on all the new icons! if not they are always going to have the same color" |
| #557 | Reviewer A | "shouldn't #1F0E37 be a variable?" |
| #575 | Reviewer A | "should this colors be defined here? what about the theme?" |
| #592 | Reviewer B | "I believe disabledText should be changed to a themeMode-based color" |

---

## 32. Remove Unused Dependencies and Dead Code

**Rule:** Remove any unused imports, dependencies, and packages.

**Frequency:** Moderate - flagged in 3+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #465 | Reviewer A | "we can remove this property altogether" (unused package.json property) |
| #550 | Reviewer A | "are you actually using this?" (on `typed-emitter` dependency) |
| #576 | Reviewer B | "This upgrade should not be necessary right?" |

---

## 33. Services Should Not Directly Expose Internal State

**Rule:** Services should expose data through methods, not direct property access. Avoid services having side effects on other services.

**Frequency:** Moderate - flagged in 3+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #228 | Reviewer A | "we should probably create functions for all of this" / "So the transactionService does not know how internally the providerService works" |
| #545 | Reviewer A | "I'm not keen on services side effects" |
| #545 | Reviewer A | "I think we should still keep this, so we don't `bypass` the underlying data logic through the UI" |

---

## 34. No Unnecessary Parentheses or Returns

**Rule:** Remove unnecessary parentheses, return statements in arrow functions, and wrapper fragments.

**Frequency:** Moderate - flagged in 3+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #139 | Reviewer A | "no need for this right? should just be onClick: handleApproveToken" (removing wrapper arrow function) |
| #489 | Reviewer A | "no need for parenthesis here" |
| #508 | Reviewer A | "no need for return. You can just `=> (...component)`" |

---

## 35. Use `toLocaleString` for Dates

**Rule:** Use `DateTime.DATE_MED` or `toLocaleString` for date formatting instead of hardcoded format strings, to support localization.

**Frequency:** Moderate - flagged in 2 PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #525 | Reviewer A | "we should use `ToLocaleFormat` for this ones" |
| #545 | Reviewer A | "DateTime.DATE_MED is what you want" / "use toLocaleString so it automatically updates once we change the language" |

---

## 36. Use MUI Component Props Over Styled Overrides

**Rule:** Prefer using MUI component props (like `variant="outlined"`, `elevation=0`, `dense`) over creating styled component overrides.

**Frequency:** Moderate - flagged in 3+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #537 | Reviewer A | "does not the background paper already this styles? You can use elevation=0 for it to be outlined" |
| #537 | Reviewer A | "What about the variant outline of the paper?" |
| #553 | Reviewer A | "cant you make this with `dense` prop?" |

---

## 37. Use `Skeleton variant="text"` for Loading States

**Rule:** Use MUI Skeleton's `text` variant (with Typography's font size) instead of manually defining width and height.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #508 | Reviewer A | "can't you use the `text` variant? So we don't have to define width and height" |
| #508 | Reviewer A | "lets use typography for this please" (on `sx={{ fontSize: '18px' }}`) |

---

## 38. Use `useReducer` Async Thunk States

**Rule:** For Redux async thunks, use `.pending`, `.fulfilled`, `.rejected` matchers instead of manual loading state actions.

**Frequency:** Moderate - flagged in 1 PR with multiple instances.

| PR | Reviewer | Quote |
|---|---|---|
| #499 | Reviewer A | "why do you need this action? The fetchPricesForChain should be able to use a `pending` state on the reducer" |
| #499 | Reviewer A | "here for example you can use `fetchPricesForChain.pending`" |

---

## 39. Return `null` Instead of Empty Fragment

**Rule:** Components that conditionally render nothing should return `null` instead of `<></>`.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #499 | Reviewer A | "You can return null" (instead of `return <></>;`) |

---

## 40. Avoid `async` When Not Needed

**Rule:** Don't use `async` on functions that don't need to `await` anything.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #499 | Reviewer A | "no async needed here" (on a map callback that just dispatches) |

---

## 41. Use `eslint-disable-next-line` Over Block Disable

**Rule:** Prefer `eslint-disable-next-line` for single-line overrides instead of `eslint-disable` / `eslint-enable` blocks.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #134 | Reviewer A | "instead of enabling and disabling eslint-rules we can just do `// eslint-disable-next-line <eslint rule>` And it just works for one line" |

---

## 42. Event Tracking Should Be Verbose

**Rule:** When tracking events with Mixpanel/analytics, be as verbose as possible with properties. Use explicit string values (e.g., `is: 'from'`) instead of booleans.

**Frequency:** Low - flagged in 1 PR with multiple instances.

| PR | Reviewer | Quote |
|---|---|---|
| #356 | Reviewer C | "From my experience with mixpanel it's better to be as verbose as possible, for example: is: 'from' or 'to'" |
| #356 | Reviewer C | "Maybe a 'from source' - 'to source' might be cool here!" |

---

## 43. Blacklist Comments Should Explain Why

**Rule:** When adding tokens to blacklists, include a comment explaining why they are blacklisted.

**Frequency:** Moderate - flagged in 2+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #249 | Reviewer F | Suggested adding `// ARBI - LPT. Disabled due to liquidity decrease` |
| #338 | Reviewer E | Suggested more descriptive comment `// POLY - stMATIC on Aave v3` |

---

## 44. Use Object Destructuring

**Rule:** Prefer object destructuring for state and function returns.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #134 | Reviewer A | "could we use object destructuring here?" |
| #134 | Reviewer A | "Same here, lets use object destructuring" |

---

## 45. Don't Spread Previous State When You Want to Reset

**Rule:** When updating state after an error, don't spread previous state if you want to reset certain values.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #134 | Reviewer A | "we probably don't want to spread the previous state, since we want to set result as undefined if it has failed" |

---

## 46. Use `reduce<Type>()` for Type Safety

**Rule:** Use generic type parameter on `reduce<T>()` instead of `as T` assertion at the end.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #499 | Reviewer A | "you can use `reduce<TokenBalancesAndPrices>` and will have no need for the bottom `as` plus it will typecheck the reducer function return value" |

---

## 47. Use Font Weight as Numbers, Not Strings

**Rule:** Use numeric values for `fontWeight` (e.g., 600), and pass them as styled-component attrs.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #570 | Reviewer A | "we try to use numbers for font weight, and you can send them as attrs" |

---

## 48. Reusable Components Should Go in `common`

**Rule:** Components that will be used across multiple features should be placed in `common` directory.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #489 | Reviewer A | "Nice component! It could go directly into `common` since we'll probably use it in other products" |

---

## 49. Props Should Use Existing SDK/Common Types

**Rule:** Use existing type definitions from the SDK or common-types package (e.g., `ChainId`, `TokenAddress`) instead of creating new ones.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #581 | Reviewer B | "There are types defined as ChainId and TokenAddress that facilitates readability for cases like this" |

---

## 50. Query Only What You Need from GraphQL/APIs

**Rule:** Don't request fields from APIs that you don't use.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #3 | Reviewer F | "I think you are only using `volumeUSD`. If so, then why are you asking for all the other properties?" |

---

## 51. Use `ContainerBox` Component Gap Prop, Not Styled Gaps

**Rule:** The `ContainerBox` component has a `gap` prop. Use it instead of adding CSS gap in styled overrides.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #580 | Reviewer B | "You can use the gap attribute" (on ContainerBox) |

---

## 52. Show Loading States Only When No Data Available

**Rule:** Only show loading indicators (skeletons, spinners) when there is NO existing data. If data exists and is being refreshed, don't show loading.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #499 | Reviewer A | "We should only show the loading indicator when there is NO value for the balance and it is loading. In case there is a value for the balance and it is loading, we should not show a loading indicator" |

---

## 53. Hooks With No Return Value Are Discouraged

**Rule:** Custom hooks that only trigger side effects (no return value) can "rapidly start polluting the code." Prefer hooks that return values.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #499 | Reviewer A | "This hooks with no return value can very rapidly start polluting the code" |

---

## 54. Merge Multiple Initial useEffects

**Rule:** When multiple useEffect hooks only run on mount (empty or same deps), merge them into one.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #537 | Reviewer A | "There are quite a few of this useEffects that only run on render, lets merge them all" |

---

## 55. Moved Constant Computations Outside Components

**Rule:** Values that don't depend on component state or props should be defined outside the component to prevent recreation on every render.

**Frequency:** Moderate - flagged in 2+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #489 | Reviewer A | "We could generate this ones outside the component so they are not different on every re-render" |
| #489 | Reviewer A | "Is this actually using anything from the component? If it isnt maybe we can declare it outside" |

---

## 56. Handle Edge Cases in Token/Chain Data

**Rule:** Always handle edge cases: tokens with same address on different chains, tokens not in token lists, undefined balances, etc.

**Frequency:** High - flagged across many PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #357 | Reviewer A | "Couldn't this generate a clash if we have two tokens with the same address on different chains?" |
| #450 | Reviewer A | "are you sure this underlying address is correct?" |
| #454 | Reviewer A | "why this added condition?" (on price calculation logic) |
| #514 | Reviewer A | "is this already chainSpecific? bc we could be fetching a token from another chain if it has the same address" |

---

## 57. Use `BackgroundPaper` or `ForegroundPaper` Over Raw `Paper`

**Rule:** Use the project's styled Paper variants instead of raw MUI Paper.

**Frequency:** Moderate - flagged in 2+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #574 | Reviewer A | "shouldn't we use a Foreground or Background paper for this?" |
| #597 | Reviewer B | "I think a BackgroundPaper suits better in order to have background: quartery" |

---

## 58. Use `catch(handleError)` Shorthand

**Rule:** When passing an error handler to `.catch()`, use the function reference directly instead of wrapping it.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #591 | Reviewer A | "could be just `catch(handleError)`" (instead of `.catch((error) => handleError(error))`) |

---

## 59. SVG Assets Should Use CDN/Token List

**Rule:** SVG assets for tokens/protocols should be uploaded to the CDN and referenced via the token list, not bundled in the app.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #518 | Reviewer A | "this should go to our token list" / "Uploaded this to cdn for you... Lets use it through our token list" |

---

## 60. Use Yield Pool ID Without Full URL

**Rule:** Yield pool IDs should be just the UUID, not the full DeFiLlama URL.

**Frequency:** Low - flagged in 1 PR with 2 instances.

| PR | Reviewer | Quote |
|---|---|---|
| #308 | Reviewer A | Suggested `'173ba0e8-d88f-43f4-9339-0e23a35a4cb0'` instead of `'https://defillama.com/yields/pool/173ba0e8-...'` |

---

## 61. Props Should Be Minimal and Focused

**Rule:** Components should receive only the props they need. Don't pass entire objects when only one field is needed. Use token's own chainId instead of passing it separately.

**Frequency:** Moderate - flagged in 3+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #139 | Reviewer A | "I would actually pass the token and the amount here, let the button define which text is going to show" |
| #489 | Reviewer A | "no need to pass `currentChainId`, `selectedToken` should already have its `chainId` set" |

---

## 62. Use Paper `variant="outlined"` for Borders

**Rule:** Use MUI Paper's `variant="outlined"` prop instead of manually adding border styles.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #537 | Reviewer A | "What about the variant outline of the paper? That should get you the border" |

---

## 63. MUI Theme Component Overrides Over Styled Selectors

**Rule:** Override MUI component styles through the theme's `components` configuration, not by targeting MUI class names in styled-components.

**Frequency:** High - flagged in 5+ PRs.

| PR | Reviewer | Quote |
|---|---|---|
| #201 | Reviewer D | "This will work for now, but might fail on future MUI updates. Could we switch to a custom selector?" |
| #516 | Reviewer A | "you could apply this styles directly to the MenuItem styles in general on `theme/components`" |
| #516 | Reviewer A | "If this is for all menus and stuff, maybe we should put this directly in the `theme/component` definition for the MuiDivider" |
| #541 | Reviewer A | "you can use the `focused` property (instead of `root`)" / "same for disabled" |

---

## 64. Test Error Cases With Descriptive Messages

**Rule:** In tests, use descriptive error messages instead of `expect(1).toEqual(2)` for verifying code should have thrown.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #418 | Reviewer D | "You can replace `expect(1).toEqual(2)` for `throw new Error('The above request succeeded when it should have failed')` to make it more explicit" |

---

## 65. Handle Optimistic UI Updates Correctly

**Rule:** When doing optimistic updates, make sure to properly revert on failure. Place operations that should be reverted inside the try block.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #474 | Reviewer A | "should this actually be inside the try? since we want to revert all that when it fails right?" |

---

## 66. Use Existing Constants

**Rule:** Use existing constants (like `LATEST_VERSION`) instead of computing them from arrays.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #442 | Reviewer A | "There is a constant that you can use `LATEST_VERSION`" (instead of `POSITIONS_VERSIONS[POSITIONS_VERSIONS.length - 1]`) |

---

## 67. Show Only Loading When Needed, Fetch Proactively

**Rule:** Fetch data proactively (during login, on app load) so users don't see loading states when navigating.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #517 | Reviewer A | "this is something that we should do once the user logs in, but not await that promise to finish the login process... If we can load it asynchronously while the user is in another tab we can save more time of the user viewing loading states" |

---

## 68. String Literal Formatting in JSX

**Rule:** Use proper string formatting. No unnecessary curly braces for string literals in JSX.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #525 | Reviewer A | Suggested `maxWidth="16ch"` instead of `maxWidth={'16ch'}` |

---

## 69. No Unnecessary Break After Return in Switch

**Rule:** Remove `break` statements after `return` in switch cases.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #523 | Reviewer B | "We could safely remove break statements due to early returns" |

---

## 70. Component vs UI Library Placement

**Rule:** Only put truly reusable UI components in the `ui-library` package. Feature-specific components stay in the app.

**Frequency:** Low - flagged in 1 PR.

| PR | Reviewer | Quote |
|---|---|---|
| #537 | Reviewer A | "This is not going to be a reusable component right? It should go directly wherever its going to be used" |

---

## Summary of Most Frequently Flagged Issues

Ranked by approximate frequency across all PRs:

1. **Remove commented-out code** (10+ PRs)
2. **Use theme spacing, not hardcoded px** (10+ PRs)
3. **Fix typos and spelling** (15+ PRs)
4. **Use optional chaining / nullish coalescing** (8+ PRs)
5. **Abstract repeated logic** (8+ PRs)
6. **Use theme colors, not hex values** (5+ PRs)
7. **FormattedMessage for i18n** (5+ PRs)
8. **Proper memoization** (6+ PRs)
9. **Typography variants over custom styles** (5+ PRs)
10. **MUI theme overrides over class selectors** (5+ PRs)
11. **No console.log** (5+ PRs)
12. **Correct useEffect dependencies** (5+ PRs)
13. **Proper copy punctuation** (5+ PRs)
14. **Handle edge cases in token/chain data** (5+ PRs)
15. **Use enums over union types** (3+ PRs)
16. **Pin SDK versions** (3+ PRs)
17. **Descriptive naming** (5+ PRs)
18. **Grid items need `xs`** (3+ PRs)
19. **No inline styles on MUI** (3+ PRs)
20. **Secrets in env vars** (2+ PRs)
