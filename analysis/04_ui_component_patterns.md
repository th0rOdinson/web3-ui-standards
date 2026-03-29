# UI Component Patterns - Web3 Frontend

## 1. Component Library

The project uses **MUI v5 (Material UI 5.14.12)** as the base component library, with **styled-components v5** as the styling engine (not Emotion). The MUI styled engine is swapped via `@mui/styled-engine-sc`.

Key dependencies:
- `@mui/material: 5.14.12` - Core UI components
- `@mui/icons-material: 5.14.12` - Icon set
- `styled-components: ^5.3.1` - CSS-in-JS engine
- `tss-react: ^4.9.0` - Used occasionally for `withStyles`
- `react-intl: ^6.3.2` - Internationalization
- `notistack: ^3.0.1` - Snackbar/toast notifications
- `recharts: 2.12.7` - Charts (pie charts, graphs)
- `react-virtuoso: ^4.6.2` - Virtualized lists/tables
- `luxon: ^2.1.1` - Date handling

The project has a **monorepo structure** with a dedicated `packages/ui-library` package that wraps and re-exports MUI components with project-specific styling.

## 2. Component Structure

### Monorepo Layout
```
packages/
  ui-library/           # Shared UI component library
    src/
      components/       # All component wrappers
      theme/            # Theme definitions
      icons/            # Custom SVG icons
      assets/           # Visual assets (donut shapes, logos)
      emojis/           # Emoji components
      common/           # Utils, notistack re-exports
      index.ts          # Single barrel export
apps/
  root/
    src/
      common/components/   # App-specific shared components
      pages/               # Page-level components
      hooks/               # Custom hooks
      constants/           # Constants and config
```

### Component File Convention
Each UI library component lives in its own directory with an `index.tsx` file:
```
packages/ui-library/src/components/
  button/index.tsx
  container-box/index.tsx
  modal/index.tsx
  ...
```

All components are re-exported from `packages/ui-library/src/components/index.tsx` as barrel exports, and then from `packages/ui-library/src/index.ts`.

### Import Pattern
App code imports everything from `ui-library`:
```tsx
import { Button, ContainerBox, Typography, Modal, colors, useSnackbar } from 'ui-library';
```

### Component Wrapping Pattern
Most UI library components are thin wrappers around MUI that add project-specific defaults or styling:

Simple re-export (no customization):
```tsx
// typography/index.tsx
import type { TypographyProps } from '@mui/material/Typography';
import Typography from '@mui/material/Typography';
export { Typography, type TypographyProps };
```

Styled wrapper (common pattern):
```tsx
// button/index.tsx
const Button = styled(MuiButton)<{ maxWidth?: CSSProperties['maxWidth'] }>`
  ${({ theme: { spacing }, maxWidth }) => `
  max-width: ${maxWidth ?? spacing(87.5)};
`}
`;
export { Button, type ButtonProps };
```

## 3. Styling Approach

### Primary: styled-components with Theme Access
The dominant pattern uses styled-components with theme destructuring. The theme object comes from MUI's `createTheme` but is also provided to styled-components via `SCThemeProvider`.

**Standard styled-component pattern:**
```tsx
const StyledContainer = styled(ContainerBox)`
  ${({ theme: { palette, spacing } }) => `
    background-color: ${colors[palette.mode].background.secondary};
    border-radius: ${spacing(4)};
    padding: ${spacing(3)};
  `}
`;
```

**styled-components with `.attrs()` for default props:**
```tsx
const StyledQuestion = styled(Typography).attrs({ variant: 'h4Bold' })`
  ${({ theme: { palette } }) => `
  color: ${colors[palette.mode].typography.typo2};
  `}
`;
```

**Typed styled-components with transient props ($prefix):**
```tsx
const StyledOverlay = styled.div<{ $isAbsolute: boolean }>`
  ${({ theme: { spacing }, $isAbsolute }) => `
    display: flex;
    ${$isAbsolute && `padding: ${spacing(6)};`}
  `}
`;
```

### Secondary: MUI `sx` Prop
Used inline for one-off styles, often with theme callback:
```tsx
<Typography
  variant="bodySmallRegular"
  color={({ palette }) => colors[palette.mode].typography.typo2}
  sx={{ display: 'flex', alignItems: 'center', gap: ({ spacing }) => spacing(1) }}
/>
```

