# IMAP over XOAUTH2

Some providers accept IMAP only with an OAuth2 access token, which expires
after about an hour. This directory is a compose project for one such
account: mbsync gets the token through `pass_cmd` from a helper script that
refreshes it on every connection and writes the new token back to disk.

Nothing in this directory holds a secret except the files listed under
"never commit". The `.gitignore` at the repository root keeps them out.

## Files

| File | Commit? | Purpose |
|---|---|---|
| `compose.yaml` | yes | The service, with `./oauth` mounted writable. |
| `accounts.example.json` | yes | Template. Copy to `accounts.json`. |
| `.env.example` | yes | Template. Copy to `.env`. |
| `renew-token.sh` | yes | One-command token renewal, see below. |
| `accounts.json` | never | Your address and folder names. |
| `.env` | never | `PUBLIC_URL`, passphrase. |
| `renew.env` | never | Where `renew-token.sh` copies the token. |
| `oauth/mutt_oauth2.py` | never | Token helper, see below. |
| `oauth/token.txt` | never | Refresh token and current access token. |
| `mail/`, `index/` | never | The mirror and the notmuch index. |

## Token helper

The helper is `mutt_oauth2.py` from the mutt project
(`contrib/mutt_oauth2.py` in https://github.com/muttmua/mutt). Download it
to `oauth/` and change two things:

1. `ENCRYPTION_PIPE = ['cat']` and `DECRYPTION_PIPE = ['cat']`. The container
   has no GPG, so the token file is stored as plain JSON; the file mode
   protects it instead.
2. In the registration for your provider, fill in the OAuth client id (and
   secret, for a confidential client) that your provider admits for IMAP.

Create the token once, on a machine with a browser:

```sh
TZ=UTC python3 oauth/mutt_oauth2.py oauth/token.txt --authorize --authflow devicecode
```

`TZ=UTC` matters: the helper stores the token's expiry as a naive local time,
and the container runs in UTC. Written on a machine an hour ahead of UTC, the
container would keep using an expired token for that hour.

Answer the registration name, `devicecode` and your address; then open the
URL it prints and enter the code. Afterwards every call without
`--authorize` prints a current access token, refreshing it when needed. That
is what `pass_cmd` runs. The helper refuses a token file whose mode is not
`600`.

Check the chain inside the image before starting the server:

```sh
docker compose run --rm --entrypoint python3 your-mail-mcp \
  /oauth/mutt_oauth2.py /oauth/token.txt
```

## Account file

`accounts.example.json` shows the shape. Two settings are not optional:

- `"auth_mechs": ["XOAUTH2"]` so mbsync offers only that mechanism.
- `exclude_folders` set explicitly. SPECIAL-USE discovery logs in with a
  plain IMAP `LOGIN`, which an OAuth2-only account refuses; a configured
  list skips discovery. The folder names depend on the provider and the
  mailbox language; the `folders` tool shows the real ones after the first
  sync.

To keep a first test small, set `patterns` to one folder. Besides it, the
INBOX bootstrap channel fetches at most the newest 1000 INBOX messages once.

## Permissions

The container runs as uid 1000 and must be able to write `oauth/`:

```sh
mkdir -p mail index oauth
chmod 700 oauth && chmod 600 oauth/*
sudo chown -R 1000:1000 mail index oauth
```

## Run

```sh
cp accounts.example.json accounts.json   # then edit
cp .env.example .env                     # then edit
docker compose up -d
docker compose exec your-mail-mcp sh -c 'cat /tmp/your-mail-mcp*/mbsyncrc'
docker compose logs -f
```

The generated mbsyncrc must show `PassCmd`, `AuthMechs XOAUTH2`, and for each
channel `Sync Pull`, `Create Near`, `Remove None`, `Expunge None`.

## Renewing the token

The refresh token stops working after a period of disuse, or sooner when the
provider enforces a sign-in frequency, for example every 30 days however
often the token is used. The symptom is an `invalid_grant` error from the
helper and an authentication failure in the log on every sync. Renewal needs
an interactive sign-in, so it cannot be automated away.

`renew-token.sh` reduces it to one command:

```sh
./renew-token.sh
```

It runs the device-code flow against the existing `oauth/token.txt`, so no
questions are asked: open the URL, enter the code, confirm. It then checks
the new token, copies it to the server when `renew.env` names one, and prints
the date by which to renew again. `renew.env` (never committed):

```sh
REMOTE_HOST=mailhost
REMOTE_DIR=/srv/your-mail-mcp
REMIND_DAYS=27
```

The server needs no restart: mbsync runs `pass_cmd` on every connection and
reads the new file on the next sync.
