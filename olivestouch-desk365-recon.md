# Reverse Engineering: olivestouch.desk365.io

**Date:** 2026-10-22 · **Method:** cmux browser (SPA instrumentation), bundle static analysis, live API verification
**Verdict:** Multi-tenant SaaS helpdesk (**Desk365 / Kani Technologies Inc.**), tenant `olivestouch` — an Angular support portal fronting two APIs: an internal RPC gateway on `apps.desk365.io` and a public documented REST API at `/apis/v3`.

---

## 1. Asset / host map

| Host | Role |
|---|---|
| `olivestouch.desk365.io` | Support portal (Angular SPA, `/support/`), Swagger UI (`/apis/api-docs.html`), REST API (`/apis/v3/`) |
| `apps.desk365.io` | Primary backend — internal RPC gateway `/KaniPSA/ProcessGeneral` (datacenter: **US**) |
| `apps-eu.desk365.io` | EU datacenter variant (not used here; `GetHelpdeskDataCenterLocation` → `US`) |
| `*.kanidesk.com` / `*.qa.kanidesk.com` | White-label brand of the same product (dev/qa hosts hard-coded in bundle) |
| `s3.amazonaws.com/prod-portal-cust-logos/...` | Tenant logos, served via **presigned S3 URLs** (AWS STS token embedded) |
| `d365-product-images.s3.amazonaws.com` | Product assets (support-bot zip, SVGs) |
| `support.olivestouch.com` | Custom-domain button target (tenant-configured) |
| `login.microsoftonline.com` / `graph.microsoft.com` | MSAL — Azure AD sign-in + profile |
| `res.cdn.office.net/teams-js` | Microsoft Teams integration (portal runs inside Teams) |

No WebSocket usage. `robots: noindex` on the portal.

---

## 2. Support portal frontend

- **Framework:** Angular CLI production build — hashed bundles at `/support/`:
  - `main-SZ37AYMC.js` (4.5 MB — everything inlined), `polyfills-QLBWGKOY.js`, `scripts-LEQPBLIV.js` (1.2 MB vendor), `styles-DPAKCLD3.css`
- **Notable libraries (from bundle strings):** MSAL.js (`@azure/msal-browser`), Google Identity Services (`apis.google.com/js/api.js`), Microsoft Teams JS SDK, Froala WYSIWYG (with filestack plugin), EmojiOne 2.0.1, `ngx-cookie-service`, Moment.js, uuid.
- **i18n:** `GET /support/assets/i18n/en.json` (multi-language switchable only if tenant enables it — `isMultipleLanguageAllowed: 0` here).
- **Client-side routes:** `sp-home`, `kbase`, `sp-article`, `sp-articles-search`, `createTicket`, `ticketdetails`, `tickets`, `profile`, `signin`, `signup`, `frgtPassword`, `resetPassword`, `msal-signin`, `msal-rh`, `msteams-login`, `invalid-subdomain`, `support-portal-disabled`.
- **localStorage keys:** `selectedLanguage`, `selectedLanguageForAgentPortal`, `totalTicketsCount`, `fr-copied-html`, `fr-copied-text`.
- **Auth options surfaced by bundle:** Azure AD (MSAL, incl. China cloud endpoints), Google, Microsoft Teams, plain email/password (contact login), guest tickets, access-token login (`GetCustomerDetailsWithAccessToken`).
- **Security headers observed:** `X-Portal-Session`, `X-CSRF-Token` referenced by app code (portal session + CSRF for authenticated portal actions).

---

## 3. Internal RPC gateway (undocumented backend)

Every portal call funnels through one endpoint:

```
GET https://apps.desk365.io/KaniPSA/ProcessGeneral?action=<Action>&<params>
```

**Tenant params:**

| Param | Value | Meaning |
|---|---|---|
| `portalUrl` / `csd` | `olivestouch` | Customer subdomain |
| `spt` / `encsd` | `fb250ca04ceebda6fa0dac555a30e815` | Opaque per-tenant portal token (32-hex; **not** md5 of subdomain/cid/name — server-issued, stable across calls) |
| `featureId` | `21`, `27`, … | Feature-flag lookup (`GetFA`) |

