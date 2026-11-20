# argus-bundles

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
