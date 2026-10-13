# Vaultwarden on a home server

Self-hosted Bitwarden-compatible password manager, reachable only over Tailscale (HTTPS),
with a daily encrypted backup to Google Drive.

```
Your devices ──(Tailscale)──> vault-tailscale (HTTPS :443) ──> vaultwarden (:80)
                                                                   │
                                          vaultwarden-backup ──> Google Drive (7z, encrypted)
```

| File | Purpose |
|---|---|
| `docker-compose.yml` | Tailscale sidecar + Vaultwarden + backup container |
| `serve.json` | Tells the sidecar to serve HTTPS and proxy to Vaultwarden |
| `.env.example` | Template for secrets/settings. Copy to `.env` (git-ignored) |

No host ports are published, so nothing clashes with other services on the machine.

## Prerequisites

- Docker + Docker Compose on the host
- A Tailscale account with **MagicDNS** and **HTTPS certificates** enabled
  (Admin console -> DNS -> Enable HTTPS)
- A Google account for backups
- A Mac/PC with a browser (for the rclone login step)

---

## 1. Create a Tailscale auth key

1. Admin console -> **Settings -> Keys -> Generate auth key**.
2. Leave "Reusable" and "Ephemeral" off. Copy the key (`tskey-auth-...`).

## 2. Configure `.env`

```bash
cp .env.example .env
chmod 600 .env
```

Edit `.env`:

- `TS_AUTHKEY` - the key from step 1
- `VAULT_DOMAIN` - `vault.<your-tailnet>.ts.net` (if the name `vault` is already taken in
  your tailnet, Tailscale appends a suffix such as `vault-1`; use the real name shown in the
  admin console)
- `SIGNUPS_ALLOWED` - `true` only for the very first account
- `ZIP_PASSWORD` - long random string. **Save a copy outside Vaultwarden** (see Disaster recovery).

## 3. Start the stack

```bash
docker compose up -d
docker logs vault-tailscale --tail 30     # should show it joined the tailnet
```

Open `https://<VAULT_DOMAIN>`. The first request can take a few seconds while the
certificate is issued.

## 4. After first start

1. Tailscale admin console -> Machines -> `vault` -> menu -> **Disable key expiry**
   (otherwise the node key expires after 180 days and the vault becomes unreachable).
2. Remove `TS_AUTHKEY` from `.env` and delete the auth key in the console. It is only used once.
3. Create your account at the web vault, then set `SIGNUPS_ALLOWED=false` in `.env` and run
   `docker compose up -d`.

---

## 5. Configure rclone with Google Drive

The backup container uploads through rclone. Its Google token is stored in the
`vaultwarden-rclone-data` volume.

**5.1 Install rclone on your Mac** (needed to complete the Google login):

```bash
brew install rclone
```

**5.2 Start the interactive config inside the container's volume:**

```bash
docker run --rm -it \
  -v vaultwarden-rclone-data:/config/ \
  ttionya/vaultwarden-backup:latest rclone config
```

**5.3 Answer the prompts** (option numbers can differ slightly between rclone versions):

| Prompt | Answer |
|---|---|
| `n/s/q>` | `n` (new remote) |
| `name>` | `VaultwardenGDrive` (must match `RCLONE_REMOTE_NAME`) |
| `Storage>` | `drive` (Google Drive) |
| `client_id>` | Enter (blank) |
| `client_secret>` | Enter (blank) |
| `scope>` | `1` (full access to Drive) |
| `service_account_file>` | Enter (blank) |
| `Edit advanced config?` | `n` |
| `Use web browser to automatically authenticate?` | **`n`** (the container is headless) |

rclone now prints a command like:

```
rclone authorize "drive" "eyJ...REDACTED..."
```

**5.4 On your Mac**, run exactly that printed command. A browser opens: sign in to Google and
allow access. The terminal then prints a token block (`{"access_token": ...}`).

**5.5 Back in the container prompt**, paste that token at `config_token>`, then:

| Prompt | Answer |
|---|---|
| `Configure this as a Shared Drive?` | `n` |
| `Keep this remote?` | `y` |
| `e/n/d/r/c/s/q>` | `q` |

**5.6 Verify the remote exists:**

```bash
docker run --rm -v vaultwarden-rclone-data:/config/ \
  ttionya/vaultwarden-backup:latest rclone listremotes
# expected: VaultwardenGDrive:
```

> Tip: with a personal Google app (your own `client_id`/`client_secret`) tokens are more
> reliable than with rclone's shared default. Optional, but worth it if you see auth errors.