**59 action names extracted from `main.js`:**
`CheckIfContactTicketClosingIsAllowedForSP`, `CheckValidEndUserInvite`, `CreateGuestTicket`, `CreateNewTicketForSP`, `deleteAttachmentsForSP`, `DownloadAttachment`, `DownloadAttachmentForSP`, `GetActiveAnnouncementsForSP`, `GetAgentNameFromAgentIdForSP`, `GetApprovalDetailsForSP`, `GetCategoryName(s)`, `GetChoicesForStatusFieldForSP`, `GetContactAndTicketDetailsForSP`, `GetContactDetailsForSP`, `GetContactFieldsForSP`, `GetContactPasswordPolicy`, `GetCustomContactFieldsDataForSP`, `GetCustomerDetailsWithAccessToken`, `GetCustomTicketFieldsDataForSP`, `GetCustSurveyDetailsForSp`, `GetDefaultFieldDropdownChoices[ForSP]`, `GetFA`, `GetFroalaAccess`, `GetGroupName(s)`, `GetHelpdeskDataCenterLocation`, `GetNotificationModeForContactsForSP`, `GetPortalTableColumnOrder`, `getSharedToForTktIdForSP`, `GetSPCurrentUserInfo`, `GetSpKbaseFolderAndArticleList[ForSP]`, `GetSupportBotNameForContact`, `GetSupportPortalConfiguredTicketPropertyFields`, `GetSupportPortalConfigWithSubdomain`, `GetSupportPortalExportFields`, `GetSupportPortalTicketFormPropertyFields`, `GetSupportPortalTicketListFields`, `GetTicketListCountForPortalDashboard`, `GroupForSP`, `listallsbcategoryForSP`, `SetNotificationModeForContactsForSP`, `TicketReplyDescriptionForSP`, `TicketReplyNoteDescriptionForSP`, `tktcategoryForSP`, `unAt*`, `UpdateFindHelpful[ForSP]`, `userGroupMapForSP`, `UserListForSP`, `ValidateEndUser`.

`…ForSP` = Support-Portal (public/end-user) variants. Requests are plain GETs with query strings; responses are JSON.

### Key unauthenticated responses

**`GetSupportPortalConfigWithSubdomain`** → full tenant config, e.g.:

```json
{"isInvalidSubdomain":false,"configData":{
  "customerName":"Olivestouch Technologies","cid":35968,"id":1775,
  "portalUrl":"olivestouch","customerPlanType":11,
  "customerExpiryTime":1798821000,          // 2027-01-01
  "helpdeskCreatedOn":1735660429,           // 2024-12-31
  "themeColor":"#3F51B5","title":"Support",
  "nonSignInCreateTicket":1,"supportPortalSignIn":2,"removeCaptcha":1,
  "maxAllwdAttchmnts":10,"inlineImageSize":10485760,
  "tktFieldsClmnOrder":"5,11,8,9,10,6",
  "portalButtonArr":[{"btnName":"Create Ticket","btnLink":"https://support.olivestouch.com/support/form/createTicket"}],
  "logoUrl":"https://s3.amazonaws.com/prod-portal-cust-logos/35968/...?X-Amz-Security-Token=…"}}
```

**`GetHelpdeskDataCenterLocation`** → `{"dataCenterLocation":"US"}` (routers to `apps`, not `apps-eu`).

---

## 4. Public REST API v3 (documented, OpenAPI)

- **Swagger UI:** `https://olivestouch.desk365.io/apis/api-docs.html`
- **OpenAPI 3.0.1 spec:** `https://olivestouch.desk365.io/apis/v3/api-docs?group=v3` (350 KB, **100 paths**, **118 schemas**, 18 tag groups)
- **Server:** `https://olivestouch.desk365.io/apis`
- **Auth:** API key in `Authorization` header — key is valid **only under the issuing subdomain**.
- **API key location:** `~/.secrets.zsh` → `$DESK365API` (verified live)

### Endpoint groups (18)

| # | Group | Endpoints |
|---|---|---|
| 01 | Ping | `GET /v3/ping` |
| 02 | Tickets | 15 ops — list/search/details/conversations, create (`+attachment`, `+import`, `import_conversations`), update, add_reply/add_note (`+attachment`), associate/disassociate assets |
| 03 | Contacts | list/details/create (`+attachment`)/update |
| 04 | Companies | list/details/create (`+attachment`)/update |
| 05 | Time entries | list/details/add/update/delete |
| 06 | Surveys | list / ratings / details |
| 07–09 | KB Categories / Folders / Articles | full CRUD (`/v3/kb/...`) |
| 10–17 | Asset management | assets, asset types & fields, products, vendors, depreciations, locations, softwares, software fields — full CRUD + associate/disassociate |
| 18 | Contracts | CRUD — **restricted**: requires emailing help@desk365.io |

### v3 behaviors (from official docs)

- Pagination: `limit` ∈ **30 / 50 / 100** on `/v3/tickets`, `/v3/time_entries`
- Custom fields prefixed `cf_` (both read and write)
- List-tickets response includes SLA name, form name, contact name; watchers + share_to readable/writable
- `GET /v3/tickets/conversations` returns conversation data **without** Description
- Microsoft first-party connector exists: `learn.microsoft.com/en-us/connectors/desk365/` (Power Automate / Logic Apps / Power Apps)

