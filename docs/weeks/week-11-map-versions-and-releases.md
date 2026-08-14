# Week 11 — Map Versions and Releases

[← Week 10: Durable command delivery](week-10-durable-command-delivery.md) · [Week index](README.md) · [Repository workflow](../repository-workflow.md) · [Week 12: WebRTC media and artifacts →](week-12-webrtc-media-and-artifacts.md)

**Time budget:** 9–12 hours

**Primary result:** a content-addressed registry in which observed geometry is
an immutable MapVersion, approved composition is an immutable MapRelease, an
optional channel points to a release, and every mission run pins one release
ID and digest.

## Outcomes

By the end of this week, you can:

- distinguish a mutable SLAM session, immutable MapVersion, immutable
  MapRelease, mutable release channel, and immutable run pin;
- package map artifacts, metadata, checksums, and provenance into one version;
- compose exactly one version with alignment, overlays, policies, and routes;
- apply quantitative gates before promotion;
- resolve a channel only before assignment and pin the selected release; and
- detect corruption and roll back without deleting history or changing a run.

## Prerequisites

- Week 6 map YAML/image pair and, when available, its native pose graph.
- Week 7 navigation evidence for that map.
- Python, JSON, YAML, hashing, and atomic-write basics.
- About 100 MB of local storage.

## Public readings

