# Yamiry gateway development handoff

Last reviewed: 2026-09-29 (America/Chicago).
Repository: tuya-local fork for Yamiry YR05/YR02.
This document is the durable continuity record. Read it with the root AGENTS.md
before continuing. Do not assume earlier chat history is available.

## Maintenance and evidence rules

Update this document whenever a meaningful implementation decision, test result,
commit, merge, release, deployment, or change of plan occurs. Before ending every
substantial session, verify it against the actual branch, HEAD, working tree, and
available release/deployment evidence. Preserve useful history; mark superseded
decisions rather than silently replacing them.

Evidence labels used here:

- **Confirmed:** inspected repository contents, refs, diffs, or tool results.
- **Tested:** an executed software test, with its scope stated.
- **User-reported hardware result:** supplied by the operator, not independently
  reproduced by this agent.
- **Hypothesis:** a plausible explanation not demonstrated on hardware.
- **Deferred/proposed:** not implemented or not authorized yet.

Never record local keys, DP71 authentication material, ble_unlock_check values,
tokens, passwords, or other credentials here. Use names/purposes only. Use
obviously synthetic data in tests and sanitize logs/diagnostics before sharing.
Do not copy HA configuration entries or raw authenticated unlock payloads here.

## Current objective and status

**Objective:** upgrade the released custom integration to the appropriate stable
September upstream release while preserving Yamiry functionality and the
hardware-tested gateway synchronization.

**Confirmed current state during the authorized September upgrade:**

- Checked-out branch: upgrade/2026.9.2-rel.
- HEAD remains f4985b58ab26ba0fa55a50db64060e835987d77a (documentation commit).
- Known-good released code/tag remain 3d1388a8 / v2026.8.1-yr05.6.
- A no-commit merge of the exact annotated 2026.9.2-rel tag is in progress.
  MERGE_HEAD is tag object 2f00da8b95912c08e51c4466c4a54761cfa0723e;
  its peeled commit is f2b3d490e5e9ba87cf75568c36983d62c258c544.
- Both textual conflicts are resolved and staged. No merge commit exists.
- feature/yr05-gateway-lock already pointed to f4985b58 when inspected in this
  session; its only change from 3d1388a8 is this documentation. It was not moved.
- Implementation and local validation are authorized; commit, push, tag,
  release, deployment, and rewriting released history remain prohibited.
- Current validation and resolution details are in the implementation record below.

## Remotes, branches, and exact anchors

Confirmed remotes (credential-free URLs):

- origin: https://github.com/Philux-X/tuya-local.git
- upstream: https://github.com/make-all/tuya-local.git

Historical branch snapshot before the user created the upgrade branch (see current state above):

| Branch/ref | Commit |
| --- | --- |
| feature/yr05-gateway-lock | 3d1388a865e3281a32f4f2050ea9ba2e7fc5a690 |
| fix/shared-gateway-synchronization | 3d1388a865e3281a32f4f2050ea9ba2e7fc5a690 |
| main (local; not upstream/main) | 8d2c0995293c94bdcb6dfb64cf385c5b51a852da |
| upstream/main (last fetched state) | 66731a6b4267b5268eee05fae4bbdb8679518713 |

Do not confuse local main with upstream/main. Recheck all refs before acting;
these are a dated snapshot, not a claim about current remote server contents.

| Anchor | Commit and role |
| --- | --- |
| v2026.8.1-yr05.6 | 3d1388a865e3281a32f4f2050ea9ba2e7fc5a690; preferred known-good rollback baseline |
| Pre-synchronization custom commit | 409763cf8a5ed9352c87cc714d003cbd1d068eb5; historical baseline, not the preferred runtime rollback |
| Actual common ancestor with September target | ebd4ddcf99edb7d6dd59c23af321005cbe267e87 |
| 2026.9.0 | 053eaef1bec375bc144aab2cb0c18bb6e6945cd7 |
| 2026.9.1 | 4551357adb34b6cf3073af8deada6a9bf13c94f0 |
| 2026.9.2 | 23ee9cd405969c340b7017eadafbf309cfbdad01 |
| Recommended target: 2026.9.2-rel | f2b3d490e5e9ba87cf75568c36983d62c258c544 |