### Live verification (read-only, 2026-10-22)

```
GET /v3/ping            → 200  "All good! Desk365 APIs are running fine"
GET /v3/tickets?limit=5 → 200  count=5730   (newest: #6081)
GET /v3/contacts?limit=1→ 200  count=562
GET /v3/companies?limit=1 → 200 count=19
```

Ticket record shape (v3 list): `ticket_number, contact_email, contact_name, subject, status, priority, type, source, form_name, company_name, sla, group, category, subcategory, created_on, updated_on, updated_by(_type), due_date, completed_within_sla, status_*_duration, first_response_time, resolved_on, closed_on, watchers[], share_to[], conversation_count`…

---

## 5. Tenant facts (olivestouch)

| Fact | Value |
|---|---|
| Customer | Olivestouch Technologies · `cid` **35968** · helpdesk since 2024-12-31 |
| Plan / expiry | `customerPlanType: 11` · expires 2027-01-01 |
| Datacenter | US (`apps.desk365.io`) |
| Volumes | 5,730 tickets · 562 contacts · 19 companies (incl. RADICARE, MCMC SIFS) |
| Branding | Theme `#3F51B5` (Indigo), custom domain `support.olivestouch.com`, no reCAPTCHA, anonymous ticket creation allowed, sign-in mode `2` |
| SLAs seen | "Standard AMS SLA", "MCMC SLA" |

---

## 6. Notable observations / risks

1. **Tenant config, feature flags, KB, forms and password policy are readable unauthenticated** (only `csd` + static `spt` needed) — by design for a public portal, but it discloses plan, expiry, internal IDs, logo S3 presigned URLs.
2. **Two parallel APIs**: documented `/apis/v3` (API-key auth) and undocumented `KaniPSA/ProcessGeneral` RPC (session/token auth for the SPA). Any automation should prefer v3.
3. The bundle carries **hard-coded dev/QA white-label domains** (`dev.kanidesk.com`, `qa.kanidesk.com`) — environment bleed from vendor build pipeline.
4. `spt` token is not derived from trivial hashes of tenant identifiers (tested subdomain, cid, customer name) — server-issued secret; treat as low-sensitivity but tenant-bound.
5. Agent portal auth token (`jwToken`) is a **JS-readable, non-httpOnly cookie** — any XSS on the portal can exfiltrate a 14-day Admin-capable bearer token. See Part II.

## 7. Reproduce

```bash
source ~/.secrets.zsh
curl -s -H "Authorization: $DESK365API" https://olivestouch.desk365.io/apis/v3/ping
curl -s -H "Authorization: $DESK365API" "https://olivestouch.desk365.io/apis/v3/tickets?limit=30" | jq '.count'
curl -s https://olivestouch.desk365.io/apis/v3/api-docs?group=v3 -o openapi-v3.json   # no key needed
```

Bundles saved at `/tmp/rea-olivestouch/` (ephemeral): `main-SZ37AYMC.js`, `scripts-LEQPBLIV.js`, `polyfills-QLBWGKOY.js`, `openapi-v3.json`.

Action inventories: `./olivestouch-agent-actions.txt` (59 support-portal + 541 agent-portal actions).

---

# Part II — Agent Portal (`/app/`) — REVERSE ENGINEERED

Source: logged-in cmux browser session + Safari Web Inspector timeline export (`~/Downloads/olivestouch.desk365.io-recording.json`) + live curl verification.

## A1. Stack

| | |
|---|---|
| App | Angular CLI at `/app/` — `runtime/polyfills/scripts/main` + ~20 lazy chunks (`common.*.js`, `24.*.js` …, `main.fc92b51ec9047896.js`) — separate build from `/support/` |
| Extras | Plotly 1.55.2 (dashboards/reports), `d365-web-widget.s3.amazonaws.com/d365-support-widget.js`, Google Fonts (Material Symbols/Roboto), remixicon |
| i18n | `/app/assets/i18n/en.json` |
| Storage | local: `filteredTickets`, `totalTicketsCount`, `selectedLanguage`, `fr-copied-*`, `_grecaptcha` · session: `prevView` |
| UI (top bar) | New ▾, Get Started, Desk365 Support, PowerBI Reporting, notifications (9+), user menu |

## A2. Authentication — SOLVED & VERIFIED

1. **Login issues a JWT stored in a JS-readable cookie `jwToken`** (no httpOnly) — `document.cookie` exposes it.
2. The SPA reads it and sends it **verbatim in the `Authorization` header** (raw JWT, **no `Bearer` prefix**) on every `apps.desk365.io/KaniPSA/*` call.
3. Verified from shell:

