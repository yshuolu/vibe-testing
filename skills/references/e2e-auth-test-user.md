# Test Users

## Test User Methodology

A test user must be **real**. Real row in the database. Real password hash. Real foreign keys. Real side effects.

The point of vibe testing is to verify the product works the way a real user would experience it. If you shortcut the auth layer — skip login, bypass middleware, inject a fake session with no backing user — you're testing a fiction. The request goes through, but the user lookup fails. The profile page crashes. The permissions check returns null. The billing query finds nothing.

A proper test user goes through the same code paths as every other user:
- Created via the app's own registration flow (or seed data that mimics it)
- Password hashed by the app's own auth system
- Stored in the real database with all required fields populated
- Related records created (team, org, default settings, whatever the app expects)
- **Marked with an `is_test_user` flag** (boolean, default false) on the user record

The `is_test_user` flag is how the system knows this user is for testing. It makes the test user:
- **Discoverable** — `SELECT * FROM users WHERE is_test_user = true`
- **Cleanable** — `DELETE FROM users WHERE is_test_user = true`
- **Excludable** — filter out of analytics, billing, email sends, usage reports

The flag does not change how the user works. The test user has the same access, same permissions, same code paths as every other user. The flag is metadata — it says "this is test data" without altering behavior. The only exception is **billing**: test users should not trigger real charges. If the app has billing, the `is_test_user` flag should skip payment processing, use a test Stripe customer, or default to a free plan.

