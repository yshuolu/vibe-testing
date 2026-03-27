# Unit Tests

Unit tests are necessary but not sufficient. They're your first line of defense, not your last. Think of them as type-checking for behavior — they catch the dumb mistakes early so you can focus your manual verification on the interesting stuff.

The key mistake agents make: writing tests that just verify "the function runs without crashing." That's not a test. A test asserts *specific behavior*. Inputs go in, expected outputs come out. If you can't articulate what the expected output is, you don't understand the code well enough to change it.

## How

**First: find the existing test setup.** Don't introduce a new framework. Match what's there.

```bash
# just look for existing tests
find . -type f -name "*.test.*" -o -name "*_test.*" -o -name "test_*" | head -5
```

**Write tests that would catch your bug if you introduced one.** The mental model: imagine you accidentally inverted a condition in your change. Would any test fail? If no, your tests are useless.

Structure is always the same — arrange, act, assert:

```
Set up the inputs.
Call the function.
Assert the output is what you expect.
```

Test boundaries, not just the happy path. Empty inputs. Nulls. Off-by-ones. The bug is almost always at the boundary.

**Run the tests yourself.** Don't assume they pass.

```bash
npm test                    # JS/TS
pytest path/to/test.py -v  # Python
go test ./... -v            # Go
cargo test -- --nocapture   # Rust
```

When a test fails: read the error. Determine if the test is wrong or the code is wrong. Never delete a failing test to make the suite green.

## What Unit Tests Can't Tell You

- Whether the UI actually renders correctly
- Whether the API returns the right response over HTTP
- Whether the pieces integrate properly
- Whether the user experience is what was intended

This is exactly why you proceed to end-to-end verification after tests pass. Unit tests are step 1. [E2E verification](./e2e-test.md) is step 2.
