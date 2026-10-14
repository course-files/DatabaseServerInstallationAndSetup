# Securing PostgreSQL: Towards Compliance with the Kenya Data Protection Act 2019

## Part I: Automated, Encrypted Backups (Optional Extension)

> **Optional.** This part is for students who want to learn how backups are set up in a professional environment. It follows Part H. Sections I.10 and I.11 contain rows for the lab's Compliance Summary Checklist and Troubleshooting table. All commands run over SSH on the Ubuntu Server VM.

**Why this matters.** In Part H you made a backup by hand. A backup that depends on a person remembering to run it will eventually not exist on the day it is needed. This part makes the backup run by itself, protects it with encryption, and proves that it can be restored.

**Legal basis.** s.41(4)(c) DPA 2019 (encryption), s.41(4)(d) (the ability to restore availability of and access to personal data in a timely manner), s.41(4)(e) (verifying that safeguards work) and s.39 (a backup is a copy of personal data and must not be kept longer than necessary). GDPR equivalents: Art. 32(1)(a), 32(1)(c), 32(1)(d) and Art. 5(1)(e).

By the end of this part you will be able to:

1. Explain the design of a professional scheduled backup and why each control exists.
2. Schedule an encrypted, verified, rotated backup with a `systemd` timer.
3. **Simulate** weeks of operation in a few minutes, including a deliberate failure.
4. Restore a backup using a private key that the server never held.

---

### I.0: The Design

```text
systemd timer  (the scheduler built into Linux: "run at 02:00 every day")
   ▼
systemd service  (runs the script as the Linux user "postgres"; alerts if it fails)
   ▼
siwaka_dishes_backup.sh
   │  pg_dump  ──►  gpg --encrypt (public key only)  ──►  file.dump.gpg
   ▼
/var/backups/siwaka_dishes/   (only "postgres" can enter; newest backups kept; older deleted)
```

| Design decision | Reason |
| --------------- | ------ |
| Linux scheduler (`systemd` timer), not `pg_cron` | `pg_cron` can run only SQL. To run `pg_dump` from it, the database must be allowed to run operating-system commands, a very powerful privilege. A Linux timer needs no such privilege. |
| Public-key encryption | The server holds only the **public** key, so it can encrypt backups but cannot decrypt them. Stealing the server does not expose the backups. |
| `pg_dump` piped into `gpg` | The unencrypted data never touches the disk. |
| Peer authentication (no password) | There is no stored password to steal, and the password expiry from Part F.3 cannot silently break the job. |
| Script owned by `root`, not writable by `postgres` | A compromised database account cannot rewrite the command that it later runs. |
| Write to `.partial`, rename when valid; delete old backups only after success | A failing job never leaves a broken file or destroys your last good backup. |

> **Where do tools like pgBackRest or Barman fit?** They add point-in-time recovery (restoring to any moment, not just to the last backup). This part uses a *logical* backup (`pg_dump`), which is the right starting point and is fully adequate for small databases. Section I.9 explains when to move to those tools.

---

### I.1: Preparation

Set the PostgreSQL version variable (repeat this if you open a new session) and create the backup directory. Only the `postgres` Linux user may enter it:

```bash
# Find the installed PostgreSQL version (needed for the configuration paths below)
PG_VER=$(ls /etc/postgresql/ | sort -V | tail -1)
echo "PostgreSQL version: ${PG_VER}"

# Create the directory. -d = directory, -m 0700 = only the owner has access,
# -o/-g = owner and group.
sudo install -d -m 0700 -o postgres -g postgres /var/backups/siwaka_dishes
sudo ls -ld /var/backups/siwaka_dishes
```

Expected: `drwx------ 2 postgres postgres ... /var/backups/siwaka_dishes`

---

### I.2: Create the Encryption Key Pair

An unattended job cannot type a passphrase, and storing one on the server would defeat the encryption. **Public-key encryption** solves this with two keys:

| Key | Where it lives | What it can do |
| --- | -------------- | -------------- |
| **Public key** | On the database server | Encrypt backups only |
| **Private key** | Off the server (in this lab: your own account) | Decrypt backups |

Generate the pair in **your own account**, not in the `postgres` account. You will be asked to choose a passphrase that protects the private key. Choose a strong one and keep it in a password manager:

