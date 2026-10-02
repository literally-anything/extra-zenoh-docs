# Key expressions

A **key** is a `/`-separated path such as `robot/arm/joint3/angle`. A **key expression** (KE) describes a
*set* of keys, using wildcards. Subscribers, queryables, storages, ACL rules, interceptors and the admin
space all use key expressions.

## Syntax

| Element | Meaning | Example | Matches |
|---|---|---|---|
| chunk | A non-empty segment between `/` | `robot` | `robot` |
| `*` | Exactly one chunk, any content (except a verbatim chunk) | `robot/*/angle` | `robot/arm/angle` |
| `**` | Zero or more chunks (except verbatim chunks) | `robot/**` | `robot`, `robot/a`, `robot/a/b/c` |
| `$*` | Any sequence of characters *inside* one chunk | `robot/arm$*` | `robot/arm`, `robot/arm42` |
| `@` prefix | **Verbatim chunk**: only matches the exact same chunk | `@/router` | Only something with the `@` chunk there |

### Rules the parser enforces

From `commons/zenoh-keyexpr/src/key_expr/borrowed.rs`:

- No empty chunks, and no leading or trailing `/` (`/a`, `a/`, `a//b` are invalid).
- `#` and `?` are forbidden anywhere, because they act as separators in [selectors](../api/query.md#selectors)
  and endpoints.
- `*` must be the whole chunk (`*` or `**`). For partial-chunk wildcards, use `$*`.
- `$` is only allowed as part of `$*`, and `$` may not follow `$*`.
- The key expression must be in **canonical form**. Use `autocanonize` (most APIs offer it) to fix a
  non-canonical expression.

### Canonical form

Canonicalization rewrites equivalent forms into one spelling (tests in `canon.rs`):

| Input | Canonical |
|---|---|
| `hello/**/**/bye` | `hello/**/bye` |
| `hello/$*/bye` | `hello/*/bye` |
| `hello/foo$*$*/bar` | `hello/foo$*/bar` |
| `hello/**/*` | `hello/*/**` |

## Intersection and inclusion

Two operations underlie everything:

- **intersects(A, B)**: at least one key belongs to both sets. Routing uses this: a subscriber on `a/*`
  receives a `put` on `a/b`, and a query on `a/**` reaches a queryable on `a/b/c`.
- **includes(A, B)**: every key in B is also in A. Storages (`complete` storages), ACL, low-pass filters and
  queryables' `complete` flag use inclusion.

`relation_to()` returns a `SetIntersectionLevel`: `Disjoint`, `Intersects`, `Includes` or `Equals`.

## Verbatim chunks (`@`)

A chunk that starts with `@` can only be matched by an identical chunk. `*` and `**` never match it.
This keeps internal namespaces out of reach of broad wildcards:

- A subscriber on `**` does **not** receive admin-space traffic under `@/<zid>/router/...`.
- [Advanced pub/sub](../api/advanced-pubsub.md) uses `@adv/...` key expressions for its internal liveliness and cache queries.
- To query the admin space you have to write the `@` chunk explicitly: `@/*/router/**`.

Examples (from `commons/zenoh-keyexpr/src/key_expr/tests.rs`):

| A | B | Intersect? | Why |
|---|---|---|---|
| `@a` | `@a` | yes | identical verbatim chunk |
| `@a` | `@ab` | no | different verbatim chunk |
| `@a` | `@a/**` | yes | `**` can match zero chunks |
| `@a/*` | `@a/@b` | no | `*` can't match `@b` |
| `@a/**` | `@a/@b` | no | `**` can't match `@b` |
| `@a/**/@b` | `@a/@b` | yes | `@b` is written explicitly |
| `@a/**/e` | `@a/b/b/@c/b/d/d/d/e` | no | `**` can't cross `@c` |
| `@a` | `**/@a` | yes | `**` matches zero chunks before `@a` |

## Key expression formats

`KeFormat` (and the `kedefine!`, `keformat!`, `kewrite!` macros in Rust) let you declare a structured key
space like `robot/${id:*}/sensor/${name:*}` and build or parse keys against it. The admin space uses this
internally, for example `@/${zid:*}/${whatami:*}/config/${key:**}`.

## Declared key expressions

`session.declare_keyexpr("robot/arm")` gives a key expression a numeric ID on the wire. Later messages can
send the short ID instead of the full string, which saves bandwidth on frequently used keys. Publishers and
subscribers do this for you automatically.

## Sources

- `commons/zenoh-keyexpr/src/key_expr/borrowed.rs`, `canon.rs`, `intersect/`
- `commons/zenoh-keyexpr/src/key_expr/format/`
- `zenoh/src/api/key_expr.rs`
