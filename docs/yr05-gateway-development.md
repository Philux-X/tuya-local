# YR05 Gateway Development Notes

This file is a persistent handoff document for experimental YR05 gateway
support in `tuya-local`.

Future Codex sessions working on this feature should read:

1. `AGENTS.md`
2. `docs/yr05-gateway-development.md`

This document captures the confirmed laboratory facts and the repository
architecture findings from the initial read-only investigation. It is intended
to avoid repeating a broad repository survey before implementation work.

## Scope

The goal is to support a YR05 lock reached locally through a Tuya SigMesh
gateway.

The preferred shape is:

- a new YAML device profile for the YR05 lock
- a small reusable Python extension in the existing lock platform
- no new Tuya-BLE repository changes
- no hardcoded secrets
- no invented DPs or authentication mechanism

Phases 1 through 3 have now been implemented on branch
`feature/yr05-gateway-lock` and published for hardware testing as prerelease
`v2026.8.1-yr05.1`. Treat this file as the persistent development record for
that experimental branch, not as evidence that the feature has been merged
upstream.

## Initial Milestone

Do not expand scope until this milestone works reliably.

Initial support should include only:

- YR05 discovered and configured as a gateway child
- battery from DP8
- a Home Assistant Lock entity
- lock through `DP46=true`
- unlock through authenticated DP71
- physical lock state from DP47

### Real-Hardware Validation Status

The initial milestone has been validated on real hardware through Home
Assistant using the SigMesh gateway path.

Confirmed behavior:

- the YR05 can be configured as a child device behind the SigMesh gateway
- protocol auto-detection selects Tuya LAN protocol 3.4
- profile `yr05_h13_lock` can be selected successfully
- DP8 battery reports correctly
- DP47 physical lock state reports correctly
- Home Assistant lock sends `DP46=true` and physically locks the YR05
- Home Assistant unlock sends authenticated DP71 and physically unlocks the
  YR05
- Home Assistant state updates correctly after Home Assistant-originated lock
  and unlock operations
- Smart Life receives the expected notification for Home Assistant-originated
  authenticated unlock
- Smart Life remains able to operate the lock while local gateway control is
  used
- local gateway control is cleaner and more reliable than the prior BLE/ESP
  path used during experimentation

Secondary features are intentionally deferred:

- DP12 fingerprint event presentation
- DP13 PIN/password event presentation
- DP19 BLE/mobile event presentation
- language
- beep volume
- automatic locking
- temporary credentials
- fingerprints
- PIN management
- any other lock features beyond the initial milestone

Implementation and tests should keep this boundary clear. It is acceptable for
the document to record confirmed secondary DPs, but the first implementation
should not depend on them.

## Confirmed Laboratory Facts

### Gateway and Transport

- The SigMesh gateway communicates locally over Tuya LAN protocol 3.4.
- The YR05 is locally addressable through gateway CID
  `cd72a26b41cc2f02`.
- TinyTuya parent/child transport works with a parent gateway plus child CID.
- `child.status()` works through the gateway.
- DP31 writes work through the gateway.
- DP46 writes work through the gateway.
- DP71 writes work through the gateway.
- Smart Life continues to receive lock notifications while local gateway
  control is used.

### Confirmed Product Identity

- product ID: `hhxgpozj`
- product name: `Smart Lock`
- category: `jtmspro`
- model: `H13电商款-新芯片`

### Confirmed YR05 DPs

Only the DPs below are confirmed for YR05 from laboratory testing:

| DP | Meaning |
| --- | --- |
| 8 | Battery |
| 12 | Fingerprint unlock / credential event |
| 13 | PIN/password unlock / credential event |
| 19 | BLE/mobile unlock event |
| 28 | Language |
| 31 | Beep volume |
| 33 | Automatic lock enable |
| 36 | Auto-lock delay |
| 46 | Manual lock command, write `true` to lock |
| 47 | Physical lock state, `false` means locked, `true` means unlocked |
| 71 | Authenticated BLE unlock RAW payload |