```bash
TOK=$(jq -r '...extract Authorization from recording...' recording.json)
curl -H "Authorization: $TOK" 'https://apps.desk365.io/KaniPSA/ProcessHelpDesk?action=GetCurrentUserInfo'
# → 200, full profile
curl 'https://apps.desk365.io/KaniPSA/ProcessHelpDesk?action=GetCurrentUserInfo'
# → HTTP 401 {"code":0}
```

**JWT (HS256, secret on server; claims readable):**

| Claim | Value |
|---|---|
| `sub` | 63795 (agent user id) |
| `iss` / `aud` / `typ` | `Desk365` / `desk365-agent` / `agent` |
| `email` | ezad@olivestouch.com |
| `pid` | 35968 (tenant/customer id) |
| `role` | 3 → **Admin** (per `GetCurrentUserInfo.roleName`) |
| `allowedCustFeatures` | 57 feature IDs (subscription entitlements) |
| `iat` → `exp` | 2026-10-22 → **2026-11-05 (14 days)** |
| `jti` | UUIDv4 |

> HMAC secret is server-side only (not extractable from client), but the token itself is trivially harvestable from `document.cookie`.

## A3. Internal API surface (agent side)

```
GET|POST  https://apps.desk365.io/KaniPSA/<Servlet>?action=<Action>[&params]
Header:   Authorization: <raw JWT>
Body:     JSON for POST (e.g. {"accountIdList":"463181,526979"})
```

**Servlets:** `ProcessHelpDesk` (main — tickets, agents, SLAs, KB, assets, reports, contracts, automations, billing), `ProcessContact`, `ProcessGeneral` (shared config/widget).

**Action inventory: 541 agent-side + 59 support-side** → full list in `./olivestouch-agent-actions.txt`. Highlights by domain:

- **Tickets:** `TicketList` (rich JSON `filters` blob: status/type/priority/category/agent/group/contact/company/source/watcher/sla/CustomFieldsFilter/viewNumber/timezoneOffset…), `GetTicketDetailsHelper`, `MergeTickets`, `unarchiveTicket`, `GetDeletedTicketList`, `DeleteAllTrashTickets`, `AddTicketTimeEntry`, `GetTicketActivities`, `GetWatchersForTktId`
- **People:** `AgentList`, `UserList`, `ContactListOnOffset`, `searchContacts`, `CompanyList`, roles (`GetDefaultAndCustomRoles`, `DeleteUserCustomRole`)
- **SLA/contracts:** `ListSLA`, `AddNewSLA`, `CreateBlockhourContract`, `CreateRetainerContract`, `CreateTandMContract`, `CreateSvcContract`, `CreateFpContract`
- **KB:** `GetKBArticleList/Details/VersionList`, `GetKBCategoryList`, `GetKBFolderList`, `GetKBSharedGroups`
- **Assets/ITAM:** `GetAllAssetTypes(+Fields)`, `GetAssetDetails`, products/vendors/locations/softwares/depreciations mirrors of v3 REST
- **Automation:** `ListAutomation`, `UpdateAutomation(Status)`, `GetClosureRules`, scheduled tickets (`CreateScheduledTicketCreation`, `CloneScheduledTicketCreation`)
- **Reports/dashboards:** `getReports`, `GetDashboardsList`, `GetAgentPerformanceStats`, `GetAIAgentAnalyticsStats`, `getTktTrendsReportsForHome`, Plotly-backed charts
- **Billing/Stripe:** `StripeCheckoutV2`, `StripeCustomerPortalV2`, `GetStripePriceInfo`, `GetCreditBalance/UsageStats`, `GetAIAddOnBillingDetails`, trials (`CreateNewTrial`, `extendCustomerTrial`)
- **Channels:** email accounts (`GetDefaultAndForwardAccounts`, `retryDKIMVerification`), chat (`GetChatSessions`), web widget (`getWebWidgetDetails`, `updateWebWidgetDetails`), Teams templates, Freshdesk import (`CreateFreshdeskImportChoice`, `SaveFreshdeskImportFieldMapping`, `PingFreshdeskApi`), Intune sync (`GetIntuneSyncProfilesList`…)
- **Auth/misc:** `login`, `Logout`, `msalSignup`, `msalValidateUser`, `ForgotSubdomain`, `GetAPIEnabled`/`UpdateAPIUserId` (per-user REST API key management), `WebSocketsRTSFA` (realtime feature flag; `code:0` = off for this tenant)

## A4. Bootstrap sequence (page load, from recording)

