# Stage I (Extract): RDS (doorway-prod) to S3 (doorway-datalake)

The first stage of the pipeline can be broken down as follows:
1. Extract and redact PII data into S3 for raw storage
2. Dump non-PII data into S3 for raw storage
3. Partition data in S3 by table name
4. Sub-partition data within each table by date the Glue job is run

**Full Relevant set of tables in RDS:**

```python
all_tables = [
    "accessibility",
    "activity_log",
    "address",
    "alternate_contact",
    "ami_chart",
    "applicant",
    "application_flagged_set",
    "application_methods",
    "applications",
    "assets",
    "cron_job",
    "demographics",
    "generated_listing_translations",
    "household_member",
    "jurisdictions",
    "listing_events",
    "listing_features",
    "listing_images",
    "listing_multiselect_questions",
    "listing_neighborhood_amenities",
    "listing_utilities",
    "listings",
    "map_layers",
    "migrations",
    "multiselect_questions",
    "paper_applications",
    "reserved_community_types",
    "translations",
    "unit_accessibility_priority_types",
    "unit_ami_chart_overrides",
    "unit_group",
    "unit_group_ami_levels",
    "unit_rent_types",
    "unit_types",
    "units",
    "units_summary",
    "user_preferences",
    "user_accounts",
    "user_roles",
]
]
```

**Filtered Set:**

```python
pii_tables = [
    "address",
    "alternate_contact",
    "applicant",
    "applications",
    "household_member",
    "user_accounts",
    "application_methods",
]

non_pii_tables = [
    "accessibility",
    "assets",
    "demographics",
    "listing_events",
    "listing_images",
    "listings",
    "paper_applications",
    "units",
]

dictionary_tables = [
    "reserved_community_types",
    "unit_accessibility_priority_types",
    "unit_rent_types",
    "unit_types",
    "jurisdictions",
    "multiselect_questions",
    "listing_multiselect_questions",
    "user_roles",
]
```

These filtered tables are those which we will include in our pipeline as they promise the most value for analysis and publication.

---

## PII Tables Breakdown

### 'address'

This table contains all the addresses in the Doorway universe (applicant, listing, mailing, etc).

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name      | Data Type      | Description                       |
|------------------|----------------|-----------------------------------|
| id               | string         | Unique identifier for the address |
| created_at       | timestamp      | Record creation timestamp         |
| updated_at       | timestamp      | Record update timestamp           |
| place_name       | string         | Place name                        |
| city             | string         | City name                         |
| county           | string         | County name                       |
| state            | string         | State or province                 |
| street           | string         | Street address                    |
| street2          | string         | Secondary street address          |
| zip_code         | string         | Postal/ZIP code                   |
| latitude         | decimal(38,18) | Latitude coordinate               |
| longitude        | decimal(38,18) | Longitude coordinate              |

</div>

**Transformation:**

To keep PII out of S3, stage I of our glue pipeline will collect all of the addresses from each PII source:

- alternate_contact
    - mailing_address_id
- applicant
    - address_id
    - work_address_id
- applications
    - mailing_address
    - alternate_address
- household_member
    - address_id
    - work_address_id

Using this collection of address ID's, we will redact the `street` and `street2` entries from the 'address' table. This will leave the 'address' table intact, serving as the single source of truth for all relevant addresses, but redacted of PII. 

We will also be dropping the `latitude` and `longitude` columns entirely.

*Note*: This table will still contain any specific listing street address information. This is not PII, and can be used for mapping or other aggregation analysis purposes.

**S3 (doorway-datalake) Schema:**

<div align="center">

| Column Name      | Data Type      | Description                       |
|------------------|----------------|-----------------------------------|
| id               | string         | Unique identifier for the address |
| created_at       | timestamp      | Record creation timestamp         |
| updated_at       | timestamp      | Record update timestamp           |
| place_name       | string         | Place name                        |
| city             | string         | City name                         |
| county           | string         | County name                       |
| state            | string         | State or province                 |
| street           | string         | Street address                    |
| street2          | string         | Secondary street address          |
| zip_code         | string         | Postal/ZIP code                   |

</div>
---

### **'alternate_contact'**