The fork is described as 2026.8.1-based and its manifest has that version, but
its actual common ancestor predates the final upstream 2026.8.1 tag. Do not
assume every change in the final August tag is already present. Missing examples
include media-player support, the newer density enum, and receive-error handling.

## Release history and Home Assistant baseline

| Release | Confirmed local anchor | Evidence/status |
| --- | --- | --- |
| v2026.8.1-yr05.1 | f7bfd50a219bb1254c9e64af6ceeaa76c2c66c76 | Local tag exists; detailed deployment/test history not established here |
| v2026.8.1-yr05.2-debug | af6986d008d30740a7926719853e2a5da491cfba | Local debug tag exists; do not select as normal rollback |
| v2026.8.1-yr05.3 | 5ee575047f4021a611a3bd3d55542bd0fb541141 | Local tag exists; detailed deployment/test history not established here |
| .4 / .5 | Not established | No corresponding local tags found; do not invent commit mappings or release results |
| v2026.8.1-yr05.6 | 3d1388a865e3281a32f4f2050ea9ba2e7fc5a690 | User reports hardware-tested and released, including synchronization fix |

**User-reported hardware result:** .6 is the known-good tested release for YR05
and YR02 sharing one SigMesh gateway. The user explicitly states the
synchronization fix in 3d1388a8 has been hardware tested.

**Last known HA baseline:** HA was upgraded to 2026.9.x. Treat .6 as the last
reported tested integration baseline, but do not claim the current installed
files were independently inspected. Exact HA patch version, installation time,
deployment checksums, and detailed hardware test logs were not provided.

The confirmed hardware result is overall success as reported by the user.
Individual unlock, Passage Mode, reconnect, pause/reload, and forced-shutdown
test results were not itemized. Do not fabricate a per-scenario pass record.

Rollback: retain .6 and an operator-created HA/configuration backup before any
future deployment. The proposed September migration advances entry minor
version to 24; do not assume replacing code alone reverses registry migrations.
No rollback or backup operation has been executed in this session.

## Hardware and communication architecture

Operator-supplied topology (identifiers below are routing identifiers, not keys):

- One Tuya SigMesh gateway at 10.10.4.140.
- Gateway device ID: eb82e66f33699c154b2osy.
- YR05 CID: cd72a26b41cc2f02.
- YR02 CID: bc7230f1597ef04f.
- Both subdevices use Tuya LAN protocol 3.5.
- HA ping and TCP port 6668 connectivity were confirmed by the operator during
  the original failure investigation.

Both child objects must reuse the same cached TinyTuya parent and shared
asyncio lock. Child requests route through that parent's LAN socket using their
distinct CIDs. Preserve gateway-scoped unique IDs and do not replace a CID with
the gateway device ID. Do not create one independent runtime LAN session per
lock as a workaround.

Parent reuse already existed upstream before the fork's Yamiry work; it is not
a wholly custom architecture. Each child still has its own receive loop.
TinyTuya's actual socket owner is the parent for a subdevice.

## Confirmed device semantics and custom implementation

Profile: custom_components/tuya_local/devices/yamiry_yr05_lock.yaml.

- Product hhxgpozj: Yamiry YR05.
- Product 6xjvratw: Yamiry YR02.
- DP47 is physical lock state: raw false = locked; raw true = unlocked.
  YAML deliberately reverses the raw boolean into HA's is_locked meaning.
  Preserve the mapping and lock_state precedence.
- DP46 remains the profile's optional lock DP. Do not substitute it for the
  confirmed DP47 physical-state semantics.
- DP71 is optional, nonpersistent, sensitive authenticated BLE unlock data.
- DP101 is an optional boolean switch named Passage mode. It is independent
  of the physical lock-state sensor; do not equate enabling it with a DP47 event.
- DP8 is battery percentage, range 0..100.
- DPs 12, 13, 19 are optional nonpersistent unlock-event values.
- DP28 controls language; chinese_simplified maps to the displayed chinese.
- DP31 controls volume: mute, low, normal, high.
- DP33 controls Automatic lock.
- DP36 controls Automatic lock delay, 5..60 seconds in steps of one.