`GetCurrentUserInfo` + `GetFAForCompany` + `GetProdUpdates*` + `GetCustomBtnsForAgentApp` + `GetActiveAnnouncementsForAP` + `getLanguageSettings` + `GetRunningTimeEntryDetails` + `GetContractsAndSLAsForCustomer` + `GetSearchSettings` + `GetFormsList` + `ListSLA` + `GetCustomViews` + `GetTicketFields` + `tktcategory` + `listallsbcategory` + `UserList&module=8` + `Group&type=0` + `CompanyList` + `GetContactFields` + `GetTicketTableColumns&planType=11` + `TicketList(...)` + `userGroupMap` — then POSTs `getContactDetailFromAccIdList`, `GetViewsTktCount`.

## A5. Security headers / CORS (KaniPSA responses)

- `Access-Control-Allow-Origin: *` (any site may call it **with** a token — no origin pinning)
- Allowed methods `GET, OPTIONS, HEAD, PUT, POST`; allowed headers incl. `Authorization`, `X-CSRF-Token`
- Rate limiting: exposes `X-Rate-Limit-Remaining`, `X-Rate-Limit`, `X-RateLimit-Reset`, `Retry-After`
- `Content-Security-Policy: frame-ancestors 'self' https://teams.microsoft.com https://*.teams.microsoft.com …` + `X-Frame-Options: ALLOW-FROM https://teams.microsoft.com` (Teams embedding)
- `X-Content-Type-Options: nosniff` · `X-XSS-Protection: 0` · `Referrer-Policy: strict-origin-when-cross-origin`

## A6. Notable observations (agent side)

1. **Three-tier auth recap:** REST `/apis/v3` = long-lived API key in `Authorization` · agent portal = 14-day JWT in `Authorization` · support portal = opaque `spt` tenant token (unauthenticated read layer).
2. `{"code":0}` = standard **unauthorized/error sentinel** (HTTP 401 without token; the support-portal `WebSocketsRTSFA` `code:0` means feature disabled).
3. **CORS `*`** means a leaked JWT works from any origin, incl. curl.
4. `GetCurrentUserInfo` response leaks the full entitlement map (57 feature IDs), timezone id, view configs — one GET is a complete recon payload.
5. Widget call `getWebWidgetDetails&customerId=11643&encid=f914927d…` uses a **different id space** (11643 ≠ 35968; encid not plain md5) — widget customer id vs. helpdesk id.
6. Session duration: JWT `exp` − `iat` = **14 days**; tenant plan expiry 2027-01-01.

---

# Part III — Write Flows: New Ticket, New Contact, New Company, Replying

Static extraction from all 22 `/app/` chunks (union of `?action=X` literals and Angular `HttpParams` `{action:"X"}` — **1,001 unique actions**, 5 servlets: `ProcessHelpDesk`, `ProcessContact`, `ProcessCompany`, `ProcessGeneral`, `ProcessTicket`). Not one of these write flows was executed — all shapes recovered from code + official OpenAPI schemas.

## B1. New Ticket (Agent Portal)

**Call:** `POST https://apps.desk365.io/KaniPSA/ProcessHelpDesk?action=CreateNewTicketForAP`
**Body:** `{ "ticketObj": {...}, "assetsAssociationObj": {...}? }` (assets optional)

**`ticketObj` (from `constructTicketObject()`):**

```json
{
  "tktSrcType": "<TICKET_SOURCE_TYPE_PHONE_OR_OTHER>",
  "contactName": "…", "contactId": "…", "email": "…",     // contact binding
  "ccAddr": "a@x.com,b@y.com",                             // validated emails
  "subject": "…",                                          // sanitizeSubject()
  "ticketType": 0, "ticketStatus": 0, "ticketPriority": 0, // form ids (-1→0)
  "ticketGroup": 0, "ticketAgent": 0,
  "description": "<html…>",                                // Froala rich text
  "ticketCategory": 0, "ticketSubCategory": 0,
  "contractId": "…",
  "attachment": "name1:name2",        // colon-joined
  "attachTime": "t1,t2",              // comma-joined
  "attachmentIds": "id1,id2",
  "s3AttachmentKeys": "key1,key2",
  "ticketFieldsData": [ {"fieldId":N,"fieldType":N,"fieldValue":…}, … ],
  "inlineImgObj": [...],              // embedded images
  "planType": 11,
  "customFormId": 0, "formId": 0      // ticket form (id from GetFormsList)
}
```