This table contains contact information for individuals designated as alternate contacts by applicants. These may include family members, case workers, or other representatives who can be reached regarding an application.

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name        | Data Type | Description                              |
|--------------------|-----------|------------------------------------------|
| id                 | string    | Unique identifier for alternate contact  |
| created_at         | timestamp | Record creation timestamp                |
| updated_at         | timestamp | Record update timestamp                  |
| type               | string    | Type of alternate contact                |
| other_type         | string    | Other type specification                 |
| first_name         | string    | First name                               |
| last_name          | string    | Last name                                |
| agency             | string    | Agency name                              |
| phone_number       | string    | Phone number                             |
| email_address      | string    | Email address                            |
| mailing_address_id | string    | Foreign key to address table             |

</div>

**Transformation:**

We will drop the following PII: `first_name`, `last_name`, `phone_number`, `email_address`.

**S3 (doorway-datalake) Schema:**

<div align="center">

| Column Name        | Data Type | Description                              |
|--------------------|-----------|------------------------------------------|
| id                 | string    | Unique identifier for alternate contact  |
| created_at         | timestamp | Record creation timestamp                |
| updated_at         | timestamp | Record update timestamp                  |
| type               | string    | Type of alternate contact                |
| other_type         | string    | Other type specification                 |
| agency             | string    | Agency name                              |
| mailing_address_id | string    | Foreign key to address table             |

</div>

---

### **'applicant'**

This table includes information about the specific applicant. This is a distinct profile from a user, or an application. The applicant describes an individual in an application, whereas a user defines a Doorway account.

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name       | Data Type | Description                           |
|-------------------|-----------|---------------------------------------|
| id                | string    | Unique identifier for applicant       |
| created_at        | timestamp | Record creation timestamp             |
| updated_at        | timestamp | Record update timestamp               |
| first_name        | string    | First name                            |
| middle_name       | string    | Middle name                           |
| last_name         | string    | Last name                             |
| email_address     | string    | Email address                         |
| no_email          | boolean   | Flag indicating no email available    |
| phone_number      | string    | Phone number                          |
| phone_number_type | string    | Type of phone number                  |
| no_phone          | boolean   | Flag indicating no phone available    |
| work_address_id   | string    | Foreign key to work address           |
| address_id        | string    | Foreign key to residential address    |
| work_in_region    | string    | Indicates if work is in region        |
| birth_year        | int       | Year of birth                         |
| birth_month       | int       | Month of birth                        |
| birth_day         | int       | Day of birth                          |
| full_time_student | string    | Full-time student status              |

</div>

**Transformation:**

We will drop the following PII: `first_name`, `middle_name`, `last_name`, `email_address`, `phone_number`.

We will drop the columns `birth_month`, `birth_day`, leaving `birth_year` column to balance anonymity and analytical integrity.

**S3 (doorway-datalake) Schema:**

<div align="center">

| Column Name       | Data Type | Description                           |
|-------------------|-----------|---------------------------------------|
| id                | string    | Unique identifier for applicant       |
| created_at        | timestamp | Record creation timestamp             |
| updated_at        | timestamp | Record update timestamp               |
| no_email          | boolean   | Flag indicating no email available    |
| phone_number_type | string    | Type of phone number                  |
| no_phone          | boolean   | Flag indicating no phone available    |
| work_address_id   | string    | Foreign key to work address           |
| address_id        | string    | Foreign key to residential address    |
| work_in_region    | string    | Indicates if work is in region        |
| birth_year        | int       | Year of birth                         |
| full_time_student | string    | Full-time student status              |

</div>

---

### 'applications'

