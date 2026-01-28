# Doorway Data Pipeline

A comprehensive ETL pipeline for extracting, transforming, and loading affordable housing application data from Doorway's production RDS database to AWS S3 and Redshift for analysis and reporting.

## Overview

The Doorway Data Pipeline processes housing application data through multiple stages, ensuring PII (Personally Identifiable Information) protection while enabling comprehensive analytics on affordable housing programs. The pipeline extracts data from a production PostgreSQL database (RDS), redacts sensitive information, transforms and normalizes the data structure, and loads it into a data warehouse for analysis.

## Architecture

```
RDS (doorway-prod) → S3 (doorway-datalake/raw) → Redshift (staging) → Redshift
       ↓                      ↓                          ↓
    Extract             PII Redaction              Transformation
    & Dump              & Raw Storage              & Normalization
```

## Pipeline Stages

### Stage I: Extract (RDS → S3 Raw)

The extraction stage pulls data from the production RDS database and stores it in S3 with appropriate PII handling.

**Key Operations:**
- Extract and redact PII data for secure storage
- Dump non-PII data as-is for raw storage
- Maintain referential integrity during redaction

**Table Categories:**


- **PII Tables** (7 tables): Require redaction before storage
  - `address`, `alternate_contact`, `applicant`, `applications`, `household_member`, `user_accounts`, `application_methods`

- **Non-PII Tables** (8 tables): Stored without modification
  - `accessibility`, `assets`, `demographics`, `listing_events`, `listing_images`, `listings`, `paper_applications`, `units`

- **Dictionary/Lookup Tables** (8 tables): Reference data
  - `reserved_community_types`, `unit_accessibility_priority_types`, `unit_rent_types`, `unit_types`, `jurisdictions`, `multiselect_questions`, `listing_multiselect_questions`, `user_roles`

**PII Redaction Strategy:**

| Table | Fields Redacted | Fields Retained |
|-------|----------------|-----------------|
| `address` | `street`, `street2` (for applicants only) | City, county, state, zip, coordinates |
| `alternate_contact` | `first_name`, `last_name`, `phone_number`, `email_address` | Type, agency, address reference |
| `applicant` | `first_name`, `middle_name`, `last_name`, `email_address`, `phone_number`, `birth_month`, `birth_day` | Birth year, work status, flags |
| `applications` | `additional_phone_number` | All metadata, references, status |
| `household_member` | `first_name`, `middle_name`, `last_name`, `birth_month`, `birth_day` | Birth year, relationship, flags |
| `user_accounts` | All personal data, authentication tokens | `mfa_enabled`, timestamps |
| `application_methods` | `phone_number` | Type, label, references |

### Stage II: Transform (S3 Raw → S3 Staging)

The transformation stage refactors the database schema for analytical efficiency, eliminating redundant tables and normalizing relationships.

**Processing Order:**

1. **Simple Staging** (6 tables): No dependencies, staged as-is
   - `address`, `applicant`, `application_methods`, `assets`, `listing_images`, `listing_multiselect_questions`

2. **Clean & Stage** (3 tables): Minimal transformations
   - `listing_events`: Drop redundant `start_date`
   - `paper_applications`: Drop duplicate timestamps
   - `household_member`: Drop duplicate timestamps

3. **Lookup Tables → Map to Target Tables** (9 tables): Inline lookup data
   - `accessibility` → `applications` (prefix: `accessibility_`)
   - `alternate_contact` → `applications` (prefix: `alternate_contact_`)
   - `demographics` → `applications` (extract ethnicity, prefix: `demographics_`)
   - `jurisdictions` → `listings` (as `jurisdiction`)
   - `reserved_community_types` → `listings` (as `reserved_community_type`)
   - `unit_accessibility_priority_types` → `units`
   - `unit_rent_types` → `units`
   - `unit_types` → `units`

4. **Transform Target Tables** (4 tables): After mappings complete
   - `units`: Drop redundant columns and foreign keys
   - `multiselect_questions`: Drop `jurisdiction_id`, `application_section`
   - `applications`: Merge `programs` → `preferences`, drop mapped foreign keys
   - `listings`: Drop replaced foreign keys and unused columns

5. **Create New Tables** (1 table): New analytical structures
   - `application_multiselect_questions`: Explode preferences from applications

**Not Staged**: `user_accounts`, `user_roles` (minimal analytical value, retained in /raw only)

**Key Transformations:**

- **Denormalization**: Inline 1:1 lookup tables into parent tables to simplify queries
- **Consolidation**: Merge `programs` and `preferences` columns in applications
- **Exploding**: Convert nested preference data into relational format
- **Cleaning**: Remove redundant timestamps and unused columns

## Data Dictionary

Detailed schemas for all tables are documented in:
- [`docs/extract.md`](docs/extract.md) - RDS source schemas and Stage I transformations
- [`docs/transform.md`](docs/transform.md) - Stage II transformations and final schemas

## Key Tables

### Core Application Data
- **applications**: Housing application records with applicant preferences and status
- **applicant**: Primary applicant information (PII-redacted)
- **household_member**: Household composition data (PII-redacted)
- **address**: Geographic data with applicant street addresses redacted

### Listing Information
- **listings**: Available housing opportunities with eligibility criteria
- **units**: Individual housing units with rent and occupancy details
- **listing_events**: Application deadlines, open houses, and other events

### Reference Data
- **multiselect_questions**: Preference questions for housing programs
- **application_multiselect_questions**: Applicant responses to preference questions
- **demographics**: Self-reported demographic information

## Technology Stack

- **Source Database**: PostgreSQL (AWS RDS)
- **Data Lake**: AWS S3
- **Data Warehouse**: AWS Redshift
- **ETL Orchestration**: AWS Glue
- **Languages**: Python, SQL

## Data Governance

### PII Protection
- Street-level addresses redacted for all applicants and household members
- Personal identifiers (names, emails, phone numbers) removed
- Birth dates aggregated to year only
- User authentication data fully stripped

### Data Retention
- Raw data with full PII redaction stored in S3 /raw
- Transformed data without sensitive fields in S3 /staging
- Both layers maintained for audit and recovery purposes

## Documentation

- [`extract.md`](docs/extract.md) - Extraction process and PII redaction details
- [`transform.md`](docs/transform.md) - Transformation logic and schema changes
- [`TODO.md`](TODO.md) - Planned enhancements and open items


