# vaultwarden-agent-skill

An [Agent Skill](https://agentskills.io) for operating a Bitwarden or
self-hosted **Vaultwarden** vault from AI coding agents (Claude Code, Hermes,
OpenClaw, Codex, etc.) via the official `bw` CLI.

## What it covers

- Session unlock flow for fresh shells (`bw unlock` + `BW_SESSION`)
- Item create/edit through `bw encode` (and why plain JSON fails)
- **Custom fields** end to end — including a verified trap where hidden
  (type 1) field values return still-encrypted on bw 2026.11.0
- Secret hygiene rules: values off the command line, read-back verification,
  show-once secret handling, trash-by-default deletes
- Sharing via expiring Bitwarden Sends instead of plaintext

## Install

```bash
# Claude Code
mkdir -p .claude/skills && cp SKILL.md .claude/skills/vaultwarden-cli.md

# Any agent that loads SKILL.md files
cp SKILL.md ~/.agents/skills/vaultwarden-cli/SKILL.md
```

## Why not just the official docs?

The official CLI reference (https://bitwarden.com/help/cli.md) lists
commands; this skill encodes the things that actually bite agents in
practice, each verified against a live Vaultwarden server:

| Trap | Detail |
|---|---|
| `bw encode` required | `bw create item` fails with "Error parsing the encoded request data" without it |
| No `login` in template | `bw get template item` omits the `login` key |
| Hidden fields may stay encrypted | type 1 field values returned as base64 blobs by `get`/`list`/`export` |
| `$` in `.env` values | must be single-quoted or sourcing expands them |
| Wrong-password errors | look like "The decryption operation failed" |
| API-key login | authenticates but the vault stays locked |

## Security

The skill contains no credentials, server URLs, or personal data. Keep your
vault credentials in a secret store (env file at mode 600, or a password
manager) — never in the skill, chat, or a repo.

## License

MIT