This table stores all submitted rental applications, linking applicants to listings and capturing relevant application details, preferences, and status. It includes references to related entities such as users, listings, addresses, and contacts, as well as metadata about the application process.

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name                      | Data Type      | Description                                  |
|----------------------------------|----------------|----------------------------------------------|
| id                               | string         | Unique identifier for application            |
| created_at                       | timestamp      | Record creation timestamp                    |
| updated_at                       | timestamp      | Record update timestamp                      |
| deleted_at                       | timestamp      | Record deletion timestamp                    |
| app_url                          | string         | Application URL                              |
| additional_phone                 | boolean        | Flag for additional phone                    |
| additional_phone_number          | string         | Additional phone number                      |
| additional_phone_number_type     | string         | Type of additional phone number              |
| contact_preferences              | array<string>  | Contact preference options                   |
| household_size                   | int            | Number of household members                  |
| housing_status                   | string         | Current housing status                       |
| send_mail_to_mailing_address     | boolean        | Mail delivery preference flag                |
| household_expecting_changes      | boolean        | Flag for expected household changes          |
| household_student                | boolean        | Flag for student in household                |
| income                           | string         | Income information                           |
| preferences                      | string         | Application preferences                      |
| programs                         | string         | Programs applied for                         |
| accepted_terms                   | boolean        | Terms acceptance flag                        |
| submission_date                  | timestamp      | Date application was submitted               |
| marked_as_duplicate              | boolean        | Duplicate application flag                   |
| confirmation_code                | string         | Application confirmation code                |
| user_id                          | string         | Foreign key to user                          |
| listing_id                       | string         | Foreign key to listing                       |
| applicant_id                     | string         | Foreign key to applicant                     |
| mailing_address_id               | string         | Foreign key to mailing address               |
| alternate_address_id             | string         | Foreign key to alternate address             |
| alternate_contact_id             | string         | Foreign key to alternate contact             |
| accessibility_id                 | string         | Foreign key to accessibility                 |
| demographics_id                  | string         | Foreign key to demographics                  |
| income_vouchers                  | array<string>  | Income voucher types                         |
| income_period                    | string         | Income reporting period                      |
| status                           | string         | Application status                           |
| language                         | string         | Application language                         |
| submission_type                  | string         | Type of submission                           |
| review_status                    | string         | Application review status                    |
| received_at                      | timestamp      | Date application was received                |
| received_by                      | string         | Person who received application              |
| was_created_externally           | boolean        | Flag for external creation                   |
| expire_after                     | timestamp      | Application expiration date                  |
| is_newest                        | boolean        | Flag for newest version                      |
| was_pii_cleared                  | boolean        | Flag indicating PII has been cleared         |

</div>

**Transformation:**

We will drop the following PII: `additional_phone_number`.

We will drop the following unneeded columns: `was_pii_cleared`, `is_newest`, `expire_after`, `confirmation_code`.

**S3 (doorway-datalake) Schema:**

<div align="center">

| Column Name                  | Data Type      | Description                                  |
|------------------------------|----------------|----------------------------------------------|
| id                           | string         | Unique identifier for application            |
| created_at                   | timestamp      | Record creation timestamp                    |
| updated_at                   | timestamp      | Record update timestamp                      |
| deleted_at                   | timestamp      | Record deletion timestamp                    |
| app_url                      | string         | Application URL                              |
| additional_phone             | boolean        | Flag for additional phone                    |
| additional_phone_number_type | string         | Type of additional phone number              |
| contact_preferences          | array<string>  | Contact preference options                   |
| household_size               | int            | Number of household members                  |
| housing_status               | string         | Current housing status                       |
| send_mail_to_mailing_address | boolean        | Mail delivery preference flag                |
| household_expecting_changes  | boolean        | Flag for expected household changes          |
| household_student            | boolean        | Flag for student in household                |
| income                       | string         | Income information                           |
| preferences                  | string         | Application preferences                      |
| programs                     | string         | Programs applied for                         |
| accepted_terms               | boolean        | Terms acceptance flag                        |
| submission_date              | timestamp      | Date application was submitted               |
| marked_as_duplicate          | boolean        | Duplicate application flag                   |
| user_id                      | string         | Foreign key to user                          |
| listing_id                   | string         | Foreign key to listing                       |
| applicant_id                 | string         | Foreign key to applicant                     |
| mailing_address_id           | string         | Foreign key to mailing address               |
| alternate_address_id         | string         | Foreign key to alternate address             |
| alternate_contact_id         | string         | Foreign key to alternate contact             |
| demographics_id              | string         | Foreign key to demographics                  |
| income_vouchers              | array<string>  | Income voucher types                         |
| income_period                | string         | Income reporting period                      |
| status                       | string         | Application status                           |
| language                     | string         | Application language                         |
| submission_type              | string         | Type of submission                           |
| review_status                | string         | Application review status                    |
| received_at                  | timestamp      | Date application was received                |
| received_by                  | string         | Person who received application              |
| was_created_externally       | boolean        | Flag for external creation                   |

