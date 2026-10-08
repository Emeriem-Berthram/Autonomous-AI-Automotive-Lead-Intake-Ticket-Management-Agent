# Autonomous-AI-Automotive-Lead-Intake-Ticket-Management-Agent
# Autonomous AI Automotive Lead Intake & Ticket Management Agent

## Overview

This n8n workflow automates the initial intake and processing of automotive customer inquiries received through Gmail.

The workflow uses AI to extract customer and vehicle information, classify the customer's intent, retrieve vehicle and service pricing information from Google Sheets, and update the corresponding HubSpot ticket.

The primary goal is to reduce manual lead triage while ensuring that every customer inquiry is captured in the CRM and enriched with structured information.

---

## Workflow Title

**Autonomous AI Automotive Lead Intake & Ticket Management Agent**

---

## Workflow Architecture

```text
Gmail Trigger
      ↓
Create HubSpot Ticket
      ↓
AI Data Extraction & Classification
      ↓
Vehicle Tier Lookup
      ↓
Pricing Master Lookup
      ↓
JavaScript Price Processing
      ↓
Update HubSpot Ticket
      ↓
HTTP Request (HubSpot API)
```

---

## 1. Gmail Trigger

**Node:** `Gmail Trigger`

The workflow monitors the connected Gmail account for incoming customer emails.

It runs:

* Every minute
* Using the configured Gmail account
* As the starting point for the automotive customer-intake process

The email subject and body are used as the source information for the rest of the workflow.

---

## 2. Create HubSpot Ticket

**Node:** `Create a ticket`

A HubSpot ticket is created immediately after the email is received.

The current configuration creates the ticket using:

* Pipeline: `0`
* Stage: `1`
* Ticket name: Gmail email subject

This ensures that the customer inquiry is captured in HubSpot before the AI enrichment process continues.

### Important

Creating the HubSpot ticket early is useful because it provides a CRM record that can subsequently be enriched with the information extracted from the customer's email.

---

## 3. AI Data Extraction & Classification

**Node:** `Basic LLM Chain`

The AI analyzes the customer email and extracts structured information.

The AI is instructed to identify:

* Client name
* Email
* Phone
* Vehicle year
* Vehicle make
* Vehicle model
* Service requested
* Intent
* Sensitive information
* Sensitivity reason
* Summary

### Supported Intent Categories

The workflow restricts intent classification to:

* `Sales`
* `Complaint`
* `Payment`
* `Booking`
* `Internal Task`

### Sensitive Inquiry Detection

The AI marks an inquiry as sensitive when it involves situations such as:

* Refunds
* Legal threats
* Attorneys
* Lawsuits
* Chargebacks
* Disputes
* Fraud allegations
* Compensation demands
* Serious formal complaints

The AI is also instructed not to invent missing information.

Missing values should be returned as `null`.

---

## 4. Groq AI Model

**Node:** `Groq Chat Model`

The workflow uses the Groq Chat Model to power the AI extraction and classification step.

Current model configuration:

```text
openai/gpt-oss-20b
```

The model receives the customer's email through the `Basic LLM Chain`.

---

## 5. Vehicle Tier Lookup

**Node:** `Get row(s) in sheet`

The workflow connects to the Google Sheets document:

**Automotive AI Data**

It reads from the:

**Vehicle Tiers**

worksheet.

This lookup is intended to determine the appropriate vehicle tier for the customer's vehicle.

The vehicle tier can subsequently be used when determining service pricing.

---

## 6. Pricing Master Lookup

**Node:** `Get row(s) in sheet1`

The workflow reads pricing information from the:

**Pricing Master**

worksheet.

The pricing data contains fields such as:

```text
Vehicle Tier
Service
Minimum Price
Maximum Price
```

This allows the workflow to retrieve the applicable price range for a requested automotive service.

---

## 7. Price Processing

**Node:** `Code in JavaScript`

The JavaScript node converts the pricing spreadsheet data into a cleaner structure.

The output contains:

```json
{
  "vehicleTier": "...",
  "service": "...",
  "minimumPrice": 0,
  "maximumPrice": 0,
  "estimatedPrice": "0 - 0"
}
```

The `estimatedPrice` field combines the minimum and maximum values into a customer-friendly price range.

For example:

```text
150 - 300
```

---

## 8. HubSpot Ticket Update

**Node:** `Update a ticket`

The workflow then updates the HubSpot ticket with the processed information.

The current workflow contains a hard-coded ticket ID:

```text
433400785139
```

### Recommended improvement

For production use, the ticket ID should **not** be hard-coded.

Instead, the workflow should capture the ID returned by the `Create a ticket` node and pass that ID dynamically into the update operation.

For example:

```text
{{ $('Create a ticket').item.json.id }}
```

