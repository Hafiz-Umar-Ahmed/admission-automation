# Admission form questions (Tally)

Build the form in Tally with **exactly** these question labels. The `Prepare Data` node in
Workflow 1 finds each answer by its **label**, so a typo or rename means that field arrives empty.

| # | Label (exact) | Tally type | Required | Maps to | Notes |
|---|---|---|---|---|---|
| 1 | `Full Name` | Short answer | Yes | `name` | |
| 2 | `CNIC / B-Form` | Short answer | Yes | `cnic_bform` | 13 digits. Add Tally validation (min/max 13 characters) so most mistakes never reach n8n. Dashes are stripped automatically. |
| 3 | `City` | Short answer or dropdown | Yes | `city` | |
| 4 | `Email` | Email | Yes | `email` | Needed to send confirmation, credentials and rejection emails. |
| 5 | `Intermediate Board` | Dropdown | Yes | `intermediate_board` | Example options: Federal Board, KPK Board, Aga Khan, Sindh Board. Dropdown answers arrive as option IDs and are converted to text automatically. |
| 6 | `Admission Track` | Dropdown | Yes | `admission_track` | Must match the `admission_track` values in the **WhatsApp** sheet tab. |
| 7 | `Target University` | Dropdown | Yes | `target_university` | |
| 8 | `Batch Number` | Short answer or dropdown | Yes | `batch_number` | Must match the `batch_number` values in the **WhatsApp** sheet tab. |
| 9 | `Payment Screenshot` | File upload | Yes | `payment_screenshot` | Single file, images only (`.jpg`, `.jpeg`, `.png`). Stored as a URL. |

## Fields added by the workflow (not on the form)

| Field | Value |
|---|---|
| `application_id` | `APP-yyyyMMdd-####`, generated in `Prepare Data` |
| `submitted_at` | ISO timestamp |
| `status` | `pending` |
| `email_send` | `no`, then `yes` after the HR email is sent |
| `confirmation_send` | `no`, then `yes` after the student confirmation is sent |

## Tally settings

1. **Integrations, then Webhooks, then Connect**: paste the **Production** URL of the `Tally Webhook` node
   (`https://<your-domain>/webhook/tally-admission`).
2. Optional but recommended: set a **signing secret** (see `docs/setup-guide.md`, section "Verify Tally's signature").
3. Add a thank-you message such as: *"Thank you! Our team will verify your payment and email you within 24 to 48 hours."*

## Changing dropdown options later

Dropdown choices live in **Tally**, not in n8n. When the client adds a new track or batch:

1. Add the option in Tally.
2. Add a matching row in the **WhatsApp** tab of the sheet (same spelling).

## Reject form (n8n, not Tally)

HR sees this when clicking **Reject** in the email. It is an n8n form, served from the same n8n instance.

| Field | Type | Notes |
|---|---|---|
| `id` | Hidden | Filled from the link (`?id=...`) |
| `token` | Hidden | Filled from the link (`&token=...`) |
| `Reason` | Textarea, required | Emailed to the applicant |

> The original project brief also mentioned change requests and appointments. This admission system has
> no change-request form. If a client later needs one (for example "update my batch"), add a second Tally
> form and a fourth workflow that finds the row by `application_id` and updates it.
