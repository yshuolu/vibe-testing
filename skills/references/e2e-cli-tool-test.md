# CLI Tool Testing

A CLI is arguably the easiest thing to vibe test. You run the command, you look at the output. There's no browser, no API, no layers. Just input → output. And yet people still skip it.

## Build First

Make sure you're testing YOUR version, not some globally installed one:

```bash
npm run build && npm link     # Node.js
go build -o ./bin/mytool ./cmd/mytool   # Go
cargo build                   # Rust
pip install -e .              # Python
```

Verify:
```bash
which mytool    # should point to your local build
```

Or just run from the build directory: `./bin/mytool --version`

## Run It Like a User Would

Don't overthink this. Run the exact command a user would run, and look at what comes out:

```bash
./bin/mytool process --input data.csv --output result.json
echo $?    # exit code 0?
cat result.json | jq .    # output looks right?
```

**Don't just check the exit code.** Read the actual output. Check the fields. Check the values. The number one CLI bug: exit code 0 but wrong output.

## Test Every Flag You Touched

```bash
./bin/mytool process --input data.csv --dry-run    # new flag works?
./bin/mytool process -i data.csv -o result.json -v  # short flags?
./bin/mytool process --input data.csv               # default output location?
```

## Break It

```bash
# Missing required args
./bin/mytool process 2>&1; echo "exit: $?"
# Expect: helpful error, non-zero exit

# Bad file path
./bin/mytool process --input nonexistent.csv 2>&1; echo "exit: $?"
# Expect: "file not found" or similar, non-zero exit

# Invalid option value
./bin/mytool process --input data.csv --format invalid 2>&1; echo "exit: $?"
# Expect: error listing valid formats

# Empty input
echo "" | ./bin/mytool process --input -
```

## Check Help Text

```bash
./bin/mytool --help          # all commands listed?
./bin/mytool process --help  # all flags described?
./bin/mytool --version       # correct version?
```

This matters more than you think. If you added a flag but forgot to document it, users won't find it.

## Test Piping and Redirection

CLIs live in pipelines. Make sure yours plays nice:

```bash
# stdout is clean (no progress bars mixed in)
./bin/mytool list > output.txt && cat output.txt

# JSON output is valid JSON
./bin/mytool list --format json | jq '.[] | .name'

# stderr and stdout are separated
./bin/mytool process --verbose > out.txt 2> err.txt
# out.txt = data only, err.txt = logs only
```

## Exit Codes

These are part of your API contract:

```bash
./bin/mytool process --input valid.csv; echo $?     # expect 0
./bin/mytool process --input missing.csv; echo $?   # expect 1
./bin/mytool lint src/; echo $?                      # 0=clean, 1=issues, 2=error
```

Clean up your test artifacts when done.
