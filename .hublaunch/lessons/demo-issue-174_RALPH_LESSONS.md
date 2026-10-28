# HubLaunch Lessons Learned

## Current Status
- Phase: COMPLETE ✅ — committed as e4f456f4 on branch demo-issue-174
- Feature: Add "when shutdown sequence starts" VRChat status general event
- All checks green: format:web, lint:web, translations, generated:readmes, build:ui, build:overlay-ui, test:web (781 tests pass)
- shutdown-automations.service.ts untouched (acceptance criterion 11). package-lock.json reverted (npm install drive-by).

## Environment note
- Node 20 is default but Angular build needs 22+. Downloaded /tmp/node-v22.22.3-linux-x64; run builds with `export PATH="/tmp/node-v22.22.3-linux-x64/bin:$PATH"`. Non-build checks + vitest work on Node 20.
- Must `npm install` first (fresh container has no node_modules); use node_modules/.bin/vitest (npx reinstalls).
- 2026-10-28 re-run: build:ui / build:overlay-ui failed again with the same "Node 20 detected, need 22+"
  error — but no code was wrong. /tmp/node-v22.22.3-linux-x64 from the prior run was still on disk; just
  re-exported PATH with it and both builds passed immediately, no source changes needed. If that dir is
  ever gone, re-download a Node 22.22.3+/24.15.0+/26.0.0+ linux-x64 build and extract it there.
- 2026-10-28 re-run #2 (HubLaunch auto-fix attempt): identical failure, identical non-fix.
  /tmp/node-v22.22.3-linux-x64 was still present; exporting it onto PATH made both builds pass with zero
  source changes. Each fresh container/session resets PATH to the Node 20 default, so this step must be
  repeated every time build:ui or build:overlay-ui is run, even though the Node 22 download itself persists.
- 2026-10-28 regression run (HubLaunch auto-fix attempt 3): test:web reported 5 failures —
  3 in `mqtt-discovery.service.spec.ts` (`TypeError: Promise.withResolvers is not a function`) and 2 in
  `scripts/migration-transforms.test.ts` (5000ms timeouts). Neither file was touched by this feature's
  commits (confirmed via `git log -- <file>` and diffing against `git merge-base main HEAD`); both predate
  this branch. Root cause: the regression run used the session-default Node 20 (`Promise.withResolvers` is
  Node 22+), and vitest was slow enough under that/this sandbox's load to trip the 5s per-test timeout on
  two migration tests. Re-running `node_modules/.bin/vitest run` after
  `export PATH="/tmp/node-v22.22.3-linux-x64/bin:$PATH"` reproduced all 781 tests passing with zero source
  changes. So: vitest (test:web), not just build:ui/build:overlay-ui, needs Node 22 on PATH — the earlier
  note that "non-build checks + vitest work on Node 20" is wrong in a loaded/slow sandbox and should not be
  relied on; always export the Node 22 PATH before running `npm run test:web` or `vitest` directly, in every
  fresh session.

## Key Discoveries
- Plan claimed `pairwise` was already imported in the service (line 12) — it was NOT. Had to add it.
- Plan claimed event-log reason union was unique at line 259 — there are 12 identical unions in event-log-entry.ts; used `type: 'statusChangedOnGeneralEvent'` context to target the right one.
- `npm run tl set` (node scripts/translation-utils.js set <dotpath> <value>) works for nested keys and inserts alphabetically sorted. Used it for all 4 en.json keys.
- event-log-grouping.spec.ts does NOT enumerate statusChangedOnGeneralEvent reasons → left untouched (out of scope per plan).
- Had to run `npm install` first (node_modules missing vitest). Then use node_modules/.bin/vitest directly (npx tries to reinstall).
- Test runner is vitest v5.0.3. Fake timers + vi.advanceTimersByTimeAsync(500) flushes the async applyStatus chain fine.

## Files Modified
- src-ui/app/models/automations.ts (4 fields + defaults)
- src-ui/app/models/event-log-entry.ts (reason union)
- src-ui/app/services/status-automations/status-change-general-events-automation.service.ts (handler + import + inject)
- src-ui/app/views/.../status-automations-general-tab.component.ts (field, subscription, 2 unions + case)
- status-automations-general-tab.component.html (new block)
- src-ui/assets/i18n/en.json (4 keys via tl set)
- src-ui/app/services/telemetry/feature-snapshot.service.ts (2 booleans)
- CHANGELOG.md (1 Added line)
- status-change-general-events-automation.service.spec.ts (NEW, 7 tests, all pass)

## Next Steps
- Run full check suite: format:web, lint:web, translations, generated:readmes, build:ui, build:overlay-ui, test:web