### DP71 design, without authentication material

lock.py detects authenticated_ble_unlock and prioritizes that unlock path.
It reads the configured ble_unlock_check source, validates base64 and a
19-byte decoded length, constructs the device-specific authenticated message
from source fields plus a fresh timestamp/fixed fields, base64-encodes it, and
sends it through DP71. Preserve the existing field ordering and tests; do not
replace it with an ordinary boolean unlock or generic code-unlock algorithm.

ble_unlock_check is required by the custom unlock path and must stay wired through:

- const.py configuration key;
- conditional password-style setup/options selectors in config_flow.py;
- merged entry data/options;
- setup_device constructor argument and device property;
- authenticated unlock logic;
- diagnostics and logging redaction.

Preserve explicit ble_unlock_check diagnostic redaction and sensitive-DP
redaction for cached/pending/received data. The current runtime log redactor
derives sensitive IDs from registered entity configurations; do not assume
every raw TinyTuya/pre-registration debug message is automatically sanitized.

## Synchronization design in released 3d1388a8

The shared parent lock serializes whole communication transactions:

- Initial refresh: protocol configuration, executor I/O, retries, retry cleanup.
- Commands: pending selection, network send, retries, cleanup.
- Receive and heartbeat: both remain protected.
- Parent persistence changes, stop, pause, receive-loop cleanup, and failed
  setup cleanup acquire or already hold the same shared lock.
- pause() is async; config flow awaits it before its existing five-second delay.
- Failed setup cleanup is async and awaited.
- Receive data is yielded only after releasing the lock.
- async_stop releases the lock before awaiting the receive task.
- _retry_on_failed_connection does NOT acquire the lock again; callers own it.
- _async_api_job shields executor work from caller cancellation and waits for
  it to finish before propagating cancellation.
- Socket recovery/backoff checks self._api.parent or self._api, not the normally
  socketless child.

Why upstream 2026.9.2-rel must not win wholesale:

- Its retry helper locks only individual executor calls.
- Its direct receive/heartbeat calls bypass that lock.
- Protocol changes and retry cleanup occur outside that narrow lock.
- Pause, stop, and other cleanup can close the parent without locking.
- Its socket recovery checks the child rather than the parent.
- It lacks the fork's cancellation protection and custom secret redaction.

CRITICAL: retaining our outer locks while accepting upstream's inner retry
lock creates nested acquisition of a non-reentrant asyncio.Lock and can deadlock.

## Original failure: evidence versus hypothesis

Operator reported both locks working before the HA upgrade, then setup errors
901/902, an earlier 914, null refreshes, and ConfigEntryNotReady/offline afterward,
despite reachable gateway IP/TCP.

**Confirmed code defect:** the former initial refresh bypassed the shared lock,
and commands used a separate per-device lock. Retry/failure closure could
interfere with sibling traffic.

**Hypothesis:** simultaneous startup or changed timing exposed that race.
No specific HA 2026.9 API/lifecycle regression was demonstrated. Hardware success
with .6 supports the repair, but does not prove every earlier failure had one cause.
TCP reachability does not prove a successful Tuya authenticated session.

## Upstream decisions already evaluated

Statuses describe released code or the proposed upgrade; distinguish them.