Do not invent additional YR05 DPs without new device logs or Tuya data model
evidence.

### Confirmed Lock Behavior

- Writing `DP46=true` locks the physical YR05.
- After DP46 lock, `DP47=false` is reported.
- Authenticated DP71 unlock works through the gateway.
- After DP71 unlock, `DP47=true` is reported.
- `DP19=1` is reported after BLE/mobile unlock.

DP47 is the authoritative physical lock state for the YR05:

- `DP47=false` means locked
- `DP47=true` means unlocked

DP46 should be treated as a one-way lock command:

- confirmed: write `true` to lock
- not confirmed: write `false` to unlock
- do not implement YR05 unlock by writing `DP46=false`

### Known Limitation: Physical Credential Events

Detailed physical credential/event-history records are not currently observed
by Home Assistant in real time through the local SigMesh gateway path.

Confirmed limitation:

- physical fingerprint and PIN unlock activity does not currently appear in
  Home Assistant as detailed credential/user records in real time
- Smart Life eventually receives those credential/event-history records later
- disabling phone Bluetooth did not make those physical events appear in Home
  Assistant through the gateway path

Current interpretation:

- DP47 state is confirmed and should remain the physical lock-state authority
- DP8, DP46, DP47, DP71, gateway-child transport, and Smart Life coexistence
  are confirmed
- real-time DP12, DP13, and DP19 event passthrough from the lock/gateway into
  `tuya-local` is not confirmed
- do not assume fingerprint, PIN, manual, or other detailed physical-event
  records are locally available through the gateway until targeted logs prove
  it

Unknowns and hypotheses:

- unknown whether the SigMesh gateway emits physical credential records over a
  parent-level LAN event channel that TinyTuya does not currently expose
- unknown whether a gateway-specific subscription or different protocol
  command is required to enable child-device event reporting
- possible that ordinary `status()` polling retrieves only current DP state,
  while credential/event-history records travel through a separate delayed app
  or cloud path
- possible that child-device messages do arrive on the gateway socket but are
  ignored because they are not decoded into the simple `{"dps": ...}` shape
  consumed by `tuya-local`

## DP71 Authentication

YR05 unlock uses an authenticated RAW payload on DP71.

The source material is an existing `ble_unlock_check` Base64 value.

Validation:

- Base64-decode the source value.
- The decoded source must be exactly 19 bytes.
- Reject missing, invalid, or wrong-length source values.

Source/report layout:

| Field | Size |
| --- | --- |
| peripheral_id | 2 bytes |
| central_id | 2 bytes |
| random | 8 bytes |
| operation | 1 byte |
| timestamp | 4 bytes |
| method | 1 byte |
| result | 1 byte |

Unlock request layout:

| Field | Value |
| --- | --- |
| central_id | bytes 2..3 from source |
| peripheral_id | bytes 0..1 from source |
| random | bytes 4..11 from source |
| operation | `0x01` |
| timestamp | current Unix timestamp, big-endian 4 bytes |
| method | `0x00` |
| result | `0x01` |

The result is 19 bytes. For TinyTuya LAN transport, Base64-encode those
19 bytes before writing DP71.

This was physically verified to unlock the YR05. It is not the same as the
existing smart-lock PIN/code authentication payloads.

## Existing Gateway Architecture

### Config Entry Identity

Relevant file:

- `custom_components/tuya_local/helpers/config.py`

`get_device_id(config)` returns `device_id/device_cid` when both are present.
This gives gateway subdevices a unique Home Assistant identity separate from
the parent gateway.

Relevant file:

- `custom_components/tuya_local/const.py`

`CONF_DEVICE_CID = "device_cid"` already exists. Protocol version `3.4` is
already in `TUYA_PROTOCOL_VERSIONS`.

### Cloud and Config Flow

Relevant file:

- `custom_components/tuya_local/config_flow.py`