</div>

---

### 'household_member'

This table stores information about each member of an applicant's household, including their relationship to the applicant, address references, birth date, and student status. It can be linked to the applicant through the `application_id`.

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name       | Data Type | Description                              |
|-------------------|-----------|------------------------------------------|
| id                | string    | Unique identifier for household member   |
| created_at        | timestamp | Record creation timestamp                |
| updated_at        | timestamp | Record update timestamp                  |
| order_id          | int       | Order of household member                |
| first_name        | string    | First name                               |
| middle_name       | string    | Middle name                              |
| last_name         | string    | Last name                                |
| relationship      | string    | Relationship to applicant                |
| address_id        | string    | Foreign key to residential address       |
| work_address_id   | string    | Foreign key to work address              |
| application_id    | string    | Foreign key to application               |
| same_address      | string    | Flag for same address as applicant       |
| work_in_region    | string    | Indicates if work is in region           |
| birth_year        | int       | Year of birth                            |
| birth_month       | int       | Month of birth                           |
| birth_day         | int       | Day of birth                             |
| full_time_student | string    | Full-time student status                 |

</div>

**Transformation:**

We will drop the following PII: `first_name`, `middle_name`, `last_name`.

We will drop the columns `birth_month`, `birth_day`, leaving `birth_year` column to balance anonymity and analytical integrity.

**S3 (doorway-datalake) Schema:**

<div align="center">

| Column Name       | Data Type | Description                              |
|-------------------|-----------|------------------------------------------|
| id                | string    | Unique identifier for household member   |
| created_at        | timestamp | Record creation timestamp                |
| updated_at        | timestamp | Record update timestamp                  |
| order_id          | int       | Order of household member                |
| relationship      | string    | Relationship to applicant                |
| address_id        | string    | Foreign key to residential address       |
| work_address_id   | string    | Foreign key to work address              |
| application_id    | string    | Foreign key to application               |
| same_address      | string    | Flag for same address as applicant       |
| work_in_region    | string    | Indicates if work is in region           |
| birth_year        | int       | Year of birth                            |
| full_time_student | string    | Full-time student status                 |

</div>

---

### 'user_accounts'

This table contains user account information for individuals registered on the Doorway platform. It includes authentication details, contact information, account status, and metadata related to user activity and preferences.

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name                     | Data Type | Description                                |
|---------------------------------|-----------|--------------------------------------------|
| id                              | string    | Unique identifier for user account         |
| password_hash                   | string    | Hashed password                            |
| password_updated_at             | timestamp | Password last update timestamp             |
| password_valid_for_days         | int       | Password validity duration in days         |
| reset_token                     | string    | Password reset token                       |
| confirmation_token              | string    | Account confirmation token                 |
| confirmed_at                    | timestamp | Account confirmation timestamp             |
| email                           | string    | Email address                              |
| first_name                      | string    | First name                                 |
| middle_name                     | string    | Middle name                                |
| last_name                       | string    | Last name                                  |
| dob                             | timestamp | Date of birth                              |
| phone_number                    | string    | Phone number                               |
| created_at                      | timestamp | Record creation timestamp                  |
| updated_at                      | timestamp | Record update timestamp                    |
| mfa_enabled                     | boolean   | Multi-factor authentication enabled flag   |
| single_use_code                 | string    | Single use code for authentication         |
| single_use_code_updated_at      | timestamp | Single use code update timestamp           |
| last_login_at                   | timestamp | Last login timestamp                       |
| failed_login_attempts_count     | int       | Count of failed login attempts             |
| phone_number_verified           | boolean   | Phone number verification flag             |
| agreed_to_terms_of_service      | boolean   | Terms of service agreement flag            |
| hit_confirmation_url            | timestamp | Confirmation URL access timestamp          |
| active_access_token             | string    | Active access token                        |
| active_refresh_token            | string    | Active refresh token                       |
| language                        | string    | User language preference                   |
| was_warned_of_deletion          | boolean   | Account deletion warning flag              |