```bash
# GPG_TTY lets gpg show its passphrase prompt in your terminal
export GPG_TTY=$(tty)

# Create an RSA key that can encrypt, valid for 1 year
gpg --pinentry-mode loopback --quick-generate-key \
    "Siwaka Dishes Backup <backup@siwakadishes.example>" rsa3072 encrypt 1y

# A key's "fingerprint" is its unique identity. The script will encrypt to this
# exact fingerprint, so that it can never pick the wrong key by mistake.
FPR=$(gpg --list-keys --with-colons backup@siwakadishes.example | awk -F: '/^fpr/ {print $10; exit}')
echo "${FPR}"
```

Give only the **public** key to the `postgres` account, then verify the separation. This is the whole point of the design:

```bash
# Export the public key and import it into the postgres account's keyring
gpg --armor --export "${FPR}" | sudo -u postgres env HOME=/var/lib/postgresql gpg --batch --import

# The postgres account now knows the public key ...
sudo -u postgres env HOME=/var/lib/postgresql gpg --list-keys

# ... and has NO secret key. This command must print nothing.
sudo -u postgres env HOME=/var/lib/postgresql gpg --list-secret-keys
```

Finally, make a copy of your private key and move it somewhere safe (your password manager or an encrypted drive on your laptop):

```bash
gpg --armor --export-secret-keys "${FPR}" > ~/siwaka_backup_private_key.asc
chmod 600 ~/siwaka_backup_private_key.asc      # only you may read it
# Then, from your LAPTOP's terminal:  scp student@<vm-ip>:siwaka_backup_private_key.asc .
```

> **Key loss is data loss.** If the private key or its passphrase is lost, every backup is permanently unreadable. That is exactly what protects you from thieves, and it is also the main operational risk of this design (see the reflection question in I.10).
> **Key expiry.** This key expires after one year. After that, encryption fails and the job reports a failure (it never silently produces unencrypted backups). Create a new key before then, and keep the old private key, because older backups still need it.
> **In production** the private key is created on an offline workstation, held by at least two named people, and **never placed on the database server**. In this lab it stays in your account only so that you can complete the restore drill.

---

### I.3: Connect to the Database Without a Password

The script runs as the Linux user `postgres` and connects as the read-only database role `siwaka_dishes_backup` (created in Part H). The common approach is a password file. That would leave on disk the password of a role that can read **every row**, and that password expires on 31 December 2026 (Part F.3), which would silently break your backups.

**Peer authentication with a user map** is better: the Linux identity of the caller proves who they are, and `pg_ident.conf` states which Linux users may act as which database roles.

```bash
sudo vim /etc/postgresql/${PG_VER}/main/pg_ident.conf
```

Add this line at the end. It reads: *under the map named `siwaka_backup`, the Linux user `postgres` may connect as the database role `siwaka_dishes_backup`*:

```text
siwaka_backup    postgres           siwaka_dishes_backup
```

```bash
sudo vim /etc/postgresql/${PG_VER}/main/pg_hba.conf
```

Add this line **immediately before** the generic `local   all   all   peer` line. PostgreSQL reads `pg_hba.conf` from top to bottom and stops at the first match (as in Part B.4), so the specific rule must come first:

```text
local   siwaka_dishes   siwaka_dishes_backup                    peer map=siwaka_backup
```

Apply the change and test it, including the **negative test**:

```bash
sudo systemctl reload postgresql

# 1. The postgres Linux user can connect as the backup role, with no password:
sudo -u postgres psql -U siwaka_dishes_backup -d siwaka_dishes -c "SELECT current_user;"

# 2. Your own account must NOT be able to:
psql -U siwaka_dishes_backup -d siwaka_dishes -c "SELECT 1;"
```

Expected from the second command: `FATAL:  Peer authentication failed for user "siwaka_dishes_backup"`.

An unattended backup fails at 02:00 on the first table the backup role cannot read, when nobody is watching. Fix the cause once, for current **and future** tables:

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
-- Everything that exists now
GRANT SELECT ON ALL TABLES    IN SCHEMA public TO siwaka_dishes_backup;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO siwaka_dishes_backup;

-- Everything that postgres creates in future
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT SELECT ON TABLES TO siwaka_dishes_backup;
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO siwaka_dishes_backup;

