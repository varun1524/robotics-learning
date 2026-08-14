# Week 10 — Durable Command Delivery

[← Week 9: Edge adapter and telemetry spool](week-09-edge-adapter-and-telemetry-spool.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Week 11: Map versions and releases →](week-11-map-versions-and-releases.md)

**Time budget:** 10–12 hours

**Primary result:** a crash-tested, at-least-once command path with idempotent effects, acceptance receipts, and terminal results.

## Outcomes

By the end of this week, you can:

- explain why broker persistence, consumer acknowledgement, command acceptance, execution, and completion are separate facts;
- use a durable NATS JetStream consumer with explicit acknowledgements and redelivery;
- create stable command and idempotency identities;
- persist an execution ledger before applying a command;
- make a simulated side effect idempotent across duplicate delivery and process crashes; and
- publish deterministic acceptance receipts and terminal results that observers can deduplicate.

## Prerequisites

- Week 9 exit criteria met.
- Python, JSON, SQLite transactions, and basic asynchronous programming.
- Docker or Podman for a local NATS server, or a locally installed `nats-server` binary.
- This lab controls only a simulated effect table. Do not connect experimental retry logic to motors, doors, elevators, or other physical actuators.

## Public readings

- [NATS JetStream model](https://docs.nats.io/nats-concepts/jetstream)
- [JetStream consumers and acknowledgements](https://docs.nats.io/nats-concepts/jetstream/consumers)
- [nats.py JetStream examples](https://github.com/nats-io/nats.py/tree/main/examples/jetstream)
- [CloudEvents specification](https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md)
- [HTTP semantics: safe and idempotent methods](https://www.rfc-editor.org/rfc/rfc9110.html#name-idempotent-methods)
- [SQLite atomic commit](https://sqlite.org/atomiccommit.html)

## Concepts

### Delivery state is not execution state

Use precise language:

| Event | What it proves | What it does not prove |
| --- | --- | --- |
| Stream publish acknowledgement | Broker durably accepted the command | Robot read or accepted it |
| Consumer delivery | Worker received a copy | Worker persisted or executed it |
| `ACCEPTED` receipt | Worker durably recorded and validated it | Effect completed |
| Consumer acknowledgement | This delivery need not be redelivered | Business result succeeded |
| Terminal result | Command reached `SUCCEEDED`, `FAILED`, `REJECTED`, or `EXPIRED` | An observer received the result exactly once |

### At-least-once means duplicates are normal

A worker may finish a side effect and crash before acknowledging the message. The stream must redeliver. The worker must recognize the stable identity, avoid repeating the effect, republish the cached result, and then acknowledge.

### Idempotency is a state-machine property

An idempotency key is not enough by itself. Persist the key, payload hash, state, effect evidence, and result atomically. Reusing a key with different command content is a conflict, not a retry.

### Commands are bounded and typed

This lab accepts one allowlisted operation, `inspect_checkpoint`, with structured arguments and an expiry. Never treat a command payload as a shell command or executable code.

## Environment and packages

```bash
export LAB_ROOT="$HOME/robotics-learning-lab"
mkdir -p "$LAB_ROOT/week10" "$LAB_ROOT/evidence/week10"
cd "$LAB_ROOT/week10"

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install 'nats-py>=2.8,<3'
python -m pip freeze > "$LAB_ROOT/evidence/week10/requirements-lock.txt"
```

Start a persistent local server with Docker:

```bash
docker volume create robotics-nats-data
docker run --rm --name robotics-nats \
  -p 4222:4222 -p 8222:8222 \
  -v robotics-nats-data:/data \
  nats:2.11-alpine -js -sd /data -m 8222
```

With Podman, replace `docker` with `podman`. With a native binary, run:

```bash
nats-server -js -sd "$LAB_ROOT/week10/nats-data" -m 8222
```

## Lab: command, receipt, effect, result

### 1. Create the command harness

Save as `$LAB_ROOT/week10/command_lab.py`:

```python
#!/usr/bin/env python3
import argparse
import asyncio
from datetime import datetime, timezone
import hashlib
import json
import os
from pathlib import Path
import sqlite3
import time
import uuid

import nats
from nats.errors import TimeoutError as NatsTimeoutError
from nats.js.api import AckPolicy, ConsumerConfig, RetentionPolicy, StorageType
from nats.js.errors import NotFoundError


NATS_URL = os.environ.get("NATS_URL", "nats://127.0.0.1:4222")
DB_PATH = os.environ.get("COMMAND_DB", "command-ledger.db")


def now_iso():
    return datetime.now(timezone.utc).isoformat().replace("+00:00", "Z")


def canonical_hash(command):
    executable = {
        "operation": command["operation"],
        "arguments": command["arguments"],
        "robot_id": command["robot_id"],
    }
    encoded = json.dumps(executable, sort_keys=True, separators=(",", ":")).encode()
    return hashlib.sha256(encoded).hexdigest()


def open_db():
    db = sqlite3.connect(DB_PATH)
    db.row_factory = sqlite3.Row
    db.execute("PRAGMA journal_mode=WAL")
    db.execute("PRAGMA synchronous=FULL")
    db.executescript("""
      CREATE TABLE IF NOT EXISTS commands (
        command_id TEXT PRIMARY KEY,
        idempotency_key TEXT NOT NULL UNIQUE,
        payload_hash TEXT NOT NULL,
        state TEXT NOT NULL,
        command_json TEXT NOT NULL,
        result_json TEXT,
        accepted_at REAL NOT NULL,
        completed_at REAL
      );
      CREATE TABLE IF NOT EXISTS effects (
        command_id TEXT PRIMARY KEY,
        effect_kind TEXT NOT NULL,
        effect_json TEXT NOT NULL,
        applied_at REAL NOT NULL,
        FOREIGN KEY(command_id) REFERENCES commands(command_id)
      );
    """)
    return db


async def ensure_stream(js, name, subjects):
    try:
        await js.stream_info(name)
    except NotFoundError:
        await js.add_stream(
            name=name,
            subjects=subjects,
            retention=RetentionPolicy.LIMITS,
            storage=StorageType.FILE,
        )


async def setup():
    nc = await nats.connect(NATS_URL)
    js = nc.jetstream()
    await ensure_stream(js, "COMMANDS", ["commands.*"])
    await ensure_stream(js, "COMMAND_EVENTS", ["receipts.*", "results.*"])
    print("streams ready: COMMANDS, COMMAND_EVENTS")
    await nc.drain()


async def emit(js, subject, event):
    event["recorded_at"] = now_iso()
    await js.publish(subject, json.dumps(event, sort_keys=True).encode())


async def publish_cached(js, row):
    accepted = {
        "event_id": f"receipt:{row['command_id']}:accepted",
        "event_type": "command_receipt",
        "command_id": row["command_id"],
        "stage": "ACCEPTED",
    }
    await emit(js, f"receipts.{json.loads(row['command_json'])['robot_id']}", accepted)
    if row["result_json"]:
        result = json.loads(row["result_json"])
        await js.publish(
            f"results.{json.loads(row['command_json'])['robot_id']}",
            json.dumps(result, sort_keys=True).encode(),
        )


async def process_message(js, msg, db):
    command = json.loads(msg.data)
    required = {"command_id", "idempotency_key", "robot_id", "operation",
                "arguments", "expires_at_unix_s"}
    missing = required - command.keys()
    if missing:
        await emit(js, "results.unknown", {
            "event_id": f"result:malformed:{uuid.uuid4()}",
            "event_type": "command_result",
            "state": "REJECTED",
            "reason": f"missing fields: {sorted(missing)}",
        })
        await msg.ack()
        return

    command_id = command["command_id"]
    idem = command["idempotency_key"]
    digest = canonical_hash(command)
    existing = db.execute(
        "SELECT * FROM commands WHERE command_id=? OR idempotency_key=?",
        (command_id, idem),
    ).fetchone()

    if existing:
        if (existing["command_id"] != command_id or
                existing["idempotency_key"] != idem or
                existing["payload_hash"] != digest):
            await emit(js, f"results.{command['robot_id']}", {
                "event_id": f"result:{command_id}:conflict",
                "event_type": "command_result",
                "command_id": command_id,
                "state": "REJECTED",
                "reason": "idempotency identity reused with different content",
            })
            await msg.ack()
            return
        await publish_cached(js, existing)
        if existing["result_json"]:
            await msg.ack()
            return
    else:
        if command["operation"] != "inspect_checkpoint":
            await emit(js, f"results.{command['robot_id']}", {
                "event_id": f"result:{command_id}:unsupported",
                "event_type": "command_result",
                "command_id": command_id,
                "state": "REJECTED",
                "reason": "operation is not allowlisted",
            })
            await msg.ack()
            return
        if float(command["expires_at_unix_s"]) <= time.time():
            await emit(js, f"results.{command['robot_id']}", {
                "event_id": f"result:{command_id}:expired",
                "event_type": "command_result",
                "command_id": command_id,
                "state": "EXPIRED",
            })
            await msg.ack()
            return

        with db:
            db.execute(
                "INSERT INTO commands(command_id,idempotency_key,payload_hash,state,"
                "command_json,accepted_at) VALUES (?,?,?,?,?,?)",
                (command_id, idem, digest, "ACCEPTED",
                 json.dumps(command, sort_keys=True), time.time()),
            )
        existing = db.execute(
            "SELECT * FROM commands WHERE command_id=?", (command_id,)
        ).fetchone()
        await publish_cached(js, existing)

    if os.environ.get("CRASH_POINT") == "after_accept":
        os._exit(70)

    # The effect and terminal state commit in one local transaction. The
    # simulated effect is an evidence row, not arbitrary code execution.
    result = {
        "event_id": f"result:{command_id}:terminal",
        "event_type": "command_result",
        "command_id": command_id,
        "idempotency_key": idem,
        "robot_id": command["robot_id"],
        "state": "SUCCEEDED",
        "output": {
            "checkpoint": command["arguments"].get("checkpoint"),
            "observation": "CLEAR",
        },
        "completed_at": now_iso(),
    }
    result_json = json.dumps(result, sort_keys=True)
    with db:
        db.execute(
            "INSERT OR IGNORE INTO effects(command_id,effect_kind,effect_json,applied_at) "
            "VALUES (?,?,?,?)",
            (command_id, "inspection_record",
             json.dumps(result["output"], sort_keys=True), time.time()),
        )
        db.execute(
            "UPDATE commands SET state='SUCCEEDED', result_json=?, completed_at=? "
            "WHERE command_id=?",
            (result_json, time.time(), command_id),
        )

    if os.environ.get("CRASH_POINT") == "after_effect_commit":
        os._exit(71)

    await js.publish(f"results.{command['robot_id']}", result_json.encode())
    await msg.ack()


async def worker(robot_id):
    nc = await nats.connect(NATS_URL)
    js = nc.jetstream()
    subject = f"commands.{robot_id}"
    consumer = ConsumerConfig(
        durable_name=f"{robot_id}-worker",
        filter_subject=subject,
        ack_policy=AckPolicy.EXPLICIT,
        ack_wait=5.0,
        max_deliver=10,
    )
    sub = await js.pull_subscribe(
        subject, durable=f"{robot_id}-worker", stream="COMMANDS", config=consumer
    )
    db = open_db()
    print(f"worker ready for {subject}; db={DB_PATH}", flush=True)
    try:
        while True:
            try:
                messages = await sub.fetch(1, timeout=1.0)
            except NatsTimeoutError:
                continue
            for message in messages:
                delivered = message.metadata.num_delivered if message.metadata else None
                print(f"delivery count={delivered} bytes={len(message.data)}", flush=True)
                await process_message(js, message, db)
    finally:
        db.close()
        await nc.drain()


async def send(args):
    command_id = args.command_id or str(uuid.uuid4())
    idem = args.idempotency_key or f"inspect:{args.robot}:{args.checkpoint}"
    command = {
        "schema_version": "1.0",
        "command_id": command_id,
        "idempotency_key": idem,
        "robot_id": args.robot,
        "operation": args.operation,
        "arguments": {"checkpoint": args.checkpoint},
        "issued_at": now_iso(),
        "expires_at_unix_s": time.time() + args.ttl,
    }
    nc = await nats.connect(NATS_URL)
    ack = await nc.jetstream().publish(
        f"commands.{args.robot}", json.dumps(command, sort_keys=True).encode()
    )
    print(json.dumps(command, sort_keys=True))
    print(f"stream={ack.stream} sequence={ack.seq}")
    await nc.drain()


async def watch():
    nc = await nats.connect(NATS_URL)

    async def show(message):
        print(f"{message.subject} {message.data.decode()}", flush=True)

    await nc.subscribe("receipts.>", cb=show)
    await nc.subscribe("results.>", cb=show)
    print("watching receipts.> and results.>", flush=True)
    while True:
        await asyncio.sleep(3600)


def inspect_db():
    db = open_db()
    for table in ("commands", "effects"):
        print(f"[{table}]")
        rows = db.execute(f"SELECT * FROM {table} ORDER BY rowid").fetchall()
        for row in rows:
            print(json.dumps(dict(row), sort_keys=True))
    db.close()


def main():
    parser = argparse.ArgumentParser()
    commands = parser.add_subparsers(dest="command", required=True)
    commands.add_parser("setup")
    work = commands.add_parser("worker")
    work.add_argument("--robot", default="robot-01")
    publish = commands.add_parser("send")
    publish.add_argument("--robot", default="robot-01")
    publish.add_argument("--checkpoint", required=True)
    publish.add_argument("--operation", default="inspect_checkpoint")
    publish.add_argument("--ttl", type=float, default=300.0)
    publish.add_argument("--command-id")
    publish.add_argument("--idempotency-key")
    commands.add_parser("watch")
    commands.add_parser("inspect")
    args = parser.parse_args()

    if args.command == "setup":
        asyncio.run(setup())
    elif args.command == "worker":
        asyncio.run(worker(args.robot))
    elif args.command == "send":
        asyncio.run(send(args))
    elif args.command == "watch":
        asyncio.run(watch())
    else:
        inspect_db()


if __name__ == "__main__":
    main()
```

Make it executable and check syntax:

```bash
chmod +x command_lab.py
python -m py_compile command_lab.py
python command_lab.py setup
curl -fsS http://127.0.0.1:8222/jsz | python -m json.tool | sed -n '1,80p'
```

### 2. Observe a normal command lifecycle

Terminal 2:

```bash
cd "$LAB_ROOT/week10"
source .venv/bin/activate
python command_lab.py watch | tee "$LAB_ROOT/evidence/week10/events.log"
```

Terminal 3:

```bash
cd "$LAB_ROOT/week10"
source .venv/bin/activate
python command_lab.py worker --robot robot-01
```

Terminal 1:

```bash
cd "$LAB_ROOT/week10"
source .venv/bin/activate
python command_lab.py send \
  --robot robot-01 \
  --checkpoint aisle-3 \
  --command-id 11111111-1111-4111-8111-111111111111 \
  --idempotency-key mission-42:aisle-3
python command_lab.py inspect | tee "$LAB_ROOT/evidence/week10/normal-ledger.txt"
```

Expected order is stream publish acknowledgement, `ACCEPTED` receipt, and terminal `SUCCEEDED` result. The ledger contains one command and one effect.

### 3. Retry the exact command

Run the same `send` command again. The worker may emit the deterministic receipt and cached result again, but:

```bash
sqlite3 command-ledger.db \
  "select count(*) as effects from effects where command_id='11111111-1111-4111-8111-111111111111';"
```

must return `1`.

### 4. Crash after durable acceptance

Use a new identity. Stop the normal worker, then run:

```bash
CRASH_POINT=after_accept python command_lab.py worker --robot robot-01
```

Send:

```bash
python command_lab.py send \
  --robot robot-01 --checkpoint loading-bay \
  --command-id 22222222-2222-4222-8222-222222222222 \
  --idempotency-key mission-42:loading-bay
```

The worker exits with status 70 without acknowledging the delivery. Wait at least five seconds, then restart without `CRASH_POINT`:

```bash
python command_lab.py worker --robot robot-01
```

The message is redelivered, resumes from `ACCEPTED`, creates one effect, publishes a result, and is acknowledged.

### 5. Crash after effect commit but before result/ack

Repeat with a third identity and:

```bash
CRASH_POINT=after_effect_commit python command_lab.py worker --robot robot-01
```

After the crash, inspect SQLite: the effect and terminal result are already committed. Restart normally after five seconds. The duplicate delivery must publish the cached result and acknowledge without creating a second effect.

## Assignment

Connect the Week 8 mission runner to this contract in simulation:

1. Dispatch an `inspect_checkpoint` command containing a named checkpoint and a pinned mission ID.
2. Have a robot-side worker invoke the mission runner or a safe stub through a typed function—not a shell string from the message.
3. Emit `ACCEPTED`, periodic progress, and one terminal result.
4. Persist enough state to resume or conclude after restart.
5. Add a bounded cancellation command with its own identity and result.
6. State which operations are safe to retry, which need a stronger effect-specific guard, and which must never be retried automatically.

Run ten commands with at least three deliberate duplicate publishes and two worker crashes.

## Failure injection

Complete and document:

1. **Crash after acceptance:** expect redelivery and one effect.
2. **Crash after effect commit:** expect cached-result replay and one effect.
3. **Identity conflict:** reuse `mission-42:aisle-3` with a different checkpoint or command ID. Expect `REJECTED` and no effect.
4. **Expired command:** send with `--ttl -1`. Expect `EXPIRED`, no acceptance, and no effect.
5. **Unsupported operation:** use `--operation run_shell`. Expect `REJECTED`; no supplied string is executed.
6. **Broker restart:** stop and restart the NATS container with the named volume. A published unacknowledged command must still be available.

## Measurements and deliverables

Store under `$LAB_ROOT/evidence/week10/`:

- harness source and dependency lock;
- normal and failure event logs;
- ledger dumps after each crash point;
- stream/consumer monitoring snapshots;
- a table of command ID, idempotency key, delivery count, receipt count, result count, terminal state, and effect count;
- latency from broker publish to `ACCEPTED` and from `ACCEPTED` to terminal result;
- the ten-command assignment report; and
- a state diagram covering new, accepted, terminal, expired, rejected, and duplicate delivery.

Use these invariants as queries:

```bash
sqlite3 -header -column command-ledger.db \
  "select c.command_id,c.state,count(e.command_id) as effects from commands c left join effects e using(command_id) group by c.command_id,c.state order by c.accepted_at;"

sqlite3 command-ledger.db \
  "select count(*) from (select idempotency_key from commands group by idempotency_key having count(*) > 1);"
```

Both duplicate-effect and duplicate-idempotency-key violations must be zero.

## Objective exit criteria

Advance only when:

- a publish acknowledgement, `ACCEPTED` receipt, and terminal result are separately observable;
- every worker acknowledgement occurs after a durable terminal result or explicit rejection/expiry decision;
- exact command retry produces one ledger command and one simulated effect;
- both crash points recover through redelivery with one effect;
- deterministic receipt/result IDs allow observers to deduplicate event replay;
- expired, conflicting, and unsupported commands create no effect; and
- all retry loops have expiry, delivery, timeout, or attempt bounds.

## Troubleshooting

| Symptom | Diagnostic | Correction |
| --- | --- | --- |
| NATS connection refused | `curl http://127.0.0.1:8222/varz` | Start the server and verify port mapping. |
| Stream setup fails | Inspect `/jsz` and server log | Confirm JetStream is enabled with `-js` and file storage is writable. |
| Crash command never redelivers | Wait beyond `ack_wait`; inspect durable consumer | Use the same durable name and subject; do not acknowledge before the crash point. |
| Duplicate effects appear | Query `effects` primary key and transaction order | Guard the effect with stable command identity and commit effect plus result atomically. |
| Results repeat | Compare deterministic `event_id` | Expected under at-least-once publication; deduplicate observers by event ID. |
| Worker rejects a retry as conflict | Compare command ID, idempotency key, robot, operation, and arguments | Retries must reuse the exact semantic identity and payload. |

## Next step

Week 11 applies the same immutable-identity discipline to maps. You will package
observed geometry as a content-addressed `MapVersion`, compose exactly one
version into an immutable approved `MapRelease`, move an optional channel
between releases, pin mission runs by release ID plus digest, and prove rollback
without overwriting history.

[← Week 9: Edge adapter and telemetry spool](week-09-edge-adapter-and-telemetry-spool.md) · [Week index](README.md) · [Week 11: Map versions and releases →](week-11-map-versions-and-releases.md)
