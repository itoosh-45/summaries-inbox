# Rules for Claude in this repository

**The single source of truth is [`docs/SPEC.md`](docs/SPEC.md). Read it fully before any work**, then follow it. Approved mockups: kept outside the repo, ask the owner.

## In short
- Talk to the owner in plain, simple Hebrew. The app UI is Hebrew, RTL. The spec and code comments are in English.
- Build one milestone at a time (SPEC §10). End each with your own tests + a numbered Hebrew checklist for the owner, then stop and wait for "עובד" / "לא עובד".
- After each milestone: tick SPEC §10, add a line to SPEC §14, update SPEC §12 if a decision changed.
- Before touching anything live (active workflow, server, database, DNS): say in one line what changes and how to undo it. Ask first before anything destructive.
- Everything stays free. Gemini free tier is tiny: tests use pinned data only, no extra real runs.
- n8n 2.x: saving is not publishing. Publish after editing an active workflow.

## Privacy (the repo may be public)
- No personal data in any file or commit message: no emails, names, IPs, server names, real domains, n8n IDs, vault names/paths, personal subscriptions. Use the placeholders listed in SPEC §0.3.
- Real values come from the cloud environment variables (`INBOX_API_URL`, `INBOX_APP_ORIGIN`), from n8n itself (look workflows and credentials up by name), or from the owner.
- Before every push: confirm the repo is private, and scan the diff for personal values.

## Secrets
- Never write secrets (passwords, tokens, API keys, `N8N_ENCRYPTION_KEY`, SSH keys) into this repo, Obsidian or chat. Write only *where* a secret lives (n8n Credentials, the server `.env`, the owner's password manager).
- Never ask the owner to paste a secret into the chat.

## Cloud session
- The cloud environment `summaries-inbox` has HTTPS access only: **no SSH to the server**. Give the owner every server command to run in PowerShell (`ssh n8n ...`). Keep remote one-liners free of `"` (Windows PowerShell 5.1 mangles them) or send scripts via stdin: `$script | ssh n8n "tr -d '\r' | bash -s"`.
- A local Postgres 16 runs in the container (same major version as the server). Test every `sql/` file and API query there first.
- n8n is reached through the n8n MCP tools.

## Layout
- `app/` PWA (plain HTML/CSS/vanilla JS, no build step) · `sql/` schema + seeds · `n8n/` exported workflow JSON (backup) · `docs/` spec.