| Commit(s) | Decision |
| --- | --- |
| cdd60504 | Historical shared-parent locking already in base; preserve parent reuse |
| 29b7d1e8, 3db282d4 | Adapted in 3d1388a8: shared refresh/send lock, context-managed receive lock, remove per-device threading lock |
| 899a4eb8 | Recovery concept adapted in 3d1388a8 using actual socket owner |
| 42aca525 | Parent-socket correction adapted in 3d1388a8; absent from 2026.9.2-rel; no duplicate cherry-pick needed |
| b121da23 / c504d683 | Earlier temporary-poll recovery experiment was reverted upstream; do not resurrect it |
| c4f67949 | Released fork independently yields outside lock; do not replace its protected I/O with upstream's later design |
| 62597468 | Reject as standalone: invalid async-with placement in synchronous functions |
| f2b3d490 | Target release commit, but reject its narrower transport-lock implementation in favor of our tested design |
| ca2c3dc6 | Deferred transport correction: receive-error disconnect/backoff for 901/902/905/906/914; not required for September API compatibility |
| ba2a71f0 | Proposed adoption: migration 13.24, matching translations/icons/profile changes, config-flow minor version 24 |
| 053eaef1, 70540a60 | Proposed adoption: improved sensitive entity diagnostics redaction and copying of state dictionaries |
| 03346816, 172f606a | Proposed adoption together: diagnostics registry changes and follow-up lookup fix |
| f7d88ce7 | Proposed adoption: UnitOfDensity/value changes with HA minimum 2026.8 |
| 38347807 | Proposed adoption: close configuration-directory scandir iterator, retain its test |
| 05abfe91 | Proposed adoption: normalize reversed color-temperature limits |
| c60a39e8 | Proposed adoption: remove SmartIR from cloud hub choices; not SigMesh |
| 934171ac | Proposed adoption: stop fetching/logging partial cloud specifications |
| b10ef2e6 | Included in -rel: LEDVANCE advanced-control DP made optional |
| e5ebd64f + 0131de6b | Deferred unrelated post-release Daybetter product move; these form a pair |
| fba0e01c | Deferred optional development test pin .365 -> .366; not runtime |
| 66731a6b | Deferred unrelated WDYK leakage-current translation |
| aa513d7a, 986d71e7 | Optional post-release cleanup only; no need to import main |

There are THREE commits strictly after e5ebd64f through 66731a6b, or FOUR when
including e5ebd64f. Main is not recommended merely because it is newer.
Other post-release changes include deprecated-entity removal; exclude unrelated
main changes from this pinned-release upgrade.

## Tests and results

Released test file: tests/test_gateway_synchronization.py (338 lines at review).
It contains nine parameterized cases using a real asyncio lock, observed
acquisition/ownership, explicit barriers, and genuine executor threads.
HA's fixture otherwise runs some mocked callables inline, so the tests explicitly
submit every gateway callable through loop.run_in_executor.

- Refresh, command, failed-setup cleanup, pause (four cases): sibling A is inside
  blocked network I/O; B is confirmed attempting the real lock; B cannot enter
  or close the parent until A finishes. Closure/sibling operation checks ownership.
- Retry cleanup: actual callback executes exactly once on the retry-owning task,
  while that task holds the lock; sibling status has not run.
- Heartbeat, receive, reconnect (three cases): each selected network operation
  checks ownership and excludes sibling I/O; parent socket state selects receive
  versus full poll; yield does not retain the lock.
- Cancellation: cancel the coroutine during real thread I/O, observe the drain
  path, prove the sibling stays blocked until thread completion and cancellation
  propagation.

**Tested in the prior implementation session:**

- Nine gateway cases passed.
- 119 existing device/config-flow/lock tests passed.
- Ruff lint, import ordering, formatting on six changed Python files passed.
- git diff --check passed.
- An earlier run also passed 123 existing device/config-flow/lock/diagnostics
  tests before the final pause/test refinements; do not conflate those scopes.
- No full-suite or September-upgrade test run has been performed.
- No tests were rerun merely to create this document.

Environment: Windows PowerShell workspace, WSL used for uv/Python tests.
Python >=3.14.2. A temporary /tmp/tuya-gateway-tests environment was used because
the old workspace environment had missing packages. Temporary environments may
not survive sessions. Follow AGENTS.md and use uv run. Do not assume an ignored
local uv.lock has been refreshed for a changed pyproject.toml.

## Deferred issues and limits

- Concurrent parent creation is still a non-atomic lookup/create sequence.
- Config-flow Test devices intentionally use a separate parent/session; these
  can contend at the physical gateway despite separate Python objects.
- Null initial responses can be treated as successful retry-helper returns
  without useful cached state. No retry-policy redesign performed.
- Receive-error classification/disconnect behavior differs from upstream;
  preserve the tested baseline initially, evaluate ca2c3dc6 separately.
- A stuck executor operation can delay siblings and shutdown.
- Forced HA shutdown can independently cancel an executor Future. A cancelled
  Future is not proof its thread stopped; the current helper protects normal
  caller cancellation, not that separate forced-shutdown case.