Indirect devices discovered from the cloud can be attached to a hub. The flow
uses the child `node_id` or `uuid` as `device_cid`.

Relevant behavior:

- indirect child devices require a selected hub
- `CONF_DEVICE_CID` is populated from child `node_id` or `uuid`
- the selected hub supplies the LAN address used to reach the child
- the config entry unique ID uses `device_id/device_cid`

Important observation:

- In the current cloud flow, when an indirect child has a `local_key`, the flow
  appears to replace the hub local key with the child local key before local
  connection testing.
- Laboratory testing confirmed TinyTuya parent/child transport works with the
  tested gateway setup.
- Do not change this local-key behavior without targeted testing. It may be
  relevant if a gateway requires the parent gateway local key rather than the
  child local key.

### TinyTuya Parent/Child Construction

Relevant file:

- `custom_components/tuya_local/device.py`

`TuyaLocalDevice.__init__` already supports `dev_cid`.

When `dev_cid` is present:

- a TinyTuya parent `Device` is created for the gateway
- a child TinyTuya `Device` is created using the child CID
- the child uses `cid=dev_cid`
- the child receives `parent=parent`
- the parent and child share the same asyncio lock

This matches the confirmed YR05 transport shape: parent gateway plus child CID.

Protocol handling already rotates and applies protocol version to both child
and parent when a parent is present.

### Runtime Device Storage

Relevant file:

- `custom_components/tuya_local/device.py`

`setup_device` stores the runtime `TuyaLocalDevice` and TinyTuya API object in
`hass.data[DOMAIN][get_device_id(config)]`.

Subdevice identity is therefore already keyed by the combined parent and child
identifier rather than only by the gateway ID.

## Existing Lock Platform

Relevant file:

- `custom_components/tuya_local/lock.py`

The lock platform currently supports:

- writable `lock` DP
- readonly `lock_state` DP
- readonly `open` DP
- unlock event DPs such as fingerprint, password, card, manual, remote, app,
  temporary, BLE, offline password, and face unlock
- secure code paths using `code_unlock`, `set_unlock_code`,
  `request_unlock`, and `approve_unlock`
- jammed/problem state

Important current behavior:

- `async_lock()` writes `true` to a writable `lock` DP.
- `async_unlock()` writes `false` to a writable `lock` DP.

This means YR05 DP46 must not be represented as the existing bidirectional
`lock` DP unless the Python logic changes. For YR05, DP46 is confirmed only as
a lock command where `true` locks the device.

## Existing Secure-Lock Authentication

Relevant file:

- `custom_components/tuya_local/lock.py`

Existing secure-lock/code authentication uses:

- `code_unlock`
- `set_unlock_code`
- an 8-character code
- member ID/source fields
- Base64 output payloads

Existing helpers:

- `build_code_unlock_msg`
- `build_code_set_msg`

What can be reused:

- the lock entity structure
- the async set-value path
- the pattern of building a binary payload and Base64-encoding it before write
- the existing `sensitive` device-config concept for diagnostics redaction
- unlock event reporting through `changed_by`

What must be added:

- a DP71-specific authenticated BLE unlock payload builder
- validation for exactly 19 decoded source bytes
- timestamp injection
- peripheral/central ID reordering
- an explicit opt-in DP name so existing passive `ble_unlock_check` profiles do
  not gain new active unlock behavior accidentally

The existing secure-lock code path is not directly compatible with YR05 DP71.

## RAW and Base64 Handling

Relevant files:

- `custom_components/tuya_local/helpers/device_config.py`
- `custom_components/tuya_local/devices/README.md`

The device config schema already supports `type: base64`.

`TuyaDpsConfig.decode_value` can decode Base64 strings to bytes.

`TuyaDpsConfig.encode_value` can Base64-encode bytes.

Normal writable DP paths ultimately send values through TinyTuya. For DP71,
the lock platform should provide the Base64 string that TinyTuya LAN expects.

