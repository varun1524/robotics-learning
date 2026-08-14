# Assignment Evidence

Generated evidence is ignored by default. Each completed week should retain a
small sanitized Markdown manifest and, when helpful, compact CSV measurements.

Recommended layout:

    evidence/
      week-NN/
        README.md
        measurements.csv

The weekly manifest should identify:

- exact commands and commit;
- environment and package versions;
- acceptance criteria and result;
- measurement method and uncertainty;
- injected failure and observed recovery;
- links to local bags, maps, screenshots, or videos;
- known limitations; and
- the next experiment.

Never commit secrets, certificates, private keys, access URLs, raw customer
data, or sensitive imagery.

Run the command below before review:

    pre-commit run --all-files

Large local evidence is reproducible only when the manifest records a public
fetch or deterministic regeneration command, byte length, and SHA-256 digest.

[Repository home](../README.md) · [Test strategy](../tests/README.md)
