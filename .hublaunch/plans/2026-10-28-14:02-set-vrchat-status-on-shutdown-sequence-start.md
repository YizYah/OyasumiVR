# Set your VRChat status when the shutdown sequence starts

## Plan Summary

- **What / why:** Add a fourth "general event" to the existing VRChat status automation so that, when the shutdown sequence starts, OyasumiVR can optionally set the user's VRChat online status and/or status message. This fixes the real-world annoyance that someone who powers their PC down via the shutdown sequence after sleeping never "wakes up," so a sleep-time status (e.g. *Ask Me*) persists on their account until changed by hand.
- **Key decision:** Implement this as a new event in `StatusChangeGeneralEventsAutomationService` (mirroring the existing *sleep preparation* handler), triggered by observing `ShutdownAutomationsService.stage` transition from `'IDLE'` → non-IDLE. **Not** a new shutdown-sequence stage, **not** a "VRChat closed" trigger. The behaviour is **ungated** (applies whenever the sequence starts and the option is on), matching the three sibling general events.
- **Most important files:** `src-ui/app/services/status-automations/status-change-general-events-automation.service.ts` (new handler), `src-ui/app/models/automations.ts` (four new config fields + defaults), and the General tab component/template under `.../status-automations-general-tab/`.
- **Scope:** TypeScript under `src-ui` only — no Rust, C#, overlay, or proto changes. No config `version` bump and no migration (additive fields are back-filled by `normalizeWithDefaults`).
- **Priority / complexity:** Medium priority (user-requested feature), low-to-medium complexity (purely additive, closely mirrors an existing, well-tested pattern).

## Problem Statement

OyasumiVR lets users run a "shutdown sequence" that turns off devices, quits SteamVR, and powers down the PC after they fall asleep. Users who rely on this flow set a sleep-time VRChat status (for example *Ask Me*) but never go through a "wake up" step, because the PC powers off while they sleep. As a result, the sleep-time status remains on their VRChat account server-side and shows the next time VRChat launches — until they notice and reset it manually. The requester describes their "sleeping" status commonly lingering for an entire day.

The existing "general events" status automation already changes the VRChat status on three events — sleep mode enabled, sleep mode disabled, and sleep preparation — but there is no event tied to the shutdown sequence. The fix adds exactly that: an opt-in "when the shutdown sequence starts" event that sets the chosen status and/or message, letting the user leave a daytime status instead of the sleep-time one.

#### Planning Context

**Key requirements**

- Add a fourth general-events status event: *when the shutdown sequence starts*, optionally set the VRChat status and/or status message.
- The option is **off by default**; when on, the user picks a status (default `Active`) and may independently enable a status message (default empty, 32-char limit — matching the siblings).
- Reuse the existing `applyStatus()` path (rate-limited `VRChatService.setStatus()`, `AUTO_UPDATED_VRC_STATUS` notification, event log) — add only a new event-log `reason`, not a new logging mechanism.
- Fire **once** when the sequence starts (the IDLE → non-IDLE transition), not on every later stage change within the same run, and not when the sequence returns to IDLE.

**Decisions made (with rationale)**

