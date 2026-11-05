# Southern Star — Fulcrum Full Data Backup

A scheduled disaster-recovery backup of Southern Star Central Gas's Fulcrum
organization. Twice a day, at 8:00 AM and 8:00 PM Central, it:

1. **Reads every Fulcrum app** through the Fulcrum API. It's read-only:
   nothing in Fulcrum is ever changed.
2. **Writes one CSV per app**, plus one per repeatable section, photo field
   and record link.
3. **Downloads the files attached to records.** Only files the destination
   doesn't have yet are downloaded.
4. **Uploads everything as one dated folder** to Southern Star's SFTP
   (`shareit.southernstar.com`), writing a `_SUCCESS` marker last.
5. **Emails an alert if anything fails.**

**To install and run it, follow [INSTALL.md](INSTALL.md).** It covers Windows
and Linux step by step: configuration, secrets, the preflight check, the test
alert, UAT, scheduling, troubleshooting and routine maintenance.

## What's in a backup

Each run adds a folder `backup_<YYYYMMDD>_<HHMM>_CT` (Central time) holding:
- one `.csv` per table, exactly as Fulcrum exports it
- `attachments/`: the new attachment files, named
  `<app>__<record id>__<attachment id>.<ext>`
- `_ATTACHMENTS.csv`: every attachment the CSVs reference, its record and
  original file name, and which backup folder holds its file
- `_SCHEMA.csv`: every table's columns and types
- `_MANIFEST.csv`: every file with its row count, size and SHA-256 checksum
- `_SUCCESS`, written last. Load only folders that have it.

## Commands

From the install folder, with the virtual environment's Python
(`.venv\Scripts\python` on Windows, `.venv/bin/python` on Linux):

| Command | What it does |
|---|---|
| `python -m src.main run` | The scheduled job: check the SFTP, export, upload |
| `python -m src.main run --dry-run` | Export to the local disk only: no upload, no email |
| `python -m src.main preflight` | Check the machine one item at a time; changes nothing |
| `python -m src.main alert-test` | Send one sample failure alert |
| `python -m src.main deliver` | Upload the newest backup not yet uploaded, e.g. to finish an interrupted one |
| `python -m src.main attachments` | Count and size the attachments; downloads nothing |
| `python -m src.main export` | Export to the local disk only, like a dry run |
| `python -m src.main export --dry-run` | Print what would be exported; fetches no data |

Add `--help` to any command for its options. INSTALL.md section 10 has the
full reference, including the exit codes.

## Layout

| Path | What it is |
|---|---|
| `src/main.py` | The commands above |
| `src/config.py` | Loading the config, environment-variable secrets, placeholder checks |
| `src/fulcrum_client.py` | Read-only Fulcrum API client |
| `src/catalog.py` | Which apps and tables to export |
| `src/export.py` | One backup to the local `staging/` folder |
| `src/attachments.py` | Attachment IDs from the CSVs, downloads, what each SFTP account already has |
| `src/naming.py` | File and folder names, from the config's `naming` section |
| `src/sftp_client.py`, `src/deliver.py` | Uploading to the SFTP, with the server key pinned |
| `src/runlog.py`, `src/alerts.py` | The log file per run, and the failure email |
| `src/housekeeping.py` | One run at a time; cleanup of old local backups |
| `src/preflight.py` | The install-time checks |
| `config/config.example.yaml` | The config template; copy it to `config/config.yaml` |
| `tests/` | Automated tests |

## Tests

The tests run entirely on the local machine. They use a fake Fulcrum API, an
in-process SFTP server and an in-process SMTP relay, and never contact
Fulcrum, the SFTP host or a mail server.

```bash
.venv/bin/python -m pip install pytest
.venv/bin/python -m pytest -q
```

Requires Python 3.11 or newer.
