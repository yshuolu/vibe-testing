# Auth & Test Users

You can't vibe test anything behind a login if you can't log in. This is the #1 blocker agents hit and then just... give up. "I couldn't test the authenticated flow." Unacceptable. Figure out the auth, get a test user, and log in.

## Step 1: Find the Auth Scheme

Before anything else, figure out what kind of auth the app uses. Look at the code:

```bash
# Check for JWT / token patterns
grep -r "jwt\|jsonwebtoken\|jose\|JWT_SECRET\|ACCESS_TOKEN\|Bearer" --include="*.{ts,js,py,go,rb}" -l .

# Check for session / cookie patterns
grep -r "express-session\|cookie-session\|session_middleware\|SESSION_SECRET\|set-cookie\|csrf" --include="*.{ts,js,py,go,rb}" -l .

# Check for OAuth / third-party auth
grep -r "oauth\|passport\|next-auth\|auth0\|supabase.*auth\|firebase.*auth\|clerk" --include="*.{ts,js,py,go,rb}" -l .

# Check for auth middleware
grep -r "requireAuth\|isAuthenticated\|protect\|authMiddleware\|@login_required\|Authorize" --include="*.{ts,js,py,go,rb}" -l .
```

This tells you which path to follow below.

## Step 2: Find or Create a Test User

### Check if test users already exist

Most projects have seed data, fixtures, or a dev setup that creates test users:

```bash
# Seed files
find . -path "*/seed*" -o -path "*/fixture*" -o -path "*/migration*" | grep -v node_modules | head -20

# Look for test credentials in code/config
grep -r "testuser\|test@\|admin@\|password.*test\|seed.*user\|demo.*user" --include="*.{ts,js,py,json,yaml,yml,sql,go}" -l . | grep -v node_modules

# Check for a seed/setup command
grep -r "seed\|setup\|migrate" package.json Makefile docker-compose.yml 2>/dev/null
```

If you find seed data with test credentials, run the seed:

```bash
npm run seed
# or
python manage.py loaddata fixtures/users.json
# or
npx prisma db seed
```

### Create a test user if none exists

If there's no seed data, create one through the app's own registration flow. This is better than inserting directly into the DB because it ensures password hashing and all the signup side-effects run properly.

**Through the API:**
```bash
curl -s -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "vibe-test@test.local", "password": "TestPass123!", "name": "Vibe Tester"}' | jq .
```

**Through the browser:**
```
browser_navigate: http://localhost:3000/register
browser_fill: [name="email"], vibe-test@test.local
browser_fill: [name="password"], TestPass123!
browser_fill: [name="name"], Vibe Tester
browser_click: "Sign Up"
browser_screenshot
```

**Through a management command (Django, Rails, etc.):**
```bash
# Django
python manage.py createsuperuser --email vibe-test@test.local --username vibetest --noinput
python manage.py shell -c "from django.contrib.auth.models import User; u = User.objects.get(username='vibetest'); u.set_password('TestPass123!'); u.save()"

# Rails
rails runner "User.create!(email: 'vibe-test@test.local', password: 'TestPass123!', name: 'Vibe Tester')"
```

**Direct DB insert as last resort** (only if no registration flow exists):
```bash
# You need to know the password hashing scheme. Don't store plaintext.
# Example for bcrypt:
node -e "const bcrypt = require('bcrypt'); bcrypt.hash('TestPass123!', 10).then(h => console.log(h))"
# Then insert with the hashed password
```

### Email verification workaround

If the app requires email verification and you're running locally:

```bash
# Check if there's a way to skip verification in dev
grep -r "VERIFY\|EMAIL_CONFIRM\|SKIP.*VERIFY\|email_verified" --include="*.{ts,js,py,env*,yaml,yml}" -l .

# Often there's an env var
# NODE_ENV=development, SKIP_EMAIL_VERIFICATION=true, etc.

# Or just update the DB directly
sqlite3 dev.db "UPDATE users SET email_verified = 1 WHERE email = 'vibe-test@test.local';"
# or
psql -d mydb -c "UPDATE users SET email_verified = true WHERE email = 'vibe-test@test.local';"
```

---

## Authenticating: JWT / Token-Based Auth

This is the most common pattern for SPAs and API-first apps. You POST credentials, get back a token, and send it with every request.

### Get the token

```bash
LOGIN_RESPONSE=$(curl -s -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "vibe-test@test.local", "password": "TestPass123!"}')

echo "$LOGIN_RESPONSE" | jq .

# Extract the token — field name varies: token, accessToken, access_token, jwt
TOKEN=$(echo "$LOGIN_RESPONSE" | jq -r '.token // .accessToken // .access_token // .jwt')
echo "Token: $TOKEN"
```

### Use it in API requests

```bash
# Every authenticated request needs the Authorization header
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:3000/api/me | jq .
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:3000/api/protected-resource | jq .
```

### Use it in the browser

If the frontend stores the JWT in localStorage (most SPAs do):

```
browser_navigate: http://localhost:3000
browser_evaluate: localStorage.setItem('token', 'YOUR_TOKEN_HERE')
browser_evaluate: localStorage.setItem('accessToken', 'YOUR_TOKEN_HERE')
browser_navigate: http://localhost:3000/dashboard
browser_screenshot
```

