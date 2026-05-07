# Issues — Mikrotik Router HACS Integration

## Current Priorities

1. ISS-260417-librouteros-4x-break — librouteros 4.0.1 breaks `connect()` kwarg; all users affected (hotfix v2.3.14 pins `<4.0`)
2. ISS-260320-new-device-discovery — New devices require HA restart to appear
3. ISS-260320-refactor-dedup — Refactor duplicated patterns
4. ISS-260326-tracker-wireless-detection — Device tracker uses old wireless detection logic

---

## Active

### ISS-260507-ups-empty-path — Empty `/system/ups` path disconnects the integration
**Type:** Bug
**Priority:** High
**Created:** 2026-05-07
**Status:** 🔴 Closed — fixed in v2.3.15 (PR for `fix/v2.3.15-ups-poe-current`)

**Symptom:**
On routers with the UPS package enabled but no UPS configured, the integration tile shows "Failed setup, will retry: Mikrotik Disconnected" and never loads. Logs:
```
ERROR ... Mikrotik <host> error while path : no such item
DEBUG ... Finished fetching ... data in 3.380 seconds (success: False)
```
Reported in #61 (RouterOS 7.22.1 on CCR2116-12G-4S+).

**Root cause:**
`coordinator.get_ups()` always issued the `/system/ups monitor` query because the parsed `enabled` field defaulted to `True` when `/system/ups` was empty. The `vals` entry for `enabled` uses `source="disabled"` with `reverse=True`; `from_entry_bool` returns `not default` when the source key is missing, so `enabled` becomes `True` for an empty path. RouterOS then rejects `monitor` with "no such item", and `_query_command` treats it as a connection failure.

**Fix:**
Bail out of `get_ups()` early when `/system/ups` returns no entries — clear `ds["ups"]` and skip both `parse_api` and the `monitor` query.

---

### ISS-260507-poe-current-unit — PoE out current displayed 1000× too large
**Type:** Bug
**Priority:** Medium
**Created:** 2026-05-07
**Status:** 🔴 Closed — fixed in v2.3.15 (PR for `fix/v2.3.15-ups-poe-current`)

**Symptom:**
PoE out current sensor displays `~1234.56 mA` for a port drawing ~25 mA (DE locale shows `1.234,56 mA`). Reported in #60 (RB5009UPr+S+IN, RouterOS 7.22.2).

**Root cause:**
`sensor_types.py` declared `poe_out_current` with `native_unit_of_measurement=AMPERE` and `suggested_unit_of_measurement=MILLIAMPERE`. RouterOS reports the value already in milliamperes (test fixtures use raw integers like `180`, `310`), so HA's unit conversion turned `180 A → 180000 mA`.

**Fix:**
Set `native_unit_of_measurement=MILLIAMPERE`; remove `suggested_unit_of_measurement`.

---

### ISS-260417-librouteros-4x-break — librouteros 4.0.1 breaks connection for all users
**Type:** Bug
**Priority:** Critical
**Created:** 2026-04-17
**Status:** 🟡 In Progress — hotfix branch `fix/librouteros-4x-pin` (v2.3.14) pins `librouteros<4.0`. Proper 4.x migration tracked separately.

**Symptom:**
Users on v2.3.13 (or any prior release) see: `connect() got an unexpected keyword argument 'login_methods'. Did you mean 'login_method'?`

**Root cause:**
librouteros 4.0.1 renamed the `connect()` keyword argument `login_methods` → `login_method`. `manifest.json` declared `librouteros>=3.4.1` with no upper bound, so HACS auto-installed 4.0.1 on upgrade. `mikrotikapi.py:102` still passes the old name.

**Related GitHub issues:**
- #55 (reported 2026-04-10) — direct stack trace
- #56 (reported 2026-04-13) — same symptom post-HA-2026.4.2 update; references upstream #477

**Hotfix (v2.3.14):**
- `manifest.json`: pin `librouteros>=3.4.1,<4.0`

**Follow-up (new ISS to be created for future release):**
- Rename kwarg in `mikrotikapi.py` to `login_method`
- Audit remaining librouteros 4.0.1 breaking changes
- Bump floor to `>=4.0`, drop upper bound

---

### ISS-260320-new-device-discovery — New devices require HA restart to appear
**Type:** Feature
**Priority:** High
**Created:** 2026-03-20
**Status:** 🟡 Backlog
**Source:** coordinator.py line 692, entity.py lines 154-168

**Context:**
The `update_sensors` dispatcher was re-enabled in v2.3.6 to fix new devices not appearing, but it caused thousands of "does not generate unique IDs" log errors every 30s because `_check_entity_exists()` doesn't guard against re-adding existing entities. Reverted in v2.3.8.

**Remaining:**
- Track previously seen UIDs per data path in the coordinator (e.g. `self._known_uids["host"]`)
- Only fire `async_dispatcher_send("update_sensors", self)` when new UIDs appear that weren't in the previous set
- Alternatively, fix `_check_entity_exists()` to skip entities already in `platform.entities`
- Test: add a new host to `ds["host"]` mid-run and verify entity is created without log errors

---

### ISS-260320-refactor-dedup — Refactor duplicated patterns
**Type:** Refactoring
**Priority:** Medium
**Created:** 2026-03-20
**Status:** 🟡 Backlog

