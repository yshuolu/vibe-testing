---
name: vibe-testing
description: "End-to-end testing that verifies code changes by actually using the running product. Opens a real browser with auth, clicks through, screenshots, and breaks things."
---

# Vibe Testing

Here's the thing about testing that most agents get completely wrong: they write some unit tests, see green checkmarks, and call it a day. That's not testing. That's wishful thinking.

Vibe testing means you actually *use* the thing you just built. You run it. You click it. You curl it. You see with your own eyes that it works. Just like a human would before shipping.

## Setup

### Install playwright-cli

We use [playwright-cli](https://github.com/yshuolu/playwright-cli) for browser testing. It's designed specifically for agents verifying code changes against running apps. The biggest problem with standard Playwright tooling is authentication — the agent launches a clean browser with no sessions, hits a login wall, and gives up. playwright-cli solves this by automatically copying the developer's Chrome profile, extracting cookies and localStorage via CDP, and injecting them into a fresh Playwright browser. The agent gets an authenticated session in one command, no login flow needed.

Clone and install inside the project (one-time setup):

```bash
git clone https://github.com/yshuolu/playwright-cli.git .claude/tools/playwright-cli
cd .claude/tools/playwright-cli && npm install
```

Make sure `.claude/tools/` is in `.gitignore` so it doesn't get committed to the project repo.

Run commands via:

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts <command>
```

### Verify

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts -h
npx tsx .claude/tools/playwright-cli/src/cli.ts profiles
```

## The Three Rules

**1. You ship it, you verify it.** Never say "you can test this by..." — YOU test it. The human should receive a working change with proof. Full stop.

**2. Green tests prove nothing about the product.** Unit tests can pass while the product is completely broken. Tests verify your *assumptions*. The product verifies *reality*. These are different things.

**3. You have to see it.** Whatever you changed, you have to observe the actual behavior:
- UI or fullstack change? Open a browser, navigate there, look at it. Screenshot it and verify UI quality.
- API change? `curl` the running server, read the response body.
- CLI change? Run the command, check the output.
- Library change? Write a tiny script that imports it, run the script.

No exceptions. If you didn't see it work in the running product, you didn't verify it.

## E2E Decision Tree

```
Detect environment:
│
│  Is there a local Chromium browser with a user profile?
│  (Chrome, Brave, Edge, or Arc installed with at least one profile)
│
├─ YES → likely a developer's laptop → SCENARIO A
├─ NO  → likely an isolated environment → SCENARIO B
└─ UNCLEAR → ask the developer
│
├─ SCENARIO A: Developer's laptop
│   │
│   │  No plumbing needed — just use the existing browser session.
│   │
│   │  playwright-cli open <url> --cookies
│   │
│   └─ Take a snapshot. Logged in?
│       ├─ YES → go to Test ↓
│       └─ NO (expired, wrong profile) → try --profile, or switch to Scenario B
│
├─ SCENARIO B: Isolated environment (cloud agent, CI, no system browser)
│   │
│   │  No browser profile to steal from. The project must have test user
│   │  plumbing — a real user in the DB + a middleware bypass.
│   │
│   └─ Is the stack ready?
│       │
│       │  Check: test user exists (is_test_user flag), middleware bypass
│       │  exists (TEST_USER_ID env var), app can start.
│       │
│       ├─ YES → set TEST_USER_ID in .env, start the app, go to Test ↓
│       │
│       └─ NO → notify the developer, then fix the plumbing:
│           ├─ No test user in DB → create one with is_test_user flag
│           ├─ No middleware bypass → add if block to auth function
│           ├─ App won't start → fix env vars, deps, DB (local-env-secrets.md)
│           └─ Commit plumbing changes alongside your feature change
│
Test:
│
├─ API change → curl the endpoints (e2e-server-api-test.md)
├─ UI / Full-stack → playwright-cli interact (e2e-ui-and-fullstack.md)
├─ CLI tool → run it, check output (e2e-cli-tool-test.md)
└─ Library → write consumer script (e2e-library-change-test.md)
```

See [E2E Testing](./references/e2e-test.md) for setup details, and [Test Users](./references/e2e-auth-test-user.md) for the test user methodology and framework examples.

## Browser Testing With playwright-cli

For any UI or fullstack change, use `playwright-cli` to open a browser and interact with the running app.

### Lifecycle

1. **`open` once** — launches a browser window. First run extracts cookies (~20s). Subsequent runs reuse cached state (~2s). The browser stays open as a background process.
2. **`navigate`, `click`, `fill`, `screenshot`, `snapshot`, `exec`** — interact with the open browser. Run as many as needed. Each command connects, acts, and exits instantly.
3. **`close` once** — kills the browser when done.

Do NOT close and reopen between tests. The browser stays alive. Use `navigate` to go to a new page.

### Commands

```bash
# Open (do this once per testing session)
npx tsx .claude/tools/playwright-cli/src/cli.ts open http://localhost:3000 --cookies

# Navigate to different pages (browser is already open)
npx tsx .claude/tools/playwright-cli/src/cli.ts navigate http://localhost:3000/dashboard

# Snapshot — use after EVERY navigation and interaction.
# Returns the ARIA accessibility tree (text). Fast. Tells you what's on the page,
# what elements exist, what state they're in. This is your primary verification tool.
npx tsx .claude/tools/playwright-cli/src/cli.ts snapshot

# Screenshot — use ONLY for UI quality checks (layout, spacing, colors, alignment).
# Do NOT use screenshot to check what page you're on or verify functionality.
# That's what snapshot is for.
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/page.png

# Interact
npx tsx .claude/tools/playwright-cli/src/cli.ts click "text=Sign In"
npx tsx .claude/tools/playwright-cli/src/cli.ts fill "#email" "test@example.com"

# If fill doesn't work (contenteditable, rich text editors, custom inputs),
# use exec with click + keyboard.type:
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "
  await page.locator('[contenteditable]').click();
  await page.keyboard.type('Hello world');
"

# Execute Playwright code
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "return await page.title();"

# Observe
npx tsx .claude/tools/playwright-cli/src/cli.ts console
npx tsx .claude/tools/playwright-cli/src/cli.ts network --method POST

# Close (when completely done with testing)
npx tsx .claude/tools/playwright-cli/src/cli.ts close
```

## When You Get Blocked

Two things block agents more than anything else: missing env vars and auth.

- **Something not working?** Start the app first — don't preemptively hunt for secrets. Many projects just work. If the app fails to start OR starts but throws runtime errors, tail the log file. The error tells you what's wrong. If it's about missing env vars or secrets, see [Getting secrets to run locally](./references/local-env-secrets.md).
- **Can't get past login?** See [E2E Testing → Auth](./references/e2e-test.md#auth). The `--cookies` flag handles most cases. If the session is expired, see [Test Users](./references/e2e-auth-test-user.md).

## The Flow

Every change follows this sequence:

1. **Unit tests** — get the logic right at the function level. [Details](./references/unit-test.md)
2. **Start the app** — figure out the project's actual start command (check `package.json` scripts, `Makefile`, `docker-compose.yml`, `README`) and run it. Common examples: `npm run dev`, `python manage.py runserver`, `go run ./cmd/server`, `docker-compose up -d`. Don't guess — discover first. **Always pipe output to a temp file** so you can check logs if something goes wrong:
   ```bash
   npm run dev > /tmp/app-startup.log 2>&1 &
   sleep 3 && tail -50 /tmp/app-startup.log
   ```
   If startup fails, `tail /tmp/app-startup.log` — the error tells you what's missing. (Need secrets? See [local env setup](./references/local-env-secrets.md).)
3. **Use the product** — interact with the running app like a real user would. Pick the right playbook:

| Changed | Do this | Playbook |
|---|---|---|
| Backend API | `curl` the endpoints | [API testing](./references/e2e-server-api-test.md) |
| Frontend UI or Full stack | `playwright-cli open --cookies` + interact | [UI & full-stack testing](./references/e2e-ui-and-fullstack.md) |
| CLI tool | Run it, check output | [CLI testing](./references/e2e-cli-tool-test.md) |
| Library | Write a consumer script | [Library testing](./references/e2e-library-change-test.md) |

Any time you're looking at UI — whether frontend-only or full-stack — you also run the [UI Quality Check](./references/e2e-ui-quality-check.md). Spacing, alignment, contrast, touch targets. The obvious stuff that separates "it renders" from "it looks right."

4. **Capture evidence** — screenshots, response bodies, terminal output. Proof.

That's it. Simple process. The hard part is actually doing it every time instead of taking shortcuts.