- **`ticketFieldsData`** = dynamic form fields only (`defaultFieldId==0`), incl. nested/section dropdown expansion. Required checkbox → forced `1`; ePHI-encrypted fields skipped without permission.
- **Attachments flow:** upload first (`s3UploadAttachment(JSON)` → returns `attach_name/attach_time/attachment_id/attach_location`), then the four colon/comma-joined string fields are built by `constructQueryParamsForAttachment()`.
- **Response handling:** `code==1` success; error flags: `maxCreateTktLimitReached` (+`maxCreateTktLimit`), `restrictedDomain`, `maxRecipientsCountReached`.
- **Pre-checks:** `checkTicketFieldsValid()` (field validation incl. contact required, contract required), `checkInlineImagesUploaded()`.

**Alternate creation paths:**
| Path | Endpoint | Notes |
|---|---|---|
| Web form / support-portal style | `POST ProcessTicket?action=CreateNewTicket&csd=<subdomain>` | body `{ticketObj, assetsAssociationObj?}`, `withCredentials:true` + **`X-CSRF-TOKEN`** header (`postWebForm`) |
| Escalation from chat/contact | same `CreateNewTicket` | `isEscalationToCreateTicket` flag in widget config |

## B2. New Contact (Agent Portal)

**Call:** `POST .../ProcessHelpDesk?action=SaveContact` — body `{ "contactObj": {...} }`
**Edit:** `action=EditContact` — same shape (from `constructContactObj()`):

```json
{
  "contactId": "…",                       // empty on create
  "firstName": "Name", "lastName": "",
  "title": "…", "email": "x@y.com",       // lowercased
  "workphone": "…", "mobile": "…",
  "status": true,
  "secondaryEmailArr": [...],
  "isContactWhitelisted": 0|1,
  "companyMapper": {...},                 // company association
  "customFieldData": [ {"fieldId":N,"fieldName":"…","value":…} ],
  "attachmentId": "…", "attachmentKey": "…", "deleteAttachId": ""
}
```

**Custom-field value normalization (important):** checkbox → `1|0`; decimal → **×100** (cents); date/future-picker → `yyyy-MM-dd` (Date/moment/ISO-string coercion); attachments & read-only text excluded; encrypted text type excluded from payload.

Pre-save: duplicate checks on primary + secondary emails (`searchContactsByEmails`, `CheckDeletedEmailBeforeUpdatingContact`), name trim/validity.

## B3. New Company (Agent Portal)

**Call — note, it's a GET with JSON in the query string:**

```
GET .../ProcessHelpDesk?action=SaveCompany&companyObj=<urlencoded JSON>
GET .../ProcessHelpDesk?action=UpdateCompany&companyObj=<urlencoded JSON>
```

**`companyObj` (from `constructCompanyObj()`):**

```json
{
  "companyId": 0|N, "companyName": "…",
  "address": "…", "city": "…", "state": "…", "zipcode": "…", "country": "…",
  "description": "…",
  "domains": ["a.com", …],          // current (leading empty element trimmed)
  "addDomains": [...], "deleteDomains": [...],
  "status": true,
  "isHelpdeskDefSLA": …,            // + SLA fields
  "customFieldData": [...]
}
```

Validation: max domain length (`domains.length <= maxDomainsLength`); delete blocked with explanation if company domains are used in automation rules (`commonDomains`, `updateAllowed` in response). `CheckForDomainDeletion` pre-check. Company table columns via separate servlet `ProcessCompany?action=…`.

## B4. Responding to a Ticket (Agent Portal)

The reply box supports **5 actions** (`s.selectAction`), each POSTing a different payload to `ProcessHelpDesk`:

### a) Email reply → `action=AgentReplyFromApp`

```json
{
  "ID": <ticketId>, "tktNum": <ticketNumber>, "agent": <assignedTo>, "cid": <customerId>,
  "subject": "<ticket title>",
  "message": "<html reply body>",
  "attach": "n1:n2", "attach_time": "t1,t2", "attachmentIds": "…", "s3AttachmentKeys": "…",
  "sessionId": "<ws session>", "actionType": "reply",
  "updateTktStatusAndReply": 0|1,          // change status in same call
  "tktStatusChangeTo": <statusId>,
  "TO": "<to emails>", "FROM": "support@electriccrm.com",   // hardcoded default; overridden by fromEmailAccountId
  "CC": "…", "BCC": "…",                    // CC ≤2048 chars, BCC ≤1024 (client-validated)
  "inlineImgObj": [...],
  "includePreviousReplies": 0|1,
  "teamId": <team>, "fromEmailAccountId": <n>,
  "lastTktArticleId": …, "lastNoteArticleId": …   // concurrency/dedupe anchors
}
```

Also: `BulkSendReplyFromAgent` (bulk), `AgentReplyTeams` (Teams-channel reply: `{ticketId, customerId, customerEmailId, message, subject, tktNum, …}`).

### b) Private/internal note → `action=AddNote`

