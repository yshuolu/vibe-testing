# UI & Full-Stack Testing

Whether you changed just the frontend or both frontend and backend — the verification flow is the same: open a browser, look at it, click through it, screenshot everything. The only difference is that full-stack changes add an extra step: cross-checking the API independently.

This is the playbook agents most often skip or half-ass. "The component renders, the test passes." Great. Does it *look right*? Did the data actually persist? Did you check the spacing? The contrast? No? Then you didn't verify anything.

## Setup

Playwright MCP (install once at user level):

```bash
claude mcp add --scope user playwright-mcp -- npx @anthropic-ai/mcp-playwright@latest
```

Discover the actual start command — don't blindly run `npm run dev`. Every project is different.

```bash
# Check package.json, Makefile, docker-compose, README
cat package.json | grep -E '"dev"|"start"|"serve"' 2>/dev/null
grep -E '^[a-z].*:' Makefile 2>/dev/null | grep -i 'run\|dev\|serve'
ls docker-compose*.yml 2>/dev/null
head -100 README.md 2>/dev/null | grep -i -A2 'run\|start\|dev\|setup'
```

Some projects are a single process (`npm run dev`), some have separate frontend/backend, some use Docker (`docker-compose up -d`). Read the project's own docs. Always log to a temp file so you can diagnose failures:

```bash
# e.g. single process
npm run dev > /tmp/app-startup.log 2>&1 &

# e.g. separate services
cd backend && npm run dev > /tmp/backend.log 2>&1 &
cd frontend && npm run dev > /tmp/frontend.log 2>&1 &
```

Wait, then tail the logs to confirm it's up:

```bash
sleep 3 && tail -50 /tmp/app-startup.log

# full-stack: check both
tail -20 /tmp/backend.log
tail -20 /tmp/frontend.log
curl -s localhost:8000/api/health > /dev/null && echo "backend up"
curl -s localhost:3000/ > /dev/null && echo "frontend up"
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

```
browser_navigate: http://localhost:3000/path/to/page
browser_screenshot
```

**Actually examine the screenshot.** Don't just confirm "it renders." Check:
- Is the thing you changed visible?
- Is anything obviously broken?
- Does the spacing look even? Alignment clean? Text readable?

Now run the **[UI Quality Check](./e2e-ui-quality-check.md)** — spacing, alignment, contrast, touch targets. Every time. This catches the embarrassing stuff.

---

## Step 3: Interact Like a User

Do exactly what a user would do to exercise your change:

```
browser_click: "Submit"                              # click a button
browser_fill: [name="email"], test@example.com       # fill a field
browser_select_option: [name="country"], US           # pick a dropdown
browser_hover: .tooltip-trigger                       # hover for tooltip
browser_press_key: Enter                              # keyboard
```

**Screenshot after every meaningful interaction.** Non-negotiable.

```
browser_click: "Add to Cart"
browser_screenshot    # cart count updated? toast appeared?

browser_fill: [name="search"], "test query"
browser_press_key: Enter
browser_screenshot    # results appeared?
```

When screenshots aren't enough, verify with code:

```
browser_get_text: .success-message
browser_evaluate: document.querySelector('#btn').disabled
browser_evaluate: document.querySelector('.list').children.length
browser_wait_for_selector: .modal-overlay
```

---

## Step 4: Full CRUD Through the Browser (Full-Stack)

If your change involves data, test the entire lifecycle through the UI. At each step, cross-check with the API to verify both layers agree.

**Create:**
```
browser_navigate: http://localhost:3000/items/new
browser_fill: [name="title"], "Vibe Test Item"
browser_fill: [name="description"], "Created during testing"
browser_fill: [name="price"], "29.99"
browser_click: "Create"
browser_screenshot
```

Cross-check:
```bash
curl -s http://localhost:8000/api/items?sort=created_at:desc | jq '.[0]'
```

**Read:**
```
browser_navigate: http://localhost:3000/items
browser_screenshot   # item in the list?
browser_click: "Vibe Test Item"
browser_screenshot   # detail page correct?
```

**Update:**
```
browser_click: "Edit"
browser_fill: [name="title"], "Updated Test Item"
browser_click: "Save"
browser_screenshot
```

Cross-check:
```bash
curl -s http://localhost:8000/api/items/$ID | jq '.title'
```

**Delete:**
```
browser_click: "Delete"
browser_click: "Confirm"
browser_screenshot
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
```
browser_navigate: http://localhost:3000/items?filter=nonexistent
browser_screenshot
```

**Error state:**
```
browser_fill: [name="email"], not-an-email
browser_click: "Submit"
browser_screenshot
```

**Loading state:**
```
browser_click: "Load Data"
browser_screenshot    # spinner/skeleton visible?
browser_wait_for_selector: .data-table
browser_screenshot    # data loaded?
```

**Long content:**
```
browser_fill: [name="title"], "This is an extremely long title that might overflow the container and break the layout entirely"
browser_screenshot
```

---

## Step 6: Validation on Both Layers (Full-Stack)

The sneakiest full-stack bugs: validation passes on one layer but fails on the other.

**Frontend validation:**
```
browser_navigate: http://localhost:3000/items/new
browser_click: "Create"    # submit empty form
browser_screenshot          # client-side errors shown?
```

**Backend validation (bypass the UI):**
```bash
curl -s -X POST http://localhost:8000/api/items \
  -H "Content-Type: application/json" -d '{}' | jq .
```

**Backend errors in the UI:**
```
browser_fill: [name="email"], "duplicate@example.com"
browser_click: "Create"
browser_screenshot    # does UI show "email already exists"?
```

---

## Step 7: Data Consistency Check (Full-Stack)

The most insidious bug: UI shows different data than the API returns.

```bash
API_DATA=$(curl -s http://localhost:8000/api/items/1)
echo "$API_DATA" | jq .
```

```
browser_navigate: http://localhost:3000/items/1
browser_get_text: .item-title
browser_get_text: .item-price
# Do these match the API response? If not, you found a bug.
```

---

## Step 8: Auth Flows (If Applicable)

```
# Unauthenticated → redirects to login?
browser_navigate: http://localhost:3000/dashboard
browser_screenshot

# Login
browser_fill: [name="email"], testuser@example.com
browser_fill: [name="password"], testpassword
browser_click: "Log In"
browser_screenshot    # dashboard loads with user info?
```

---

## Step 9: Responsive Check (If Layout Changed)

```
browser_evaluate: window.resizeTo(375, 812)
browser_screenshot    # mobile

browser_evaluate: window.resizeTo(768, 1024)
browser_screenshot    # tablet

browser_evaluate: window.resizeTo(1440, 900)
browser_screenshot    # back to desktop
```

---

## Step 10: Quick Accessibility Check

```
browser_press_key: Tab
browser_screenshot    # focus ring visible?
browser_press_key: Tab
browser_screenshot    # logical focus order?
browser_press_key: Enter    # activates the focused element?
```

---

After everything: look at your screenshots. Does the UI match what the change was supposed to achieve? Any regressions in surrounding elements? Run the [UI Quality Check](./e2e-ui-quality-check.md) one more time on the final state.
