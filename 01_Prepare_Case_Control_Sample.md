# 01_Prepare_Case_Control_Sample
## This notebook retrieves EHR, demographic, and survey data for our OUD case-control samples from the All of Us database
### Load Packages 
```
library(bigrquery)
library(lubridate)
library(tidyverse)
Loading required package: timechange
```
### Create Functions
**Combine results from multiple queries**
```
formulate_and_run_multipart_query <- function(subqueries, final_tbl) {
    query <- str_c('WITH\n', str_c(subqueries, collapse = ',\n\n'), str_glue('\n\n\nSELECT * FROM {final_tbl}'))
    message(query)               
    results <- bq_table_download(bq_dataset_query(Sys.getenv('WORKSPACE_CDR'),
                                                  query,
                                                  billing = Sys.getenv('GOOGLE_PROJECT')),
                                 bigint = 'integer64')
    message(str_glue('Dimensions of result: num_rows={nrow(results)} num_cols={ncol(results)}'))
    return(results)
}
```
**Store file names for case and control cohorts**
```
DATESTAMP <- strftime(now(), '%Y%m%d')
DESTINATION <- str_glue('{Sys.getenv("WORKSPACE_BUCKET")}/data/aou/pheno/{DATESTAMP}/')

CASE_FILENAME <- 'case.csv'
CASE_ID_FILENAME <- 'case_ids.tsv'

CONTROL_FILENAME <- 'control.csv'
CONTROL_ID_FILENAME <- 'control_ids.tsv'

# files for testing purposes
TEST_CASE_FILENAME <- 'test_case.csv'
TEST_CASE_ID_FILENAME <- 'test_case_ids.tsv'

TEST_CONTROL_FILENAME <- 'test_control.csv'
TEST_CONTROL_ID_FILENAME <- 'test_control_ids.tsv'
```
**Query for demographics data, select on everyone on All of Us that has srWGS data**
```
DEMOGRAPHICS_QUERY <- str_glue('
-- This query represents dataset "Demographics for AoU WGS cohort" for domain "person" and was
-- generated for All of Us Controlled Tier Dataset v5 alpha and then further edited.
demographics_tbl AS (
    SELECT
        person.BIRTH_DATETIME as date_of_birth,
        person.PERSON_ID as person_id,
        p_race_concept.concept_name as race,
        p_gender_concept.concept_name as gender,
        p_ethnicity_concept.concept_name as ethnicity,
        p_sex_at_birth_concept.concept_name as sex_at_birth
    FROM
        `person` person
    LEFT JOIN `concept` p_race_concept on person.race_concept_id = p_race_concept.CONCEPT_ID
    LEFT JOIN `concept` p_gender_concept on person.gender_concept_id = p_gender_concept.CONCEPT_ID
    LEFT JOIN `concept` p_ethnicity_concept on person.ethnicity_concept_id = p_ethnicity_concept.CONCEPT_ID
    LEFT JOIN `concept` p_sex_at_birth_concept on person.sex_at_birth_concept_id = p_sex_at_birth_concept.CONCEPT_ID
    WHERE person.PERSON_ID IN (
    SELECT person_id from `cb_search_person` cb_search_person
    WHERE cb_search_person.person_id in (
        SELECT person_id from `cb_search_person` p
        WHERE has_whole_genome_variant = 1))
)')
```
```
demographics <- formulate_and_run_multipart_query(c(DEMOGRAPHICS_QUERY), 'demographics_tbl')
head(demographics)

WITH
-- This query represents dataset "Demographics for AoU WGS cohort" for domain "person" and was
-- generated for All of Us Controlled Tier Dataset v5 alpha and then further edited.
demographics_tbl AS (
    SELECT
        person.BIRTH_DATETIME as date_of_birth,
        person.PERSON_ID as person_id,
        p_race_concept.concept_name as race,
        p_gender_concept.concept_name as gender,
        p_ethnicity_concept.concept_name as ethnicity,
        p_sex_at_birth_concept.concept_name as sex_at_birth
    FROM
        `person` person
    LEFT JOIN `concept` p_race_concept on person.race_concept_id = p_race_concept.CONCEPT_ID
    LEFT JOIN `concept` p_gender_concept on person.gender_concept_id = p_gender_concept.CONCEPT_ID
    LEFT JOIN `concept` p_ethnicity_concept on person.ethnicity_concept_id = p_ethnicity_concept.CONCEPT_ID
    LEFT JOIN `concept` p_sex_at_birth_concept on person.sex_at_birth_concept_id = p_sex_at_birth_concept.CONCEPT_ID
    WHERE person.PERSON_ID IN (
    SELECT person_id from `cb_search_person` cb_search_person
    WHERE cb_search_person.person_id in (
        SELECT person_id from `cb_search_person` p
        WHERE has_whole_genome_variant = 1))
)

SELECT * FROM demographics_tbl
```
*Output*
```
Dimensions of result: num_rows=245388 num_cols=6

1936-06-15	3524098	I prefer not to answer	I prefer not to answer	PMI: Prefer Not To Answer	I prefer not to answer
2001-06-15	8307916	I prefer not to answer	I prefer not to answer	PMI: Prefer Not To Answer	I prefer not to answer
2001-06-15	1794805	Asian	Male	Not Hispanic or Latino	I prefer not to answer
2001-06-15	1419254	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept
2002-06-15	3326982	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept
2002-06-15	1468095	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept
```

**a. Query for case sample, select on people with ICD10 codes 438120, 438130, 440693; these are the ICD10 codes that characterize a stringent definition of OUD**

```
OPIOID_CONDITIONS_QUERY <- "
opioid_conditions_tbl AS (
    SELECT
        c_occurrence.person_id as person_id,
        c_occurrence.condition_concept_id as condition_concept_id,
        c_occurrence.condition_start_datetime as condition_start_datetime,
        c_occurrence.condition_end_datetime as condition_end_datetime,
        c_standard_concept.concept_name as standard_concept_name,
        c_standard_concept.concept_code as standard_concept_code
    FROM
        ( SELECT
            * 
        FROM
            `condition_occurrence` c_occurrence 
        WHERE
            (
                condition_concept_id IN  (
                    SELECT
                        DISTINCT c.concept_id 
                    FROM
                        `cb_criteria` c 
                    JOIN
                        (
                            select
                                cast(cr.id as string) as id 
                            FROM
                                `cb_criteria` cr 
                            WHERE
                                concept_id IN (438120, 438130, 440693) 
                                AND full_text LIKE '%_rank1]%'
                        ) a 
                            ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                            OR c.path LIKE CONCAT('%.', a.id) 
                            OR c.path LIKE CONCAT(a.id, '.%') 
                            OR c.path = a.id) 
                        WHERE
                            is_standard = 1 
                            AND is_selectable = 1
                        )
                )  
                AND (
                    c_occurrence.PERSON_ID IN (
                        SELECT
                            distinct person_id  
                        FROM
                            `cb_search_person` cb_search_person  
                        WHERE
                            cb_search_person.person_id IN (
                                SELECT
                                    person_id 
                                FROM
                                    `cb_search_person` p 
                                WHERE
                                    has_whole_genome_variant = 1 
                            ) 
                            AND cb_search_person.person_id IN (
                                SELECT
                                    criteria.person_id 
                                FROM
                                    (SELECT
                                        DISTINCT person_id,
                                        entry_date,
                                        concept_id 
                                    FROM
                                        `cb_search_all_events` 
                                    WHERE
                                        (
                                            concept_id IN (
                                                SELECT
                                                    DISTINCT c.concept_id 
                                                FROM
                                                    `cb_criteria` c 
                                                JOIN
                                                    (
                                                        select
                                                            cast(cr.id as string) as id 
                                                        FROM
                                                            `cb_criteria` cr 
                                                        WHERE
                                                            concept_id IN (438120, 438130, 440693) 
                                                            AND full_text LIKE '%_rank1]%'
                                                    ) a 
                                                        ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                                                        OR c.path LIKE CONCAT('%.', a.id) 
                                                        OR c.path LIKE CONCAT(a.id, '.%') 
                                                        OR c.path = a.id) 
                                                    WHERE
                                                        is_standard = 1 
                                                        AND is_selectable = 1
                                                    ) 
                                                    AND is_standard = 1 
                                            )
                                        ) criteria 
                                    ) 
                                    AND cb_search_person.person_id IN (
                                        SELECT
                                            criteria.person_id 
                                        FROM
                                            (SELECT
                                                DISTINCT person_id,
                                                entry_date,
                                                concept_id 
                                            FROM
                                                `cb_search_all_events` 
                                            WHERE
                                                (
                                                    concept_id IN (
                                                        SELECT
                                                            DISTINCT ca.descendant_id 
                                                        FROM
                                                            `cb_criteria_ancestor` ca 
                                                        JOIN
                                                            (
                                                                select
                                                                    distinct c.concept_id 
                                                                FROM
                                                                    `cb_criteria` c 
                                                                JOIN
                                                                    (
                                                                        select
                                                                            cast(cr.id as string) as id 
                                                                        FROM
                                                                            `cb_criteria` cr 
                                                                        WHERE
                                                                            concept_id IN (21604254) 
                                                                            AND full_text LIKE '%_rank1]%'
                                                                    ) a 
                                                                        ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                                                                        OR c.path LIKE CONCAT('%.', a.id) 
                                                                        OR c.path LIKE CONCAT(a.id, '.%') 
                                                                        OR c.path = a.id) 
                                                                    WHERE
                                                                        is_standard = 1 
                                                                        AND is_selectable = 1
                                                                    ) b 
                                                                        ON (
                                                                            ca.ancestor_id = b.concept_id
                                                                        )
                                                                ) 
                                                                AND is_standard = 1
                                                            )
                                                    ) criteria 
                                                ) 
                                            ))) c_occurrence 
                                LEFT JOIN
                                    `concept` c_standard_concept 
                                        ON c_occurrence.condition_concept_id = c_standard_concept.concept_id)"
```

**b. Query for filling all entries in start and end times so that we don't have null values. This is because some condition records do not have an end time. When that is the case, use the start time as the end time.**

```
OPIOID_CONDITIONS_WITH_END_TIME_FILLED_QUERY <- '
opioid_conditions_with_end_time_filled_tbl AS (
    SELECT
        person_id,
        condition_concept_id,
        condition_start_datetime,
        IFNULL(condition_end_datetime, condition_start_datetime) AS condition_end_datetime,
        standard_concept_code,
        standard_concept_name
    FROM
        opioid_conditions_tbl
)'
```

**Query for counting the condition occurrences for each person. This is because one person can be diagnosed with the same condition at different times. We also want to determine the outer bounds of the time interval over which the person has been diagnosed with OUD.**

```
OPIOID_CONDITIONS_SUMMARY_PER_PERSON_QUERY <- '
opioid_conditions_summary_per_person_tbl AS (
    SELECT
        person_id,
        MIN(condition_start_datetime) AS condition_start_datetime,
        MAX(condition_end_datetime) AS condition_end_datetime,
        COUNT(1) AS opioid_condition_count,
        COUNT(DISTINCT standard_concept_code) AS opioid_conditions_mult_count,
        STRING_AGG(DISTINCT standard_concept_name, ", ") AS opioid_conditions
    FROM 
        opioid_conditions_with_end_time_filled_tbl
    GROUP BY
        person_id
)'
```

