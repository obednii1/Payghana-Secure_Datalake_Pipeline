# PayGhana Secure Data Lake — Databricks Build

A secure, governed data lakehouse pipeline for Ghanaian fintech KYC data, built on Databricks Unity Catalog.


## The Problem

Ghanaian fintechs, banks and mobile money providers collect sensitive KYC data — Ghana Card numbers,
phone numbers, transaction history — but most platforms have certain compounding gaps:

No governance over who sees sensitive data.
PII ends up scattered across spreadsheets, exports, and analysts' laptops with no access
control beyond "who has the password." A single careless export of raw Ghana Card numbers
is a serious incident under Ghana's Data Protection Act, 2012 (Act 843) — and most teams
can't quickly answer a regulator's most basic question: *who accessed this customer's data,
and when?*



## What PayGhana Does

PayGhana Secure Data Lake addresses these gaps: a governed Bronze/Silver/Gold lakehouse with
masking, row-level access control, and a full audit trail (Databricks).

## What's in this repo

- **`PayGhana KYC Synthetic Data Pipeline`** — a Databricks notebook that:
  1. Creates a `payghana` catalog with `raw`, `silver`, and `gold` schemas (Unity Catalog).
  2. Generates synthetic Ghanaian KYC records (names, Ghana Card numbers, phone numbers, region) using Faker — no real customer data is used anywhere in this project.
  3. Loads and validates the data into a `raw` → `silver` pipeline: deduplicating on Ghana Card number, dropping incomplete rows, and enforcing regex format checks on phone numbers and Ghana Card numbers before promotion.
  4. Creates a `gold` view over the validated Silver table.
  
## Governance layer (companion SQL, run separately)

The access-control half of this project — the part that actually enforces who can see what — is a SQL script (`governance_and_audit.sql`) run in the Databricks SQL editor after this notebook, which:

- Attaches a **column mask** to the Ghana Card column, so it's shown masked (`GHA-*******8`) to everyone except a `compliance_leads` group.
- Attaches a **row filter** so a `compliance_team` group only sees Greater Accra records, while `risk_analysts` see every region but never the raw card.
- Creates two role-scoped views (`v_risk_analyst`, `v_compliance_accra`) with `GRANT`s matching that split.
- Includes audit queries against Unity Catalog's system tables for "who accessed this record."


## Why Databricks specifically

This build leans into Unity Catalog's native governance primitives rather than translating another platform's approach line-for-line:

- Masking and row filtering are first-class table-level operations (`ALTER TABLE ... SET MASK`, `ALTER TABLE ... SET ROW FILTER`), not application-layer workarounds.
- Group membership checks use `is_member()` — a deliberate choice for **Databricks Free Edition**, which has no account console and therefore only supports workspace-local groups (not the account-level groups that `IS_ACCOUNT_GROUP_MEMBER()` checks). 
- The Bronze/Silver/Gold layering is the same "medallion architecture" Databricks documents as its own recommended pattern.

## Known platform limitations (Free Edition)

Worth being upfront about rather than discovered the hard way:

- **No account console.** Groups are created per-workspace instead (Settings → Identity and access → Groups), which is why this project uses `is_member()` as noted above.
- **Unity Catalog Data Classification is not usable.** Attempting to enable it on Free Edition returns "Model is not accessible" — its underlying hosted AI model doesn't appear to be exposed at this tier. This doesn't affect masking or row-filter enforcement, which don't depend on classification.
- **System tables for audit** (`system.access.audit`, `system.query.history`) are normally enabled by an account admin — unavailable here for the same reason as the account console.
- **Testing denial requires a second, non-owner user.** Unity Catalog object owners implicitly have every privilege on objects they create, regardless of explicit `GRANT`s to other principals — so testing "does this group actually get blocked" can't be done from the account that built the project. It requires inviting a genuinely separate user into the workspace.

## What's simulated vs. real

The KYC records and login events are entirely synthetic (Faker-generated) — no real customer data. The masking, row-filtering, and access-control logic, however, is built and tested against real Unity Catalog enforcement, not mocked.

## Regulatory context

Built with Ghana's Data Protection Act, 2012 (Act 843) and Bank of Ghana's fintech oversight in mind — specifically the ability to answer "where is this customer's data, and who has accessed it," which is the kind of question a DPC audit or a BoG Cyber and Information Security Directive review would ask. Exact statute section numbers should be verified against the primary source before being cited elsewhere (e.g., a thesis or CV) — see the project's main README for a longer note on this.
