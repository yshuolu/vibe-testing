# Vibe Testing

**A Claude Code skill that teaches AI agents to test like humans.**

Coding agents write code, run unit tests, see green checkmarks, and call it done. But unit tests don't catch the button that doesn't click, the page that doesn't load, or the API that returns the wrong data. Vibe Testing fixes this — it teaches agents to actually *use* the product they just changed.

## What It Does

After every code change, the agent:

1. **Starts the app** and verifies it runs
2. **Opens a browser** with the developer's authenticated session — no login flow needed
3. **Interacts with the product** like a real user would — clicks, fills forms, navigates
4. **Screenshots and snapshots** as proof that it works
5. **Breaks it** — tries bad inputs, edge cases, error states

Not "the test passes." The product works.

## Install

```bash
npx skills add https://github.com/yshuolu/vibe-testing --skill vibe-testing
```

### Dependencies

This skill uses [playwright-cli](https://github.com/yshuolu/playwright-cli) for browser automation. Install it once:

```bash
git clone https://github.com/yshuolu/playwright-cli.git ~/playwright-cli
cd ~/playwright-cli && npm install
```

## How Auth Works

The #1 blocker for agent testing is authentication. The agent launches a clean browser, hits a login wall, and gives up.

Vibe Testing solves this with a two-path approach:

**Path 1 — Steal the developer's session** (default). The agent runs `playwright-cli open <url> --cookies`, which copies the developer's Chrome profile, extracts cookies + localStorage via CDP, and injects them into a fresh Playwright browser. If the developer is logged in, the agent is logged in.

**Path 2 — Test user fallback**. If no browser session exists, the agent creates a real test user in the database (with an `is_test_user` flag) and authenticates via a dev-only middleware bypass. No username/password login. The skill includes framework-specific examples for Next.js, Firebase, Supabase, Django, Rails, and Express.

## What's Covered

| Change type | How it's tested |
|---|---|
| **UI / Full-stack** | Browser via playwright-cli — navigate, click, fill, screenshot, ARIA snapshot |
| **Backend API** | `curl` the running server, read the response, test the full CRUD lifecycle |
| **CLI tool** | Run it, check stdout/stderr, verify exit codes |
| **Library/package** | Write a tiny consumer script, run it |
| **Unit logic** | Standard unit tests (the agent runs the existing test suite) |

Plus a **UI Quality Check** on every UI change — spacing, alignment, contrast, touch targets.

## Why Not Just Use Playwright MCP?

`@anthropic-ai/playwright-mcp` launches a clean browser with no auth state. For public pages, that's fine. For anything behind a login, the agent is stuck.

[playwright-cli](https://github.com/yshuolu/playwright-cli) solves this by extracting the developer's browser session and injecting it automatically. One command, authenticated browser, no login flow.

## License

MIT