**c. Query for combining the three queries defined above**

```
opioid_conditions_summary_per_person <- formulate_and_run_multipart_query(
    c(DEMOGRAPHICS_QUERY, 
      OPIOID_CONDITIONS_QUERY,
      OPIOID_CONDITIONS_WITH_END_TIME_FILLED_QUERY,
      OPIOID_CONDITIONS_SUMMARY_PER_PERSON_QUERY),
    'opioid_conditions_summary_per_person_tbl')
WITH
-- This query represents dataset "Demographics for AoU WGS cohort" for domain "person" and was
-- generated for All of Us Controlled Tier Dataset v5 alpha and then further edited.
demographics_tbl AS (
    SELECT
        person.BIRTH_DATETIME as date_of_birth,
        person.PERSON_ID as person_id,
        p_race_concept.concept_name as race,
        p_gender_concept.concept_name as gender,
        p_ethnicity_concept.concept_name as ethnicity,
        p_sex_at_birth_concept.concept_name as sex_at_birth
    FROM
        `person` person
    LEFT JOIN `concept` p_race_concept on person.race_concept_id = p_race_concept.CONCEPT_ID
    LEFT JOIN `concept` p_gender_concept on person.gender_concept_id = p_gender_concept.CONCEPT_ID
    LEFT JOIN `concept` p_ethnicity_concept on person.ethnicity_concept_id = p_ethnicity_concept.CONCEPT_ID
    LEFT JOIN `concept` p_sex_at_birth_concept on person.sex_at_birth_concept_id = p_sex_at_birth_concept.CONCEPT_ID
    WHERE person.PERSON_ID IN (
    SELECT person_id from `cb_search_person` cb_search_person
    WHERE cb_search_person.person_id in (
        SELECT person_id from `cb_search_person` p
        WHERE has_whole_genome_variant = 1))
),
```
```
opioid_conditions_tbl AS (
    SELECT
        c_occurrence.person_id as person_id,
        c_occurrence.condition_concept_id as condition_concept_id,
        c_occurrence.condition_start_datetime as condition_start_datetime,
        c_occurrence.condition_end_datetime as condition_end_datetime,
        c_standard_concept.concept_name as standard_concept_name,
        c_standard_concept.concept_code as standard_concept_code
    FROM
        ( SELECT
            * 
        FROM
            `condition_occurrence` c_occurrence 
        WHERE
            (
                condition_concept_id IN  (
                    SELECT
                        DISTINCT c.concept_id 
                    FROM
                        `cb_criteria` c 
                    JOIN
                        (
                            select
                                cast(cr.id as string) as id 
                            FROM
                                `cb_criteria` cr 
                            WHERE
                                concept_id IN (438120, 438130, 440693) 
                                AND full_text LIKE '%_rank1]%'
                        ) a 
                            ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                            OR c.path LIKE CONCAT('%.', a.id) 
                            OR c.path LIKE CONCAT(a.id, '.%') 
                            OR c.path = a.id) 
                        WHERE
                            is_standard = 1 
                            AND is_selectable = 1
                        )
                )  
                AND (
                    c_occurrence.PERSON_ID IN (
                        SELECT
                            distinct person_id  
                        FROM
                            `cb_search_person` cb_search_person  
                        WHERE
                            cb_search_person.person_id IN (
                                SELECT
                                    person_id 
                                FROM
                                    `cb_search_person` p 
                                WHERE
                                    has_whole_genome_variant = 1 
                            ) 
                            AND cb_search_person.person_id IN (
                                SELECT
                                    criteria.person_id 
                                FROM
                                    (SELECT
                                        DISTINCT person_id,
                                        entry_date,
                                        concept_id 
                                    FROM
                                        `cb_search_all_events` 
                                    WHERE
                                        (
                                            concept_id IN (
                                                SELECT
                                                    DISTINCT c.concept_id 
                                                FROM
                                                    `cb_criteria` c 
                                                JOIN
                                                    (
                                                        select
                                                            cast(cr.id as string) as id 
                                                        FROM
                                                            `cb_criteria` cr 
                                                        WHERE
                                                            concept_id IN (438120, 438130, 440693) 
                                                            AND full_text LIKE '%_rank1]%'
                                                    ) a 
                                                        ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                                                        OR c.path LIKE CONCAT('%.', a.id) 
                                                        OR c.path LIKE CONCAT(a.id, '.%') 
                                                        OR c.path = a.id) 
                                                    WHERE
                                                        is_standard = 1 
                                                        AND is_selectable = 1
                                                    ) 
                                                    AND is_standard = 1 
                                            )
                                        ) criteria 
                                    ) 
                                    AND cb_search_person.person_id IN (
                                        SELECT
                                            criteria.person_id 
                                        FROM
                                            (SELECT
                                                DISTINCT person_id,
                                                entry_date,
                                                concept_id 
                                            FROM
                                                `cb_search_all_events` 
                                            WHERE
                                                (
                                                    concept_id IN (
                                                        SELECT
                                                            DISTINCT ca.descendant_id 
                                                        FROM
                                                            `cb_criteria_ancestor` ca 
                                                        JOIN
                                                            (
                                                                select
                                                                    distinct c.concept_id 
                                                                FROM
                                                                    `cb_criteria` c 
                                                                JOIN
                                                                    (
                                                                        select
                                                                            cast(cr.id as string) as id 
                                                                        FROM
                                                                            `cb_criteria` cr 
                                                                        WHERE
                                                                            concept_id IN (21604254) 
                                                                            AND full_text LIKE '%_rank1]%'
                                                                    ) a 
                                                                        ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                                                                        OR c.path LIKE CONCAT('%.', a.id) 
                                                                        OR c.path LIKE CONCAT(a.id, '.%') 
                                                                        OR c.path = a.id) 
                                                                    WHERE
                                                                        is_standard = 1 
                                                                        AND is_selectable = 1
                                                                    ) b 
                                                                        ON (
                                                                            ca.ancestor_id = b.concept_id
                                                                        )
                                                                ) 
                                                                AND is_standard = 1
                                                            )
                                                    ) criteria 
                                                ) 
                                            ))) c_occurrence 
                                LEFT JOIN
                                    `concept` c_standard_concept 
                                        ON c_occurrence.condition_concept_id = c_standard_concept.concept_id),
```
```
opioid_conditions_with_end_time_filled_tbl AS (
    SELECT
        person_id,
        condition_concept_id,
        condition_start_datetime,
        IFNULL(condition_end_datetime, condition_start_datetime) AS condition_end_datetime,
        standard_concept_code,
        standard_concept_name
    FROM
        opioid_conditions_tbl
),
```
```
opioid_conditions_summary_per_person_tbl AS (
    SELECT
        person_id,
        MIN(condition_start_datetime) AS condition_start_datetime,
        MAX(condition_end_datetime) AS condition_end_datetime,
        COUNT(1) AS opioid_condition_count,
        COUNT(DISTINCT standard_concept_code) AS opioid_conditions_mult_count,
        STRING_AGG(DISTINCT standard_concept_name, ", ") AS opioid_conditions
    FROM 
        opioid_conditions_with_end_time_filled_tbl
    GROUP BY
        person_id
)

SELECT * FROM opioid_conditions_summary_per_person_tbl
```
*Output*
```
Dimensions of result: num_rows=6144 num_cols=6

opioid_conditions_summary_per_person %>%
    arrange(desc(opioid_condition_count)) %>%
    head(n = 20)
3251251	2005-12-08 14:32:00	2020-10-26 08:30:00	3071	6	Combined opioid with other drug dependence, continuous, Combined opioid with other drug dependence, Continuous opioid dependence, Opioid abuse, Opioid dependence in remission, Opioid dependence
2657305	2014-08-27 10:30:00	2019-12-10 12:19:52	1920	5	Combined opioid with other drug dependence, continuous, Continuous opioid dependence, Opioid abuse, Opioid dependence, Opioid dependence in remission
1318679	2011-06-06 14:00:00	2022-07-01 11:38:01	1283	5	Continuous opioid dependence, Opioid dependence, Opioid abuse, Combined opioid with other drug dependence, Opioid dependence in remission
1512090	2004-10-28 00:00:00	2022-05-18 11:59:59	1045	3	Opioid dependence, Continuous opioid dependence, Opioid dependence in remission
1241772	2012-12-31 11:00:00	2022-06-09 08:30:00	957	5	Combined opioid with other drug dependence, Opioid abuse, Opioid dependence in remission, Opioid dependence, Continuous opioid dependence
3334972	2020-07-08 00:00:00	2022-05-26 00:00:00	926	2	Opioid abuse, Opioid dependence
1113550	2000-04-19 08:50:38	2022-04-05 09:34:34	911	6	Combined opioid with other drug dependence, Opioid abuse, Continuous opioid dependence, Opioid dependence in remission, Combined opioid with other drug dependence, continuous, Opioid dependence
1588187	2001-06-27 00:00:00	2022-05-26 11:59:59	908	4	Opioid dependence in remission, Opioid abuse, Combined opioid with other drug dependence, Opioid dependence
1927924	2002-03-13 13:45:06	2022-06-10 09:15:00	846	4	Continuous opioid dependence, Opioid dependence in remission, Opioid dependence, Opioid abuse
1717757	2012-09-06 09:30:00	2022-05-19 10:00:00	768	5	Opioid abuse, Opioid dependence, Continuous opioid dependence, Episodic opioid dependence, Opioid dependence in remission
3418223	2006-11-22 00:00:00	2022-06-22 00:00:00	755	3	Opioid dependence, Opioid dependence in remission, Continuous opioid dependence
3132224	2009-11-10 10:02:03	2022-05-26 11:15:00	739	3	Opioid dependence in remission, Opioid abuse, Opioid dependence
1541065	2009-09-29 13:00:00	2022-05-24 00:00:00	733	3	Opioid dependence in remission, Continuous opioid dependence, Opioid dependence
1904878	2018-10-04 00:00:00	2022-05-26 00:00:00	676	2	Opioid dependence, Opioid abuse
3297582	2015-04-03 00:00:00	2022-06-08 00:00:00	667	4	Opioid abuse, Continuous opioid dependence, Opioid dependence in remission, Opioid dependence
2153830	2012-01-28 00:00:00	2022-04-19 11:59:59	644	5	Opioid dependence in remission, Episodic opioid dependence, Opioid dependence, Continuous opioid dependence, Opioid abuse
1271964	2003-01-29 11:00:00	2022-05-31 11:35:32	632	4	Opioid dependence in remission, Continuous opioid dependence, Opioid dependence, Opioid abuse
1068273	2002-05-28 00:00:00	2022-05-27 11:59:59	588	7	Combined opioid with other drug dependence in remission, Opioid dependence in remission, Combined opioid with other drug dependence, Opioid abuse, Episodic opioid dependence, Opioid dependence, Combined opioid with other drug dependence, continuous
1070882	2012-03-01 00:00:00	2022-05-25 11:59:59	581	4	Combined opioid with other drug dependence, Opioid abuse, Opioid dependence, Opioid dependence in remission
3002988	2013-03-01 00:00:00	2022-05-18 11:59:59	574	4	Opioid abuse, Opioid dependence, Continuous opioid dependence, Opioid dependence in remission
```