- [Nav2 map server](https://docs.nav2.org/configuration/packages/map_server/configuring-map-server.html)
- [ROS OccupancyGrid](https://docs.ros.org/en/jazzy/p/nav_msgs/msg/OccupancyGrid.html)
- [The Update Framework](https://theupdateframework.github.io/specification/latest/)
- [Open Container Initiative descriptor digest model](https://github.com/opencontainers/image-spec/blob/main/descriptor.md)
- [RFC 8785 JSON canonicalization](https://www.rfc-editor.org/rfc/rfc8785)
- [Python hashlib](https://docs.python.org/3/library/hashlib.html)

## Concepts

### Five objects with different lifecycles

| Object | Mutable? | Meaning |
| --- | --- | --- |
| SLAM session | Yes | Working pose graph, tuning, and survey state |
| MapVersion | No | Observed native/canonical geometry, metadata, checksums, provenance |
| MapRelease | No | Exactly one MapVersion plus approved alignment, overlays, policies, routes |
| Release channel | Yes, audited | Optional selector such as staging that points to a release |
| Mission-run pin | No | Exact MapRelease ID/digest and MapVersion frozen at assignment |

Do not call a mutable channel a release. Do not pin a run to a bare
MapVersion: the MapRelease contains the approved policy context needed to
interpret its geometry.

### Geometry and composition changes are different

A new survey, changed walls, or materially changed traversability creates a
new MapVersion, followed by a reviewed MapRelease. A corrected reference
alignment, no-go zone, speed zone, dock, route, or inspection pose creates a
new MapRelease over the same MapVersion.

### Resolve once, then pin

A channel is a pre-assignment convenience. Resolve it before creating a run,
copy the selected MapRelease ID and digest into the execution bundle, and never
read the channel during execution. Moving or rolling back the channel affects
only later assignments.

## Environment and packages

~~~bash
sudo apt update
sudo apt install -y python3-venv

export LAB_ROOT="$HOME/robotics-learning-lab"
mkdir -p "$LAB_ROOT/week11" "$LAB_ROOT/evidence/week11"
cd "$LAB_ROOT/week11"
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install 'PyYAML>=6,<7' 'Pillow>=10,<12'
python -m pip freeze > "$LAB_ROOT/evidence/week11/requirements-lock.txt"
~~~

## Lab: version geometry, release composition, pin, and roll back

### 1. Create the local registry

Save this as mapctl.py:

~~~python
#!/usr/bin/env python3
import argparse
from datetime import datetime, timezone
import hashlib
import json
import os
from pathlib import Path
import shutil
import tempfile

from PIL import Image
import yaml


MAP_FIELDS = (
    "image", "resolution", "origin", "negate", "occupied_thresh", "free_thresh"
)
COMPOSITION_FIELDS = ("alignment", "overlays", "policies", "routes")


def now_iso():
    return datetime.now(timezone.utc).isoformat().replace("+00:00", "Z")


def canonical(value):
    return json.dumps(value, sort_keys=True, separators=(",", ":")).encode()


def digest_bytes(value):
    return hashlib.sha256(value).hexdigest()


def digest_file(path):
    digest = hashlib.sha256()
    with Path(path).open("rb") as stream:
        for block in iter(lambda: stream.read(1024 * 1024), b""):
            digest.update(block)
    return digest.hexdigest()


def open_registry(root):
    root = Path(root).resolve()
    for child in ("versions", "releases", "channels", "pins"):
        (root / child).mkdir(parents=True, exist_ok=True)
    return root


def atomic_json(path, value):
    path = Path(path)
    path.parent.mkdir(parents=True, exist_ok=True)
    fd, temporary = tempfile.mkstemp(prefix=path.name + ".", dir=path.parent)
    try:
        with os.fdopen(fd, "w", encoding="utf-8") as stream:
            json.dump(value, stream, indent=2, sort_keys=True)
            stream.write("\n")
            stream.flush()
            os.fsync(stream.fileno())
        os.replace(temporary, path)
    finally:
        if os.path.exists(temporary):
            os.unlink(temporary)


def load_source(yaml_path):
    yaml_path = Path(yaml_path).resolve()
    metadata = yaml.safe_load(yaml_path.read_text(encoding="utf-8"))
    missing = [name for name in MAP_FIELDS if name not in metadata]
    if missing:
        raise ValueError(f"map YAML missing fields: {missing}")
    image = Path(metadata["image"])
    if not image.is_absolute():
        image = yaml_path.parent / image
    image = image.resolve()
    if not image.is_file():
        raise FileNotFoundError(f"map image not found: {image}")
    if float(metadata["resolution"]) <= 0 or len(metadata["origin"]) != 3:
        raise ValueError("invalid map resolution or origin")
    free = float(metadata["free_thresh"])
    occupied = float(metadata["occupied_thresh"])
    if not 0 <= free < occupied <= 1:
        raise ValueError("thresholds must satisfy 0 <= free < occupied <= 1")
    return metadata, image


def verify_version(root, version_id):
    directory = root / "versions" / version_id
    manifest_path = directory / "manifest.json"
    if not manifest_path.is_file():
        raise FileNotFoundError(f"unknown MapVersion: {version_id}")
    manifest = json.loads(manifest_path.read_text(encoding="utf-8"))
    descriptor = manifest["descriptor"]
    if descriptor.get("kind") != "MapVersion":
        raise ValueError("object is not a MapVersion")
    if digest_bytes(canonical(descriptor)) != version_id:
        raise ValueError("MapVersion descriptor digest mismatch")
    if manifest["mapVersionId"] != version_id:
        raise ValueError("MapVersion ID mismatch")
    for name, expected in descriptor["files"].items():
        path = directory / name
        if not path.is_file() or digest_file(path) != expected["sha256"]:
            raise ValueError(f"MapVersion file mismatch: {name}")
    return manifest


def ingest(root, map_id, yaml_path, provenance):
    metadata, source_image = load_source(yaml_path)
    image_name = "map" + source_image.suffix.lower()
    packaged = {key: metadata[key] for key in metadata if key != "image"}
    packaged["image"] = image_name
    yaml_data = yaml.safe_dump(packaged, sort_keys=True).encode()
    descriptor = {
        "kind": "MapVersion",
        "schemaVersion": 1,
        "mapId": map_id,
        "semantic": packaged,
        "files": {
            "map.yaml": {"sha256": digest_bytes(yaml_data)},
            image_name: {"sha256": digest_file(source_image)},
        },
        "provenance": provenance,
    }
    version_id = digest_bytes(canonical(descriptor))
    target = root / "versions" / version_id
    if target.exists():
        verify_version(root, version_id)
        print(version_id)
        return
    staged = Path(tempfile.mkdtemp(prefix="map-version-", dir=root / "versions"))
    try:
        (staged / "map.yaml").write_bytes(yaml_data)
        shutil.copyfile(source_image, staged / image_name)
        (staged / "manifest.json").write_text(
            json.dumps(
                {
                    "mapVersionId": version_id,
                    "createdAt": now_iso(),
                    "descriptor": descriptor,
                },
                indent=2,
                sort_keys=True,
            ) + "\n",
            encoding="utf-8",
        )
        os.replace(staged, target)
    finally:
        if staged.exists():
            shutil.rmtree(staged)
    verify_version(root, version_id)
    print(version_id)


def load_composition(path):
    value = json.loads(Path(path).read_text(encoding="utf-8"))
    missing = [name for name in COMPOSITION_FIELDS if name not in value]
    if missing:
        raise ValueError(f"release composition missing fields: {missing}")
    return value


def verify_release(root, release_id):
    path = root / "releases" / f"{release_id}.json"
    if not path.is_file():
        raise FileNotFoundError(f"unknown MapRelease: {release_id}")
    manifest = json.loads(path.read_text(encoding="utf-8"))
    descriptor = manifest["descriptor"]
    if descriptor.get("kind") != "MapRelease":
        raise ValueError("object is not a MapRelease")
    if digest_bytes(canonical(descriptor)) != release_id:
        raise ValueError("MapRelease descriptor digest mismatch")
    if manifest["mapReleaseId"] != release_id:
        raise ValueError("MapRelease ID mismatch")
    verify_version(root, descriptor["mapVersionId"])
    return manifest


def create_release(root, version_id, composition_path):
    version = verify_version(root, version_id)
    descriptor = {
        "kind": "MapRelease",
        "schemaVersion": 1,
        "mapId": version["descriptor"]["mapId"],
        "mapVersionId": version_id,
        "composition": load_composition(composition_path),
    }
    release_id = digest_bytes(canonical(descriptor))
    path = root / "releases" / f"{release_id}.json"
    if not path.exists():
        atomic_json(
            path,
            {
                "mapReleaseId": release_id,
                "createdAt": now_iso(),
                "descriptor": descriptor,
            },
        )
    verify_release(root, release_id)
    print(release_id)


def promote(root, channel, release_id):
    release = verify_release(root, release_id)
    pointer = {
        "channel": channel,
        "mapId": release["descriptor"]["mapId"],
        "mapReleaseId": release_id,
        "mapReleaseDigest": release_id,
        "changedAt": now_iso(),
    }
    atomic_json(root / "channels" / f"{channel}.json", pointer)
    print(json.dumps(pointer, sort_keys=True))


def resolve_channel(root, channel):
    path = root / "channels" / f"{channel}.json"
    if not path.is_file():
        raise FileNotFoundError(f"unknown release channel: {channel}")
    pointer = json.loads(path.read_text(encoding="utf-8"))
    verify_release(root, pointer["mapReleaseId"])
    return pointer


def pin(root, run_id, channel, release_id):
    if channel:
        pointer = resolve_channel(root, channel)
        release_id = pointer["mapReleaseId"]
        selected_from = {"channel": channel}
    else:
        verify_release(root, release_id)
        selected_from = {"mapReleaseId": release_id}
    release = verify_release(root, release_id)
    value = {
        "runId": run_id,
        "mapId": release["descriptor"]["mapId"],
        "mapVersionId": release["descriptor"]["mapVersionId"],
        "mapReleaseId": release_id,
        "mapReleaseDigest": release_id,
        "selectedFrom": selected_from,
        "pinnedAt": now_iso(),
    }
    path = root / "pins" / f"{run_id}.json"
    if path.exists():
        existing = json.loads(path.read_text(encoding="utf-8"))
        if existing != value:
            raise FileExistsError(f"{run_id} already has an immutable map pin")
        print(json.dumps(existing, sort_keys=True))
        return
    atomic_json(path, value)
    print(json.dumps(value, sort_keys=True))


def show(root, channel, run_id):
    if channel:
        return resolve_channel(root, channel)
    path = root / "pins" / f"{run_id}.json"
    if not path.is_file():
        raise FileNotFoundError(f"run has no map pin: {run_id}")
    value = json.loads(path.read_text(encoding="utf-8"))
    release = verify_release(root, value["mapReleaseId"])
    if value["mapVersionId"] != release["descriptor"]["mapVersionId"]:
        raise ValueError("run pin does not match its MapRelease")
    return value


def quality(root, version_id):
    manifest = verify_version(root, version_id)
    directory = root / "versions" / version_id
    metadata = yaml.safe_load((directory / "map.yaml").read_text(encoding="utf-8"))
    image = Image.open(directory / metadata["image"]).convert("L")
    values = list(image.getdata())
    negate = int(metadata["negate"])
    occupied_threshold = float(metadata["occupied_thresh"])
    free_threshold = float(metadata["free_thresh"])
    occupied = free = unknown = 0
    for pixel in values:
        probability = pixel / 255.0 if negate else (255 - pixel) / 255.0
        if probability > occupied_threshold:
            occupied += 1
        elif probability < free_threshold:
            free += 1
        else:
            unknown += 1
    total = len(values)
    print(json.dumps({
        "mapVersionId": version_id,
        "mapId": manifest["descriptor"]["mapId"],
        "widthPx": image.width,
        "heightPx": image.height,
        "resolutionMPerPx": float(metadata["resolution"]),
        "knownFraction": round((free + occupied) / total, 6),
        "freeFraction": round(free / total, 6),
        "occupiedFraction": round(occupied / total, 6),
        "unknownFraction": round(unknown / total, 6),
    }, indent=2, sort_keys=True))


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--registry", default="registry")
    commands = parser.add_subparsers(dest="command", required=True)

    add = commands.add_parser("ingest")
    add.add_argument("--map-id", required=True)
    add.add_argument("--yaml", required=True)
    add.add_argument("--provenance", required=True)
    check_v = commands.add_parser("verify-version")
    check_v.add_argument("--version", required=True)
    inspect = commands.add_parser("quality")
    inspect.add_argument("--version", required=True)
    create = commands.add_parser("create-release")
    create.add_argument("--version", required=True)
    create.add_argument("--composition", required=True)
    check_r = commands.add_parser("verify-release")
    check_r.add_argument("--release", required=True)
    move = commands.add_parser("promote")
    move.add_argument("--channel", required=True)
    move.add_argument("--release", required=True)
    attach = commands.add_parser("pin")
    attach.add_argument("--run-id", required=True)
    selection = attach.add_mutually_exclusive_group(required=True)
    selection.add_argument("--channel")
    selection.add_argument("--release")
    display = commands.add_parser("show")
    display_selection = display.add_mutually_exclusive_group(required=True)
    display_selection.add_argument("--channel")
    display_selection.add_argument("--run-id")

    args = parser.parse_args()
    root = open_registry(args.registry)
    if args.command == "ingest":
        ingest(root, args.map_id, args.yaml, args.provenance)
    elif args.command == "verify-version":
        print(json.dumps(verify_version(root, args.version), indent=2, sort_keys=True))
    elif args.command == "quality":
        quality(root, args.version)
    elif args.command == "create-release":
        create_release(root, args.version, args.composition)
    elif args.command == "verify-release":
        print(json.dumps(verify_release(root, args.release), indent=2, sort_keys=True))
    elif args.command == "promote":
        promote(root, args.channel, args.release)
    elif args.command == "pin":
        pin(root, args.run_id, args.channel, args.release)
    else:
        print(json.dumps(show(root, args.channel, args.run_id), indent=2, sort_keys=True))


if __name__ == "__main__":
    main()
~~~

Check it before creating state:

~~~bash
chmod +x mapctl.py
python -m py_compile mapctl.py
python mapctl.py --help
~~~

### 2. Ingest Week 6 geometry as MapVersion A

~~~bash
export SOURCE_MAP="$LAB_ROOT/maps/week06/site_v1.yaml"
test -r "$SOURCE_MAP"
export VERSION_A="$(python mapctl.py ingest \
  --map-id training-lab \
  --yaml "$SOURCE_MAP" \
  --provenance week06-slam-run-a | tail -n 1)"
echo "VERSION_A=$VERSION_A" | tee "$LAB_ROOT/evidence/week11/version-a.txt"
python mapctl.py verify-version --version "$VERSION_A" > \
  "$LAB_ROOT/evidence/week11/version-a-manifest.json"
python mapctl.py quality --version "$VERSION_A" | tee \
  "$LAB_ROOT/evidence/week11/version-a-quality.json"
~~~

Run ingest again. Identical input and provenance must return the same digest
and create no second version directory.

### 3. Create approved MapRelease A

Create an explicit composition:

~~~bash
python - <<'PY'
import json
from pathlib import Path

composition = {
    "alignment": {
        "kind": "SE2",
        "referenceFrame": "site",
        "translationM": [0.0, 0.0],
        "yawRad": 0.0,
    },
    "overlays": {"noGoZones": [], "speedZones": [], "inspectionPoses": []},
    "policies": {"defaultMaxSpeedMps": 0.25},
    "routes": [],
}
Path("composition-a.json").write_text(
    json.dumps(composition, indent=2, sort_keys=True) + "\n"
)
PY

export RELEASE_A="$(python mapctl.py create-release \
  --version "$VERSION_A" --composition composition-a.json | tail -n 1)"
echo "RELEASE_A=$RELEASE_A" | tee "$LAB_ROOT/evidence/week11/release-a.txt"
python mapctl.py verify-release --release "$RELEASE_A" > \
  "$LAB_ROOT/evidence/week11/release-a-manifest.json"
~~~

Creating the same release again must return the same ID.

### 4. Prove an overlay-only change creates a release, not a version

~~~bash
python - <<'PY'
import json
from pathlib import Path

value = json.loads(Path("composition-a.json").read_text())
value["overlays"]["noGoZones"] = [
    {"id": "keep-out", "polygonM": [[1, 1], [2, 1], [2, 2], [1, 2]]}
]
Path("composition-a-overlay.json").write_text(
    json.dumps(value, indent=2, sort_keys=True) + "\n"
)
PY

export RELEASE_A_OVERLAY="$(python mapctl.py create-release \
  --version "$VERSION_A" --composition composition-a-overlay.json | tail -n 1)"
test "$RELEASE_A" != "$RELEASE_A_OVERLAY"
python mapctl.py verify-release --release "$RELEASE_A_OVERLAY" | \
  tee "$LAB_ROOT/evidence/week11/release-a-overlay-manifest.json"
~~~

Both releases must reference VERSION_A.

### 5. Create a geometry change as MapVersion B

~~~bash
mkdir -p candidate-b
python - "$SOURCE_MAP" "$PWD/candidate-b" <<'PY'
from pathlib import Path
import sys
import yaml
from PIL import Image, ImageDraw

source_yaml = Path(sys.argv[1]).resolve()
out = Path(sys.argv[2]).resolve()
metadata = yaml.safe_load(source_yaml.read_text())
source_image = Path(metadata["image"])
if not source_image.is_absolute():
    source_image = source_yaml.parent / source_image
candidate_image = out / ("candidate" + source_image.suffix.lower())
image = Image.open(source_image).convert("L")
draw = ImageDraw.Draw(image)
x0, y0 = image.width // 2, image.height // 2
draw.rectangle((x0, y0, x0 + 5, y0 + 20), fill=0)
image.save(candidate_image)
metadata["image"] = candidate_image.name
(out / "candidate.yaml").write_text(yaml.safe_dump(metadata, sort_keys=True))
PY

export VERSION_B="$(python mapctl.py ingest \
  --map-id training-lab \
  --yaml candidate-b/candidate.yaml \
  --provenance week11-geometry-change | tail -n 1)"
test "$VERSION_A" != "$VERSION_B"
export RELEASE_B="$(python mapctl.py create-release \
  --version "$VERSION_B" --composition composition-a.json | tail -n 1)"
python mapctl.py quality --version "$VERSION_B" | tee \
  "$LAB_ROOT/evidence/week11/version-b-quality.json"
python mapctl.py verify-release --release "$RELEASE_B" > \
  "$LAB_ROOT/evidence/week11/release-b-manifest.json"
~~~

### 6. Apply release gates

Compare:

- YAML semantics, image dimensions, and every checksum;
- known/free/occupied/unknown fractions and changed-cell count;
- alignment residuals, overlays, policies, routes, and robot capabilities;
- AMCL scan alignment at three poses;
- five Nav2 goals, including a path near the changed area; and
- navigation success and final-error thresholds from Week 7.

Record a pass/fail row for each item in release-gate.md. An unexplained change
fails even when aggregate percentages look small.

### 7. Resolve a channel once and pin a run

~~~bash
python mapctl.py promote --channel staging --release "$RELEASE_A"
python mapctl.py pin --run-id run-001 --channel staging
python mapctl.py show --run-id run-001 | tee \
  "$LAB_ROOT/evidence/week11/run-001-pin-before.json"

python mapctl.py promote --channel staging --release "$RELEASE_B"
python mapctl.py show --channel staging | tee \
  "$LAB_ROOT/evidence/week11/channel-after-move.json"
python mapctl.py show --run-id run-001 | tee \
  "$LAB_ROOT/evidence/week11/run-001-pin-after-channel-move.json"
python mapctl.py pin --run-id run-002 --channel staging
~~~

Run 001 remains pinned to RELEASE_A; run 002 resolves RELEASE_B. Attempting to
repin run 001 must fail. Create a new run instead of rewriting history.

### 8. Roll the channel back without deleting history

~~~bash
python mapctl.py promote --channel staging --release "$RELEASE_A"
python mapctl.py show --channel staging | tee \
  "$LAB_ROOT/evidence/week11/rollback-pointer.json"
python mapctl.py verify-version --version "$VERSION_A" >/dev/null
python mapctl.py verify-version --version "$VERSION_B" >/dev/null
python mapctl.py verify-release --release "$RELEASE_A" >/dev/null
python mapctl.py verify-release --release "$RELEASE_B" >/dev/null
~~~

Rollback changes only the channel. Both versions, both releases, and every run
pin remain verifiable.

## Assignment

Execute a staged rollout in simulation:

1. Publish RELEASE_A and create a run pinned to it.
2. Publish the overlay-only successor over VERSION_A; prove observed geometry
   and the active run pin did not change.
3. Publish RELEASE_B over changed VERSION_B.
4. Run the same three navigation goals with new runs pinned to A and B.
5. Compare convergence, path success, length, terminal error, and behavior near
   the changed area.
6. Move the channel to B if it passes; otherwise roll it back to A without
   altering a run or deleting an object.
7. Produce a report with all IDs, composition, gates, timestamps, decision, and
   rollback instructions.

One simulator instance is sufficient when identities and evidence remain
separate.

## Failure injection

Complete all six:

1. Remove the image referenced by source YAML; ingest must fail before creating
   a MapVersion.
2. Copy the registry and alter a packaged image; verify-version must reject it.
3. Modify a copied release manifest; verify-release must reject its digest.
4. Promote a nonexistent release; the channel must remain unchanged.
5. Move a channel after pinning; the run stays fixed and repinning it fails.
6. Model an overlay edit as a MapVersion, explain why that loses the
   observation/approval distinction, then correct it as a MapRelease.

Never corrupt the primary registry.

## Measurements and deliverables

Keep under the Week 11 evidence directory:

- mapctl.py and dependency lock;
- version A/B manifests, digests, provenance, and quality reports;
- release A, overlay-only, and B manifests and composition inputs;
- visual diff, changed-cell count, and completed release gate;
- channel pointers and run pins before/after movement;
- navigation results for each pinned release;
- rollback record; and
- failure-injection logs.

Measure changed pixels, occupancy fractions, map dimensions and semantics,
alignment residual, overlay validation, navigation success/length/duration/
terminal error, rollout time, rollback time, and runs per release.

## Objective exit criteria

Advance only when:

- identical geometry and provenance produce the same MapVersion ID;
- changed geometry or semantic metadata produces a new MapVersion;
- a composition-only change preserves the version and creates a MapRelease;
- every release references exactly one verified version;
- every file and both descriptor types pass digest verification;
- incomplete or corrupt inputs cannot be promoted;
- channels point to releases rather than bare versions;
- channel movement never mutates an existing run pin;
- each navigation result names its MapRelease ID/digest and MapVersion; and
- objective gates, not appearance alone, decide promotion.

## Troubleshooting

| Symptom | Diagnostic | Correction |
| --- | --- | --- |
| Identical-looking geometry gets a new version | Compare YAML, provenance, and encoded bytes | Decide whether the exact difference is meaningful |
| Overlay edit changed the version | Inspect the version descriptor | Restore geometry and create a new release |
| Run changes after channel movement | Search execution code for channel lookup | Resolve only at assignment and use the pin |
| Map appears shifted | Compare origin, resolution, alignment, dimensions | Reject and publish corrected immutable objects |
| Verification fails after copy | Compare checksum and descriptor digest | Restore a known verified object |
| Rollback does not affect an active run | Inspect its release pin | Expected: rollback affects later assignments |

## Next step

Week 12 adds live media and durable artifacts. Real-time streams are not
archival evidence; large objects need checksums and bounded upload; every
capture names its run, robot, time, MapRelease, and MapVersion.

[← Week 10: Durable command delivery](week-10-durable-command-delivery.md) · [Week index](README.md) · [Week 12: WebRTC media and artifacts →](week-12-webrtc-media-and-artifacts.md)
