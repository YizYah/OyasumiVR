# HubLaunch Lessons Learned

## Current Status
- Phase: COMPLETE ✅ — committed as e4f456f4 on branch demo-issue-174
- Feature: Add "when shutdown sequence starts" VRChat status general event
- All checks green: format:web, lint:web, translations, generated:readmes, build:ui, build:overlay-ui, test:web (781 tests pass)
- shutdown-automations.service.ts untouched (acceptance criterion 11). package-lock.json reverted (npm install drive-by).

## Environment note
- Node 20 is default but Angular build needs 22+. Downloaded /tmp/node-v22.22.3-linux-x64; run builds with `export PATH="/tmp/node-v22.22.3-linux-x64/bin:$PATH"`. Non-build checks + vitest work on Node 20.
- Must `npm install` first (fresh container has no node_modules); use node_modules/.bin/vitest (npx reinstalls).

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
