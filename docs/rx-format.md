# RX Format Spec

This document is the formal grammar and encoding reference for the `.rx` text format used by `@creationix/rx`. It is intended to make the format understandable without reading the source code.

RX covers the same data model as JSON: objects, arrays, strings, numbers, booleans, and `null`. Pointers, chains, refs, and indexes are encoding features that make large documents smaller and faster to query.

For interactive inspection, paste any RX or JSON into the live viewer at <https://rx.run/>.

## Reading direction

RX is parsed right-to-left. Every value has a tag character with a base64 varint to its right, and may have a body to its left:

```text
[body][tag][b64 varint]
            ◄── read this way ──
```

The parser starts at the rightmost byte and scans left past base64 digits until it hits a non-b64 byte — that byte is the tag. The b64 digits to its right are the varint. The tag then determines whether there is a body to the left and how to interpret it.

For example, in `hi,2`:
- Start at the right: `2` is a b64 digit (varint = 2)
- Next byte left: `,` is not a b64 digit — this is the tag (string)
- The tag says there are 2 bytes of body to the left: `hi`

## Grammar overview

*Railroad diagram coming soon — top-level value production showing all branches.*

A **value** is one of:

```
value = integer | decimal | string | ref | object | array | pointer | chain
```

Each branch is described below with its tag, encoding layout, and (where added) a railroad diagram.

## Base64 varint

*Railroad diagram coming soon — b64 digit alphabet.*

RX uses the alphabet `0-9 a-z A-Z - _` (64 characters, URL-safe, no padding) for variable-length unsigned integers.

- Big-endian digit order
- Zero is an empty string (zero digits)
- Signed integers use zigzag encoding: 0 → 0, -1 → 1, 1 → 2, -2 → 3, ...

| Decimal | Zigzag | B64 digits | As written with `+` tag |
|---------|--------|------------|------------------------|
| 0 | 0 | (empty) | `+` |
| 1 | 2 | `2` | `+2` |
| -1 | 1 | `1` | `+1` |
| 42 | 84 | `1k` | `+1k` |
| 255 | 510 | `7-` | `+7-` |

## Tags

| Tag | Name | Layout | Description |
|-----|------|--------|-------------|
| `+` | Integer | `+[b64 zigzag]` | Zigzag-decoded signed integer |
| `*` | Decimal | `[base node]*[b64 zigzag exponent]` | `base × 10^exponent`; base is a full `+` node |
| `,` | String | `[UTF-8 bytes],[b64 length]` | Raw UTF-8, length in bytes |
| `'` | Ref | `'[name bytes]` | Built-in literal or external ref name |
| `:` | Object | `[children]:[b64 content-size]` | Key/value pairs |
| `;` | Array | `[children];[b64 content-size]` | Ordered child values |
| `^` | Pointer | `^[b64 delta]` | Backward delta to an earlier byte offset |
| `.` | Chain | `[segments].[b64 content-size]` | Concatenated string segments |
| `#` | Index | `[entries]#[b64 compound]` | Sorted lookup table for a container |

## Primitives

### Integer (`+`)

*Railroad diagram coming soon.*

```text
+[b64 zigzag-value]
```

The varint is zigzag-decoded to produce a signed integer. Small integers with no trailing zeroes use this tag directly. Larger integers with trailing zeroes may be encoded as decimals.

| JSON | RX |
|------|----|
| `0` | `+` |
| `1` | `+2` |
| `-1` | `+1` |
| `42` | `+1k` |

### Decimal (`*`)

*Railroad diagram coming soon.*

```text
[+ base node]*[b64 zigzag-exponent]
```

Represents `base × 10^exponent`. The base is encoded as a full integer node (with its own `+` tag and zigzag-encoded varint). The exponent is zigzag-encoded and appears as the `*` tag's own varint.

The parser reads `*` and its b64 digits first (getting the exponent), then recursively reads the node to the left (getting the base).

| JSON | RX | Base node | Exponent |
|------|----|-----------|----------|
| `3.14` | `+9Q*3` | `+9Q` (314) | `*3` (-2) |
| `1000000` | `+2*c` | `+2` (1) | `*c` (6) |

Special float values use refs instead: `'inf` (+Infinity), `'nif` (-Infinity), `'nan` (NaN).

### String (`,`)

*Railroad diagram coming soon.*

```text
[UTF-8 bytes],[b64 byte-length]
```

The body contains raw UTF-8 bytes. The varint gives the byte length (not character count). Strings may contain any bytes including nulls and non-ASCII unicode.

| JSON | RX |
|------|----|
| `""` | `,` |
| `"hi"` | `hi,2` |
| `"alice"` | `alice,5` |

### Ref / simple literal (`'`)

*Railroad diagram coming soon.*

```text
'[name bytes]
```

Refs are unique among tags: the "varint" bytes after `'` are not a numeric value but a name. The parser reads the bytes to the right of `'` as the ref name and checks for built-in names first.

Built-in refs encode JSON literals and special floats:

| Value | RX |
|-------|----|
| `true` | `'t` |
| `false` | `'f` |
| `null` | `'n` |
| `undefined` | `'u` |
| `+Infinity` | `'inf` |
| `-Infinity` | `'nif` |
| `NaN` | `'nan` |

Non-built-in ref names refer to entries in an external dictionary agreed between encoder and decoder.

