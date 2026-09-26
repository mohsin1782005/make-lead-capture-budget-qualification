# Lead Capture & Budget-Based Qualification Automation (Make.com)

Two plug-and-play Make.com automations that capture inbound leads through a webhook, log them to Google Sheets, qualify them by budget and reply instantly with personalized HTML emails. Every user-editable setting lives in a single configuration module, so no module mappings ever need to be touched.

![Lead Capture & Budget-Based Qualification Automation (Make)](<Lead Capture & Budget-Based Qualification Automation (Make).jpg>)

## Overview

Small and medium businesses lose leads when follow-up is slow or generic. A lead who fills in a form expects a reply within minutes, and a high-budget prospect deserves a different response than someone just exploring options. Doing this by hand means copying form entries into a spreadsheet and writing each email yourself.

This suite automates that first touchpoint:

- **Scenario 1** logs every new lead to Google Sheets and sends a branded welcome email within seconds.
- **Scenario 2** reads the lead's stated budget, sorts it into a tier and sends an email written for that tier. Leads with a missing or invalid budget go to a fallback route instead of crashing the scenario.

Both scenarios are built for non-technical users. After importing the blueprint, a user connects Google once, pastes their settings into the **⚙️ CONFIG** module and is ready to go.

---

## Architecture & Workflow

### Scenario 1: Webhook → Google Sheets + Instant Welcome Email

```text
        [ Webhook: name, email, phone, source ]
                         │
                         ▼
             ┌───────────────────────┐
             │  FILTER: valid email  │   email contains "@"
             └───────────┬───────────┘
                         ▼
             ┌───────────────────────┐
             │  ⚙️ CONFIG (edit only) │   Sheet ID · Sheet name
             │                       │   Business name · Subject
             └───────────┬───────────┘
                         ▼
             ┌───────────────────────┐
             │  📊 Save Lead to Sheet │ ──► error → Resume
             │  Timestamp · Name ·   │     (email still sends)
             │  Email · Phone ·      │
             │  Source               │
             └───────────┬───────────┘
                         ▼
             ┌───────────────────────┐
             │ ✉️ Send Welcome Email  │ ──► error → Skip
             │  Branded HTML         │
             └───────────────────────┘
```

### Scenario 2: Webhook → Lead Qualification Router (Budget Tiers)

```text
        [ Webhook: name, email, phone, budget, service ]
                              │
                              ▼
                  ┌───────────────────────┐
                  │  FILTER: valid email  │
                  └───────────┬───────────┘
                              ▼
                  ┌───────────────────────┐
                  │  ⚙️ CONFIG (edit only) │   Business name · Booking link
                  │                       │   HIGH_BUDGET_MIN · LOW_BUDGET_MAX
                  └───────────┬───────────┘
                              ▼
                  ┌───────────────────────┐
                  │ 🔢 Clean Budget Value  │   "$7,500" → 7500
                  │                       │ ──► error → Resume (budget = 0)
                  └───────────┬───────────┘
                              ▼
                  ┌───────────────────────┐
                  │ 🔀 Qualify by Budget   │
                  └───────────┬───────────┘
        ┌──────────────┬──────┴───────┬──────────────┐
        ▼              ▼              ▼              ▼
   💎 High          📈 Mid         🌱 Low         ❓ Fallback
   ≥ HIGH_MIN      between        ≤ LOW_MAX      no match /
                   thresholds     and > 0        invalid budget
        │              │              │              │
   Book a call     Tailored       Starter        General
   email + CTA     proposal       options        follow-up
```

### 1. Lead Capture

- **Custom Webhook**: accepts JSON from any form builder, website or tool (Typeform, Tally, Webflow, custom forms and so on), with a predefined data structure.
- **Email validation filter**: only leads with a valid-looking email continue, so junk submissions never reach the Sheet or the inbox.

### 2. Centralized Configuration

- **⚙️ CONFIG module** (Tools › Set multiple variables): holds every user-editable value, including Sheet ID, business name, email subject, budget thresholds and booking link.
- **No hardcoded accounts**: Google Sheets uses *Enter manually* mode, with the Spreadsheet ID and sheet name mapped from CONFIG.

### 3. Budget Qualification