### Color Access Pattern
Colors are ALWAYS accessed through the `colors` object from the theme, using `palette.mode` to switch between light/dark:
```tsx
import { colors } from 'ui-library'; // or '../../theme'

// In styled-components:
color: ${colors[palette.mode].typography.typo2};

// In sx prop:
color={({ palette }) => colors[palette.mode].typography.typo2}

// In component via useTheme:
const { palette: { mode } } = useTheme();
<Icon sx={{ color: colors[mode].typography.typo3 }} />
```

**NEVER use hardcoded hex colors.** The codebase systematically replaced `#ffffff`, `rgba(...)` and `baseColors.disabledText` references with the semantic `colors[mode].*` tokens.

## 4. Theming Patterns

### Theme Provider Setup
Dual theme providers wrap the app - MUI's ThemeProvider and styled-components' ThemeProvider share the same theme object:
```tsx
// packages/ui-library/src/components/theme-provider/index.tsx
const ThemeProvider = ({ mode, children }: ThemeProviderProps) => (
  <MuiThemeProvider theme={mode === 'dark' ? darkTheme : lightTheme}>
    <SCThemeProvider theme={mode === 'dark' ? darkTheme : lightTheme}>
      <CssBaseline />
      {children}
    </SCThemeProvider>
  </MuiThemeProvider>
);
```

### Dark/Light Mode
Two complete themes are built:
```tsx
export const darkTheme = createTheme({
  typography: buildTypographyVariant('dark'),
  palette: darkModePallete,
  components: darkModeComponents,
  spacing: DEFAULT_SPACING, // 4
  space: baseSpacingScale,
  shape: { borderRadius: DEFAULT_BORDER_RADIUS }, // 16
});
```

Mode toggle is stored in Redux state (`@state/config`) and accessed via `useThemeMode()`.

### Color Token System

**Base colors** (raw palette):
- `violet`: violet100-violet900 (brand purple)
- `aqua`: aqua100-aqua900 (brand teal/green)
- `greyscale`: greyscale0-greyscale9
- `semantic.green/red/yellow`: 100-900 ranges

**Semantic colors** (mode-aware, the ones you actually use):

| Token Path | Purpose |
|---|---|
| `colors[mode].typography.typo1` | Primary text (headings) |
| `colors[mode].typography.typo2` | Secondary text (body, labels) |
| `colors[mode].typography.typo3` | Tertiary text (descriptions, captions) |
| `colors[mode].typography.typo4` | Muted text |
| `colors[mode].typography.typo5` | Disabled/placeholder text |
| `colors[mode].background.primary` | Page background |
| `colors[mode].background.secondary` | Card/paper background |
| `colors[mode].background.tertiary` | Elevated/focused surfaces |
| `colors[mode].background.quartery` | Backdrop paper (with alpha) |
| `colors[mode].background.modals` | Modal background |
| `colors[mode].background.emphasis` | Emphasized sections |
| `colors[mode].border.border1` | Primary border |
| `colors[mode].border.border2` | Secondary border |
| `colors[mode].border.border3` | Subtle border (with alpha) |
| `colors[mode].border.accent` | Accent-colored border |
| `colors[mode].accent.primary` | Primary accent (aqua in dark, violet in light) |
| `colors[mode].accent.accent100-600` | Accent scale |
| `colors[mode].accent.primaryEmphasis` | Accent background |
| `colors[mode].semantic.success/warning/error` | Status colors (primary, darker, light) |
| `colors[mode].semanticBackground.success/warning/error` | Status background colors (10% opacity) |
| `colors[mode].dropShadow.dropShadow100-400` | Elevation shadows |

**Key inversion**: In dark mode, the aqua/violet scales are reversed (aqua100 = aqua900 base value), and the accent primary color swaps (dark = aqua, light = violet).

### Spacing System
Base unit is `4px`. Spacing function: `SPACING(n) = n * 4 + "px"`.

```tsx
// Usage via theme
theme.spacing(3)  // "12px"
theme.spacing(6)  // "24px"

// Spacing scale constants
space.s01 = 4px
space.s02 = 8px
space.s03 = 12px
space.s04 = 16px
space.s05 = 24px
space.s06 = 32px
space.s07 = 48px
```