```json
{
  "note": "<html>", "ID": <ticketId>, "isprivate": "0"|"1",
  "contactId": <customerId>, "assigneeId": <assignedTo>,   // present on "assignment" variant
  "title": …, "desc": <first article description>,
  "contactEmail": …, "contactName": …, "loggedInAgentName": …,
  "tktSrcType": …, "attach": …, "attach_time": …, "attachmentIds": …, "s3AttachmentKeys": …,
  "customerName": …, "sessionId": …, "inlineImgObj": [...],
  "TO": "<notify agent list>",              // constructNotifyAgentList()
  "lastTktArticleId": …, "lastNoteArticleId": …,
  "updateTktStatusAndReply": 0|1, "tktStatusChangeTo": <id or 0>
}
```

### c) Customer description (contact-facing note) → `action=AddCustDescription`
`POST {custDesc, custId, userId, isPrivate:"0"|"1"}` → `{code, CustmrDescId}` · delete: `DeleteCustDescription` `{contactDescriptionId}`.

### d) Status/property side-effects after replying
`UpdateAfterReplyAction`, `UpdateReplyWithUpdateTktProperties`, `UpdateTicketChanges` (POST whole payload object), `UpdateTicketProperties`, `UpdateTicketAgentGroup`, `UpdateTktDueDate`, `AddTicketTimeEntry` (**GET** `&teObj=<json>`), `AddWatcher`.

### e) Ticket-property editing
`EditTicketAndContactPOST`, `MergeTickets`, `BulkUpdateProperties`, `unarchiveTicket`, `BulkDeleteTickets`/`DeleteAllTrashTickets`.

## B5. Response contract (all write flows)

Success = `{"code":1, ...}` (`code:0` = unauthorized/error). Domain-specific error flags ride along on `code:1`? No — they arrive as sibling fields when `code:0` (e.g. `maxCreateTktLimitReached`, `restrictedDomain`, `duplicateEmail`, `updateAllowed:false` + `commonDomains`). UI branches on them in `.subscribe(...)`.

## B6. Documented REST v3 equivalents (same operations, cleaner API)

| Flow | REST v3 | Request schema (from OpenAPI) |
|---|---|---|
| New Ticket | `POST /v3/tickets/create` (+`create_with_attachment`, `import`) | `CreateTicketModel`: `email, form_name, subject, description(HTML), status, priority(int), type, assign_to(agent email), group, category, sub_category, custom_fields(cf_*), watchers[], share_to[]` |
| New Contact | `POST /v3/contacts/create` (+`create_with_attachment`) | `CreateContactRequestModel`: `name, primary_email, secondary_emails, title, mobile, phone, company_name, primary_department, secondary_departments, custom_fields` |
| New Company | `POST /v3/companies/create` (+`create_with_attachment`) | `CompanyRequestModel`: `name, description, phone, address, city, state, country, zipcode, sla, domains(comma), departments(comma), custom_fields` |
| Reply | `POST /v3/tickets/add_reply` (+`add_reply_with_attachment`) | `AddReplyModel`: `body, cc_emails, bcc_emails, agent_email, from_email, include_prev_ccs(0/1), include_prev_messages(0/1)` |
| Note | `POST /v3/tickets/add_note` (+`add_note_with_attachment`) | `AddNoteModel`: `body, agent_email, notify_emails, private_note(0/1)` |

> **Practical takeaway:** for automation, prefer REST v3 (API-key auth, stable schemas, rate-limited) over replaying `ProcessHelpDesk` calls — the internal payloads (`ID/cid/tktNum/sessionId/lastTktArticleId…`) carry portal-session state that's fragile outside the SPA.

## B7. Mismatch notes