**Remaining:**
- coordinator.py: extract firewall rule dedup helper (get_nat/get_mangle/get_filter share ~75 LOC pattern)
- switch.py: extract base class for NAT/Mangle/Filter/Queue UID lookup (~50 LOC)
- ~~apiparser.py: extract shared path traversal from from_entry/from_entry_bool~~ ✅ Done in PR #30
- *_types.py: extract shared entity description base class (~80 LOC)

**Reference:** SonarCloud CPD exclusions already cover sensor_types.py and coordinator.py intentional repetition

---

## Backlog

### ISS-260326-tracker-wireless-detection — Device tracker uses old wireless detection logic
**Type:** Bug
**Priority:** Medium
**Created:** 2026-03-26
**Status:** 🟡 Backlog

**Context:**
`device_tracker.py` lines 157, 169, 199 check `source in ["capsman", "wireless"]` to determine wireless behavior (connection state, icon, attributes). The new `_is_wireless_host()` method in coordinator.py correctly detects wireless clients via bridge host table (fixing hAP ac2), but device_tracker still uses the old check.

**Impact:**
On routers with empty registration tables (hAP ac2 with new WiFi package), wireless clients discovered via bridge table have `source="arp"`, so the device tracker:
- Uses timeout-based `is_connected` instead of registration-based
- Shows wired icon instead of wireless
- Does not show wireless signal/rate attributes

**Fix:**
- Add `is_wireless` bool field to host data in coordinator (set by `_is_wireless_host`)
- Update device_tracker.py to check `self._data.get("is_wireless")` instead of source

---

## Completed

### ISS-260320-test-coverage — Increase test coverage to ≥80%
**Type:** Testing | **Priority:** High | **Created:** 2026-03-20
**Status:** 🔴 Closed — 86% coverage achieved (565 tests, Phase 5 PR)

### ISS-260321-cognitive-complexity — Reduce cognitive complexity to ≤15 per function
**Type:** Quality | **Priority:** High | **Created:** 2026-03-21
**Status:** 🔴 Closed — fixed in refactor/legacy-cleanup (PR #30 + PR #51)

### ISS-260321-silent-failures — Silent failure patterns from security audit
**Type:** Bug/Quality | **Priority:** Medium | **Created:** 2026-03-21
**Status:** 🔴 Closed — fixed in refactor/legacy-cleanup (PR #30 + PR #51)

### ISS-260326-slow-load — Startup bottlenecks blocking HA loading
**Type:** Bug/Performance | **Priority:** High | **Created:** 2026-03-26
**Status:** 🔴 Closed — fixed in v2.3.12 (claude/fix-homeassistant-slow-load)

### ISS-260320-deprecated-datetime — Remaining naive datetime.now() calls
**Type:** Bug | **Priority:** Medium | **Created:** 2026-03-20
**Status:** 🔴 Closed — fixed in v2.3.12 (claude/fix-homeassistant-slow-load)

### ISS-260325-attribute-bloat — ~1300 junk attributes on interface and tracker entities
**Type:** Bug/Quality | **Priority:** High | **Created:** 2026-03-25
**Status:** 🔴 Closed — fixed in v2.3.11-beta.1 (feature/attribute-cleanup)

### ISS-260325-mangle-dedup — Mangle rules with different interfaces removed as duplicates
**Type:** Bug | **Priority:** High | **Created:** 2026-03-25
**Status:** 🔴 Closed — fixed in PR #40 (fix/mangle-duplicate-interface)

### ISS-260324-arp-incomplete — ARP "incomplete" status incorrectly shows device as home
**Type:** Bug | **Priority:** High | **Created:** 2026-03-24
**Status:** 🔴 Closed — fixed in v2.3.10 (PR #38)

### ISS-260322-upstream-frs — Port upstream feature requests
**Type:** Feature | **Priority:** High | **Created:** 2026-03-22
**Status:** 🔴 Closed — released in v2.3.9 (PR #32)

### ISS-260320-options-flow-crash — Options flow crash on HA 2025.12+
**Type:** Bug | **Priority:** Critical | **Created:** 2026-03-20
**Status:** 🔴 Closed — fixed in v2.3.6 (PR #19)

### ISS-260320-blocking-io — Blocking I/O in async methods
**Type:** Bug | **Priority:** High | **Created:** 2026-03-20
**Status:** 🔴 Closed — fixed in v2.3.6 (PR #19)

### ISS-260320-deadlock-run-script — Deadlock in mikrotikapi.py run_script
**Type:** Bug | **Priority:** Critical | **Created:** 2026-03-20
**Status:** 🔴 Closed — fixed in v2.3.6 (PR #19)

### ISS-260320-sonarcloud-token — SonarCloud token expired
**Type:** Infrastructure | **Priority:** Medium | **Created:** 2026-03-20
**Status:** 🔴 Closed — burner token set, SonarCloud passing

### ISS-260320-dispatcher-spam — Duplicate entity log errors from update_sensors
**Type:** Bug | **Priority:** High | **Created:** 2026-03-20
**Status:** 🔴 Closed — dispatcher disabled in v2.3.8 (PR #26). Proper fix tracked as ISS-260320-new-device-discovery

### ISS-260320-ruff-migration — Migrate from Black+flake8 to Ruff
**Type:** Quality | **Priority:** Low | **Created:** 2026-03-20
**Status:** 🔴 Closed — completed in PR #29 (feature/tests-and-refactor). CI uses ruff, all files formatted.
