# Graves’ Disease Patient Identification Through Steroid Utilization Analysis

## Project Overview

This project uses SQL-based healthcare claims analytics to identify patients with Graves’ disease who demonstrate evidence of sustained corticosteroid utilization during a defined observation period.

The analysis processes prescription claims, standardizes National Drug Codes (NDCs), calculates daily prednisone-equivalent dosage, removes duplicate claims, and constructs treatment episodes to identify patients meeting a predefined duration threshold.

The final output is a distinct list of patient identifiers (`PTID`) for patients with at least one qualifying steroid treatment episode lasting 30 days or longer.

## Business Objective

The objective is to develop a reproducible patient identification algorithm that can support healthcare analytics and pharmaceutical research by:

- Identifying patients with qualifying corticosteroid prescription claims.
- Standardizing drug identifiers and calculating prednisone-equivalent daily dosage.
- Reconstructing treatment episodes from prescription claim dates and days supplied.
- Applying a minimum treatment duration criterion to identify sustained steroid utilization.
- Generating a patient-level cohort for downstream analysis.

## Data Sources

The analysis uses the following datasets:

| Dataset | Purpose |
|---|---|
| `final_cohort_2026_updated` | Defines the eligible patient cohort and provides first diagnosis dates and eligibility end dates. |
| `argenx_graves_202605_clm_rx` | Provides prescription claims, including fill dates, NDCs, days supplied, dispensed quantities, and transaction types. |
| `steroids_pred_eq` | Provides the steroid drug mapping and prednisone-equivalent strength required for dosage calculations. |

**Key data elements:** Patient ID (`PTID`), prescription fill date, National Drug Code (`NDC`), days supplied (`DoS`), dispensed quantity, first diagnosis date, eligibility end date, and prednisone-equivalent strength.

*Note: The SQL references environment-specific schemas and tables. Access to the underlying datasets is required to execute the analysis.*

## Methodology

### 1. Define the Patient Cohort

The analysis begins with the predefined patient cohort in `final_cohort_2026_updated`.

The cohort supplies the patient identifiers and the dates required to establish the observation window for each patient.

### 2. Extract and Standardize Prescription Claims

Prescription claims are retrieved for patients in the defined cohort.

The following filters and transformations are applied:

- Retain claims with a transaction type of `PAID`.
- Convert prescription fill dates to date format.
- Standardize NDC values to an 11-digit format by adding leading zeros where necessary.
- Extract days supplied and dispensed quantity for subsequent utilization calculations.

### 3. Apply the Observation Window

Prescription claims are restricted to the period beginning 365 days before the patient's first diagnosis date and ending on the patient's eligibility end date.

This creates a patient-specific observation window for the steroid utilization analysis.

### 4. Clean and Deduplicate Claims

Claims are filtered to retain records with valid, positive days supplied and non-null dispensed quantities.

Where multiple claims exist for the same patient, fill date, and NDC, a priority rule selects one record based on:

1. Highest days supplied.
2. Lowest dispensed quantity when days supplied is tied.

The analysis then maps the claims to the steroid reference table using standardized NDCs and retains records with positive prednisone-equivalent strength.

### 5. Calculate Daily Prednisone-Equivalent Dosage

Daily dosage is estimated using the dispensed quantity, days supplied, and prednisone-equivalent strength.

The calculation implemented in the SQL is:

`Daily dose = ROUND(Quantity dispensed / Days supplied, 2) × Prednisone-equivalent strength`

Only claims with a calculated daily dosage of **at least 7.5** are retained.

This threshold is applied to the calculated prednisone-equivalent dosage; the units and clinical interpretation depend on the source drug mapping.

### 6. Construct Steroid Treatment Episodes

The qualifying claims are organized chronologically for each patient.

The episode-building logic uses prescription dates and days supplied to estimate treatment coverage and identify gaps between successive claims.

Key rules include:

- Calculate the expected end of coverage for the previous prescription using its fill date and days supplied.
- Calculate the gap between expected coverage and the next prescription fill date.
- Set negative gaps to zero, treating overlapping coverage as continuous.
- Apply a grace-period parameter of **1 day** to determine whether claims belong to the same treatment episode.
- Assign episode numbers using cumulative window calculations.
- Derive episode start dates, end dates, and durations.
- Cap episode end dates at the patient's eligibility end date.

This process produces patient-level steroid treatment episodes based on the coverage assumptions encoded in the SQL.

### 7. Apply the Minimum Episode Duration Criterion

The final step retains patients with at least one steroid treatment episode satisfying:

`Episode length ≥ 30 days`

The episode length is calculated as the difference, in calendar days, between the episode start and end dates.

The query returns distinct patient identifiers (`PTID`) meeting this criterion.

## Workflow Summary

1. Define the patient cohort.
2. Extract paid prescription claims.
3. Standardize NDCs and apply patient-specific observation windows.
4. Clean and deduplicate claims.
5. Map steroid products to prednisone-equivalent strengths.
6. Calculate daily dosage and apply the dosage threshold.
7. Construct treatment episodes using days supplied and a one-day grace period.
8. Identify patients with qualifying episodes of at least 30 days.
9. Return a distinct patient-level output.

## Key Business Rules

| Parameter | Rule implemented |
|---|---|
| Claim status | Paid claims only |
| Observation start | 365 days before first diagnosis |
| Observation end | Patient eligibility end date |
| NDC standardization | 11-digit format |
| Days supplied | Greater than zero and non-null |
| Dispensed quantity | Non-null |
| Prednisone-equivalent strength | Greater than zero |
| Daily dosage threshold | At least 7.5 |
| Episode grace period | 1 day |
| Minimum episode duration | At least 30 days |
| Final output | Distinct patient identifiers |

## Expected Output

The output is a patient-level list containing distinct `PTID` values for individuals with at least one treatment episode that meets the specified dosage, episode construction, and duration criteria.

This cohort can serve as an input to subsequent analyses, such as:

- Patient funnel development.
- Treatment utilization and persistence analysis.
- Healthcare resource utilization studies.
- Cohort characterization and segmentation.
- Downstream pharmaceutical market research.

The output identifies patients meeting the algorithm's operational criteria. It does not independently establish clinical appropriateness, adherence, or confirmed continuous medication use.

## Technical Implementation

**Technology:** SQL

**Key SQL techniques:**
- Common Table Expressions (CTEs)
- Joins and subqueries
- Date arithmetic and date-difference calculations
- String manipulation and NDC standardization
- Conditional filtering using `CASE WHEN`
- Window functions, including `LAG()`, `LEAD()`, `ROW_NUMBER()`, and cumulative `SUM()`
- Aggregation using `MIN()` and `MAX()`
- Patient-level deduplication

The query uses SQL functions such as `DATE_ADD`, `DATE_DIFF`, and `LEAST`, consistent with the date-function conventions of the target SQL engine.

## Important Considerations

- **Claims-based inference:** Prescription claims indicate billed dispensing activity and do not confirm that medication was taken as prescribed.
- **Dosage assumptions:** The calculation depends on the accuracy and consistency of the steroid reference table and its prednisone-equivalent strength definitions.
- **Episode definitions:** The one-day grace period and days-supplied calculations determine how treatment continuity is inferred.
- **Eligibility censoring:** Episodes are capped at the eligibility end date, which can affect calculated treatment duration.
- **Validation:** The episode construction, boundary conditions, and dosage calculations should be validated against the study protocol before using the resulting cohort for clinical or commercial conclusions.
- **Privacy:** Patient-level identifiers and source healthcare data should remain within authorized environments and should not be committed to a public repository.

## Disclaimer

This repository documents a healthcare claims analytics workflow for research and analytical purposes. Patient identification depends on the cohort definition, source data quality, reference mappings, and operational rules implemented in the SQL. Results should be interpreted in accordance with the applicable study protocol and data-use requirements.
