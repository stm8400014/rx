# Sample Files

These sample documents are meant to show where RX is useful: large-ish JSON-shaped data with repeated structure, shared prefixes, or container lookups that benefit from built-in indexing.

Each sample includes the original JSON and its `.rx` equivalent.

| File | Data shape | What RX is exercising | Typical sparse-read query |
|------|------------|------------------------|---------------------------|
| **quest-log** | Deep nested RPG state | Repeated sub-objects (`rarity`, `reward`), unicode names, ZWJ emoji | Read one quest, inventory item, or stat path |
| **site-manifest** | Large route/path manifest | Shared path prefixes, repeated `auth`/`component` values, object-key lookup | Read one URL path or prefix of routes |
| **emoji-census** | Mixed nested emoji metadata | Unicode-heavy strings, emoji object keys, irregular shapes | Read one category or one emoji record |
| **sensor-grid** | Telemetry-like structured arrays | Packed integer arrays, negative decimals, empty arrays, timestamps | Read one sensor block or time slice |

## Viewing

```sh
# Install `rx` CLI with `npm i -g @creationix/rx`

# Pretty-print as a tree
rx samples/quest-log.rx

# Convert between formats
rx samples/quest-log.rx -j    # rx → JSON
rx samples/quest-log.json -r  # JSON → rx

# Select into a value
rx samples/quest-log.rx -s hero stats
```

The `.rx` files also open in the Rex VS Code extension as an interactive data viewer.

If you are evaluating the format, start with `site-manifest` and `quest-log`. They are the closest to the “large artifact, tiny query” workload RX is designed for.

## Regenerating

To regenerate all `.rx` files from their JSON sources:

```sh
for f in samples/*.json; do rx "$f" -w; done
```