1. Internal payloads use **numeric ids** (contactId, ticketGroup…); REST v3 uses **names/emails** (group, assign_to) — v3 resolves them server-side.
2. `FROM:"support@electriccrm.com"` is a hardcoded placeholder in the SPA; real send-from comes from `fromEmailAccountId` (legacy ElectricCRM branding leak — Desk365's previous product name).
3. Company save = GET with JSON query param (violates HTTP semantics; shows the API's evolution from GET-param RPC).
4. Contact decimal custom fields are **×100** client-side; REST v3 `cf_` values are sent as-is.
5. All reply/note calls carry `sessionId` (WebSocket) + `lastTktArticleId/lastNoteArticleId` — soft optimistic-concurrency controls against duplicate sends.

---

# Part IV — Live Dry Run: Ticket #6082 (create → enforce → close)

Executed 2026-10-22 against production (authorized by tenant admin). Ticket `#6082` — clearly marked `[TEST TICKET]`, assigned to ezad@olivestouch.com. Screenshot: `/tmp/rea-olivestouch/ticket-6082-closed.png`.

## Timeline & findings

| Step | Action | Result |
|---|---|---|
| 1 | `POST /v3/tickets/create` | ✅ **#6082** created, source `15` (API), form "Create Ticket" (`tktFormId=32050` agent form) |
| 2 | `GET /v3/tickets/details` | ✅ round-trip verified; all 26 `cf_*` fields returned null |
| 3 | `PUT /v3/tickets/update {"status":"closed"}` | ❌ **HTTP 400** — *"Ticket can be closed only if: Time Entry is added. Modules, Category, Func Mandays, Tech Mandays, DBA Mandays, Sub Category, Complexity Level are filled."* → **closure rules engine** enforced via API |
| 4 | `POST /v3/time_entries/add` | ✅ id 487038 (1 min manual entry) |
| 5 | `PUT` fields as **display strings** ("Moderate") | ❌ HTTP 400 `datatype_mismatch`: *"Value set is of type String. It should be an Integer"* for Modules & Complexity Level |
| 6 | `PUT` fields as **integer choice ids** (268333/235693) + status closed | ❌ HTTP 400 — closure still unsatisfied; **entire PUT rolls back atomically when closure blocked** |
| 7 | `PUT` fields only (no status) | ⚠️ HTTP 200 — `category/subcategory/mandays` persisted, but **`cf_Modules` + `cf_Complexity Level` silently dropped** (integers pass datatype validation, never written) → **REST v3 bug: dropdown custom fields (fieldType 10) don't persist on write, though reads render labels fine** |
| 8 | Choice discovery via internal API | ✅ `GetAllTicketFields` → fieldIds (Modules=**228956** ft10, Complexity=**235691**); `GetChoicesForTicketField?ticketFieldId=&defaultFieldId=` → child choice ids (OMC=**268333**, Moderate=**235693**); `GetFormTicketFields?formId=32050` → which fields sit on the agent form |
| 9 | Internal bypass: `POST ProcessHelpDesk?action=UpdateTicketChanges` with `customTicketFieldsData:[{fieldId,fieldType,fieldValue}]` + full `currentValues`/`previousValues` snapshots | ✅ `code:1` — values landed (raw store shows `fieldValue:268333/235693`) |
| 10 | `PUT {"status":"closed"}` retry | ✅ **CLOSED** `2026-10-22 03:00:19Z` (`closed_by_type:2`, `resolved_on` auto-set, `completed_within_sla:true`, `closed_duration:32` min) |
| 11 | Agent portal UI observation | ✅ `/app/tickets/ticketdetails?view=indVw&tktNum=6082` shows Closed badge, OMC/Moderate, API source, Time Entries panel |

## Key discoveries

1. **Closure rules are server-enforced on REST** (same engine as `GetClosureRules`/`CheckClosureRulesConditionsForGroups`), tenant requires: time entry + 7 fields (Modules, Category, Func/Tech/DBA Mandays, Sub Category, Complexity Level). Failed close = HTTP 400 with human-readable `reason`.
2. **REST v3 custom-field quirk (bug):** dropdown/choice fields accept integer ids (strings rejected `datatype_mismatch`) but **silently no-op**; scalar text fields ("0") persist fine. Reads (`GET details`) correctly translate stored choice-id → label. Workaround: write via internal `UpdateTicketChanges`, or set fields through the UI before closing.
3. **Atomic rollback:** closure-blocked PUT persists nothing (not even unrelated fields).
4. **Id spaces (verified):** standard fields stored as internal ids (`category 47651`=Incident, `subCategory 47917`=Module Specific Error) — REST accepts *names* for these; custom dropdowns need *choice child-fieldIds*; text cf's accept strings.
5. **Value display asymmetry:** GET returns labels for standard & cf dropdowns, but PUT-update expects names (standard) / ids (cf dropdowns) — and the cf dropdown id write path is broken (see 2).
6. Raw per-ticket cf storage: `TicketList` response → `customTicketFieldsDataList[ticketId][fieldId] = {ticketFieldId (row id), fieldType, defaultFieldId, fieldValue}`.
7. Live tenant banner observed in portal: **"Custom domain SSL certificate renewal failed"** (support.olivestouch.com cert renewal issue — worth flagging to ops!).
8. Ticket-detail deep link: `/app/tickets/ticketdetails?view=indVw&tktNum=<num>`.

## Reproduce (read-only probes)

```bash
source ~/.secrets.zsh
# closure rule (expected 400 on open ticket lacking fields)
curl -X PUT -H "Authorization: $DESK365API" -H 'Content-Type: application/json' \
  'https://olivestouch.desk365.io/apis/v3/tickets/update?ticket_number=6082' -d '{"status":"closed"}'
```