No schema change appears necessary only to support DP71 as a Base64-capable DP,
but tests and README known-DP documentation should be updated if new DP names
are introduced.

## State Caching and Push Behavior

Relevant file:

- `custom_components/tuya_local/device.py`

The device runtime maintains `_cached_state`.

Receive/update behavior:

- incoming status data is merged into `_cached_state`
- entities are notified through `entity.on_receive`
- non-persistent DPs can be cleared after full polls
- pending writes are overlaid briefly through `_pending_updates`

For YR05:

- DP47 should be treated as authoritative lock state.
- Do not fake DP47 after DP46 or DP71.
- Rely on the confirmed DP47 report after lock/unlock.
- DP19 can be used as an unlock event source after BLE/mobile unlock.

The confirmed gateway behavior shows DP47 updates after both local lock and
authenticated unlock, so the minimal implementation does not need special
gateway polling logic.

### Gateway Event-Passthrough Findings

Focused code inspection after real-hardware testing found no generic
gateway-specific event subscription or child-event passthrough layer in
`tuya-local`.

Repository facts:

- `TuyaLocalDevice.__init__` constructs a TinyTuya child `Device` with
  `cid=dev_cid` and `parent=parent` when `device_cid` is configured.
- `TuyaLocalDevice.async_receive` alternates `updatedps()` and `status()` for
  full refreshes, sends `heartbeat()` while persistent, and otherwise waits on
  `self._api.receive()`.
- `TuyaLocalDevice.receive_loop` handles dictionaries returned by TinyTuya. If
  the dictionary contains `dps`, it unwraps that map, merges it into
  `_cached_state`, calls each entity's `on_receive`, and schedules entity state
  updates.
- The receive loop does not decode raw gateway envelopes, node-event frames,
  or BLE-style message bodies itself.
- If TinyTuya child transport returns an unsolicited child update as a normal
  `dps` map, current `tuya-local` code should cache it and notify entities.
- If the gateway emits physical credential records only on the parent socket,
  in a non-`dps` payload shape, or behind a required subscription command, the
  current code has no explicit decoder or subscription path for that.

Why state can work while detailed events do not:

- DP47 is a durable current-state datapoint, so polling and normal DP updates
  are sufficient for lock state.
- DP12, DP13, and DP19 are event-like records. If they are transient,
  non-persistent, absent from normal child `status()` responses, or delivered
  through a separate gateway/event-history mechanism, the current gateway path
  can miss them even while DP47 continues to work.
- The current minimal YR05 profile intentionally does not expose DP12, DP13, or
  DP19. Even if those DPs arrived as ordinary `dps` updates, there is currently
  no YR05 entity to present them.

This limitation looks partly unimplemented in `tuya-local` and partly
unconfirmed at the gateway protocol layer. Current code can consume ordinary
child `dps` pushes, but there is no evidence in the inspected code of a
generic gateway event-history passthrough mechanism.

## Known-Working BLE Reference

The local `tuya_local_ble` implementation in `C:\Projects\Tuya-BLE` is a
known-working behavioral reference for YR05 physical activity in Home
Assistant. It was inspected read-only and should not be modified as part of
gateway support.

Repository facts from the BLE reference:

- `custom_components/tuya_local_ble/lock.py` maps YR05 product `hhxgpozj` with
  state DP47, lock command DP46, and authenticated unlock DP71.
- YR05 BLE lock state is derived from DP47. In the BLE lock entity,
  `DP47=false` means locked and `DP47=true` means unlocked.
- BLE lock sends the YR05 lock command by setting DP46 to `true`.
- BLE authenticated unlock builds a raw DP71 request from configured
  `ble_unlock_check`, sends it as a raw datapoint, tracks the exact request,
  validates the DP71 response for the active transaction, and then waits for
  DP47 to confirm physical unlock.
