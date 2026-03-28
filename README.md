# CFPB Complaint Risk Control Tower

A Databricks lakehouse application for explainable complaint-driven risk monitoring, triage, and lightweight issue management using public CFPB complaint data.

## Overview

CFPB Complaint Risk Control Tower is a scoped demo application built on Databricks to show how public consumer complaint data can be transformed into an operational risk-monitoring workflow.

Rather than stopping at notebook analysis, this project models a small end-to-end lakehouse app:

- Public complaint data is ingested and curated
- Complaint activity is grouped into issue clusters
- Explainable scoring logic identifies elevated risk patterns
- Alerts are promoted into a triage queue
- Users can manage issues through persistent workflow state and audit history

The project is intentionally designed as a realistic v1, but it is certainly not intended as an end-to-end complaint management platform!

## Business Problem

Banks and financial institutions receive signals of emerging risk from many places, including consumer complaints. But complaint data is often either:

1. Analyzed only in static reporting; or
2. Handled in disconnected manual workflows.

This project demonstrates how complaint data can support a more consistent, deterministic, and automated approach to managing consumer complaint risk. Among the features of this approach:

- Identify emerging issue patterns by institution, product, issue, and time period
- Prioritize elevated clusters using transparent business logic
- Move alerts into a manageable queue
- Support ownership, status changes, notes, and history tracking
- Provide a dashboard for both summary monitoring and issue investigation

## Project Goals

This demo was built to show:

- Databricks lakehouse design using bronze / silver / gold / app layers
- Explainable, configurable risk logic instead of black-box modeling
- Persistent operational tables for issue workflow
- Reusable SQL views for queue, timeline, KPI, and summary reporting
- A working Databricks dashboard

## Core Workflow

The app follows this high-level flow:

**public complaint data → issue clustering → explainable risk scoring → triage queue → workflow / audit trail**

It supports two conceptual modes:

- **Institution Mode**: operational view for a single bank
- **Market Context Mode**: benchmarking view across a small peer set

## Scope

### v1 constraints

This is a deliberately scoped v1.

- Uses a limited CFPB complaint export for **five banks only**
- Bronze is a demo raw source, not the full CFPB universe
- Scoring is **rule-based and explainable**, not ML-driven
- Workflow is intentionally lightweight
- The design favors realism and clarity over feature breadth

## Architecture

The project uses a medallion-style lakehouse pattern in Databricks.

### Catalog

- `cfpb_risk`

### Schemas

- `bronze`
- `silver`
- `gold`
- `reference`
- `app`

### Table structure

#### Bronze
- `cfpb_risk.bronze.cfpb_complaints_raw`

#### Reference
- `cfpb_risk.reference.bank_map`
- `cfpb_risk.reference.issue_weights`

#### Silver
- `cfpb_risk.silver.cfpb_complaints_clean`

#### Gold
- `cfpb_risk.gold.issue_clusters_daily`
- `cfpb_risk.gold.issue_clusters_monthly`
- `cfpb_risk.gold.risk_alerts`

#### App
- `cfpb_risk.app.issues`
- `cfpb_risk.app.issue_events`  

The design uses monthly issue clusters as the primary operational scoring grain, while daily clusters support charts and investigation views. The main operational grain is:

**institution × product × issue × month** :contentReference

## Data Design Principles

A few important design choices shaped the implementation:

- Silver is curated at the complaint level, with reusable row-level derived fields
- Gold contains both Daily and Monthly issue-cluster tables
- The Gold-Monthly table is the main scoring grain; The Gold-Daily table exists for trend analysis and investigation
- Deterministic IDs are generated using hashing rather than auto-increment patterns
- App tables are persistent operational tables, not fully rebuilt presentation layers
- Views are used for reusable logic and dashboard consumption
- SQL transformations are split into explicit CTE layers where downstream calculations depend on earlier derived fields

These choices were made to keep the project credible as a small operational lakehouse application.

## Risk Scoring Approach

The scoring model is designed to be explainable and configurable.

