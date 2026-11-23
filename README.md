# argus-bundles

[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/igor-vii/argus-bundles/badge)](https://scorecard.dev/viewer/?uri=github.com/igor-vii/argus-bundles)

Public storage of sealed Argus engine bundles, per battery:

```
bundles/<battery>/engine.bundle.tar.gz   # the engine payload
bundles/<battery>/manifest.json          # argus.bundle.v0 + Ed25519 seal
```

This repo is **untrusted transport** by design. Integrity is enforced by the
consumer (`igor-vii/argus-action`), which verifies before executing anything:

1. `manifest.schema === "argus.bundle.v0"`
2. `manifest.key_id` must be `active` in the keys pinned **in the action repo**
   (`keys/bundle-keys.json` — never from the SUT repo)
3. Ed25519 `signature` over the canonical form of the manifest (without the
   `signature` field) must verify against that key
4. sha256 of the downloaded `engine.bundle.tar.gz` must equal
   `manifest.tree_sha256`

Broken seal → the engine is never executed (`action_required`, zero charge).

Bundles are built by the `build-bundle` workflow in `igor-vii/argus-lab`
(preflight secret↔anchor is fail-closed, шаг 0 of the publication runbook) and
published here once verification passes.

## Publishing rules

- **Publish = a normal commit to `main`** (git data API or web UI): put the
  new `bundles/<battery>/engine.bundle.tar.gz` + `bundles/<battery>/manifest.json`
  pair in one commit, message states version + `key_id` + `tree_sha256`
  prefix and the `argus-lab` build run id.
- **`main` is protected**: force-push and branch deletion are disabled
  (branch protection). History here is append-only in practice.
- **Never edit a published bundle in place**: bump the engine version and
  publish a new manifest+tarball pair. Consumers (`argus-action`) cache by
  engine version; a rewritten bundle under the same version breaks the
  cached-sha trust chain.
- The manifest must reference a key that is `active` in the action's pinned
  `keys/bundle-keys.json`; retired keys stay in that file for history but
  never verify a new bundle.
- Distribution to CI goes through the console
  (`GET /api/v1/engine/bundle`, 302 → signed URL); this repo is the raw
  storage behind that redirect. See the canonical contract:
  [`igor-vii/argus-action` → `docs/api-contract.md`](https://github.com/igor-vii/argus-action/blob/main/docs/api-contract.md).
