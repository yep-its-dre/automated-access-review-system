# Automated Access Review System

An n8n workflow that automates the quarterly access review process — cross-referencing employee directories, application access records, and role-based access policies to surface violations automatically, then writing findings to a report and posting a summary to Slack.

## The Problem

Access reviews are a standard requirement for SOC 2 compliance and general security hygiene. Done manually, they mean exporting spreadsheets from multiple systems, comparing them against a policy document, and flagging exceptions — a process that can take hours and is prone to being rushed or skipped. Departed employees keeping access, engineers with Salesforce admin rights they never needed, and dormant accounts that were provisioned and forgotten are exactly the risks this kind of review is supposed to catch.

This workflow runs the full review in under two minutes and produces a structured findings report ready for IT or a compliance auditor to action.

## What It Detects

| Flag | Description |
|---|---|
| **Orphaned Account** | Departed employee still has active application access |
| **Over-Provisioned** | User has access to an app their role/department should not have |
| **Elevated Access** | User holds Admin rights when policy specifies User level |
| **Stale Account** | No login activity in 90+ days |

## Workflow Overview

```
Manual Trigger
    │
    ▼
Read Employee Directory (Google Sheets)
    │
    ▼
Read Access Matrix (Google Sheets)
    │
    ▼
Read Access Policy (Google Sheets)
    │
    ▼
Compare Access Against Policy (Code)
    │  ↳ flags orphaned, over-provisioned, elevated, stale
    ▼
Write Review Findings → Google Sheets (append to report tab)
    │
    ▼
Generate Slack Summary (Code)
    │
    ▼
Post to #access-review-alerts (Slack)
```

## Data Structure

The workflow reads from three Google Sheets tabs and writes to a fourth.

**Tab 1 — Employee Directory**

| Column | Description |
|---|---|
| Name | Full name |
| Email | Work email (join key) |
| Department | Engineering, Sales, HR, etc. |
| Role | Job title used for policy lookup |
| Status | `Active` or `Departed` |

**Tab 2 — Access Matrix**

| Column | Description |
|---|---|
| Employee Email | Join key to directory |
| App | Application name (Google Workspace, Jira, Salesforce, etc.) |
| Access Level | `User` or `Admin` |
| Last Login | ISO date — used to calculate staleness |

**Tab 3 — Access Policy**

| Column | Description |
|---|---|
| Department | Join key with directory |
| Role | Join key with directory |
| App | Application name |
| Expected Access | `User`, `Admin`, or `None` (no access expected) |

**Tab 4 — Review Report** (output)

| Column | Description |
|---|---|
| Review Date | Date the workflow ran |
| Employee | Name |
| Email | Email |
| Department | Department |
| App | Application flagged |
| Current Access | What they actually have |
| Expected Access | What the policy says they should have |
| Flag | Orphaned Account / Over-Provisioned / Elevated Access / Stale Account |
| Reviewer | `Automated` |
| Action Taken | `Pending Review` (updated manually after remediation) |
| Action Date | Left blank for auditor to fill in |

## Files

| File | Description |
|---|---|
| `Access_Review_System.json` | n8n workflow — import directly into your n8n instance |
| `Employee_Directory.csv` | 10-employee sample dataset for Google Sheets Tab 1 |
| `Access_Matrix.csv` | 39-row sample access records for Tab 2 |
| `Access_Policy.csv` | 49-row role-based access policies for Tab 3 |

The sample data includes one instance of each flag type to demonstrate all detection paths:
- **Orphaned:** Amanda Foster (Departed, Sales Director) still has Google Workspace and Salesforce access
- **Over-Provisioned:** Sarah Mitchell (Engineer) has Salesforce access — Engineering policy says `None`
- **Elevated:** David Park (Senior Engineer) has Jira Admin — policy says User
- **Stale:** Rachel Chen (Finance) last logged into Google Workspace on 2025-12-01

## Setup

### 1. Create the Google Sheet

Import the four CSV files into a single Google Sheet with four tabs named exactly:
- `access-review-tab1-employee-directory`
- `access-review-tab2-access-matrix`
- `access-review-tab3-access-policy`
- `access-review-tab4-review-report` (create empty with column headers matching Tab 4 above)

### 2. Import the Workflow

In n8n: **Workflows** → **Import from File** → select `Access_Review_System.json`

### 3. Connect Credentials

The workflow uses two credentials:
- **Google Sheets OAuth2** — connect your Google account in n8n credentials
- **Slack OAuth2** — connect your Slack workspace

Update the Google Sheet document ID in each of the four Google Sheets nodes to point to your new sheet.

### 4. Create the Slack Channel

Create a `#access-review-alerts` channel in Slack and update the channel ID in the **Post Access Review Alert** node.

### 5. Run

Click the **Run Access Review** manual trigger. Results appear in the report tab and a Slack summary posts within seconds.

## Sample Slack Output

```
Access Review Completed — 2026-06-16

Findings Summary:
• Orphaned Accounts: 2
• Over-Provisioned: 1
• Elevated Access: 1
• Stale Accounts (90+ days): 1

Total Flags Requiring Action: 5

Full report available in the Access Review Report tab.
```

## Built With

- [n8n](https://n8n.io) — workflow automation
- Google Sheets — data source and report output
- Slack — findings notification
- JavaScript — policy comparison logic
