# Week 12 — WebRTC Media and Artifact Transfer

[← Week 11: Map versions and releases](week-11-map-versions-and-releases.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Week 13: Physical robot bring-up →](week-13-physical-robot-bring-up.md)

**Estimated effort:** 8–10 hours

**Lab mode:** laptop only; a webcam is helpful but not required

## Outcomes

By the end of this week you will be able to:

- explain why robot control messages, telemetry, live media, and durable files need different transports;
- create a role-scoped WebRTC room with one publisher and one or more viewers;
- keep session invitations free of room credentials, SDP, ICE data, and media bytes;
- upload evidence through a short-lived, object-scoped URL and verify its SHA-256 digest;
- distinguish durable session metadata from ephemeral media-plane state; and
- measure time to first frame, round-trip time, packet loss, upload throughput, and integrity failures.

## Prerequisites

- Completed Week 11, including an immutable map release or equivalent versioned artifact.
- Comfortable with HTTP, JSON, environment variables, and Docker.
- Docker Engine or Docker Desktop, Python 3.11+, Node.js 20+, `curl`, `jq`, and `openssl`.
- A browser with WebRTC support. Use `getUserMedia()` only from `localhost` or HTTPS.
- At least 4 GB of free memory and ports `7880`, `7881`, `9000`, and `9001` available.

## Public readings

Read these before the lab:

1. [MDN: WebRTC API](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)
2. [MDN: WebRTC connectivity and ICE](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Connectivity)
3. [LiveKit: rooms, participants, and tracks](https://docs.livekit.io/home/get-started/api-primitives/)
4. [LiveKit: access tokens and grants](https://docs.livekit.io/home/get-started/authentication/)
5. [Amazon S3: presigned URL uploads](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)
6. [W3C WebRTC statistics](https://www.w3.org/TR/webrtc-stats/)

## Concepts

| Concept | Meaning in this lab |
| --- | --- |
| Signaling | WSS/HTTPS exchange that lets participants join a room; it is not the media stream. |
| ICE / STUN / TURN | Candidate selection, public-address discovery, and relay fallback for restrictive networks. |
| SFU | A server that receives one publication and forwards it to authorized viewers without requiring another robot upload per viewer. |
| Track | A named camera, audio, or data publication within a room. |
| Session generation | A monotonically increasing attempt number. A replacement room gets a new generation. |
| Ephemeral media | Frames may be lost and are not reconstructed after a room failure. |
| Durable artifact | A file with stable identity, digest, metadata, and completion state. |
| Presigned upload | A narrow bearer capability for one object and operation, with a short expiry. |

Keep these boundaries explicit:

```text
session invitation -> durable control path
room join          -> HTTPS/WSS with a short-lived token
live frames        -> WebRTC
metrics/state      -> telemetry path
recording/evidence -> object upload plus durable completion event
```

## Environment and packages

Create a disposable workspace:

```bash
mkdir -p ~/robotics-lab/week12/{app,objects,evidence}
cd ~/robotics-lab/week12
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install fastapi uvicorn livekit-api boto3 python-multipart
python -m pip freeze > requirements.lock.txt
npm init -y
npm install livekit-client
```

Record the exact versions used:

```bash
python --version | tee evidence/runtime-versions.txt
node --version | tee -a evidence/runtime-versions.txt
docker version --format '{{.Server.Version}}' | tee -a evidence/runtime-versions.txt
python -m pip freeze | tee -a evidence/runtime-versions.txt
npm list --depth=0 | tee -a evidence/runtime-versions.txt
```

The services below use intentionally simple development credentials and unencrypted localhost endpoints. Never expose this configuration beyond your machine.

## Lab: separate live media from durable evidence

### 1. Start the local media and object services

Start LiveKit in development mode:

```bash
docker run --rm --name week12-livekit \
  -p 7880:7880 -p 7881:7881 \
  livekit/livekit-server \
  --dev --bind 0.0.0.0
```

In a second terminal, start an S3-compatible object store:

```bash
docker run --rm --name week12-objects \
  -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=labadmin \
  -e MINIO_ROOT_PASSWORD=labadmin-change-me \
  -v "$HOME/robotics-lab/week12/objects:/data" \
  quay.io/minio/minio server /data --console-address ':9001'
```

Confirm both are reachable:

```bash
curl -fsS http://127.0.0.1:7880 >/dev/null
curl -fsS http://127.0.0.1:9000/minio/health/live
```

### 2. Create the session and artifact service

Create `app/service.py`. It must expose these operations:

| Operation | Required behavior |
| --- | --- |
| `POST /sessions` | Create opaque `sessionId`, room name, generation `1`, requested tracks, viewer limit, expiry, and `REQUESTED` state. |
| `POST /sessions/{id}/viewer-token` | Reauthorize the caller and return a short-lived subscribe-only token. |
| `POST /sessions/{id}/publisher-token` | Require the expected robot identity and return a publish-only token for the current generation. |
| `DELETE /sessions/{id}` | Mark the session terminal and delete the room. |
| `POST /artifacts/{id}/upload-authorization` | Create a presigned `PUT` URL for exactly one object. |
| `POST /artifacts/{id}/complete` | Verify object existence, byte count, and SHA-256 before marking it complete. |

Use the LiveKit development credentials `devkey` and `secret` only for this localhost lab. A minimal token helper is:

```python
from datetime import timedelta
from livekit import api

def room_token(identity: str, room: str, *, publish: bool, subscribe: bool) -> str:
    grant = api.VideoGrants(
        room_join=True,
        room=room,
        can_publish=publish,
        can_subscribe=subscribe,
    )
    return (
        api.AccessToken("devkey", "secret")
        .with_identity(identity)
        .with_ttl(timedelta(minutes=2))
        .with_grants(grant)
        .to_jwt()
    )
```

For the object store, initialize the client with:

```python
import boto3

s3 = boto3.client(
    "s3",
    endpoint_url="http://127.0.0.1:9000",
    aws_access_key_id="labadmin",
    aws_secret_access_key="labadmin-change-me",
    region_name="us-east-1",
)
s3.create_bucket(Bucket="robot-evidence")
```

Generate upload URLs with a 60-second expiry. Store only the artifact ID, object key, expected media type, expected digest, expected bytes, state, and timestamps. Never store the presigned URL as artifact state.

Run the API:

```bash
cd ~/robotics-lab/week12
source .venv/bin/activate
uvicorn app.service:app --host 127.0.0.1 --port 8080 --reload
```

### 3. Create a session without leaking credentials

Create the session:

```bash
curl -fsS -X POST http://127.0.0.1:8080/sessions \
  -H 'content-type: application/json' \
  -d '{
    "robotId":"robot-lab-01",
    "requestedTracks":["front-camera"],
    "viewerLimit":2,
    "expiresInSeconds":600
  }' | tee evidence/session-create.json
```

From that response, construct the durable invitation that a command channel would carry:

```json
{
  "messageType": "media.session.start.v1",
  "sessionId": "<session-id>",
  "generation": 1,
  "requestedTracks": ["front-camera"],
  "expiresAt": "<timestamp>"
}
```

Validate that the invitation contains none of these keys:

```bash
jq -e '
  (has("token") or has("jwt") or has("sdp") or has("ice") or has("e2eeKey"))
  | not
' evidence/session-invitation.json
```

The robot must redeem the invitation through the authenticated publisher-token operation. A captured invitation alone must be insufficient to join the room.

### 4. Build one publisher and two viewers

Create a small browser client with `livekit-client` that:

1. requests a token from the appropriate API;
2. connects to `ws://127.0.0.1:7880`;
3. when acting as `robot-lab-01`, creates a local video track and publishes it as `front-camera`;
4. when acting as a viewer, subscribes and attaches the received track to a `<video autoplay playsinline>` element; and
5. calls `RTCPeerConnection.getStats()` once per second and records inbound bitrate, RTT, packet loss, jitter, frame dimensions, and frames decoded.

Use a generated canvas track if no webcam is available:

```javascript
const canvas = document.createElement('canvas');
canvas.width = 640; canvas.height = 360;
const ctx = canvas.getContext('2d');
setInterval(() => {
  ctx.fillStyle = '#101828'; ctx.fillRect(0, 0, 640, 360);
  ctx.fillStyle = '#7dd3fc'; ctx.font = '28px sans-serif';
  ctx.fillText(new Date().toISOString(), 28, 190);
}, 100);
const stream = canvas.captureStream(10);
```

Open one publisher tab and two viewer tabs. Confirm that adding viewer two does not create another capture track or another publication from the robot.

### 5. Measure the media path

Record:

- session API acceptance time;
- token issuance time;
- time from room connect to first decoded frame;
- median and p95 RTT over two minutes;
- packet loss and frames dropped;
- robot outbound bitrate with one viewer and then two viewers; and
- browser CPU use.

Save raw samples as `evidence/webrtc-stats.jsonl` and summarize them in `evidence/media-report.md`.

### 6. Upload one durable evidence object

Capture a still frame or create a deterministic test object:

```bash
dd if=/dev/urandom of=evidence/capture.bin bs=1024 count=256
sha256sum evidence/capture.bin | tee evidence/capture.sha256
wc -c evidence/capture.bin | tee evidence/capture.bytes
```

Create an artifact ID before requesting upload authorization. Request a presigned URL for that exact ID, expected digest, byte count, and media type. Upload it:

```bash
curl --fail --request PUT \
  --header 'content-type: application/octet-stream' \
  --upload-file evidence/capture.bin \
  "$UPLOAD_URL"
```

Call the completion operation and require the server to verify bytes and digest before returning `COMPLETE`. Publish only this durable completion event through the control path:

```json
{
  "messageType": "artifact.upload.completed.v1",
  "artifactId": "artifact-<stable-id>",
  "sessionId": "<session-id>",
  "mediaType": "application/octet-stream",
  "sha256": "<digest>",
  "bytes": 262144
}
```

The event contains no bucket credentials, object-store administrative identity, or upload URL.

### 7. Reconcile durable session state

Persist these session states in your API: `REQUESTED`, `CONNECTING`, `ACTIVE`, `TERMINATING`, `TERMINATED`, and `FAILED`.

Do not make a browser or SFU event the durable source of truth. Write a reconciliation loop that compares your session record with the room service, expires overdue sessions, deletes orphan rooms, and ignores events from an older generation.

## Deliberate failure injection

Perform all four experiments and retain evidence.

1. **Wrong role:** use a viewer token in the publisher tab. Pass if publication is denied while subscription still works.
2. **Expired join token:** wait beyond the two-minute TTL before connecting. Pass if a new join fails; do not assume expiry disconnects an already joined participant.
3. **Media-node loss:** stop the LiveKit container during a session. Pass if command handling and artifact metadata remain available, media is reported unavailable, and a replacement room uses generation `2` rather than reusing generation `1`.
4. **Expired upload URL:** wait 61 seconds before uploading. Pass if upload fails, a replacement authorization keeps the same artifact ID, and only a verified object can become `COMPLETE`.

Also submit the same completion event twice. The artifact projection must remain a single record.

## Assignment

Implement a reusable `MediaAndArtifactDemo` that starts an authorized camera session for `robot-lab-01`, serves two viewers, captures one evidence object, uploads it through a scoped URL, and emits a durable completion event. Add automated negative tests for wrong-room, wrong-role, expired-token, bad-digest, and duplicate-completion behavior.

## Measurements

Use this table in `evidence/media-report.md`:

| Measurement | One viewer | Two viewers | Failure run | Notes |
| --- | ---: | ---: | ---: | --- |
| Time to first frame (ms) | | | | |
| Median / p95 RTT (ms) | | | | |
| Robot outbound bitrate (kb/s) | | | | |
| Packet loss (%) | | | | |
| Upload throughput (MiB/s) | | | | |
| Digest verification time (ms) | | | | |

## Evidence and deliverables

- `requirements.lock.txt`, `package-lock.json`, and runtime versions
- redacted session records and invitation payload
- token grant tests without storing live tokens
- browser client source and two-viewer screenshot
- `webrtc-stats.jsonl` and `media-report.md`
- artifact manifest, SHA-256, completion event, and object-store metadata
- failure-injection log with timestamps and expected/observed results
- a one-page boundary diagram showing control, media, telemetry, and artifact paths

## Objective exit criteria

You are done only when all are true:

- [ ] The robot publishes one track that two authorized viewers can receive.
- [ ] The second viewer does not double robot capture or outbound publication count.
- [ ] Viewer credentials cannot publish; publisher credentials cannot subscribe.
- [ ] Wrong-room and expired-token joins are rejected.
- [ ] No invitation or durable record contains a JWT, SDP, ICE credential, E2EE key, or media bytes.
- [ ] Media failure does not stop the command API or corrupt durable session state.
- [ ] Upload authorization is object-scoped and expires.
- [ ] A replacement upload authorization preserves artifact identity.
- [ ] Bad bytes or a bad digest cannot reach `COMPLETE`.
- [ ] Duplicate completion is idempotent.
- [ ] Raw measurements and a reproducible runbook are checked into your learning workspace.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Browser cannot access camera | Use `localhost`, grant browser permission, or use the canvas source. |
| Token is rejected immediately | Confirm API key/secret, system clock, room name, identity, grants, and TTL. |
| Connected but no video | Confirm publisher track name, `canPublish`, viewer `canSubscribe`, autoplay policy, and browser console. |
| Works for one viewer only | Check viewer quota and ensure every viewer has a unique participant identity. |
| Object upload returns 403 | Check URL expiry, HTTP method, exact object key, and signed content type. |
| Completion reports a digest mismatch | Hash the exact uploaded bytes; do not hash base64 or JSON wrapping. |
| Replacement room still accepts old participants | Use a new opaque room name, increment generation, and reject stale-generation events. |

## Next step

Week 13 moves from laptop-only integrations to physical hardware. Keep this week’s media and artifact services optional during first bring-up: they must never be prerequisites for local motion safety.