- Migration rollback and exact live HA installed state require operator evidence.
- Hardware-tested does not mean every shutdown/recovery scenario has been tested.

## September upgrade inventory and strategy

Recommended target: 2026.9.2-rel at f2b3d490e5e9ba87cf75568c36983d62c258c544.
Published release: https://github.com/make-all/tuya-local/releases/tag/2026.9.2-rel

The -rel tag differs from 2026.9.2 in device.py's three locking revisions and
the LEDVANCE optional-DP correction. Select it for published release identity,
not because its gateway synchronization is stronger.

From common ancestor ebd4ddcf:

- 200 upstream-only paths (180 device YAML paths plus 20 other files).
- Seven custom-only paths: const.py, devices/yamiry_yr05_lock.yaml, lock.py,
  tests/test_config_flow.py, tests/test_device.py,
  tests/test_gateway_synchronization.py, tests/test_lock.py.
  Runtime paths above are under custom_components/tuya_local.
- 29 paths changed on both sides: __init__.py, config_flow.py, device.py,
  diagnostics.py; tests/test_device_config.py, tests/test_diagnostics.py;
  all 23 translations (bg, ca, cs, de, el, en, es, fr, hu, id, it, ja, no-NB,
  pl, pt-BR, pt-PT, ro, ru, sv, uk, ur, zh-Hans, zh-Hant).
- Read-only legacy three-way merge preview found textual conflicts in device.py
  (five regions) and tests/test_diagnostics.py (one). This is a preview, not a
  performed merge or a guarantee of the eventual merge engine's exact output.
- Clean textual merges still require semantic review, especially diagnostics,
  migrations, config flow, tests, and all BLE translation labels.
- device_config_schema.json is unchanged. Test-schema additions include float
  conditions and media_player; retain authenticated_ble_unlock in KNOWN_DPS.

Manifest runtime requirements stay tinytuya==1.20.0 and
tuya-device-sharing-sdk~=0.2.4. Target manifest version is 2026.9.2.
Development HA test pin changes .355 -> .365; HA minimum changes 2026.6 -> 2026.8.
Python minimum remains >=3.14.2. The new release's migration is version 13,
minor 24, renaming inlet/outlet temperature entities; it does not target Yamiry.

**Proposed Git strategy:** create a new upgrade branch FROM released 3d1388a8,
then merge the exact tag with --no-commit and explicit conflict review. Preserve
both ancestries. Do not rebase published history or reconstruct the fork from
upstream by selectively guessing which custom commits matter.

Initial upgrade should preserve device.py's tested transport behavior and custom
auth/redaction. Adopt upstream diagnostics improvements while retaining
ble_unlock_check redaction. Bring migration 13.24, minor version, translations,
icons, profiles, and relevant tests together. Consider additional transport
corrections only as separate reviewed changes after the baseline upgrade.

## Original implementation checklist (historical; see current next steps below)

1. Read this document and AGENTS.md. Check git status, current branch, HEAD,
   release tag targets, and remotes without exposing credentials.
2. Preserve this handoff document and any user changes. Reconfirm the actual
   working tree rather than assuming it is clean after documentation creation.
3. If no new upgrade authorization exists, stop after read-only review/report.
4. Once authorized, create the agreed upgrade branch from 3d1388a8, then begin a
   no-commit merge of pinned 2026.9.2-rel (not moving upstream/main).
5. Resolve device.py around the released implementation. Audit every lock and
   closure, retain parent-aware recovery, async pause, awaited cleanup,
   cancellation helper, and all secret redaction.
6. Combine both diagnostics test groups and review the updated diagnostics API.
7. Review all 29 overlapping paths even if Git reports no conflict. Confirm the
   Yamiry profile and lock.py custom behavior remain intact.
8. Adopt the coherent upstream migration/dependency/profile/test changes.
9. Refresh the test environment for the target development pin; use uv.
10. Run focused gateway/device/config-flow/lock/diagnostics/profile/translation/
    entity/media-player tests. Add a minor-23 -> 24 Yamiry migration test that
    preserves auth options, configuration, and entity identities.
11. Run full pytest and required repository checks (ruff check, import checks,
    format --check, yamllint devices, git diff --check). Follow AGENTS.md; do not
    hide failures or change device semantics to make unrelated tests pass.