Common spacing values: `gap={2}` (8px), `gap={3}` (12px), `gap={4}` (16px), `gap={6}` (24px).

### Border Radius
Default: 16px (`DEFAULT_BORDER_RADIUS`).
Common overrides: `spacing(2)` (8px) for inner elements, `spacing(4)` (16px) for cards, `spacing(15)` or `spacing(25)` for pills.

## 5. Typography Patterns

Font: **Inter** (sans-serif)

### Typography Variants

| Variant | Size | Weight | Default Color | Use |
|---|---|---|---|---|
| `h1Bold` | 2.5rem (40px) | 800 | typo1 | Page titles |
| `h2Bold` | 2rem (32px) | 700 | - | Section titles |
| `h3Bold` | 1.5rem (24px) | 700 | typo1 | Modal titles, subsections |
| `h4Bold` | 1.25rem (20px) | 600 | - | Card headers, FAQ questions |
| `h5Bold` | 1.125rem (18px) | 600 | typo2 | Subsection headers |
| `h6Bold` | 1rem (16px) | 600 | typo2 | Small headers |
| `bodyLargeRegular` | 1.125rem (18px) | 400 | typo3 | Page descriptions |
| `bodyLargeBold` | 1.125rem (18px) | 700 | typo3 | Emphasized body large |
| `bodyRegular` | 1rem (16px) | 500 | typo2 | Default body text |
| `bodySemibold` | 1rem (16px) | 600 | typo2 | Semi-bold body |
| `bodyBold` | 1rem (16px) | 700 | typo2 | Bold body (line-height: 2) |
| `bodySmallRegular` | 0.875rem (14px) | 500 | typo3 | Small body text |
| `bodySmallSemibold` | 0.875rem (14px) | 600 | typo3 | Semi-bold small |
| `bodySmallBold` | 0.875rem (14px) | 700 | typo3 | Bold small |
| `bodyExtraSmall` | 0.75rem (12px) | 500 | typo3 | Extra small |
| `bodyExtraSmallBold` | 0.75rem (12px) | 700 | typo3 | Bold extra small |
| `bodyExtraExtraSmall` | 0.625rem (10px) | 500 | typo3 | Tiniest text |
| `labelExtraLarge` | 1rem (16px) | 600 | typo2 | Large labels |
| `labelLarge` | 0.875rem (14px) | 600 | typo2 | Labels |
| `labelRegular` | 0.75rem (12px) | 500 | typo3 | Small labels, table headers |
| `labelSemiBold` | 0.75rem (12px) | 600 | typo2 | Semi-bold labels |
| `linkRegular` | 1rem (16px) | 500 | accentPrimary | Links |
| `linkSmall` | 0.875rem (14px) | 500 | accentPrimary | Small links |
| `confirmationLoading` | 3.25rem (52px) | 600 | typo2 | Transaction loading overlay |

### Responsive Typography
h1Bold-h3Bold have responsive breakpoints at `md`:
```tsx
h1Bold: {
  fontSize: '2.5rem',
  [breakpoints.down('md')]: {
    fontSize: '2.25rem',
    fontWeight: 700,
  },
}
```

### Common Typography Usage Pattern
Color override is almost always explicit via the `color` prop:
```tsx
<Typography variant="h2Bold" color={({ palette }) => colors[palette.mode].typography.typo1}>
  Title
</Typography>
<Typography variant="bodyRegular" color={({ palette }) => colors[palette.mode].typography.typo2}>
  Description
</Typography>
```

### Pre-styled Typography Variants
A common pattern is creating reusable styled Typography wrappers:
```tsx
const StyledBodySmallRegularTypo2 = styled(Typography).attrs(
  ({ theme: { palette: { mode } }, ...rest }) => ({
    variant: 'bodySmallRegular',
    color: colors[mode].typography.typo2,
    noWrap: true,
    ...rest,
  })
)``;
```

## 6. Layout Patterns

