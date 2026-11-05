# Fulcrum Full Data Backup — Install and Operations Guide

For Southern Star's IT team (Brent Davis, Justin Holder). It takes you from
an empty VM to a backup that runs on its own twice a day, with nothing to
install from Fulcrum's side and no call needed. Each step gives the exact
command, what you should see, and what to do if you don't.

Plan on one to two hours, most of it waiting for values from other teams
(the token, the relay settings) rather than doing work.

## Contents

0. [What this is](#0-what-this-is)
1. [Before you start](#1-before-you-start)
2. [Keep your own copy of the code](#2-keep-your-own-copy-of-the-code)
3. [Install on Windows](#3-install-on-windows)
4. [Install on Linux](#4-install-on-linux)
5. [Test it (UAT)](#5-test-it-uat)
6. [Schedule it](#6-schedule-it)
7. [What a good run looks like](#7-what-a-good-run-looks-like)
8. [When something fails](#8-when-something-fails)
9. [Routine tasks](#9-routine-tasks)
10. [Reference](#10-reference)

---

## 0. What this is

A Python script that runs on one of your VMs at **8:00 AM and 8:00 PM Central**.
Each run:

1. **Reads every app in your Fulcrum organization** through the Fulcrum API.
   It's read-only: nothing in Fulcrum is ever created, changed or deleted.
2. **Writes one CSV per app**, plus one per repeatable section, photo field
   and record link, into a dated folder on the VM.
3. **Downloads the files attached to records.** Only files the SFTP doesn't
   have yet are downloaded.
4. **Uploads the folder to `shareit.southernstar.com` over SFTP**, and writes
   a `_SUCCESS` marker last.
5. **Emails Myles Shade and help@southernstar.com if anything fails.** The
   help@ address opens a Freshservice ticket.

**Size.** The first run carries every existing attachment: about 1.1 GB, as
of October 2026, and around 8 minutes. After that, each run is about 52 MB of
CSVs plus only new attachments.

**What's included.** The CSVs and the files in attachment fields, such as
PDFs, documents and images uploaded as files. Photos taken in photo fields
appear in the CSVs as photo IDs, not as files.

---

## 1. Before you start

### 1.1 The VM

- **Windows Server or Linux.** Both are covered below; use whichever section
  matches.
- **Python 3.11 or newer.** Python 3.11 is what Southern Star runs elsewhere,
  and the install steps cover it.
- **At least 5 GB of free disk** for the install folder. The first backup is
  about 1.1 GB. Older local copies are deleted automatically, keeping the
  newest 14.
- **A service account to run the job,** for example
  `SOUTHERNSTAR\svc-fulcrum-backup` on Windows or `fulcrum-backup` on Linux.
  It should be a normal account, not an administrator. It needs read and
  write access to the install folder.
- **The right timezone, or a known one.** The schedule is in Central time.
  Section 6 handles a VM that runs on UTC.

### 1.2 Values to collect

The config file has a placeholder like `<<SOUTHERN_STAR: SMTP sender address>>`
for each of these. The script refuses to make a real run until every one is
filled in, and it tells you exactly which are still missing.

| # | What | Who usually has it | Where it goes |
|---|---|---|---|
| 1 | The VM's hostname | IT | `site.vm_hostname` in the config |
| 2 | The install folder, e.g. `C:\FulcrumBackup` or `/opt/fulcrum-backup` | IT | `site.install_path` in the config |
| 3 | **Fulcrum API token** from a dedicated Fulcrum service account with the **Owner** role. It must be separate from the token the ArcGIS sync uses. | Myles Shade (Fulcrum owner) | Environment variable `FULCRUM_API_TOKEN`, **never in a file** |
| 4 | Production SFTP username. The `fulcrum_pd` account created in September works. | Brent | `sftp.accounts.prod.username` |
| 5 | Production SFTP password | Brent | Environment variable `SFTP_PROD_PASSWORD` |
| 6 | Production SFTP target directory. `.` means the account's home folder. | Brent | `sftp.accounts.prod.remote_dir` |
| 7 | SMTP sender address for the alerts, a From address your relay accepts | IT / mail team | `alerts.sender` |
| 8 | SMTP relay host name | IT / mail team | `alerts.smtp_host` |
| 9 | SMTP relay port, usually 25 for an internal relay | IT / mail team | `alerts.smtp_port` |
| 10 | SMTP security: `none`, `starttls` or `ssl` | IT / mail team | `alerts.security` |
| 11 | SMTP login, **only if the relay needs one** | IT / mail team | `alerts.username`, plus environment variable `SMTP_PASSWORD` |

For the UAT tests in section 5 you also need the **UAT SFTP password**, for
the `fulcrum_uat` account. It goes in the environment variable
`SFTP_UAT_PASSWORD`.

**How to create the Fulcrum token (Myles).** Create a dedicated user in
Fulcrum for the backup, give it the Owner role, sign in as it, and generate
an API token in its settings. Give the token straight to whoever installs
the backup; it doesn't need to go to Fulcrum.

### 1.3 Network access from the VM

| Destination | Port | Why | When |
|---|---|---|---|
| `api.fulcrumapp.com` | 443 | Fulcrum data | Always |
| `content.fulcrumapp.com` | 443 | Attachment downloads. It runs on **Amazon CloudFront, whose IP addresses change, so allow it by host name**, not IP. | Always |
| `shareit.southernstar.com` | 22 | SFTP upload | Always |
| Your SMTP relay | Usually 25 | Failure alerts | Always |
| `pypi.org`, `files.pythonhosted.org` | 443 | Installing the Python packages | Install and updates only. There's an offline option in 3.3 and 4.3 |
| Your Git host, e.g. `github.com` | 443 | Getting the code | Install and updates only, and only if the VM clones it directly |

---

## 2. Keep your own copy of the code

**Do this first, once.** The code is shared from a Fulcrum engineer's GitHub
account, and Southern Star should not depend on that repository staying
there. Copy it into a repository Southern Star controls, such as your
GitHub, Azure DevOps or GitLab, and install from **your** copy.

1. **Accept the GitHub invitation.** It was sent to Brent for
   `cristobalcastrillon/southern-star-fulcrum-backup`.
2. **Create an empty private repository** in Southern Star's own account or
   organization, for example `southern-star-fulcrum-backup`. Don't add a README,
   license or `.gitignore`; it must be completely empty.
3. **Clone Fulcrum's copy, rename that remote to `fulcrum`, and push
   everything to yours.** On any machine with Git, replacing
   `<YOUR-REPO-URL>` with the new repository's URL:

   ```bash
   git clone https://github.com/cristobalcastrillon/southern-star-fulcrum-backup.git
   cd southern-star-fulcrum-backup
   git remote rename origin fulcrum
   git remote add origin <YOUR-REPO-URL>
   git push -u origin main
   ```

   Your repository now has the full code and history. `origin` is yours;
   `fulcrum` is Fulcrum's copy.
4. **Optional: keep a single-file archive as well,** stored somewhere safe:

   ```bash
   git bundle create southern-star-fulcrum-backup.bundle --all
   ```

   You can clone from it later with `git clone southern-star-fulcrum-backup.bundle`,
   even if neither remote exists any more.

Later, if Fulcrum publishes a fix, bring it into your copy with
`git pull fulcrum main` and then `git push origin main`. Section 9.1 covers
updating the VM after that.

---

## 3. Install on Windows

The examples use the install folder `C:\FulcrumBackup` and the service
account `SOUTHERNSTAR\svc-fulcrum-backup`. Replace both with yours.
Run the commands in **PowerShell**.

### 3.1 Install Python 3.11

1. **Install Python.** Download the Python 3.11 Windows installer (64-bit)
   from https://www.python.org/downloads/windows/, or use your software
   distribution tool.
2. **Check two installer options:** "Install launcher for all users" and,
   under *Customize installation*, "Install for all users". The default
   location is `C:\Program Files\Python311`.
3. **Check it worked.** In a new PowerShell window:

   ```powershell
   py -3.11 --version
   ```

   **You should see:** `Python 3.11.x`. If `py` isn't found, the launcher
   wasn't installed: rerun the installer and choose *Modify*.

### 3.2 Get the code

**Either** clone it from your repository (section 2):

```powershell
git clone <YOUR-REPO-URL> C:\FulcrumBackup
```

**or**, if the VM has no Git, copy the repository folder to
`C:\FulcrumBackup`, including its hidden `.git` folder, so updates later can
be pulled.

Then give the service account modify rights on the folder:

```powershell
icacls C:\FulcrumBackup /grant "SOUTHERNSTAR\svc-fulcrum-backup:(OI)(CI)M"
```

### 3.3 Create the Python environment

```powershell
cd C:\FulcrumBackup
```

```powershell
py -3.11 -m venv .venv
```

```powershell
.venv\Scripts\python -m pip install -r requirements.txt
```

**You should see:** it ends with `Successfully installed ...`, listing
`requests`, `PyYAML`, `paramiko` and `tzdata`, among others.

**If the VM can't reach `pypi.org`,** download the packages on a Windows
machine that can, with the same Python 3.11:

```powershell
py -3.11 -m pip download -r requirements.txt -d wheels
```

Then copy the `wheels` folder into `C:\FulcrumBackup` and install from it:

```powershell
.venv\Scripts\python -m pip install --no-index --find-links wheels -r requirements.txt
```

### 3.4 Fill in the config

```powershell
Copy-Item config\config.example.yaml config\config.yaml
```

```powershell
notepad config\config.yaml
```

Fill in every `<<SOUTHERN_STAR: ...>>` value from the table in 1.2,
**except the token and the passwords**: `fulcrum.api_token` and
`sftp.accounts.prod.password`. Leave those two placeholders exactly as they
are, and leave `alerts.password` empty. They come from environment variables
(next step), which take the place of whatever is in the file.

A few rules for the file, which is YAML:
- **Indent with spaces, never tabs,** and keep the indentation as it is.
- **Keep the quotes** around values. Replacing the whole quoted placeholder
  is safest, for example
  `sender: "svc-fulcrum-backup@southernstar.com"`.
- **Write the port as a plain number:** `smtp_port: 25`.
- **`security` is `none`, `starttls` or `ssl`.** `none` is typical for an
  internal relay on port 25.
- **Leave `alerts.username` empty** if the relay doesn't need a login.

Don't change the `naming` section. The file and folder names must stay the
same as the samples already on the UAT landing zone.

`config\config.yaml` is never committed to Git; it's excluded on purpose.

### 3.5 Set the secrets

Secrets are **user environment variables of the service account**, so only
that account (and administrators) can read them. Open PowerShell **as the
service account**:

```powershell
runas /user:SOUTHERNSTAR\svc-fulcrum-backup powershell
```

In that new window, run each line below. Each one prompts for the value
without showing it.

```powershell
[Environment]::SetEnvironmentVariable('FULCRUM_API_TOKEN', [Net.NetworkCredential]::new('', (Read-Host 'Fulcrum API token' -AsSecureString)).Password, 'User')
```

```powershell
[Environment]::SetEnvironmentVariable('SFTP_PROD_PASSWORD', [Net.NetworkCredential]::new('', (Read-Host 'Production SFTP password' -AsSecureString)).Password, 'User')
```

For the UAT tests in section 5:

```powershell
[Environment]::SetEnvironmentVariable('SFTP_UAT_PASSWORD', [Net.NetworkCredential]::new('', (Read-Host 'UAT SFTP password' -AsSecureString)).Password, 'User')
```

Only if your SMTP relay needs a login:

```powershell
[Environment]::SetEnvironmentVariable('SMTP_PASSWORD', [Net.NetworkCredential]::new('', (Read-Host 'SMTP password' -AsSecureString)).Password, 'User')
```

**Close that window and open a new one as the service account** (the same
`runas` command); new variables only show up in new windows. To check they're
set without showing them:

```powershell
'FULCRUM_API_TOKEN','SFTP_PROD_PASSWORD','SFTP_UAT_PASSWORD','SMTP_PASSWORD' | ForEach-Object { "$_ is " + $(if ([Environment]::GetEnvironmentVariable($_, 'User')) { 'set' } else { 'NOT set' }) }
```

If the service account can't log on interactively, as with a group managed
service account, ask Fulcrum about the alternatives. Machine-level variables
work, but any user who logs on to the VM can read them.

### 3.6 Run the preflight check

In the window running as the service account:

```powershell
cd C:\FulcrumBackup; .venv\Scripts\python -m src.main preflight
```

Continue with [section 5](#5-test-it-uat), which explains the output.

---

## 4. Install on Linux

The examples use the install folder `/opt/fulcrum-backup` and the service
account `fulcrum-backup`. Replace both with yours. Commands starting with
`sudo` need an administrator; the rest run as the service account.

### 4.1 Install Python 3.11 and create the account

On RHEL, Rocky or Alma Linux 9:

```bash
sudo dnf install -y python3.11 git
```

On Ubuntu or Debian (Python 3.11 or newer):

```bash
sudo apt-get install -y python3.11 python3.11-venv git
```

Then create the service account:

```bash
sudo useradd --system --create-home --shell /bin/bash fulcrum-backup
```

Check Python:

```bash
python3.11 --version
```

**You should see:** `Python 3.11.x`, or newer if you installed a later version.

### 4.2 Get the code

```bash
sudo mkdir -p /opt/fulcrum-backup && sudo chown fulcrum-backup: /opt/fulcrum-backup
```

```bash
sudo -iu fulcrum-backup git clone <YOUR-REPO-URL> /opt/fulcrum-backup
```

If the VM has no Git access, copy the repository folder there instead,
including the hidden `.git` folder, and run
`sudo chown -R fulcrum-backup: /opt/fulcrum-backup`.

From here on, work as the service account:

```bash
sudo -iu fulcrum-backup
```

```bash
cd /opt/fulcrum-backup
```

### 4.3 Create the Python environment

```bash
python3.11 -m venv .venv
```

```bash
.venv/bin/python -m pip install -r requirements.txt
```

**You should see:** it ends with `Successfully installed ...`.

**If the VM can't reach `pypi.org`,** run
`python3.11 -m pip download -r requirements.txt -d wheels` on a Linux machine
that can, with the same Python version and CPU type. Copy `wheels/` over and
run:

```bash
.venv/bin/python -m pip install --no-index --find-links wheels -r requirements.txt
```

### 4.4 Fill in the config

```bash
cp config/config.example.yaml config/config.yaml
```

```bash
nano config/config.yaml
```

Same rules as the Windows section, [3.4](#34-fill-in-the-config): fill in every
`<<SOUTHERN_STAR: ...>>` value except the token and the passwords, keep the
quotes and the indentation, and don't change `naming`.

### 4.5 Set the secrets

Keep them in a file only the service account can read, outside the install
folder. As an administrator:

```bash
sudo install -d -m 700 -o fulcrum-backup /etc/fulcrum-backup
```

```bash
sudo -u fulcrum-backup install -m 600 /dev/null /etc/fulcrum-backup/secrets.env
```

```bash
sudo -u fulcrum-backup nano /etc/fulcrum-backup/secrets.env
```

Put one `NAME='value'` line per secret, with the value in single quotes:

```bash
FULCRUM_API_TOKEN='...'
SFTP_PROD_PASSWORD='...'
SFTP_UAT_PASSWORD='...'
# Only if the relay needs a login:
# SMTP_PASSWORD='...'
```

If a value itself contains a single quote, write it as `'\''`.

Load the secrets into your shell, as the service account, before running any
command by hand:

```bash
set -a && . /etc/fulcrum-backup/secrets.env && set +a
```

### 4.6 Run the preflight check

```bash
cd /opt/fulcrum-backup && .venv/bin/python -m src.main preflight
```

Continue with [section 5](#5-test-it-uat).

---

## 5. Test it (UAT)

The commands below are written for Windows. On Linux, use `.venv/bin/python`
instead of `.venv\Scripts\python`, and load the secrets first (4.5). Run
everything **from the install folder, as the service account**.

### 5.1 Preflight until it says READY

```powershell
.venv\Scripts\python -m src.main preflight
```

Preflight checks one item at a time and changes nothing:
- **SFTP:** it logs in and lists the target directory, but uploads nothing.
- **SMTP:** it asks the relay whether it would accept the email, then
  cancels, so nothing is sent.

Items still left as placeholders are marked SKIP and listed at the end.
A ready machine looks like this:

```
=== Preflight (PROD landing zone) ===
  [PASS] placeholders: none left
  [PASS] python: 3.11.8
  [PASS] timezone: America/Chicago, so cycles are named in Central time
  [PASS] staging folder: C:\FulcrumBackup\staging
  [PASS] state folder: C:\FulcrumBackup\state
  [PASS] logs folder: C:\FulcrumBackup\logs
  [PASS] machine: SSCGP-BKP01
  [PASS] install path: C:\FulcrumBackup
  [PASS] Fulcrum API: https://api.fulcrumapp.com reachable, token accepted, 21 apps visible
  [PASS] Fulcrum Query API: 213 tables
  [PASS] attachment downloads: content.fulcrumapp.com:443 reachable (HTTP 403; any answer is fine, real downloads use signed links)
  [PASS] SFTP login: shareit.southernstar.com:22 as fulcrum_pd, server key matches the pinned one
  [PASS] SFTP landing directory: / (12 entries)
  [PASS] SMTP relay: smtp.example.local:25 accepts svc-fulcrum-backup@southernstar.com -> myles.shade@southernstar.com, help@southernstar.com (nothing sent)

  RESULT: READY -- 14 pass
```

The numbers will differ. For anything other than PASS, see
[8.2](#82-preflight-items-that-dont-pass). To check the UAT account instead
of Production, add `--env uat`.

### 5.2 Test the failure alert

```powershell
.venv\Scripts\python -m src.main alert-test
```

This sends **one sample failure alert** with the real subject,
`[Fulcrum Backup] FAILED`, and the real From address. Its body starts with
`*** THIS IS A TEST ALERT`. Check two things:
- the email reached Myles and help@southernstar.com
- the Freshservice ticket was opened **and routed** the way your IT intends

The subject and the From address never change from run to run, so build the
routing on either or both.

**You should see:** the command ends with `RESULT: SENT`.

### 5.3 Dry run

```powershell
.venv\Scripts\python -m src.main run --dry-run
```

This pulls everything from Fulcrum into `staging\backup_<date>_<time>_CT`,
including every attachment, about 1.1 GB. **It uploads nothing and sends
no email.** It takes around 6 minutes.

**You should see:** `RESULT: COMPLETE (_SUCCESS written)`, then
`Dry run: the backup is in ...`.

Open that folder. It should look like the sample folders Fulcrum put on the
UAT landing zone, which have a `dev_` prefix only because they were test
runs:
- one `.csv` per table
- `attachments\`
- `_ATTACHMENTS.csv`, `_SCHEMA.csv`, `_MANIFEST.csv`
- `_SUCCESS`

You can delete the dry-run folder afterwards. It's never uploaded, and it's
cleaned up automatically after 14 newer backups.

### 5.4 Real runs to the UAT landing zone

The first run sends everything (about 1.1 GB):

```powershell
.venv\Scripts\python -m src.main run --env uat
```

**You should see:** `RESULT: DELIVERED (_SUCCESS written last)`, then exit code
0. A new folder `backup_<date>_<time>_CT` appears on the UAT SFTP.

**Incremental check (SOW acceptance criterion 5):**
1. **Add one attachment** to a test record in Fulcrum. Note its file size.
2. **Run the same command again.**
3. **Check the summary.** Its `attachments:` line should read
   `1 new (<that size> bytes), <N> already delivered`.
4. **Check the new folder.** Its `attachments\` folder on the SFTP should
   hold exactly that one file.

That proves each backup sends only new attachments.

### 5.5 First Production run

```powershell
.venv\Scripts\python -m src.main run
```

Without `--env`, it delivers to the account set in `delivery.env`, which is
`prod`. Like the first UAT run, the first Production run sends everything.
Then schedule it (section 6). The SOW's completion test is one unattended
Production run with no failures.

---

## 6. Schedule it

The backup must run at **8:00 AM and 8:00 PM Central**, including across
daylight-saving changes. Two runs never overlap: if a run is still going when
the next starts, the second one does nothing, exits with code 6 and sends an
alert. So **always let a new run start**, even if the last one is still
running; that's how a stuck run gets noticed.

### 6.1 Windows: Task Scheduler

First check the server's timezone:

```powershell
(Get-TimeZone).Id
```

- If it says **`Central Standard Time`**, Windows handles daylight saving
  itself. Use two triggers, at **8:00 AM and 8:00 PM**, and the program
  arguments **without** `--at-hours`.
- **Anything else** (for example `UTC`): use **four** daily triggers at the
  server's local times for 1:00, 2:00, 13:00 and 14:00 UTC, and **add**
  `--at-hours 8,20` to the arguments. On a UTC server those are simply
  1:00 AM, 2:00 AM, 1:00 PM and 2:00 PM.

  Only the two triggers that are really 8:00 and 20:00 Central go ahead. The
  others exit straight away and write nothing.

In **Task Scheduler**, choose *Create Task* (not *Create Basic Task*):

1. **General tab**
   - Name: `Fulcrum Backup`
   - *Change User or Group*: the service account
   - Select **Run whether user is logged on or not**
   - Leave *Run with highest privileges* unchecked
2. **Triggers tab:** add the two (or four) **Daily** triggers described above.
3. **Actions tab**, *New* > *Start a program*:
   - Program/script: `C:\FulcrumBackup\.venv\Scripts\python.exe`
   - Add arguments: `-m src.main run`, or `-m src.main run --at-hours 8,20`
   - Start in: `C:\FulcrumBackup`. **This one is required:** the script
     won't start without it.
4. **Settings tab**
   - *If the task is already running, then the following rule applies*:
     **Run a new instance in parallel**. The script itself stops the second
     one and sends the alert.
   - Leave *Stop the task if it runs longer than* at its default.
5. **OK.** Windows asks for the service account's password once, so the task
   can run while nobody is logged on.

**Check that the task sees the secrets** before relying on it. Temporarily
change the action to:
- Program/script: `C:\Windows\System32\cmd.exe`
- Add arguments: `/c .venv\Scripts\python.exe -m src.main preflight > logs\preflight-from-task.txt 2>&1`
- Start in: `C:\FulcrumBackup`

Right-click the task, choose **Run**, and open
`C:\FulcrumBackup\logs\preflight-from-task.txt`. It should end with
`RESULT: READY`.

If it says the token is missing, the task isn't loading the account's
environment variables. Recheck section 3.5 and that the task runs as the
same account. Then **set the action back** to `python.exe` and `-m src.main run`.

### 6.2 Linux: cron

Check the timezone with `timedatectl`. Then, as the service account, run
`crontab -e` and add **one** of these lines.

**For any cron, with the server on UTC or any other timezone.** This is the
recommended option:

```cron
0 1,2,13,14 * * * cd /opt/fulcrum-backup && set -a && . /etc/fulcrum-backup/secrets.env && set +a && .venv/bin/python -m src.main run --at-hours 8,20 >/dev/null 2>&1
```

8:00 AM Central is 13:00 UTC in summer and 14:00 UTC in winter; 8:00 PM
Central is 01:00 or 02:00 UTC. Cron starts the job at all four times, and
`--at-hours 8,20` lets through only the two that are really 8 and 20
Central. The others exit at once and write nothing.

**If your cron supports `CRON_TZ`** (cronie, as on RHEL), you can schedule in
Central time directly:

```cron
CRON_TZ=America/Chicago
0 8,20 * * * cd /opt/fulcrum-backup && set -a && . /etc/fulcrum-backup/secrets.env && set +a && .venv/bin/python -m src.main run >/dev/null 2>&1
```

The output is discarded because every run writes its own log file (7.2).

**Check that cron's environment sees the secrets.** Run the same command
line once by hand with `preflight` instead of `run ...`:

```bash
cd /opt/fulcrum-backup && set -a && . /etc/fulcrum-backup/secrets.env && set +a && .venv/bin/python -m src.main preflight
```

---

## 7. What a good run looks like

### 7.1 On screen and in the log

A successful run ends like this:

```
=== Export ===
  21 apps, 159 tables to export (21 app tables + 138 child tables); 32 unqueryable tables skipped
  folder:  C:\FulcrumBackup\staging\backup_20261104_2000_CT
  tables:  159 OK, 0 failed, 32 skipped (unqueryable field types, listed in the manifest)
  rows:    128918
  attachments: 3 new (2,481,113 bytes), 457 already delivered
  elapsed: 61s
  RESULT: COMPLETE (_SUCCESS written)

=== Delivery to PROD ===
  remote:  /backup_20261104_2000_CT  (PROD)
  files:   166 sent, 0 already there (resumed)
  bytes:   54,512,330
  elapsed: 40s
  state:   3 attachments recorded as delivered (C:\FulcrumBackup\state\attachments-prod.json)
  RESULT:  DELIVERED (_SUCCESS written last)
Log: C:\FulcrumBackup\logs\fulcrum-backup_run_20261104_200002.log
```

It exits with code 0. The "32 unqueryable tables skipped" line is expected:
those are attachment-field tables Fulcrum's query service doesn't serve. The
attachments themselves come through the attachment download instead.

### 7.2 Logs

Every run writes `logs\fulcrum-backup_run_<date>_<time>.log` in the install
folder, with every line timestamped. Logs older than 90 days are deleted
automatically (`logging.keep_days` in the config).

### 7.3 On the SFTP

Each run adds one folder, `backup_<YYYYMMDD>_<HHMM>_CT`, named in Central time:

| File | What it is |
|---|---|
| `<app>__<timestamp>.csv` | One per app, exactly as Fulcrum exports it |
| `<app>__<child>__<timestamp>.csv` | One per repeatable section, photo field and record link. They link to their record through `_record_id` |
| `attachments/` | The new attachment files, named `<app>__<record id>__<attachment id>.<ext>` |
| `_ATTACHMENTS.csv` | Every attachment the CSVs reference: its record, field and original file name, and which backup folder holds its file |
| `_SCHEMA.csv` | Every table's columns and types, for building SQL tables |
| `_MANIFEST.csv` | Every file with its row count, size and SHA-256 checksum |
| `_SUCCESS` | Written **last**. Load only folders that have it |

**Each attachment is sent once.** When you load a backup, copy its
attachments out, or keep the old folders. Retention on the SFTP is yours to
decide; the script never deletes anything there.

### 7.4 On the VM

| Folder | What it is | Can I delete it? |
|---|---|---|
| `staging\` | The local copy of each backup | Yes. Only the newest 14 are kept anyway |
| `state\` | Which attachments each SFTP account already has | **No.** Without it, the next run sends every attachment again (about 1.1 GB). Copy it if you move the install |
| `logs\` | One log per run | Yes. Kept 90 days anyway |
| `config\config.yaml` | Your settings | No |

---

## 8. When something fails

### 8.1 The alert email

A failed run sends **one** email to Myles Shade and help@southernstar.com,
with the subject `[Fulcrum Backup] FAILED`. The body has:
- the machine, the landing zone and the backup folder
- the exit code and what it means
- **every** table and attachment that failed, and why
- what to check
- the log file's path, and the log's last 60 lines

Dry runs never send email. If the alert itself can't be sent, for example
because the relay refuses the sender, the log says why. The run still exits
with its own code, so the scheduler records the failure either way.

**To re-run by hand,** as the service account from the install folder, run
`.venv\Scripts\python -m src.main run`. It's always safe to re-run:
- A folder that already has `_SUCCESS` on the SFTP is never touched.
- An interrupted upload resumes where it stopped.
- Attachments that failed are tried again by the next run.

To finish an interrupted upload without exporting again, run
`.venv\Scripts\python -m src.main deliver`.

### 8.2 Preflight items that don't pass

| Item | Status | What to do |
|---|---|---|
| placeholders | FAIL | Fill in each value listed at the end of the output (1.2, 3.4/4.4). For the token and passwords, set the environment variables (3.5/4.5). |
| python | FAIL | Install Python 3.11 or newer and recreate `.venv` (3.1, 3.3). |
| timezone | FAIL | Run the `pip install` again: the `tzdata` package is missing from `.venv`. |
| staging / state / logs folder | FAIL | Give the service account modify rights on the install folder (3.2). |
| machine / install path | WARN | `site.vm_hostname` or `site.install_path` doesn't match where it's running. Fix the config if the VM was renamed or the folder moved. |
| Fulcrum token | WARN | The token is in the config file. Set `FULCRUM_API_TOKEN` instead, and put the placeholder back in `fulcrum.api_token`. |
| Fulcrum API | FAIL "no token" | `FULCRUM_API_TOKEN` isn't set for this account. Open a new window after setting it (3.5). |
| Fulcrum API | FAIL "token rejected" | The token is wrong, revoked, or from another organization. Generate a new one (9.2). |
| Fulcrum API | FAIL "can't reach" | Open outbound 443 to `api.fulcrumapp.com` (1.3). |
| attachment downloads | FAIL | Open outbound 443 to `content.fulcrumapp.com`, **by host name** (1.3). |
| SFTP login | FAIL "timed out" or "refused" | Open outbound 22 to `shareit.southernstar.com`. |
| SFTP login | FAIL "authentication failed" | Wrong SFTP username or password, or the account is locked or disabled. |
| SFTP login | FAIL "host key MISMATCH" | The SFTP server's key changed. **Stop.** Confirm with the server's owner that the change is expected before updating `sftp.host_key_fingerprint`. |
| SFTP landing directory | FAIL | `remote_dir` doesn't exist for that account. |
| SMTP relay | FAIL "refused the sender" | Ask the mail team to allow `alerts.sender` from this VM. |
| SMTP relay | FAIL "refused the connection", "timed out" or "doesn't resolve" | Check `alerts.smtp_host`, `alerts.smtp_port` and the firewall. |
| SMTP relay | FAIL "rejected alerts.username / alerts.password" | Wrong SMTP login, or the relay doesn't need one: leave `alerts.username` empty. |

### 8.3 Exit codes

| Code | Meaning | What to do |
|---|---|---|
| 0 | Done | Nothing |
| 1 | Incomplete: a table failed, an upload was interrupted, or an attachment didn't download | The alert lists what. Re-run (8.1). A failed table means nothing was uploaded this time. A failed attachment means the backup was uploaded without it, and the next run tries it again |
| 2 | Configuration problem, including placeholders still in the config | The message names the setting |
| 3 | Fulcrum unreachable, or the token was rejected | See "Fulcrum API" in 8.2 |
| 4 | SFTP unreachable, login rejected, or host key changed | See "SFTP login" in 8.2 |
| 5 | Unexpected error | Send the log file to Fulcrum |
| 6 | Another run was still in progress, so this one did nothing | The alert names the run holding the lock and when it started. If that's hours ago, it's stuck: end that `python` process, which releases the lock, and re-run |

---

## 9. Routine tasks

### 9.1 Update the code

When there's a new version in your repository (section 2), as the service
account from the install folder, on Windows:

```powershell
git pull
```

```powershell
.venv\Scripts\python -m pip install -r requirements.txt
```

```powershell
.venv\Scripts\python -m src.main preflight
```

On Linux, use the same three commands with `.venv/bin/python`. Your
`config.yaml`, `state`, `staging` and `logs` folders aren't touched by an
update.

### 9.2 Rotate the Fulcrum token

1. **Create a new token** in Fulcrum, signed in as the backup's service
   account (or an owner).
2. **Put it in place.**
   - Windows: set `FULCRUM_API_TOKEN` again, as in 3.5.
   - Linux: replace it in `/etc/fulcrum-backup/secrets.env`.

   No restart is needed; the next run reads it.
3. **Run `preflight`** and check that the `Fulcrum API` line passes.
4. **Revoke the old token** in Fulcrum.

A run with a revoked token exits with code 3 and sends an alert. The SFTP
and SMTP passwords rotate the same way, with `SFTP_PROD_PASSWORD`,
`SFTP_UAT_PASSWORD` and `SMTP_PASSWORD`.

### 9.3 Other changes

- **Alert recipients, subject or sender:** edit `alerts.recipients`,
  `alerts.subject` and `alerts.sender` in the config, then run `alert-test`.
- **Send every attachment again,** for example after losing files on the
  SFTP: `.venv\Scripts\python -m src.main run --full-snapshot`.
- **Count and size the attachments without downloading:**
  `.venv\Scripts\python -m src.main attachments`.
- **Leave an app out:** add its exact name or form ID to
  `export.exclude_apps`. By default every app is backed up, including apps
  added later.
- **Keep more or fewer local backups:** change `export.keep_cycles`.
- **Move to another VM:** install there as in sections 3 or 4. Copy
  `config\config.yaml` and the whole `state\` folder across; without
  `state\`, the next run sends every attachment again. Set the secrets on
  the new VM.
- **Stop it:** disable the scheduled task or comment out the cron line.
  Nothing else runs in the background.

---

## 10. Reference

### 10.1 Commands

Run from the install folder as the service account. On Windows the command
starts with `.venv\Scripts\python -m src.main`; on Linux, with
`.venv/bin/python -m src.main`.

| Command | What it does |
|---|---|
| `run` | The scheduled job. It checks the SFTP, exports, then uploads, to the account set in `delivery.env` (`prod`) |
| `run --env uat` | The same, to the UAT account |
| `run --dry-run` | Exports to the VM only: no upload, no email |
| `run --at-hours 8,20` | Runs only at those hours, Central time; otherwise exits quietly (6.1, 6.2) |
| `run --full-snapshot` | Sends every attachment, not only new ones |
| `preflight` | Checks the machine, one item at a time; changes nothing (5.1) |
| `alert-test` | Sends one sample failure alert (5.2) |
| `deliver` | Uploads the newest backup not yet uploaded, for example to finish an interrupted one |
| `attachments` | Counts and sizes the attachments; downloads nothing |

Add `--help` to any command for its options.

### 10.2 Environment variables

| Variable | What it holds |
|---|---|
| `FULCRUM_API_TOKEN` | The Fulcrum service account's API token. Required |
| `SFTP_PROD_PASSWORD` | Production SFTP password. Required |
| `SFTP_UAT_PASSWORD` | UAT SFTP password, only for `--env uat` |
| `SMTP_PASSWORD` | SMTP password, only if the relay needs a login |

### 10.3 Contact

Cristóbal Castrillón, Fulcrum Professional Services:
cristobal.castrillon@fulcrumapp.com. When reporting a problem, include the
log file from `logs\`. It never contains passwords or the token; secrets
appear as `***`.
