# Contributing

Small project, few rules, all of them about evidence.

## Setup

```bash
git clone https://github.com/isearchxsolve/life-archive-engine
cd life-archive-engine
python -m venv .venv && . .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -e ".[dev]"
pytest tests/python -q            # expect 32 passed
```

No service is required to run the suite. Meilisearch, Postgres and the object stores are optional;
the engine degrades to filesystem storage plus a manifest scan (`docs/LIMITATIONS.md` says exactly
which layers are stubs today).

## What a good change looks like

- **One behaviour per pull request**, with a test that fails before it and passes after it.
- **No weakening of tests to make a change pass.** If an existing test is genuinely wrong (it can
  happen - see the note at the end), change the *assertion* with a paragraph explaining why the old
  expectation was wrong, in the same commit. Never delete a test to get green.
- **Provenance is the product.** Anything that writes to the archive goes through the manifest
  writer. A code path that stores bytes without recording where they came from will be refused.
- **Do not trust input paths.** The storage layer's `_resolve` guard exists because a personal
  archive is the most attractive path-traversal target there is. New call sites use it; none
  bypass it with `os.path.join`.
- **Parsers stay defensive.** `defusedxml` for XML, `zipfile` with size checks for archives, no
  `eval`, no `pickle` on archive content.

## Adding a connector

A connector lives in `python/life_archive/connectors/` and provides:

| Method | Contract |
|---|---|
| `discover()` | async iterator of discovered items - must be idempotent and must not modify the source |
| `hash()` | a stable content hash for the item (true content hash preferred; if there is no single byte stream, hash a deterministic manifest and say so in the docstring) |
| `store()` | write the immutable copy through the storage layer |
| `record()` | append the `ManifestEntry` with full provenance |
| `run()` | orchestrate the above and return a result with counts and errors |

Then register it in `CONNECTORS` in `cli.py` and add a test module that builds its own synthetic
fixture in `tmp_path` (never read a real user's data, never hit a live API from the suite - mock
the transport, as `test_github_connector.py` does with `httpx.MockTransport`).

If a connector is only validated against synthetic data, say so in `docs/LIMITATIONS.md`. An
unvalidated connector that is documented as unvalidated is honest; one that is documented as
working is a lie waiting to reach a user's archive.

## Reporting

- **Bugs**: open an issue with the command you ran, the archive layout (paths only), and the exact
  output. `docs/DEMO.md` shows the level of detail expected.
- **Security issues**: do not open a public issue. Email the maintainer with the details and a
  reproduction; a fix and a note will be published together.

## Where the honest gaps are

`docs/LIMITATIONS.md` is the backlog of record: unimplemented `index()` / `normalize()` layers,
the GitHub connector's lack of live validation, and the absence of scheduled ingestion. If you want
to contribute but do not know where to start, an issue opened from that file is a good one.

## Note on history

This repository was extracted from a private personal archive: the engine, its tests and the design
documents were public-spirited code; the archive contents were not and never will be. The engine
carries no personal data, which is why it can be developed in the open. Please keep it that way -
no fixtures containing real names, real addresses or real documents, ever.