\q
```

---

### I.4: Install the Backup Script

Read the comments in the script before you install it: they explain every step. The command below creates the file for you (everything from `<<'SCRIPT'` to the final `SCRIPT` line is written into the file):

```bash
sudo tee /usr/local/bin/siwaka_dishes_backup.sh > /dev/null <<'SCRIPT'
#!/usr/bin/env bash
#
# siwaka_dishes_backup.sh
# Creates an encrypted backup of the siwaka_dishes database, checks it,
# and deletes the oldest backups so that only the newest few are kept.
#
# Runs as the Linux user "postgres" (systemd starts it; see the .service file).
# The number of backups to keep comes from the KEEP_COUNT environment variable
# (default: 7).
#
# DPA 2019 s.41(4)(c),(d),(e) and s.39  |  GDPR Art. 32(1)(a),(c) and Art. 5(1)(e)

# ---- Safety switches ---------------------------------------------------------
# -e  stop at the first command that fails
# -u  treat use of an undefined variable as an error (catches typing mistakes)
# -o pipefail  a pipeline fails if ANY command in it fails, not just the last one.
#              This matters below: if pg_dump fails in "pg_dump | gpg", we must stop.
# -E  make the ERR trap (below) also work inside functions
set -Eeuo pipefail

# New files are readable by their owner only (permissions 600).
umask 077

# ---- Settings (the only part you normally change) ----------------------------
DB_NAME="siwaka_dishes"
DB_USER="siwaka_dishes_backup"             # the read-only role from Part H
DB_SOCKET="/var/run/postgresql"            # a local Unix socket: no password needed
BACKUP_DIR="/var/backups/siwaka_dishes"
GPG_FINGERPRINT="__GPG_FINGERPRINT__"      # the PUBLIC key to encrypt with (set in I.5)
KEEP="${KEEP_COUNT:-7}"                    # "use KEEP_COUNT if set, otherwise 7"

# A service starts with a nearly empty environment, so set what we need.
export HOME="/var/lib/postgresql"          # gpg looks for its keyring in $HOME/.gnupg
export PATH="/usr/bin:/bin"