```
opioid_conditions_summary_per_person %>%
    arrange(desc(opioid_condition_count)) %>%
    tail(n = 20)
3428408	2019-08-28 12:12:44	2019-08-28 12:12:44	1	1	Opioid abuse
2748007	2013-09-19 05:00:00	2013-09-19 05:00:00	1	1	Opioid abuse
3371472	2019-04-03 06:34:09	2019-04-03 06:34:09	1	1	Opioid abuse
2894932	2021-03-11 04:25:26	2021-03-11 04:25:26	1	1	Opioid abuse
1036269	2019-11-17 23:18:00	2019-11-17 23:18:00	1	1	Opioid abuse
2065699	2019-06-11 20:26:00	2019-06-11 20:26:00	1	1	Opioid abuse
1378661	2019-07-28 01:29:15	2019-07-28 01:29:15	1	1	Opioid dependence
2727741	2019-04-25 17:34:01	2019-04-25 17:34:01	1	1	Opioid dependence
2075402	2019-06-19 00:00:00	2019-06-20 11:59:59	1	1	Opioid dependence
3383250	2018-09-14 11:12:00	2018-09-14 11:12:00	1	1	Opioid dependence
1750436	2019-12-09 02:25:39	2019-12-09 02:25:39	1	1	Opioid dependence
2420175	2022-02-16 15:30:00	2022-02-16 15:30:00	1	1	Opioid dependence
1272781	2022-06-17 19:37:00	2022-06-17 19:37:00	1	1	Opioid dependence
1731615	2017-05-04 06:00:00	2017-05-04 06:00:00	1	1	Opioid dependence
3051640	2021-11-01 22:22:52	2021-11-01 22:22:52	1	1	Opioid abuse
2211906	2012-07-12 05:00:00	2012-07-12 05:00:00	1	1	Opioid abuse
2962758	2015-12-15 06:00:00	2015-12-15 06:00:00	1	1	Opioid abuse
1548652	2017-01-15 23:20:00	2017-01-15 23:20:00	1	1	Opioid abuse
8609040	2020-12-09 00:00:00	2020-12-09 00:00:00	1	1	Opioid dependence
1896390	2018-01-30 06:00:00	2018-01-30 06:00:00	1	1	Opioid dependence
opioid_conditions_summary_per_person %>%
    arrange(desc(opioid_conditions_mult_count)) %>%
    head(n = 20)
1427961	2006-02-01 12:19:59	2022-06-30 11:31:00	429	9	Combined opioid with other drug dependence, episodic, Opioid abuse, Opioid dependence, Combined opioid with other drug dependence in remission, Combined opioid with other drug dependence, Continuous opioid dependence, Opioid dependence in remission, Episodic opioid dependence, Combined opioid with other drug dependence, continuous
1708434	2005-02-04 00:00:00	2021-12-06 11:59:59	133	8	Combined opioid with other drug dependence, continuous, Continuous opioid dependence, Opioid abuse, Combined opioid with other drug dependence, episodic, Opioid dependence, Opioid dependence in remission, Combined opioid with other drug dependence in remission, Combined opioid with other drug dependence
1993997	2000-05-04 09:10:00	2019-06-24 13:51:50	158	7	Continuous opioid dependence, Opioid dependence in remission, Opioid abuse, Combined opioid with other drug dependence, continuous, Opioid dependence, Episodic opioid dependence, Combined opioid with other drug dependence
1068273	2002-05-28 00:00:00	2022-05-27 11:59:59	588	7	Combined opioid with other drug dependence in remission, Opioid dependence in remission, Combined opioid with other drug dependence, Opioid abuse, Episodic opioid dependence, Opioid dependence, Combined opioid with other drug dependence, continuous
1489973	2005-03-17 05:00:00	2022-06-20 11:59:59	193	7	Opioid dependence in remission, Combined opioid with other drug dependence, continuous, Opioid abuse, Combined opioid with other drug dependence, episodic, Opioid dependence, Episodic opioid dependence, Continuous opioid dependence
3089611	2009-09-23 00:00:00	2020-01-12 11:59:59	121	7	Combined opioid with other drug dependence, continuous, Opioid dependence, Continuous opioid dependence, Combined opioid with other drug dependence, Combined opioid with other drug dependence in remission, Opioid dependence in remission, Opioid abuse
2144304	2002-08-15 10:40:28	2014-01-21 11:00:00	114	6	Opioid dependence in remission, Combined opioid with other drug dependence, continuous, Episodic opioid dependence, Continuous opioid dependence, Opioid dependence, Opioid abuse
1050714	2002-11-01 12:47:00	2022-05-23 14:42:53	402	6	Combined opioid with other drug dependence, Continuous opioid dependence, Opioid abuse, Combined opioid with other drug dependence, continuous, Opioid dependence, Opioid dependence in remission
1246869	2003-12-25 19:18:00	2021-11-12 00:00:00	33	6	Combined opioid with other drug dependence, continuous, Episodic opioid dependence, Opioid dependence in remission, Opioid dependence, Opioid abuse, Continuous opioid dependence
1922769	2014-11-01 20:57:00	2021-09-14 04:11:00	69	6	Methadone dependence, Opioid abuse, Opioid dependence, Continuous opioid dependence, Combined opioid with other drug dependence, Opioid dependence in remission
1468620	2007-12-05 07:58:00	2022-02-20 00:00:00	82	6	Combined opioid with other drug dependence, Opioid dependence in remission, Opioid abuse, Continuous opioid dependence, Combined opioid with other drug dependence, continuous, Opioid dependence
1725601	2004-05-15 05:00:00	2022-02-20 05:00:00	51	6	Opioid dependence in remission, Opioid abuse, Combined opioid with other drug dependence, continuous, Combined opioid with other drug dependence, Opioid dependence, Continuous opioid dependence
1409508	2011-06-01 10:40:00	2020-10-27 08:20:00	193	6	Episodic opioid dependence, Opioid dependence in remission, Combined opioid with other drug dependence, Continuous opioid dependence, Opioid abuse, Opioid dependence
1495418	2001-08-28 11:46:00	2022-06-30 15:00:00	467	6	Episodic opioid dependence, Opioid dependence in remission, Opioid abuse, Opioid dependence, Combined opioid with other drug dependence, continuous, Continuous opioid dependence
1113550	2000-04-19 08:50:38	2022-04-05 09:34:34	911	6	Combined opioid with other drug dependence, Opioid abuse, Continuous opioid dependence, Opioid dependence in remission, Combined opioid with other drug dependence, continuous, Opioid dependence
3251251	2005-12-08 14:32:00	2020-10-26 08:30:00	3071	6	Combined opioid with other drug dependence, continuous, Combined opioid with other drug dependence, Continuous opioid dependence, Opioid abuse, Opioid dependence in remission, Opioid dependence
1684205	2007-10-01 10:00:00	2015-11-11 13:40:00	143	5	Episodic opioid dependence, Opioid abuse, Continuous opioid dependence, Opioid dependence, Opioid dependence in remission
1076613	2004-04-08 10:48:00	2018-04-10 20:55:00	32	5	Combined opioid with other drug dependence, continuous, Continuous opioid dependence, Episodic opioid dependence, Opioid abuse, Opioid dependence
3138149	2012-04-27 21:29:00	2019-04-15 00:00:00	292	5	Combined opioid with other drug dependence, continuous, Opioid dependence, Opioid dependence in remission, Continuous opioid dependence, Opioid abuse
1450821	2008-05-08 10:14:07	2016-11-02 10:30:00	79	5	Combined opioid with other drug dependence, continuous, Opioid abuse, Opioid dependence, Combined opioid with other drug dependence, Episodic opioid dependence
```

**d. Query for combining demographics and OUD conditions into one dataframe for case sample**

```
CASE_QUERY <- '
case_tbl AS (
    SELECT
        demog.*,
        opioid_conditions.* EXCEPT(person_id)
    FROM
        opioid_conditions_summary_per_person_tbl AS opioid_conditions
    LEFT JOIN
        demographics_tbl AS demog
    ON
        demog.person_id = opioid_conditions.person_id
)'
case <- formulate_and_run_multipart_query(
    c(DEMOGRAPHICS_QUERY,
      OPIOID_CONDITIONS_QUERY,
      OPIOID_CONDITIONS_WITH_END_TIME_FILLED_QUERY,
      OPIOID_CONDITIONS_SUMMARY_PER_PERSON_QUERY,
      CASE_QUERY),
    'case_tbl')
WITH
-- This query represents dataset "Demographics for AoU WGS cohort" for domain "person" and was
-- generated for All of Us Controlled Tier Dataset v5 alpha and then further edited.
demographics_tbl AS (
    SELECT
        person.BIRTH_DATETIME as date_of_birth,
        person.PERSON_ID as person_id,
        p_race_concept.concept_name as race,
        p_gender_concept.concept_name as gender,
        p_ethnicity_concept.concept_name as ethnicity,
        p_sex_at_birth_concept.concept_name as sex_at_birth
    FROM
        `person` person
    LEFT JOIN `concept` p_race_concept on person.race_concept_id = p_race_concept.CONCEPT_ID
    LEFT JOIN `concept` p_gender_concept on person.gender_concept_id = p_gender_concept.CONCEPT_ID
    LEFT JOIN `concept` p_ethnicity_concept on person.ethnicity_concept_id = p_ethnicity_concept.CONCEPT_ID
    LEFT JOIN `concept` p_sex_at_birth_concept on person.sex_at_birth_concept_id = p_sex_at_birth_concept.CONCEPT_ID
    WHERE person.PERSON_ID IN (
    SELECT person_id from `cb_search_person` cb_search_person
    WHERE cb_search_person.person_id in (
        SELECT person_id from `cb_search_person` p
        WHERE has_whole_genome_variant = 1))
),
```

