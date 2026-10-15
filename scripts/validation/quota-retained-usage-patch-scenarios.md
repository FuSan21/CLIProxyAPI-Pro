# Retained Claude quota customization regression

Recorded before implementation for Management v1.26.4
(`13f94b47f3400e681dfdede65acf1a1242ee3fce`), failed Actions run
`38650799796`.

1. Both `QuotaCard.tsx` and `AuthFileQuotaSection.tsx` now render normal quotas
   under `quota && !retainedClaudeUsage`, and retained Claude usage under a
   separate guard. The old whole-ternary anchor fails before dependencies/tests.
2. Patching only the first component leaves the second failing. Patching only
   the normal body loses the Pro timestamp for successful retained Claude usage.
3. Preserve both upstream guards, loading/error messages and body renderers;
   insert one timestamp alongside each rendered body. Keep the timestamp's
   existing success-only policy, so loading/error observations do not appear fresh.
4. The earlier single-body layout must still apply. Missing or duplicate normal
   body anchors must fail explicitly, without flushing partial edits to disk.
5. Applying twice must produce identical source, without duplicate imports,
   timestamps or quota bodies. Full patch-surface validation must remain unchanged.
6. At final validation, run the complete Management validator on the exact clean
   upstream, including upstream Claude retained-usage rendering tests, lint,
   type checking and production build. Run the generated frontend against the
   repaired Core using `e2e/pro-route-contract/run.py` and retain its receipt and
   HTTP transcript. The HTTP E2E covers route/cache persistence, not external
   providers or visual browser layout.

Focused reproduction: `python3 -m unittest discover -s cliproxyapi-pro-management/tests -p test_quota_card_customization.py`.
Final commands and exact inputs are retained with the task's validation logs.

## Additional failure exposed by the complete validator

The upstream `claudeOAuthUsage.test.ts` raw API-call equality assertion passes
on clean v1.26.4, but fails on the generated tree because Pro already adds the
required `ApiCallResult.hasStatusCode` field. Its typed `result(statusCode, body)`
fixture must include `hasStatusCode: true` for its explicit numeric statuses.
Keep strict equality and the authenticated route, status, body and error-message
assertions; do not drop the production metadata or weaken the test to a subset.
This fixture adaptation adds one reviewed upstream test file to the patch surface.