# ---- Check the input ---------------------------------------------------------
# Never trust a value that came from outside the script, even your own config.
if ! [[ "${KEEP}" =~ ^[0-9]+$ ]] || (( 10#${KEEP} < 1 || 10#${KEEP} > 365 )); then
    echo "ERROR: KEEP_COUNT must be a whole number from 1 to 365 (got '${KEEP}')" >&2
    exit 2
fi
KEEP=$((10#${KEEP}))                       # 10# stops values like 08 being read as octal

# ---- File names --------------------------------------------------------------
# UTC avoids confusion between time zones; the name sorts oldest-to-newest.
STAMP="$(date -u +%Y%m%dT%H%M%SZ)"
FINAL="${BACKUP_DIR}/siwaka_dishes_${STAMP}.dump.gpg"
PARTIAL="${FINAL}.partial"                 # work in progress; renamed when complete

# If anything goes wrong, say where, and never leave a half-written file behind.
trap 'echo "ERROR: backup failed at line ${LINENO}" >&2' ERR
trap 'rm -f -- "${PARTIAL}"' EXIT

echo "Starting backup of ${DB_NAME} (keeping the newest ${KEEP})"

# ---- Step 1: dump and encrypt in a single stream -----------------------------
# pg_dump writes the database to its standard output; the "|" passes it straight
# into gpg, which encrypts it. The unencrypted data never touches the disk.
#   --format=custom         compressed, and lets pg_restore pick parts to restore
#   --exclude-schema=cron   do not back up the pg_cron job tables
#   --no-password           fail instead of asking for a password nobody can type
#   --trust-model always    use the key we named without asking "do you trust it?"
#   --compress-algo none    the dump is already compressed; do not do it twice
pg_dump --host="${DB_SOCKET}" --username="${DB_USER}" --no-password \
        --format=custom --exclude-schema=cron "${DB_NAME}" \
| gpg --batch --yes --no-tty --trust-model always \
      --encrypt --recipient "${GPG_FINGERPRINT}" \
      --compress-algo none --output "${PARTIAL}"

# ---- Step 2: check the result before accepting it ----------------------------
# The file must exist, be non-empty, and really be public-key encrypted.
[[ -s "${PARTIAL}" ]] || { echo "ERROR: backup file is empty" >&2; exit 1; }
PACKETS="$(gpg --batch --no-tty --list-packets "${PARTIAL}" 2>/dev/null || true)"
[[ "${PACKETS}" == *"pubkey enc packet"* ]] \
    || { echo "ERROR: file is not a valid encrypted backup" >&2; exit 1; }

# ---- Step 3: publish --------------------------------------------------------
# Renaming is instant, so no other program ever sees a half-written backup.
mv -- "${PARTIAL}" "${FINAL}"
# Record a checksum: later, "sha256sum -c" proves the file has not been altered.
( cd "${BACKUP_DIR}" && sha256sum -- "${FINAL##*/}" > "${FINAL##*/}.sha256" )

# ---- Step 4: delete the oldest backups --------------------------------------
# Only now, after a NEW backup is safely in place, do we delete old ones.
# (A failing job therefore never destroys your last good backups.)
mapfile -t ALL < <(find "${BACKUP_DIR}" -maxdepth 1 -type f -name 'siwaka_dishes_*.dump.gpg' -printf '%f\n' | sort)
if (( ${#ALL[@]} > KEEP )); then
    for old in "${ALL[@]:0:${#ALL[@]}-KEEP}"; do
        rm -f -- "${BACKUP_DIR}/${old}" "${BACKUP_DIR}/${old}.sha256"
        echo "Deleted old backup: ${old}"
    done
fi

echo "Completed: ${FINAL##*/} ($(stat -c %s -- "${FINAL}") bytes). Backups kept: $(( ${#ALL[@]} > KEEP ? KEEP : ${#ALL[@]} ))"
SCRIPT
```

Set the owner and permissions, and insert your key fingerprint:

```bash
# root owns the script and nobody else may change it; everyone may run it (755).
# The postgres user can therefore RUN the script but cannot MODIFY it.
sudo chown root:root /usr/local/bin/siwaka_dishes_backup.sh
sudo chmod 755 /usr/local/bin/siwaka_dishes_backup.sh

# Replace the placeholder with your key's fingerprint
sudo sed -i "s/__GPG_FINGERPRINT__/${FPR}/" /usr/local/bin/siwaka_dishes_backup.sh
grep -n 'GPG_FINGERPRINT=' /usr/local/bin/siwaka_dishes_backup.sh
```

The last command must show your fingerprint, not `__GPG_FINGERPRINT__`. If it does not, the variable `FPR` is empty in this session: repeat the `FPR=$(...)` command from I.2.

> **Windows line endings.** If the script was copied through a Windows editor, Linux may report `bash\r: No such file or directory`. Repair it with `sudo sed -i 's/\r$//' /usr/local/bin/siwaka_dishes_backup.sh`.

**Run it once by hand**, as the account that will run it. Always test an automated job manually before you schedule it:

```bash
sudo -u postgres /usr/local/bin/siwaka_dishes_backup.sh
echo "exit status: $?"                          # 0 means success
sudo ls -lh /var/backups/siwaka_dishes/
```

Expected output (the size depends on your data). The `Z` in the file name means UTC, which is three hours behind East Africa Time:

```text
Starting backup of siwaka_dishes (keeping the newest 7)
Completed: siwaka_dishes_20261013T142307Z.dump.gpg (10478 bytes). Backups kept: 1
exit status: 0
```

Verify the checksum, and confirm that the account that made the backup **cannot read it**:

```bash
# The checksum proves the file has not changed since it was created
sudo -u postgres bash -c 'cd /var/backups/siwaka_dishes && sha256sum -c *.sha256'

# The postgres account has only the public key, so decryption must fail
LATEST=$(sudo sh -c 'ls -1 /var/backups/siwaka_dishes/*.dump.gpg' | sort | tail -1)
sudo -u postgres env HOME=/var/lib/postgresql gpg --batch --decrypt "${LATEST}" > /dev/null
```

Expected: `OK` for the checksum, then `gpg: decryption failed: No secret key`.

---

### I.5: Create the Scheduler Units

`systemd` uses small text files called **units**. You need three: a **service** (what to run), a **timer** (when to run it), and a second service that raises an **alert** if the first one fails.

```bash
# 1. The service: what to run, and as whom
sudo tee /etc/systemd/system/siwaka-backup.service > /dev/null <<'UNIT'
[Unit]
Description=Encrypted backup of the siwaka_dishes database
After=postgresql.service
OnFailure=siwaka-backup-failed.service

[Service]
Type=oneshot
User=postgres
Group=postgres
Environment=KEEP_COUNT=7
ExecStart=/usr/local/bin/siwaka_dishes_backup.sh
NoNewPrivileges=true
PrivateTmp=true
TimeoutStartSec=2h
UNIT
```

```bash
# 2. The failure alert. In production, replace the logger line with something
#    that notifies a person (an email, a Slack message). It is deliberately simple here.
sudo tee /etc/systemd/system/siwaka-backup-failed.service > /dev/null <<'UNIT'
[Unit]
Description=Alert: the siwaka_dishes backup failed

[Service]
Type=oneshot
ExecStart=/usr/bin/logger -t siwaka_dishes_backup -p user.crit "BACKUP FAILED. Run: journalctl -u siwaka-backup.service"
UNIT
```

```bash
# 3. The timer: when to run. It runs the service of the same name.
sudo tee /etc/systemd/system/siwaka-backup.timer > /dev/null <<'UNIT'
[Unit]
Description=Run the siwaka_dishes backup every day at 02:00 (East Africa Time)

[Timer]
OnCalendar=*-*-* 02:00:00 Africa/Nairobi
Persistent=true

[Install]
WantedBy=timers.target
UNIT
```

What the important lines mean:

| Line | Meaning |
| ---- | ------- |
| `Type=oneshot` | The service runs once and finishes (it is not a long-running program) |
| `User=postgres` | Run as the `postgres` Linux user, **not** root |
| `Environment=KEEP_COUNT=7` | The number of backups to keep. This is how you configure the script without editing it |
| `OnFailure=...` | If the service fails, start the alert service |
| `NoNewPrivileges`, `PrivateTmp` | Basic hardening: the job cannot gain extra privileges and gets its own private `/tmp` |
| `OnCalendar=*-*-* 02:00:00 Africa/Nairobi` | Every day at 02:00 East Africa Time. The time zone is written explicitly, so the server's own time zone does not matter |
| `Persistent=true` | If the server was off at 02:00, run the backup as soon as it starts. Cron-style schedulers silently skip the missed run |

Tell `systemd` about the new files and check them for mistakes. `systemd-analyze` prints nothing when the files are correct:

```bash
sudo systemctl daemon-reload
sudo systemd-analyze verify /etc/systemd/system/siwaka-backup.service \
                            /etc/systemd/system/siwaka-backup-failed.service \
                            /etc/systemd/system/siwaka-backup.timer

# Ask systemd when the schedule will next fire (no need to wait to find out!)
systemd-analyze calendar '*-*-* 02:00:00 Africa/Nairobi'
```

Run the service **once through systemd** (rather than by hand) to prove that the whole chain works. It runs in the foreground and returns when the backup is done:

```bash
sudo systemctl start siwaka-backup.service
sudo systemctl status siwaka-backup.service --no-pager
sudo journalctl -u siwaka-backup.service --no-pager -n 5
```

`systemctl status` should show `Result: success` (or `status=0/SUCCESS`), and the journal (the system log) shows the script's own messages.

---

### I.6: Simulate Weeks of Operation in Minutes

A production backup runs once a day. You cannot wait a week in a lab session to see rotation, and you should never discover a failure mode for the first time in production. **Compress time:** run the same script every minute and keep only 3 backups.

| Production | Simulation |
| ---------- | ---------- |
| Daily at 02:00 | Every minute |
| Keeps 7 backups | Keeps 3 backups |
| Rotation visible after 8 days | Rotation visible after 4 minutes |

> **Do this before you enable the real timer (I.8).** With `KEEP_COUNT=3`, the script deletes every backup older than the three newest.

`systemd-run` creates a **temporary** timer and service (they disappear when stopped or on reboot), so you do not need to edit the real units:

```bash
sudo systemd-run \
    --unit=siwaka-backup-sim \
    --on-calendar=minutely \
    --property=Type=oneshot \
    --property=User=postgres \
    --property=OnFailure=siwaka-backup-failed.service \
    --setenv=KEEP_COUNT=3 \
    /usr/local/bin/siwaka_dishes_backup.sh
```

Open two terminals to watch it. In the first, follow the job's log (`-f` = follow; press `Ctrl+C` to stop):

```bash
sudo journalctl -u siwaka-backup-sim.service -f
```

In the second, watch the backup files and the timer. After about four minutes you will see the log line `Deleted old backup: ...`, and **exactly three** `.dump.gpg` files will remain. That is rotation: the implementation of s.39, so that copies of personal data do not accumulate indefinitely.

```bash
sudo watch -n 5 "ls -l --time-style=+%H:%M:%S /var/backups/siwaka_dishes | grep 'dump.gpg\$'"
systemctl list-timers siwaka-backup-sim.timer --no-pager     # when it last ran and runs next
```

#### Break it on purpose

A backup system has not been tested until you have watched it fail. Simulate a realistic fault: an administrator carelessly changes the backup role.

```bash
sudo -u postgres psql -c "ALTER ROLE siwaka_dishes_backup NOLOGIN;"
```

Within a minute the first terminal shows the cause directly (systemd keeps the script's error messages):

```text
pg_dump: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  role "siwaka_dishes_backup" is not permitted to log in
ERROR: backup failed at line 66
```

Check the three consequences of the failure:

```bash
systemctl is-failed siwaka-backup-sim.service                    # prints: failed
sudo journalctl -t siwaka_dishes_backup --no-pager -n 3          # the alert from the OnFailure unit
sudo ls -a /var/backups/siwaka_dishes | grep -c partial          # prints 0: no half-written file
```

The earlier good backups are still there (check the second terminal): a failed run never deletes anything. Now repair the fault and watch the job recover by itself on the next minute:

```bash
sudo -u postgres psql -c "ALTER ROLE siwaka_dishes_backup LOGIN;"
```

#### Clean up the simulation

```bash
sudo systemctl stop siwaka-backup-sim.timer
sudo systemctl reset-failed siwaka-backup-sim.service 2>/dev/null || true
systemctl list-timers --no-pager | grep -c siwaka-backup-sim      # prints 0
```

Stop the two watching terminals with `Ctrl+C`.

---

### I.7: Verify a Backup by Restoring It

**A backup that has never been restored is a hope, not a backup.** This drill proves that a backup made without human involvement can be recovered, using a private key that the server never held. If your backup directory is empty, create one first with `sudo systemctl start siwaka-backup.service`.

```bash
# Make files created in this drill private to you (the decrypted data is NOT encrypted)
umask 077

# 1. Check integrity, then copy the newest backup into your account
LATEST=$(sudo sh -c 'ls -1 /var/backups/siwaka_dishes/*.dump.gpg' | sort | tail -1)
sudo sh -c "cd /var/backups/siwaka_dishes && sha256sum -c $(basename "${LATEST}").sha256"
sudo cp "${LATEST}" ~/restore_drill.dump.gpg
sudo chown "$USER":"$USER" ~/restore_drill.dump.gpg

# 2. Decrypt with your PRIVATE key (you will be asked for its passphrase)
export GPG_TTY=$(tty)
gpg --output ~/restore_drill.dump --decrypt ~/restore_drill.dump.gpg

# 3. Look inside without restoring anything (the first 25 entries)
pg_restore --list ~/restore_drill.dump | head -25
```

The backup records the `pg_cron` **extension** from Part G. That extension can exist only in `siwaka_dishes`, so restoring it into a scratch database would produce errors, and a drill that "mostly works" teaches nothing. The command below saves the list of contents minus that entry, and restores from that edited list:

```bash
# 4. Make an edited table of contents without the pg_cron entries
pg_restore --list ~/restore_drill.dump | grep -v -E '(EXTENSION|COMMENT).*pg_cron' > /tmp/restore_drill.toc
chmod 644 /tmp/restore_drill.toc               # so that the postgres user can read it

# 5. Restore into a new, empty scratch database.
#    "< file" feeds the dump to pg_restore, because the postgres user cannot read your home folder.
sudo -u postgres createdb siwaka_dishes_restore_test
sudo -u postgres pg_restore --use-list=/tmp/restore_drill.toc \
                            --dbname=siwaka_dishes_restore_test < ~/restore_drill.dump

# 6. Compare with production. Both numbers must be identical (if no data changed since the backup).
sudo -u postgres psql -At -d siwaka_dishes_restore_test -c "SELECT count(*) FROM customer;"
sudo -u postgres psql -At -d siwaka_dishes              -c "SELECT count(*) FROM customer;"
```

A clean restore prints nothing at step 5. Any line that begins with `pg_restore: error` must be investigated before you declare the backup good. Finally, remove everything that contains unencrypted personal data:

```bash
sudo -u postgres dropdb siwaka_dishes_restore_test
shred -vfzu ~/restore_drill.dump ~/restore_drill.dump.gpg /tmp/restore_drill.toc
```

> **Backups and the right to erasure.** s.40(1)(b) DPA 2019 gives data subjects a right to erasure, but you cannot reasonably edit one record inside an encrypted backup. The accepted practice is (1) to keep backup retention short and documented, so erased data cannot survive for long, and (2) after **any** restore, to re-apply every erasure request processed since the backup date before the restored database is used. Record this procedure in your data protection documentation.

---

### I.8: Switch On the Real Schedule

Now enable the production schedule: **daily at 02:00 East Africa Time, keeping 7 backups**.

```bash
# enable = start automatically at every boot;  --now = also start it immediately
sudo systemctl enable --now siwaka-backup.timer

# Confirm: the NEXT column shows tomorrow at 02:00
systemctl list-timers siwaka-backup.timer --no-pager
```

Record the retention decision in the governance table you created in Part G.1, because backups are personal data:

```bash
sudo -u postgres psql -d siwaka_dishes
```

```sql
INSERT INTO data_retention_policy
VALUES ('database_backups', TRUE, FALSE, 7,
        'DPA 2019 s.39(2), s.41(4)(d) — encrypted copies kept for disaster recovery only; oldest deleted after 7 daily backups',
        'DELETE', 'Data Protection Officer', CURRENT_DATE);
\q
```

> If you later change the schedule or `KEEP_COUNT`, update this row: the policy and the running job must agree. To change the schedule, edit the unit file and run `sudo systemctl daemon-reload`.

---

### I.9: Monitoring, and Where This Design Stops

The most dangerous backup system is one that has failed and told nobody. The `OnFailure` alert covers a job that runs and fails. It cannot cover a job that **never runs** (for example, a disabled timer). So also check *freshness*: is there a recent backup?

```bash
# Is there a backup file from the last 26 hours (24-hour schedule + 2 hours of tolerance)?
if sudo find /var/backups/siwaka_dishes -name '*.dump.gpg' -mmin -1560 | grep -q .; then
    echo "OK: a recent backup exists"
else
    echo "ALERT: no backup in the last 26 hours"
fi
```

In production, run this check from a monitoring system (for example Zabbix or Prometheus) and send the alert to a **named person**. A check that nobody reads is not monitoring. This verification is itself the safeguard that s.41(4)(e) requires.

**Limits of this design, and what professionals add:**

| Limitation | What is done in production |
| ---------- | -------------------------- |
| Backups are on the **same server** as the database. A fire, disk failure or ransomware destroys both. | The **3-2-1 rule**: three copies, on two kinds of media, one off-site. The backups are already encrypted, so copying them is safe. |
| Encrypted backups are **still personal data**. | Copying them to storage outside Kenya, such as a cloud region abroad, engages ss.48–49 DPA 2019. Encryption is a safeguard you can cite, not an exemption. |
| A daily backup can lose up to **24 hours** of data (the Recovery Point Objective). | A business decision for the Data Protection Officer. Where 24 hours is too much, use continuous WAL archiving with **pgBackRest** or **Barman**, which can restore to any moment. |
| Anyone with `root`, or the `postgres` account, can still tamper with the job or read the data. | Restricted administrator access, logging, and separation of duties. |

---

### I.10: Checklist Rows and Deliverables

Add these rows to the lab's Compliance Summary Checklist (numbering continues from row 17):

| # | Control | Command to Verify | DPA 2019 | GDPR |
| --- | ------- | ----------------- | -------- | ---- |
| 18 | Server cannot decrypt backups | `sudo -u postgres env HOME=/var/lib/postgresql gpg --list-secret-keys` → empty | s.41(4)(c) | Art. 32(1)(a) |
| 19 | No stored backup password | `pg_ident.conf` contains `siwaka_backup`; `sudo ls /var/lib/postgresql/.pgpass` → no such file | s.41(4)(a) | Art. 32(1)(b) |
| 20 | Script cannot be modified by the database account | `ls -l /usr/local/bin/siwaka_dishes_backup.sh` → `-rwxr-xr-x root root` | s.41(4)(b) | Art. 32(1)(b) |
| 21 | Backups run on a schedule | `systemctl list-timers siwaka-backup.timer` | s.41(4)(d) | Art. 32(1)(c) |
| 22 | Backups are rotated per documented retention | `sudo ls /var/backups/siwaka_dishes`; the `data_retention_policy` row | s.39 | Art. 5(1)(e) |
| 23 | A backup has been restored successfully | I.7 output: no errors, matching counts | s.41(4)(d), (e) | Art. 32(1)(c), (d) |
| 24 | A failed backup is detected | I.6 failure output and the I.9 freshness check | s.41(4)(e), (f) | Art. 32(1)(d) |

Additional Lab Deliverables:

| # | Deliverable | What to include |
| --- | ----------- | --------------- |
| 10 | **Key Separation** | Screenshots of an **empty** `gpg --list-secret-keys` for `postgres` (I.2) and of the failed decryption by `postgres` (I.4) |
| 11 | **Simulation Evidence** | (a) The journal showing at least four runs and a `Deleted old backup` line. (b) The directory listing showing exactly three backups. (c) The failed run **together with** its `pg_dump: error` line |
| 12 | **Restore Drill** | The `pg_restore` command with its (empty) output, and matching `COUNT(*)` results |
| 13 | **Written Reflection (300 words)** | *"The private key is held by one person, who leaves the organisation. Describe what could go wrong, and design a key custody procedure (who holds the key, how it is recovered, how it is rotated) that prevents it."* |

---

### I.11: Troubleshooting: Additional Rows

| Symptom | Likely Cause | Resolution |
| ------- | ------------ | ---------- |
| `Peer authentication failed for user "siwaka_dishes_backup"` | The caller is not the `postgres` Linux user, or the configuration change was not applied | Check both files in I.3 (the map line must come **before** `local all all peer`), then `sudo systemctl reload postgresql` |
| `pg_dump: error: ... permission denied for table <name>` | A table was created by another role after the last grant | Repeat the grants in I.3 |
| `gpg: <fingerprint>: skipped: No public key` | The public key is not in the `postgres` keyring, or the fingerprint in the script is wrong | Repeat the import in I.2; compare `grep GPG_FINGERPRINT= /usr/local/bin/siwaka_dishes_backup.sh` with `echo $FPR` |
| `/usr/bin/env: 'bash\r': No such file or directory` | The script has Windows line endings | `sudo sed -i 's/\r$//' /usr/local/bin/siwaka_dishes_backup.sh` |
| `ERROR: KEEP_COUNT must be a whole number from 1 to 365` | A mistake in `Environment=KEEP_COUNT=` | Correct the unit file, then `sudo systemctl daemon-reload` |
| `systemd-analyze verify` prints a warning | A typo in a unit file | Read the line number in the message and compare it with I.5 |
| `gpg: public key decryption failed: No secret key` | You are decrypting in an account without the private key | Expected for `postgres`. Decrypt in the account that holds the private key |
| `Inappropriate ioctl for device` while decrypting | gpg cannot show its passphrase prompt | Run `export GPG_TTY=$(tty)` first |
| The timer never fires | The timer is not enabled | `sudo systemctl enable --now siwaka-backup.timer`, then `systemctl list-timers` |
| Need the reason for a failure | `systemd` stores only the exit status in its summary | `sudo journalctl -u siwaka-backup.service --since "2 days ago"` shows the script's own error messages |

---

### I.12: Further Reading

- **systemd timers and `OnCalendar` syntax:** [https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html) and [https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html)
- **`pg_dump` and `pg_restore`:** [https://www.postgresql.org/docs/current/app-pgdump.html](https://www.postgresql.org/docs/current/app-pgdump.html) and [https://www.postgresql.org/docs/current/app-pgrestore.html](https://www.postgresql.org/docs/current/app-pgrestore.html)
- **`pg_hba.conf` and user maps:** [https://www.postgresql.org/docs/current/auth-username-maps.html](https://www.postgresql.org/docs/current/auth-username-maps.html)
- **Backup tooling with point-in-time recovery:** pgBackRest [https://pgbackrest.org](https://pgbackrest.org) and Barman [https://pgbarman.org](https://pgbarman.org)
- **GnuPG manual:** [https://www.gnupg.org/documentation/](https://www.gnupg.org/documentation/)