## 6. Test the backup

```bash
docker compose up -d                       # make sure the backup container is running
docker exec vaultwarden-backup bash /app/backup.sh
docker logs vaultwarden-backup --tail 30
```

Check that a `.7z` file appeared in Google Drive under `VaultwardenBackup/`.
Backups then run daily at 02:00 (`CRON` in the compose file).

## 7. Test a restore (do this once, on a scratch volume)

An untested backup is not a backup.

1. Download a backup file from Drive into an empty folder, e.g. `~/vw-restore/`.
2. Restore into a **throwaway** volume, not your live one:

   ```bash
   docker run --rm -it \
     -v vw-restore-test:/bitwarden/data/ \
     -v ~/vw-restore:/bitwarden/restore/ \
     ttionya/vaultwarden-backup:latest \
     restore --zip-file <backup-file>.7z --password '<ZIP_PASSWORD>'
   ```
3. Start a temporary Vaultwarden against `vw-restore-test` and confirm your items are there.
4. Delete the test volume: `docker volume rm vw-restore-test`.

Restore flags can change between image versions; check the
[image's README](https://github.com/ttionya/vaultwarden-backup) if the command is rejected.

---

## Disaster recovery (host died)

1. Install Docker on the new host; join it to Tailscale if you also want host access.
2. Clone this repo, copy `.env.example` to `.env`, fill in values (new auth key; use the
   **same** `ZIP_PASSWORD` as before).
3. Recreate the rclone remote (section 5) so you can reach Drive.
4. Download the latest backup from Drive and restore it into the `vaultwarden-data` volume
   (section 7 command, with `-v vaultwarden-data:/bitwarden/data/`) **before** the first
   `docker compose up -d`.
5. `docker compose up -d`, then log in and verify.
6. In Tailscale, remove the old `vault` machine first if the name is taken, so the new one
   gets the same hostname.

**Keep these outside Vaultwarden** (printed, or in a second password manager):
`ZIP_PASSWORD`, your Google account login/recovery, and your Tailscale login.

## Importing an existing Bitwarden export

1. Web vault -> **Tools -> Import data**.
2. Format: **Bitwarden (zip)** (includes attachments). Enter the export password if it has one.
3. Import once only; imports add items, they do not overwrite.

If the zip option is missing: unzip, import `data.json` as **Bitwarden (json)**, and
re-attach files manually or with `bw create attachment`.
Note: an *account-restricted* export can only be imported into the same account.

## Moving data from another Vaultwarden server

On the old server:

```bash
docker compose stop vaultwarden
docker run --rm -v vaultwarden-data:/data -v "$PWD":/out alpine \
  tar czf /out/vw-data.tgz -C /data .
```

Copy `vw-data.tgz` to the new host, then:

```bash
docker volume create vaultwarden-data
docker run --rm -v vaultwarden-data:/data -v "$PWD":/in alpine \
  tar xzf /in/vw-data.tgz -C /data
docker compose up -d
```

Delete the `.tgz` on both machines afterwards.

## Update the Bitwarden clients

Log out, set the server URL to `https://<VAULT_DOMAIN>` (Settings -> Self-hosted), and log in
again on each app, browser extension, and desktop client.

## Updating

```bash
docker exec vaultwarden-backup bash /app/backup.sh   # backup first
docker compose pull
docker compose up -d
```

## Security notes

- `.env` holds secrets and is git-ignored. Never commit it or paste it in chats/issues.
- Vaultwarden has no published ports; access is limited to your tailnet. Consider Tailscale
  ACLs so only your own devices can reach `vault`.
- `SIGNUPS_ALLOWED` stays `false` except while creating the first account.
- Backups are encrypted (7z + `ZIP_PASSWORD`), but the vault itself is also protected by your
  master password. Both matter.

## Troubleshooting

| Symptom | Check |
|---|---|
| Site not reachable | `docker logs vault-tailscale`; is `vault` online in the admin console? Is key expiry disabled? |
| Certificate error | HTTPS certificates enabled in the Tailscale DNS settings? Wait a few seconds on first load |
| Login/links use wrong URL | `VAULT_DOMAIN` must match the real tailnet name of the sidecar |
| Backup fails with auth error | Redo section 5; token may have been revoked |
| `vault` name has a `-1` suffix | Old machine still registered; delete it in the admin console, then recreate the sidecar |
| Chrome page hangs but apps work | Seen once with the old VPS setup; test from another device and browser before blaming the server |
