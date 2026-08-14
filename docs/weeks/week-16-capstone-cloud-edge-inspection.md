# Week 16 — Capstone: Cloud–Edge Inspection

[← Week 15: Identity, security, and lifecycle](week-15-identity-security-and-lifecycle.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Full learning plan →](../learning-plan.md)

**Estimated effort:** 12–16 hours

**Lab mode:** simulator first; optional low-speed physical run only after the simulated acceptance suite passes

## Outcomes

By the end of this week you will be able to:

- compose the previous weeks into one bounded inspection system without bypassing component contracts;
- deliver retry-safe mission assignments to `mission_runner`, which composes
  primitive calls through the canonical robot adapter;
- validate and execute a signed, immutable mission bundle while disconnected;
- pin every run to one immutable map release, even when a newer map is published;
- buffer telemetry locally, replay it after reconnection, and deduplicate it in the cloud service;
- keep WebRTC preview and evidence upload independent from motion and mission correctness;
- use mTLS identity and lifecycle state at every robot-facing trust boundary; and
- prove the result with reproducible measurements, an append-only audit trail, and an objective acceptance matrix.

## Prerequisites

Bring forward working outputs from Weeks 1–15:

- a simulator world and, optionally, the low-speed robot accepted in Week 13;
- the Week 9 canonical adapter exposing only `GetState`/`ObserveState`,
  `Halt`/`Stop`, `NavigateTo`, and `CaptureImage`;
- calibrated odometry and sensor transforms;
- a navigation configuration that passed the fixed Week 14 course;
- immutable `MapVersion` geometry and immutable `MapRelease` compositions;
- durable command, receipt, event, and artifact stores;
- the Week 12 WebRTC invitation/token flow and object-store upload flow; and
- the active `robot-lab-01` certificate from Week 15.

You also need Docker with Compose, ROS 2 and Nav2, Python 3.11+, OpenSSL 3.x, `jq`, `sqlite3`, `curl`, `mosquitto-clients`, and `ros2 bag`.

### Physical safety gate

Pass the entire capstone in simulation before placing a robot on the floor. For a physical run:

- retain the Week 13 low speed and acceleration limits;
- use a clear, bounded course with a spotter holding the physical cutoff;
- confirm the local watchdog, local halt, and physical cutoff before every run;
- keep collision monitoring local and independent of Wi-Fi, broker, cloud service, media, and object storage; and
- never inject a cutoff, battery, collision, tip-over, or uncontrolled-motion fault.

If the robot behaves unexpectedly, use the physical cutoff, remove power when safe, and diagnose on blocks. A remote command is not an emergency stop.

## Public readings