```
opioid_conditions_tbl AS (
    SELECT
        c_occurrence.person_id as person_id,
        c_occurrence.condition_concept_id as condition_concept_id,
        c_occurrence.condition_start_datetime as condition_start_datetime,
        c_occurrence.condition_end_datetime as condition_end_datetime,
        c_standard_concept.concept_name as standard_concept_name,
        c_standard_concept.concept_code as standard_concept_code
    FROM
        ( SELECT
            * 
        FROM
            `condition_occurrence` c_occurrence 
        WHERE
            (
                condition_concept_id IN  (
                    SELECT
                        DISTINCT c.concept_id 
                    FROM
                        `cb_criteria` c 
                    JOIN
                        (
                            select
                                cast(cr.id as string) as id 
                            FROM
                                `cb_criteria` cr 
                            WHERE
                                concept_id IN (438120, 438130, 440693) 
                                AND full_text LIKE '%_rank1]%'
                        ) a 
                            ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                            OR c.path LIKE CONCAT('%.', a.id) 
                            OR c.path LIKE CONCAT(a.id, '.%') 
                            OR c.path = a.id) 
                        WHERE
                            is_standard = 1 
                            AND is_selectable = 1
                        )
                )  
                AND (
                    c_occurrence.PERSON_ID IN (
                        SELECT
                            distinct person_id  
                        FROM
                            `cb_search_person` cb_search_person  
                        WHERE
                            cb_search_person.person_id IN (
                                SELECT
                                    person_id 
                                FROM
                                    `cb_search_person` p 
                                WHERE
                                    has_whole_genome_variant = 1 
                            ) 
                            AND cb_search_person.person_id IN (
                                SELECT
                                    criteria.person_id 
                                FROM
                                    (SELECT
                                        DISTINCT person_id,
                                        entry_date,
                                        concept_id 
                                    FROM
                                        `cb_search_all_events` 
                                    WHERE
                                        (
                                            concept_id IN (
                                                SELECT
                                                    DISTINCT c.concept_id 
                                                FROM
                                                    `cb_criteria` c 
                                                JOIN
                                                    (
                                                        select
                                                            cast(cr.id as string) as id 
                                                        FROM
                                                            `cb_criteria` cr 
                                                        WHERE
                                                            concept_id IN (438120, 438130, 440693) 
                                                            AND full_text LIKE '%_rank1]%'
                                                    ) a 
                                                        ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                                                        OR c.path LIKE CONCAT('%.', a.id) 
                                                        OR c.path LIKE CONCAT(a.id, '.%') 
                                                        OR c.path = a.id) 
                                                    WHERE
                                                        is_standard = 1 
                                                        AND is_selectable = 1
                                                    ) 
                                                    AND is_standard = 1 
                                            )
                                        ) criteria 
                                    ) 
                                    AND cb_search_person.person_id IN (
                                        SELECT
                                            criteria.person_id 
                                        FROM
                                            (SELECT
                                                DISTINCT person_id,
                                                entry_date,
                                                concept_id 
                                            FROM
                                                `cb_search_all_events` 
                                            WHERE
                                                (
                                                    concept_id IN (
                                                        SELECT
                                                            DISTINCT ca.descendant_id 
                                                        FROM
                                                            `cb_criteria_ancestor` ca 
                                                        JOIN
                                                            (
                                                                select
                                                                    distinct c.concept_id 
                                                                FROM
                                                                    `cb_criteria` c 
                                                                JOIN
                                                                    (
                                                                        select
                                                                            cast(cr.id as string) as id 
                                                                        FROM
                                                                            `cb_criteria` cr 
                                                                        WHERE
                                                                            concept_id IN (21604254) 
                                                                            AND full_text LIKE '%_rank1]%'
                                                                    ) a 
                                                                        ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                                                                        OR c.path LIKE CONCAT('%.', a.id) 
                                                                        OR c.path LIKE CONCAT(a.id, '.%') 
                                                                        OR c.path = a.id) 
                                                                    WHERE
                                                                        is_standard = 1 
                                                                        AND is_selectable = 1
                                                                    ) b 
                                                                        ON (
                                                                            ca.ancestor_id = b.concept_id
                                                                        )
                                                                ) 
                                                                AND is_standard = 1
                                                            )
                                                    ) criteria 
                                                ) 
                                            ))) c_occurrence 
                                LEFT JOIN
                                    `concept` c_standard_concept 
                                        ON c_occurrence.condition_concept_id = c_standard_concept.concept_id),
```

```
opioid_conditions_with_end_time_filled_tbl AS (
    SELECT
        person_id,
        condition_concept_id,
        condition_start_datetime,
        IFNULL(condition_end_datetime, condition_start_datetime) AS condition_end_datetime,
        standard_concept_code,
        standard_concept_name
    FROM
        opioid_conditions_tbl
),
```

```
opioid_conditions_summary_per_person_tbl AS (
    SELECT
        person_id,
        MIN(condition_start_datetime) AS condition_start_datetime,
        MAX(condition_end_datetime) AS condition_end_datetime,
        COUNT(1) AS opioid_condition_count,
        COUNT(DISTINCT standard_concept_code) AS opioid_conditions_mult_count,
        STRING_AGG(DISTINCT standard_concept_name, ", ") AS opioid_conditions
    FROM 
        opioid_conditions_with_end_time_filled_tbl
    GROUP BY
        person_id
),
```

```
case_tbl AS (
    SELECT
        demog.*,
        opioid_conditions.* EXCEPT(person_id)
    FROM
        opioid_conditions_summary_per_person_tbl AS opioid_conditions
    LEFT JOIN
        demographics_tbl AS demog
    ON
        demog.person_id = opioid_conditions.person_id
)

SELECT * FROM case_tbl
```
*Output*
```
Dimensions of result: num_rows=6144 num_cols=11

colnames(case)
head(case)
'date_of_birth'
'person_id'
'race'
'gender'
'ethnicity'
'sex_at_birth'
'condition_start_datetime'
'condition_end_datetime'
'opioid_condition_count'
'opioid_conditions_mult_count'
'opioid_conditions'
1979-06-15	1357846	White	Male	Not Hispanic or Latino	Intersex	2015-08-08 12:28:00	2020-09-09 18:59:00	26	3	Opioid abuse, Opioid dependence, Continuous opioid dependence
1994-06-15	1686051	PMI: Skip	I prefer not to answer	PMI: Skip	I prefer not to answer	2015-10-02 20:23:00	2020-11-24 19:44:00	53	2	Opioid abuse, Opioid dependence
1958-06-15	3291352	White	Male	Not Hispanic or Latino	None	2007-05-08 00:00:00	2022-03-23 11:59:59	80	2	Opioid dependence, Opioid abuse
1963-06-15	1027261	Black or African American	Gender Identity: Transgender	Not Hispanic or Latino	I prefer not to answer	2009-06-02 00:00:00	2009-06-04 11:59:59	1	1	Opioid abuse
1986-06-15	1179190	None of these	Gender Identity: Non Binary	What Race Ethnicity: Race Ethnicity None Of These	Intersex	2022-05-12 23:21:15	2022-05-12 23:21:15	1	1	Opioid abuse
1999-06-15	2366246	White	Not man only, not woman only, prefer not to answer, or skipped	Not Hispanic or Latino	None	2018-11-15 17:40:00	2018-11-15 17:40:00	1	1	Opioid abuse
```


**Query for control sample, select on people who answered "yes" to some degrees of opioid consumption, but excluding those who are already in our case sample**
```
## control
OPIOID_SURVEYS_QUERY <- "
opioid_surveys_tbl AS (
    SELECT
        answer.person_id,
        answer.survey,
        answer.question,
        answer.answer
    FROM
        `ds_survey` answer   
    WHERE
        (question_concept_id IN (1585636, 1585692, 1585698))  
        AND (answer.PERSON_ID IN (
                SELECT
                    distinct person_id  
                FROM
                    `cb_search_person` cb_search_person  
                WHERE
                    cb_search_person.person_id IN (
                        SELECT
                            person_id 
                        FROM
                            `cb_search_person` p 
                        WHERE
                            has_whole_genome_variant = 1 
                    ) 
                    AND cb_search_person.person_id IN (
                        SELECT
                            person_id 
                        FROM
                            `cb_search_all_events` 
                        WHERE
                            person_id IN (
                                SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585636) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585644)
                                UNION
                                ALL SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585636) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585645)
                                UNION
                                ALL SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585698) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585703)
                                UNION
                                ALL SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585692) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585697)
                            )
                    ) 
                    AND cb_search_person.person_id NOT IN (
                        SELECT
                            criteria.person_id 
                        FROM
                            (SELECT
                                DISTINCT person_id,
                                entry_date,
                                concept_id 
                            FROM
                                `cb_search_all_events` 
                            WHERE
                                (
                                    concept_id IN (
                                        SELECT
                                            DISTINCT c.concept_id 
                                        FROM
                                            `cb_criteria` c 
                                        JOIN
                                            (
                                                select
                                                    cast(cr.id as string) as id 
                                                FROM
                                                    `cb_criteria` cr 
                                                WHERE
                                                    concept_id IN (438120, 440693, 438130) 
                                                    AND full_text LIKE '%_rank1]%'
                                            ) a 
                                                ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                                                OR c.path LIKE CONCAT('%.', a.id) 
                                                OR c.path LIKE CONCAT(a.id, '.%') 
                                                OR c.path = a.id) 
                                            WHERE
                                                is_standard = 1 
                                                AND is_selectable = 1
                                            ) 
                                            AND is_standard = 1 
                                    )
                                ) criteria 
                            ))))"
```
```
opioid_surveys_summary_per_person <- formulate_and_run_multipart_query(
    c(OPIOID_SURVEYS_QUERY),
    'opioid_surveys_tbl')
WITH

opioid_surveys_tbl AS (
    SELECT
        answer.person_id,
        answer.survey,
        answer.question,
        answer.answer
    FROM
        `ds_survey` answer   
    WHERE
        (question_concept_id IN (1585636, 1585692, 1585698))  
        AND (answer.PERSON_ID IN (
                SELECT
                    distinct person_id  
                FROM
                    `cb_search_person` cb_search_person  
                WHERE
                    cb_search_person.person_id IN (
                        SELECT
                            person_id 
                        FROM
                            `cb_search_person` p 
                        WHERE
                            has_whole_genome_variant = 1 
                    ) 
                    AND cb_search_person.person_id IN (
                        SELECT
                            person_id 
                        FROM
                            `cb_search_all_events` 
                        WHERE
                            person_id IN (
                                SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585636) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585644)
                                UNION
                                ALL SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585636) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585645)
                                UNION
                                ALL SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585698) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585703)
                                UNION
                                ALL SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585692) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585697)
                            )
                    ) 
                    AND cb_search_person.person_id NOT IN (
                        SELECT
                            criteria.person_id 
                        FROM
                            (SELECT
                                DISTINCT person_id,
                                entry_date,
                                concept_id 
                            FROM
                                `cb_search_all_events` 
                            WHERE
                                (
                                    concept_id IN (
                                        SELECT
                                            DISTINCT c.concept_id 
                                        FROM
                                            `cb_criteria` c 
                                        JOIN
                                            (
                                                select
                                                    cast(cr.id as string) as id 
                                                FROM
                                                    `cb_criteria` cr 
                                                WHERE
                                                    concept_id IN (438120, 440693, 438130) 
                                                    AND full_text LIKE '%_rank1]%'
                                            ) a 
                                                ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                                                OR c.path LIKE CONCAT('%.', a.id) 
                                                OR c.path LIKE CONCAT(a.id, '.%') 
                                                OR c.path = a.id) 
                                            WHERE
                                                is_standard = 1 
                                                AND is_selectable = 1
                                            ) 
                                            AND is_standard = 1 
                                    )
                                ) criteria 
                            ))))

SELECT * FROM opioid_surveys_tbl
```

```
Dimensions of result: num_rows=90351 num_cols=4

opioid_surveys_summary_per_person %>%
    head(n = 20)
3201396	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
3248443	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
1363551	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
2413258	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
2962803	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
3124504	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
4887789	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
2783217	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
1829458	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
2582168	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
2313053	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
1897057	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
1896164	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
1591042	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
1688658	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
2236698	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
3416422	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
9335747	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
2823696	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
1356080	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
```

