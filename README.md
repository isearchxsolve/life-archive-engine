# life-archive

[![tests](https://github.com/isearchxsolve/life-archive-engine/actions/workflows/tests.yml/badge.svg)](https://github.com/isearchxsolve/life-archive-engine/actions/workflows/tests.yml)

**A provenance-first ingestion engine for a personal digital archive.**

Point it at the places your life is stored — a filesystem, Git repositories, GitHub accounts,
Google Takeout exports, WhatsApp chat exports, LinkedIn data dumps — and it produces one
auditable archive: every object content-addressed, every ingestion run recorded, every item
classified with a privacy level before it is written.

This is the engine only. It runs on your machine, against your own data, and ships with no data.

```
00_RAW/            untouched originals, exactly as collected
01_MANIFEST/       append-only JSONL: what was ingested, when, from where, with what hash
02_...             derived layers (index, timeline, knowledge graph) - see docs/
```

## Why this exists

Personal archives rot in three specific ways, and each one is an engineering problem:

1. **You cannot prove what you have.** Files get renamed, drive letters change, two versions of
   "final" exist. Here every object is keyed by its SHA-256 and logged in an append-only manifest,
   so `life-archive verify` can tell you what changed, what vanished, and what was never ingested.
2. **Collection is destructive.** Copying files in by hand loses timestamps and mutates originals.
   Connectors read the source, hash it, store it under a content-addressed key, and never write
   back to the source.
3. **Sensitive material gets mixed with harmless material.** A life archive contains medical
   records next to shopping receipts. The privacy classifier assigns a level to each item
   (and can redact) *before* storage, so a later export or publish step has something real to
   filter on rather than a folder name someone remembered to use.

## Install

```bash
git clone https://github.com/isearchxsolve/life-archive-engine
cd life-archive
python -m venv .venv && . .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -e ".[dev]"
```

Requires Python 3.11+. Optional extras: `.[search]` (Meilisearch lexical index),
`.[media]` (media metadata, xxhash).

## Use

```bash
life-archive status                 # objects, bytes, last run, per-connector counts
life-archive sources                # which connectors are configured and ready
life-archive ingest --source local-files --root <path-to-your-files>
life-archive ingest --source git --root <path-to-a-repo>
life-archive ingest --source github --account <user>        # needs GITHUB_PAT
life-archive ingest --source google-takeout --zip <takeout.zip>
life-archive ingest --source whatsapp --zip <chat-export.zip>
life-archive verify                 # re-hash stored objects against the manifest
life-archive search "query"         # lexical search when Meilisearch is up, else manifest scan
```

Configuration is environment-based; copy `.env.example` to `.env`. `docker-compose.yml` brings up
the optional backing services (Meilisearch, Postgres, Qdrant, MinIO) if you want them — the engine
runs without them and degrades to manifest scanning.

## Connectors

| Connector | Source | Credentials | Notes |
|---|---|---|---|
| `local-files` | any directory | none | hashes and stores files, preserves size/mtime |
| `git` | local repositories | none | `git bundle create --all` + commit metadata |
| `github` | GitHub account | `GITHUB_PAT` | issues/PRs/releases via REST, paginated with rate-limit backoff |
| `google-takeout` | Takeout `.zip` | none (manual export) | fans services out under `00_RAW/google/<service>/` |
| `whatsapp` | chat export `.txt`/`.zip` | none (manual export) | structured JSONL + media |
| `linkedin` | LinkedIn data export | none (manual export) | posts/messages |

Adding a connector means implementing the connector interface in `python/life_archive/connectors/`
and registering it; the manifest, hashing, privacy classification and storage layers are shared.

## Design notes worth knowing before you trust it

- **Content addressing, not paths.** The storage layer maps keys to paths behind a guard that
  rejects absolute paths, `..`, and anything resolving outside the archive root (CWE-22).
- **The manifest is append-only.** `01_MANIFEST/*.jsonl` is written as JSONL and never rewritten,
  so an ingestion run cannot silently overwrite history. `hashes.jsonl` is the integrity baseline
  `verify` diffs against.
- **Parsers are not trusted with XML.** Document extraction goes through `defusedxml` when
  available; entity-expansion is refused rather than resolved (CWE-611).
- **Privacy classification happens before write.** Levels are assigned per item, and redaction is
  available for text; the point is that the decision is recorded with the object, not applied later
  by a script that guesses from filenames.

## Status — measured, not promised

`docs/LIMITATIONS.md` is the honest phase-1 accounting: which connectors are complete, which
index layers are stubs, and what is deliberately not built yet. Read it before assuming a feature
exists. `docs/01_ARCHITECTURE_ASSESSMENT.md` … `06_IMPLEMENTATION_PLAN.md` are the original design
documents, kept as the audit trail.

Test suite:

```bash
pytest tests/python -q
```

## License

MIT — see `LICENSE`. Copyright (c) 2026 Kunal Das.
