# Lead Generation Machine v2

An end-to-end n8n workflow that automates lead sourcing, enrichment, qualification, and outbound calling. A single webhook endpoint (`/lead-machine-v2`) routes requests through five distinct pipelines, and a scheduled trigger powers automated VAPI phone outreach.

---

## Architecture Overview

```
Webhook (/lead-machine-v2)
        │
        ▼
    ┌─ Switch (query.action) ─────────────────────────────────┐
    │                                                         │
    ├─► ScrapeApollo          – Scrape leads from Apollo      │
    ├─► EnrichApolloProfiles  – Enrich Apollo leads           │
    ├─► ScrapeLinkedInProfiles – Google → LinkedIn scraping    │
    ├─► EnrichLinkedInProfiles – Apify LinkedIn enrichment     │
    └─► QualifyLeads          – AI-powered lead scoring       │

Schedule Trigger (separate entry point)
        │
        ▼
    Search Airtable → Format Data → VAPI Outbound Call → CRM Update
```

---

## Pipelines

### 1. ScrapeApollo
Scrapes prospect data from Apollo using the Apify Apollo scraper actor. Results are inserted into the **Apollo Leads** table in Airtable, and the campaign status is updated to "Scraping Complete."

**Flow:** `Webhook → Switch → Get Campaign Record → Apollo Scraper (Apify) → Insert into Airtable → Update Campaign Status`

### 2. EnrichApolloProfiles
*Reserved route for enriching Apollo-sourced profiles with additional data.*

### 3. ScrapeLinkedInProfiles
Builds a Google search query (`site:linkedin.com/in/ <Job Title> <Company Type> <Location>`) using campaign parameters from Airtable, then searches via **SerperDev**. Supports two paths:

- **Paginated scraping** – Splits the desired lead count into batches of 20 and loops through SerperDev pages.
- **Single-batch scraping** – Fetches up to the campaign's `LeadNumber` in one request.

Results are parsed, split into individual organic results, and inserted into the **Google Search Leads** table in Airtable with ICP details attached.

**Flow:** `Webhook → Switch → Get Campaign → Set Campaign Name → Build ICP → Build Google Query → SerperDev → Parse & Split Results → Insert into Airtable → Update Campaign Status`

### 4. EnrichLinkedInProfiles
Takes a lead record from Airtable, scrapes the full LinkedIn profile via the **Apify LinkedIn Profile Scraper**, and maps enriched fields (name, headline, about, experience, email, company details, mobile number) back into the **Google Search Leads** table.

**Flow:** `Webhook → Switch → Get Lead Record → Apify LinkedIn Scraper → Manual Field Mapping → Update Airtable Record`

### 5. QualifyLeads
AI-powered lead qualification pipeline. Extracts the lead's company website content via **Tavily**, then uses an **OpenAI LLM Chain** with a detailed scoring prompt to rate leads 0–10 against the target ICP. The score is written back to Airtable.

**Scoring criteria:**
| Score | Fit Level |
|-------|-----------|
| 10 | Perfect fit – all ICP criteria matched |
| 8–9 | Strong fit – minor deviations |
| 7 | Good fit – worth pursuing |
| 5–6 | Moderate fit – ambiguous |
| 2–4 | Weak fit – significant mismatch |
| 0–1 | Unqualified – fundamental mismatch |

**Flow:** `Webhook → Switch → Get Lead Record → Tavily Website Extract → OpenAI LLM Scoring → Update Qualification Score in Airtable`

### 6. Automated Outbound Calling (Schedule Trigger)
A separate scheduled entry point that searches Airtable for leads with `Call = 'To Do'`, formats their data, and initiates an outbound phone call via **VAPI**. A companion webhook handles the VAPI tool-call response by sending appointment booking emails through **Gmail** with a Calendly link.

**Flow:** `Schedule Trigger → Search Airtable (Call='To Do') → Format Data → Check Conditions → VAPI Call → Update CRM Record`

**Booking flow:** `VAPI Call Response Webhook → Gmail (Calendly link) → Respond to Webhook`

---

## Integrations

| Service | Purpose |
|---------|---------|
| **Airtable** | CRM / lead database (campaigns, leads, scores) |
| **SerperDev** | Google Search API for LinkedIn profile discovery |
| **Apify** | LinkedIn profile scraping & Apollo.io scraping |
| **Tavily** | Website content extraction for lead qualification |
| **OpenAI** | LLM-based ICP scoring (via LangChain node) |
| **VAPI** | AI-powered outbound phone calls |
| **Gmail** | Automated appointment booking emails |

---

## Airtable Schema

### Lead Generation Machine v2 Base
| Table | Purpose |
|-------|---------|
| **Google Search Campaigns** | Campaign definitions (job title, company type, location, size, industry) |
| **Google Search Leads** | Leads discovered via Google/LinkedIn scraping |
| **Apollo Leads** | Leads sourced from Apollo.io |
| **Aggregated Emails** | Consolidated lead data for qualification |

### Protect Fortune Leads Base
| Table | Purpose |
|-------|---------|
| **Sheet1** | Outbound call queue (filtered by `Call = 'To Do'`) |

---

## Required Credentials

| Credential | Type |
|------------|------|
| Airtable Personal Access Token | API Token |
| SerperDev (Header Auth) | API Key via `X-API-KEY` header |
| Apify | API Token (passed as query param) |
| OpenAI | API Key (for LangChain LLM node) |
| VAPI | Bearer token for outbound calls |
| Gmail OAuth2 | OAuth2 for sending emails |

---

## Setup

1. **Import the workflow** – In your n8n instance, import `workflow.json`.
2. **Configure credentials** – Set up each credential listed above in n8n's credential manager.
3. **Update Airtable base/table IDs** – Replace the base and table IDs with your own Airtable references.
4. **Replace API keys** – Search for `INSERT_API_KEY_HERE` in the workflow and replace with your actual Apify tokens.
5. **Update VAPI config** – Replace the `assistantId`, `phoneNumberId`, and Bearer token with your VAPI account details.
6. **Activate the workflow** – Toggle the workflow to active to enable the webhook and schedule trigger.

---

## Usage

Trigger any pipeline via a GET/POST request to the webhook:

```bash
# Scrape LinkedIn profiles for a campaign
curl "https://<your-n8n-instance>/webhook/lead-machine-v2?action=ScrapeLinkedInProfiles&recordId=<airtable-record-id>"

# Enrich a LinkedIn profile
curl "https://<your-n8n-instance>/webhook/lead-machine-v2?action=EnrichLinkedInProfiles&recordId=<airtable-record-id>"

# Qualify a lead with AI scoring
curl "https://<your-n8n-instance>/webhook/lead-machine-v2?action=QualifyLeads&recordId=<airtable-record-id>"

# Scrape Apollo leads
curl "https://<your-n8n-instance>/webhook/lead-machine-v2?action=ScrapeApollo&recordId=<airtable-record-id>"
```

The automated calling pipeline runs on a schedule and requires no manual triggering.
