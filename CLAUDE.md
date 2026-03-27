# vibe-testing

This is a Claude Code skill repo. The skill teaches coding agents how to properly test and verify their code changes — not just run unit tests, but actually use the running product like a human QA would.

## Repo Structure

```
skills/
  skill.md                            # The skill entry point (agents load this)
  references/
    unit-test.md                      # Unit testing playbook
    e2e-test.md                       # E2E testing overview + routing
    e2e-server-api-test.md            # API-only verification via curl
    e2e-ui-and-fullstack.md           # UI + full-stack browser verification (combined)
    e2e-ui-quality-check.md           # Visual quality: spacing, alignment, contrast, touch targets
    e2e-cli-tool-test.md              # CLI tool verification
    e2e-library-change-test.md        # Library/package verification via consumer scripts
    auth-test-user.md                 # Auth workarounds: test user creation, JWT, cookies, OAuth
    local-env-secrets.md              # Getting secrets to run locally: cloud CLIs, .env discovery
```

## How the Skill Works

- `skills/skill.md` is the main entry point. It defines the philosophy (three rules) and routes agents to the appropriate reference playbook based on what they changed.
- Each reference file in `skills/references/` is a detailed, self-contained playbook for a specific testing scenario.
- The `e2e-ui-quality-check.md` is referenced from the UI/full-stack playbook and provides concrete visual quality checks (Apple HIG-based numbers for spacing, contrast, alignment, touch targets).

## Writing Style

All docs are written in a direct, opinionated style:
- Lead with the insight, not the procedure
- Short sentences, no corporate fluff
- Code examples are practical and minimal
- Strong opinions stated plainly