12. Review final diffs against BOTH parents. Report expected versus actual
    transport changes and update this document with results.
13. Stop before commit/release/deployment unless explicitly authorized.
14. With separate operator approval, back up HA, repeat two-lock hardware tests,
    record exact deployed revision/version and per-scenario results, and only
    then consider a new release.

## Review gates: MUST NOT do without review

- Do not merge main merely because it is newer; use the pinned reviewed target.
- Do not replace the integration directory or accept upstream device.py wholesale.
- Do not combine outer transaction locks with upstream's inner retry lock.
- Do not remove await from pause/failed cleanup or await a receive task while
  holding the shared gateway lock.
- Do not allow receive/heartbeat/commands or shared-parent closure to bypass it.
- Do not change DP47 polarity, DP71 payload design, DP101 meaning, CID routing,
  protocol 3.5, battery/configuration entities, or authentication/redaction.
- Do not print or store credentials, real auth payloads, or raw secret-bearing logs.
- Do not rewrite released history, retag .6, or select an unverified .5 rollback.
- Do not deploy, unlock hardware, change Passage Mode, or perform destructive
  hardware tests without operator authorization.
- Do not silently include deferred architecture, retry, or forced-shutdown work.
- Do not equate a passing mock test with a real-thread concurrency test or with
  hardware validation.
- Do not treat a plan in this document as user approval for Git/deployment actions.

## Continuity log

- 2026-09-28: investigated two-lock setup failures. Identified missing startup/
  command serialization and distinguished parent-socket recovery from initial
  setup. No HA 2026.9-specific setup API requirement established.
- 2026-09-28: implemented focused synchronization and recovery adaptation;
  review identified unlocked pause and weak scheduling-based tests.
- 2026-09-28: made pause awaited/locked and strengthened nine tests with ownership
  barriers and real executor threads; 9 focused + 119 existing tests passed.
- 2026-09-29: user reported hardware-tested release .6 at 3d1388a8. Repository
  refs confirm feature and repair branches point to that commit.
- 2026-09-29: read-only September upgrade review recommended pinned -rel merge,
  preserving released transport; upgrade remains unimplemented.
- 2026-09-29: created this handoff document after verifying refs, remotes, clean
  starting state, release anchors, and test definitions. No implementation or
  deployment changes made in this documentation session.
- 2026-09-29: reverified branch, HEAD, .6 tag, September target, upstream/main, and common ancestor before the proposed upgrade; reviewed handoff completeness and credential exclusion. Only this untracked document changed; no merge begun.

## Authorized September merge implementation record (2026-09-29)

Merge: `git merge --no-commit --no-ff 2026.9.2-rel` on the existing upgrade
branch, starting from clean f4985b58. The initial sandbox denied Git metadata
writes; the authorized merge was retried with filesystem approval.

Conflicts resolved:

- device.py: retained the complete released implementation byte-for-byte in Git.
  This intentionally overrides upstream's inner retry lock, unprotected direct
  receive/heartbeat/cleanup, child socket checks, and ca2c3dc6 receive-error
  classification/backoff. No new transport policy was needed for this merge.
- tests/test_diagnostics.py: retained our BLE configuration redaction test AND
  upstream's four sensitive attribute/state redaction tests; formatted the union.

All 29 overlapping files were reviewed: __init__.py, config_flow.py, device.py,
diagnostics.py, test_device_config.py, test_diagnostics.py and all 23 translation
files. Structural comparison of every translation leaf confirmed all upstream
content and our custom BLE setup/options labels survive. Automatic Python merges
were compared against both parents: migration 13.24 coexists with awaited failed
cleanup; config-flow minor 24 retains awaited pause and BLE options; diagnostics
retains explicit BLE source redaction with upstream's state/attribute changes;
profile validation retains authenticated_ble_unlock and Yamiry-specific tests.

All custom-only files were verified against 3d1388a8. const.py, lock.py, the
Yamiry profile, test_device.py, test_lock.py, and test_gateway_synchronization.py
are unchanged. test_config_flow.py has only the new migration test described
below. device.py is also unchanged from 3d1388a8. Thus DP47/DP71/DP101, battery,
entities, parent reuse, CID, protocol 3.5, outer shared locks, executor draining,
parent-aware recovery, pause/stop/receive and failed-setup cleanup are retained.
No inner retry-lock acquisition was introduced.

