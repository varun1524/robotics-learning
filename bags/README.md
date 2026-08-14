# Bag Recordings

Store local rosbag2 or MCAP output under this directory. Large recordings are
ignored by Git.

For a useful recording, keep a small Markdown manifest containing:

- assignment and run ID;
- UTC start/end;
- source commit and ROS distribution;
- simulator or robot identity;
- topics, QoS overrides, and frame assumptions;
- launch command and parameters;
- expected phenomenon and injected fault;
- checksum and local filename; and
- privacy review notes.

Do not publish people, private locations, credentials, serial numbers, or other
sensitive sensor content.

If a recording is required for a repeatable test but is too large for Git,
track a public fetch or deterministic regeneration command plus its byte length
and SHA-256 digest. Verification must fail on a digest mismatch.

[Repository home](../README.md) · [Evidence policy](../evidence/README.md)