```
CONTROL_QUERY <- '
control_tbl AS (
    SELECT
        demog.*,
        opioid_surveys.* EXCEPT(person_id)
    FROM
        opioid_surveys_tbl AS opioid_surveys
    LEFT JOIN
        demographics_tbl AS demog
    ON
        demog.person_id = opioid_surveys.person_id
)'
control <- formulate_and_run_multipart_query(
    c(DEMOGRAPHICS_QUERY,
      OPIOID_SURVEYS_QUERY,
      CONTROL_QUERY),
    'control_tbl')
WITH
-- This query represents dataset "Demographics for AoU WGS cohort" for domain "person" and was
-- generated for All of Us Controlled Tier Dataset v5 alpha and then further edited.
demographics_tbl AS (
    SELECT
        person.BIRTH_DATETIME as date_of_birth,
        person.PERSON_ID as person_id,
        p_race_concept.concept_name as race,
        p_gender_concept.concept_name as gender,
        p_ethnicity_concept.concept_name as ethnicity,
        p_sex_at_birth_concept.concept_name as sex_at_birth
    FROM
        `person` person
    LEFT JOIN `concept` p_race_concept on person.race_concept_id = p_race_concept.CONCEPT_ID
    LEFT JOIN `concept` p_gender_concept on person.gender_concept_id = p_gender_concept.CONCEPT_ID
    LEFT JOIN `concept` p_ethnicity_concept on person.ethnicity_concept_id = p_ethnicity_concept.CONCEPT_ID
    LEFT JOIN `concept` p_sex_at_birth_concept on person.sex_at_birth_concept_id = p_sex_at_birth_concept.CONCEPT_ID
    WHERE person.PERSON_ID IN (
    SELECT person_id from `cb_search_person` cb_search_person
    WHERE cb_search_person.person_id in (
        SELECT person_id from `cb_search_person` p
        WHERE has_whole_genome_variant = 1))
),
```

```
opioid_surveys_tbl AS (
    SELECT
        answer.person_id,
        answer.survey,
        answer.question,
        answer.answer
    FROM
        `ds_survey` answer   
    WHERE
        (question_concept_id IN (1585636, 1585692, 1585698))  
        AND (answer.PERSON_ID IN (
                SELECT
                    distinct person_id  
                FROM
                    `cb_search_person` cb_search_person  
                WHERE
                    cb_search_person.person_id IN (
                        SELECT
                            person_id 
                        FROM
                            `cb_search_person` p 
                        WHERE
                            has_whole_genome_variant = 1 
                    ) 
                    AND cb_search_person.person_id IN (
                        SELECT
                            person_id 
                        FROM
                            `cb_search_all_events` 
                        WHERE
                            person_id IN (
                                SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585636) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585644)
                                UNION
                                ALL SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585636) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585645)
                                UNION
                                ALL SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585698) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585703)
                                UNION
                                ALL SELECT
                                    person_id 
                                FROM
                                    `cb_search_all_events` 
                                WHERE
                                    concept_id IN (1585692) 
                                    AND is_standard = 0  
                                    AND  value_source_concept_id IN (1585697)
                            )
                    ) 
                    AND cb_search_person.person_id NOT IN (
                        SELECT
                            criteria.person_id 
                        FROM
                            (SELECT
                                DISTINCT person_id,
                                entry_date,
                                concept_id 
                            FROM
                                `cb_search_all_events` 
                            WHERE
                                (
                                    concept_id IN (
                                        SELECT
                                            DISTINCT c.concept_id 
                                        FROM
                                            `cb_criteria` c 
                                        JOIN
                                            (
                                                select
                                                    cast(cr.id as string) as id 
                                                FROM
                                                    `cb_criteria` cr 
                                                WHERE
                                                    concept_id IN (438120, 440693, 438130) 
                                                    AND full_text LIKE '%_rank1]%'
                                            ) a 
                                                ON (c.path LIKE CONCAT('%.', a.id, '.%') 
                                                OR c.path LIKE CONCAT('%.', a.id) 
                                                OR c.path LIKE CONCAT(a.id, '.%') 
                                                OR c.path = a.id) 
                                            WHERE
                                                is_standard = 1 
                                                AND is_selectable = 1
                                            ) 
                                            AND is_standard = 1 
                                    )
                                ) criteria 
                            )))),
```

```
control_tbl AS (
    SELECT
        demog.*,
        opioid_surveys.* EXCEPT(person_id)
    FROM
        opioid_surveys_tbl AS opioid_surveys
    LEFT JOIN
        demographics_tbl AS demog
    ON
        demog.person_id = opioid_surveys.person_id
)

SELECT * FROM control_tbl
```

```
Dimensions of result: num_rows=90351 num_cols=9

colnames(control)
head(control)
'date_of_birth'
'person_id'
'race'
'gender'
'ethnicity'
'sex_at_birth'
'survey'
'question'
'answer'
1984-06-15	1447530	I prefer not to answer	I prefer not to answer	PMI: Prefer Not To Answer	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Inhalants Use
1982-06-15	3028890	I prefer not to answer	Gender Identity: Additional Options	PMI: Prefer Not To Answer	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Sedatives Use
1974-06-15	2256992	None Indicated	Male	Hispanic or Latino	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use
1986-06-15	2079342	White	Not man only, not woman only, prefer not to answer, or skipped	Not Hispanic or Latino	Intersex	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Hallucinogens Use
1990-06-15	2291241	Black or African American	Male	Not Hispanic or Latino	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Methamphetamine Use
1974-06-15	4933067	White	I prefer not to answer	Not Hispanic or Latino	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Marijuana Use
```

4a. Take a look at number of responses for each question in a temporary dataframe, we want to select on people who have used OUD but not on a daily bases, since those people are most likely undiagnosed with OUD.

- we want our control to be everyone who has taken opioid
- exclude those who have taken opioid on a daily basis, since they are most likely undiagonosed cases
- exclude those who have answered "Never" to taking opioids

```
temp1 <- 
    control %>%
    dplyr::group_by(question, answer) %>%
    dplyr::summarise(countp = n_distinct(person_id))
```

```
temp1
`summarise()` has grouped output by 'question'. You can override using the
`.groups` argument.
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	PMI: Prefer Not To Answer	21
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	PMI: Skip	27
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Daily	111
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Monthly	62
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Never	1904
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Once Or Twice	231
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Weekly	38
Past 3 Month Use Frequency: Street Opioid 3 Month Use	PMI: Prefer Not To Answer	15
Past 3 Month Use Frequency: Street Opioid 3 Month Use	PMI: Skip	30
Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Daily	59
Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Monthly	16
Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Never	89
Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Once Or Twice	51
Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Weekly	29
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use	11037
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Hallucinogens Use	8494
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Inhalants Use	3932
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Marijuana Use	16329
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Methamphetamine Use	6207
Recreational Drug Use: Which Drugs Used	Which Drugs Used: None Of These Drugs	2
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Other Specify	230
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	17260
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Stimulants Use	7769
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Sedatives Use	9390
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	7018
```
```
temp1$control <- NULL
temp1[grep("Opioid", temp1$answer), "control"] <- 1 #we want our control to be everyone who has taken opioid  
temp1[grep("Daily", temp1$answer), "control"] <- NA #exclude those who have taken opioid on a daily basis, since they are most likely undiagonosed cases
temp1[grep("Never", temp1$answer), "control"] <- NA #exclude those who have answered "Never" to taking opioids
temp1
```

```
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	PMI: Prefer Not To Answer	21	NA
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	PMI: Skip	27	NA
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Daily	111	NA
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Monthly	62	1
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Never	1904	NA
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Once Or Twice	231	1
Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Weekly	38	1
Past 3 Month Use Frequency: Street Opioid 3 Month Use	PMI: Prefer Not To Answer	15	NA
Past 3 Month Use Frequency: Street Opioid 3 Month Use	PMI: Skip	30	NA
Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Daily	59	NA
Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Monthly	16	1
Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Never	89	NA
Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Once Or Twice	51	1
Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Weekly	29	1
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Cocaine Use	11037	NA
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Hallucinogens Use	8494	NA
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Inhalants Use	3932	NA
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Marijuana Use	16329	NA
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Methamphetamine Use	6207	NA
Recreational Drug Use: Which Drugs Used	Which Drugs Used: None Of These Drugs	2	NA
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Other Specify	230	NA
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	17260	1
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Stimulants Use	7769	NA
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Sedatives Use	9390	NA
Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	7018	1
```


4b. After confirming the desired set of people, run the code above on control dataframe


```
control$control <- NULL
control[grep("Opioid", control$answer), "control"] <- 1
control[grep("Daily", control$answer), "control"] <- NA
control[grep("Never", control$answer), "control"] <- NA
control_opioid_only <- control[!is.na(control$control),]
```
```
dim(control_opioid_only)
head(control_opioid_only)
```
```
24705
10

1998-06-15	1709008	White	Male	Not Hispanic or Latino	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1984-06-15	1447530	I prefer not to answer	I prefer not to answer	PMI: Prefer Not To Answer	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1986-06-15	2079342	White	Not man only, not woman only, prefer not to answer, or skipped	Not Hispanic or Latino	Intersex	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1969-06-15	1904084	Black or African American	Female	Not Hispanic or Latino	Intersex	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1973-06-15	1643978	I prefer not to answer	I prefer not to answer	PMI: Prefer Not To Answer	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1990-06-15	2291241	Black or African American	Male	Not Hispanic or Latino	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
Look for and remove duplicates in case and control
After taking a look a one person, it turns out that there are duplicated person_id because those people have answered "yes" to both taken prescription and street opioids. In that case, remove the duplicated rows.
case[duplicated(case$person_id),]
control_opioid_only[duplicated(control_opioid_only$person_id)|duplicated(control_opioid_only$person_id, fromLast = TRUE), ]
1973-06-15	1643978	I prefer not to answer	I prefer not to answer	PMI: Prefer Not To Answer	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1980-06-15	1382073	White	Gender Identity: Transgender	Not Hispanic or Latino	Intersex	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1998-06-15	1709008	White	Male	Not Hispanic or Latino	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1989-06-15	2449537	I prefer not to answer	Not man only, not woman only, prefer not to answer, or skipped	PMI: Prefer Not To Answer	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1993-06-15	1246866	White	Male	Not Hispanic or Latino	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1984-06-15	1447530	I prefer not to answer	I prefer not to answer	PMI: Prefer Not To Answer	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1990-06-15	2291241	Black or African American	Male	Not Hispanic or Latino	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1979-06-15	1659969	Black or African American	Female	Not Hispanic or Latino	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1982-06-15	3028890	I prefer not to answer	Gender Identity: Additional Options	PMI: Prefer Not To Answer	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1961-06-15	2864529	None of these	Not man only, not woman only, prefer not to answer, or skipped	What Race Ethnicity: Race Ethnicity None Of These	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1960-06-15	3044961	White	Gender Identity: Additional Options	Not Hispanic or Latino	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1963-06-15	2708441	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1964-06-15	3282191	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1968-06-15	1681686	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1939-06-15	2048994	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1984-06-15	1464149	More than one population	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1991-06-15	2923795	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1955-06-15	2714897	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1968-06-15	3192191	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Weekly	1
1986-06-15	1278128	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1971-06-15	1030897	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1989-06-15	2927122	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1969-06-15	2046587	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1967-06-15	3223661	Asian	Male	Not Hispanic or Latino	Male	Lifestyle	Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Once Or Twice	1
1981-06-15	3211117	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1959-06-15	1404102	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1985-06-15	2367762	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1990-06-15	3310696	White	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1950-06-15	1738656	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Past 3 Month Use Frequency: Street Opioid 3 Month Use	Street Opioid 3 Month Use: Monthly	1
1966-06-15	2505905	I prefer not to answer	Male	PMI: Prefer Not To Answer	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮
1974-06-15	3288269	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1988-06-15	1741588	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1991-06-15	1402458	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1980-06-15	2826679	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1969-06-15	9922851	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1997-06-15	1175429	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1965-06-15	1351688	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1979-06-15	1840576	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1967-06-15	3371108	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1987-06-15	1941285	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1961-06-15	3069119	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1943-06-15	1509870	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Past 3 Month Use Frequency: Prescription Opioid 3 Month Use	Prescription Opioid 3 Month Use: Once Or Twice	1
1975-06-15	1384683	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1955-06-15	1319463	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1974-06-15	1181347	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1984-06-15	1210660	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1954-06-15	1417027	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1988-06-15	2082232	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1955-06-15	3323556	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1965-06-15	2867951	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1956-06-15	1900379	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1950-06-15	3019844	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1954-06-15	3463780	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1988-06-15	3109464	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1992-06-15	1617928	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1979-06-15	3905005	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1967-06-15	2233768	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1956-06-15	1207323	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1957-06-15	1806939	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1986-06-15	2984206	PMI: Skip	PMI: Skip	PMI: Skip	No matching concept	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
```

