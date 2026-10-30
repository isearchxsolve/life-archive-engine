# Demo - one ingest run, end to end

Real terminal output from a fresh archive, captured on 2026-10-30. Nothing here is mocked: the
archive was empty, three files were dropped into `00_RAW/`, and the commands below were run in
order. Copied verbatim, including the two defects this demo exposed (see *What this demo found*).

```
### sources
              Available Connectors              
+----------------------------------------------+
| Source         | Credentials Required        |
|----------------+-----------------------------|
| git            | None (local git CLI)        |
| github         | GITHUB_PAT env var required |
| google-takeout | None (manual export)        |
| linkedin       | None (manual export)        |
| local-files    | None                        |
| whatsapp       | None (manual export)        |
+----------------------------------------------+
### ingest
Starting ingestion from: local-files
Archive root: D:\tmp\demo\archive

Ingestion Results
Run ID: 0f728ce8-9550-4afa-b5d9-00d97d201150
Status: PASS
Duration: 13.18s
         Statistics         
+--------------------------+
| Metric           | Value |
|------------------+-------|
| Items Discovered | 3     |
| Items Processed  | 3     |
| Items Failed     | 0     |
| Items Skipped    | 0     |
+--------------------------+

Manifest files updated:
  - 01_MANIFEST/source_manifest.jsonl
  - 01_MANIFEST/hashes.jsonl
  - 01_MANIFEST/acquisition_log.jsonl
  - 01_MANIFEST/ingestion_runs.jsonl
### status
Archive root: D:\tmp\demo\archive

Archive Summary
       By Source       
+---------------------+
| Source      | Count |
|-------------+-------|
| local-files |     3 |
+---------------------+
    By Status     
+----------------+
| Status | Count |
|--------+-------|
| STORED |     3 |
+----------------+

Total size: 0.00 MB

Recent Ingestion Runs
+---------------------------------------------------------------+
| Run ID      | Connector   | Status | Items | Time             |
|-------------+-------------+--------+-------+------------------|
| 0f728ce8... | local-files | PASS   |     3 | 2026-10-30 03:59 |
+---------------------------------------------------------------+
### re-ingest (idempotency: no new raw objects, repeats marked DUPLICATE_OF)
Ingestion Results
Run ID: b8555e5b-d511-467d-b080-bc2ce59857ae
Status: PASS
Duration: 13.54s
         Statistics         
+--------------------------+
| Metric           | Value |
|------------------+-------|
| Items Discovered | 3     |
| Items Processed  | 3     |
| Items Failed     | 0     |
| Items Skipped    | 0     |
+--------------------------+

Manifest files updated:
  - 01_MANIFEST/source_manifest.jsonl
  - 01_MANIFEST/hashes.jsonl
  - 01_MANIFEST/acquisition_log.jsonl
  - 01_MANIFEST/ingestion_runs.jsonl
### verify
Verifying archive integrity
Archive root: D:\tmp\demo\archive
  Checking sha256:c720e9c0b2e27... OK
  Checking sha256:40a30305e5c73... OK
  Checking sha256:de4a933fc48b2... OK

Verification Results
  Verified: 3
  Missing: 0
  Hash mismatch: 0

[OK] Archive integrity verified
### search
Searching for: journal

Results (2)
+--------------------------------------------------------------------------------------+
| Filename              | Source      | MIME Type                |   Size | Date       |
|-----------------------+-------------+--------------------------+--------+------------|
| 2026-03-14-journal.md | local-files | application/octet-stream | 0.1 KB | 2026-10-30 |
| 2026-03-14-journal.md | local-files | application/octet-stream | 0.1 KB | 2026-10-30 |
+--------------------------------------------------------------------------------------+
```

## What just happened

1. **`ingest local-files`** walked `00_RAW/`, hashed all three files, and wrote an immutable copy
   of each under `00_RAW/local-files/<date>/<first-2-sha-chars>/<full-sha256>_<filename>`. The
   original files were not moved, renamed or modified.
2. **`01_MANIFEST/source_manifest.jsonl`** gained one JSON line per item, carrying the full
   provenance record - sha256, xxhash64, size, original timestamp, acquisition timestamp,
   connector, run id, privacy classification, dedupe status:

   ```json
   {"archive_id": "sha256:c720e9c0b2e27826c830d50253fbdd37469d705ec93d5fe326d362dcdd0d519a", "source": "local-files", "source_identifier": "inbox\meeting.txt", "original_filename": "meeting.txt", "mime_type": "text/plain", "size_bytes": 50, "sha256": "c720e9c0b2e27826c830d50253fbdd37469d705ec93d5fe326d362dcdd0d519a", "xxhash64": "7094a434734b0aba", "privacy_classification": "PRIVATE", "processing_status": "STORED", "dedupe_status": "UNIQUE", "dedupe_of": null, "error": null}
   ```
3. **Re-running the same ingest** is safe: content addressing means the raw store still holds
   **3** objects, and the three repeat entries are recorded as `DUPLICATE_OF`, not as new content.
   That is the idempotency claim, demonstrated rather than asserted.
4. **`verify`** re-hashed every stored object against the manifest: 3 verified, 0 missing,
   0 mismatches.
5. **`search`** ran without Meilisearch and fell back to scanning the manifest, as documented.

## What this demo found (two real defects, left visible)

Running the demo was worth more than writing it, because it caught things the unit tests do not:

- **`search` double-counts after a re-ingest.** The manifest is append-only by design, so it holds
  both the `UNIQUE` row and the `DUPLICATE_OF` row for the same `archive_id`; the fallback search
  iterates every row and therefore lists the same object twice. The index path must collapse by
  `archive_id` (prefer the original entry). Tracked as issue #2.
- **MIME types fall back to `application/octet-stream` for `.md`.** `mimetypes.guess_type` returns
  `None` for `.md` here, so the guard substitutes the generic type. A small extension map fixes it.
  Tracked as issue #3.

Neither is hidden or worked around: a demo that only shows the happy path teaches a reader nothing
about whether the authors measured their own software.

## Reproduce it

```bash
mkdir -p LIFE_ARCHIVE/00_RAW/notes
printf '# entry
shipped the pipeline
' > LIFE_ARCHIVE/00_RAW/notes/2026-03-14-journal.md
printf 'meeting notes
'                 > LIFE_ARCHIVE/00_RAW/notes/meeting.txt

life-archive ingest local-files
life-archive status
life-archive ingest local-files     # idempotent: no new raw objects
life-archive verify
life-archive search journal
```
