# Tour Chain CRM — Public Integration API

> A practical guide for third-party engineers who need to push **companies** and their **agents** into Tour Chain CRM. No prior knowledge of the internal CRM is required.

**Base URL**

```
https://tourchain.icstravelgroup.com/b2badminapi/api/public/crm
```

All endpoints are JSON in / JSON out. Auth & rate-limit headers are described in the [Operational notes](#operational-notes).

**Required headers**

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
Content-Type: application/json
```

If your client or proxy strips the `Authorization` header, send the same token with `X-ICS-API-Token` instead.

---

## Table of contents

1. [Quick start (3 steps)](#quick-start)
2. [Concepts in 60 seconds](#concepts)
3. [Company endpoints](#company-endpoints)
   - [Create company](#create-company)
   - [Update company](#update-company)
   - [Delete company](#delete-company)
4. [Agent endpoints](#agent-endpoints)
   - [Add agents](#add-agents)
   - [Update an agent](#update-agent)
   - [Delete agents](#delete-agents)
5. [Operational notes](#operational-notes)

---

<a id="quick-start"></a>
## 1. Quick start

End-to-end onboarding takes three calls:

```
┌─────────────────────────┐    ┌─────────────────────────┐    ┌─────────────────────────┐
│ 1. Create the company   │ ─▶ │ 2. Add its agents       │ ─▶ │ 3. Use companyCode in   │
│ POST /company           │    │ POST /agents            │    │    your booking calls   │
│ → returns companyCode   │    │ → returns agentsAdded   │    │                         │
└─────────────────────────┘    └─────────────────────────┘    └─────────────────────────┘
```

**Step 1 — Create the company**

```bash
curl -X POST "$BASE_URL/company" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4" \
  -H "Content-Type: application/json" \
  -d '{ "name": "ABC Travel", "companyCode": "ABC001", "email": "info@abc.com" }'
```

**Step 2 — Add agents**

```bash
curl -X POST "$BASE_URL/agents" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4" \
  -H "Content-Type: application/json" \
  -d '{
    "companyCode": "ABC001",
    "agents": [
      { "agentCode": "AGT001", "name": "Nguyen Van A", "email": "a@abc.com" }
    ]
  }'
```

**Step 3 — You're done.** From now on, any subsequent CRM call (update, delete, future booking APIs) uses the `companyCode` you chose in step 1.

---

<a id="concepts"></a>
## 2. Concepts in 60 seconds

- **Company** — a travel agency / tour operator / DMC. Identified by a unique **`companyCode`** that *you* choose.
- **Agent** — a person who works for that company (sales, booker, etc.). Each agent belongs to exactly one company and is identified by **`agentCode`** unique within the company.
- **`agentToken`** — a credential **issued by Tour Chain** for every agent. Returned **once** in the response of `POST /agents` (one per added agent) and on every `PUT /agents` response. Save it on your side and use it to verify the agent on subsequent front-end logins. Tour Chain also stores it in MongoDB.
- **Soft delete vs. hard delete** — by default companies are *soft-deleted* (hidden, recoverable). Pass `?hardDelete=true` to wipe permanently.

That's it. Everything else is detail.

---

<a id="company-endpoints"></a>
## 3. Company endpoints

<a id="create-company"></a>
### 3.1 Create company

```http
POST https://tourchain.icstravelgroup.com/b2badminapi/api/public/crm/company
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
Content-Type: application/json
```

**Required:** `name`. **Recommended:** `companyCode` (otherwise the company can only be looked up by Mongo `_id`).

**Minimal request**

```json
{
  "name": "ABC Travel Company",
  "companyCode": "ABC001",
  "email": "info@abctravel.com"
}
```

**Full request (all optional fields shown)**

```json
{
  "companyCode": "ABC001",
  "name": "ABC Travel Company",
  "agencyTitle": "ABC-TRAVEL",
  "email": "info@abctravel.com",
  "country": "Vietnam",
  "address": "123 Main Street, District 1",
  "city": "Ho Chi Minh",
  "state": "HCM",
  "adtel": "+84-28-1234567",
  "bookerEmail": "booking@abctravel.com",
  "bookerUrl": "https://abctravel.com",
  "agencyGroup": "DMC",
  "affiliate": "Partner Network",
  "network": ["Network1", "Network2"],
  "isPrivate": false,
  "isActive": true,
  "logoAgency": "<base64 or URL>",
  "ref": "REF-001"
}
```

**Response (200)**

```json
{
  "success": true,
  "message": "Agency created successfully",
  "companyId": "507f1f77bcf86cd799439011",
  "companyCode": "ABC001"
}
```

**Common errors**

| HTTP | Body `message` | Meaning |
|------|----------------|---------|
| 400  | `Agency with code 'ABC001' already exists` | Pick a different `companyCode`. |
| 400  | `'name' must not be empty` | `name` is required. |

> See the [appendix](#appendix) for the full optional field reference and length limits.

---

<a id="update-company"></a>
### 3.2 Update company

```http
PUT https://tourchain.icstravelgroup.com/b2badminapi/api/public/crm/company
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
Content-Type: application/json
```

**Partial update** — only the fields you send are changed; everything else is left intact.

```json
{
  "companyCode": "ABC001",
  "name": "ABC Travel Company (Updated)",
  "address": "456 New Street",
  "isActive": true
}
```

- Identify the company with **`companyCode`** *or* **`companyId`** (one is required).
- To rename the code itself, pass `newCompanyCode`. The new value must be unique.

**Response (200)**

```json
{
  "success": true,
  "message": "Agency updated successfully",
  "companyId": "507f1f77bcf86cd799439011",
  "companyCode": "ABC001"
}
```

---

<a id="delete-company"></a>
### 3.3 Delete company

Two equivalent ways — pick whichever fits your code.

**By URL (recommended)**

```http
DELETE https://tourchain.icstravelgroup.com/b2badminapi/api/public/crm/company/{companyCode}
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4