### ContainerBox - The Primary Layout Primitive
Custom flex container that is the most-used layout component:
```tsx
interface ContainerBoxProps {
  flexDirection?: CSSProperties['flexDirection'];  // default: 'row'
  justifyContent?: CSSProperties['justifyContent']; // default: 'flex-start'
  alignItems?: CSSProperties['alignItems'];         // default: 'stretch'
  flexWrap?: CSSProperties['flexWrap'];
  flexGrow?: CSSProperties['flexGrow'];
  alignSelf?: CSSProperties['alignSelf'];
  flex?: CSSProperties['flex'];
  gap?: number;         // multiplied by theme.spacing
  fullWidth?: boolean;  // sets width: 100%
}
```

Usage:
```tsx
<ContainerBox flexDirection="column" gap={6}>
  <ContainerBox flexDirection="column" gap={2}>
    <Typography variant="h1Bold">Title</Typography>
    <Typography variant="bodyLargeRegular">Subtitle</Typography>
  </ContainerBox>
  <ContainerBox gap={3} alignItems="center">
    {/* Horizontal items */}
  </ContainerBox>
</ContainerBox>
```

### Paper Hierarchy
Three levels of paper/surface components:
1. **BackgroundPaper** - `background.quartery` with `backdrop-filter: blur(30px)`, padding `space.s06` (32px)
2. **ForegroundPaper** - `background.secondary`, `border-radius: spacing(4)` (16px)
3. Modal paper - `background.modals`, padding `space.s07` (48px)

### MUI Grid for Multi-Column
```tsx
<Grid container spacing={4}>
  <Grid item xs={12} md={6}>...</Grid>
  <Grid item xs={12} md={6}>...</Grid>
</Grid>
```

### Page Layout Pattern
```tsx
<ContainerBox flexDirection="column" gap={20}>
  <ContainerBox flexDirection="column" gap={6}>
    <ContainerBox flexDirection="column" gap={2}>
      <Typography variant="h1Bold" color={...}>Page Title</Typography>
      <Typography variant="bodyLargeRegular" color={...}>Description</Typography>
    </ContainerBox>
    {/* Page content */}
  </ContainerBox>
</ContainerBox>
```

### Max Width
Forms have a standard max width: `MAX_FORM_WIDTH = SPACING(158)` = 632px.

## 7. Form Patterns

### InputContainer
A styled flex container with states for focused/disabled/hasValue:
```tsx
<InputContainer isFocused={isFocused} disabled={disabled} hasValue={hasValue}>
  {/* Input content */}
</InputContainer>
```

States map to different background/border combos:
- Empty: `background.secondary` + `border.border1`
- Focused: `background.tertiary` + `accentPrimary` border
- Disabled: `background.secondary` + `accentPrimary` border + `opacity: 0.5`

### TokenAmountUsdInput
Complex composite input for token amounts with USD conversion toggle:
```tsx
<TokenAmounUsdInput
  token={selectedToken}
  balance={balance}
  tokenPrice={price}
  value={amount}
  onChange={setAmount}
  onMaxCallback={handleMax}
/>
```

### TextField Usage
MUI TextField with theme-aware styling:
```tsx
<TextField
  size="small"
  placeholder={intl.formatMessage(defineMessage({ ... }))}
  onChange={(e) => handleSearchChange(e.target.value)}
  InputProps={{
    startAdornment: (
      <InputAdornment position="start">
        <SearchIcon htmlColor={colors[palette.mode].typography.typo3} />
      </InputAdornment>
    ),
  }}
  fullWidth
/>
```

### Form Validation Pattern
Validation is typically handled at the hook level, with disabled states on buttons:
```tsx
<Modal
  actions={[
    {
      label: <FormattedMessage ...>,
      onClick: handleSubmit,
      disabled: !isValid || isLoading,
      variant: 'contained',
      color: 'primary',
    },
  ]}
/>
```

## 8. Modal/Dialog Patterns

