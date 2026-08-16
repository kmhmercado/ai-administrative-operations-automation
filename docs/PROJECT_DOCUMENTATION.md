# Project Documentation

## AI Administrative Operations Automation

This document provides a detailed technical and functional explanation of the **AI Administrative Operations Automation** project.

The system was built in n8n and combines AI classification, Google Sheets tracking, Google Calendar scheduling, RSVP monitoring, Gmail notifications, and scheduled operational reporting.

---

## 1. Project Purpose

The purpose of this project is to automate repetitive administrative operations tasks that would otherwise require manual review, routing, scheduling, follow-up, and reporting.

The system is designed to support common administrative scenarios such as:

- receiving internal administrative requests
- classifying requests by type
- assigning priority
- logging requests into a tracker
- escalating High and Urgent requests
- scheduling meetings
- tracking RSVP responses
- updating meeting statuses
- identifying overdue work
- generating a daily administrative operations summary

The project demonstrates how multiple automated workflows can work together as one connected operational system.

---

## 2. System Components

The project contains three n8n workflows.

### Workflow 1 — AI Administrative Request Management System

Primary responsibility:

**Receive, classify, route, log, notify, and schedule administrative requests.**

Main flow:

```text
Webhook
   |
   v
Normalize Request Data
   |
   v
AI Request Classifier
   |
   v
Extract Request Classification
   |
   v
Route by Request Category
   |
   +--------------------------+
   |                          |
   v                          v
Scheduling              Other Requests
   |                          |
   v                          v
Extract Meeting         Route by Priority
Details                       |
   |                    +-----+-----+-----+
   v                    |     |     |     |
Create Calendar        Urgent High Medium Low
Event                    |     |     |     |
   |                     +-----+-----+-----+
   v                           |
Log Scheduling                 v
Request                 Log Administrative
   |                     Request
   v                           |
Send Meeting                   +--> High/Urgent
Invitation Notice                   Priority Alert
```

---

### Workflow 2 — Calendar RSVP Status Tracker

Primary responsibility:

**Synchronize Google Calendar RSVP responses with the Administrative Request Tracker.**

Main flow:

```text
Google Calendar Event Updated
          |
          v
Verify Managed Calendar Event
          |
          v
Map RSVP Status
          |
          v
Update Matching Google Sheets Row
```

The workflow uses the Calendar Event ID as the matching value when updating the tracker.

RSVP status mapping:

| Calendar Response | Internal Status |
|---|---|
| accepted | Confirmed |
| declined | Declined |
| tentative | Tentative |
| needsAction | Pending Confirmation |

---

### Workflow 3 — Daily Administrative Operations Brief

Primary responsibility:

**Review active administrative work and generate a daily operational report.**

Main flow:

```text
Daily Schedule Trigger
        |
        v
Load Administrative Requests
        |
        v
Filter Requests Needing Attention
        |
        v
Exclude Closed Requests
        |
        v
Aggregate Attention Items
        |
        +------------------------+
        |                        |
        v                        v
Load Today's Calendar       Request Data
Events                       |
        |                    |
        +---------+----------+
                  |
                  v
        Combine Requests & Schedule
                  |
                  v
        Generate Daily Operations Brief
                  |
                  v
        Send Daily Operations Brief
```

---

## 3. Workflow 1 — Request Processing Logic

Workflow 1 begins with a webhook that receives administrative request data.

The request is normalized into a consistent structure containing:

- requester name
- requester email
- department
- subject
- request details
- requested deadline
- status

The initial status is:

`New`

---

## 4. AI Request Classification

The normalized request is passed to an AI model.

The AI returns structured information including:

- `category`
- `priority`
- `summary`
- `action_required`
- `suggested_deadline`
- `requires_notification`

Supported categories:

- Document Request
- Scheduling
- Finance/Admin
- Procurement
- Customer/Client Concern
- Internal Support
- General Inquiry
- Other

Supported priority levels:

- Low
- Medium
- High
- Urgent

Priority is determined from:

- actual deadline
- business impact
- operational consequences
- urgency of required action

The workflow is specifically designed not to classify a request as Urgent only because the requester uses words such as `urgent` or `ASAP`.

---

## 5. Request Category Routing

After classification, the workflow routes requests according to category.

### Scheduling Requests

Scheduling requests are sent to a dedicated scheduling branch.

The system extracts:

- meeting title
- meeting date
- start time
- end time
- attendee email
- meeting purpose

If an end time is not explicitly provided, the workflow can infer a one-hour meeting duration.

The Calendar event is created using the `Asia/Manila` timezone.

After event creation:

- the request is logged into Google Sheets
- the Calendar Event ID is stored
- the Calendar Event link is stored
- the initial status becomes `Pending Confirmation`
- an automated scheduling notification is sent through Gmail

---

### Other Administrative Requests

Non-scheduling requests are routed according to priority.

All priority levels are logged into Google Sheets.

High and Urgent requests also trigger Gmail alerts.

```text
Low      -> Google Sheets
Medium   -> Google Sheets
High     -> Google Sheets + Gmail Alert
Urgent   -> Google Sheets + Gmail Alert
```

---

## 6. Administrative Request Tracker

Google Sheets acts as the central operational tracker.

The tracker includes fields such as:

- request_id
- submitted_at
- requester_name
- requester_email
- department
- subject
- request_details
- category
- priority
- summary
- action_required
- suggested_deadline
- status
- meeting_title
- meeting_date
- start_time
- end_time
- attendee_email
- calendar_event_id
- calendar_event_link

Request IDs are generated automatically using a timestamp-based format similar to:

```text
ADM-20260814083606
```

---

## 7. Calendar Event Management

Scheduling requests create Google Calendar events automatically.

The event contains:

- meeting title
- date
- start time
- end time
- attendee
- meeting purpose
- requester information
- original request context

The attendee receives the Calendar invitation and can respond using:

- Yes
- No
- Maybe

The RSVP response is handled by Workflow 2.

---

## 8. RSVP Status Synchronization

Workflow 2 monitors Calendar event updates.

Only managed project events are processed.

The workflow checks the event description for the project marker:

```text
Requested By:
```

This prevents unrelated personal Calendar events from being processed.

The attendee response is extracted and mapped to an internal status.

Example:

```text
accepted
   |
   v
Confirmed
```

The correct Google Sheets row is found using:

`calendar_event_id`

The workflow updates the existing row rather than creating a duplicate.

---

## 9. Daily Attention Filtering

Workflow 3 reviews the Administrative Request Tracker for items requiring attention.

A request is considered relevant if it meets at least one of these conditions:

- Priority = Urgent
- Priority = High
- Status = Pending Confirmation
- Status = Tentative
- Suggested deadline is before the current day

After that, closed statuses are excluded.

Excluded statuses include:

- Completed
- Confirmed
- Cancelled
- Declined

This ensures the daily report focuses only on active work.

---

## 10. Calendar Review

Workflow 3 also reads Google Calendar for events scheduled during the current day.

The Calendar query uses the beginning and end of the current day as the time window.

The workflow is configured so that even if there are no events, the automation can continue and generate a report.

---

## 11. AI Daily Operations Brief

Active administrative requests and Calendar data are passed to an AI model.

The AI generates a structured HTML report containing:

1. Daily Overview
2. Urgent Items
3. High Priority Items
4. Overdue Items
5. Pending or Tentative Meetings
6. Today's Schedule
7. Recommended Priorities

The prompt includes guardrails designed to reduce hallucinated information.

The AI is instructed to:

- use only provided data
- treat `suggested_deadline` as authoritative
- identify overdue items based on the current date
- avoid inventing systems, penalties, approvals, or consequences
- avoid outdated relative wording
- avoid creating unsupported follow-up actions
- return HTML only

---

## 12. HTML Email Reporting

The final report is sent through Gmail as HTML.

The report uses:

- structured headings
- compact tables
- visual separation between sections
- subtle priority highlighting
- red emphasis for Urgent and Overdue items
- dark-orange emphasis for High-priority items

The email ends with:

```text
Generated automatically by the Administrative Operations Automation System.
```

---

## 13. Trigger Types Used

The project demonstrates three different automation trigger patterns.

### Webhook Trigger

Used in Workflow 1.

Purpose:

Receive new administrative requests.

### Calendar Event Trigger

Used in Workflow 2.

Purpose:

Detect attendee RSVP changes.

### Schedule Trigger

Used in Workflow 3.

Purpose:

Generate the Daily Administrative Operations Brief automatically.

---

## 14. Integrations

The project uses the following integrations:

### n8n

Used for workflow orchestration and automation logic.

### Groq

