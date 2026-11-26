# Setup guide

From an empty server to a working admission system. Allow 2 to 3 hours the first time.

## Contents

1. [Prerequisites](#1-prerequisites)
2. [Deploy n8n](#2-deploy-n8n)
3. [Prepare Google resources](#3-prepare-google-resources)
4. [Create credentials in n8n](#4-create-credentials-in-n8n)
5. [Import the workflows](#5-import-the-workflows)
6. [Replace the placeholders](#6-replace-the-placeholders)
7. [Build the Tally form and connect it](#7-build-the-tally-form-and-connect-it)
8. [Connect the client's software](#8-connect-the-clients-software)
9. [Activate and test](#9-activate-and-test)
10. [Optional: verify Tally's signature](#10-optional-verify-tallys-signature)
11. [Local development](#11-local-development)
12. [Backups and updates](#12-backups-and-updates)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. Prerequisites

| Need | Details |
|---|---|
| Server | Linux VPS, 2 GB RAM or more, Docker and Docker Compose installed |
| Domain | A subdomain such as `n8n.clientdomain.com`, with an **A record** pointing to the server IP |
| Ports | 80 and 443 open (needed for HTTPS certificates) |
| Google account | For Sheets, Drive and Gmail (a dedicated sender account is best) |
| Tally account | Plan that supports webhooks |
| Client software details | How it accepts new students (API, database, import) |

---

## 2. Deploy n8n

```bash
git clone <this-repo> admission-automation
cd admission-automation

cp .env.example .env
```

Edit `.env`:

```bash
# generate each secret separately
openssl rand -hex 32     # use for N8N_ENCRYPTION_KEY
openssl rand -hex 32     # use for LINK_SECRET
```

Set `DOMAIN`, `POSTGRES_PASSWORD`, `N8N_ENCRYPTION_KEY`, `LINK_SECRET`, and the timezone.

```bash
docker compose up -d
docker compose ps          # all three services should be "running"
docker compose logs -f n8n # watch for startup errors (Ctrl+C to exit)
```

Open `https://<your-domain>` and create the **owner account**.

> **Back up `N8N_ENCRYPTION_KEY` now.** If it is lost or changed, every saved credential becomes unreadable.

> Pin the n8n image to a tested version tag in `docker-compose.yml` instead of `latest` before go-live.

---

## 3. Prepare Google resources

### 3.1 Google Sheet

1. In Google Drive choose **New, then File upload** and upload `templates/google-sheet-template.xlsx`.
2. Open it and choose **File, then Save as Google Sheets**. Name it **Admissions**.
3. In the `Applications` tab check that row 1 headers are unchanged. **Delete the yellow example row.**
4. In the `WhatsApp` tab replace the example row with real rows: one per `admission_track` + `batch_number`.
5. Copy the **Sheet ID** from the URL:
   `https://docs.google.com/spreadsheets/d/`**`THIS_PART`**`/edit`

### 3.2 Drive folder

1. Create a folder named **Admission Screenshots**.
2. Share it **only** with HR and the technical owner. Do **not** make it public.
3. Copy the **Folder ID** from the URL:
   `https://drive.google.com/drive/folders/`**`THIS_PART`**

---

## 4. Create credentials in n8n

In n8n go to **Credentials, then Create credential**.

| Credential | Used by | Notes |
|---|---|---|
| Google Sheets OAuth2 API | Every Sheets node | Needs a Google Cloud OAuth client. Follow n8n's on-screen guide. Authorised redirect URI is shown in the credential screen. |
| Google Drive OAuth2 API | `Upload screenshot` | Same Google Cloud project |
| Gmail OAuth2 API | All email nodes | Sign in as the dedicated sender account |
| Header Auth | `Push to software` | Name/value as required by the client API (for example `Authorization: Bearer <key>`) |

In the Google Cloud console enable the **Google Sheets API**, **Google Drive API** and **Gmail API**. If the OAuth consent screen is in "Testing" mode, add the sender account as a test user and publish the app for production use, otherwise tokens expire after 7 days.

---

## 5. Import the workflows

For each file in `workflows/`, in this order: `3-admission-rejected.json`, `2-admission-approved.json`, `1-admission-intake.json`, `error-handler.json`:

1. n8n, then **Workflows, then Add workflow**, then the **...** menu, then **Import from file**.
2. Select the JSON file.
3. Open the sticky note named **Setup note** and follow its checklist.

The exports are sanitized: they contain **no credentials, no secrets and placeholder IDs only**.

> After importing, click through each node once. If a node shows a warning (for example a changed parameter name in your n8n version), re-select its **Operation** and re-assign its credential. Node parameters differ slightly between n8n versions.

### Link between workflows

Workflow 1 builds Accept and Reject links. They point to:

| Link | Path |
|---|---|
| Accept | `https://<domain>/webhook/admission-approve` (Workflow 2) |
| Reject | `https://<domain>/form/admission-reject` (Workflow 3) |

If you change a Webhook or Form path, update `Build links` in Workflow 1 to match.

---

## 6. Replace the placeholders

| Placeholder | Where | Replace with |
|---|---|---|
| `YOUR_GOOGLE_SHEET_ID` | Every Google Sheets node (all 3 workflows) | Sheet ID from 3.1 |
| `YOUR_DRIVE_FOLDER_ID` | W1 `Upload screenshot` | Folder ID from 3.2 |
| `https://n8n.clientdomain.com` | W1 `Build links` | Your real public URL, **no trailing slash** |
| `hr@clientdomain.com` | W1 `Email HR` | HR approver address (comma-separated for several) |
| `tech@clientdomain.com` | W2 `Alert tech`, Error Handler | Technical alert address |
| `https://CLIENT-SOFTWARE.example.com/api/students` | W2 `Push to software` | Client API endpoint |
| `https://portal.clientdomain.com` | `templates/emails/3-approval-welcome.html`, then the `Email student (approved)` node | Student portal URL |
| `+92-XXX-XXXXXXX`, `support@clientdomain.com`, `Your Institute` | Email templates | Real details |

To change email wording, edit the HTML in the matching Gmail node (the files in `templates/emails/` are the source copies; keep them in sync).

### Wire up LINK_SECRET

The Crypto nodes read `{{ $env.LINK_SECRET }}`. The value comes from `.env` via `docker-compose.yml`, so it is identical in all three workflows. If your n8n blocks `$env` in expressions, set `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` (already set in the compose file) or paste the secret into the three Crypto nodes (then treat exported JSON as secret).

### Set the error workflow

In each of Workflows 1, 2 and 3: **Settings, then Error Workflow, then Error Handler**.

---

## 7. Build the Tally form and connect it

1. Build the form from [`templates/form-questions.md`](../templates/form-questions.md). **Labels must match exactly.**
2. In Workflow 1 open the `Tally Webhook` node and copy the **Production URL**
   (`https://<domain>/webhook/tally-admission`).
3. In Tally: **Integrations, then Webhooks, then Connect**, paste the URL, save.
4. To see the real payload first: paste the **Test URL**, click **Listen for test event** in n8n, submit the form, and inspect `body.data.fields`. Then switch to the Production URL.

---

## 8. Connect the client's software

The `Push to software` node is a template. Choose the pattern that matches the client system.

| Client software accepts | What to do |
|---|---|
| **REST API** | Keep the HTTP Request node. Set URL, auth, and map the JSON body keys to **their** field names. If account creation is a separate endpoint, add a second HTTP Request after it. |
| **Direct database** | Replace the node with the MySQL / Postgres / MSSQL node and an INSERT. Use a restricted DB user. |
| **CSV import only** | Replace with a node that appends to a "ready to import" sheet or emails a CSV to the admin. Student credentials must then be created manually. |

Keep these settings on the node: **Retry On Fail** (3 tries, 5000 ms) and **On Error: Continue (using error output)**. The error output is what triggers the failure alert.

Test with a **throwaway student** in the client's test environment before pointing at production.

---

## 9. Activate and test

1. Toggle **Active** on Workflows 3, 2, 1 and the Error Handler.
2. Submit the Tally form with test data using your own email.
3. Work through the **Test Log** tab of the sheet (10 tests). All must pass.
4. Remove test rows from the sheet, Drive and the client software.

---

## 10. Optional: verify Tally's signature

Stops anyone who discovers the webhook URL from posting fake applications.

1. In Tally's webhook settings set a **Signing secret**.
2. In the `Tally Webhook` node: **Options, then Raw Body: on**.
3. Add a **Crypto** node after it: Action **Hmac**, Type **SHA256**, Encoding **base64**, Secret = the signing secret, Value = the raw body (`{{ $json.body }}` in raw mode), property `computed`.
4. Add an **IF** node: `{{ $json.headers['tally-signature'] }}` equals `{{ $json.computed }}`. Continue only on **true**.

The exact field holding the raw body depends on your n8n version. Inspect the webhook output to confirm.

---

## 11. Local development

Tally cannot reach `localhost`. Use a tunnel and tell n8n the public URL.

```bash
cloudflared tunnel --url http://localhost:5678
# note the printed https://<random>.trycloudflare.com URL

WEBHOOK_URL=https://<random>.trycloudflare.com/ LINK_SECRET=dev-secret n8n start
```

Quick tunnel URLs change every restart. When they do, update `WEBHOOK_URL`, the base URL in `Build links`, and the Tally webhook.

---

## 12. Backups and updates

### Backups (daily)

```bash
# database
docker compose exec -T postgres pg_dump -U $POSTGRES_USER $POSTGRES_DB | gzip > backup-$(date +%F).sql.gz

# n8n data folder
docker run --rm -v admission-automation_n8n_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/n8n-data-$(date +%F).tar.gz -C /data .
```

Copy backups off the server. **Test a restore** before go-live. Store `N8N_ENCRYPTION_KEY` and `LINK_SECRET` in a password manager.

(The volume name depends on your folder name. Run `docker volume ls` to confirm.)

### Updates

1. Read the n8n release notes.
2. Back up first.
3. Change the image tag in `docker-compose.yml`, then `docker compose pull && docker compose up -d`.
4. Re-run the Test Log.

### Rotating LINK_SECRET

Do this only when no applications are pending: all Accept/Reject links already sent stop working when it changes.

---

## 13. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Tally shows delivery failed / 404 | Workflow inactive, or the Test URL was used. Use the **Production** URL and activate. |
| Webhook URLs display `localhost` | `WEBHOOK_URL` not set or n8n not restarted |
| HTTPS certificate not issued | DNS not pointing at the server yet, or ports 80/443 blocked. Check `docker compose logs caddy`. |
| Fields arrive empty | Tally label differs from the one in `Prepare Data`. Labels must match exactly, including capitalisation and spacing; check the webhook output. |
| Dropdown shows an ID | The field has no `options` array in the payload. Inspect `body.data.fields`. |
| `payment_screenshot` empty | File question label is not exactly `Payment Screenshot` |
| `Referenced node is unexecuted` | A node name was changed. Node names must match the expressions. |
| Accept link shows 404 | Workflow 2 inactive, or the base URL in `Build links` is wrong |
| Accept page hangs | A branch of Workflow 2 ends without a **Respond to Webhook** node |
| Reject page hangs | A branch of Workflow 3 ends without a Form **Completion** node |
| "Invalid link" for a genuine click | `LINK_SECRET` differs between workflows, or was changed after the email was sent |
| Leading zeros lost in the sheet | Columns not formatted as Plain text. Use the provided template. |
| Duplicate rows after Tally retries | Keep the webhook responding **immediately** and the duplicate check enabled |
| Image missing in HR email | Tally file URL expired. Use the Drive link or the attached copy. |
| Gmail "insufficient permissions" | OAuth scopes missing, or the consent screen is still in Testing mode |
| All credentials broken after a migration | `N8N_ENCRYPTION_KEY` differs from the original |
| Link triggers without a human click | An email security scanner pre-fetches links. Move the action behind a confirmation page. |