1. [ROS 2 actions](https://docs.ros.org/en/rolling/Concepts/Basic/About-Actions.html)
2. [BehaviorTree.CPP tutorials](https://www.behaviortree.dev/docs/tutorial-basics/tutorial_01_first_tree/)
3. [Nav2 behavior trees](https://docs.nav2.org/behavior_trees/index.html)
4. [MQTT 5.0 specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
5. [JSON Canonicalization Scheme, RFC 8785](https://www.rfc-editor.org/rfc/rfc8785)
6. [OpenTelemetry trace context](https://opentelemetry.io/docs/specs/otel/overview/#context-propagation)
7. [WebRTC statistics API](https://www.w3.org/TR/webrtc-stats/)
8. [Amazon S3 presigned URL upload pattern](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)
9. [ROS 2 bag recording and playback](https://docs.ros.org/en/rolling/Tutorials/Intermediate/Recording-And-Playing-Back-Data/Recording-And-Playing-Back-Data.html)

## Concepts

| Concept | Meaning in the capstone |
| --- | --- |
| Canonical adapter | The primitive robot-abstraction boundary; vendor or simulator details stay behind it. |
| Durable command | An intent persisted by the sender and receiver, correlated with a durable receipt, and safe to redeliver. |
| Mission version | Immutable, signed instructions and constraints that can be reused by many runs. |
| Mission run | One assignment of a mission version to one robot, map release, and policy revision. |
| `MapVersion` | Immutable observed geometry bundle containing native and canonical artifacts. |
| `MapRelease` | Immutable approved composition of exactly one `MapVersion` plus alignment, overlays, policies, and routes. |
| Pinned map | The `MapRelease` ID and digest frozen at assignment; a run never follows a mutable channel or alias. |
| Offline authority | Explicitly bounded work the edge may continue after losing connectivity. |
| Local journal | Transactional inbox, run state, node attempts, and outbox that survive process or network failure. |
| Telemetry replay | Ordered resend from the local outbox after reconnect, with receiver-side deduplication. |
| Media session | Ephemeral WebRTC preview with short-lived authorization; it is not evidence or a control channel. |
| Evidence artifact | Durable bytes with a stable artifact ID, digest, capture context, and completion record. |
| Audit event | Append-only record of a security- or lifecycle-relevant decision, without secrets or unrestricted payloads. |

The end-to-end responsibility split is:

```text
operator
  |
  v
cloud API -- durable intent --> command broker -- redelivery --> edge inbox
   |                                                       |
   |                                                       v
   |                                             signed mission bundle
   |                                                       |
   |                                                       v
   |                                               mission_runner
   |                                                       |
   |                                              canonical adapter
   |                                                       |
   |                                           ROS 2 / simulator / robot
   |
   +<-- receipts + replayed telemetry -- edge outbox
   +<-- artifact metadata ------------ object storage
   +<-- ephemeral preview ------------ WebRTC service
   +<-- append-only decisions -------- audit store
```

Broker acknowledgment only proves broker delivery. Command acceptance occurs after the edge has authenticated, validated, and transactionally persisted the command. Mission success occurs only after the required behavior and evidence are complete.

## Environment and packages

Create a fresh integration workspace rather than modifying the accepted outputs from earlier weeks:

```bash
mkdir -p ~/robotics-lab/week16/{cloud,edge,contracts,missions,maps,evidence,keys,test-results}
cd ~/robotics-lab/week16

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install \
  fastapi 'uvicorn[standard]' pydantic httpx \
  paho-mqtt cryptography rfc8785 boto3 livekit-api \
  pytest pytest-asyncio

python --version | tee evidence/versions.txt
openssl version | tee -a evidence/versions.txt
docker version --format '{{.Server.Version}}' | tee -a evidence/versions.txt
ros2 doctor --report > evidence/ros2-doctor.txt
```

Use the local broker, LiveKit service, and S3-compatible object store from Weeks 12 and 15. Do not copy private keys into containers that do not need them. Suggested repository layout:

```text
week16/
├── cloud/                 # API, command outbox, event/artifact/audit stores
├── edge/                  # command inbox, executor, telemetry outbox
├── contracts/             # versioned JSON Schemas and test vectors
├── missions/              # canonical mission versions and signatures
├── maps/                  # immutable MapVersions and MapReleases
├── keys/                  # lab-only signing material; private keys ignored
├── scripts/               # repeatable startup, outage, and evidence commands
├── evidence/              # sanitized proof bundle
├── test-results/          # machine-readable acceptance reports
├── compose.yaml
└── README.md
```

Add `keys/*.key`, certificates with private material, databases containing live credentials, `.env`, and plaintext tokens to `.gitignore` before continuing.

## Lab: run a disconnected inspection mission with complete evidence

The mission visits three inspection targets, captures one still image at each target, returns to the home pose, and tolerates a planned network outage. WebRTC gives an operator a live preview when available, but preview failure must not change motion state. Simulator target markers may be AprilTags, QR codes, or visibly distinct panels.

### 1. Freeze the acceptance profile before testing

Create `test-results/acceptance-profile.json` with thresholds based on your Week 13 and Week 14 baselines:

```json
{
  "profileVersion": 1,
  "reliabilityRunCount": 10,
  "minimumCompletedRuns": 8,
  "handoffRunIndices": [8, 9, 10],
  "requiredConsecutiveHandoffSuccesses": 3,
  "commandAcceptanceP95Ms": 1000,
  "mediaJoinP95Ms": 10000,
  "telemetryReplayDeadlineSeconds": 30,
  "artifactCompletionDeadlineSeconds": 60,
  "minimumObstacleClearanceMeters": 0.30,
  "maximumLinearSpeedMetersPerSecond": 0.15,
  "localHaltDeadlineMs": 300,
  "maximumOfflineDurationSeconds": 900,
  "maximumLocalizationUncertainty": "<copy the numeric Week 14 bound>"
}
```

Replace the placeholder with the numeric localization bound from Week 14. Do not loosen thresholds after seeing results. If a threshold is unrealistic, record the failed profile, justify a new version, and rerun the complete suite.

Write `evidence/safety-card.md` containing the physical course boundary, spotter, cutoff test result, speed limit, halt deadline, battery limits, and abort conditions. For simulation, mark physical-only fields `not applicable`; do not silently omit them.

### 2. Define the canonical adapter boundary

Reuse the Week 9 contract in `contracts/canonical-adapter-v1.md`. It must expose
only these primitive capabilities (choose one name from each slash pair in the
actual versioned interface):

```text
GetState() / ObserveState() -> freshness-qualified robot state
Halt(command_id, reason) / Stop(command_id, reason) -> durable result
NavigateTo(goal_id, map_release_id, map_release_digest, pose, limits) -> action handle
CaptureImage(capture_id, camera, context) -> local artifact path + digest
```

Require caller-provided idempotency IDs for every effect. `NavigateTo` returns an
action handle that owns status and cancellation; cancellation is not another
robot capability. Translate simulator or hardware-specific messages only
inside the adapter. `mission_runner` owns `ExecuteMission` and composes these
primitives; the adapter accepts neither a mission graph nor an
`ExecuteMission` request. The runner may not publish directly to a drive topic
or call a vendor motion API.

Add contract tests that run unchanged against both your simulator adapter and a fake adapter:

```bash
pytest -q edge/tests/test_adapter_contract.py \
  --junitxml=test-results/adapter-contract.xml
```

The fake adapter should record calls and permit deterministic fault injection. A physical adapter is optional and must not be needed to pass the integration tests.

### 3. Build an immutable `MapVersion` and `MapRelease`

Copy the accepted Week 14 observed geometry into
`maps/versions/training-lab-map-v1/`. Create a `MapVersion` manifest containing
its native and canonical artifacts:

```json
{
  "kind": "MapVersion",
  "schemaVersion": 1,
  "mapVersionId": "training-lab-map-v1",
  "coordinateFrame": "map",
  "resolutionMetersPerPixel": 0.05,
  "origin": [0.0, 0.0, 0.0],
  "nativeArtifacts": [
    {"path": "site.posegraph", "sha256": "<sha256>"}
  ],
  "canonicalArtifacts": [
    {"path": "map.yaml", "sha256": "<sha256>"},
    {"path": "map.pgm", "sha256": "<sha256>"}
  ],
  "createdAt": "<RFC3339 timestamp>"
}
```

Canonicalize that manifest and call its digest the `mapVersionDigest`. Then
create `maps/releases/map-release-001/manifest.json`:

```json
{
  "kind": "MapRelease",
  "schemaVersion": 1,
  "mapReleaseId": "map-release-001",
  "mapVersion": {
    "mapVersionId": "training-lab-map-v1",
    "mapVersionDigest": "<sha256>"
  },
  "alignment": {"frame": "map", "transform": [0.0, 0.0, 0.0]},
  "overlays": {"noGoZones": [], "speedZones": []},
  "policies": {"navigationParameters": "nav2-params-v1"},
  "routes": {"inspection-a": [1.0, 0.5, 0.0], "inspection-b": [1.8, -0.4, 1.57]},
  "createdAt": "<RFC3339 timestamp>"
}
```

The schema must require exactly one `mapVersion` object and reject arrays or a
second version reference. Canonicalize the release manifest and call its digest
the `mapReleaseDigest`:

```bash
cd ~/robotics-lab/week16
sha256sum maps/versions/training-lab-map-v1/map.yaml \
  maps/versions/training-lab-map-v1/map.pgm \
  | tee evidence/map-file-digests.txt

python -c 'import json,rfc8785,sys; sys.stdout.buffer.write(rfc8785.dumps(json.load(sys.stdin)))' \
  < maps/versions/training-lab-map-v1/manifest.json \
  > maps/versions/training-lab-map-v1/manifest.canonical.json
sha256sum maps/versions/training-lab-map-v1/manifest.canonical.json \
  | tee evidence/map-version-digest.txt

python -c 'import json,rfc8785,sys; sys.stdout.buffer.write(rfc8785.dumps(json.load(sys.stdin)))' \
  < maps/releases/map-release-001/manifest.json \
  > maps/releases/map-release-001/manifest.canonical.json
sha256sum maps/releases/map-release-001/manifest.canonical.json \
  | tee evidence/map-release-digest.txt
```

Fail preflight if either digest or any referenced artifact differs. An optional
mutable channel may point to the release ID and digest, but resolve it before
assignment. Do not repair active work in place; publish a new `MapVersion` for
changed geometry or a new `MapRelease` for changed composition.

### 4. Create and sign the mission version

Create `missions/inspection-v1/mission.json`:

```json
{
  "schemaVersion": 1,
  "missionVersionId": "inspection-v1",
  "requiredCapabilities": [
    "observe_state.v1",
    "navigate.v1",
    "halt.v1",
    "capture_still.v1"
  ],
  "behaviorTree": {
    "type": "Sequence",
    "children": [
      {"type": "Preflight"},
      {"type": "Navigate", "target": "inspection-a"},
      {"type": "WaitForStablePose", "seconds": 2},
      {"type": "CaptureStill", "evidenceSlot": "inspection-a-front"},
      {"type": "Navigate", "target": "inspection-b"},
      {"type": "WaitForStablePose", "seconds": 2},
      {"type": "CaptureStill", "evidenceSlot": "inspection-b-front"},
      {"type": "Navigate", "target": "inspection-c"},
      {"type": "WaitForStablePose", "seconds": 2},
      {"type": "CaptureStill", "evidenceSlot": "inspection-c-front"},
      {"type": "Navigate", "target": "home"},
      {"type": "Finalize"}
    ]
  },
  "targets": {
    "inspection-a": {"frame": "map", "x": 1.0, "y": 0.5, "yaw": 0.0},
    "inspection-b": {"frame": "map", "x": 1.8, "y": -0.4, "yaw": 1.57},
    "inspection-c": {"frame": "map", "x": 0.8, "y": -1.2, "yaw": 3.14},
    "home": {"frame": "map", "x": 0.0, "y": 0.0, "yaw": 0.0}
  },
  "requiredEvidenceSlots": [
    "inspection-a-front",
    "inspection-b-front",
    "inspection-c-front"
  ]
}
```

Generate a lab mission-signing key, canonicalize the document with RFC 8785, sign the exact canonical bytes, and verify the signature:

```bash
openssl genpkey -algorithm ED25519 -out keys/mission-signer.key
openssl pkey -in keys/mission-signer.key -pubout -out keys/mission-signer.pub

python -c 'import json,rfc8785,sys; sys.stdout.buffer.write(rfc8785.dumps(json.load(sys.stdin)))' \
  < missions/inspection-v1/mission.json \
  > missions/inspection-v1/mission.canonical.json

openssl pkeyutl -sign -rawin \
  -inkey keys/mission-signer.key \
  -in missions/inspection-v1/mission.canonical.json \
  -out missions/inspection-v1/mission.sig

openssl pkeyutl -verify -rawin -pubin \
  -inkey keys/mission-signer.pub \
  -in missions/inspection-v1/mission.canonical.json \
  -sigfile missions/inspection-v1/mission.sig
```

The edge trusts only the public key. Test that a one-byte mission change fails verification. Signature success does not make a mission safe: the edge must also validate schema, capabilities, lifecycle state, map digest, limits, and local safety policy.

### 5. Define the immutable run bundle and offline policy

When the operator creates `run-001`, the cloud service snapshots all run inputs into one `MissionBundle`:

```json
{
  "schemaVersion": 1,
  "runId": "run-001",
  "assignmentRevision": 1,
  "robotId": "robot-lab-01",
  "robotEpoch": "<current-boot-uuid>",
  "missionVersionId": "inspection-v1",
  "missionCanonicalSha256": "<sha256>",
  "missionSignature": "<base64>",
  "mapReleaseId": "map-release-001",
  "mapReleaseDigest": "<sha256>",
  "connectivityPolicy": {
    "onDisconnect": "CONTINUE_CURRENT_RUN",
    "maximumOfflineDurationSeconds": 900,
    "allowNewRunWhileOffline": false,
    "mediaRequiredForMotion": false,
    "artifactUploadRequiredForMotion": false
  },
  "limits": {
    "maximumLinearSpeedMetersPerSecond": 0.15,
    "minimumObstacleClearanceMeters": 0.30
  },
  "createdAt": "<RFC3339 timestamp>",
  "expiresAt": "<RFC3339 timestamp after the planned test>"
}
```

Canonicalize and sign the complete bundle. The edge must download, authenticate, verify, and atomically persist the bundle and all referenced map/mission content before acknowledging `ACCEPTED`. A URL is transport, not identity; the digest is the content identity.

The offline policy authorizes only continuation of this already accepted run. It does not authorize a new assignment, policy change, software update, certificate recovery, or map substitution while disconnected.

### 6. Make command delivery durable and retry-safe

Use the identity-scoped topics from Week 15:

```text
robots/robot-lab-01/commands
robots/robot-lab-01/receipts
robots/robot-lab-01/events
```

Publish a versioned command envelope at QoS 1:

```json
{
  "schemaVersion": 1,
  "messageId": "msg-assign-run-001",
  "messageType": "mission.assign.v1",
  "robotId": "robot-lab-01",
  "robotEpoch": "<current-boot-uuid>",
  "sequence": 42,
  "correlationId": "run-001",
  "createdAt": "<RFC3339 timestamp>",
  "expiresAt": "<RFC3339 timestamp>",
  "payload": {
    "runId": "run-001",
    "bundleSha256": "<sha256>",
    "bundleUrl": "<short-lived local URL>"
  }
}
```

Implement this ordering:

1. The cloud API authenticates the operator, validates the request, and stores run plus command intent in one transaction.
2. A cloud outbox worker publishes the stored command and records publish attempts.
3. The edge derives robot identity from mTLS, then checks schema, topic, payload robot ID, lifecycle, epoch, sequence, time window, and bundle signature/digests.
4. The edge inserts the command into an inbox with `messageId` unique and persists the accepted run in the same transaction.
5. Only after commit does the edge emit an `ACCEPTED` receipt.
6. Duplicate delivery returns the previously stored receipt and never creates a second run.
7. The cloud service correlates receipts by both `messageId` and `runId`; a timeout is `UNKNOWN`, not automatic failure.

A receipt needs `receiptId`, `messageId`, `runId`, robot ID, edge timestamp, outcome, reason code, and the accepted bundle digest. Never trust a payload `robotId` over the authenticated certificate identity.

### 7. Journal edge execution before effects

Create the edge database with WAL and foreign keys enabled. At minimum, implement:

```sql
CREATE TABLE inbox (
  message_id TEXT PRIMARY KEY,
  envelope_sha256 TEXT NOT NULL,
  outcome TEXT NOT NULL,
  receipt_json TEXT NOT NULL,
  received_at TEXT NOT NULL
);

CREATE TABLE mission_run (
  run_id TEXT PRIMARY KEY,
  bundle_sha256 TEXT NOT NULL,
  map_release_id TEXT NOT NULL,
  map_release_digest TEXT NOT NULL,
  state TEXT NOT NULL,
  next_node_index INTEGER NOT NULL,
  offline_since TEXT,
  updated_at TEXT NOT NULL
);

CREATE TABLE node_attempt (
  attempt_id TEXT PRIMARY KEY,
  run_id TEXT NOT NULL,
  node_index INTEGER NOT NULL,
  effect_id TEXT NOT NULL UNIQUE,
  state TEXT NOT NULL,
  started_at TEXT NOT NULL,
  finished_at TEXT,
  result_json TEXT,
  FOREIGN KEY(run_id) REFERENCES mission_run(run_id)
);

CREATE TABLE event_outbox (
  event_id TEXT PRIMARY KEY,
  robot_epoch TEXT NOT NULL,
  stream_sequence INTEGER NOT NULL,
  event_json TEXT NOT NULL,
  acknowledged_at TEXT,
  UNIQUE(robot_epoch, stream_sequence)
);
```

Before calling the adapter, persist a deterministic `effect_id` such as `run-001/node-03/attempt-01`. On restart, reconcile an incomplete attempt with the adapter action result before deciding to retry. Navigation and capture calls use that same effect ID as their idempotency key.

Record a state transition and its event-outbox row in one transaction. This is the transaction boundary that prevents a completed transition from disappearing when telemetry publication fails.

### 8. Execute once while connected

Start the simulator, navigation stack, broker, cloud service, media service, object store, and a network-fault proxy. Configure the edge to use only the proxy endpoints for broker, API, and object-store traffic; the operator continues to use the direct API port.

```bash
cd ~/robotics-lab/week16
docker compose up -d broker cloud-api livekit object-store network-proxy

curl -fsS -X POST http://localhost:8474/proxies \
  -H 'Content-Type: application/json' \
  -d '{"name":"edge-broker","listen":"0.0.0.0:1884","upstream":"broker:1883"}'
curl -fsS -X POST http://localhost:8474/proxies \
  -H 'Content-Type: application/json' \
  -d '{"name":"edge-api","listen":"0.0.0.0:8001","upstream":"cloud-api:8000"}'
curl -fsS -X POST http://localhost:8474/proxies \
  -H 'Content-Type: application/json' \
  -d '{"name":"edge-object-store","listen":"0.0.0.0:9001","upstream":"object-store:9000"}'

# Use separate terminals for simulator/navigation and the edge.
export SIMULATION_PACKAGE=replace_with_simulation_package
export WORLD_LAUNCH_FILE=replace_with_world_launch_filename
export NAVIGATION_PACKAGE=replace_with_navigation_package
export NAVIGATION_LAUNCH_FILE=replace_with_navigation_launch_filename
test "$SIMULATION_PACKAGE" != replace_with_simulation_package
test "$WORLD_LAUNCH_FILE" != replace_with_world_launch_filename
test "$NAVIGATION_PACKAGE" != replace_with_navigation_package
test "$NAVIGATION_LAUNCH_FILE" != replace_with_navigation_launch_filename
ros2 launch "$SIMULATION_PACKAGE" "$WORLD_LAUNCH_FILE"
ros2 launch "$NAVIGATION_PACKAGE" "$NAVIGATION_LAUNCH_FILE" \
  map:=~/robotics-lab/week16/maps/versions/training-lab-map-v1/map.yaml

source .venv/bin/activate
python -m edge.main --config edge/config.yaml
```

Expose the proxy API and listener ports in `compose.yaml`; a public `ghcr.io/shopify/toxiproxy` service is sufficient. Point `edge/config.yaml` at `127.0.0.1:1884`, `http://127.0.0.1:8001`, and `http://127.0.0.1:9001`. Replace the two ROS launch placeholders with the exact accepted commands from Week 14 and record all commands in `README.md`. Rerunning proxy creation may return “already exists”; inspect and reuse the existing definitions rather than silently creating new names.

Create and assign the run through your API:

```bash
curl -fsS -X POST http://localhost:8000/v1/mission-runs \
  -H 'Content-Type: application/json' \
  -d @missions/run-001-create.json | jq .

curl -fsS -X POST \
  http://localhost:8000/v1/mission-runs/run-001:assign \
  -H 'Content-Type: application/json' \
  -d '{"robotId":"robot-lab-01","idempotencyKey":"assign-run-001"}' | jq .
```

Verify before motion that the edge reports the exact mission and map digests, required capabilities, accepted speed/clearance limits, good localization, no active fault, and an available local halt. Run to completion and capture a baseline bag:

```bash
ros2 bag record -o evidence/connected-baseline \
  /tf /tf_static /odom /scan /cmd_vel /amcl_pose /diagnostics
```

Stop recording after the robot returns home. Store the run summary, adapter calls, node attempts, event sequences, path metrics, and two local evidence digests.

### 9. Disconnect the cloud path during an accepted run

Create `run-002`, wait for its durable `ACCEPTED` receipt, and verify the complete bundle is local. Then disable only the three edge-to-cloud proxy routes. Do not stop ROS 2, Nav2, the local adapter, watchdog, or collision monitor.

```bash
for proxy in edge-broker edge-api edge-object-store; do
  curl -fsS -X POST "http://localhost:8474/proxies/${proxy}" \
    -H 'Content-Type: application/json' \
    -d '{"enabled":false}'
done
```

Confirm from the edge logs that all three routes are unreachable. If any cloud operation still succeeds, fix the edge configuration and repeat with a fresh run; the outage is not valid. Do not improvise firewall rules on a physical robot during motion.

Verify while disconnected:

- the current accepted mission continues under the frozen policy;
- a new mission is not accepted;
- the map stays `map-release-001`;
- state transitions and evidence metadata accumulate in the local database;
- live preview becomes unavailable without changing mission state; and
- the local halt and collision monitor remain effective.

Reconnect before the 900-second offline limit:

```bash
for proxy in edge-broker edge-api edge-object-store; do
  curl -fsS -X POST "http://localhost:8474/proxies/${proxy}" \
    -H 'Content-Type: application/json' \
    -d '{"enabled":true}'
done
```

The edge should resume authenticated sessions, replay the outbox, upload pending artifacts through refreshed URLs, and reconcile the cloud run without executing any node twice.

### 10. Replay telemetry with explicit sequence semantics

Every event uses this envelope:

```json
{
  "schemaVersion": 1,
  "eventId": "<uuid>",
  "eventType": "mission.node.completed.v1",
  "robotId": "robot-lab-01",
  "robotEpoch": "<boot-uuid>",
  "streamSequence": 108,
  "runId": "run-002",
  "correlationId": "run-002",
  "observedAt": "<RFC3339 timestamp>",
  "monotonicNanoseconds": 1234567890,
  "payload": {"nodeIndex": 3, "outcome": "SUCCEEDED"}
}
```

The cloud event store enforces uniqueness on `(robotId, robotEpoch, eventId)` and on `(robotId, robotEpoch, streamSequence)`. It acknowledges the highest contiguous sequence, not merely the highest seen number. The edge deletes nothing until the acknowledgment is durable.

After reconnection, compare both sides:

```bash
sqlite3 edge/state.db \
  'select robot_epoch,count(*),min(stream_sequence),max(stream_sequence) from event_outbox group by robot_epoch;'
sqlite3 cloud/state.db \
  'select robot_epoch,count(*),min(stream_sequence),max(stream_sequence) from robot_event group by robot_epoch;'
```

Export the ordered IDs from both stores and compare them. Duplicate resend must leave the cloud row count unchanged. A genuine sequence gap remains visible and blocks contiguous acknowledgment until repaired or explicitly declared lost with an audited operator decision.

### 11. Add ephemeral WebRTC preview without coupling it to control

Use the Week 12 API to create a media invitation for `run-002`. Return an opaque session ID and redemption URL to the operator; do not return a long-lived JWT, SDP, or ICE credentials in the mission bundle. Redeem a short-lived, role-scoped viewer token only when joining.

Measure join time and one-minute WebRTC statistics: round-trip time, inbound bitrate, frames decoded, frame loss, jitter, and reconnect count. Then stop the media service for 30 seconds. The mission executor must receive no cancel, pause, or speed command from that failure. Restore media as a new generation while keeping the same durable session resource and record the generation transition.

The WebRTC recording, if enabled, is still not the authoritative inspection artifact. Required evidence follows the durable artifact path in the next step.

### 12. Upload evidence by stable identity and digest

For each evidence slot, create one stable artifact resource before upload:

```json
{
  "artifactId": "artifact-run-002-inspection-a-front",
  "runId": "run-002",
  "slot": "inspection-a-front",
  "captureId": "run-002/node-03/attempt-01",
  "mediaType": "image/jpeg",
  "byteLength": 184233,
  "sha256": "<sha256 of exact bytes>",
  "capturedAt": "<RFC3339 timestamp>",
  "mapReleaseId": "map-release-001",
  "mapReleaseDigest": "<sha256>",
  "pose": {"frame":"map","x":1.0,"y":0.5,"yaw":0.0}
}
```

Request a short-lived upload URL, upload the exact bytes, and call completion. The service verifies object key, size, media type, digest, and artifact/run ownership before marking `AVAILABLE`. If the URL expires, request a new URL for the same `artifactId`; do not recapture or create a second logical artifact unless the capture itself failed.

Keep local bytes until completion is durable and retention policy permits deletion. Prove that two completion calls return the same result and one artifact row.

### 13. Prove map pinning during release change

While `run-002` is offline, publish a second immutable `MapRelease` with a
different approved overlay and release digest. Move the optional `staging`
channel to its ID and digest.

The active run must continue using release 001. Its telemetry, artifact
metadata, navigation inputs, and final audit record must all reference
`map-release-001` and its original `mapReleaseDigest`. Only a newly created
`run-003` may resolve `staging` and pin release 002. Neither run pins a bare
`MapVersion`.

Never hot-swap a map under a moving physical robot for this test. Demonstrate the release change in simulation; for a physical demonstration, inspect identifiers without starting the new map.

### 14. Build the append-only audit timeline

Emit an audit event for each consequential decision:

- operator authentication and run creation;
- command publication, redelivery, acceptance, or rejection;
- certificate identity and lifecycle authorization result;
- mission and map digest verification;
- offline-policy entry, timeout, and exit;
- local halt, cancellation, or safety-policy decision;
- media invitation creation and generation replacement;
- artifact URL issue, verification, and completion;
- map release publication and run pin selection; and
- run terminal state and evidence completeness.

Each record has a unique event ID, actor or authenticated principal, action, resource IDs, outcome, reason code, correlation ID, timestamp, and previous-record digest. Hash-chain daily audit files:

```text
recordDigest = SHA-256(canonicalRecordWithoutDigest || previousRecordDigest)
```

This makes later mutation detectable; it does not by itself prevent deletion. Record that residual risk and retain an exported final digest separately.

Scan all exported logs and audit files for secrets:

```bash
if rg -n \
  'BEGIN (RSA |EC |)PRIVATE KEY|Authorization: Bearer|bootstrapGrant|sessionToken|presignedUrl' \
  evidence test-results cloud/logs edge/logs; then
  echo 'FAIL: potential secret material found'
  exit 1
else
  echo 'PASS: no configured secret patterns found' \
    | tee evidence/secret-scan.txt
fi
```

Review the result manually as well; a pattern scan cannot prove that every credential format is absent.

### 15. Run the deliberate failure campaign

Use a fresh run ID for each case and preserve before/after database counts. Automate these cases in `pytest` wherever possible:

1. **Duplicate assignment:** publish the identical command three times; expect one run, one inbox row, one effect sequence, and stable receipt content.
2. **Conflicting reuse:** reuse `messageId` with different bytes; expect a terminal integrity rejection and audit event.
3. **Expired command:** deliver after `expiresAt`; expect no run and no adapter call.
4. **Wrong identity:** connect with another valid client certificate and publish to the robot topic; expect broker or service denial and no inbox row.
5. **Tampered bundle:** change one target after signing; expect signature/digest rejection before adapter use.
6. **Stale epoch:** replay a command addressed to a previous boot epoch; expect rejection or explicit re-resolution, never silent execution.
7. **Network loss:** disconnect after durable acceptance; expect bounded offline continuation, local journaling, and complete replay after reconnect.
8. **Edge restart:** terminate the edge after a node is journaled but before its event is acknowledged; expect deterministic reconciliation and no duplicate effect.
9. **Media loss:** stop LiveKit while navigating in simulation; expect mission state and velocity policy to remain unchanged.
10. **Expired upload URL:** delay upload past expiry; expect a refreshed URL for the same artifact and one completion record.
11. **Duplicate telemetry:** replay an already acknowledged range; expect unchanged cloud counts and an acknowledgment of the same contiguous sequence.
12. **New map release:** publish release 002 mid-run; expect the active run and artifacts to remain pinned to release 001.
13. **Localization loss:** in simulation only, make localization exceed the frozen bound; expect local controlled halt or safe pause and no blind continuation.
14. **Offline-policy expiry:** exceed the maximum duration in accelerated simulation; expect the documented safe terminal or paused state, not indefinite execution.

Do not inject a physical obstacle, collision, battery fault, disabled watchdog, or failed cutoff. Use the fake adapter or simulator for hazardous paths.

Run the automated suite and save machine-readable results:

```bash
pytest -q cloud/tests edge/tests integration/tests \
  --junitxml=test-results/capstone.xml
```

### 16. Run the ten-attempt reliability and handoff campaign

Reset only run-specific state; do not delete audit history or alter the acceptance profile. Execute ten clean attempts from the same start conditions. Predesignate attempts 8, 9, and 10 as the handoff subset before attempt 1 starts. For those three attempts, another learner follows only the clean-start runbook from stopped processes and may not receive undocumented commands or corrections.

At least 8 of 10 attempts must complete all three inspections, make all three artifacts available, return home, and reach the successful terminal state. Attempts 8–10 must all succeed consecutively. Every attempt—including either allowed mission non-success—must preserve all safety and integrity invariants: zero contact, bounded motion, truthful terminal state, pinned inputs, one logical effect per ID, loss-visible telemetry, and complete audit. Preserve and explain every non-success; never discard or silently rerun an unfavorable seed.

For each attempt, capture:

- bundle, mission, and map digests;
- command publish-to-accept latency and duplicate counts;
- node start/completion times and adapter effect IDs;
- path length, duration, minimum obstacle clearance, maximum speed, localization quality, and final pose error;
- offline start/end, maximum outbox depth, replay duration, duplicates received, and unresolved gaps;
- WebRTC join time and statistics;
- artifact ID, byte length, digest, retry count, and completion time;
- local halt and safety-monitor health; and
- audit event coverage and final hash-chain digest.

Write one JSON summary per attempt and a combined `test-results/acceptance-summary.json` that lists all ten attempts, total completions, failure classifications, and the predesignated handoff subset. Averages never hide an unsafe outlier: safety and integrity invariants pass on all 10 attempts even though the mission-reliability threshold permits up to two explicit safe non-successes.

## Measurements

Use monotonic time for local durations and RFC 3339 UTC for cross-system correlation. Report at least:

| Area | Required measurement |
| --- | --- |
| Reliability | attempted/completed/safely-not-completed counts across all 10, failure reasons, consecutive handoff result |
| Commands | publish-to-accept p50/p95, retries, duplicates, conflicts, expired/stale rejections |
| Mission | duration, node attempts, reconciliations, terminal state, final pose error |
| Safety | maximum speed, minimum clearance, localization bound, local halt latency |
| Offline | disconnected duration, maximum outbox rows/bytes, replay time, unresolved gaps |
| Telemetry | produced, received, unique, duplicate, rejected, and acknowledged-contiguous counts |
| Media | join p50/p95, round-trip time, jitter, loss, frames decoded, reconnects |
| Artifacts | capture-to-available time, bytes, digest checks, URL refreshes, duplicate completions |
| Security | authentication/authorization denials, lifecycle state, secret-scan findings |
| Audit | expected decisions, recorded decisions, missing correlations, hash-chain verification |

Preserve raw observations as well as aggregates. Include the exact query or script that produced every reported number.

## Assignment

Deliver a 10-minute capstone demonstration and a short engineering report:

1. Show the signed mission bundle, pinned `MapRelease` ID and digest, active robot certificate identity, and frozen acceptance profile.
2. Assign a run twice and prove it executes once.
3. Disconnect the network after acceptance and show continued local execution plus a growing outbox.
4. Show WebRTC loss without control impact, then restore the preview.
5. Reconnect and prove ordered telemetry replay and all three artifact completions by digest.
6. Publish a newer map and prove the active run remains pinned.
7. Trigger one safe failure in simulation and show the expected state, receipt, and audit records.
8. Show the complete ten-attempt reliability table and the three consecutive predesignated handoff successes.
9. Finish with the acceptance matrix and identify the largest remaining production risk.

The report should distinguish observed fact, inferred cause, and design assumption. Include one sequence diagram for the offline/reconnect path and one postmortem for a failed test, even if you fixed it.

## Evidence and deliverables

Submit a sanitized archive with this structure:

```text
evidence/
├── README.md
├── safety-card.md
├── versions.txt
├── ros2-doctor.txt
├── acceptance-profile.json
├── acceptance-summary.json
├── adapter-contract.xml
├── capstone.xml
├── reliability-runs/
│   ├── attempt-01.json
│   ├── ...
│   └── attempt-10.json
├── handoff-runbook.md
├── command-receipts.jsonl
├── node-attempts.jsonl
├── telemetry-comparison.json
├── media-stats.json
├── artifacts.json
├── audit.jsonl
├── audit-final-digest.txt
├── map-file-digests.txt
├── map-version-digest.txt
├── map-release-digest.txt
├── secret-scan.txt
├── connected-baseline/
├── offline-run/
└── architecture-and-postmortem.md
```

Include public keys and public certificates only when needed for verification. Exclude private keys, bootstrap grants, bearer tokens, presigned URLs, cookies, broker passwords, raw environment files, and images containing private information.

## Objective acceptance matrix

Fill the evidence reference for every row. `PASS` requires the stated observation; “appears to work” is not evidence.

| ID | Invariant | Test stimulus | Pass criterion | Required evidence |
| --- | --- | --- | --- | --- |
| A01 | Safety is local | Remove cloud/broker connectivity | Watchdog, collision monitor, halt, and cutoff remain healthy; physical halt meets the frozen deadline | Safety card, bag, halt timing |
| A02 | Adapter boundary is canonical | Run contract suite against fake and simulator adapters | Same tests pass; executor contains no direct drive/vendor call | JUnit and dependency review |
| A03 | Mission input is immutable | Alter one canonical mission byte | Signature verification fails before inbox acceptance or adapter call | Negative-test log and call count |
| A04 | Map is content-addressed | Change one map file without a new manifest | Preflight rejects the digest mismatch | Manifest and rejection receipt |
| A05 | Active run is pinned | Move a channel to release 002 during run on 001 | Every navigation, event, artifact, and terminal record retains release 001 ID and digest | Run/event/artifact queries |
| A06 | Assignment is durable | Stop publisher after edge `ACCEPTED` | Inbox, run, and receipt survive restart | Database rows before/after |
| A07 | Redelivery is idempotent | Send identical assignment three times | One run and effect sequence; stable stored receipt | Counts and adapter calls |
| A08 | Conflicts are detected | Reuse message ID with different content | No effect; integrity rejection and audit event | Receipt and audit record |
| A09 | Identity controls routing | Publish with wrong valid certificate | Denied; zero new inbox and adapter rows | Broker/service log and counts |
| A10 | Offline authority is bounded | Disconnect after acceptance | Current run follows frozen policy; no new run/config/map accepted | Offline timeline and state query |
| A11 | Restart is recoverable | Restart edge at a journal boundary | Run reconciles without duplicate effect or skipped required node | Node attempts and effect IDs |
| A12 | Replay is loss-visible | Reconnect with queued events and one duplicate range | All produced IDs arrive once; contiguous acknowledgment has no hidden gap | Ordered ID comparison |
| A13 | Media is non-authoritative | Stop and replace media service during simulated navigation | Mission state and safety limits are unchanged; preview recovers as new generation | Media and mission timelines |
| A14 | Artifact identity is stable | Expire URL and repeat completion | Same artifact ID reaches `AVAILABLE` once with matching size and digest | Object metadata and DB row |
| A15 | Localization failure is safe | Exceed bound in simulation | Local controlled halt or documented safe pause; no blind navigation | Bag, state transition, audit |
| A16 | Audit is complete and tamper-evident | Verify expected decisions and hash chain | 100% required decisions correlated; chain verifies from first to final digest | Audit coverage report |
| A17 | Evidence contains no configured secret | Scan and manually review archive | Zero secret findings; only public verification material included | Scan result and review sign-off |
| A18 | Result is reliable and repeatable | Run ten clean attempts with attempts 8–10 predesignated for independent handoff | At least 8/10 complete; attempts 8–10 succeed consecutively; safety and integrity invariants pass 10/10; thresholds remain profile version 1 | Ten attempt summaries and handoff record |

## Objective exit criteria

Week 16 is complete only when all of the following are true:

- all automated tests pass from a clean checkout with documented startup commands;
- every row A01–A18 is `PASS` with a concrete evidence reference;
- at least 8 of 10 clean attempts complete successfully, and predesignated handoff attempts 8–10 succeed consecutively from the documented clean-start runbook;
- duplicate, stale, expired, conflicting, and wrong-identity commands cause zero unintended adapter effects;
- the offline run completes or pauses exactly according to its signed policy, and reconnect produces no missing or duplicate logical event;
- the active run never changes mission digest or pinned `MapRelease` ID/digest;
- all three required evidence artifacts in every successful attempt are durable, digest-verified, and linked to the correct run, inspection pose, and map;
- media failure has no control or safety effect;
- the localization-loss test reaches the documented safe state;
- the audit chain verifies and covers every required decision; and
- the delivered evidence archive passes secret scanning and manual review.

If any safety invariant fails, stop physical testing and return to simulation or blocks. Do not average, waive, or explain away a failed safety row.

## Troubleshooting

| Symptom | Checks and likely cause |
| --- | --- |
| Duplicate mission executes | Inbox insert and run creation are not atomic, message ID is regenerated, or adapter effect IDs are not stable. |
| Receipt exists but run disappears after restart | Receipt was emitted before durable commit or database volume is ephemeral. |
| Replayed events are missing | Edge deleted on publish rather than durable contiguous acknowledgment, or epoch/sequence changed unexpectedly. |
| Cloud counts more events than edge produced | Receiver lacks unique constraints or event IDs change during retry. |
| Run silently adopts a new map | Code resolves a channel during execution instead of using the bundle's `MapRelease` ID and digest. |
| Edge rejects a good bundle | Compare canonical bytes, digest encoding, public key, signature mode, schema version, and clock bounds. |
| Mission stops when video drops | Media health leaked into the executor or navigation behavior tree; remove that dependency. |
| Evidence completion reports mismatch | Hash exact uploaded bytes; verify content encoding, object key, byte length, and metadata. |
| Physical and simulated adapters differ | Move translation behind the adapter and rerun the same contract tests; do not fork mission logic. |
| Halt misses the deadline | Treat as safety failure; stay on blocks/simulation and inspect local scheduling, watchdog, controller, and measurement clock. |
| Audit chain breaks | Find first differing canonical record or previous digest; preserve both copies and investigate rather than rewriting history. |
| Certificate authenticates but is unauthorized | Check lifecycle state, authorization version, topic ACL, certificate identity binding, and connection fencing. |

## Next step

Return to the [full learning plan](../learning-plan.md), review your measurements and postmortem, and choose the weakest subsystem for a second iteration. Good extensions are multi-robot scheduling, hardware-backed keys, signed software updates with rollback, richer perception, fleet observability, or formal safety analysis—but keep the same discipline: explicit authority, immutable inputs, local safety, durable state, and objective evidence.
