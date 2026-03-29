# web3-ui-review

A code review skill for web3 frontend development. Run it before opening a PR and get senior-level feedback on your diff.

Based on patterns from building production DeFi applications, primarily the [Balmy Interface](https://github.com/Balmy-protocol/dca-fe) with 4 years of development and 1,300+ PRs.

## Quick start

### As a Claude Code skill

```bash
# Copy to your Claude Code commands directory
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/web3-review.md https://raw.githubusercontent.com/th0rOdinson/web3-ui-standards/main/review.md
```

Then in any project:
```
/web3-review
```

### As a project-level skill

```bash
# Copy to your project's Claude commands
mkdir -p .claude/commands
curl -o .claude/commands/web3-review.md https://raw.githubusercontent.com/th0rOdinson/web3-ui-standards/main/review.md
```

## What it catches

**Critical** - Financial math with floating point, address comparison with `===`, hardcoded secrets, wrong chainId usage, token display with `.toFixed()`

**High** - console.log, commented-out code, hardcoded colors/pixels, missing i18n, unlimited approvals, multi-chain fetch without error isolation

**Medium** - Unnecessary async, over-memoization, constants inside components, unpinned deps, stateful services, token comparison without chainId

**Low** - Verbose booleans, manual null chains, naming conventions, unnecessary syntax

## Files

- `review.md` - The review skill (this is what you use)
- `rules.md` - Full reference of all standards (background reading)
- `analysis/` - Raw analysis from the source codebase

## License

MIT
