# Stage III: Redshift (doorway_staging) to Redshift (doorway_prod)

The third stage of the pipeline transforms the data loaded into the `doorway_staging` schema and refactors it for easier analysis and more efficient organization. Transformed data is moved to the `doorway_prod` schema, the end point of the pipeline, while also maintaining the integrity of `doorway_staging`.

## Transformation Types

The transformation pipeline follows a strict dependency order to ensure data integrity:

```
Direct Copy
├── address
├── applicant  
├── application_methods
├── assets
├── listing_images
└── listing_multiselect_questions

Remove Unnecessary/Redundant Columns 
├── listing_events (drop start_date)
├── paper_applications (drop created_at, updated_at)
└── household_member (drop created_at, updated_at)

Lookup Tables → Map to Target Tables
├── accessibility → applications
├── alternate_contact → applications
├── demographics → applications (extract ethnicity first)
├── jurisdictions → listings
├── reserved_community_types → listings
├── unit_accessibility_priority_types → units
├── unit_rent_types → units
└── unit_types → units

Refactored Target Tables
├── units (drop redundant columns & foreign keys)
├── multiselect_questions (drop jurisdiction_id, application_section)
├── applications (merge programs→preferences, drop mapped foreign keys)
└── listings (drop jurisdiction_id, reserved_community_type_id, assets)

New Tables
└── application_multiselect_questions (explode preferences from applications)

Drop (retain in S3 and doorway_staging only)
├── user_accounts
└── user_roles
```

## Transformation Summary Table

<div align="center">

| Table Name | Type | Key Transformations | In Prod? |
|------------|-------|--------------|----------|
| **address** | Copy | None | Yes |
| **applicant** | Copy | None | Yes |
| **application_methods** | Copy | None | Yes |
| **assets** | Copy | None | Yes |
| **listing_images** | Copy | None | Yes |
| **listing_multiselect_questions** | Copy | None | Yes |
| **listing_events** | Column Removal | Drop `start_date` (redundant with `start_time`) | Yes |
| **paper_applications** | Column Removal | Drop `created_at`, `updated_at` (duplicates from application_methods) | Yes |
| **household_member** | Column Removal | Drop `created_at`, `updated_at` (duplicates from application_methods) | Yes |
| **accessibility** | Mapping | Map `mobility`, `vision`, `hearing`, `other` → applications (prefix: `accessibility_`) | No |
| **alternate_contact** | Mapping | Map `type`, `other_type`, `agency`, `mailing_address_id` → applications (prefix: `alternate_contact_`) | No |
| **demographics** | Mapping | Extract ethnicity from race; map `race`, `ethnicity`, `gender`, `sexual_orientation`, `how_did_you_hear`, `spoken_language` → applications | No |
| **jurisdictions** | Mapping | Map `name` → listings as `jurisdiction`; document lookup info | No |
| **reserved_community_types** | Mapping | Map `name` → listings as `reserved_community_type`; document lookup info | No |
| **unit_accessibility_priority_types** | Mapping | Map `name` → units on `priority_type_id` | No |
| **unit_rent_types** | Mapping | Map `name` → units on `rent_type_id` | No |
| **unit_types** | Mapping | Map `name`, `num_bedrooms` → units on `unit_type_id` | No |
| **units** | Refactored | Drop `bmr_program_chart`, `ami_chart_id`, `ami_chart_override_id`, `status`; drop `unit_type_id`, `unit_rent_type_id`, `priority_type_id` after mapping | Yes |
| **multiselect_questions** | Refactored | Drop `jurisdiction_id`, `application_section` | Yes |
| **applications** | Refactored | Merge `programs` → `preferences`; drop `preferences` (moved to new table); drop `accessibility_id`, `alternate_contact_id`, `demographics_id`, `user_id` | Yes |
| **listings** | Refactored | Drop `jurisdiction_id`, `reserved_community_type_id`, `assets`, `result_id`, `features_id`, `utilities_id`, `neighborhood_amenities_id`, `documents_id`, `copy_of_id` | Yes |
| **application_multiselect_questions** | Create New | Explode `preferences` from applications into rows with `application_id`, `multiselect_question_id`, `preferences` | Yes |
| **user_accounts** | Drop | No transformation; informational value minimal | No |
| **user_roles** | Drop | No transformation; informational value minimal | No |

</div>

## Detailed Table Transformations

### Direct Copy

These tables require no transformation and are moved directly to staging:

- **address**: Already clean
- **applicant**: Already clean  
- **application_methods**: Already clean
- **assets**: Already clean
- **listing_images**: Already clean
- **listing_multiselect_questions**: Already clean

<br>

### Remove Unnecessary/Redundant Columns 

#### 'listing_events'


**Transformation:** Drop `start_date`

**Rationale:** The `start_time` column already provides this information with greater precision.

---

#### 'paper_applications'

**Transformation:** Drop `created_at`, `updated_at`

**Rationale:** These timestamps duplicate those in 'application_methods', which references this table. Removing them reduces redundancy.

---

#### 'household_member'

**Transformation:** Drop `created_at`, `updated_at`

**Rationale:** These timestamps duplicate those in 'application_methods'. Removing them reduces redundancy.

<br>