</div>

**Transformation:**

We will drop the following PII: `password_hash`, `reset_token`, `confirmation_token`, `email_address`, `first_name`, `middle_name`, `last_name`, `dob`, `phone_number`, `single_use_code`, `single_use_code_updated_at`, `active_access_token`, `active_refresh_token`.

We will drop the following unneeded columns: `password_updated_at`, `password_valid_for_days`, `confirmed_at`, `hit_confirmation_url`, `phone_number_verified`, `last_login_at`, `failed_login_attempts_count`, `agreed_to_terms_of_service`, `language`, `was_warned_of_deletion`.

**S3 (doorway-datalake) Schema:**

<div align="center">

| Column Name                | Data Type | Description                                |
|----------------------------|-----------|--------------------------------------------|
| id                         | string    | Unique identifier for user account         |
| created_at                 | timestamp | Record creation timestamp                  |
| updated_at                 | timestamp | Record update timestamp                    |
| mfa_enabled                | boolean   | Multi-factor authentication enabled flag   |

</div>

---

### 'application_methods'

... TODO

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name                      | Data Type | Description                                |
|----------------------------------|-----------|--------------------------------------------|
| id                               | string    | Unique identifier for application method   |
| created_at                       | timestamp | Record creation timestamp                  |
| updated_at                       | timestamp | Record update timestamp                    |
| type                             | string    | Type of application method                 |
| label                            | string    | Label for application method               |
| external_reference               | string    | External reference identifier              |
| accepts_postmarked_applications  | boolean   | Flag for accepting postmarked applications |
| phone_number                     | string    | Phone number for applications              |
| listing_id                       | string    | Foreign key to listing                     |

</div>

**Transformation:**

We will drop the following PII: `phone_number`.

**S3 (doorway-datalake) Schema:**

<div align="center">

| Column Name                      | Data Type | Description                                |
|----------------------------------|-----------|--------------------------------------------|
| id                               | string    | Unique identifier for application method   |
| created_at                       | timestamp | Record creation timestamp                  |
| updated_at                       | timestamp | Record update timestamp                    |
| type                             | string    | Type of application method                 |
| label                            | string    | Label for application method               |
| external_reference               | string    | External reference identifier              |
| accepts_postmarked_applications  | boolean   | Flag for accepting postmarked applications |
| listing_id                       | string    | Foreign key to listing                     |

</div>

<br>

## Non-PII Tables Breakdown

### 'accessibility'

This table records accessibility needs associated with applications, such as mobility, vision, hearing, and other requirements.

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name | Data Type | Description                          |
|-------------|-----------|--------------------------------------|
| id          | string    | Unique identifier for accessibility  |
| created_at  | timestamp | Record creation timestamp            |
| updated_at  | timestamp | Record update timestamp              |
| mobility    | boolean   | Mobility accessibility need          |
| vision      | boolean   | Vision accessibility need            |
| hearing     | boolean   | Hearing accessibility need           |
| other       | boolean   | Other accessibility need             |

</div>

**Transformation:**

No transformation at this stage. Just bring it into /raw.

**S3 (doorway-datalake) Schema:**

No change in schema at this stage.

---


### 'assets'

This table stores metadata about digital assets associated with listings, such as images, documents, or other files. 

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name | Data Type | Description                      |
|-------------|-----------|----------------------------------|
| id          | string    | Unique identifier for asset      |
| created_at  | timestamp | Record creation timestamp        |
| updated_at  | timestamp | Record update timestamp          |
| file_id     | string    | File identifier                  |
| label       | string    | Asset label                      |

</div>

**Transformation:**

No transformation at this stage. Just bring it into /raw.

**S3 (doorway-datalake) Schema:**

No change in schema at this stage.

---

### 'demographics'

This table captures demographic information provided by applicants, including ethnicity, gender, sexual orientation, race, spoken language, and how they heard about Doorway.

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name        | Data Type      | Description                          |
|--------------------|----------------|--------------------------------------|
| id                 | string         | Unique identifier for demographics   |
| created_at         | timestamp      | Record creation timestamp            |
| updated_at         | timestamp      | Record update timestamp              |
| ethnicity          | string         | Ethnicity information                |
| gender             | string         | Gender information                   |
| sexual_orientation | string         | Sexual orientation information       |
| how_did_you_hear   | array<string>  | Sources of information               |
| race               | array<string>  | Race information                     |
| spoken_language    | string         | Spoken language preference           |