- BLE notifications are received through GATT notify, decoded by
  `_notification_handler`, parsed as Tuya BLE receive messages such as
  `FUN_RECEIVE_DP`, `FUN_RECEIVE_SIGN_DP`, `FUN_RECEIVE_TIME_DP`, and V4
  variants, then surfaced through datapoint callbacks into Home Assistant's
  coordinator.
- For YR05, the inspected BLE HA mappings expose battery DP8, lock state/control
  DP47/DP46/DP71, passage mode DP101, and a manual refresh button.

Important comparison:

- The BLE parser is generic enough to store any decoded datapoint that the lock
  sends, so BLE logs may show additional raw physical-event DPs even when no HA
  entity presents them.
- The inspected BLE HA mappings do not currently expose YR05 DP12, DP13, or
  DP19 as detailed fingerprint/PIN/BLE credential-event entities.
- Therefore, the known-working BLE behavior visible in code is primarily
  DP47-driven physical lock-state activity plus DP71 transaction handling, not
  confirmed detailed credential/user-record presentation.
- The SigMesh gateway path currently depends on TinyTuya LAN child-device
  decoding. It does not have an equivalent BLE-notification parser for raw
  Tuya BLE receive frames.

Hypothesis to test:

- physical credential events may be available locally only as BLE notification
  datapoints, as gateway parent-level messages, or after a gateway-specific
  subscription, rather than through ordinary child `status()` polling.

## Relevant Existing YAML Profiles

These profiles are useful references, but none should be copied blindly.

### Hornbill Y4 Smart Lock

File:

- `custom_components/tuya_local/devices/hornbill_y4_smart_lock.yaml`

Relevant similarities:

- DP8 battery
- DP12 fingerprint unlock
- DP13 password unlock
- DP19 BLE unlock
- DP46 lock-like command
- DP47 mapped lock state where `false` means locked and `true` means unlocked
- DP28 language
- DP31 volume
- DP33 automatic lock
- DP36 auto-lock delay
- DP71 `ble_unlock_check`

Important difference:

- It includes many DPs not confirmed for YR05.

### Raykube A1 Pro Max Lock

File:

- `custom_components/tuya_local/devices/raykube_a1promax_lock.yaml`

Relevant similarities:

- DP46 lock-like command
- DP71 `ble_unlock_check`
- Raykube-style secure lock structure

Important difference:

- DP47 is modeled as `open`, not physical lock state.
- YR05 DP47 is confirmed as physical lock state.

### Primebras Athenas Lock

File:

- `custom_components/tuya_local/devices/primebras_athenas_lock.yaml`

Relevant similarities:

- DP46 lock-like command
- DP47 mapped lock state
- DP70/DP71 Base64 sensitive values
- DP8 battery

Important difference:

- It has extra DPs not confirmed for YR05.

### Other BLE Lock Profiles

Several BLE lock profiles contain `ble_unlock_check`, including:

- `ble_positivo_smart_fechadura.yaml`
- `bstuokey_invisible_lock.yaml`
- `gainsboroughliberty_entrance_lock.yaml`
- `lucking_hs6_lock.yaml`
- `nice_digi_lock.yaml`
- `orion_dl033ha_lock.yaml`
- `xcase_nx4964_lockbox.yaml`

Because existing profiles already use `ble_unlock_check` passively, do not make
that DP name alone trigger active local unlock behavior.

## Current YR05 DP Names

Current branch behavior uses the existing lock DP name for DP46 and an
explicit new opt-in DP name for DP71:

- `lock`
- `authenticated_ble_unlock`

Current semantics:

- `lock`: for YR05 this maps to DP46 and writes `true` to lock. DP71
  authenticated unlock takes precedence in `async_unlock`, so YR05 unlock does
  not write `DP46=false`.
- `authenticated_ble_unlock`: active unlock DP. For YR05 this maps to DP71 and
  writes the computed authenticated Base64 request.

Recommended source material name:

- `ble_unlock_check`

Recommended handling:

- source material only
- hidden
- sensitive
- optional
- not exposed as an entity attribute
- not sufficient by itself to enable unlock behavior

## Recommended YR05 YAML Shape

The initial milestone profile should be conservative and only include the DPs
needed for gateway discovery, battery, lock/unlock, and physical state.

Initial milestone profile structure:

- top-level `name: Door lock`
- lock entity:
  - DP46 `lock`
  - DP47 `lock_state`, mapped so `false` means locked and `true` means unlocked
  - DP71 `authenticated_ble_unlock`, optional, sensitive
- battery sensor:
  - DP8 battery percentage

Deferred profile additions after the initial milestone works reliably:

- DP12 `unlock_fingerprint`, optional, non-persistent
- DP13 `unlock_password`, optional, non-persistent
- DP19 `unlock_ble`, optional, non-persistent
- DP28 language, only with confirmed enum values
- DP31 beep volume, only with confirmed enum values or numeric range
- DP33 automatic lock enable
- DP36 auto-lock delay, only with confirmed range/unit/scale

Do not invent enum values for DP28 or DP31. Existing profiles often use values
such as `mute`, `low`, `normal`, `high`, `english`, and
`chinese_simplified`, but those should not be assumed for YR05 unless confirmed
by the device data model or logs.

Do not invent ranges or units for DP36. Use Tuya data model evidence if
available.

The known gateway CID `cd72a26b41cc2f02` is not a product ID and should not be
used in the YAML `products:` section. The current profile uses confirmed
product ID `hhxgpozj` with model metadata derived from the confirmed H13
new-chip lock identity.

## `ble_unlock_check` Supply and Storage

Do not hardcode or commit real `ble_unlock_check` values.

Recommended approach:

- Use a secret-like config entry/option for `ble_unlock_check`.
- Store it in Home Assistant config entry data/options similarly to `local_key`.
- Redact it from diagnostics.
- Let the lock entity read it from the runtime device/config.

Reasoning:

- The value is authentication material.
- It should not live in a YAML profile.
- It should not appear in logs, diagnostics, tests, or fixtures except as
  synthetic dummy data.
- A config option lets users supply the value without modifying source files.
- Repeated gateway child status queries did not report DP71, so the initial
  gateway implementation must not rely on cached/reported DP71 as the source.

Implemented config-entry storage touches:

- `custom_components/tuya_local/const.py`
- `custom_components/tuya_local/config_flow.py`
- `custom_components/tuya_local/device.py`
- `custom_components/tuya_local/diagnostics.py`

Keep the UI and storage change narrowly scoped to lock profiles that opt in to
authenticated BLE unlock.

## Implemented Minimal Python Shape

Current branch behavior:

1. `TuyaLocalLock.__init__` consumes `authenticated_ble_unlock` as an explicit
   active unlock DP.
2. DP46 continues to use the existing lock DP name `lock`.
3. `async_lock()` remains on the existing writable `lock` path and writes
   `true`.
4. `async_unlock()` checks `authenticated_ble_unlock` before the existing
   writable `lock=false` branch, so the YR05 profile does not unlock by
   writing `DP46=false`.
5. `build_ble_unlock_msg` builds the DP71 Base64 request from configured
   `ble_unlock_check`.
6. Existing secure-code paths are unchanged.
7. Existing passive `ble_unlock_check` profiles do not opt in to active unlock
   behavior unless they declare `authenticated_ble_unlock`.

## Tests To Add or Update

### Lock Tests

Relevant file:

- `tests/test_lock.py`

Add tests for:

- DP71 builder produces the exact expected Base64 payload from synthetic
  non-secret 19-byte source data and a fixed timestamp
- invalid Base64 source is rejected
- decoded source length other than 19 bytes is rejected
- `async_unlock()` writes expected DP71 value when authenticated unlock is
  configured
- `async_unlock()` sends nothing useful and raises/fails cleanly when source
  material is missing
