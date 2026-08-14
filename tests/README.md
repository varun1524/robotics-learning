# Test Strategy

The curriculum grows four complementary test layers:

| Layer | Purpose |
| --- | --- |
| Unit | Pure transforms, validation, state transitions, and policy |
| Contract | The same canonical behavior against simulator, dummy, and physical adapters |
| Integration | ROS graph, journals, broker, storage, WebRTC, and mission interactions |
| Fault injection | Process loss, stale data, duplicates, corruption, packet loss, and WAN outage |

Simulation success alone is insufficient. Every physical effect must have an
idempotency test, every queue a capacity test, every credential a negative
scope test, and every immutable artifact a checksum-corruption test.

Use deterministic seeds and explicit timeouts. Store measurements in small
Markdown or CSV summaries; keep bags, databases, screenshots, and videos local
unless deliberately publishing sanitized samples.

The capstone acceptance matrix is defined in
[Week 16](../docs/weeks/week-16-capstone-cloud-edge-inspection.md).

## Required pre-commit secret scan

The repository pins the public Gitleaks pre-commit hook. Install and run it
before adding course-generated credentials or evidence:

    python3 -m pip install --user pre-commit
    pre-commit install
    pre-commit run --all-files

A detected credential blocks the commit. Do not bypass the hook for course
work. Remove the secret from Git, rotate it if it was real, and keep only a
redacted example. Ignore rules are a second guard, not permission to store
credentials carelessly.

[Repository home](../README.md) · [Evidence policy](../evidence/README.md)