Find out where the app stores it:

```
browser_evaluate: (() => {
  const keys = Object.keys(localStorage);
  const tokenKeys = keys.filter(k => /token|auth|jwt|session/i.test(k));
  return tokenKeys.map(k => ({ key: k, value: localStorage.getItem(k)?.slice(0, 50) + '...' }));
})()
```

### Handle token refresh

If the token expires quickly, check if there's a refresh token:

```bash
REFRESH=$(echo "$LOGIN_RESPONSE" | jq -r '.refreshToken // .refresh_token')

# Refresh when needed
curl -s -X POST http://localhost:3000/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\": \"$REFRESH\"}" | jq .
```

---

## Authenticating: Cookie / Session-Based Auth

This is the traditional pattern — POST credentials, server sets a cookie, browser sends it automatically. Common in server-rendered apps (Rails, Django, Express with sessions).

### Get the session cookie via curl

```bash
# -c saves cookies to a file, -b sends them
curl -s -c cookies.txt -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "vibe-test@test.local", "password": "TestPass123!"}' | jq .

# Now use -b to send cookies with every request
curl -s -b cookies.txt http://localhost:3000/api/me | jq .
curl -s -b cookies.txt http://localhost:3000/api/protected-resource | jq .
```

For form-based login (not JSON API):

```bash
# Handle CSRF token if needed
CSRF=$(curl -s -c cookies.txt http://localhost:3000/login | grep -o 'name="csrf[^"]*" value="[^"]*"' | grep -o 'value="[^"]*"' | cut -d'"' -f2)

curl -s -c cookies.txt -b cookies.txt -X POST http://localhost:3000/login \
  -d "email=vibe-test@test.local&password=TestPass123!&_csrf=$CSRF"
```

### Log in via the browser (preferred for cookie auth)

Cookie auth works naturally in the browser — just log in through the UI:

```
browser_navigate: http://localhost:3000/login
browser_fill: [name="email"], vibe-test@test.local
browser_fill: [name="password"], TestPass123!
browser_click: "Log In"
browser_screenshot
```

The browser handles cookies automatically after this. Every subsequent `browser_navigate` will be authenticated.

### Verify you're logged in

```
browser_evaluate: document.cookie
```

```bash
# curl
curl -s -b cookies.txt http://localhost:3000/api/me | jq .
```

### Clean up cookies

```bash
rm -f cookies.txt
```

---

## Authenticating: OAuth / Third-Party (Auth0, Supabase, Firebase, etc.)

This is the tricky one. You can't easily automate "Login with Google." But there are workarounds.

### Option 1: Find the bypass for local dev

Most OAuth setups have a local development mode or test credentials:

```bash
# Check for dev/test auth config
grep -r "AUTH0\|SUPABASE\|FIREBASE\|NEXTAUTH\|CLERK" .env* --include="*.env*" 2>/dev/null
grep -r "credentials.*provider\|CredentialsProvider\|email.*provider" --include="*.{ts,js}" -l .
```

Many apps (especially Next.js with next-auth) have a **credentials provider** enabled in development that lets you log in with email/password even if production uses OAuth only.

### Option 2: Create a session directly

If you have access to the auth provider's admin:

```bash
# Supabase — create user via CLI
npx supabase auth admin create-user --email vibe-test@test.local --password TestPass123!

# Firebase — use admin SDK
node -e "
const admin = require('firebase-admin');
admin.initializeApp();
admin.auth().createUser({ email: 'vibe-test@test.local', password: 'TestPass123!', emailVerified: true })
  .then(u => console.log('Created:', u.uid));
"
```

### Option 3: Inject the session/token directly

If you can get a valid token from the provider's API:

```bash
# Supabase example
RESPONSE=$(curl -s -X POST 'http://localhost:54321/auth/v1/token?grant_type=password' \
  -H "apikey: $SUPABASE_ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email": "vibe-test@test.local", "password": "TestPass123!"}')
TOKEN=$(echo "$RESPONSE" | jq -r '.access_token')
```

Then inject into browser:

```
browser_evaluate: localStorage.setItem('supabase.auth.token', JSON.stringify({currentSession: {access_token: 'YOUR_TOKEN'}}))
```

### Option 4: Ask the user

If none of the above works — the auth provider requires real OAuth, there's no dev bypass, and you can't create test users programmatically — ask the user. Don't waste 20 minutes trying to hack around it. Just say: "I need a test user session to verify the authenticated flows. Can you log in at localhost:3000 and give me the auth cookie/token from devtools?"

---

## Test User Maintenance

**Don't leave trash in the database.** If you created a test user, clean it up when you're done, or use a user that's clearly marked as test data:

- Use a recognizable email: `vibe-test@test.local`, `agent-test@example.com`
- Use a recognizable name: `Vibe Tester`, `Agent Test User`
- Delete after testing if the user was created just for this session:

```bash
curl -s -X DELETE -H "Authorization: Bearer $ADMIN_TOKEN" http://localhost:3000/api/admin/users/$USER_ID

# or direct DB
sqlite3 dev.db "DELETE FROM users WHERE email = 'vibe-test@test.local';"
```

If the project already has persistent test users in seed data, prefer those over creating new ones.