The app uses transparent business logic supported by reference/configuration tables. This makes the output easier to understand, defend, and tune.

At a high level, the model evaluates complaint clusters using business-defined issue weighting and alerting logic, then assigns alert levels that can be triaged operationally.

## Operational Workflow

A core feature of the project is that alerts do not stop at analytics. They move into a persistent app layer for issue management.

### `app.issues`
Stores the current operational state of each issue, including fields such as:

- issue owner
- current status
- priority
- due date
- latest note
- timestamps

### `app.issue_events`
Stores the audit trail of workflow actions.

Standardized event types include:

- `CREATED`
- `OWNER_ASSIGNED`
- `STATUS_CHANGED`
- `NOTE_ADDED`

The workflow pattern consistently inserts the audit event first, then updates the issue record second.

### Supported v1 workflow actions

- assign / reassign owner
- update issue status
- add issue note

### Allowed v1 statuses

- `New`
- `In Review`
- `Escalated`
- `Closed`

Validation is intentionally narrow in v1: blank or invalid statuses are rejected, and same-status updates are prevented. The goal is to keep the workflow disciplined without building a full workflow engine.

## Views

The app includes reusable operational and dashboard-facing views.

### `v_issue_queue`
Active triage queue built over `app.issues`.

Key traits:

- Excludes closed issues
- Highlights unassigned and overdue items
- Supports priority-based sorting
- Intended to answer: **“What should I work next?”**

Recommended sort pattern:

1. Overdue first
2. Unassigned next
3. Priority / alert severity
4. Risk score
5. Due date / age

### `v_issue_timeline`
Timeline/history view over `app.issue_events` for issue investigation.

### `v_issue_kpis`
Single-row KPI view over `app.issues`, including:

- Open issue count
- Escalated issue count
- Overdue issue count
- Unassigned issue count
- High-risk open issue count

This remains a catalog view rather than a physical table.

### Other summary views
Grouped summary views exist by:

- Institution
- Current status
- Alert level

## Dashboard

A working Databricks dashboard has been built for v1. 

### Page 1: Control Tower Overview

Designed to provide a fast operational summary using:

- KPI cards from `v_issue_kpis`
- Bar charts by institution
- Bar charts by current status
- Bar charts by alert level
- Top active issues table from `v_issue_queue`

This page is intended to communicate:

- What the demo does
- How to interpret the queue
- Why the application is operationally credible

### Page 2: Issue Investigation

Designed for drill-down and issue review using:

- Issue selector / summary
- Timeline table from `v_issue_timeline`
- Optional investigation/trend visual

This page helps distinguish current issue state from historical workflow events.

## Demo Data Strategy

In ordert to populate the Dashboard view with realistic-looking issue-managent data, the app-layer has been seeded with deterministically-generated demo data.

To keep the backend honest, only the app layer is seeded with realistic workflow state. The bronze, silver, and gold layers remain grounded in real derived complaint and alert data. 

The demo reset script populates:

- Issue status mix
- Ownership distribution
- Priorities
- Due dates
- Notes
- Audit events

Seeded workflow history includes realistic event sequences such as owner assignment, review progression, escalation, note entry, and closure.

The seeding logic was later improved so timestamps feel historically real:

- `created_ts` is distributed across roughly the last 90 days
- Later event timestamps are anchored to created dates
- `updated_ts` reflects workflow stage rather than just current time
- Due dates are seeded relative to created date

This makes the dashboard, timelines, and KPIs feel less synthetic while keeping production logic separate from demo logic.

## Possible Build-Outs for v2

The following were intentionally left out of v1, but are the most promising future directions for a v2 build:

- Workflow transition matrix / workflow engine
- SLA engine
- Notifications
- Auth / entitlements
- Full front-end productization
- ML / predictive models

## Suggested Repo Structure

```text
cfpb-complaint-control-tower/
├── README.md
├── screenshots/
├── sql/
│   ├── bronze/
│   ├── silver/
│   ├── gold/
│   ├── app/
│   └── views/
├── notebooks/
└── docs/
