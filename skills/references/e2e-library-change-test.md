# Library Change Testing

A library has no UI. No server. No CLI. The product is the API surface. So how do you "see" it work? You **become the consumer.** Write a tiny script that imports your library and calls it. Run the script. That's your vibe test.

This is actually the most fun one because it forces you to use your own API. And you'll immediately feel where it's awkward.

## Build It

```bash
npm run build        # JS/TS → check dist/ exists
pip install -e .     # Python
go build ./...       # Go
cargo build          # Rust
```

## Write a Consumer Script

Not a unit test. Not an integration test. A standalone script that does `import yourlib` and uses it, exactly like a real consumer would. Put it somewhere outside the repo, like `/tmp`.

**JavaScript:**
```js
// /tmp/test-consumer.mjs
import { processData, DataProcessor } from '/path/to/lib/dist/index.js';

// Does the basic thing work?
const result = processData({ input: [1, 2, 3], options: { sort: true } });
console.log("Result:", JSON.stringify(result, null, 2));
console.assert(result.sorted === true);
console.assert(result.data.length === 3);

// Does the class work?
const proc = new DataProcessor({ verbose: true });
console.log("Output:", proc.run([4, 5, 6]));

// Does it handle garbage gracefully?
try { processData(null); console.error("BUG: should have thrown"); }
catch (e) { console.log("Correctly rejected null:", e.message); }

console.log("All good.");
```

```bash
node /tmp/test-consumer.mjs
```

**Python:**
```python
# /tmp/test_consumer.py
from mylib import process_data, DataProcessor

result = process_data(input_data=[1, 2, 3], sort=True)
assert result["sorted"] is True
assert len(result["data"]) == 3

proc = DataProcessor(verbose=True)
output = proc.run([4, 5, 6])
assert output["success"] is True

try:
    process_data(input_data=None)
    raise AssertionError("should have thrown")
except (TypeError, ValueError) as e:
    print(f"Correctly rejected None: {e}")

print("All good.")
```

```bash
python /tmp/test_consumer.py
```

**Go:**
```go
// /tmp/test-consumer/main.go
package main

import (
    "fmt"
    "log"
    "github.com/yourorg/yourlib"
)

func main() {
    result, err := yourlib.ProcessData([]int{1, 2, 3}, yourlib.Options{Sort: true})
    if err != nil { log.Fatal(err) }
    fmt.Printf("Result: %+v\n", result)

    _, err = yourlib.ProcessData(nil, yourlib.Options{})
    if err == nil { log.Fatal("should have errored on nil") }
    fmt.Printf("Correctly rejected nil: %v\n", err)

    fmt.Println("All good.")
}
```

Use a `replace` directive in `go.mod` to point at your local copy.

## Focus on What You Changed

The consumer script should exercise the specific behavior you modified:

- **Added a parameter?** Call with and without it. Without should use the default (backward compat).
- **Changed return type?** Verify the new shape. If breaking, verify old usage fails clearly.
- **Fixed a bug?** Reproduce the exact bug scenario in the script. Confirm it no longer occurs.

## Backward Compatibility

If this isn't a major version bump, old call patterns must still work:

```js
// Old API — should still work
const result = processData([1, 2, 3]);  // no options arg
console.assert(result !== undefined, "Old API broke");
```

## TypeScript Types (If Applicable)

Types are part of your public API. Verify they compile:

```ts
// /tmp/test-types.ts
import { processData, ProcessOptions, ProcessResult } from '/path/to/lib';

const opts: ProcessOptions = { sort: true, limit: 10 };
const result: ProcessResult = processData([1, 2, 3], opts);

// @ts-expect-error — should reject wrong type
processData("invalid");
```

```bash
npx tsc --noEmit /tmp/test-types.ts
```

## Verify Exports

Quick sanity check that you didn't accidentally remove a public export:

```bash
node -e "const lib = require('./dist'); console.log(Object.keys(lib))"
```

Clean up `/tmp/test-*.{mjs,py,ts}` when done.