DELETE https://tourchain.icstravelgroup.com/b2badminapi/api/public/crm/company/{companyCode}?hardDelete=true
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
```

- Without `hardDelete` → soft delete (default).
- With `?hardDelete=true` → permanent delete.

**By body**

```http
DELETE https://tourchain.icstravelgroup.com/b2badminapi/api/public/crm/company
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
Content-Type: application/json
```

```json
{ "companyCode": "ABC001", "hardDelete": false }
```

| Mode | Effect |
|------|--------|
| Soft (default) | Company is hidden (`isActive=false`, `isdelete=true`). Recoverable. |
| Hard           | Document removed from MongoDB. **Irreversible.** |

> `DELETE /api/public/crm/deleteProCodes` is **not** for companies — it targets a different collection. Don't use it here.
> If you still need that endpoint, it also requires:
>
> ```http
> Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
> Content-Type: application/json
> ```

---

<a id="agent-endpoints"></a>
## 4. Agent endpoints

All agent endpoints require either **`companyCode`** or **`companyId`** to know which company the agent(s) belong to.

<a id="add-agents"></a>
### 4.1 Add agents

```http
POST https://tourchain.icstravelgroup.com/b2badminapi/api/public/crm/agents
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
Content-Type: application/json
```

```json
{
  "companyCode": "ABC001",
  "agents": [
    {
      "agentCode": "AGT001",
      "name": "Nguyen Van A",
      "title": "Sales Manager",
      "email": "nguyenvana@abctravel.com",
      "tel": "+84-28-1234567",
      "active": true
    },
    {
      "agentCode": "AGT002",
      "name": "Tran Thi B",
      "email": "tranthib@abctravel.com"
    }
  ]
}
```

**Per-agent fields**

| Field        | Required | Notes |
|--------------|----------|-------|
| `agentCode`  | No       | Unique inside the company. If omitted, the agent is always appended (no de-dup). |
| `name`       | **Yes**  | Full name. |
| `title`, `tel`, `email`, `url` | No | Contact info. |
| `active`     | No       | Defaults to `true`. |

> 🔑 **`agentToken` is server-issued.** Do **not** send it in the request — Tour Chain generates a unique, cryptographically secure token for every new agent and returns it in `addedAgents[].agentToken` below. Persist that token on your side: it is shown **only once** in this response.

**Response (200)**

```json
{
  "success": true,
  "message": "Successfully added 2 agent(s) to agency",
  "companyId": "507f1f77bcf86cd799439011",
  "agentsAdded": 2,
  "agentsSkipped": 0,
  "skippedAgentCodes": null,
  "addedAgents": [
    { "agentCode": "AGT001", "agentToken": "k7f2u9...base64url..." },
    { "agentCode": "AGT002", "agentToken": "q1x4z8...base64url..." }
  ]
}
```

> Agents whose `agentCode` already exists in the company are **skipped** (not overwritten) and listed in `skippedAgentCodes`. Use [Update an agent](#update-agent) to modify them.

#### Auto-create company on agent creation (network-resilient flow)

In real-world integrations, the `POST /company` call may occasionally fail or never reach Tour Chain because of a transient network issue on the client side. When that happens, the follow-up `POST /agents` call would normally fail with `404 Company not found`, and **no `agentToken` would ever be returned** — leaving the agent unusable on your side.

To make the onboarding flow resilient, `POST /agents` performs a **lookup-then-create** on the company before adding any agent:

1. The server looks up the company by `companyCode` (or `companyId`).
2. **If the company exists**, agents are added to it as usual.
3. **If the company does NOT exist**, the server automatically creates a new company using the `companyCode` (and any optional `company` payload you send alongside, see below) and then adds the agents to that newly created company.
4. In **both** cases the response contains `addedAgents[].agentToken` for every newly created agent, so the client always receives the token — even when the original `POST /company` call was lost.

**Optional `company` block** — when you want the auto-created company to carry more than just a code, include a `company` object in the same `POST /agents` request. It is **only used when the company has to be created**; it is ignored if the company already exists.

```json
{
  "companyCode": "ABC001",
  "company": {
    "name": "ABC Travel",
    "email": "info@abc.com",
    "tel": "+84-28-1234567",
    "country": "VN"
  },
  "agents": [
    { "agentCode": "AGT001", "name": "Nguyen Van A", "email": "nguyenvana@abctravel.com" }
  ]
}
```

**Response (200) — company was auto-created**

```json
{
  "success": true,
  "message": "Company auto-created. Successfully added 1 agent(s) to agency",
  "companyId": "507f1f77bcf86cd799439011",
  "companyAutoCreated": true,
  "agentsAdded": 1,
  "agentsSkipped": 0,
  "skippedAgentCodes": null,
  "addedAgents": [
    { "agentCode": "AGT001", "agentToken": "k7f2u9...base64url..." }
  ]
}
```

- `companyAutoCreated` — `true` when this call also created the company, `false` (or omitted) when the company already existed.
- The endpoint remains **idempotent on retry**: calling it again with the same `companyCode` will not create a duplicate company; it will just add/skip agents as documented above.
- Recommended client behavior: always call `POST /agents` after `POST /company`. If `POST /company` failed or timed out, `POST /agents` will recover automatically and still return the `agentToken`s.

---

<a id="update-agent"></a>
### 4.2 Update an agent

```http
PUT https://tourchain.icstravelgroup.com/b2badminapi/api/public/crm/agents
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
Content-Type: application/json
```

```json
{
  "companyCode": "ABC001",
  "agentCode": "AGT001",
  "agent": {
    "newCode": "AGT001-NEW",
    "name": "Nguyen Van A (Updated)",
    "title": "Senior Sales Manager",
    "tel": "+84-28-9999999",
    "email": "nguyenvana.new@abctravel.com",
    "active": true
  }
}
```

- `agent.newCode` — optional; renames the `agentCode`.
- `agent.agentToken` — **ignored on input**. The token is server-managed; the response always returns the current `agentToken` so you can re-sync your storage if needed. If a legacy agent has no token yet, one is generated automatically on the first update.
- All other fields are updated only when present and non-empty.

**Response (200)**

```json
{
  "success": true,
  "message": "Agent updated successfully",
  "companyId": "507f1f77bcf86cd799439011",
  "agentCode": "AGT001-NEW",
  "agentToken": "k7f2u9...base64url..."
}
```

#### Auto-create company and/or agent on update (migration-friendly flow)

`PUT /agents` is **upsert-aware** so customers can migrate their existing agents into Tour Chain in a single call, even when the company or the agent has not been onboarded yet. The endpoint mirrors the network-resilient flow of [`POST /agents`](#add-agents) and adds an extra layer for the agent itself:

1. Look up the company by `companyCode` (or `companyId`).
2. **If the company does NOT exist** and `companyCode` is provided, the server **auto-creates** the company using `companyCode` (and any optional `company` block, see below). `companyId`-only updates cannot auto-create — the server has no way to manufacture a Mongo `_id`.
3. Look up the agent by `agentCode` inside that company.
4. **If the agent does NOT exist**, the server **creates the agent** with the fields from the request `agent` block and issues a brand-new `agentToken`. This makes `PUT /agents` safe to use as a single "save" call regardless of prior state.
5. **If the agent exists**, only the non-empty fields you sent are updated; the existing `agentToken` is preserved.

The response always carries the current `agentToken` plus two booleans so you can tell what actually happened:

| Field | Meaning |
|-------|---------|
| `companyAutoCreated` | `true` when this call also created the company. |
| `agentAutoCreated`   | `true` when the agent did not exist and was created by this call. |

**Optional `company` block** — used **only** when the company has to be created. Ignored when the company already exists.

```json
{
  "companyCode": "ABC001",
  "agentCode": "AGT001",
  "company": {
    "name": "ABC Travel",
    "email": "info@abc.com",
    "tel": "+84-28-1234567",
    "country": "VN"
  },
  "agent": {
    "name": "Nguyen Van A",
    "title": "Sales Manager",
    "email": "nguyenvana@abctravel.com",
    "tel": "+84-28-1234567",
    "active": true
  }
}
```

**Response (200) — both company and agent were auto-created**

```json
{
  "success": true,
  "message": "Company and agent auto-created. Agent saved successfully",
  "companyId": "507f1f77bcf86cd799439011",
  "agentCode": "AGT001",
  "agentToken": "k7f2u9...base64url...",
  "companyAutoCreated": true,
  "agentAutoCreated": true
}
```

- The endpoint is **idempotent on retry**: calling it again with the same `companyCode` + `agentCode` will not duplicate either entity. The second call simply updates the existing agent and returns `companyAutoCreated=false`, `agentAutoCreated=false`, plus the same `agentToken`.
- Recommended client behavior for migration: just call `PUT /agents` for every agent you want to sync. If the company is missing, it is created. If the agent is missing, it is created and the `agentToken` is returned for you to persist. If both already exist, the agent is updated in place.
---

<a id="delete-agents"></a>
### 4.3 Delete agents (bulk)

```http
DELETE https://tourchain.icstravelgroup.com/b2badminapi/api/public/crm/agents
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
Content-Type: application/json
```

```json
{
  "companyCode": "ABC001",
  "agentCodes": ["AGT001", "AGT002"]
}
```

```json
{
  "success": true,
  "message": "Successfully deleted 2 agent(s)",
  "companyId": "507f1f77bcf86cd799439011",
  "agentsDeleted": 2,
  "deletedAgentCodes": ["AGT001", "AGT002"],
  "notFoundAgentCodes": null
}
```

---

<a id="operational-notes"></a>
## 5. Operational notes

| HTTP code | Meaning |
|-----------|---------|
| 200 | Success — always check `success: true` in the body too. |
| 400 | Validation or duplicate error — read `message`. |
| 401 | Missing / invalid `Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4`. |
| 404 | Endpoint or company / agent not found. |
| 500 | Server error — please report. |

**Missing token response (401)**

```json
{
  "status": 401,
  "message": "Unauthorized"
}
```

- **Authentication**: send the ICSTransfer token on every request with `Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4`.
- **Auth fallback**: if `Authorization` is not forwarded by your client/proxy, send `X-ICS-API-Token: <same-token>` or `X-API-Key: <same-token>`.
- **ICSTransfer token**: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4`
- **Rate limit (planned)**: 100 requests/min per token. Watch `X-RateLimit-Remaining` / `X-RateLimit-Reset`.
- **Timezone**: all timestamps in responses are UTC ISO-8601.

> ⚠️ Status: Bearer token enforcement and rate limiting are **scheduled** but not yet active in the current build (see appendix). Treat the API as public-but-unannounced for now.

---

Please report any other rough edges to the CRM team.