</div>

**Transformation:**

No transformation at this stage. Just bring it into /raw.

**S3 (doorway-datalake) Schema:**

No change in schema at this stage.

---

### 'listing_events'

This table tracks events related to listings, such as open houses, application deadlines, or informational sessions. 

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name | Data Type | Description                          |
|-------------|-----------|--------------------------------------|
| id          | string    | Unique identifier for listing event  |
| created_at  | timestamp | Record creation timestamp            |
| updated_at  | timestamp | Record update timestamp              |
| type        | string    | Type of listing event                |
| start_date  | timestamp | Event start date                     |
| start_time  | timestamp | Event start time                     |
| end_time    | timestamp | Event end time                       |
| url         | string    | Event URL                            |
| note        | string    | Event notes                          |
| label       | string    | Event label                          |
| listing_id  | string    | Foreign key to listing               |
| file_id     | string    | File identifier                      |

</div>

**Transformation:**

No transformation at this stage. Just bring it into /raw.

**S3 (doorway-datalake) Schema:**

No change in schema at this stage.

---

### 'listing_images'

This table stores the images associated with each listing, including their order and references to both the listing and the image file.

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name | Data Type | Description                    |
|-------------|-----------|--------------------------------|
| ordinal     | int       | Image order/position           |
| listing_id  | string    | Foreign key to listing         |
| image_id    | string    | Image identifier               |

</div>

**Transformation:**

No transformation at this stage. Just bring it into /raw.

**S3 (doorway-datalake) Schema:**

No change in schema at this stage.

---

### 'listings'

**RDS (doorway-prod) Schema:**

This table contains information about each housing listing available on the Doorway platform. It includes attributes such as application methods, amenities, building details, eligibility criteria, contact information, and status. Listings are linked to related entities like addresses, assets, and jurisdictions.

<div align="center">

