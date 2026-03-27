# vibe-testing

This is a Claude Code skill repo. The skill teaches coding agents how to properly test and verify their code changes — not just run unit tests, but actually use the running product like a human QA would.

## Dependencies

This skill uses [playwright-cli](https://github.com/yshuolu/playwright-cli) for browser automation with built-in auth state injection.

## Repo Structure

```
skills/
  SKILL.md                       # Entry point: philosophy, auth decision tree, commands
  references/
    e2e-test.md                  # E2E setup: auth, start app, pick playbook
    e2e-auth-test-user.md        # Test user: methodology + framework examples
    e2e-server-api-test.md       # API verification via curl
    e2e-ui-and-fullstack.md      # UI + full-stack browser verification
    e2e-ui-quality-check.md      # Visual quality: spacing, contrast, alignment
    e2e-cli-tool-test.md         # CLI tool verification
    e2e-library-change-test.md   # Library/package verification via consumer scripts
    unit-test.md                 # Unit testing playbook
    local-env-secrets.md         # Getting secrets to run locally
```

## Decision Tree

```
SKILL.md (entry point)
  ├─ Unit tests → unit-test.md
  ├─ Start the app → local-env-secrets.md (if blocked)
  └─ E2E test → e2e-test.md
       ├─ Scenario A: Developer's laptop → --cookies
       ├─ Scenario B: Isolated env → TEST_USER_ID
       │    ├─ Stack not ready? Fix plumbing → e2e-auth-test-user.md
       │    └─ Stack ready → set env var, proceed
       └─ Test
            ├─ API → e2e-server-api-test.md
            ├─ UI / Full-stack → e2e-ui-and-fullstack.md
            │    └─ Quality check → e2e-ui-quality-check.md
            ├─ CLI → e2e-cli-tool-test.md
            └─ Library → e2e-library-change-test.md
```

## Writing Style

All docs are written in a direct, opinionated style:
- Lead with the insight, not the procedure
- Short sentences, no corporate fluff
- Code examples are practical and minimal
- Strong opinions stated plainly
