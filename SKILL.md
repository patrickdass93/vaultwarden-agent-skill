---
name: vaultwarden-cli
description: Use when fetching or storing passwords, API keys, TOTP codes, notes or custom fields via the bw CLI against Bitwarden or a self-hosted Vaultwarden server. Covers session unlock, item create/edit, custom fields, verified CLI traps (bw encode, hidden fields), and safe secret handling.
---

# Vaultwarden / Bitwarden via the bw CLI

Working pattern for agents operating a Bitwarden or self-hosted Vaultwarden
vault through the official `bw` CLI. Everything here was verified against a
live Vaultwarden server (bw 2026.11.0).

Credentials (client id/secret, master password) must come from an approved
secret store or env file — never hardcode them in notes, skills, repos, or
chat, and never echo them into output.

## Session flow (each shell is fresh)

```bash
set -a; source /path/to/secret-env; set +a   # provides BW_PASSWORD etc.
export BW_SESSION=$(bw unlock "$BW_PASSWORD" --raw 2>/dev/null)
bw list items --session "$BW_SESSION"        # every command needs --session
bw get password "Item Name" --session "$BW_SESSION"
```

- `bw login --apikey` authenticates but the vault stays **locked** — reading
  items always requires the unlock step.
- Session keys do not persist across shells. Re-unlock in each call.

## Rules

1. **Keep secrets off the command line.** When building item JSON, pass
   values through environment variables (`jq '.login.password=$ENV.PASS'`),
   not `jq --arg` (which lands in the process list) and not inline strings.
2. **Fetch only what you need.** Search first, then read the single item or
   field. Never dump the whole vault into the conversation.
3. **Never print secrets.** When verifying, compare and report only
   length/boolean ("matches: true", "len 40").
4. **Deletes go to trash by default** (`bw delete item <id>`); only
   `--permanent` on explicit user request. `bw restore item <id>` recovers.
5. **Verify writes by reading back** through the same CLI path.

## Creating and editing items

- `bw create item` / `bw edit item` need the item JSON passed through
  `bw encode` FIRST — plain or template JSON gives
  `Error parsing the encoded request data`.
- `bw get template item` has **no `login` key** — add it manually:
  `login: {type:1, username:..., password:..., uris:[...]}`.
- Edit pattern (read-modify-write, secret via env):

```bash
export NEW_PASS="$(bw generate -ulns --length 24 --session "$BW_SESSION")"
ID=$(bw get item "Service" --session "$BW_SESSION" | jq -r .id)
bw get item "$ID" --session "$BW_SESSION" \
  | jq '.login.password = $ENV.NEW_PASS' \
  | bw encode | bw edit item "$ID" --session "$BW_SESSION" >/dev/null
unset NEW_PASS
```

- **Show-once secrets** (API tokens issued at creation): write the secret to
  a 0600 temp file IMMEDIATELY, save to Vaultwarden, then delete the temp
  file. A mid-script failure otherwise loses it permanently.

## Custom fields (do NOT skip these)

- Fields live in the item JSON's `fields` array:
  `{"type": 0, "name": "...", "value": "..."}`. Types: **0=text, 1=hidden,
  2=boolean** (value `true`/`false`). `bw get template item` ships
  `'fields': []` — append entries yourself when creating/editing.
- Reading: there is NO `bw get field` subcommand. Get the whole item and
  pick fields out of the JSON:

```bash
bw get item "Item" --session "$BW_SESSION" \
  | jq -r '.fields[] | select(.name=="My Field") | .value'
```

- **VERIFIED TRAP (bw 2026.11.0):** hidden (type 1) field values may come
  back STILL ENCRYPTED through `bw get item`, `bw list items`, and even
  `bw export` — an opaque base64 blob instead of the secret. Text (type 0)
  fields decrypt normally. Consequence: store API keys/tokens as a **text
  field or the login password** if the CLI must read them later. (Observed
  on one item while another item's hidden field decrypted fine — check the
  value for base64/space patterns when a read-back looks wrong.)
- Verify a field write-back like a password: read the item, compare values,
  print only length/boolean.

## Other getters and traps

- `bw get username|uri|totp|notes "Item"` exist; fields do not.
- **`.env` values containing `$` MUST be single-quoted** in the file, or
  sourcing expands them away. Wrong-password errors from `bw` look like
  "The decryption operation failed", not "invalid password".
- `bw sync` before operations if another client recently changed the vault.

## Search (metadata only)

```bash
bw list items --search "<term>" --session "$BW_SESSION" \
  | jq '[.[] | {id, name, username: .login.username, uri: .login.uris[0].uri}]'
```

Ambiguous name → `bw get` fails with "More than one result was found";
search, then use the item **id**.

## Sharing: Bitwarden Send (never plaintext)

To share a secret with someone, create a Send and share only the link.
Defaults unless told otherwise: 3-day expiry, 3 max accesses, text hidden.

```bash
export SEND_TEXT='the secret'
bw send template send.text --session "$BW_SESSION" \
  | jq '.name="label" | .text.text=$ENV.SEND_TEXT | .text.hidden=true
        | .maxAccessCount=3' \
  | bw encode | bw send create --session "$BW_SESSION" \
  | jq '{accessUrl, deletionDate, maxAccessCount}'
unset SEND_TEXT
```

## Reference

- Official CLI docs: https://bitwarden.com/help/cli/
- Markdown version (fetchable by agents): https://bitwarden.com/help/cli.md
- Item types: 1=Login, 2=Secure Note, 3=Card, 4=Identity, 5=SSH Key.
- URI match: 0=Domain, 1=Host, 2=StartsWith, 3=Exact, 4=Regex, 5=Never.
