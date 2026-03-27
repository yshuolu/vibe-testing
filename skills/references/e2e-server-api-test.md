# Server API Testing

You changed something on the backend. The only way to know it works is to hit the running server with real HTTP requests and look at what comes back. Not "the code looks right." Not "the test passes." Actually `curl` it and read the response.

## Start the Server

Discover the actual start command — don't guess:

```bash
# Check package.json, Makefile, docker-compose, README
cat package.json | grep -E '"dev"|"start"|"serve"' 2>/dev/null
grep -E '^[a-z].*:' Makefile 2>/dev/null | grep -i 'run\|dev\|serve'
ls docker-compose*.yml 2>/dev/null
```

Common examples: `npm run dev`, `python manage.py runserver`, `go run ./cmd/server`, `docker-compose up -d`. Always log to a temp file so you can diagnose failures:

```bash
# e.g. npm run dev
npm run dev > /tmp/app-startup.log 2>&1 &
sleep 3 && tail -50 /tmp/app-startup.log

# wait for it to be ready
for i in $(seq 1 30); do curl -s localhost:3000/health > /dev/null 2>&1 && break; sleep 1; done
```

If it fails, `tail /tmp/app-startup.log` — the error tells you what's missing. See [local env setup](./local-env-secrets.md) if it's about secrets.

## Trace Your Blast Radius

Before you start curling, figure out what you actually need to test. Your change might affect more than the one endpoint you touched:
- Did you modify shared middleware? Every route using it is affected.
- Did you change a DB query helper? Every endpoint calling it is affected.
- Did you alter the schema? Every query reading those columns is affected.

## Hit Every Affected Endpoint

Use `curl`. Pipe through `jq`. Read the output.

```bash
# GET — read data
curl -s http://localhost:3000/api/users | jq .
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/users  # just the status code

# POST — create data
curl -s -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Test User", "email": "test@example.com"}' | jq .

# PUT/PATCH — update data
curl -s -X PUT http://localhost:3000/api/users/123 \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated Name"}' | jq .

# DELETE — remove data
curl -s -o /dev/null -w "%{http_code}" -X DELETE http://localhost:3000/api/users/123
```

**Don't just check for 200.** Read the actual response body. Check the fields. Check the values. Check that sensitive fields (like `password`) are NOT in the response.

```bash
RESPONSE=$(curl -s http://localhost:3000/api/users/1)
echo "$RESPONSE" | jq '.name'              # right name?
echo "$RESPONSE" | jq 'has("password")'    # should be false
```

## Test the Full Lifecycle

The most important test: create something, read it back, update it, read again, delete it, confirm it's gone.

```bash
# Create
ID=$(curl -s -X POST http://localhost:3000/api/items \
  -H "Content-Type: application/json" \
  -d '{"title": "Vibe Test"}' | jq -r '.id')

# Read it back
curl -s http://localhost:3000/api/items/$ID | jq .

# Update
curl -s -X PUT http://localhost:3000/api/items/$ID \
  -H "Content-Type: application/json" \
  -d '{"title": "Updated Vibe Test"}' | jq .

# Read again — verify update stuck
curl -s http://localhost:3000/api/items/$ID | jq '.title'

# Delete
curl -s -X DELETE http://localhost:3000/api/items/$ID

# Confirm it's gone
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/items/$ID
# should be 404
```

This is the test that catches 90% of real bugs. Data didn't persist. Update didn't stick. Delete left orphans.

## Break It

```bash
# Bad input → expect 400
curl -s -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" -d '{"name": ""}' | jq .

# No auth → expect 401
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/protected

# Wrong permissions → expect 403
curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $USER_TOKEN" \
  http://localhost:3000/api/admin/settings

# Missing resource → expect 404
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/users/999999
```

## Check the Logs

While you're at it, make sure the server isn't silently throwing errors:

```bash
grep -i "error\|exception\|traceback" logs/server.log
```

If there's nothing obvious, also check the DB directly for sanity if your change touches the schema:

```bash
sqlite3 dev.db "SELECT * FROM users ORDER BY id DESC LIMIT 3;"
```