**Do not:**
- Insert a row with a plaintext password
- Mock the auth middleware to return a hardcoded user object
- Mint a JWT or session token for a user ID that doesn't exist in the database
- Create a user without the `is_test_user` flag (you won't be able to find or clean it up)

**Do:**
- Use the app's registration API or CLI command to create the user
- Set `is_test_user = true` on the record (via seed script, migration, or post-creation update)
- Run the same seed/migration the rest of the team uses
- Verify the user can actually log in and load a protected page
- Confirm the user has the data the app expects (profile, org, permissions)

---

## The Process

Before creating anything, study what the project already has. Prefer reusing an existing test user over creating a new one.

### Step 1: Check for an existing test user setup

Research the project's codebase for any existing test user setup — seed scripts, fixtures, dev auth configuration, or documented test credentials.

### Step 2: Evaluate what you find

**If the project has an existing test user setup:**

Check if it conforms to our philosophy — the test user must be a real database record, not a mock or a session with no backing data.

- Does the setup create a real user record in the database?
- Does the user have the `is_test_user` flag (or equivalent)?
- Can you authenticate as that user and hit protected endpoints? (This could be through UI login, API login, a special header, a dev auth env var — the mechanism doesn't matter as long as the user is real in the DB.)
- Does the user have the related data the app expects (profile, team, permissions)?

If yes — use it and proceed.

If no — the existing setup is a shortcut (e.g., a mock middleware that returns a hardcoded user object with no database record, or a session bypass where the user ID doesn't exist in the DB). **Tell the developer**: "The current test user setup has no real database record. This means downstream code that queries user data will fail. I'd recommend adding a seed script that creates a real test user in the database."

### Step 3: If no test user setup exists

Create one as a code change. This is part of your contribution — making the project testable is as important as the feature itself.

What to add depends on the framework (see examples below), but the pattern is always:
1. A seed script or DB operation that creates a real user record with `is_test_user = true`
2. A dev-only auth mechanism to authenticate as that user without UI login (env var, special header, API key, direct session creation, etc.)
3. Any related data the user needs (org, team, default settings)

Commit this alongside your feature change. The next developer (or agent) benefits too.

---

## Framework Examples

Every example follows two steps:
1. **Create** a real user record in the database with `is_test_user = true`
2. **Authenticate** by adding an `if` block at the top of the existing auth middleware — if `TEST_USER_ID` is set and it's not production, load the test user from DB and skip the normal auth check

No new routes. No new endpoints. No username/password login. No UI clicking. Just a middleware change.

### The Middleware Change (Same Pattern, Every Framework)

Find the project's existing auth function. Add this at the top, before the normal auth logic:

```
if environment is NOT production AND TEST_USER_ID is set:
  user = db.users.findById(TEST_USER_ID)
  attach user to request context
  return (skip normal auth)

// ... normal auth below (JWT verify, session check, OAuth, etc.)
```

That's it. The downstream code receives a real user object loaded from the real database. It doesn't know or care that the auth check was skipped.

### Framework-Specific Examples

Each example shows: how to create the test user record, and where to put the middleware `if` block.

#### Next.js (Auth.js, WorkOS, or any auth)

**Create:**
```bash
# Prisma
npx prisma db execute --stdin <<< "
INSERT INTO users (email, name, is_test_user)
VALUES ('vibe-test@test.local', 'Vibe Tester', true)
ON CONFLICT (email) DO NOTHING;
"
```

**Middleware change:** In whatever function resolves the current user (`getSession()`, `getUser()`, `withAuth()`, or the middleware in `middleware.ts`):

```typescript
// At the top of getSession(), before cookie/token verification
if (process.env.NODE_ENV !== 'production' && process.env.TEST_USER_ID) {
  return { user: { id: process.env.TEST_USER_ID } };
}
```

This works the same whether the app uses Auth.js, WorkOS, Clerk, or any other auth provider. The provider-specific verification is skipped; the user is loaded from the app's own database.

For WorkOS specifically: if the user also needs to exist in WorkOS (for webhook sync), create it via the WorkOS API first, wait for the sync, then flag the local record.

#### Firebase

**Create:**
```bash
node -e "
const admin = require('firebase-admin');
admin.initializeApp();
admin.auth().createUser({
  email: 'vibe-test@test.local',
  emailVerified: true,
  displayName: 'Vibe Tester',
}).then(u => console.log('Created:', u.uid));
"
```

If the app syncs Firebase users to its own database, trigger the sync and flag the local record with `is_test_user = true`.

**Middleware change:** In the function that calls `admin.auth().verifyIdToken()`:

```typescript
if (process.env.NODE_ENV !== 'production' && process.env.TEST_USER_ID) {
  req.user = await db.users.findById(process.env.TEST_USER_ID);
  return next();
}
```

#### Supabase

**Create:**
```bash
npx supabase auth admin create-user --email vibe-test@test.local
```

If the app has its own `profiles` table, flag it:
```sql
UPDATE profiles SET is_test_user = true WHERE email = 'vibe-test@test.local';
```

**Middleware change:** Same pattern — in the function that verifies the Supabase JWT, add the `if` block at the top.

#### Django

**Create:**
```bash
python manage.py shell -c "
from django.contrib.auth.models import User
u, created = User.objects.get_or_create(
    username='vibetest',
    defaults={'email': 'vibe-test@test.local', 'first_name': 'Vibe', 'last_name': 'Tester'}
)
u.profile.is_test_user = True
u.profile.save()
"
```

**Middleware change:** In the auth middleware or authentication backend:

```python
# At the top, before normal auth
if settings.DEBUG and os.environ.get('TEST_USER_ID'):
    request.user = User.objects.get(id=os.environ['TEST_USER_ID'])
    return
```

#### Rails + Devise

**Create:**
```bash
rails runner "
User.find_or_create_by!(email: 'vibe-test@test.local') do |u|
  u.name = 'Vibe Tester'
  u.password = SecureRandom.hex(16)
  u.is_test_user = true
end
"
```

**Middleware change:** In a before_action or Warden hook:

```ruby
# In ApplicationController or a concern
if !Rails.env.production? && ENV['TEST_USER_ID']
  sign_in(User.find(ENV['TEST_USER_ID']))
end
```

#### Express / Node.js

**Create:**
```javascript
// seed-test-user.js
await db.users.upsert({
  where: { email: 'vibe-test@test.local' },
  update: {},
  create: { email: 'vibe-test@test.local', name: 'Vibe Tester', is_test_user: true },
});
```

**Middleware change:** In the existing auth middleware:

```javascript
if (process.env.NODE_ENV !== 'production' && process.env.TEST_USER_ID) {
  req.user = await db.users.findById(process.env.TEST_USER_ID);
  return next();
}
```

#### Any Other Framework

Find the auth middleware. Add the `if` block at the top. Load the test user from the database. Skip the normal auth check. The pattern is identical regardless of language or framework — it's always 3-5 lines at the top of the existing auth function.

Set `TEST_USER_ID` in `.env` (gitignored). Never set it in production.

---

## Test User Conventions

- **Flag:** `is_test_user = true` — always set, on every test user
- **Discoverable:** `SELECT * FROM users WHERE is_test_user = true`
- **Auth:** via dev-only mechanism, never via username/password login
- **Reuse:** if a flagged test user already exists, authenticate as it — don't create a duplicate
- **Cleanup:** `DELETE FROM users WHERE is_test_user = true`
- **Document:** after researching or updating the test user setup, document it in the project's `CLAUDE.md` — how to authenticate as the test user, what env vars are needed, what seed to run. Do not include credentials or secrets in `CLAUDE.md`.
