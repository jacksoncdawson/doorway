# TODO
- Finish updating 'ingest' tables with RDS validation. Jobs updated on 2/25 are good to go. the rest need work.
- Look into Great Expectations
- Jack: take a deep look at documentation / refactor
- Consider ephemeral tables for temp


# Future Refactors Needed

## Admin Needed
- validation step between RDS and S3
- Use different user role/permission other thant 'jcdawson' in redshift queries (validate-upsert, upsert-S3_to_redshift)

## No Admin Needed
- Break apart Race/Ethnicity column
- Create 'application_multiselect_questions' 

## Possible Refactors
- Streamline redshift.staging 