```
#take a look a one person, it turns out that there are duplicated person_id because those people have answered "yes" to both taken prescription and street opioids
control_opioid_only %>% filter(person_id == 2449537)
1989-06-15	2449537	I prefer not to answer	Not man only, not woman only, prefer not to answer, or skipped	PMI: Prefer Not To Answer	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1989-06-15	2449537	I prefer not to answer	Not man only, not woman only, prefer not to answer, or skipped	PMI: Prefer Not To Answer	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
```

```
control_opioid_only_nodup <- control_opioid_only %>% distinct(person_id, .keep_all = TRUE)
dim(control_opioid_only)
head(control_opioid_only)
```


```
24705
10
1998-06-15	1709008	White	Male	Not Hispanic or Latino	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1984-06-15	1447530	I prefer not to answer	I prefer not to answer	PMI: Prefer Not To Answer	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1986-06-15	2079342	White	Not man only, not woman only, prefer not to answer, or skipped	Not Hispanic or Latino	Intersex	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1969-06-15	1904084	Black or African American	Female	Not Hispanic or Latino	Intersex	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1973-06-15	1643978	I prefer not to answer	I prefer not to answer	PMI: Prefer Not To Answer	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1990-06-15	2291241	Black or African American	Male	Not Hispanic or Latino	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
```



Save case and control as csv files

```
write_csv(case, CASE_FILENAME)
write_tsv(case  %>%
              mutate(
                  IID = person_id,
                  FID = IID
              ) %>%
              select(FID, IID) %>%
              distinct(),
          CASE_ID_FILENAME)
write_csv(control_opioid_only, CONTROL_FILENAME)
write_tsv(control_opioid_only  %>%
              mutate(
                  IID = person_id,
                  FID = IID
              ) %>%
              select(FID, IID) %>%
              distinct(),
          CONTROL_ID_FILENAME)
```

```
system(str_glue('gsutil cp {CASE_FILENAME} {CASE_ID_FILENAME} {CONTROL_FILENAME} {CONTROL_ID_FILENAME} {DESTINATION}'), intern = T)
system(str_glue('gsutil ls -lh {DESTINATION}'), intern = T)
```

```
' 934.5 KiB  2023-08-11T15:58:40Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/case.csv'
' 96.01 KiB  2023-08-11T15:58:40Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/case_ids.tsv'
'  3.99 MiB  2023-08-11T15:58:43Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/control.csv'
' 319.9 KiB  2023-08-11T15:58:43Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/control_ids.tsv'
'TOTAL: 4 objects, 5568266 bytes (5.31 MiB)'
```


Dividing case s and controls into ancestry groups

```
#subset of case and control
#take a look at the race  data
dim(case)
dim(control_opioid_only)

case %>%
    count(race)
control_opioid_only %>%
    count(race)
```

```
6144
11
24705
10
Asian	27
Black or African American	1779
I prefer not to answer	67
Middle Eastern or North African	13
More than one population	108
Native Hawaiian or Other Pacific Islander	9
None Indicated	809
None of these	99
PMI: Skip	158
White	3075
Asian	347
Black or African American	4489
I prefer not to answer	161
Middle Eastern or North African	85
More than one population	592
Native Hawaiian or Other Pacific Islander	34
None Indicated	2661
None of these	349
PMI: Skip	477
White	15510
```


```
#subset of black or african american
case_afr <- case %>%
    filter(race == "Black or African American")
case_afr
```