## Containers

### Array (`;`)

*Railroad diagram coming soon.*

```text
[child_N]...[child_1];[b64 content-size]
```

Children are encoded in the body from left to right. The varint gives the total byte size of the content region. The parser computes the left edge as `tag_position - content_size` and iterates children right-to-left from the tag back to that edge.

Large arrays may include an index (see Index section) between the last child and the `;` tag.

| JSON | RX | Children (right-to-left parse order) |
|------|----|--------------------------------------|
| `[]` | `;` | (none) |
| `[1,2,3]` | `+6+4+2;6` | `+2` → 3, `+4` → 2, `+6` → 1 |

### Object (`:`)

*Railroad diagram coming soon.*

```text
[value_N][key_N]...[value_1][key_1]:[b64 content-size]
```

Keys and values alternate in the body: key₁, value₁, key₂, value₂, ... (in left-to-right byte order). Key order is preserved. Keys are typically strings but may be pointers or chains.

Large objects may include an index and/or a schema between the last key-value pair and the `:` tag. When present, the rightmost item is the schema, followed by the index. The parser checks for these before setting the content boundary.

| JSON | RX |
|------|----|
| `{}` | `:` |
| `{"a":1,"b":2}` | `+4b,1+2a,1:a` |
| `{"users":["alice","bob"],"version":3}` | `+6version,7bob,3alice,5;cusers,5:w` |

## Sharing and random access

### Pointer (`^`)

*Railroad diagram coming soon.*

```text
^[b64 delta]
```

A pointer refers to an earlier value by backward delta — the distance in bytes from the pointer's tag position back to the target value's right edge. To resolve: `target = tag_position - delta`, then read the value at that offset.

Pointers enable:
- **Value deduplication** — identical strings, objects, or subtrees are written once
- **Schema sharing** — objects with the same keys reference a shared key layout

### Chain (`.`)

*Railroad diagram coming soon.*

```text
[segment_N]...[segment_1].[b64 content-size]
```

A chain is a concatenated string built from segments. Each segment is itself a value — typically a string, pointer, or another chain. The varint gives the total byte size of the segments.

Chains compress keys with shared prefixes. For example, `/docs/getting-started` and `/docs/encoding` might share a `/docs/` prefix segment via a pointer, with only the suffix differing.

### Index (`#`)

*Railroad diagram coming soon.*

```text
[fixed-width entries]#[b64 compound]
```

An index is a sorted lookup table attached to a container (array or object). It appears inside the container body, between the content and the container's tag.

The compound varint packs two values: the low 3 bits encode `width - 1` (digits per entry, supporting widths 1–8), and the remaining upper bits encode `count` (number of entries):

```
compound = (count << 3) | (width - 1)
```

Each entry is a fixed-width base64 number giving the backward delta from the index position to the corresponding child. For objects, entries point to keys and are sorted in UTF-8 byte order.

Indexes enable:
- **O(1) array access** — jump directly to the Nth element
- **O(log n) object key lookup** — binary search on sorted keys
- **O(log n + m) prefix search** — find the first matching key, then scan forward

Without an index, array access and key lookup are O(n) linear scans.

## Schemas

Objects can store their keys separately from their values using a schema reference. This is useful when many objects share the same key set (e.g., rows in a table-like structure).

A schema object stores only values in its content body. The schema node appears as the rightmost item inside the object (before any index). The parser identifies it by checking the tag:

- **Pointer schema** (`^`) — points to another object or array whose keys become this object's keys
- **Ref schema** (`'`) — names an external dictionary entry containing the key list

The encoder detects shared key sets automatically and emits schema pointers where beneficial. The first object with a given key set is encoded normally; subsequent objects with the same keys store only their values and a pointer back to the first object's key layout.

## External refs

Encoders and decoders can share an external dictionary of values. When a value matches a ref entry by identity, the encoder writes `'name` instead of the full value. The decoder looks up the name in the same dictionary to reconstruct the original value.

Refs are not part of the JSON data model. They are an agreed-upon external table — useful for values that appear across multiple documents or are too expensive to embed repeatedly.

## Encoding example walkthrough

Given this JSON:

```json
{"users":["alice","bob"],"version":3}
```

The RX encoding is: `+6version,7bob,3alice,5;cusers,5:w`

Reading right-to-left:

1. `:w` — tag `:` (object), b64 `w` = 32 → content is 32 bytes to the left
2. `users,5` — tag `,` (string), b64 `5` = 5 → "users" (5 bytes), this is key₁
3. `;c` — tag `;` (array), b64 `c` = 12 → content is 12 bytes, this is value₁
4. `alice,5` — tag `,` (string), b64 `5` = 5 → "alice", array element₁
5. `bob,3` — tag `,` (string), b64 `3` = 3 → "bob", array element₂
6. `version,7` — tag `,` (string), b64 `7` = 7 → "version", this is key₂
7. `+6` — tag `+` (integer), b64 `6` → zigzag 6 = 3, this is value₂

## Versioning

This document describes the current encoding used by `@creationix/rx`. The format originated as the internal bytecode for rex (a DSL for HTTP routing), which is why pointers, chains, and indexing are first-class concepts.

Future versions may add new tags or encoding features. The tag character set and right-to-left reading direction are stable.