- **Input cleaning**: `parseNumber()` combined with `replace()` strips currency symbols and thousands separators, because form tools send budgets as text such as `"$7,500"`.
- **Router with four routes**: High, Mid and Low tiers, each driven by CONFIG thresholds, plus Make's native **fallback route** for anything unmatched.
- **Tailored HTML emails**: each tier gets its own message. High-budget leads receive a *Book Your Strategy Call* button.

### 4. Fault Tolerance

- **Resume** handlers keep the scenario running when a step fails (Sheet unavailable, non-numeric budget).
- **Skip** handlers on every Gmail module end the run cleanly without flagging the scenario as errored.

---

## Budget Routing Logic

| Route | Condition | Email sent |
|---|---|---|
| 💎 High budget | `LEAD_BUDGET ≥ HIGH_BUDGET_MIN` (default 5000) | Priority strategy call with booking button |
| 📈 Mid budget | `LOW_BUDGET_MAX < LEAD_BUDGET < HIGH_BUDGET_MIN` | Tailored proposal within 24 hours |
| 🌱 Low budget | `0 < LEAD_BUDGET ≤ LOW_BUDGET_MAX` (default 1000) | Starter packages and options |
| ❓ Fallback | No other route matches (missing or invalid budget) | General follow-up email |

---

## Sample Webhook Payloads

**Scenario 1**

```json
{
  "name": "John Carter",
  "email": "john@example.com",
  "phone": "+1 512 555 0142",
  "source": "Website Form"
}
```

**Scenario 2**

```json
{
  "name": "Sarah Mitchell",
  "email": "sarah@example.com",
  "phone": "+1 415 555 0198",
  "budget": "$7,500",
  "service": "Website Redesign"
}
```

---

## Testing

| Test payload | Expected result | Outcome |
|---|---|---|
| Lead with name, email, phone, source | Row added to Sheet + welcome email | ✅ Passed |
| Budget `"$7,500"` | 💎 High route | ✅ Passed |
| Budget `"$2,500"` | 📈 Mid route | ✅ Passed |
| Budget `"$500"` | 🌱 Low route | ✅ Passed |
| Budget `"not sure"` | ❓ Fallback route, no crash | ✅ Passed |

---

## Setup

1. In Make, create a new scenario, open the **⋯** menu, choose **Import Blueprint** and select a `.json` file from `/blueprints`.
2. Connect your Google account on the Google Sheets and Gmail modules. Every module reuses the same connection.
3. Open the **⚙️ CONFIG** module and fill in your values.
4. For Scenario 1, create a Google Sheet with a tab named `Leads` and the headers `Timestamp | Name | Email | Phone | Source`.
5. Copy your webhook URL from the first module and paste it into your form tool.
6. Click **Run once**, submit a test lead and switch the scenario **ON**.

---

## Repository Structure

```text
├── blueprints/
│   ├── 01_webhook_to_sheets_welcome_email.json    # Scenario 1 Make blueprint
│   └── 02_lead_qualification_router.json          # Scenario 2 Make blueprint
├── docs/
│   └── project_report.pdf                         # 2-page project report
├── Lead Capture & Budget-Based Qualification Automation (Make).jpg
└── README.md
```

---

## Key Features

- **Single-module setup**: users edit one CONFIG module, and budget thresholds change without touching any filter.
- **Handles messy input**: budgets like `"$7,500"`, `"500"` or `"not sure"` are handled without errors.
- **Native fallback route**: every lead gets a reply, even when the budget can't be qualified.
- **Error handling throughout**: Resume and Skip handlers keep runs green and avoid silent failures.
- **Self-documenting**: every module is renamed with a descriptive label and carries notes inside the Make editor.
- **Branded HTML emails**: inline-styled templates personalized with the lead's name, service and business name.

---

## Future Additions

- CRM sync (HubSpot, GoHighLevel, Pipedrive) per budget tier
- AI lead scoring on the message field using an LLM
- Slack or WhatsApp alerts for high-budget leads
- Multi-step nurture email sequences
- Lead logging in Scenario 2 with a conversion dashboard
- Duplicate detection and data enrichment

---

## Tech Stack

| Layer | Technology |
|---|---|
| Workflow Orchestration | Make.com |
| Trigger | Custom Webhook (JSON) |
| Logic | Router, filters, fallback route, Set variable(s) |
| Data Storage | Google Sheets |
| Email Delivery | Gmail (Raw HTML) |
| Error Handling | Resume and Skip directives |
