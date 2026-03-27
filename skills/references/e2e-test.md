# E2E Testing

> If you didn't see it work in the running product, you didn't verify it.

This is the whole point of vibe testing. You interact with the running application the way a real user would. Unit tests, type checks, code review — all good, all insufficient. The only proof is direct observation.

## Prerequisites

**playwright-cli** must be installed (see [Setup](../skill.md#setup)).

## Auth

If the app requires login, use `--cookies` to steal the developer's browser session:

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts open http://localhost:3000 --cookies
```

This automatically:
1. Finds the developer's default Chrome/Brave/Edge/Arc profile
2. Copies it to a temp directory
3. Extracts all cookies + localStorage for the target domain via CDP
4. Injects them into a fresh Playwright browser
5. Navigates to the URL

Take a snapshot to verify you're logged in:

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts snapshot
```

If you see a login page instead of authenticated content, the session is expired or the wrong profile was used. Try `--profile`:

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts profiles
npx tsx .claude/tools/playwright-cli/src/cli.ts close
npx tsx .claude/tools/playwright-cli/src/cli.ts open http://localhost:3000 --cookies --profile "Work"
```

If no profile has a valid session, create a real test user. See [Test Users](./e2e-auth-test-user.md) for the methodology and framework-specific examples.

## Start the App

Figure out the project's actual start command — don't guess. Check these sources:

```bash
# Check package.json scripts
cat package.json | grep -E '"dev"|"start"|"serve"' 2>/dev/null

# Check Makefile
grep -E '^[a-z].*:' Makefile 2>/dev/null | grep -i 'run\|dev\|serve\|start'

# Check docker-compose
ls docker-compose*.yml 2>/dev/null

# Check README for setup instructions
head -100 README.md 2>/dev/null | grep -i -A2 'run\|start\|dev\|setup'
```

Common examples: `npm run dev`, `python manage.py runserver`, `go run ./cmd/server`, `docker-compose up -d`. Run whatever the project actually uses, then verify it's up:

```bash
curl -s http://localhost:3000/ > /dev/null && echo "ready"
```

## Pick Your Playbook

- Backend API only → [Server API Testing](./e2e-server-api-test.md)
- Frontend UI or Full stack → [UI & Full-Stack Testing](./e2e-ui-and-fullstack.md)
- CLI tool → [CLI Testing](./e2e-cli-tool-test.md)
- Library/package → [Library Testing](./e2e-library-change-test.md)

Any time you're looking at UI, also run the [UI Quality Check](./e2e-ui-quality-check.md).

## The Pattern (All Scenarios)

1. **What changed?** List every user-visible behavior your change affects.
2. **Use the product.** Send requests. Click buttons. Run commands. Whatever the user would do.
3. **Capture proof.** Screenshots, response bodies, terminal output.
4. **Break it.** Don't just test the happy path. Try bad inputs, error states, edge cases.
5. **Clean up.** Remove test data, stop servers.