1. **Placement — a fourth general event, not a shutdown stage** (per the maintainer's answer to the scoping question). Adding a shutdown *stage* would touch `ShutdownSequenceStageOrder`, the overlay, and the shutdown view; a general event is the smaller, self-contained change and lives beside its three siblings. Rationale: smallest change that fully resolves the request, in the repo's existing conventions.
2. **Trigger — subscribe to `ShutdownAutomationsService.stage` and detect the pairwise `'IDLE'` → non-IDLE transition.** This detects the sequence starting *without changing* `shutdown-automations.service.ts`. `stage` is already a public observable (`shutdown-automations.service.ts:75`). A new `sequenceStarted` observable on the shutdown service is explicitly acceptable *only if clearly cleaner*, but the pairwise approach touches no other file and is preferred. Rationale: the maintainer required that nothing else in `shutdown-automations.service.ts` change.
3. **Ungated behaviour** (per the answer to the gating question). The user turns the option on and picks the status; there is no extra "only when sleep mode is active" toggle. This matches the three sibling general events and is the simplest behaviour.
4. **No config version bump / no migration.** The four new fields are additive and are back-filled by `normalizeWithDefaults`; this repo reserves `version` bumps for data transforms. Rationale: follow the repo convention the maintainer confirmed.
5. **No circular DI risk.** `StatusChangeGeneralEventsAutomationService` will inject `ShutdownAutomationsService`. `ShutdownAutomationsService` does not depend on the status service (verified: its constructor at `shutdown-automations.service.ts:83-94` injects sleep, config, app settings, OpenVR, device control, lighthouse, event log, translate, vrchat, and device manager — not the status automation), so the new dependency is acyclic.
6. **Debounce no longer than the siblings.** `handleSleepPreparation` uses `debounceTime(500)`; the new handler uses the same, because the PC may power down shortly after the sequence starts and a longer debounce risks the status change never being sent.

**Out of scope**

- A "when VRChat closes" trigger (the `vrchatProcessActive` observable) — explicitly left out.
- A new shutdown-sequence stage, and any change to stage order, the overlay, or the shutdown view.
- Any Rust, C#, or proto change.
- Any locale file other than `src-ui/assets/i18n/en.json`.
- Event-log *grouping* causes (the reason is not added to grouping logic).
- `.hublaunch/goals.md` or any file outside the fix.

## Detailed Requirements

### Functional

- A new setting block appears on the status automations **General** tab, below the "When I prepare to go to sleep" block: a toggle + status dropdown ("When the shutdown sequence starts → Set my status to …") and an independently-toggleable status message input (32-char max).
- When the shutdown sequence starts and the status toggle is on, the user's VRChat status is set to the chosen value. When the message toggle is on, the status message is set to the chosen value. Each applies independently; if neither toggle is on, nothing is sent.
- The change applies only when the parent automation (`CHANGE_STATUS_GENERAL_EVENTS.enabled`) is on and the VRChat user is logged in and **not** Offline — identical gating to the sibling handlers.
- The change fires once per sequence run (on IDLE → non-IDLE), producing an event-log entry with reason `SHUTDOWN_SEQUENCE` and the existing `AUTO_UPDATED_VRC_STATUS` notification when enabled.

### Technical

- Add four fields to `ChangeStatusGeneralEventsAutomationConfig` and their defaults.
- Add `'SHUTDOWN_SEQUENCE'` to the `reason` union of `EventLogStatusChangedOnGeneralEvent`.
- Add a `handleShutdownSequence()` method shaped exactly like `handleSleepPreparation()`, wired from `init()`.
- Extend the General tab component (`setStatusOption`/`updateStatusMessage` unions, a new `onShutdownSequenceStatusOption` field, config subscription) and its template (one more block copied from the sleep-preparation block).
- Add English copy only; extend the telemetry boolean list; add a CHANGELOG line.

### Non-Functional

- Follow **REVIEW.md**: use Angular signal APIs where the file already does, `takeUntilDestroyed()` as the last pipe operator in components, no accessibility-only code, and telemetry must carry booleans only (no VRChat IDs/usernames/messages).
- Follow **AGENTS.md** comment rules: signposts only where they aid reading, one-line body comments, no history/rationale in code comments.
- No new dependencies. Smallest change that resolves the request.

## Proposed Solution

Add the event in the same service that already owns the three sibling events, reusing the entire `applyStatus()` plumbing. The only new wiring is a subscription that watches the shutdown sequence's `stage` observable and emits once when it leaves `'IDLE'`.

```
ShutdownAutomationsService.stage  ──(IDLE → non-IDLE, pairwise)──►  handleShutdownSequence()
                                                                        │  gate on enabled + non-offline user
                                                                        │  map config → {status?, message?}
                                                                        ▼
                                              applyStatus(status, message, 'SHUTDOWN_SEQUENCE')
                                                 └─ VRChatService.setStatus() ─ notification ─ eventLog
```

The General tab gains one more toggle/dropdown + toggle/input block; everything else is additive config, copy, telemetry, and tests.

<details><summary><b>Implementation Detail</b></summary>

### Implementation Steps

#### Phase 1 — Config model and defaults

**File:** `src-ui/app/models/automations.ts`

- In `ChangeStatusGeneralEventsAutomationConfig` (interface at lines 308–321), after the sleep-preparation fields (line 320), add:
  ```ts
  changeStatusOnShutdownSequence: boolean;
  statusOnShutdownSequence: UserStatus;
  changeStatusMessageOnShutdownSequence: boolean;
  statusMessageOnShutdownSequence: string;
  ```
- In `AUTOMATION_CONFIGS_DEFAULT.CHANGE_STATUS_GENERAL_EVENTS` (lines 826–840), after `statusMessageOnSleepPreparation: ''` (line 839), add:
  ```ts
  changeStatusOnShutdownSequence: false,
  statusOnShutdownSequence: UserStatus.Active,
  changeStatusMessageOnShutdownSequence: false,
  statusMessageOnShutdownSequence: '',
  ```
  `UserStatus.Active` is the default because it is the value sleep-mode-disable already resets to (`automations.ts:833`) and represents a normal "awake" status. **Do not** bump the `AutomationConfigs` version (line 53) and **do not** add a migration step — these additive fields are back-filled by `normalizeWithDefaults`.

#### Phase 2 — Event log reason

**File:** `src-ui/app/models/event-log-entry.ts`

- In `EventLogStatusChangedOnGeneralEvent` (lines 257–264), extend the `reason` union (line 259) to:
  ```ts
  reason: 'SLEEP_MODE_ENABLED' | 'SLEEP_MODE_DISABLED' | 'SLEEP_PREPARATION' | 'SHUTDOWN_SEQUENCE';
  ```
  No other change to this file.

#### Phase 3 — The automation handler

**File:** `src-ui/app/services/status-automations/status-change-general-events-automation.service.ts`

- Import and inject `ShutdownAutomationsService` in the constructor (lines 29–37), alongside the existing injected services. Import from `../shutdown-automations.service`.
- In `init()` (lines 39–48), call `this.handleShutdownSequence();` after `this.handleSleepPreparation();`.
- Add `handleShutdownSequence()` modelled on `handleSleepPreparation()` (lines 86–106). It subscribes to `this.shutdownAutomations.stage`, detects the start with `pairwise()` + `filter`, then mirrors the sleep-preparation map/gate/apply. Sketch:
  ```ts
  private handleShutdownSequence() {
    this.shutdownAutomations.stage
      .pipe(
        // fire once when the sequence starts
        pairwise(),
        filter(([prev, next]) => prev === 'IDLE' && next !== 'IDLE'),
        filter(() => this.config.enabled),
        filter(() => Boolean(this.vrcUser && this.vrcUser.status !== UserStatus.Offline)),
        map(() => {
          const status: UserStatus | null = this.config.changeStatusOnShutdownSequence
            ? this.config.statusOnShutdownSequence
            : null;
          const statusMessage: string | null = this.config.changeStatusMessageOnShutdownSequence
            ? this.config.statusMessageOnShutdownSequence
            : null;
          return { status, statusMessage };
        }),
        filter((data) => Boolean(data.status || data.statusMessage)),
        debounceTime(500)
      )
      .subscribe(({ status, statusMessage }) =>
        this.applyStatus(status, statusMessage, 'SHUTDOWN_SEQUENCE')
      );
  }
  ```
  - `pairwise` and `filter`/`map`/`debounceTime` are already imported at line 12.
  - `ShutdownSequenceStage` values come from `shutdown-automations.service.ts:54-59`; the service seeds `stage` with `'IDLE'` (line 74) and sets it back to `'IDLE'` at run end, so `prev === 'IDLE' && next !== 'IDLE'` fires exactly once at the start of each run.
  - Keep the `debounceTime(500)` identical to the sibling; do **not** lengthen it (the PC may power down shortly after start).
  - `applyStatus` (lines 108–142) already handles the notification and event log; pass the new reason string.
- **Circular-DI check:** confirmed safe — `ShutdownAutomationsService` (`shutdown-automations.service.ts:83-94`) does not inject the status service.

#### Phase 4 — General tab UI

**Component:** `src-ui/app/views/dashboard-view/views/status-automations-view/tabs/status-automations-general-tab/status-automations-general-tab.component.ts`

- Add field `onShutdownSequenceStatusOption?: SelectBoxItem;` after `onSleepPreparationStatusOption?` (line 75).
- In the config subscription (lines 93–104), after setting `onSleepPreparationStatusOption` (lines 101–103), add:
  ```ts
  this.onShutdownSequenceStatusOption = this.statusOptions.find(
    (s) => s.id === this.config.statusOnShutdownSequence
  );
  ```
- Extend the `automation` union in `setStatusOption` (line 116) to include `'ON_SHUTDOWN_SEQUENCE'`, and add a `case` (after the `ON_SLEEP_PREPARATION` case at lines 131–135):
  ```ts
  case 'ON_SHUTDOWN_SEQUENCE':
    await this.updateConfig({ statusOnShutdownSequence: option.id! as UserStatus });
    break;
  ```
- Extend the `automation` union in `updateStatusMessage` (line 140) to include `'ON_SHUTDOWN_SEQUENCE'`, and extend the key selection (lines 144–149) so `'ON_SHUTDOWN_SEQUENCE'` maps to `'statusMessageOnShutdownSequence'`. Keep the existing ternary idiom used in the file.

**Template:** `src-ui/app/views/dashboard-view/views/status-automations-general-tab/status-automations-general-tab.component.html`

- Copy the entire sleep-preparation block (lines 142–210 — the two `app-setting-row`s: toggle + status dropdown, then toggle + message input) and paste it after line 210, replacing throughout:
  - `onSleepPreparation` label/description keys → `onShutdownSequence` equivalents.
  - `changeStatusOnSleepPreparation` → `changeStatusOnShutdownSequence`; `statusOnSleepPreparation` → `statusOnShutdownSequence`; `changeStatusMessageOnSleepPreparation` → `changeStatusMessageOnShutdownSequence`; `statusMessageOnSleepPreparation` → `statusMessageOnShutdownSequence`.
  - `[selected]="onSleepPreparationStatusOption"` → `[selected]="onShutdownSequenceStatusOption"`.
  - `setStatusOption('ON_SLEEP_PREPARATION', …)` → `setStatusOption('ON_SHUTDOWN_SEQUENCE', …)`; `updateStatusMessage('ON_SLEEP_PREPARATION', …)` → `updateStatusMessage('ON_SHUTDOWN_SEQUENCE', …)`.
  - Template ref `#onSleepPreparationStatusMessageInput` → `#onShutdownSequenceStatusMessageInput` (and its use in the `(input)` handler).
- No accessibility-only markup (REVIEW.md); match the surrounding structure exactly.

#### Phase 5 — English copy

**File:** `src-ui/assets/i18n/en.json` (only)

- Under `status-automations.general` (object starting near line with the `onSleepPreparation` copy — the General tab block contains `description`, `onSleepEnable`, `onSleepDisable`, `onSleepPreparation`), add an `onShutdownSequence` sibling. The existing `onSleepPreparation` copy is:
  ```json
  "onSleepPreparation": {
    "description": { "status": "Set my status to", "statusMessage": "Set my status message to" },
    "title": "When I prepare to go to sleep"
  }
  ```
  Add:
  ```json
  "onShutdownSequence": {
    "description": { "status": "Set my status to", "statusMessage": "Set my status message to" },
    "title": "When the shutdown sequence starts"
  }
  ```
- In `comp.event-log-entry.type.statusChangedOnGeneralEvent.reason` (lines 644–648), add a `SHUTDOWN_SEQUENCE` entry matching the phrasing style of the siblings ("As sleep mode was enabled", "As you prepared to go to sleep"):
  ```json
  "SHUTDOWN_SEQUENCE": "As the shutdown sequence started"
  ```
- Prefer `npm run tl set` for adding these `en.json` keys (writes `en.json` alone, per AGENTS.md). Do not touch any other locale file.

#### Phase 6 — Telemetry

**File:** `src-ui/app/services/telemetry/feature-snapshot.service.ts`

- In the `vrchat_status_on_events` boolean list (lines 119–128), add the two new booleans after `status.changeStatusMessageOnSleepPreparation` (line 127):
  ```ts
  status.changeStatusOnShutdownSequence,
  status.changeStatusMessageOnShutdownSequence,
  ```
  Booleans only — no user-identifying data (REVIEW.md).

#### Phase 7 — Changelog

**File:** `CHANGELOG.md`

- Add one line under `## [Unreleased]` → `### Added`, in the file's existing "Added a …" wording style, e.g.:
  ```
  - Added an option to change your VRChat status when the shutdown sequence starts
  ```

#### Phase 8 — Tests

**File (new):** `src-ui/app/services/status-automations/status-change-general-events-automation.service.spec.ts`

Write in the style of the sibling service specs (`src-ui/app/services/brightness-cct-automation.service.spec.ts:1-60`, `gpu-automations.service.spec.ts`): Vitest, `vi.mock` for the Tauri plugin/event/core modules, a `createService()` factory that constructs the service with mocked dependencies fed by `BehaviorSubject`s, and assertions via `vi.waitFor()`.

- Mock `VRChatService` with `user` (a `BehaviorSubject<CurrentUser | null>`), `setStatus` (a `vi.fn()` resolving `true`), and anything `applyStatus` touches; mock `NotificationService.notificationTypeEnabled` → `false` (so the notification branch is skipped); mock `EventLogService.logEvent`; mock `ShutdownAutomationsService` with a `stage` `BehaviorSubject<ShutdownSequenceStage>('IDLE')`; mock `AutomationConfigService.configs` with a `BehaviorSubject` carrying `CHANGE_STATUS_GENERAL_EVENTS`. Mock `SleepService`/`SleepPreparationService` with empty observables so their handlers stay quiet.
- Build config via `structuredClone(AUTOMATION_CONFIGS_DEFAULT.CHANGE_STATUS_GENERAL_EVENTS)` and override per test.
- Cases:
  1. **Sets status and message when the sequence starts and both options are on:** `enabled: true`, `changeStatusOnShutdownSequence: true`, `statusOnShutdownSequence: UserStatus.Active`, `changeStatusMessageOnShutdownSequence: true`, `statusMessageOnShutdownSequence: 'gm'`, user status `AskMe`. After `init()`, push `stage` → `'TURNING_OFF_DEVICES'`; expect `setStatus(UserStatus.Active, 'gm')` called once and an event logged with `reason: 'SHUTDOWN_SEQUENCE'`.
  2. **Status only / message only:** with just the status toggle on, expect `setStatus(status, null)`; with just the message toggle on, expect `setStatus(null, message)`.
  3. **Does nothing when the option is off:** both `changeStatus*OnShutdownSequence` false → `setStatus` not called (the `filter(status || statusMessage)` short-circuits).
  4. **Does nothing when the automation is disabled:** `enabled: false` → `setStatus` not called.
  5. **Does nothing when the user is offline:** user status `Offline` (or user `null`) → `setStatus` not called.
  6. **Fires once per run, not on later stage changes:** after the start transition and one `setStatus` call, pushing `stage` → `'QUITTING_STEAMVR'` then `'POWERING_DOWN'` must not trigger further `setStatus` calls; returning to `'IDLE'` must not trigger a call either.
- Account for the `debounceTime(500)` by using `vi.useFakeTimers()` + `vi.advanceTimersByTimeAsync(500)` or `vi.waitFor()`, following whichever timer idiom the sibling specs use.

If `event-log-grouping.spec.ts` enumerates `statusChangedOnGeneralEvent` reasons, extend it so the new reason is covered; otherwise leave grouping untouched (grouping causes are out of scope).

### Edge Cases

- **Sequence cancelled before it starts:** the trigger is the IDLE → non-IDLE transition, which happens when `runSequence()` sets the first stage; a cancellation that never leaves IDLE never fires the event. A cancellation *after* a stage has begun does not re-fire (the event already fired once on the leading edge).
- **Both toggles off but automation enabled:** `filter((data) => Boolean(data.status || data.statusMessage))` short-circuits, so no `setStatus` call and no event — matching the siblings.
- **User logged out / Offline:** gated out identically to the sibling handlers.
- **Rapid successive runs:** each run ends by returning `stage` to `'IDLE'`, so the next run's IDLE → non-IDLE transition fires again as expected.
- **Status-change rate limit:** `VRChatService.setStatus()` is already rate-limited (6/min) and shared across all status automations; no extra handling needed.

### Performance

Negligible: one additional RxJS subscription on an observable that emits only during shutdown sequences.

### Security / Privacy

- Telemetry additions are booleans only; no VRChat IDs, usernames, or status messages are sent (REVIEW.md).
- No new external calls; reuses the existing sanitized, rate-limited `setStatus` path.

### Dependencies & Related Work

- No new npm or Cargo dependencies.
- Related existing code: the three sibling general events (`status-change-general-events-automation.service.ts:50,86`), the shutdown sequence (`shutdown-automations.service.ts`), and the player-count status automation (which this does *not* modify).

</details>

## Technical Considerations

- **Dependency wiring:** injecting `ShutdownAutomationsService` into `StatusChangeGeneralEventsAutomationService` is acyclic (verified above). Both are `providedIn: 'root'` singletons.
- **Config compatibility:** no version bump; `normalizeWithDefaults` (`src-ui/app/migrations/migration-defaults.ts`) back-fills the four new fields for existing users. This matches the maintainer's explicit instruction and the repo's "version bumps only for data transforms" convention.
- **i18n:** only `en.json` changes; use `npm run tl set` so it writes `en.json` alone. Other locales are left for separate translation work (AGENTS.md).
- **Checks:** run the relevant `check:*` npm scripts for the changed TypeScript/Angular code; translation and generated-README checks are read-only.

## Testing Strategy

- **Unit (Vitest):** the new `status-change-general-events-automation.service.spec.ts` covering the cases in Phase 8 (sets status/message on start; status-only; message-only; does nothing when off / disabled / user offline; fires once per run, not on later stage changes or on return to IDLE).
- **Integration:** if `event-log-grouping.spec.ts` enumerates the general-event reasons, extend it to include `SHUTDOWN_SEQUENCE`; otherwise no integration test change (grouping is out of scope).
- **Manual (desktop UI verification, per `docs/agents/desktop-ui-testing.md`):** on the status automations General tab, confirm the new "When the shutdown sequence starts" block renders below the sleep-preparation block, the toggles enable/disable the dropdown and input, and the 32-char limit applies. Trigger a shutdown sequence (with devices/SteamVR/power-down configured so a stage runs) and confirm the VRChat status changes once and an event-log entry with the new reason appears. This requires VRChat login and a running SteamVR; note in the PR body if the sandbox cannot exercise the full sequence.

## Documentation Updates

- `CHANGELOG.md`: one line under `[Unreleased] / ### Added` (Phase 7).
- No README or generated-doc change is required (this adds a setting within an existing feature area; the READMEs are generated from `docs/readmes/src` and do not enumerate individual status events).

## Acceptance Criteria

1. `ChangeStatusGeneralEventsAutomationConfig` has the four new fields, with defaults `false`, `UserStatus.Active`, `false`, `''`; no `AutomationConfigs` version bump and no new migration step.
2. `EventLogStatusChangedOnGeneralEvent['reason']` includes `'SHUTDOWN_SEQUENCE'`.
3. When the shutdown sequence starts, the automation is enabled, and the user is logged in and not Offline: with the status toggle on, the VRChat status is set to the chosen value; with the message toggle on, the message is set — each independently. Nothing is sent when both toggles are off.
4. The status change fires exactly once per sequence run (on IDLE → non-IDLE), not on later stage changes within the run and not on return to IDLE.
5. Nothing fires when the automation is disabled or the user is Offline/logged out.
6. An event-log entry with `reason: 'SHUTDOWN_SEQUENCE'` is produced, and the `AUTO_UPDATED_VRC_STATUS` notification is sent when that notification type is enabled.
7. The General tab shows a new "When the shutdown sequence starts" block (toggle + status dropdown, toggle + 32-char message input) styled identically to the sleep-preparation block, with working reset-to-default.
8. English copy exists for the new tab block and the new event-log reason; no other locale file is modified.
9. Telemetry's `vrchat_status_on_events` includes the two new booleans; no user-identifying data is added.
10. `CHANGELOG.md` has one `### Added` line in the file's style.
11. New Vitest spec passes and the existing suite still passes; `shutdown-automations.service.ts` is unmodified (unless a minimal, clearly-cleaner `sequenceStarted` observable is added there and nothing else changes).
12. No Rust/C#/overlay/proto changes; no new dependencies; no issue/PR number in any commit, branch, or PR text.

## Assumptions

- **Current checkout vs snapshot:** HEAD is `f60fdce3`, which matches the snapshot commit `f60fdce36ec5c443474b26c0c2e6d7e0ceb03c33`; the working branch is `demo-issue-174` rather than `develop`, but the tree is identical — no drift. All referenced `file:line` locations were verified against this checkout. Per the fork rules, cut the implementation branch from `develop` and target `develop`; do not reference the upstream issue/PR number anywhere.
- **Trigger implementation:** use the pairwise `stage` IDLE → non-IDLE subscription inside `StatusChangeGeneralEventsAutomationService`, leaving `shutdown-automations.service.ts` untouched. A new `sequenceStarted` observable on the shutdown service is adopted *only* if it proves clearly cleaner during implementation and nothing else in that file changes; default to the pairwise approach.
- **Default status:** `UserStatus.Active` (the awake/normal status, consistent with the sleep-mode-disable default).
- **Copy wording:** tab title "When the shutdown sequence starts"; event-log reason "As the shutdown sequence started"; reuse the sibling "Set my status to" / "Set my status message to" descriptions. If the maintainer prefers different phrasing, it is a one-line copy change in `en.json`.
- **Message length limit:** 32 characters, matching the sibling inputs.
- **Debounce:** keep `debounceTime(500)`, identical to `handleSleepPreparation`, to avoid losing the change if the PC powers down soon after the sequence starts.
- **Event-log grouping:** the new reason is added to the grouping spec only if that spec already enumerates general-event reasons; grouping cause logic itself is out of scope.
