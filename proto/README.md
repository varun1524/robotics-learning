# Future Protocol Contracts

This directory will hold versioned, public protocol definitions introduced by
the command-delivery assignments. No schema is frozen in the scaffold.

Contracts should eventually distinguish:

- submission acceptance by the local sender;
- durable receipt by the peer;
- application progress and terminal result;
- commands from observations;
- live media session control from media packets; and
- artifact identity from temporary upload credentials.

Messages need stable identity, explicit schema version, creation time,
correlation, optional expiry, bounded size, and deterministic duplicate
behavior. Payloads too large for the command limit must use an artifact
reference.

Mission assignment messages resolve any mutable map channel before publication
and carry a `MapRelease` ID plus digest. A `MapRelease` references exactly one
immutable `MapVersion`; messages never pin a bare version or ask a running robot
to resolve a channel.

Generated language bindings belong in ignored build output, not source control.
Compatibility tests must accompany every schema evolution.

[Repository home](../README.md) · [Week 10](../docs/weeks/week-10-durable-command-delivery.md)