### Standard Modal
The project has a custom `Modal` component wrapping MUI's `Dialog`:
```tsx
<Modal
  open={isOpen}
  onClose={handleClose}
  showCloseIcon          // Shows X button (top-right, absolutely positioned)
  showCloseButton        // Shows "Close" button in actions
  maxWidth="sm"          // MUI breakpoint
  customMaxWidth="632px" // Override maxWidth with exact value
  title={<FormattedMessage description="modal.title" defaultMessage="Edit Label" />}
  subtitle="Optional subtitle"
  headerButton={<IconButton />}  // Extra button in header
  fullHeight             // Sets height: 90vh
  keepMounted            // Keep DOM when closed
  closeOnBackdrop        // Close on backdrop click
  actionsAlignment="horizontal" | "vertical"  // Default: vertical
  actions={[
    {
      label: <FormattedMessage .../>,
      onClick: handleAction,
      disabled: false,
      color: 'primary',
      variant: 'contained',
      options: splitButtonOptions,  // Makes it a SplitButton
    },
  ]}
  extraActions={[<Button key="extra" />]}
>
  {/* Modal content */}
</Modal>
```

### Modal State Management Pattern
```tsx
const [showModal, setShowModal] = React.useState(false);

// Open
<Button onClick={() => setShowModal(true)}>Open</Button>

// Close
<Modal open={showModal} onClose={() => setShowModal(false)}>
```

### Transaction Confirmation Modal
Multi-step modal for blockchain transactions with loading/success/error states:
```tsx
const [setModalSuccess, setModalLoading, setModalError, setModalClosed] = useTransactionModal();

// Loading state
setModalLoading({
  content: (
    <Typography variant="bodyRegular">
      <FormattedMessage
        description="loading.message"
        defaultMessage="Processing your transaction..."
      />
    </Typography>
  ),
});

// Success state (shows receipt + satisfaction survey)
setModalSuccess({ ... });
```

## 9. Loading States

### Skeleton Pattern
MUI Skeleton is re-exported and used with `animation="wave"`:
```tsx
// Text skeleton
<Skeleton variant="text" animation="wave" width="15ch" />
<Skeleton variant="text" animation="wave" width="5ch" height="2ch" />

// Circular skeleton (avatars/icons)
<Skeleton variant="circular" width={32} height={32} animation="wave" />

// Using SPACING for width
<Skeleton variant="text" animation="wave" width={SPACING(25)} />
```

### CenteredLoadingIndicator
Custom component for full-section loading states.

### Loading Props Pattern
Components typically have an `isLoading` prop:
```tsx
const MyComponent = ({ data, isLoading }: Props) => {
  if (isLoading) {
    return <Skeleton ... />;
  }
  return <ActualContent data={data} />;
};
```

### CircularProgress
Used in transaction confirmation overlays:
```tsx
<CircularProgress size={spacing(20)} thickness={2.5} />
```

## 10. i18n Patterns

### Library: react-intl

### FormattedMessage (inline JSX)
The primary pattern for static text:
```tsx
<FormattedMessage
  description="unique.description.key"
  defaultMessage="The actual text shown to users"
/>
```

With interpolation:
```tsx
<FormattedMessage
  description="greeting"
  defaultMessage="Hello {name}, you have {count} items"
  values={{ name: userName, count: itemCount }}
/>
```

With rich text (JSX in values):
```tsx
<FormattedMessage
  description="howMuchToSell"
  defaultMessage="How much <b>{from}</b> are you planning to invest?"
  values={{
    from: token.symbol,
    b: (chunks) => <b>{chunks}</b>,
  }}
/>
```

### intl.formatMessage (programmatic)
For placeholders, aria labels, or non-JSX contexts:
```tsx
const intl = useIntl();

placeholder={intl.formatMessage(
  defineMessage({
    description: 'searchPlaceholder',
    defaultMessage: 'Search by name, token, or protocol',
  })
)}
```

### defineMessage (for reuse)
```tsx
const DEFAULT_CLOSE_MESSAGE = defineMessage({
  defaultMessage: 'Close',
  description: 'modal.close',
});

// Used later:
intl.formatMessage(closeMessage || DEFAULT_CLOSE_MESSAGE)
```

### Key Naming Conventions
- `description` field is the unique key (used for translation extraction)
- Format: `dotted.path.description` or `camelCaseDescription`
- Examples: `"earn.strategy-management.withdraw.modal.loading"`, `"portfolioBalanceCol"`, `"low liquidity title"`
- Keys tend to describe the UI location/purpose, not the content

### Translation Files
- Located in `lang/` directory (e.g., `lang/es.json`)
- Compiled versions in `src/config/lang/es.json`
- Auto-extracted via `babel-plugin-formatjs`
- Upload workflow: `.github/workflows/uploadTransalations.yml`
- Supported languages: English (default), Spanish, Turkish
- Language selection controlled via `ENABLED_TRANSLATIONS` env var
- Currently language selection is re-enabled in the UI after being temporarily disabled