```
1963-06-15	1027261	Black or African American	Gender Identity: Transgender	Not Hispanic or Latino	I prefer not to answer	2009-06-02 00:00:00	2009-06-04 11:59:59	1	1	Opioid abuse
1988-06-15	1596947	Black or African American	I prefer not to answer	Not Hispanic or Latino	None	2011-10-18 00:00:00	2013-06-04 00:00:00	4	3	Opioid dependence, Continuous opioid dependence, Opioid abuse
1952-06-15	3010176	Black or African American	Male	Not Hispanic or Latino	Male	2003-05-27 18:00:00	2013-10-22 12:18:00	2	2	Episodic opioid dependence, Opioid dependence
1980-06-15	1076613	Black or African American	Male	Not Hispanic or Latino	Male	2004-04-08 10:48:00	2018-04-10 20:55:00	32	5	Continuous opioid dependence, Combined opioid with other drug dependence, continuous, Opioid abuse, Opioid dependence, Episodic opioid dependence
1969-06-15	1932259	Black or African American	Male	Not Hispanic or Latino	Male	2018-06-07 05:00:00	2021-09-21 00:00:00	21	2	Opioid abuse, Opioid dependence
1960-06-15	1761236	Black or African American	Male	Not Hispanic or Latino	Male	2019-06-30 05:00:00	2020-10-01 00:00:00	4	2	Opioid dependence, Opioid abuse
1991-06-15	1305113	Black or African American	Male	Not Hispanic or Latino	Male	2016-10-02 17:39:49	2019-04-03 07:35:41	2	2	Opioid abuse, Opioid dependence
1978-06-15	5114681	Black or African American	Male	Not Hispanic or Latino	Male	2021-04-24 20:05:00	2021-10-13 16:42:00	4	1	Opioid abuse
1971-06-15	3096402	Black or African American	Male	Not Hispanic or Latino	Male	2019-03-03 04:08:00	2019-03-03 23:27:00	2	1	Opioid abuse
1958-06-15	1427961	Black or African American	Male	Hispanic or Latino	Male	2006-02-01 12:19:59	2022-06-30 11:31:00	429	9	Opioid abuse, Combined opioid with other drug dependence, episodic, Opioid dependence, Combined opioid with other drug dependence in remission, Continuous opioid dependence, Combined opioid with other drug dependence, Opioid dependence in remission, Combined opioid with other drug dependence, continuous, Episodic opioid dependence
1952-06-15	2747454	Black or African American	Male	Not Hispanic or Latino	Male	2011-09-16 00:00:00	2012-09-19 00:00:00	3	2	Opioid abuse, Continuous opioid dependence
1960-06-15	1684129	Black or African American	Male	Not Hispanic or Latino	Male	2019-05-10 21:14:00	2021-02-11 23:24:00	14	2	Opioid dependence, Opioid abuse
1967-06-15	1017208	Black or African American	Male	Not Hispanic or Latino	Male	2015-09-14 00:00:00	2020-03-06 00:00:00	4	2	Opioid abuse, Opioid dependence
1968-06-15	3172619	Black or African American	Male	Not Hispanic or Latino	Male	2011-12-07 00:00:00	2020-08-17 00:00:00	8	2	Opioid dependence, Opioid abuse
1962-06-15	1612701	Black or African American	Male	Not Hispanic or Latino	Male	2005-10-18 00:00:00	2014-11-21 00:00:00	4	1	Opioid abuse
1955-06-15	1221609	Black or African American	Male	Not Hispanic or Latino	Male	2021-05-23 00:00:00	2021-05-23 00:00:00	24	1	Opioid abuse
1958-06-15	2750438	Black or African American	Male	Not Hispanic or Latino	Male	2014-05-23 15:22:08	2017-11-16 11:00:00	2	2	Opioid abuse, Opioid dependence in remission
1959-06-15	1975964	Black or African American	Male	Not Hispanic or Latino	Male	2020-11-24 00:00:00	2020-11-24 00:00:00	1	1	Opioid abuse
1993-06-15	2067196	Black or African American	Male	Not Hispanic or Latino	Male	2021-12-08 05:57:31	2021-12-08 05:57:31	1	1	Opioid abuse
1962-06-15	2015078	Black or African American	Male	Not Hispanic or Latino	Male	2018-07-17 00:00:00	2018-08-10 00:00:00	14	2	Opioid abuse, Continuous opioid dependence
1957-06-15	2500249	Black or African American	Male	Not Hispanic or Latino	Male	2020-06-22 00:00:00	2020-06-22 00:00:00	1	1	Opioid abuse
1967-06-15	1623453	Black or African American	Male	Not Hispanic or Latino	Male	2022-01-04 00:00:00	2022-04-09 00:00:00	6	1	Opioid abuse
1955-06-15	1265112	Black or African American	Male	Not Hispanic or Latino	Male	2019-05-20 00:00:00	2020-11-10 00:00:00	4	2	Opioid abuse, Opioid dependence
1975-06-15	1299117	Black or African American	Male	Not Hispanic or Latino	Male	2015-04-14 00:00:00	2016-10-03 00:00:00	2	2	Opioid abuse, Opioid dependence
1970-06-15	1356978	Black or African American	Male	Not Hispanic or Latino	Male	2009-12-04 00:00:00	2021-11-10 11:59:59	132	3	Opioid dependence in remission, Opioid dependence, Opioid abuse
1971-06-15	3363765	Black or African American	Male	Not Hispanic or Latino	Male	2012-05-27 00:00:00	2021-03-29 00:00:00	18	2	Opioid dependence, Opioid abuse
1965-06-15	1698185	Black or African American	Male	Not Hispanic or Latino	Male	2008-06-12 05:00:00	2020-04-01 11:59:59	25	2	Opioid dependence, Opioid abuse
1978-06-15	1683290	Black or African American	Male	Not Hispanic or Latino	Male	2019-04-04 09:39:00	2020-01-24 17:50:00	6	2	Opioid dependence, Opioid abuse
1986-06-15	1104938	Black or African American	Female	Not Hispanic or Latino	Male	2012-03-12 02:37:00	2019-09-20 08:35:00	8	2	Opioid abuse, Opioid dependence in remission
1976-06-15	1258046	Black or African American	Male	Not Hispanic or Latino	Male	2019-12-28 19:40:00	2019-12-28 19:40:00	1	1	Opioid abuse
⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮
1965-06-15	2049366	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	2020-08-06 00:00:00	2020-08-06 00:00:00	2	1	Opioid dependence
1953-06-15	3305384	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2014-03-10 00:00:00	2016-10-12 00:00:00	4	1	Opioid dependence
1956-06-15	1448842	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2012-10-04 05:00:00	2014-10-16 11:59:59	2	1	Opioid abuse
1959-06-15	1140818	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2014-01-21 06:00:00	2021-12-03 00:00:00	8	2	Opioid abuse, Opioid dependence
1953-06-15	2340521	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	2020-07-13 00:18:00	2020-09-03 06:33:00	2	1	Opioid abuse
1946-06-15	2719726	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2004-07-10 05:00:00	2007-03-23 05:00:00	3	1	Opioid abuse
1975-06-15	1880604	Black or African American	PMI: Skip	Not Hispanic or Latino	PMI: Skip	2017-03-23 19:03:00	2019-11-24 11:30:00	3	1	Opioid dependence
1962-06-15	2654448	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2014-04-16 05:00:00	2015-07-07 05:00:00	3	2	Opioid abuse, Opioid dependence
1961-06-15	1811050	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2022-03-04 20:00:00	2022-03-04 20:00:00	1	1	Opioid dependence
1959-06-15	1575027	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	1999-11-12 09:20:00	1999-11-12 09:20:00	1	1	Opioid dependence
1959-06-15	1057161	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2020-01-21 00:00:00	2020-04-03 00:00:00	3	1	Opioid abuse
1978-06-15	1071725	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	2005-04-06 05:00:00	2021-12-30 11:59:59	28	2	Opioid dependence, Opioid abuse
1968-06-15	1093408	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2018-03-02 00:00:00	2021-09-20 00:00:00	5	2	Opioid abuse, Opioid dependence
1965-06-15	3445399	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	2010-04-04 05:00:00	2010-04-04 05:00:00	1	1	Opioid abuse
1964-06-15	1563170	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2014-02-28 09:58:00	2019-12-02 00:00:00	83	3	Continuous opioid dependence, Opioid dependence, Opioid abuse
1955-06-15	3374279	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2013-05-02 00:00:00	2015-12-25 17:16:00	25	2	Opioid dependence, Continuous opioid dependence
1988-06-15	1455974	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2019-02-21 04:34:00	2021-07-24 05:49:00	11	1	Opioid dependence
1960-06-15	1491554	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2006-06-06 06:00:00	2006-06-06 06:00:00	3	1	Opioid dependence
1953-06-15	1520263	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	2017-05-24 05:00:00	2017-05-24 05:00:00	1	1	Opioid abuse
1966-06-15	2419084	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	2016-06-28 00:00:00	2016-06-28 00:00:00	1	1	Opioid dependence
1971-06-15	2281347	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2018-08-30 00:00:00	2021-06-10 00:00:00	8	2	Opioid dependence, Opioid abuse
1951-06-15	2011147	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2013-06-04 00:00:00	2020-08-02 16:28:00	19	2	Opioid dependence, Opioid abuse
1967-06-15	2460367	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2017-06-19 05:00:00	2017-06-25 05:00:00	2	1	Opioid abuse
1978-06-15	1262726	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2020-06-26 03:49:00	2020-06-28 21:12:00	3	1	Opioid abuse
1964-06-15	2647754	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	2008-10-14 05:00:00	2008-10-14 05:00:00	1	1	Opioid dependence
1982-06-15	2438386	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	2016-06-11 04:31:00	2016-06-11 04:31:00	1	1	Opioid abuse
1949-06-15	3202801	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2017-05-28 14:01:00	2017-05-28 14:01:00	1	1	Opioid abuse
1966-06-15	3126984	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2018-10-20 05:00:00	2018-10-20 05:00:00	1	1	Opioid dependence
1962-06-15	2520937	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	2021-12-17 17:58:42	2021-12-17 19:42:00	2	1	Opioid dependence
1953-06-15	2462253	Black or African American	PMI: Skip	Not Hispanic or Latino	PMI: Skip	2019-07-22 20:11:00	2019-12-11 22:36:00	3	1	Opioid dependence
#subet of europeans
case_eur <- case %>%
    filter(race == "White")
case_eur
1979-06-15	1357846	White	Male	Not Hispanic or Latino	Intersex	2015-08-08 12:28:00	2020-09-09 18:59:00	26	3	Opioid abuse, Opioid dependence, Continuous opioid dependence
1958-06-15	3291352	White	Male	Not Hispanic or Latino	None	2007-05-08 00:00:00	2022-03-23 11:59:59	80	2	Opioid dependence, Opioid abuse
1999-06-15	2366246	White	Not man only, not woman only, prefer not to answer, or skipped	Not Hispanic or Latino	None	2018-11-15 17:40:00	2018-11-15 17:40:00	1	1	Opioid abuse
1971-06-15	2726287	White	Male	Not Hispanic or Latino	None	2017-10-26 02:49:21	2021-05-11 06:38:55	27	3	Opioid abuse, Opioid dependence, Opioid dependence in remission
1954-06-15	1728944	White	Gender Identity: Additional Options	Not Hispanic or Latino	None	2015-11-04 00:00:00	2016-03-15 00:00:00	2	1	Continuous opioid dependence
1991-06-15	2153830	White	Male	Not Hispanic or Latino	Male	2012-01-28 00:00:00	2022-04-19 11:59:59	644	5	Opioid dependence in remission, Opioid abuse, Continuous opioid dependence, Episodic opioid dependence, Opioid dependence
1969-06-15	1684205	White	Male	Not Hispanic or Latino	Male	2007-10-01 10:00:00	2015-11-11 13:40:00	143	5	Continuous opioid dependence, Opioid dependence, Opioid dependence in remission, Opioid abuse, Episodic opioid dependence
1977-06-15	2900980	White	Male	Not Hispanic or Latino	Male	2006-10-03 00:00:00	2021-09-22 11:59:59	55	5	Opioid abuse, Episodic opioid dependence, Combined opioid with other drug dependence, continuous, Opioid dependence in remission, Opioid dependence
1978-06-15	1138798	White	Male	Not Hispanic or Latino	Male	2008-05-22 05:00:00	2008-05-22 05:00:00	1	1	Episodic opioid dependence
1949-06-15	3344183	White	Male	Not Hispanic or Latino	Male	2011-05-02 13:25:55	2022-01-12 12:39:50	169	5	Continuous opioid dependence, Opioid dependence in remission, Episodic opioid dependence, Opioid dependence, Opioid abuse
1972-06-15	1592557	White	Male	Not Hispanic or Latino	Male	2018-07-08 00:04:39	2018-12-14 09:54:09	3	1	Opioid abuse
1961-06-15	1999136	White	Male	Not Hispanic or Latino	Male	2014-01-17 03:42:33	2022-01-06 08:18:52	18	2	Opioid dependence, Opioid abuse
1962-06-15	1529191	White	Male	Not Hispanic or Latino	Male	2017-12-21 00:38:02	2019-09-11 21:37:13	6	2	Opioid abuse, Opioid dependence
1954-06-15	2745205	White	Male	Not Hispanic or Latino	Male	2017-10-27 09:01:16	2021-08-26 09:07:23	10	3	Opioid abuse, Opioid dependence in remission, Opioid dependence
1973-06-15	3078363	White	Male	Not Hispanic or Latino	Male	2016-12-06 18:00:00	2019-07-17 09:56:00	5	3	Continuous opioid dependence, Opioid abuse, Opioid dependence
1981-06-15	1700270	White	Male	Not Hispanic or Latino	Male	2009-07-14 11:00:00	2021-04-14 11:00:00	326	3	Opioid dependence in remission, Opioid abuse, Opioid dependence
1958-06-15	3083546	White	Male	Not Hispanic or Latino	Male	2010-06-22 11:30:00	2022-07-01 13:00:00	441	3	Opioid abuse, Opioid dependence, Opioid dependence in remission
1981-06-15	2867402	White	Male	Not Hispanic or Latino	Male	2013-12-20 00:00:00	2018-10-04 00:00:00	144	3	Opioid dependence in remission, Opioid abuse, Opioid dependence
1975-06-15	1708434	White	PMI: Skip	Not Hispanic or Latino	Male	2005-02-04 00:00:00	2021-12-06 11:59:59	133	8	Combined opioid with other drug dependence, episodic, Opioid dependence, Combined opioid with other drug dependence, Combined opioid with other drug dependence in remission, Opioid dependence in remission, Combined opioid with other drug dependence, continuous, Opioid abuse, Continuous opioid dependence
1982-06-15	2433690	White	Male	Not Hispanic or Latino	Male	2019-02-25 06:00:00	2020-03-11 05:00:00	23	2	Opioid dependence, Opioid abuse
1959-06-15	1228109	White	Male	Not Hispanic or Latino	Male	2006-03-06 05:00:00	2017-08-09 05:00:00	24	5	Opioid abuse, Opioid dependence, Combined opioid with other drug dependence, continuous, Combined opioid with other drug dependence, Continuous opioid dependence
1967-06-15	1386430	White	Male	Not Hispanic or Latino	Male	2007-05-07 05:00:00	2022-04-29 05:00:00	47	2	Opioid abuse, Opioid dependence
1982-06-15	3451900	White	Male	Not Hispanic or Latino	Male	2018-07-03 00:31:00	2019-11-18 13:45:00	20	2	Opioid abuse, Opioid dependence
1985-06-15	1195996	White	Male	Not Hispanic or Latino	Male	2014-08-09 20:13:00	2022-05-10 16:23:00	34	3	Opioid abuse, Opioid dependence, Combined opioid with other drug dependence
1990-06-15	8967782	White	Male	Not Hispanic or Latino	Male	2014-11-11 00:00:00	2022-05-13 11:59:59	355	3	Opioid abuse, Opioid dependence, Opioid dependence in remission
1973-06-15	1639244	White	Male	Not Hispanic or Latino	Male	2019-02-20 04:07:12	2019-02-20 04:07:12	1	1	Opioid abuse
1991-06-15	1275228	White	Male	Not Hispanic or Latino	Male	2017-09-21 00:00:00	2017-10-16 00:21:00	5	2	Opioid abuse, Opioid dependence
1986-06-15	3324539	White	Male	Not Hispanic or Latino	Male	2017-03-27 05:00:00	2017-03-27 11:59:59	2	1	Opioid abuse
1959-06-15	2572715	White	Male	Not Hispanic or Latino	Male	2017-02-04 10:30:00	2017-05-06 07:31:00	8	3	Continuous opioid dependence, Opioid abuse, Opioid dependence
1956-06-15	2738037	White	Male	Not Hispanic or Latino	Male	2003-10-30 12:25:00	2021-03-26 00:00:00	77	4	Opioid abuse, Opioid dependence, Opioid dependence in remission, Continuous opioid dependence
⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮
1976-06-15	1707683	White	Female	Not Hispanic or Latino	PMI: Skip	2020-04-08 12:30:00	2022-06-30 14:00:00	50	2	Opioid abuse, Opioid dependence
1988-06-15	2063363	White	Female	Not Hispanic or Latino	PMI: Skip	2007-12-21 12:03:00	2022-05-18 12:05:00	31	4	Continuous opioid dependence, Opioid dependence, Opioid dependence in remission, Opioid abuse
1976-06-15	3387482	White	Female	Not Hispanic or Latino	PMI: Skip	2014-04-08 09:48:00	2021-05-25 00:00:00	18	3	Combined opioid with other drug dependence, Opioid dependence, Opioid abuse
1964-06-15	1597152	White	PMI: Skip	Not Hispanic or Latino	PMI: Skip	2021-04-21 05:00:00	2021-04-21 11:59:59	2	1	Opioid dependence
1971-06-15	3283736	White	Female	Not Hispanic or Latino	PMI: Skip	2021-08-04 07:43:29	2021-08-04 07:43:29	1	1	Opioid dependence
1956-06-15	1431601	White	Female	Not Hispanic or Latino	PMI: Skip	2019-11-23 03:12:09	2019-12-07 08:04:44	2	1	Opioid dependence
1966-06-15	3021788	White	Female	Not Hispanic or Latino	PMI: Skip	2019-07-04 09:17:56	2019-07-04 09:17:56	1	1	Opioid dependence
1961-06-15	1245628	White	Male	Not Hispanic or Latino	PMI: Skip	2016-01-19 00:00:00	2022-04-14 11:59:59	132	3	Opioid abuse, Opioid dependence, Opioid dependence in remission
1981-06-15	2337376	White	Female	Not Hispanic or Latino	PMI: Skip	2013-09-10 05:00:00	2022-06-16 11:59:59	4	2	Opioid abuse, Opioid dependence
1996-06-15	1516054	White	Female	Not Hispanic or Latino	PMI: Skip	2020-03-11 18:37:00	2020-04-01 04:58:00	8	2	Opioid abuse, Opioid dependence
1986-06-15	1389790	White	Female	Not Hispanic or Latino	PMI: Skip	2011-08-24 02:02:00	2021-08-25 07:30:00	14	2	Opioid abuse, Opioid dependence
1960-06-15	3446019	White	Female	Not Hispanic or Latino	PMI: Skip	2013-05-14 06:00:00	2013-09-04 06:00:00	2	1	Opioid abuse
1972-06-15	2466431	White	Female	Not Hispanic or Latino	PMI: Skip	2018-10-04 00:00:00	2022-05-27 11:59:59	131	2	Opioid dependence in remission, Opioid dependence
1956-06-15	1419256	White	PMI: Skip	Not Hispanic or Latino	PMI: Skip	1999-09-22 00:00:00	2017-11-06 09:30:00	107	4	Opioid dependence in remission, Continuous opioid dependence, Opioid dependence, Opioid abuse
1952-06-15	2463877	White	Female	Not Hispanic or Latino	PMI: Skip	2021-11-18 09:25:28	2022-01-12 19:57:12	2	1	Opioid dependence
1958-06-15	1358812	White	Male	Not Hispanic or Latino	PMI: Skip	2015-02-16 17:15:00	2017-06-13 15:00:00	6	1	Opioid dependence
1990-06-15	1624228	White	Male	Not Hispanic or Latino	PMI: Skip	2019-01-30 00:00:00	2019-11-22 11:59:59	2	1	Opioid dependence
1961-06-15	1115857	White	PMI: Skip	Not Hispanic or Latino	PMI: Skip	2019-03-15 00:55:00	2019-03-15 00:55:00	1	1	Opioid dependence
1976-06-15	2554139	White	Female	Not Hispanic or Latino	PMI: Skip	2019-07-15 00:00:00	2021-05-20 00:00:00	19	1	Opioid dependence
1974-06-15	3040387	White	Female	Not Hispanic or Latino	PMI: Skip	2019-10-28 06:19:05	2019-10-28 06:19:05	1	1	Opioid dependence
1955-06-15	5539150	White	Male	Not Hispanic or Latino	PMI: Skip	2012-03-14 00:00:00	2019-01-22 07:55:08	3	1	Opioid dependence
1963-06-15	2247937	White	Female	Not Hispanic or Latino	PMI: Skip	2018-10-19 09:44:05	2019-06-19 00:05:56	4	2	Opioid abuse, Opioid dependence
1949-06-15	1881474	White	Male	Not Hispanic or Latino	PMI: Skip	1999-10-25 08:15:00	2022-01-24 14:00:00	8	2	Episodic opioid dependence, Opioid dependence in remission
1992-06-15	1584596	White	Male	Not Hispanic or Latino	PMI: Skip	2019-10-16 12:00:00	2022-06-13 12:00:00	24	1	Opioid dependence
1944-06-15	1672539	White	Male	Not Hispanic or Latino	PMI: Skip	2003-04-07 09:00:00	2004-10-25 09:00:00	3	1	Opioid dependence
1956-06-15	3387202	White	PMI: Skip	Not Hispanic or Latino	PMI: Skip	2021-01-14 17:16:00	2021-01-14 17:16:00	1	1	Opioid dependence
1975-06-15	3091706	White	Female	Not Hispanic or Latino	PMI: Skip	2021-08-03 00:53:35	2021-08-03 00:53:35	1	1	Opioid dependence in remission
1950-06-15	1369627	White	Female	Not Hispanic or Latino	PMI: Skip	2019-04-02 00:00:00	2022-02-14 00:00:00	5	1	Opioid dependence
1949-06-15	1909727	White	Male	Not Hispanic or Latino	PMI: Skip	2017-02-21 07:15:04	2017-02-21 07:15:04	1	1	Opioid dependence
1965-06-15	1476879	White	Male	Not Hispanic or Latino	PMI: Skip	2018-06-01 09:15:00	2021-09-02 13:35:00	2	1	Opioid dependence
#subset of amr
control_ME <- control_opioid_only %>%
    filter(race == "Black or African American")
control_ME
1969-06-15	1904084	Black or African American	Female	Not Hispanic or Latino	Intersex	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1990-06-15	2291241	Black or African American	Male	Not Hispanic or Latino	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1979-06-15	1659969	Black or African American	Female	Not Hispanic or Latino	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1989-06-15	2969002	Black or African American	Gender Identity: Non Binary	Not Hispanic or Latino	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1967-06-15	1625463	Black or African American	Female	Not Hispanic or Latino	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1990-06-15	2291241	Black or African American	Male	Not Hispanic or Latino	None	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1979-06-15	1659969	Black or African American	Female	Not Hispanic or Latino	I prefer not to answer	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1990-06-15	1420461	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1961-06-15	2944395	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1968-06-15	2125629	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1964-06-15	1833211	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1977-06-15	1804474	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1949-06-15	1389459	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1973-06-15	2607584	Black or African American	Male	Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1982-06-15	1302985	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1955-06-15	1970489	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1953-06-15	1030991	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1959-06-15	1904952	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1965-06-15	3167307	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1968-06-15	3192191	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1990-06-15	3317492	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1982-06-15	3117875	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1960-06-15	4572006	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1941-06-15	1450042	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1953-06-15	2508429	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1954-06-15	1516077	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1960-06-15	1361791	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1996-06-15	2674886	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1960-06-15	1900165	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1986-06-15	1787230	Black or African American	Male	Not Hispanic or Latino	Male	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮	⋮
1950-06-15	3380397	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1939-06-15	2516739	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1953-06-15	2745603	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1963-06-15	2214747	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1955-06-15	1698831	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1960-06-15	3368483	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1971-06-15	1948514	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1956-06-15	1501928	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1956-06-15	1502386	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1959-06-15	2530084	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1959-06-15	1255433	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1976-06-15	3509733	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1957-06-15	1940858	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1960-06-15	2651610	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1976-06-15	1079899	Black or African American	Male	Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1976-06-15	2069148	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1952-06-15	1100899	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1967-06-15	2366378	Black or African American	PMI: Skip	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1993-06-15	2929757	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1961-06-15	1372851	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1947-06-15	3004960	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1968-06-15	2674706	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1964-06-15	3290337	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1957-06-15	1940858	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1961-06-15	2006174	Black or African American	PMI: Skip	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1963-06-15	1321710	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1962-06-15	9993483	Black or African American	Female	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1967-06-15	1701811	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
1961-06-15	2185486	Black or African American	PMI: Skip	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Street Opioids Use	1
1960-06-15	2651610	Black or African American	Male	Not Hispanic or Latino	PMI: Skip	Lifestyle	Recreational Drug Use: Which Drugs Used	Which Drugs Used: Prescription Opioids Use	1
```