- `async_lock()` with YR05 `lock` DP writes `DP46=true`
- `async_unlock()` does not write `DP46=false`

Defer DP12, DP13, and DP19 event presentation tests until after the initial
milestone works reliably.

### Device Config Tests

Relevant file:

- `tests/test_device_config.py`

Update known lock DPs to include:

- `authenticated_ble_unlock`
- `ble_unlock_check`, if not already accepted for the relevant entity path

The new YAML profile should pass device config parsing.

### Gateway/Subdevice Tests

Relevant file:

- `tests/test_device.py`

Existing tests cover subdevice unique IDs and receive behavior. Add or extend a
focused test to assert:

- parent TinyTuya device is created for the gateway
- child TinyTuya device is created with `cid=dev_cid`
- child receives `parent=parent`
- protocol version is applied to parent and child

Do not broaden this unless implementation changes require it.

### Config Flow and Diagnostics Tests

Only needed if adding config entry storage for `ble_unlock_check`.

Relevant files:

- `tests/test_config_flow.py`
- `tests/test_diagnostics.py`

Add tests that:

- the optional value can be stored/updated
- the value is redacted from diagnostics
- connection testing does not require the value unless unlocking is attempted

## Validation Commands After Implementation

For YAML-only or YAML-focused changes:

```sh
uv run pytest tests/test_device_config.py
uv run duplicates <CONFIG_FILENAME>
```

For lock/authentication Python changes:

```sh
uv run pytest tests/test_lock.py
```

For gateway/config-flow/diagnostics changes:

```sh
uv run pytest tests/test_device.py tests/test_config_flow.py tests/test_diagnostics.py
```

Before committing or opening a PR, follow `AGENTS.md` and run:

```sh
uv run pytest
uv run ruff check .
uv run ruff check --select -I .
uv run ruff format --check .
uv run yamllint custom_components/tuya_local/devices
```

## Constraints and Guardrails

- Do not modify production Tuya-BLE repositories.
- Do not expose or hardcode local keys.
- Do not expose or hardcode `ble_unlock_check` values.
- Do not commit secrets.
- Do not invent DPs.
- Do not invent DP71 authentication beyond the confirmed 19-byte layout.
- Prefer the smallest upstream-friendly change.
- Reuse existing lock and device abstractions where they fit.
- Keep behavior for existing `ble_unlock_check` profiles unchanged unless they
  explicitly opt in to authenticated BLE unlock.
- Clearly distinguish confirmed laboratory facts from inferences in future PR
  text.

## Open Questions Before Productionizing

- What are the confirmed DP28 language enum values?
- What are the confirmed DP31 beep-volume enum values or numeric range?
- What is the confirmed DP36 range, unit, and scale?
- Does the cloud config flow's indirect-device local-key behavior match all
  YR05 gateway setups, or only the tested one?
- Is real-time physical credential/event passthrough available from the
  SigMesh gateway, and if so, what message shape or subscription enables it?

## Current Recommended Next Step

The initial gateway-control milestone is validated. Do not add deferred DPs to
the YR05 profile until the physical-event path is better understood.

Recommended next research step:

1. Capture redacted TinyTuya LAN receive output from both the child object and
   the parent gateway object during physical fingerprint, PIN, manual lock, and
   manual unlock operations.
2. Record only message shape, command names/codes if available, DP IDs, and
   value lengths/types. Do not record local keys, `ble_unlock_check`, full raw
   DP71 payloads, or credential values.
3. Compare those captures with the BLE reference's decoded datapoint callbacks,
   especially whether physical activity produces DP47 only or also DP12, DP13,
   DP19, or another gateway event envelope.
4. Test whether `updatedps()` with explicit DP12, DP13, DP19, and DP47 IDs
   changes what the child reports after a physical operation.
5. If the child remains silent, test whether the parent gateway receive socket
   emits a child-event envelope and whether TinyTuya exposes enough raw data to
   decode it safely.
