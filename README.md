# n8n email automation — Upwork → profile match → Telegram

Self-hosted [n8n](https://n8n.io/) workflow that reads **Gmail** (Upwork notifications about new postings), compares each message against a **developer profile** fetched with the **Google Docs** node, scores postings with **Anthropic Claude Haiku**, and sends a **Telegram notification for each match** that clears the score threshold.

Everything runs inside the workflow: ingestion, evaluation logic, and notifications.

## What it does

1. **Gmail** — Trigger on new messages (e.g. Upwork job alerts).
2. **Google Docs** — Load your profile text from a **Google Doc** (hosted on Drive). Start from [`profile-template.md`](profile-template.md), copy the content into a Doc, and point the **Get a document** node at it.
3. **Evaluate** — **Anthropic Claude Haiku** scores each posting against the profile and returns **structured JSON**. Downstream **Code** nodes parse the response and **drop items below the score threshold**; there is no batching or digest aggregation in this workflow.
4. **Telegram** — When a posting passes the filter, send a message with score, rationale, excerpt, and link. *(A periodic digest of multiple matches is not implemented yet—you can add it as a later enhancement.)*

Credentials for Google, Anthropic, and Telegram are configured in the n8n UI (OAuth for Google; API keys where required), not in this repo.

### Scoring model (LLM choice)

**Claude Haiku** is the default in the workflow: it is enough for “score this posting vs profile and return JSON,” and it stays cheap at volume. If you see repeated misclassification on messy descriptions, try **Claude Sonnet** instead (higher cost and latency).

## Requirements

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose
- Google Cloud project with **Gmail API**, **Google Drive API**, and **Google Docs API** enabled (the workflow uses the **Google Docs** node to read profile text), OAuth consent configured, and your user added as a test user (or app published) as needed
- [Anthropic](https://www.anthropic.com/) API access for the Haiku scoring step
- Telegram bot (via [@BotFather](https://t.me/BotFather)) and chat ID for where to send notifications
- **n8n** with a working **Code** node and task runners (this repo’s `docker-compose.yml` uses current official images and `N8N_RUNNERS_ENABLED=true`; use a recent n8n version if you deploy elsewhere)

## Run n8n locally

1. Copy the environment template and adjust if needed:

   ```bash
   cp .env.example .env
   ```

   Optionally set **`N8N_ENCRYPTION_KEY`** (e.g. `openssl rand -hex 32`) so credential encryption remains stable if you recreate volumes. Set **`N8N_HOST`** / **`N8N_PROTOCOL`** if your instance is not reached at `http://localhost:5678`.

2. Start the stack:

   ```bash
   docker compose up -d
   ```

3. Open **http://localhost:5678**. On first launch, **create the owner account** in the UI (email and password). That account protects the editor and stored credentials—there is no separate HTTP basic-auth layer in this compose file.

4. Import [`workflows/email-automation.json`](workflows/email-automation.json). The export is sanitized for git: it does **not** contain instance credential IDs. After import, **attach credentials** on each node (**Gmail**, **Google Docs**, **Anthropic** on the HTTP Request via Header Auth with `x-api-key`, **Telegram**), then fill in values that were removed from the JSON:
   - **Google Doc URL** — In **Get a document**, set `documentURL` to your real Doc link (the placeholder `YOUR_GOOGLE_DOC_ID` must be replaced with your document’s ID).
   - **Telegram chat ID** — In **Send a text message**, set `chatId` to your numeric chat or channel ID (replace `YOUR_TELEGRAM_CHAT_ID`).

### Google OAuth redirect

For self-hosted n8n, add the OAuth redirect URI that n8n shows in the credential dialog to your Google Cloud **OAuth 2.0 Client** (often `http://localhost:5678/rest/oauth2-credential/callback` when using `N8N_HOST=localhost` and `N8N_PROTOCOL=http`). If you use a domain or HTTPS, set `N8N_HOST` and `N8N_PROTOCOL` in `.env` to match the URL you use in the browser.

### Docker Compose notes

- **Image** — Uses the official registry `docker.n8n.io/n8nio/n8n` as in [n8n Docker docs](https://docs.n8n.io/hosting/installation/docker/).
- **Persistence** — Named volume `n8n_data` maps to `/home/node/.n8n` (workflows, credentials, execution history).
- **Time** — `TZ` and `GENERIC_TIMEZONE` align the container clock with schedule-oriented nodes (e.g. if you add a **Schedule** trigger later).
- **Runners** — `N8N_RUNNERS_ENABLED=true` enables task runners for Code and related nodes as recommended upstream.

## Cost (ballpark)

Rough order of magnitude for a personal instance:

- **Hosting:** about **$5/month** on something like [Railway](https://railway.app/) for a small always-on container—or **$0** on a free-tier VPS (e.g. Oracle Cloud Always Free) if you operate your own VM.
- **LLM:** **pennies per month** in typical use: Haiku is priced for high volume, and you only call it once per new posting (plus any retries), not on a fixed timer.

Exact numbers depend on email volume and provider pricing; treat this as a sanity check, not a quote.

## Project layout

| Path | Purpose |
| --- | --- |
| `docker-compose.yml` | n8n service and persistent volume |
| `.env.example` | Template for optional encryption key and URL-related variables |
| `.env` | Your local values (create from `.env.example`, do not commit) |
| `profile-template.md` | Starter profile to copy into a Google Doc for Haiku scoring |
| `workflows/email-automation.json` | Workflow export — import, attach credentials, set doc URL and chat ID |

## Security

- Never commit `.env` or API keys.
- Prefer HTTPS and a proper hostname when exposing n8n beyond your machine.
- Restrict Telegram bot tokens, Anthropic keys, and Google OAuth client secrets to n8n’s credential store.

## License

MIT. See [`LICENSE`](LICENSE).

n8n itself is licensed separately; see [n8n’s license](https://github.com/n8n-io/n8n/blob/master/LICENSE.md).