```
write_csv(case_ME, TEST_CASE_FILENAME)
write_tsv(case_ME  %>%
              mutate(
                  IID = person_id,
                  FID = IID
              ) %>%
              select(FID, IID) %>%
              distinct(),
          TEST_CASE_ID_FILENAME)
write_csv(control_ME, TEST_CONTROL_FILENAME)
write_tsv(control_ME  %>%
              mutate(
                  IID = person_id,
                  FID = IID
              ) %>%
              select(FID, IID) %>%
              distinct(),
          TEST_CONTROL_ID_FILENAME)
```

```
system(str_glue('gsutil cp {TEST_CASE_FILENAME} {TEST_CASE_ID_FILENAME} {TEST_CONTROL_FILENAME} {TEST_CONTROL_ID_FILENAME} {DESTINATION}'), intern = T)
system(str_glue('gsutil ls -lh {DESTINATION}'), intern = T)
```


```
' 934.5 KiB  2023-08-11T15:58:40Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/case.csv'
' 96.01 KiB  2023-08-11T15:58:40Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/case_ids.tsv'
'  3.99 MiB  2023-08-11T15:58:43Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/control.csv'
' 319.9 KiB  2023-08-11T15:58:43Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/control_ids.tsv'
'288.64 KiB  2023-08-11T15:58:48Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/test_case.csv'
'  27.8 KiB  2023-08-11T15:58:48Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/test_case_ids.tsv'
'803.52 KiB  2023-08-11T15:58:49Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/test_control.csv'
' 59.24 KiB  2023-08-11T15:58:49Z  gs://fc-secure-c8b84a93-2a47-44c5-bd2c-56358bb9a84e/data/aou/pheno/20230811/test_control_ids.tsv'
'TOTAL: 8 objects, 6775778 bytes (6.46 MiB)'
```


```
devtools::session_info()
```


```
#plot demographic data
race_counts <- select(case, race) %>% group_by(race)
race_counts <- count(race_counts, race)
colnames(race_counts) <- c('race','count')
par(las = 1) # make label text perpendicular to axis
par(mar=c(3,15,3,1)) # increase y-axis margin


race_counts
barplot(race_counts$count, main="Race and Ancestry Distr for Case", horiz = TRUE, 
        names.arg = race_counts$race, cex.names = 0.8)
```

```
Asian	27
Black or African American	1779
I prefer not to answer	67
Middle Eastern or North African	13
More than one population	108
Native Hawaiian or Other Pacific Islander	9
None Indicated	809
None of these	99
PMI: Skip	158
White	3075
```

```
control_counts <- select(control, race) %>% group_by(race)
control_counts <- count(control_counts, race)
colnames(control_counts) <- c('race','count')
par(las = 1) # make label text perpendicular to axis
par(mar=c(3,15,3,1)) # increase y-axis margin


control_counts
barplot(control_counts$count, main="Race and Ancestry Distr for Control", horiz = TRUE, 
        names.arg = control_counts$race, cex.names = 0.8)
```


```
Asian	1210
Black or African American	13087
I prefer not to answer	622
Middle Eastern or North African	305
More than one population	2353
Native Hawaiian or Other Pacific Islander	124
None Indicated	9244
None of these	1363
PMI: Skip	1709
White	60334
```
