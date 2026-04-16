# Review: IBM/ibmi-mcp-server#138 – library list selection for multi-environment support

## PR Under Review

[IBM/ibmi-mcp-server#138](https://github.com/IBM/ibmi-mcp-server/pull/138) — `feat(config): add library list selection for multi-environment support`
**Author:** vijaygovindaraja | **Closes:** IBM/ibmi-mcp-server#133 | **Status:** Open, no reviews yet

---

## Summary

Adds a `library-list` configuration option so that unqualified SQL name resolution can target different library lists (DEV / STG / PROD) without separate server instances. Two input paths:

1. **Env var** — `DB2i_LIBRARY_LIST=MYLIB,DEVDATA,QGPL` (comma-separated)
2. **YAML source** — `library-list:` field (array or comma-separated string)

The library list is threaded through `PoolConnectionConfig` → mapepire-js `Pool({ opts: { libraries: [...] } })` using the native `JDBCOptions.libraries` field. 6 files changed, 600 additions, 1 deletion, 27 new tests.

---

## Overall Assessment

**Positive — well-scoped, cleanly layered, solid test coverage.** The PR follows the project's existing patterns faithfully. A few issues below range from low-severity nitpicks to items that need manual verification on a real IBM i system.

---

## Detailed Findings

### 1. Correctness — opts spread may clobber future JDBCOptions fields (Low)

```ts
// baseConnectionPool.ts — new Pool constructor call
...(poolState.config.libraryList?.length
  ? { opts: { libraries: poolState.config.libraryList } }
  : {}),
```

The `opts` object is constructed with *only* `{ libraries }`. If mapepire-js or a future PR adds other `JDBCOptions` fields (e.g., `naming`, `dateFormat`), this spread will silently discard them because it replaces the entire `opts` key rather than merging into an existing one. Today this is fine — no other code sets `opts` — but it's a latent footgun.

**Suggestion:** Build an `opts` object incrementally and only include the key when it's non-empty:

```ts
const jdbcOpts: JDBCOptions = {};
if (poolState.config.libraryList?.length) {
  jdbcOpts.libraries = poolState.config.libraryList;
}
// ... future opts here ...
const hasOpts = Object.keys(jdbcOpts).length > 0;

poolState.pool = new Pool({
  creds: server,
  maxSize: poolState.config.maxSize || 10,
  startingSize: poolState.config.startingSize || 2,
  ...(hasOpts ? { opts: jdbcOpts } : {}),
});
```

### 2. Schema validation gap — empty array passes silently (Low)

```ts
// config.ts — SourceConfigSchema
"library-list": z.union([
  z.array(z.string().min(1)),
  z.string().transform(...)
])
```

The array branch allows `[]` (empty array). The string branch transforms `""` → `[]`. Both are valid per the schema, so config like `library-list: []` or `library-list: ""` will pass validation and produce an empty array. The downstream code guards with `?.length` so nothing breaks, but an empty library list is semantically meaningless and could confuse users.

**Suggestion:** Add `.min(1)` on the array branch, or document that an empty value is equivalent to omitting the field.

### 3. AuthenticatedPoolManager not wired up (Low — out of scope but worth noting)

`AuthenticatedPoolManager.createPool()` builds a `PoolConnectionConfig` from `IBMiCredentials` but does not include `libraryList`. This means per-token authenticated connections (HTTP auth mode) cannot use library lists. The `IBMiCredentials` interface would need a `libraryList` field and the auth flow would need to carry it.

This is likely out of scope for this PR (library list is a deployment-time config, not a per-token setting), but it should be documented or tracked separately if the feature is ever needed.

### 4. No documentation updates (Medium)

The PR does not update:
- `server/README.md` (environment variable table)
- `docs/configuration.mdx` (Mintlify docs)
- `.env.example` or `server/.env.example`
- `CLAUDE.md` (environment variable table)

Previous PRs in this repo (e.g., rate limiting #112, pool timeouts #121) consistently updated all four. This should follow the same pattern.

### 5. Health endpoint could surface library list (Low — nice to have)

`getHealthSummary()` returns `{ initialized, connecting, healthStatus, lastActivityAt }` per source. Including `libraryList` would make it easier to verify at a glance which library list a running source is using — useful for the exact multi-env debugging scenario this feature enables.

### 6. EnvSchema declares DB2i_LIBRARY_LIST but config.db2i bypasses it (Informational)

The env var is added to `EnvSchema` for documentation/validation purposes, but `config.db2i` reads directly from `process.env.DB2i_LIBRARY_LIST` (not from the parsed `env` object). This is consistent with how `config.db2i` handles all its fields (it's a lazy getter that reads `process.env` directly for CLI late-binding support), so this is **not a bug** — just worth noting that the Zod schema provides no runtime validation for this particular env var.

### 7. Log output includes library list — no sensitive data concern (Positive)

The init log now conditionally includes `libraryList` alongside host/port/masked-user. Library names are not sensitive, so this is appropriate and helpful for debugging.

---

## Gaps Requiring Manual Verification

These items cannot be verified from code review alone and require a live IBM i system:

- [ ] **JDBC library list behavior**: Confirm that `JDBCOptions.libraries` actually sets the JDBC library list (`*LIBL`) for unqualified name resolution — not just the initial library path. Verify with a query like `SELECT * FROM MYTABLE` where `MYTABLE` exists only in one of the listed libraries.
- [ ] **Library list ordering**: Verify that the order of libraries in the array is preserved in the JDBC connection and that name resolution follows that order (first match wins).
- [ ] **Invalid library names**: Test what happens when a non-existent library is included (e.g., `["REALLIB", "DOESNOTEXIST"]`). Does mapepire/JDBC reject the connection, silently ignore the bad entry, or error on first query?
- [ ] **Library name case sensitivity**: IBM i library names are typically uppercase. Verify behavior with lowercase input (e.g., `"mylib"`) — does mapepire uppercase automatically or does it fail?
- [ ] **Special characters / length limits**: IBM i library names are max 10 characters. Confirm the server behaves sensibly with names exceeding that limit.
- [ ] **Interaction with SET PATH / SET CURRENT SCHEMA**: If a YAML tool's SQL statement includes an explicit `SET PATH` or `SET CURRENT SCHEMA`, confirm whether the library list is additive or overridden.
- [ ] **Pool re-use correctness**: When a pool is shared across tools, confirm the library list applies to all connections in the pool — not just the first one.

---

## Summary Table

| # | Finding | Severity | Actionable |
|---|---------|----------|------------|
| 1 | opts spread may clobber future JDBCOptions | Low | Refactor to incremental build |
| 2 | Empty array passes schema validation | Low | Add `.min(1)` or document |
| 3 | AuthenticatedPoolManager not wired | Low | Track separately |
| 4 | No docs updates (README, .env.example, Mintlify) | Medium | Add before merge |
| 5 | Health endpoint could surface library list | Low | Nice to have |
| 6 | EnvSchema vs process.env bypass | Info | No action needed |
| 7 | Log output appropriate | Positive | — |