Adopted release content includes migration 13.24 and matching translations/icons/
profiles, diagnostics improvements, UnitOfDensity, media-player support/tests,
scandir closure, color-temperature limit normalization, cloud hub/logging changes,
September profiles, manifest 2026.9.2, HA minimum 2026.8, and development test pin
0.13.365. Runtime TinyTuya and sharing-SDK requirements are unchanged. Historical
"proposed adoption" rows above now describe adopted content for this uncommitted
merge, except the explicitly rejected/deferred transport and post-release changes.

Added test: test_yamiry_migration_13_23_preserves_configuration_and_entities,
parameterized for two synthetic sibling CIDs. It runs production migration with
the real profile and HA registry and checks version 13/minor 24, unchanged entry
identity, all profile entity unique IDs/entity IDs, protocol/CID/auth configuration,
and option overrides. All new authentication fixtures are explicitly fake strings.

Validation so far (WSL, Python 3.14.7, uv environment /tmp/tuya-gateway-tests):

- `uv run pytest tests/test_gateway_synchronization.py tests/test_device.py tests/test_config_flow.py tests/test_lock.py tests/test_diagnostics.py -q`: 136 passed.
- `uv run ruff format tests/test_config_flow.py tests/test_diagnostics.py`: one file formatted.
- `uv run ruff check .`: passed.
- `uv run ruff check --select I .`: passed.
- `uv run ruff format --check .`: passed, 112 files.
- `uv run yamllint custom_components/tuya_local/devices`: exit 0; upstream marpou_ceiling_lamp_ledlight.yaml line 3 has a comment indentation warning.
- `uv run pytest --cov=custom_components/tuya_local --cov-report=term:skip-covered -q`: 458 passed in 617.52 seconds; 71% aggregate coverage. Includes profile/schema and translation tests.
- `uv run untranslated_entities`: exits 0 but emits an error for the pre-existing Yamiry select.volume translation key. Verified the released 3d1388a8 profile and English translation already have this mismatch. Deferred rather than changing profile/entity naming in this upgrade.
- `git diff --check`: unstaged changes passed at initial review.
- `git diff --cached --check`: inherited upstream CRLF lines are reported as trailing whitespace, plus ACKNOWLEDGEMENTS.md's upstream blank EOF line. With `-c core.whitespace=cr-at-eol`, only that EOF blank remains. No unrelated release files were rewritten to hide these findings.
- No unresolved index entries or merge markers found in runtime/tests/docs.

Current next steps:

1. Local validation is complete. Review the merge and the inherited validation findings below; do not restart or auto-commit the merge.
2. Human review the staged merge plus handoff update. Do not restart the merge.
3. Commit only with explicit authorization. No commit/push/release/deploy permitted
   by the implementation request.
4. Hardware validation of this merged tree has NOT occurred. Retain .6 as the
   known-good baseline; after separate approval/backups test both locks together,
   restart/reload/reconnect, unlock/state/Passage Mode/battery/options/redaction.
5. Deferred architecture, null-response policy, separate config-flow sessions,
   ca2c3dc6, and independently cancelled executor Futures at forced HA shutdown
   remain as documented; no new guarantees for those cases are claimed.

Final review: ready for human review before commit, with the inherited findings
above explicitly disclosed. Compared final tree against f4985b58, 3d1388a8, and
f2b3d490. No unresolved index entries or conflict markers. Credential-related
added lines were reviewed: upstream semantic names, documentation, and synthetic
test fixtures only; no real authentication values or credentials added. A simple
private-key/token-pattern scan found no matches (not a comprehensive security scan).
Final Git state: 229 staged merge paths (71 added, 151 modified, 6 renamed,
1 deleted), plus this one unstaged handoff update. No untracked source files.
HEAD, feature branch and known-good tag are unchanged by this session. Ordinary
unstaged diff checking passes; the full/staged diff has the inherited upstream
whitespace findings described above. No hardware deployment or new release.
