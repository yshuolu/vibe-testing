# Getting Secrets to Run Locally

Not every project needs secrets. Many apps just work out of the box with local defaults. **Don't preemptively go hunting for env vars.** Just start the app, check the logs, and only come here if something is actually broken.

## How You Got Here

You started the app (with output piped to a temp log file, as you should). Either it crashed on startup, or it's running but throwing runtime errors. You tailed the log and saw errors about missing env vars, failed DB connections, undefined config, or invalid API keys.

If the error is about missing dependencies (`npm install`, `pip install`), that's not a secrets problem — fix that first. If it's about a missing database, check if the project has a `docker-compose.yml` with a DB and just run `docker-compose up -d`. If the error specifically mentions missing env vars or config — read on.

---

## Step 1: Discover What Already Exists

```bash
# Check for existing env files
ls -la .env .env.local .env.development .env.development.local .env.test 2>/dev/null

# Check for env templates
ls -la .env.example .env.sample .env.template env.example 2>/dev/null

# Check docker-compose for env references
grep -r "env_file\|\.env" docker-compose*.yml 2>/dev/null
```

If `.env` exists and has values, the problem is probably a specific var that's wrong or expired — check the error message.

If `.env.example` exists but `.env` doesn't:

```bash
cp .env.example .env
```

Try starting again. Check the logs:

```bash
tail -50 /tmp/app-startup.log
```

If it still fails, check what's still missing:

```bash
grep -E '=$|=your_|=xxx|=TODO|=REPLACE|=<' .env
```

---

## Step 2: Pull Secrets from Cloud CLI (If Available)

Check what CLIs are installed and authenticated:

```bash
which gcloud && gcloud auth list 2>/dev/null
which aws && aws sts get-caller-identity 2>/dev/null
which vercel && vercel whoami 2>/dev/null
which railway && railway whoami 2>/dev/null
which fly && fly auth whoami 2>/dev/null
which heroku && heroku auth:whoami 2>/dev/null
which doppler && doppler me 2>/dev/null
which infisical && infisical user get 2>/dev/null
```

### Vercel

```bash
vercel env pull .env.local
```

### Google Cloud (Secret Manager)

```bash
gcloud secrets list --project=$PROJECT_ID

# Pull all secrets matching .env.example keys
grep -oP '[A-Z_]{3,}' .env.example | sort -u | while read KEY; do
  VALUE=$(gcloud secrets versions access latest --secret="$KEY" --project=$PROJECT_ID 2>/dev/null)
  if [ $? -eq 0 ]; then echo "$KEY=$VALUE"; fi
done > .env
```

### AWS (Secrets Manager / SSM)

```bash
# Secrets Manager
aws secretsmanager get-secret-value --secret-id my-app/dev --query SecretString --output text | jq .

# SSM Parameter Store
aws ssm get-parameters-by-path --path "/myapp/dev/" --with-decryption --output json | \
  jq -r '.Parameters[] | "\(.Name | split("/") | last)=\(.Value)"' > .env
```

### Railway

```bash
railway env
```

### Doppler

```bash
doppler secrets download --no-file --format env > .env
```

### Heroku

```bash
heroku config -a my-app --shell > .env
```

After pulling, restart the app and tail the log to see if the errors are gone.

---

## Step 3: Check for Local Dev Defaults

Some projects run fine locally with hardcoded defaults or docker services:

```bash
# Does the code have fallback values?
grep -r "process.env\.\w\+ || " --include="*.{ts,js}" -l . | head -5
grep -r "os.environ.get.*default" --include="*.py" -l . | head -5

# Does docker-compose provide local services?
grep -E "postgres|mysql|redis|mongo" docker-compose*.yml 2>/dev/null
```

If there's a docker-compose with a DB, the app might just need:

```bash
docker-compose up -d
# and maybe:
echo 'DATABASE_URL=postgres://postgres:postgres@localhost:5432/myapp' >> .env
```

Restart the app and tail the log to confirm.

---

## Step 4: Ask the User

If none of the above worked, ask — but be specific:

1. **Show the error** from the startup log
2. **List the specific vars** that are missing (not all 30 of them — just the ones blocking startup)
3. **Say what you tried** — which CLIs you checked

> The app fails to start with this error:
> ```
> Error: DATABASE_URL is required
> Error: AUTH_SECRET is required
> ```
> I checked for `.env.example` (none found), and `vercel`/`gcloud`/`aws` CLIs are not authenticated.
> Can you provide these values or point me to where the team stores dev secrets?

Never ask for production secrets. You need *development* credentials. Make that clear.

---

## Security Rules

- **Never commit `.env` files.** Check `.gitignore`:
  ```bash
  grep '\.env' .gitignore
  ```
- **Never log secret values in full.** To verify a secret exists:
  ```bash
  grep 'SECRET' .env | sed 's/=.*/=***/'
  ```
- **Never hardcode secrets in source files.**
- **Clean up** — make sure any `.env` you created is in `.gitignore`.