| Column Name                                      | Data Type       | Description                                    |
|--------------------------------------------------|-----------------|------------------------------------------------|
| id                                               | string          | Unique identifier for listing                  |
| created_at                                       | timestamp       | Record creation timestamp                      |
| updated_at                                       | timestamp       | Record update timestamp                        |
| additional_application_submission_notes          | string          | Additional submission notes                    |
| digital_application                              | boolean         | Digital application available flag             |
| common_digital_application                       | boolean         | Common digital application flag                |
| paper_application                                | boolean         | Paper application available flag               |
| referral_opportunity                             | boolean         | Referral opportunity flag                      |
| assets                                           | string          | Listing assets                                 |
| accessibility                                    | string          | Accessibility features                         |
| amenities                                        | string          | Building amenities                             |
| building_total_units                             | int             | Total units in building                        |
| developer                                        | string          | Developer name                                 |
| household_size_max                               | int             | Maximum household size                         |
| household_size_min                               | int             | Minimum household size                         |
| neighborhood                                     | string          | Neighborhood name                              |
| pet_policy                                       | string          | Pet policy details                             |
| smoking_policy                                   | string          | Smoking policy details                         |
| units_available                                  | int             | Number of available units                      |
| unit_amenities                                   | string          | Unit amenities                                 |
| services_offered                                 | string          | Services offered                               |
| year_built                                       | int             | Year building was built                        |
| application_due_date                             | timestamp       | Application due date                           |
| application_open_date                            | timestamp       | Application open date                          |
| application_fee                                  | string          | Application fee amount                         |
| application_organization                         | string          | Application organization                       |
| application_pick_up_address_office_hours         | string          | Pick up address office hours                   |
| application_drop_off_address_office_hours        | string          | Drop off address office hours                  |
| building_selection_criteria                      | string          | Building selection criteria                    |
| costs_not_included                               | string          | Costs not included                             |
| credit_history                                   | string          | Credit history requirements                    |
| criminal_background                              | string          | Criminal background requirements               |
| deposit_min                                      | string          | Minimum deposit                                |
| deposit_max                                      | string          | Maximum deposit                                |
| deposit_helper_text                              | string          | Deposit helper text                            |
| disable_units_accordion                          | boolean         | Disable units accordion flag                   |
| leasing_agent_email                              | string          | Leasing agent email                            |
| leasing_agent_name                               | string          | Leasing agent name                             |
| leasing_agent_office_hours                       | string          | Leasing agent office hours                     |
| leasing_agent_phone                              | string          | Leasing agent phone                            |
| leasing_agent_title                              | string          | Leasing agent title                            |
| name                                             | string          | Listing name                                   |
| postmarked_applications_received_by_date         | timestamp       | Postmarked applications deadline               |
| program_rules                                    | string          | Program rules                                  |
| rental_assistance                                | string          | Rental assistance information                  |
| rental_history                                   | string          | Rental history requirements                    |
| required_documents                               | string          | Required documents                             |
| special_notes                                    | string          | Special notes                                  |
| waitlist_current_size                            | int             | Current waitlist size                          |
| waitlist_max_size                                | int             | Maximum waitlist size                          |
| what_to_expect                                   | string          | What to expect information                     |
| status                                           | string          | Listing status                                 |
| review_order_type                                | string          | Review order type                              |
| display_waitlist_size                            | boolean         | Display waitlist size flag                     |
| reserved_community_description                   | string          | Reserved community description                 |
| reserved_community_min_age                       | int             | Reserved community minimum age                 |
| result_link                                      | string          | Result link                                    |
| is_waitlist_open                                 | boolean         | Waitlist open flag                             |
| waitlist_open_spots                              | int             | Waitlist open spots                            |
| custom_map_pin                                   | boolean         | Custom map pin flag                            |
| published_at                                     | timestamp       | Publication timestamp                          |
| closed_at                                        | timestamp       | Closing timestamp                              |
| afs_last_run_at                                  | timestamp       | AFS last run timestamp                         |
| last_application_update_at                       | timestamp       | Last application update timestamp              |
| building_address_id                              | string          | Foreign key to building address                |
| application_pick_up_address_id                   | string          | Foreign key to pick up address                 |
| application_drop_off_address_id                  | string          | Foreign key to drop off address                |
| application_mailing_address_id                   | string          | Foreign key to mailing address                 |
| building_selection_criteria_file_id              | string          | Foreign key to selection criteria file         |
| jurisdiction_id                                  | string          | Foreign key to jurisdiction                    |
| leasing_agent_address_id                         | string          | Foreign key to leasing agent address           |
| reserved_community_type_id                       | string          | Foreign key to reserved community type         |
| result_id                                        | string          | Foreign key to result                          |
| features_id                                      | string          | Foreign key to features                        |
| utilities_id                                     | string          | Foreign key to utilities                       |
| requested_changes                                | string          | Requested changes                              |
| requested_changes_date                           | timestamp       | Requested changes date                         |
| requested_changes_user_id                        | string          | Foreign key to user who requested changes      |
| ami_percentage_max                               | int             | Maximum AMI percentage                         |
| ami_percentage_min                               | int             | Minimum AMI percentage                         |
| home_type                                        | string          | Type of home                                   |
| hrd_id                                           | string          | HRD identifier                                 |
| is_verified                                      | boolean         | Verification flag                              |
| management_company                               | string          | Management company name                        |
| management_website                               | string          | Management website URL                         |
| marketing_year                                   | int             | Marketing year                                 |
| marketing_season                                 | string          | Marketing season                               |
| marketing_type                                   | string          | Marketing type                                 |
| neighborhood_amenities_id                        | string          | Foreign key to neighborhood amenities          |
| owner_company                                    | string          | Owner company name                             |
| phone_number                                     | string          | Contact phone number                           |
| region                                           | string          | Region                                         |
| section8_acceptance                              | boolean         | Section 8 acceptance flag                      |
| temporary_listing_id                             | int             | Temporary listing identifier                   |
| verified_at                                      | timestamp       | Verification timestamp                         |
| what_to_expect_additional_text                   | string          | Additional what to expect text                 |
| application_pick_up_address_type                 | string          | Pick up address type                           |
| application_drop_off_address_type                | string          | Drop off address type                          |
| application_mailing_address_type                 | string          | Mailing address type                           |
| content_updated_at                               | timestamp       | Content update timestamp                       |
| lottery_last_run_at                              | timestamp       | Lottery last run timestamp                     |
| lottery_status                                   | string          | Lottery status                                 |
| lottery_opt_in                                   | boolean         | Lottery opt-in flag                            |
| lottery_last_published_at                        | timestamp       | Lottery last published timestamp               |
| copy_of_id                                       | string          | Copy of ID                                     |
| community_disclaimer_description                 | string          | Community disclaimer description               |
| community_disclaimer_title                       | string          | Community disclaimer title                     |
| include_community_disclaimer                     | boolean         | Include community disclaimer flag              |
| was_created_externally                           | boolean         | External creation flag                         |
| last_updated_by_user_id                          | string          | Foreign key to last updating user              |
| coc_info                                         | string          | COC information                                |
| deposit_range_max                                | int             | Maximum deposit range                          |
| deposit_range_min                                | int             | Minimum deposit range                          |
| deposit_type                                     | string          | Deposit type                                   |
| deposit_value                                    | decimal(38,30)  | Deposit value                                  |
| has_hud_ebll_clearance                           | boolean         | HUD EBLL clearance flag                        |
| listing_type                                     | string          | Listing type                                   |