This ensures that every incoming email updates its own newly created HubSpot ticket.

---

## 9. HubSpot HTTP Request

**Node:** `HTTP Request`

The workflow also contains a direct HubSpot API request using:

```text
PATCH
```

against the HubSpot ticket endpoint.

Current configuration targets:

```text
/crm/v3/objects/tickets/433400785139
```

### Important

This is currently connected in parallel with the `Update a ticket` node.

Therefore, the workflow is currently attempting two HubSpot update paths:

```text
Code in JavaScript
       ├──→ Update a ticket
       └──→ HTTP Request
```

If both nodes perform the same update, this can create unnecessary duplicate API operations.

For a cleaner production workflow, use **one** HubSpot update method unless the HTTP Request performs a specific update that the native HubSpot node cannot perform.

---

# Data Flow

The overall data flow is:

```text
Customer Email
      ↓
Gmail
      ↓
HubSpot Ticket Created
      ↓
AI Extraction
      ↓
Customer Information
Vehicle Information
Service Information
Intent
Sensitivity
      ↓
Vehicle Tier Lookup
      ↓
Pricing Lookup
      ↓
Price Range Generated
      ↓
HubSpot Ticket Enriched
```

---

# Example Customer Inquiry

Example incoming email:

```text
Subject:
Brake service request

Body:
Hi I'm James Doe I have a Jeep Highlander 2025
and need brake service. Please let me know the
estimated cost.
```

The AI should extract information similar to:

```json
{
  "client_name": "James Doe",
  "vehicle_year": "2025",
  "vehicle_make": "Jeep",
  "vehicle_model": "Highlander",
  "service_requested": "Brake Service",
  "intent": "Booking",
  "sensitive": false
}
```

The pricing lookup can then determine the applicable service price range based on the vehicle tier and requested service.

---

# Key Technologies

| Technology    | Purpose                   |
| ------------- | ------------------------- |
| n8n           | Workflow automation       |
| Gmail         | Customer inquiry intake   |
| HubSpot       | CRM and ticket management |
| Groq          | AI processing             |
| Google Sheets | Vehicle and pricing data  |
| JavaScript    | Data transformation       |
| HubSpot API   | Direct CRM updates        |

---

# Current Limitations

The current workflow is a working foundation but should be improved before production deployment.

### 1. Hard-coded HubSpot Ticket ID

The ticket ID:

```text
433400785139
```

should be replaced with the ID returned from the ticket creation node.

### 2. Vehicle Tier Filtering

The Vehicle Tiers Google Sheets node currently has no filters configured.

The workflow should eventually match the extracted vehicle information against the correct spreadsheet row.

### 3. Pricing Filtering

The Pricing Master node currently retrieves spreadsheet data without explicit filtering.

Ideally, pricing should be matched using:

```text
Vehicle Tier + Service
```

### 4. Duplicate HubSpot Update Path

The JavaScript node currently connects to both:

```text
Update a ticket
HTTP Request
```

Only one should be used for the same update operation.

### 5. HubSpot Ticket Properties

The extracted AI fields should be mapped explicitly to the appropriate HubSpot ticket properties, including:

* Client name
* Contact information
* Vehicle make
* Vehicle model
* Vehicle year
* Service requested
* Intent
* Vehicle tier
* Estimated price
* Summary
* Sensitivity status

---

# Recommended Production Architecture

A more complete production version should follow this structure:

```text
Gmail Trigger
      ↓
Generate Tracking Number
      ↓
Create HubSpot Ticket
      ↓
AI Extract Information
      ↓
AI Classify Intent
      ↓
Vehicle Tier Lookup
      ↓
Pricing Lookup
      ↓
Update HubSpot Ticket
      ↓
Find/Create HubSpot Contact
      ↓
Associate Contact with Ticket
      ↓
Is Sensitive?
    ↙       ↘
  YES        NO
   ↓          ↓
Manual       Activity
Review       Log
   ↓          ↓
Slack        Slack
Review       Notification
              ↓
             Gmail
              ↓
          Customer Reply
```

This architecture provides a more complete autonomous automotive customer-intake system while keeping sensitive or high-risk cases available for human review.

---

# Objective

The workflow is designed to transform an unstructured automotive customer email into a structured CRM record automatically.

The desired outcome is:

**Email → AI Understanding → Vehicle/Service Lookup → CRM Enrichment → Automated Customer Handling**

while minimizing manual data entry and allowing human intervention for sensitive cases.

---

# Status

**Workflow Type:** Automotive AI Operations Agent
**Platform:** n8n
**CRM:** HubSpot
**Email Source:** Gmail
**AI Provider:** Groq
**Data Source:** Google Sheets
**Primary Function:** Automated automotive lead intake, classification, pricing lookup, and HubSpot ticket enrichment
