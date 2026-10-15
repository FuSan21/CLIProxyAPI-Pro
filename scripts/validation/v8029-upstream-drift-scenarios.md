# Core v8.0.29 replay scenarios
1. Claude constructor comment moves into ForDialerAndRoots: full generator must replay without requiring the comment beside wrapper signature.
2. NewUtlsHTTPClient inserts TLS handshake timeout capture and deferred transport wrapper: runtime proxy resolution must change without removing timeout wrapper or Decorate.
3. Both dialer helper functions, TLS roots, session cache, timeout context and upstream tests must remain unchanged.
4. Second generator replay must be byte-identical.
Exact upstream: 5fc580b1d72d87a7bcfe5cfb6f479c1aea5625b3.

## Publication fence migration failures (recorded before implementation)
5. Existing generator requires old registerResolvedModelsForAuth body; upstream now delegates to registerResolvedModelsForAuthFenced. Full replay currently fails at this anchor (replay-after.log).
6. A current local queue token must NOT allow an auth update based on a superseded upstream identity (prefix/excluded-model changes). Retain UpdateIfRegistrationCurrent CAS and its early-return path.
7. A current local queue token must NOT overwrite or unregister a newer registry publication. Pass the original upstream fence to both publish and unregister; do not rebuild it at commit time.
8. A stale local queue token must still reject a model publication, unregister, or plugin auth update after removal/recreation or a newer queued registration.
9. A successfully CAS-updated plugin auth must refresh only fence auth inputs, retaining the original registry epoch.
10. Preserve upstream test entrypoint signatures and semantics, plus delayed completion/cancellation/disabled/empty-model cases.
11. Preserve authModelCommitMu -> coreManager -> registry lock ordering and release it after removal and policy cleanup.
Existing coverage: upstream model_registration_publication_regression_test.go, plugin auth publication tests, local pro_features_service_test.go. Final focused Go tests to run after full generator replay succeeds.

## Scheduler lifecycle drift (recorded before implementation)
12. Upstream updateInternal now calls finishAuthUpdate after releasing the auth mutation gate. Register still contains the original snapshot/upsert block; the generator incorrectly expects two identical blocks (fence-after.log).
13. Both registration and successful updates, including UpdateIfRegistrationCurrent, must still route scheduler refresh through policy-aware RefreshSchedulerEntry. Keep finishAuthUpdate after mutation unlock, its hooks, structural epoch and persistence behavior unchanged.
14. Inspection refresh guard is inserted into updateInternalLocked, whose new return arity is (*Auth, bool, bool, error). The old injected `return nil, ErrInspectionAuthChanged` fails compilation (fence-tests.jsonl); rejection must return nil,false,false,error, preventing scheduler/hook side effects.

## Reproduction and evidence

Evidence directory for the 2026-10-15 repair: `/private/tmp/core-worker-20261015`.
The paths above are local artifacts, not inputs required by the patch.

- `replay-before.log`: original generator rejects the moved Claude cache comment.
- `focused-before.log` and `focused-after.json`: the same AST-extracted uTLS replacements fail before the repair and succeed afterward; compare the dialer helpers and timeout wrapper byte-for-byte.
- `fence-before.log`: replay rejects the old model registration body before the fence migration.
- `scheduler-before.log`: replay rejects the old two-upsert assumption.
- `inspection-build-before.jsonl`: the Go compiler rejects the old two-value inspection return inside the new four-value helper.
- `focused-tests.jsonl`: focused Go regression results after replay.

Extract the exact upstream tarball into a fresh temporary directory, then run from the Pro repository:

```sh
SRC_ROOT=/path/to/fresh/CLIProxyAPI python3 cliproxyapi-pro-core/patches/apply_upstream_patches.py
GOTOOLCHAIN=go1.26.0 go -C /path/to/fresh/CLIProxyAPI mod tidy
GOTOOLCHAIN=go1.26.0 go -C /path/to/fresh/CLIProxyAPI test ./sdk/cliproxy ./sdk/cliproxy/auth ./internal/runtime/executor/helps -run 'Test(CurrentRegistrationToken|ModelRegistration|QueuedModelRegistration|StalePluginAuth|OAuthPolicyInFlight|BuiltInOAuthPolicy|Manager_SchedulerTracksRegisterAndUpdate|.*InspectionRefresh|.*TLSHandshake|.*ClaudeCodeRoundTripper|.*RuntimeProxy)' -count=1 -json > focused-tests.jsonl
```

The generator intentionally rejects a second application without modifying the already-patched tree; scenario 4 requires this guard and unchanged file hashes, not a successful second application. Final full validation is run by the integration owner after development.

## Fence integration invariants

The Pro queue token gate remains outside the upstream publication fence. Lock order remains `authModelCommitMu` -> manager per-auth mutation gate -> registry mutex. Do not compare execution-only `Auth.Generation`. Upstream `RegistrationEpoch`, prefix, provider, disabled state, auth kind, attributes, and the original registry epoch remain authoritative.

The pre-implementation regression additions cover a still-current Pro token with superseded auth inputs and a still-current Pro token with a newer registry publication (both publish and unregister). Existing tests cover queued stale work, removal/recreation, in-flight policy filtering, and upstream prefix/generation transitions. Scheduler refresh remains after mutation unlock in both Register and finishAuthUpdate; it uses RefreshSchedulerEntry so account policy overlays remain execution-only.

## Focused verification result

On the exact upstream commit with Go 1.26.0, all three focused packages passed: 180 test/subtest results. Repeating the same selection with `-race -count=3` passed all 540 results (`focused-race-tests.jsonl`). The reapplication guard rejected the second application and all 2,200 file hashes remained unchanged (`reapply-result.json`). The uTLS helper/timeout-wrapper preservation check also passed. These focused results do not replace the integration owner's final clean full validation.