</div>

**Transformation:**

No transformation at this stage. Just bring it into /raw.

**S3 (doorway-datalake) Schema:**

No change in schema at this stage.

---

### 'paper_applications'

... TODO

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name             | Data Type | Description                              |
|-------------------------|-----------|------------------------------------------|
| id                      | string    | Unique identifier for paper application  |
| created_at              | timestamp | Record creation timestamp                |
| updated_at              | timestamp | Record update timestamp                  |
| file_id                 | string    | File identifier                          |
| application_method_id   | string    | Foreign key to application method        |
| language                | string    | Application language                     |

</div>

**Transformation:**

No transformation at this stage. Just bring it into /raw.

**S3 (doorway-datalake) Schema:**

No change in schema at this stage.

---

### 'units'

This table contains detailed information about individual housing units within a listing, including occupancy limits, rent amounts, income requirements, physical attributes (bedrooms, bathrooms, square footage), and references to related charts and types. Each unit is linked to its parent listing and categorized by type and rent structure.

**RDS (doorway-prod) Schema:**

<div align="center">

| Column Name                         | Data Type      | Description                              |
|-------------------------------------|----------------|------------------------------------------|
| id                                  | string         | Unique identifier for unit               |
| created_at                          | timestamp      | Record creation timestamp                |
| updated_at                          | timestamp      | Record update timestamp                  |
| ami_percentage                      | string         | AMI percentage                           |
| annual_income_min                   | string         | Minimum annual income                    |
| monthly_income_min                  | string         | Minimum monthly income                   |
| floor                               | int            | Floor number                             |
| annual_income_max                   | string         | Maximum annual income                    |
| max_occupancy                       | int            | Maximum occupancy                        |
| min_occupancy                       | int            | Minimum occupancy                        |
| monthly_rent                        | string         | Monthly rent amount                      |
| num_bathrooms                       | int            | Number of bathrooms                      |
| num_bedrooms                        | int            | Number of bedrooms                       |
| number                              | string         | Unit number                              |
| sq_feet                             | decimal(8,2)   | Square footage                           |
| monthly_rent_as_percent_of_income   | decimal(8,2)   | Monthly rent as percent of income        |
| bmr_program_chart                   | boolean        | BMR program chart flag                   |
| ami_chart_id                        | string         | Foreign key to AMI chart                 |
| listing_id                          | string         | Foreign key to listing                   |
| unit_type_id                        | string         | Foreign key to unit type                 |
| unit_rent_type_id                   | string         | Foreign key to unit rent type            |
| priority_type_id                    | string         | Foreign key to priority type             |
| ami_chart_override_id               | string         | Foreign key to AMI chart override        |
| status                              | string         | Unit status                              |

</div>

**Transformation:**

No transformation at this stage. Just bring it into /raw.

**S3 (doorway-datalake) Schema:**

No change in schema at this stage.
