# Client onboarding and handover checklist

Use this with the client from kickoff to go-live. Tick items as they are done.

## 1. Discovery (before building)

- [ ] Admission tracks, universities, batches and boards listed
- [ ] Who in HR approves applications, and the email address(es) used
- [ ] **How does the existing software accept students?** (API / direct database / CSV import / none)
- [ ] API documentation, test environment and API key obtained (if an API)
- [ ] Does the software support forced password change on first login?
- [ ] Preferred username format (email, CNIC, generated ID)
- [ ] Expected volume (applications per day or batch)
- [ ] Wording and branding approved for the four emails
- [ ] Privacy and retention expectations for CNIC data and payment screenshots

## 2. Accounts the client must own

- [ ] Domain name (e.g. `n8n.clientdomain.com`) with DNS access
- [ ] Server / VPS account (2 GB RAM or more)
- [ ] Google account for Sheets, Drive and Gmail (dedicated sender recommended)
- [ ] Tally account (webhooks enabled on the plan)
- [ ] WhatsApp groups created per track and batch, with invite links

## 3. Build and configuration

- [ ] Google Sheet created from `templates/google-sheet-template.xlsx`
- [ ] Example rows deleted from `Applications` and `WhatsApp`
- [ ] WhatsApp invite links entered for every active track + batch
- [ ] Private Drive folder created and shared only with HR
- [ ] Tally form built from `templates/form-questions.md` (labels exact)
- [ ] Server deployed with `docker-compose.yml`; HTTPS working
- [ ] `.env` filled with strong, unique secrets (not the examples)
- [ ] Workflows imported from `workflows/`; credentials created in n8n
- [ ] Sheet ID, Drive folder ID, base URL, HR and tech emails replaced in the nodes
- [ ] `Push to software` mapped to the client's real API fields
- [ ] Email templates customised (institute name, portal URL, support phone)
- [ ] Error Handler set as the Error Workflow on all three workflows
- [ ] Tally webhook pointed at the **Production** URL; all workflows **active**

## 4. Acceptance testing (use the `Test Log` tab)

- [ ] All 10 tests passed with the real HR inbox
- [ ] Test data removed from the sheet, Drive and the client software
- [ ] One real end-to-end application watched by the client

## 5. Handover

- [ ] n8n owner account created for the client
- [ ] Client shown: the sheet, the Drive folder, the executions list, the error email
- [ ] Client shown how to add a new batch (Tally option + WhatsApp row)
- [ ] Workflow JSON exports saved in the repository (secrets removed)
- [ ] Backups configured and a restore tested (Postgres + `n8n_data` + `N8N_ENCRYPTION_KEY`)
- [ ] `LINK_SECRET` and `N8N_ENCRYPTION_KEY` stored in the client's password manager
- [ ] Support contact and change-request process agreed
- [ ] n8n license terms reviewed (Sustainable Use License) for how the client will use it

## 6. After go-live

- [ ] Day 1: watch the first applications and the error inbox
- [ ] Week 1: review pending applications older than 48 hours
- [ ] Monthly: check backups, update n8n in a staging copy first, review access to the sheet and Drive folder

## Information to collect from the client

| Item | Value |
|---|---|
| Institute name | |
| HR approver email(s) | |
| Tech alert email | |
| Portal URL | |
| Support phone / email | |
| Domain for n8n | |
| Client software contact person | |