Used as the LLM inference provider.

### GPT-OSS 120B

Used for:

- request classification
- structured information extraction
- scheduling information extraction
- daily report generation

### Google Sheets

Used as the central Administrative Request Tracker.

### Google Calendar

Used for:

- automated meeting creation
- attendee invitations
- RSVP monitoring
- daily schedule retrieval

### Gmail

Used for:

- High-priority alerts
- Urgent alerts
- meeting notifications
- Daily Administrative Operations Brief delivery

---

## 15. Testing Strategy

The system was tested in stages before final end-to-end validation.

### Workflow 1 Tests

Test cases:

- Low
- Medium
- High
- Urgent
- Scheduling

Expected results:

```text
Low        -> Tracker
Medium     -> Tracker
High       -> Tracker + Gmail Alert
Urgent     -> Tracker + Gmail Alert
Scheduling -> Calendar + Tracker + Gmail
```

---

### Workflow 2 Tests

RSVP changes were tested for:

```text
needsAction -> Pending Confirmation
tentative   -> Tentative
accepted    -> Confirmed
declined    -> Declined
```

The tests also verified that:

- the same Sheet row is updated
- the Calendar Event ID remains consistent
- duplicate tracker rows are not created

---

### Workflow 3 Tests

The reporting workflow was tested for:

- active High items
- active Urgent items
- overdue items
- closed request exclusion
- pending meeting detection
- Calendar retrieval
- no-event conditions
- AI report generation
- HTML formatting
- Gmail delivery

---

## 16. End-to-End Validation

The final system was tested as one continuous operational scenario.

```text
Administrative Request Submitted
        |
        v
Workflow 1
        |
        v
AI Classification
        |
        v
Scheduling Request Detected
        |
        v
Google Calendar Event Created
        |
        v
Tracker = Pending Confirmation
        |
        v
Attendee Accepts Invitation
        |
        v
Workflow 2
        |
        v
Tracker = Confirmed
        |
        v
Workflow 3
        |
        v
Administrative Data Reviewed
        |
        v
AI Daily Brief Generated
        |
        v
Gmail Report Delivered
```

---

## 17. Security and Public Repository Preparation

The original n8n exports contained deployment-specific information.

Before publishing the workflow templates, the public JSON files were sanitized.

Removed or replaced information includes:

- credential reference IDs
- personal Gmail addresses
- Google Sheet IDs
- Google Calendar IDs
- cached Google URLs
- webhook IDs
- workflow IDs
- workflow version IDs
- n8n instance metadata

The public workflow templates are also set to:

```json
"active": false
```

This prevents the public copies from being treated as active production workflows immediately after import.

---

## 18. Public Workflow Configuration

Users importing the public workflow files must configure their own:

- Groq API credentials
- Google Sheets OAuth credentials
- Google Calendar OAuth credentials
- Gmail OAuth credentials
- Google Sheet ID
- Google Calendar ID
- notification email address

The public files are intended as reusable templates and architecture examples.

---

## 19. Operational Lifecycle

The complete automated lifecycle can be summarized as:

```text
Receive
   |
   v
Normalize
   |
   v
Analyze
   |
   v
Prioritize
   |
   v
Route
   |
   +------> Notify
   |
   +------> Schedule
                 |
                 v
               Track
                 |
                 v
               Report
```

Or more simply:

**Receive → Analyze → Prioritize → Route → Schedule → Track → Report**

---

## 20. Skills Demonstrated

This project demonstrates hands-on experience with:

- n8n workflow design
- multi-workflow automation architecture
- webhook automation
- scheduled automation
- event-driven automation
- AI request classification
- LLM prompt engineering
- structured information extraction
- conditional routing
- filter logic
- Google Sheets integration
- Google Calendar integration
- RSVP synchronization
- Gmail automation
- OAuth authentication
- operational status tracking
- automated reporting
- HTML email generation
- workflow testing
- debugging
- end-to-end validation
- security sanitization
- GitHub portfolio documentation

---

## 21. Final Outcome

The final project functions as an automated administrative operations assistant.

It reduces the need to manually:

- review incoming requests
- assign priority
- update trackers
- send urgent notifications
- create Calendar events
- monitor attendee responses
- update meeting statuses
- identify overdue work
- prepare daily reports

The result is a connected administrative automation system that demonstrates a practical business use of AI, workflow automation, and Google Workspace integrations.