## 11. Responsive Design

### Breakpoints
Standard MUI breakpoints (default values):
- `xs`: 0px
- `sm`: 600px
- `md`: 900px
- `lg`: 1200px
- `xl`: 1536px

### Usage Patterns

**useMediaQuery hook:**
```tsx
import { useMediaQuery } from 'ui-library';

const theme = useTheme();
const isMobile = useMediaQuery(theme.breakpoints.down('md'));
```

**useCurrentBreakpoint hook** (app-level):
```tsx
function useCurrentBreakpoint() {
  // Returns 'xs' | 'sm' | 'md' | 'lg' | 'xl'
}
```

**Dedicated mobile detection hook:**
```tsx
// useIsEarnMobile.ts
const useIsEarnMobile = () => {
  const theme = useTheme();
  return useMediaQuery(theme.breakpoints.down('md'));
};
```

**Conditional rendering:**
```tsx
const shouldShowMobileList = useMediaQuery(theme.breakpoints.down('md'));

{shouldShowMobileList ? (
  <StrategiesList ... />
) : (
  <StrategiesTable ... />
)}
```

**MUI Hidden component** (re-exported as `Hidden`):
```tsx
import { Hidden } from 'ui-library';
```

### Typography Responsiveness
h1Bold-h3Bold automatically scale down at the `md` breakpoint:
- h1Bold: 40px -> 36px
- h2Bold: 32px -> 30px
- h3Bold: 24px, weight 700 -> 600

## 12. Animation/Transitions

### CSS Transitions
Mostly simple CSS transitions for interactive states:
```tsx
// Button/interactive elements
transition: all 300ms;
transition-property: box-shadow, background-color;

// Options menu hover
transition: background-color 0.2s ease-in-out;

// Table row hover
transition: background-color 0.2s ease-in-out;

// Toggle buttons
transition: background-color 300ms, border-color 300ms;

// Chevron rotation
transition: transform 0.15s ease;
```

### MUI Transition Components
Re-exported and used for mount/unmount animations:
```tsx
import { Slide, Zoom, Grow, Collapse } from 'ui-library';

<Zoom in={open}>{content}</Zoom>
<Slide direction="up" in={visible}>{content}</Slide>
```

### Loading Animations
Keyframe animations exist in the baseline stylesheet for RainbowKit wallet connection UI and loading spinners.

## 13. Reusable Component Conventions

### Props Patterns

**Transient props** (styled-components, not forwarded to DOM):
```tsx
styled.div<{ $isActive?: boolean }>`
  ${({ $isActive }) => $isActive && `...`}
`
```

**Spread rest props:**
```tsx
const ForegroundPaper = ({ children, ...otherProps }: PaperProps) => (
  <StyledForegroundPaper {...otherProps}>{children}</StyledForegroundPaper>
);
```

**Component variants via styled + attrs:**
```tsx
const PillTab = styled(Button).attrs({
  variant: 'outlined',
  size: 'small',
  color: 'info',
})<{ selected: boolean }>`
  ${({ selected, theme: { palette: { mode } } }) =>
    selected && `
    background-color: ${colors[mode].background.secondary};
    border: 1.5px solid ${colors[mode].border.border1};
    color: ${colors[mode].accent.primary};
  `}
`;
```

### Composition Patterns

**Styled components with attrs for layout:**
```tsx
const StyledDialogHeader = styled(ContainerBox).attrs({
  justifyContent: 'space-between',
  fullWidth: true,
  alignItems: 'center',
})``;

const StyledConfirmationContainer = styled(ContainerBox).attrs({
  alignSelf: 'stretch',
  justifyContent: 'center',
  alignItems: 'center',
})``;
```

**Theme-aware attrs (dynamic defaults from theme):**
```tsx
const StyledModalDialogChildren = styled(ContainerBox).attrs(({ theme: { space } }) => ({
  gap: space.s05,
  alignItems: 'center',
  justifyContent: 'center',
  flexDirection: 'column',
  fullWidth: true,
}))`
  ${({ theme: { space } }) => `
    padding: 0px;
    flex-grow: 1;
    gap: ${space.s05}
  `}
