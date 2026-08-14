# Week 9 — Edge Adapter and Telemetry Spool

[← Week 8: Behavior-tree inspection mission](week-08-behavior-tree-inspection-mission.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Week 10: Durable command delivery →](week-10-durable-command-delivery.md)

**Time budget:** 10–12 hours

**Primary result:** the course's canonical robot adapter implementation, plus a disk-backed telemetry spool with idempotent replay through a local MQTT broker.

## Outcomes

By the end of this week, you can:

- implement and contract-test a vendor-neutral primitive robot boundary;
- keep `ExecuteMission` in `mission_runner`, above that boundary;
- separate source-protocol adaptation from a stable telemetry contract;
- normalize units, frames, timestamps, quality, and provenance without fabricating values;
- create stable event identities across process and network retries;
- persist telemetry to SQLite before attempting network delivery;
- drain backlog after an outage using MQTT QoS 1; and
- measure backlog age, replay throughput, duplicates, and quarantine volume.

## Prerequisites

- Weeks 1–8 completed; a live robot is optional because the lab supplies deterministic source messages.
- Python 3.10 or newer and basic `argparse`, JSON, and SQL knowledge.
- Understanding that a transport acknowledgement and downstream durable storage are different claims.

## Public readings

- [MQTT 5.0 specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)
- [Eclipse Paho Python client](https://eclipse.dev/paho/files/paho.mqtt.python/html/)
- [SQLite write-ahead logging](https://sqlite.org/wal.html)
- [SQLite transactions](https://sqlite.org/lang_transaction.html)
- [Pydantic models](https://docs.pydantic.dev/latest/concepts/models/)
- [ROS REP 103: units and coordinate conventions](https://www.ros.org/reps/rep-0103.html)
- [ROS REP 105: mobile-platform frames](https://www.ros.org/reps/rep-0105.html)

## Concepts

### Adapt at a narrow boundary

A source adapter knows the source schema and converts it into a versioned, bounded contract. Consumers should not contain branches such as `if vendor == ...`. In this lab one source reports battery as `0..1`, pose in millimetres, and heading in degrees; another reports percentage, metres, and radians. Both become the same representation.

Week 9 is also the canonical implementation milestone. The robot adapter has
only these vendor-neutral primitive semantics:

| Capability | Contract role |
| --- | --- |
| `GetState` or `ObserveState` | Return a freshness-qualified snapshot or stream of robot state. |
| `Halt` or `Stop` | Preempt application-commanded motion through one high-priority safety operation. |
| `NavigateTo` | Start one bounded, idempotent navigation action; cancellation belongs to its action handle. |
| `CaptureImage` | Capture one image under a caller-supplied idempotency ID and return artifact metadata. |

Choose one name from each slash pair in `canonical-adapter-v1`; the alternatives
are naming aliases, not separate behaviors. `ExecuteMission` belongs to
`mission_runner`, which composes these primitives. The canonical package must
not import a mission graph, simulator type, vendor SDK, DDS type, or ROS message.
The normalized telemetry path below is the persisted output side of
`GetState`/`ObserveState`, not a second robot API.

### Preserve meaning and provenance

The normalized event includes units and frame explicitly, plus source schema, boot ID, sequence, observation time, and adapter version. Missing stays missing. Out-of-range or unsafe-to-convert input is quarantined with an error; it is not clamped to a believable value.

### Persist before send

The sequence is:

```text
source event → validate/normalize → SQLite commit → publish → MQTT PUBACK → mark broker_acked
```

If the process dies after publish but before updating SQLite, it republishes the same `event_id`. This is expected at-least-once behavior. A downstream consumer must deduplicate by stable identity.

### Know what `PUBACK` means

For QoS 1, `PUBACK` means the broker accepted the publication under the MQTT protocol. It does not prove that an analytics store, dashboard, or other subscriber persisted the event.

## Environment and packages

```bash
sudo apt update
sudo apt install -y mosquitto mosquitto-clients sqlite3 python3-venv

export LAB_ROOT="$HOME/robotics-learning-lab"
mkdir -p "$LAB_ROOT/week09" "$LAB_ROOT/evidence/week09"
cd "$LAB_ROOT/week09"
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install 'paho-mqtt>=2.1,<3' 'pydantic>=2.7,<3'
python -m pip freeze > "$LAB_ROOT/evidence/week09/requirements-lock.txt"
```

This lab runs a foreground broker on port `1884`, avoiding conflict with any system broker.

## Lab: normalize, spool, disconnect, and replay

### 1. Create the adapter and spool

Save as `$LAB_ROOT/week09/edge_adapter.py`:

```python
#!/usr/bin/env python3
import argparse
from datetime import datetime, timezone
import hashlib
import json
import math
import os
from pathlib import Path
import sqlite3
import sys
import time
import uuid

import paho.mqtt.client as mqtt
from pydantic import BaseModel, ConfigDict, Field, ValidationError


ADAPTER_VERSION = "1.0.0"
EVENT_NAMESPACE = uuid.UUID("c8a47f59-a6e6-4ae7-a600-3fbd05e9a3b4")


class Pose2D(BaseModel):
    model_config = ConfigDict(extra="forbid")
    frame_id: str = Field(min_length=1)
    x_m: float
    y_m: float
    yaw_rad: float = Field(ge=-math.pi, le=math.pi)


class Source(BaseModel):
    model_config = ConfigDict(extra="forbid")
    source_schema: str
    adapter_version: str


class TelemetryEvent(BaseModel):
    model_config = ConfigDict(extra="forbid")
    schema_version: str = "1.0"
    event_id: str
    robot_id: str = Field(min_length=1)
    boot_id: str = Field(min_length=1)
    sequence: int = Field(ge=0)
    observed_at: datetime
    battery_percent: float = Field(ge=0.0, le=100.0)
    pose: Pose2D
    source: Source
    quality_state: str
    warnings: list[str] = []


def utc_now():
    return datetime.now(timezone.utc).isoformat().replace("+00:00", "Z")


def event_id_for(robot_id, boot_id, sequence):
    key = f"{robot_id}:{boot_id}:{sequence}"
    return str(uuid.uuid5(EVENT_NAMESPACE, key))


def normalize_yaw(angle):
    return math.atan2(math.sin(angle), math.cos(angle))


def normalize(native):
    schema = native.get("source_schema")
    robot_id = native["robot_id"]
    boot_id = native["boot_id"]
    sequence = int(native["sequence"])
    common = {
        "event_id": event_id_for(robot_id, boot_id, sequence),
        "robot_id": robot_id,
        "boot_id": boot_id,
        "sequence": sequence,
        "observed_at": native["observed_at"],
        "source": {"source_schema": schema, "adapter_version": ADAPTER_VERSION},
        "quality_state": "GOOD",
        "warnings": [],
    }

    if schema == "alpha_state/v1":
        common["battery_percent"] = float(native["battery_fraction"]) * 100.0
        common["pose"] = {
            "frame_id": native["frame"],
            "x_m": float(native["x_mm"]) / 1000.0,
            "y_m": float(native["y_mm"]) / 1000.0,
            "yaw_rad": normalize_yaw(math.radians(float(native["heading_deg"]))),
        }
    elif schema == "beta_state/v2":
        common["battery_percent"] = float(native["battery_percent"])
        common["pose"] = {
            "frame_id": native["frame_id"],
            "x_m": float(native["x_m"]),
            "y_m": float(native["y_m"]),
            "yaw_rad": normalize_yaw(float(native["yaw_rad"])),
        }
    else:
        raise ValueError(f"unsupported source_schema: {schema!r}")

    return TelemetryEvent.model_validate(common)


def connect_db(path):
    db = sqlite3.connect(path)
    db.execute("PRAGMA journal_mode=WAL")
    db.execute("PRAGMA synchronous=FULL")
    db.executescript("""
      CREATE TABLE IF NOT EXISTS spool (
        event_id TEXT PRIMARY KEY,
        topic TEXT NOT NULL,
        payload TEXT NOT NULL,
        payload_sha256 TEXT NOT NULL,
        state TEXT NOT NULL CHECK(state IN ('pending', 'broker_acked')),
        created_at REAL NOT NULL,
        attempts INTEGER NOT NULL DEFAULT 0,
        acked_at REAL
      );
      CREATE INDEX IF NOT EXISTS spool_pending ON spool(state, created_at);
      CREATE TABLE IF NOT EXISTS quarantine (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        received_at REAL NOT NULL,
        source_payload TEXT NOT NULL,
        error TEXT NOT NULL
      );
    """)
    return db


def spool_native(db, native):
    try:
        event = normalize(native)
        payload = event.model_dump_json()
        digest = hashlib.sha256(payload.encode()).hexdigest()
        with db:
            cursor = db.execute(
                "INSERT OR IGNORE INTO spool "
                "(event_id, topic, payload, payload_sha256, state, created_at) "
                "VALUES (?, ?, ?, ?, 'pending', ?)",
                (event.event_id, f"telemetry/{event.robot_id}", payload, digest, time.time()),
            )
        return "inserted" if cursor.rowcount == 1 else "duplicate"
    except (KeyError, TypeError, ValueError, ValidationError) as error:
        with db:
            db.execute(
                "INSERT INTO quarantine(received_at, source_payload, error) VALUES (?, ?, ?)",
                (time.time(), json.dumps(native, sort_keys=True), str(error)),
            )
        return "quarantined"


def generate(source, count, boot_id):
    for sequence in range(count):
        common = {
            "robot_id": "robot-01",
            "boot_id": boot_id,
            "sequence": sequence,
            "observed_at": utc_now(),
        }
        if source == "alpha":
            yield common | {
                "source_schema": "alpha_state/v1",
                "battery_fraction": 0.90 - sequence * 0.0001,
                "frame": "map",
                "x_mm": sequence * 10,
                "y_mm": 500,
                "heading_deg": 450.0,
            }
        else:
            yield common | {
                "source_schema": "beta_state/v2",
                "battery_percent": 90.0 - sequence * 0.01,
                "frame_id": "map",
                "x_m": sequence * 0.01,
                "y_m": 0.5,
                "yaw_rad": math.pi / 2,
            }


def drain(db, host, port):
    client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2,
                         client_id=f"edge-{os.getpid()}", clean_session=False)
    client.connect(host, port, keepalive=20)
    client.loop_start()
    sent = 0
    try:
        rows = db.execute(
            "SELECT event_id, topic, payload FROM spool "
            "WHERE state='pending' ORDER BY created_at"
        ).fetchall()
        for event_id, topic, payload in rows:
            with db:
                db.execute("UPDATE spool SET attempts=attempts+1 WHERE event_id=?", (event_id,))
            info = client.publish(topic, payload, qos=1, retain=False)
            info.wait_for_publish(timeout=5.0)
            if not info.is_published():
                raise TimeoutError(f"no PUBACK for {event_id}")
            with db:
                db.execute(
                    "UPDATE spool SET state='broker_acked', acked_at=? WHERE event_id=?",
                    (time.time(), event_id),
                )
            sent += 1
    finally:
        client.loop_stop()
        client.disconnect()
    return sent


def print_status(db):
    counts = dict(db.execute(
        "SELECT state, COUNT(*) FROM spool GROUP BY state"
    ).fetchall())
    quarantined = db.execute("SELECT COUNT(*) FROM quarantine").fetchone()[0]
    oldest = db.execute(
        "SELECT MIN(created_at) FROM spool WHERE state='pending'"
    ).fetchone()[0]
    print(json.dumps({
        "pending": counts.get("pending", 0),
        "broker_acked": counts.get("broker_acked", 0),
        "quarantined": quarantined,
        "oldest_pending_age_s": None if oldest is None else round(time.time() - oldest, 3),
    }, sort_keys=True))


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--db", default=os.environ.get("SPOOL_DB", "spool.db"))
    commands = parser.add_subparsers(dest="command", required=True)

    make = commands.add_parser("generate")
    make.add_argument("--source", choices=("alpha", "beta"), required=True)
    make.add_argument("--count", type=int, default=10)
    make.add_argument("--boot-id", default="week09-run1")

    ingest = commands.add_parser("ingest")
    ingest.add_argument("json_file")

    send = commands.add_parser("drain")
    send.add_argument("--host", default="127.0.0.1")
    send.add_argument("--port", type=int, default=1884)
    commands.add_parser("status")

    args = parser.parse_args()
    db = connect_db(args.db)
    try:
        if args.command == "generate":
            results = {"inserted": 0, "duplicate": 0, "quarantined": 0}
            for native in generate(args.source, args.count, args.boot_id):
                outcome = spool_native(db, native)
                results[outcome] += 1
            print(json.dumps(results, sort_keys=True))
        elif args.command == "ingest":
            native = json.loads(Path(args.json_file).read_text(encoding="utf-8"))
            print(spool_native(db, native))
        elif args.command == "drain":
            print(json.dumps({"broker_acked_this_run": drain(db, args.host, args.port)}))
        print_status(db)
    finally:
        db.close()


if __name__ == "__main__":
    main()
```

Then:

```bash
chmod +x edge_adapter.py
python -m py_compile edge_adapter.py
python edge_adapter.py generate --source alpha --count 20 --boot-id week09-a
python edge_adapter.py generate --source beta --count 20 --boot-id week09-b
python edge_adapter.py status
sqlite3 -header -column spool.db \
  "select event_id,state,attempts,json_extract(payload,'$.battery_percent') as battery,json_extract(payload,'$.pose.x_m') as x_m from spool limit 5;"
```

Verify that alpha's `450°` normalizes to approximately `1.5708 rad` and millimetres become metres.

### 2. Prove stable identity and local idempotency

Run the same source boot and sequences again:

```bash
python edge_adapter.py generate --source alpha --count 20 --boot-id week09-a
sqlite3 spool.db "select count(*) from spool;"
```

The second generation reports duplicates, and the total remains 40. Stable identity derives from robot, boot, and sequence; wall-clock send time is deliberately excluded.

### 3. Start the local broker and observer

Terminal 2:

```bash
mosquitto -p 1884 -v
```

Terminal 3:

```bash
mosquitto_sub -h 127.0.0.1 -p 1884 -q 1 -t 'telemetry/+' -v | tee \
  "$LAB_ROOT/evidence/week09/subscriber.log"
```

Terminal 1:

```bash
source "$LAB_ROOT/week09/.venv/bin/activate"
cd "$LAB_ROOT/week09"
python edge_adapter.py drain
python edge_adapter.py status
```

All 40 rows should become `broker_acked`. Inspect one observed JSON event.

### 4. Exercise offline spool and replay

Stop the foreground broker with `Ctrl-C`. Generate a backlog and attempt delivery:

```bash
python edge_adapter.py generate --source alpha --count 100 --boot-id outage-1
python edge_adapter.py status
python edge_adapter.py drain || true
python edge_adapter.py status
```

The drain fails, but all new rows remain `pending`. Start the broker and subscriber again, then measure replay:

```bash
/usr/bin/time -f 'elapsed_s=%e max_rss_kb=%M' \
  python edge_adapter.py drain 2>&1 | tee \
  "$LAB_ROOT/evidence/week09/replay-timing.txt"
python edge_adapter.py status
```

### 5. Quarantine unsafe input

Create `$LAB_ROOT/week09/bad-native.json`:

```json
{
  "source_schema": "alpha_state/v1",
  "robot_id": "robot-01",
  "boot_id": "bad-run",
  "sequence": 1,
  "observed_at": "2026-01-01T00:00:00Z",
  "battery_fraction": 1.5,
  "frame": "map",
  "x_mm": 100,
  "y_mm": 200,
  "heading_deg": 0
}
```

Ingest and inspect it:

```bash
python edge_adapter.py ingest bad-native.json
sqlite3 -header -column spool.db \
  "select id,datetime(received_at,'unixepoch') as received,error from quarantine;"
```

The out-of-range battery must be quarantined. It must not appear as `100` or `150` in telemetry.

## Assignment

Complete both halves of the Week 9 boundary:

1. Create a `canonical_adapter` package whose versioned interface contains only
   `GetState` or `ObserveState`, `Halt` or `Stop`, `NavigateTo`, and
   `CaptureImage`.
2. Implement a deterministic fake backend and a simulator backend. Run the same
   contract suite against both, including unsupported capability, stale state,
   duplicate operation ID, navigation cancellation, capture digest, and
   high-priority halt cases.
3. Keep `ExecuteMission` in a separate `mission_runner` package. Add a dependency
   test showing the runner may import the adapter contract while the adapter
   cannot import the runner or any mission graph.
4. Replace one synthetic telemetry source with `/odom` or a replay of the Week 8
   bag. Use the message header timestamp, preserve its frame, and keep a sequence
   scoped to a persisted boot/session ID.
5. Normalize position and yaw into the same event contract. If battery is
   unavailable, version the schema to make it optional; do not invent a value.
6. Spool at least 500 `ObserveState`-derived events through a 60-second broker
   outage and drain them afterward.

Document the source-to-normalized mapping and the canonical-to-simulator
operation mapping. Include source units, target units, frame, missing-value
behavior, validation rule, and the exact layer that owns each translation.

## Failure injection

Complete these experiments:

1. **Broker outage:** the demonstrated outage must leave every generated event pending and replay it later.
2. **Adapter crash during replay:** generate at least 5,000 pending events, start drain, and terminate the adapter during replay. Restart it. Some events may be seen twice by the subscriber, but event IDs must remain identical and the spool must converge to zero pending.
3. **Malformed or unsafe source:** submit a missing field, unknown schema, invalid timestamp, or out-of-range value. It must enter quarantine with safe diagnostic text.
4. **Duplicate input:** submit the same boot/sequence twice. The spool count must not increase.

Do not use `kill -9` on a physical robot control process. This lab process only handles telemetry.

## Measurements and deliverables

Keep under `$LAB_ROOT/evidence/week09/`:

- dependency lock file and adapter source;
- the versioned canonical contract, fake and simulator backends, and unchanged
  contract-test report for both;
- a dependency test proving `ExecuteMission` exists only in `mission_runner`;
- source-to-normalized mapping table;
- five sample normalized events from each source;
- SQLite schema and status output before/during/after outage;
- subscriber log and replay timing;
- quarantine examples;
- crash-replay duplicate counts grouped by `event_id`; and
- the ROS-backed 500-event test report.

Useful queries:

```bash
sqlite3 -header -column "$LAB_ROOT/week09/spool.db" \
  "select state,count(*) as events,min(attempts) as min_attempts,max(attempts) as max_attempts from spool group by state;"

sqlite3 "$LAB_ROOT/week09/spool.db" \
  "select round(max(0,strftime('%s','now')-min(created_at)),1) from spool where state='pending';"
```

Measure:

- normalization/quarantine counts;
- maximum pending events and oldest-pending age;
- replay events per second;
- events observed more than once versus distinct `event_id` values; and
- time from broker restart until backlog reaches zero.

## Objective exit criteria

Advance only when:

- the Week 9 `canonical_adapter` exposes only the four primitive capability
  semantics and contains no `ExecuteMission` or mission-tree dependency;
- unchanged contract tests pass against fake and simulator implementations;
- unsupported operations fail before backend dispatch, duplicate IDs produce
  one effect, and `Halt`/`Stop` can preempt an active `NavigateTo`;
- two incompatible source schemas produce the same validated telemetry schema and units;
- rerunning identical source events does not add spool rows;
- no event is marked `broker_acked` before a QoS 1 publish completes;
- a 60-second outage loses zero persisted events and drains without changing event IDs;
- a replay crash may duplicate transport delivery but loses zero logical events;
- invalid data is quarantined rather than clamped or fabricated; and
- a reviewer can trace a normalized event back to source schema, boot, sequence, frame, and observation time.

## Troubleshooting

| Symptom | Diagnostic | Correction |
| --- | --- | --- |
| `ConnectionRefusedError` | `ss -ltn | grep 1884` | Start the foreground broker or use the configured host/port. |
| Database is locked | Check for multiple writers and long shell transactions | Keep transactions short; use one adapter writer; preserve WAL mode. |
| Events change ID on retry | Compare robot, boot, and sequence inputs | Derive identity from stable source identity, never publish time. |
| Subscriber sees duplicates | Group by `event_id` | Expected after an ambiguous crash; implement consumer deduplication rather than random IDs. |
| Battery becomes plausible after bad input | Inspect normalization code | Reject/quarantine; never silently clamp semantic errors. |
| Backlog grows while connected | Inspect `attempts`, broker log, publish timeout, and disk | Fix connectivity or throughput; preserve pending rows until acknowledged. |

## Next step

Week 10 applies the same at-least-once reasoning to commands, where duplicates can cause physical side effects. You will add durable delivery, idempotent execution, acceptance receipts, terminal results, and crash recovery with a local persistent message stream.

[← Week 8: Behavior-tree inspection mission](week-08-behavior-tree-inspection-mission.md) · [Week index](README.md) · [Week 10: Durable command delivery →](week-10-durable-command-delivery.md)
