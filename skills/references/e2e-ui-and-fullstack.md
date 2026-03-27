# UI & Full-Stack Testing

Whether you changed just the frontend or both frontend and backend — the verification flow is the same: open a browser, look at it, click through it, screenshot everything. The only difference is that full-stack changes add an extra step: cross-checking the API independently.

This is the playbook agents most often skip or half-ass. "The component renders, the test passes." Great. Does it *look right*? Did the data actually persist? Did you check the spacing? The contrast? No? Then you didn't verify anything.

## Setup

playwright-cli must be installed (see [SKILL.md Setup](../SKILL.md#setup)).

Start the app. Discover the actual start command — don't blindly run `npm run dev`. Every project is different.

```bash
cat package.json | grep -E '"dev"|"start"|"serve"' 2>/dev/null
grep -E '^[a-z].*:' Makefile 2>/dev/null | grep -i 'run\|dev\|serve'
ls docker-compose*.yml 2>/dev/null
head -100 README.md 2>/dev/null | grep -i -A2 'run\|start\|dev\|setup'
```

Always log to a temp file so you can diagnose failures:

```bash
npm run dev > /tmp/app-startup.log 2>&1 &
sleep 3 && tail -50 /tmp/app-startup.log
```

If it fails, the logs tell you why. See [local env setup](./local-env-secrets.md) if it's about missing secrets.

---

## Step 1: API Sanity Check (Full-Stack Only)

If you touched the backend, verify the API works in isolation *before* opening the browser. This way, when something is broken in the UI, you already know which layer to blame.

```bash
curl -s http://localhost:8000/api/items | jq .
curl -s http://localhost:8000/api/items/1 | jq 'keys'
```

See [Server API Testing](./e2e-server-api-test.md) for the full curl playbook.

If your change is frontend-only, skip this step.

---

## Step 2: Navigate and Look

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts open http://localhost:3000/path/to/page --cookies
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/page.png
```

**Actually examine the screenshot.** Don't just confirm "it renders." Check:
- Is the thing you changed visible?
- Is anything obviously broken?
- Does the spacing look even? Alignment clean? Text readable?

Now run the **[UI Quality Check](./e2e-ui-quality-check.md)** — spacing, alignment, contrast, touch targets. Every time. This catches the embarrassing stuff.

---

## Step 3: Interact Like a User

Do exactly what a user would do to exercise your change:

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Submit"
npx tsx .claude/tools/playwright-cli/src/cli.ts fill "[name='email']" "test@example.com"
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.selectOption('[name=\"country\"]', 'US');"
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.hover('.tooltip-trigger');"
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.keyboard.press('Enter');"
```

**Screenshot after every meaningful interaction.** Non-negotiable.

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Add to Cart"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/after-add.png

npx tsx .claude/tools/playwright-cli/src/cli.ts fill "[name='search']" "test query"
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.keyboard.press('Enter');"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/search-results.png
```

When screenshots aren't enough, verify with code:

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "return await page.textContent('.success-message');"
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "return await page.locator('#btn').isDisabled();"
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "return await page.locator('.list').locator('> *').count();"
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.waitForSelector('.modal-overlay');"
```

---

## Step 4: Full CRUD Through the Browser (Full-Stack)

If your change involves data, test the entire lifecycle through the UI. At each step, cross-check with the API to verify both layers agree.

**Create:**
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts navigate http://localhost:3000/items/new
npx tsx .claude/tools/playwright-cli/src/cli.ts fill "[name='title']" "Vibe Test Item"
npx tsx .claude/tools/playwright-cli/src/cli.ts fill "[name='description']" "Created during testing"
npx tsx .claude/tools/playwright-cli/src/cli.ts fill "[name='price']" "29.99"
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Create"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/created.png
```

Cross-check:
```bash
curl -s http://localhost:8000/api/items?sort=created_at:desc | jq '.[0]'
```

**Read:**
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts navigate http://localhost:3000/items
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/list.png
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Vibe Test Item"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/detail.png
```

**Update:**
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Edit"
npx tsx .claude/tools/playwright-cli/src/cli.ts fill "[name='title']" "Updated Test Item"
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Save"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/updated.png
```

Cross-check:
```bash
curl -s http://localhost:8000/api/items/$ID | jq '.title'
```

**Delete:**
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Delete"
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Confirm"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/deleted.png
```

Cross-check:
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/api/items/$ID
# should be 404
```

Both sides agree at every step. That's the test.

---

## Step 5: Test the States People Forget

This is where the real bugs live:

**Empty state:**
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts navigate "http://localhost:3000/items?filter=nonexistent"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/empty.png
```

**Error state:**
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts fill "[name='email']" "not-an-email"
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Submit"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/error.png
```

**Loading state:**
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Load Data"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/loading.png
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.waitForSelector('.data-table');"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/loaded.png
```

**Long content:**
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts fill "[name='title']" "This is an extremely long title that might overflow the container and break the layout entirely"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/overflow.png
```

---

## Step 6: Validation on Both Layers (Full-Stack)

The sneakiest full-stack bugs: validation passes on one layer but fails on the other.

**Frontend validation:**
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts navigate http://localhost:3000/items/new
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Create"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/validation.png
```

**Backend validation (bypass the UI):**
```bash
curl -s -X POST http://localhost:8000/api/items \
  -H "Content-Type: application/json" -d '{}' | jq .
```

**Backend errors in the UI:**
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts fill "[name='email']" "duplicate@example.com"
npx tsx .claude/tools/playwright-cli/src/cli.ts click "Create"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/backend-error.png
```

---

## Step 7: Data Consistency Check (Full-Stack)

The most insidious bug: UI shows different data than the API returns.

```bash
API_DATA=$(curl -s http://localhost:8000/api/items/1)
echo "$API_DATA" | jq .
```

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts navigate http://localhost:3000/items/1
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "return await page.textContent('.item-title');"
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "return await page.textContent('.item-price');"
# Do these match the API response? If not, you found a bug.
```

---

## Step 8: Responsive Check (If Layout Changed)

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.setViewportSize({ width: 375, height: 812 });"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/mobile.png

npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.setViewportSize({ width: 768, height: 1024 });"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/tablet.png

npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.setViewportSize({ width: 1440, height: 900 });"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/desktop.png
```

---

## Step 9: Quick Accessibility Check

```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.keyboard.press('Tab');"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/focus1.png

npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.keyboard.press('Tab');"
npx tsx .claude/tools/playwright-cli/src/cli.ts screenshot --output /tmp/focus2.png

npx tsx .claude/tools/playwright-cli/src/cli.ts exec "await page.keyboard.press('Enter');"
```

Use the ARIA snapshot to verify semantic structure:
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts snapshot
```

---

After everything: look at your screenshots. Does the UI match what the change was supposed to achieve? Any regressions in surrounding elements? Run the [UI Quality Check](./e2e-ui-quality-check.md) one more time on the final state.

When done:
```bash
npx tsx .claude/tools/playwright-cli/src/cli.ts close
```