`;
```

### Button Variants

| Variant | Color | Use |
|---|---|---|
| `contained` + `primary` | Accent bg, accent100 text | Primary actions |
| `contained` + `secondary` | Secondary bg, typo3 text | Secondary actions |
| `contained` + `error` | Error darker bg | Destructive actions |
| `outlined` + `primary` | Accent border, accent text | Alternative actions |
| `outlined` + `info` | border2, rounded pill | Filter pills |
| `text` + `primary` | Accent text, no bg | Tertiary/link actions |

Button sizes: `large` (primary CTAs), `medium`, `small` (icon buttons, pills).

### Notification Pattern (Snackbar)
```tsx
import { useSnackbar } from 'ui-library';

const snackbar = useSnackbar();
snackbar.enqueueSnackbar(
  intl.formatMessage(defineMessage({
    description: 'walletLabelUpdated',
    defaultMessage: 'Wallet label updated',
  })),
  { variant: 'success' }
);
```

Variants: `success`, `error`, `warning`. Custom styled via `StyledMaterialDesignContent` with matching icons (TickCircleIcon, WarningCircleIcon, WarningTriangleIcon).

### Token Display Components
- `TokenIcon` - Displays a token's icon image
- `ComposedTokenIcon` - Overlapping token pair icons
- `TokenIconWithNetwork` - Token icon with chain badge
- `TokenPickerButton` - Pill button showing selected token with dropdown arrow

### Table Pattern
Virtualized tables using `react-virtuoso`'s `TableVirtuoso`:
```tsx
<VirtualizedTable
  data={items}
  columns={columns}
  itemContent={renderRow}
  fixedHeaderContent={renderHeader}
/>
```

With pre-styled typography variants for cells:
```tsx
const StyledBodySmallRegularTypo2 = styled(Typography).attrs(
  ({ theme: { palette: { mode } }, ...rest }) => ({
    variant: 'bodySmallRegular',
    color: colors[mode].typography.typo2,
    noWrap: true,
    ...rest,
  })
)``;
```

### MUI Component Variant Overrides
Global component variants are defined in `packages/ui-library/src/theme/variants/` files. Each exports a `buildXxxVariant(mode)` function that returns MUI `Components` overrides. These are merged into the theme at build time.

Variant files exist for: accordion, alert, appbar, button, card, chip, circular-progress, divider, drawer, form-control-label, input-base, linear-progress, paper, popover, select, svgicon, table, toggle-button-group, tooltip.

### Divider Components
The project has custom divider variants:
```tsx
import { DividerBorder1, DividerBorder2 } from 'ui-library';
```

### Event Tracking Integration
Components frequently include event tracking:
```tsx
const trackEvent = useTrackEvent();
trackEvent('Earn - Withdraw position submitting', { strategyId: strategy.id });
```

## Summary of Conventions for Component Generation

When generating new UI components for this codebase:

1. **Import from `ui-library`**, not directly from MUI
2. **Use `styled-components`** with theme interpolation, not Emotion or CSS modules
3. **Access colors via `colors[mode].*`**, never hardcode hex values
4. **Use `ContainerBox`** as the primary layout primitive (not raw `div` or `Box`)
5. **Use the project's Typography variants** (h1Bold, bodyRegular, etc.), not MUI defaults (h1, body1)
6. **Wrap text in `<FormattedMessage>`** with `description` and `defaultMessage` props
7. **Use `theme.spacing(n)`** for all spacing values (base unit 4px)
8. **Use transient props (`$prefix`)** for styled-component-only props
9. **Create pre-styled Typography wrappers** with `.attrs()` for repeated text styles
10. **Use `BackgroundPaper`/`ForegroundPaper`** instead of raw `Paper`
11. **Use the `Modal` component** for dialogs, not raw MUI Dialog
12. **Use `Skeleton` with `animation="wave"`** for loading states
13. **Use `useMediaQuery` with `theme.breakpoints.down('md')`** for mobile detection
14. **Use `useSnackbar().enqueueSnackbar()`** for notifications
15. **Keep spacing consistent**: gap={2-6} for ContainerBox, spacing(3-6) for padding
