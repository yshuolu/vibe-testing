# Vibe Testing

Here's the thing about testing that most agents get completely wrong: they write some unit tests, see green checkmarks, and call it a day. That's not testing. That's wishful thinking.

Vibe testing means you actually *use* the thing you just built. You run it. You click it. You curl it. You see with your own eyes that it works. Just like a human would before shipping.

## Setup

Install Playwright MCP at user level (do this once, works everywhere):

```bash
claude mcp add --scope user playwright-mcp -- npx @anthropic-ai/mcp-playwright@latest
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

## When You Get Blocked

Two things block agents more than anything else: missing env vars and auth.

- **Something not working?** Start the app first — don't preemptively hunt for secrets. Many projects just work. If the app fails to start OR starts but throws runtime errors, tail the log file. The error tells you what's wrong. If it's about missing env vars or secrets, see [Getting secrets to run locally](./references/local-env-secrets.md).
- **Can't get past login?** [Auth & test users](./references/auth-test-user.md) — find or create test users, handle JWT vs cookie vs OAuth.

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
| Frontend UI or Full stack | Browser via Playwright MCP (+ API cross-check for full stack) | [UI & full-stack testing](./references/e2e-ui-and-fullstack.md) |
| CLI tool | Run it, check output | [CLI testing](./references/e2e-cli-tool-test.md) |
| Library | Write a consumer script | [Library testing](./references/e2e-library-change-test.md) |

Any time you're looking at UI — whether frontend-only or full-stack — you also run the [UI Quality Check](./references/e2e-ui-quality-check.md). Spacing, alignment, contrast, touch targets. The obvious stuff that separates "it renders" from "it looks right."

4. **Capture evidence** — screenshots, response bodies, terminal output. Proof.

That's it. Simple process. The hard part is actually doing it every time instead of taking shortcuts.