### Lookup Tables → Map to Target Tables

These tables are lookup/dictionary tables that will be mapped into their parent tables and then dropped:

#### 'accessibility'

**Transformation:** Map `mobility`, `vision`, `hearing`, `other` → applications (prefix: `accessibility_`)

**Rationale:** Only four boolean columns provide value, and each accessibility record maps 1:1 to an application record. Inlining eliminates unnecessary joins.

---

#### 'alternate_contact'

**Transformation:** Map `type`, `other_type`, `agency`, `mailing_address_id` → applications (prefix: `alternate_contact_`)

**Rationale:** Each alternate_contact has a unique `id` and maps 1:1 to applications. No reason to maintain as separate table.

---

#### 'demographics'

**Transformation:** 
1. Extract `ethnicity` from `race` column (currently merged)
2. Map `race`, `ethnicity`, `gender`, `sexual_orientation`, `how_did_you_hear`, `spoken_language` → (prefix: `demographics`)

**Rationale:** Each demographics entry is unique and maps 1:1 to an application. The `id`, `created_at`, and `updated_at` columns are redundant. Inlining simplifies queries.

---

#### 'jurisdictions'

**Transformation:** Map `name` → listings as `jurisdiction`; document remaining lookup information

**Rationale:** This is a 9-row lookup table where all information except `name` is identical across records. Condense to documentation; keep only jurisdiction name for aggregation.

---

#### 'reserved_community_types'

**Transformation:** Map `name` → listings as `reserved_community_type`; document lookup information

**Rationale:** 1:1 mapping as categorical feature with 10 possible classes. Eliminate table to reduce complexity.

---

#### 'unit_accessibility_priority_types'

**Transformation:** Map `name` → units on `priority_type_id`

**Rationale:** 1:1 mapping as categorical feature with 7 possible classes. Inlining removes unnecessary join.

---

#### 'unit_rent_types'

**Transformation:** Map `name` → units on `rent_type_id`

**Rationale:** 1:1 mapping as categorical feature with 2 possible classes. Inlining simplifies schema.

---

#### 'unit_types'

**Transformation:** Map `name`, `num_bedrooms` → units on `unit_type_id`

**Rationale:** 1:1 mapping as categorical feature with 7 possible classes. Inlining reduces join complexity.

<br>

### Refactored Target Tables

These transformations must occur **after** Stage 3 mappings are complete:

#### 'units'

**Transformation:** 
1. Drop `bmr_program_chart`, `ami_chart_id`, `ami_chart_override_id`, `status` (redundant/no analytical value)
2. Drop `unit_type_id`, `unit_rent_type_id`, `priority_type_id` (foreign keys no longer needed after mapping)

**Rationale:** Clean up redundant columns and foreign keys that referenced now-inlined lookup tables.

---

#### 'multiselect_questions'

**Transformation:** Drop `jurisdiction_id`, `application_section`

**Rationale:** 
- `jurisdiction_id` is unnecessary since 'listing_multiselect_questions' already provides jurisdiction context
- `application_section` references `programs` and `preferences` columns that will be merged into one

---

#### 'applications'

**Transformation:** 
1. Merge `programs` column into `preferences` column
2. Drop `preferences` (will be moved to new 'application_multiselect_questions' table)
3. Drop `accessibility_id`, `alternate_contact_id`, `demographics_id`, `user_id` (foreign keys to inlined/dropped tables)

**Rationale:** Consolidate preference data and remove foreign keys to tables that no longer exist in staging or were inlined.

---

#### 'listings'

**Transformation:** Drop `jurisdiction_id`, `reserved_community_type_id`, `assets`, `result_id`, `features_id`, `utilities_id`, `neighborhood_amenities_id`, `documents_id`, `copy_of_id`

**Rationale:** 
- `jurisdiction_id` replaced by `jurisdiction` column
- `reserved_community_type_id` replaced by `reserved_community_type` column
- `assets` provides no analytical value
- `result_id` contains all null values
- `features_id`, `utilities_id`, `neighborhood_amenities_id`, `documents_id` link to Detroit-only tables
- `copy_of_id` provides no analytical value

<br>

### New Tables

#### 'application_multiselect_questions'

**Purpose:** Link applications to their multiselect question responses.

**Transformation:** 
1. Extract `preferences` column from 'applications'
2. Explode the list into rows: one row per question per application
3. Include `application_id`, `multiselect_question_id`, and `preferences` object

**Rationale:** Enables easy querying of which applicants answered specific questions while keeping the full preference data structure intact for detailed analysis.

**Schema:**

<div align="center">

| Column Name              | Data Type      | Description                                         |
|--------------------------|----------------|-----------------------------------------------------|
| application_id           | string         | Foreign key to application                          |
| multiselect_question_id  | string         | Foreign key to multiselect question                 |
| preferences              | object         | Object containing selected preferences/responses    |

</div>

<br>

### Not Staged (Retained in S3 /raw only)

#### 'user_accounts' & 'user_roles'

**Rationale:** Minimal analytical value. The only potentially informative column in 'user_accounts' is `mfa_enabled`, which is marginally useful at best. Both tables will remain in the S3 datalake for reference but won't be brought into Redshift for analysis.
