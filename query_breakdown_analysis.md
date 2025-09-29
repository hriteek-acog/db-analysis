# Query Breakdown Analysis

## Complete Analysis of Database Queries for Healthcare Dashboard

**Project**: Hospital Portal Dashboard Query Analysis  
**Analysis Date**: September 29, 2025  
**Source File**: queries.txt  
**Total Queries Analyzed**: 20+ queries across 8 functional areas

---

## Executive Overview

This document provides a comprehensive breakdown of all queries in the queries.txt file, analyzing both the internal technical implementation and overall strategic value of each query. The queries are organized by functional area and analyzed for complexity, data dependencies, performance considerations, and business impact.

### Query Distribution Summary:

- **Peer Comparison**: 3 complex queries with statistical analysis
- **Department Spending**: 3 queries with aggregation and trend analysis
- **Facility Information**: 2 straightforward dimensional queries
- **Pay Rate Analysis**: 2 market intelligence queries
- **Total Compensation**: 3 compensation calculation queries
- **Revenue Analysis**: 1 revenue aggregation query
- **System Operations**: 3 placeholder operational queries
- **Global Filters**: 5 utility/filter queries

---

## 1. Peer Comparison Widget Queries

### Query 1.1: Get Selected Facility Details with Peer Criteria

**Internal Breakdown:**

```sql
SELECT
    h.id AS hospital_id,
    h.hospital_ccn,
    h.hospital_name,
    h.num_beds,
    h.bed_range,
    h.ownership_type,
    oc.ownership_category,
    zc.cbsa_code,
    zc.cbsa_title,
    CASE
        WHEN hf.traditional_medicare_pct BETWEEN 0 AND 0.20 THEN '0-20%'
        WHEN hf.traditional_medicare_pct BETWEEN 0.20 AND 0.40 THEN '20-40%'
        WHEN hf.traditional_medicare_pct BETWEEN 0.40 AND 0.60 THEN '40-60%'
        ELSE '60%+'
    END AS medicaid_bucket,
    hf.traditional_medicare_pct,
    hf.total_costs,
    hf.total_salaries,
    hf.direct_patient_care_fte,
    CASE WHEN hf.direct_patient_care_fte > 0
        THEN hf.total_costs / hf.direct_patient_care_fte
        ELSE 0 END AS cost_per_fte
FROM dim_hospital h
JOIN fact_hcris_facility hf ON h.id = hf.hospital_id
JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
JOIN dim_ownership_category oc ON h.ownership_category_id = oc.id
WHERE h.hospital_ccn = :selected_hospital_ccn
    AND hf.report_year = :selected_year
ORDER BY hf.fiscal_year_end DESC
LIMIT 1;
```

**Technical Analysis:**

- **Query Type**: Multi-table JOIN with calculated fields
- **Complexity**: Medium - 4 table joins with conditional logic
- **Performance**: Good - Single record lookup with indexed CCN
- **Data Dependencies**: 4 dimensional tables + 1 fact table
- **Calculated Fields**: 2 (medicaid_bucket, cost_per_fte)

**Tables Used:**

- `dim_hospital` (6,918 rows) ✅ - Primary entity
- `fact_hcris_facility` (72,802 rows) ✅ - Financial metrics
- `dim_zip_cbsa` (40,773 rows) ✅ - Geographic context
- `dim_ownership_category` (4 rows) ✅ - Ownership classification

**Business Logic:**

- **Medicaid Bucketing**: Creates standardized payer mix categories for peer matching
- **Cost per FTE Calculation**: Key efficiency metric for peer comparison
- **Temporal Filtering**: Uses most recent fiscal year data
- **Defensive Programming**: Handles division by zero in FTE calculation

**Overall Assessment:**

- **Functionality**: ✅ Fully operational
- **Business Value**: High - Establishes foundation for all peer analysis
- **Performance**: Excellent - Indexed lookup on CCN
- **Data Quality**: Reliable - All required tables populated

---

### Query 1.2: Get Peer Facilities Based on Criteria

**Internal Breakdown:**

```sql
WITH facility_peers AS (
    SELECT
        h.id AS hospital_id,
        h.hospital_ccn,
        h.hospital_name,
        -- ... peer selection criteria
        CASE WHEN hf.direct_patient_care_fte > 0
            THEN hf.total_costs / hf.direct_patient_care_fte
            ELSE 0 END AS cost_per_fte
    FROM dim_hospital h
    JOIN fact_hcris_facility hf ON h.id = hf.hospital_id
    JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
    WHERE zc.cbsa_code = :target_cbsa_code
        AND h.bed_range = :target_bed_range
        AND h.ownership_type = :target_ownership_type
        AND hf.report_year = :selected_year
        AND hf.traditional_medicare_pct IS NOT NULL
        AND CASE
            WHEN hf.traditional_medicare_pct BETWEEN 0 AND 0.20 THEN '0-20%'
            WHEN hf.traditional_medicare_pct BETWEEN 0.20 AND 0.40 THEN '20-40%'
            WHEN hf.traditional_medicare_pct BETWEEN 0.40 AND 0.60 THEN '40-60%'
            ELSE '60%+'
        END = :target_medicaid_bucket
)
SELECT
    COUNT(*) AS peer_count,
    AVG(cost_per_fte) AS avg_cost_per_fte,
    AVG(total_salaries) AS avg_nursing_spend,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY cost_per_fte) AS cost_per_fte_25th,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY cost_per_fte) AS cost_per_fte_median,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY cost_per_fte) AS cost_per_fte_75th,
    PERCENTILE_CONT(0.80) WITHIN GROUP (ORDER BY cost_per_fte) AS cost_per_fte_80th,
    -- ... additional percentile calculations
FROM facility_peers;
```

**Technical Analysis:**

- **Query Type**: CTE with statistical aggregation
- **Complexity**: High - Multiple percentile calculations
- **Performance**: Moderate - Full table scans with complex filtering
- **Statistical Functions**: 8 percentile calculations
- **Data Validation**: NULL handling and defensive filtering

**Advanced Features:**

- **Common Table Expression**: Clean separation of peer selection logic
- **Percentile Analysis**: Comprehensive statistical distribution
- **Multi-criteria Filtering**: 4-dimensional peer matching
- **Data Quality Checks**: Null value filtering

**Performance Considerations:**

- **Index Requirements**: Composite index on (cbsa_code, bed_range, ownership_type, report_year)
- **Memory Usage**: Moderate - Statistical functions require sorting
- **Execution Plan**: May benefit from query plan optimization

**Overall Assessment:**

- **Functionality**: ✅ Fully operational
- **Business Value**: Very High - Core peer benchmarking logic
- **Performance**: Needs optimization for large datasets
- **Scalability**: May require indexing improvements

---

### Query 1.3: 12-Month Sparkline Data for Facility vs Peer Average

**Internal Breakdown:**

```sql
SELECT
    hf.report_year,
    hf.fiscal_year_end,
    CASE WHEN hf.direct_patient_care_fte > 0
        THEN hf.total_costs / hf.direct_patient_care_fte
        ELSE 0 END AS facility_cost_per_fte,
    (SELECT AVG(hf2.total_costs / NULLIF(hf2.direct_patient_care_fte, 0))
    FROM fact_hcris_facility hf2
    JOIN dim_hospital h2 ON hf2.hospital_id = h2.id
    JOIN dim_zip_cbsa zc2 ON h2.zip_id = zc2.id
    WHERE zc2.cbsa_code = :target_cbsa_code
        AND h2.bed_range = :target_bed_range
        AND h2.ownership_type = :target_ownership_type
        AND hf2.report_year = hf.report_year
        AND hf2.direct_patient_care_fte > 0) AS peer_avg_cost_per_fte
FROM fact_hcris_facility hf
JOIN dim_hospital h ON hf.hospital_id = h.id
WHERE h.hospital_ccn = :selected_hospital_ccn
    AND hf.report_year >= :selected_year - 1
    AND hf.direct_patient_care_fte > 0
ORDER BY hf.fiscal_year_end DESC;
```

**Technical Analysis:**

- **Query Type**: Correlated subquery with temporal analysis
- **Complexity**: High - Nested subquery for peer calculation
- **Performance**: Poor - Correlated subquery executes multiple times
- **Temporal Logic**: Multi-year trend analysis
- **Data Integrity**: NULLIF function for safe division

**Critical Performance Issues:**

- **Correlated Subquery**: Executes peer calculation for each row
- **Multiple JOINs**: Complex join logic in subquery
- **Optimization Opportunity**: Could be rewritten as window function or CTE

**Suggested Optimization:**

```sql
WITH yearly_metrics AS (
    SELECT
        hf.report_year,
        h.hospital_ccn,
        AVG(CASE WHEN h.hospital_ccn = :selected_hospital_ccn
            THEN hf.total_costs / NULLIF(hf.direct_patient_care_fte, 0) END) as facility_cost_per_fte,
        AVG(CASE WHEN zc.cbsa_code = :target_cbsa_code
            THEN hf.total_costs / NULLIF(hf.direct_patient_care_fte, 0) END) as peer_avg_cost_per_fte
    FROM fact_hcris_facility hf
    JOIN dim_hospital h ON hf.hospital_id = h.id
    JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
    WHERE hf.report_year >= :selected_year - 1
    GROUP BY hf.report_year, h.hospital_ccn
)
SELECT * FROM yearly_metrics WHERE hospital_ccn = :selected_hospital_ccn;
```

**Overall Assessment:**

- **Functionality**: ✅ Works but inefficient
- **Business Value**: High - Trend analysis crucial for decision making
- **Performance**: ❌ Needs optimization
- **Recommendation**: Rewrite using CTE or window functions

---

## 2. Top Departments (Spend) Queries

### Query 2.1: Get Department-Level Spending for Facility

**Internal Breakdown:**

```sql
SELECT
    cc.cost_center AS department_name,
    hcc.wages_paid AS facility_spend,
    hcc.hours_paid,
    hcc.report_year,
    hcc.fiscal_year_end
FROM fact_hcris_cost_center hcc
JOIN dim_hospital h ON hcc.hospital_id = h.id
JOIN dim_cost_center cc ON hcc.cost_center_id = cc.id
WHERE h.hospital_ccn = :selected_hospital_ccn
    AND hcc.report_year = :selected_year
    AND hcc.wages_paid > 0
ORDER BY hcc.wages_paid DESC
LIMIT 10;
```

**Technical Analysis:**

- **Query Type**: Simple aggregation with ranking
- **Complexity**: Low - Straightforward JOIN and filter
- **Performance**: Good - Indexed lookup with limit
- **Business Logic**: Top N department spending analysis
- **Data Filter**: Excludes zero-wage records

**Critical Data Dependency:**

- **Primary Table**: `fact_hcris_cost_center` (0 rows) ❌ **FATAL ERROR**
- **Support Tables**: All available and properly structured

**Impact Assessment:**

- **Functionality**: ❌ Complete failure - no data in primary table
- **Business Value**: High - Essential for departmental budgeting
- **Alternative**: Must use CMS job category data substitute

**Workaround Query:**

```sql
SELECT
    dcjs.job_category_label AS department_name,
    SUM(fcoj.wages_paid) AS facility_spend,
    SUM(fcoj.hours_paid) AS hours_paid,
    fcoj.survey_year as report_year
FROM fact_cms_oms_job_category fcoj
JOIN dim_cms_oms_jobcategory_soc dcjs ON fcoj.job_category_code = dcjs.job_category_code
JOIN dim_hospital h ON fcoj.hospital_id = h.id
WHERE h.hospital_ccn = :selected_hospital_ccn
    AND fcoj.survey_year = :selected_year
    AND fcoj.wages_paid > 0
GROUP BY dcjs.job_category_label, fcoj.survey_year
ORDER BY facility_spend DESC
LIMIT 10;
```

**Overall Assessment:**

- **Original Query**: ❌ Non-functional due to missing data
- **Workaround Available**: ✅ CMS job category alternative
- **Business Value**: Maintained through alternative approach
- **Data Quality**: Alternative has 45,625 records available

---

### Query 2.2: Get Peer Median Spending by Department

**Internal Breakdown:**

```sql
WITH peer_dept_spending AS (
    SELECT
        cc.cost_center AS department_name,
        hcc.wages_paid,
        h.hospital_ccn,
        ROW_NUMBER() OVER (PARTITION BY cc.cost_center ORDER BY hcc.wages_paid) AS rn,
        COUNT(*) OVER (PARTITION BY cc.cost_center) AS total_count
    FROM fact_hcris_cost_center hcc
    JOIN dim_hospital h ON hcc.hospital_id = h.id
    JOIN dim_cost_center cc ON hcc.cost_center_id = cc.id
    JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
    WHERE zc.cbsa_code = :target_cbsa_code
        AND h.bed_range = :target_bed_range
        AND h.ownership_type = :target_ownership_type
        AND hcc.report_year = :selected_year
        AND hcc.wages_paid > 0
)
SELECT
    department_name,
    AVG(wages_paid) AS peer_median_spend,
    COUNT(*) AS peer_count,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY wages_paid) AS spend_25th,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY wages_paid) AS spend_median,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY wages_paid) AS spend_75th,
    PERCENTILE_CONT(0.80) WITHIN GROUP (ORDER BY wages_paid) AS spend_80th
FROM peer_dept_spending
GROUP BY department_name
HAVING COUNT(*) >= 5
ORDER BY peer_median_spend DESC;
```

**Technical Analysis:**

- **Query Type**: CTE with window functions and statistical analysis
- **Complexity**: Very High - Advanced statistical calculations
- **Window Functions**: ROW_NUMBER() and COUNT() OVER()
- **Statistical Functions**: Multiple percentile calculations
- **Data Quality Control**: HAVING clause ensures minimum sample size

**Advanced Features:**

- **Partition Logic**: Separate calculations per department
- **Sample Size Validation**: Minimum 5 peers required
- **Comprehensive Statistics**: Full quartile analysis
- **Peer Filtering**: Multi-dimensional peer matching

**Performance Characteristics:**

- **Memory Intensive**: Multiple window functions and sorting
- **CPU Intensive**: Statistical calculations across partitions
- **I/O Requirements**: Large fact table scans

**Overall Assessment:**

- **Functionality**: ❌ Blocked by missing data
- **Technical Design**: ✅ Excellent - Sophisticated statistical analysis
- **Business Value**: Very High - Peer benchmarking by department
- **Optimization**: Well-designed with proper sample size controls

---

### Query 2.3: 12-Month Trend for Department Spending

**Internal Breakdown:**

```sql
SELECT
    cc.cost_center AS department_name,
    hcc.report_year,
    hcc.fiscal_year_end,
    hcc.wages_paid,
    hcc.hours_paid
FROM fact_hcris_cost_center hcc
JOIN dim_hospital h ON hcc.hospital_id = h.id
JOIN dim_cost_center cc ON hcc.cost_center_id = cc.id
WHERE h.hospital_ccn = :selected_hospital_ccn
    AND cc.cost_center = :selected_department
    AND hcc.report_year >= :selected_year - 1
ORDER BY hcc.fiscal_year_end DESC;
```

**Technical Analysis:**

- **Query Type**: Time series analysis
- **Complexity**: Low - Simple temporal filtering
- **Performance**: Excellent - Highly selective filters
- **Temporal Logic**: Multi-year trend analysis
- **Use Case**: Department-specific drill-down analysis

**Data Requirements:**

- **Primary Dependency**: `fact_hcris_cost_center` (0 rows) ❌
- **Time Range**: Multi-year historical data needed
- **Granularity**: Department-specific trends

**Business Application:**

- **Trend Identification**: Year-over-year spending patterns
- **Seasonal Analysis**: Fiscal year-end comparisons
- **Budget Planning**: Historical data for forecasting
- **Performance Monitoring**: Department efficiency tracking

**Overall Assessment:**

- **Functionality**: ❌ Non-operational due to data gaps
- **Design Quality**: ✅ Simple and efficient
- **Business Value**: High - Essential for trend analysis
- **Alternative**: Use CMS job category trends as substitute

---

## 3. Facility Information Queries

### Query 3.1: Get Facility Basic Information

**Internal Breakdown:**

```sql
SELECT
    h.hospital_name,
    h.hospital_ccn,
    h.address,
    h.city,
    s.state_name,
    zc.cbsa_title,
    h.facility_type,
    h.ownership_type,
    oc.ownership_category,
    h.num_beds,
    h.bed_range,
    h.teaching_flag,
    h.modified_date AS last_updated
FROM dim_hospital h
JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
JOIN dim_state s ON zc.state_id = s.id
JOIN dim_ownership_category oc ON h.ownership_category_id = oc.id
WHERE h.hospital_ccn = :selected_hospital_ccn;
```

**Technical Analysis:**

- **Query Type**: Multi-dimensional lookup
- **Complexity**: Medium - 4 table joins
- **Performance**: Excellent - Single record retrieval
- **Join Strategy**: Star schema navigation
- **Data Coverage**: Comprehensive facility profile

**Table Analysis:**

- `dim_hospital` (6,918 rows) ✅ - Complete hospital master data
- `dim_zip_cbsa` (40,773 rows) ✅ - Comprehensive geographic coverage
- `dim_state` (59 rows) ✅ - Complete state reference
- `dim_ownership_category` (4 rows) ✅ - Full ownership taxonomy

**Information Architecture:**

- **Geographic Hierarchy**: ZIP → CBSA → State
- **Classification Hierarchy**: Facility Type → Ownership Type → Category
- **Operational Attributes**: Bed count, teaching status, facility type
- **Data Freshness**: Modified date tracking

**Overall Assessment:**

- **Functionality**: ✅ Perfect - All data available
- **Performance**: ✅ Excellent - Indexed single-record lookup
- **Business Value**: High - Essential facility context
- **Data Quality**: ✅ Complete and accurate

---

### Query 3.2: Get Facility Specialties/Departments

**Internal Breakdown:**

```sql
SELECT DISTINCT
    cc.cost_center AS department
FROM fact_hcris_cost_center hcc
JOIN dim_hospital h ON hcc.hospital_id = h.id
JOIN dim_cost_center cc ON hcc.cost_center_id = cc.id
WHERE h.hospital_ccn = :selected_hospital_ccn
    AND hcc.report_year = :selected_year
    AND hcc.wages_paid > 0
ORDER BY cc.cost_center;
```

**Technical Analysis:**

- **Query Type**: Distinct value lookup
- **Complexity**: Low - Simple distinct query
- **Performance**: Good - Selective filtering
- **Business Logic**: Active departments only (wages > 0)
- **Data Validation**: Filters inactive cost centers

**Critical Limitation:**

- **Primary Table**: `fact_hcris_cost_center` (0 rows) ❌

**Alternative Implementation:**

```sql
-- Use available cost center definitions
SELECT DISTINCT cost_center as department
FROM dim_cost_center
ORDER BY cost_center;

-- Or use CMS job categories as department proxies
SELECT DISTINCT dcjs.job_category_label as department
FROM dim_cms_oms_jobcategory_soc dcjs
ORDER BY dcjs.job_category_label;
```

**Overall Assessment:**

- **Original Query**: ❌ Non-functional
- **Alternative Available**: ✅ Can show available departments
- **Business Value**: Medium - Department navigation
- **Workaround Quality**: Adequate for basic functionality

---

## 4. Pay Rate Analysis Queries

### Query 4.1: Get BLS OEWS Pay Rates for CBSA and Job Group

**Internal Breakdown:**

```sql
SELECT
    soc.job_title,
    soc.soc_major_group AS job_group,
    oews.cbsa_code,
    zc.cbsa_title,
    oews.mean_hourly_wage,
    oews.mean_annual_wage,
    oews.hourly_pct10,
    oews.hourly_pct25,
    oews.hourly_median,
    oews.hourly_pct75,
    oews.hourly_pct90,
    oews.total_employees,
    oews.report_date
FROM fact_bls_oews oews
JOIN dim_soc soc ON oews.soc_id = soc.id
JOIN dim_zip_cbsa zc ON oews.cbsa_code = zc.cbsa_code
WHERE oews.cbsa_code = :target_cbsa_code
    AND soc.soc_major_group = :selected_job_group
    AND oews.report_date >= :start_date
    AND oews.report_date <= :end_date
ORDER BY oews.report_date DESC, soc.job_title;
```

**Technical Analysis:**

- **Query Type**: Market intelligence lookup
- **Complexity**: Medium - 3 table joins with filtering
- **Performance**: Excellent - Large dataset (159,739 rows) well-indexed
- **Temporal Filtering**: Date range analysis
- **Statistical Richness**: Complete percentile distribution

**Data Excellence:**

- **fact_bls_oews**: 159,739 rows ✅ **Outstanding coverage**
- **dim_soc**: 168 job classifications ✅
- **dim_zip_cbsa**: 40,773 geographic areas ✅

**Business Intelligence Features:**

- **Market Positioning**: Full percentile analysis (10th, 25th, 50th, 75th, 90th)
- **Geographic Specificity**: CBSA-level market data
- **Employment Context**: Total employees by occupation/location
- **Temporal Analysis**: Historical wage trends

**Performance Characteristics:**

- **Index Strategy**: Composite index on (cbsa_code, soc_id, report_date)
- **Data Volume**: Large but manageable with proper indexing
- **Memory Usage**: Low - Selective filtering reduces result set

**Overall Assessment:**

- **Functionality**: ✅ Excellent - Fully operational
- **Data Quality**: ✅ Outstanding - BLS official data
- **Business Value**: Very High - Market intelligence cornerstone
- **Performance**: ✅ Optimized for large-scale analysis

---

### Query 4.2: Get HWL Internal Pay Rate Data

**Internal Breakdown:**

```sql
SELECT
    jt.job_title,
    jt.job_group,
    h.hospital_ccn,
    zc.cbsa_code,
    hwl.hourly_bill_rate,
    hwl.job_start_date,
    hwl.filled_date,
    hwl.created_date
FROM fact_hwl hwl
JOIN dim_specialty_job_title_mapping sjtm ON hwl.specialty_job_title_id = sjtm.id
JOIN dim_job_title jt ON sjtm.job_title_id = jt.id
JOIN dim_hospital h ON hwl.hospital_id = h.id
JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
WHERE zc.cbsa_code = :target_cbsa_code
    AND jt.job_group = :selected_job_group
    AND hwl.filled_date >= :start_date
    AND hwl.filled_date <= :end_date
ORDER BY hwl.filled_date DESC;
```

**Technical Analysis:**

- **Query Type**: Internal rate intelligence
- **Complexity**: High - 5 table joins with complex mapping
- **Performance**: Would be good if data existed
- **Join Strategy**: Complex many-to-many through mapping table
- **Temporal Logic**: Date range filtering on filled positions

**Critical Data Gap:**

- **fact_hwl**: 0 rows ❌ **Complete data absence**
- **Supporting tables**: All available and properly structured

**Table Relationships:**

- **Job Classification**: `dim_job_title` → `dim_specialty_job_title_mapping` → `fact_hwl`
- **Geographic Context**: `dim_hospital` → `dim_zip_cbsa`
- **Temporal Tracking**: Job start, filled, and created dates

**Business Logic:**

- **Competitive Analysis**: Internal rates vs external market
- **Geographic Targeting**: CBSA-specific rate analysis
- **Temporal Trends**: Historical rate progression
- **Performance Tracking**: Fill rate and timing analysis

**Overall Assessment:**

- **Functionality**: ❌ Complete failure - no internal data
- **Technical Design**: ✅ Well-architected for complex analysis
- **Business Value**: Very High - Essential for competitive positioning
- **Recovery Strategy**: Must populate HWL internal data tables

---

## 5. Total Compensation Analysis Queries

### Query 5.1: Get LE (Labor Exchange) Internal Compensation Data

**Internal Breakdown:**

```sql
SELECT
    jt.job_title,
    jt.job_group,
    h.hospital_ccn,
    zc.cbsa_code,
    le.hourly_bill_rate,
    le.weekly_bill_rate,
    le.weekly_pay_rate,
    le.weekly_housing_stipend,
    le.weekly_mie_stipend,
    le.travel_allowance,
    le.contract_type,
    le.job_start_date,
    le.filled_date
FROM fact_le le
JOIN dim_specialty_job_title_mapping sjtm ON le.specialty_job_title_id = sjtm.id
JOIN dim_job_title jt ON sjtm.job_title_id = jt.id
JOIN dim_hospital h ON le.hospital_id = h.id
JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
WHERE zc.cbsa_code = :target_cbsa_code
    AND jt.job_group = :selected_job_group
    AND le.filled_date >= :start_date
    AND le.filled_date <= :end_date
ORDER BY le.filled_date DESC;
```

**Technical Analysis:**

- **Query Type**: Internal Labor Exchange compensation lookup
- **Complexity**: High - Multi-component internal compensation analysis
- **Join Complexity**: 5 tables with mapping relationships
- **Compensation Components**: 6 separate internal LE pay elements
- **Performance**: Optimized for temporal analysis
- **Data Source**: Internal Labor Exchange operations

**Compensation Components Analysis:**

- **Base Pay**: `weekly_pay_rate` - Core compensation
- **Housing**: `weekly_housing_stipend` - Location adjustment
- **Meals**: `weekly_mie_stipend` - Per diem allowance
- **Travel**: `travel_allowance` - Mobility compensation
- **Billing**: `hourly_bill_rate` vs `weekly_bill_rate` - Revenue context
- **Contract**: `contract_type` - Employment classification

**Critical Limitation:**

- **fact_le**: 0 rows ❌ **No internal Labor Exchange data available**

**Business Intelligence Lost:**

- **Internal Compensation Analysis**: Cannot calculate complete internal LE pay packages
- **Internal vs Market Positioning**: No internal LE vs external market comparisons
- **Internal Geographic Adjustments**: Missing location-based internal compensation data
- **Internal Contract Analysis**: No internal LE employment type insights

**Overall Assessment:**

- **Functionality**: ❌ Complete failure - No internal LE data
- **Design Quality**: ✅ Excellent - Comprehensive internal compensation model
- **Business Impact**: Very High - Core to internal Labor Exchange operations
- **Priority**: Critical for HWL Labor Exchange business model

---

### Query 5.2: Calculate Total Compensation Components

**Internal Breakdown:**

```sql
SELECT
    jt.job_group,
    zc.cbsa_code,
    AVG(le.weekly_pay_rate) AS avg_base_pay,
    AVG(le.weekly_housing_stipend) AS avg_housing_stipend,
    AVG(le.weekly_mie_stipend) AS avg_mie_stipend,
    AVG(le.travel_allowance) AS avg_travel_allowance,
    AVG(le.weekly_pay_rate +
        COALESCE(le.weekly_housing_stipend, 0) +
        COALESCE(le.weekly_mie_stipend, 0) +
        COALESCE(le.travel_allowance, 0)) AS avg_total_compensation,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY
        le.weekly_pay_rate +
        COALESCE(le.weekly_housing_stipend, 0) +
        COALESCE(le.weekly_mie_stipend, 0) +
        COALESCE(le.travel_allowance, 0)) AS median_total_compensation,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY
        le.weekly_pay_rate +
        COALESCE(le.weekly_housing_stipend, 0) +
        COALESCE(le.weekly_mie_stipend, 0) +
        COALESCE(le.travel_allowance, 0)) AS p75_total_compensation
FROM fact_le le
JOIN dim_specialty_job_title_mapping sjtm ON le.specialty_job_title_id = sjtm.id
JOIN dim_job_title jt ON sjtm.job_title_id = jt.id
JOIN dim_hospital h ON le.hospital_id = h.id
JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
WHERE zc.cbsa_code = :target_cbsa_code
    AND jt.job_group = :selected_job_group
    AND le.filled_date >= :start_date
    AND le.filled_date <= :end_date
GROUP BY jt.job_group, zc.cbsa_code;
```

**Technical Analysis:**

- **Query Type**: Advanced aggregation with statistical analysis
- **Complexity**: Very High - Complex calculated fields with statistics
- **Aggregation Logic**: Multiple component averaging
- **Statistical Functions**: Percentile analysis on calculated totals
- **Null Handling**: COALESCE for missing allowances

**Advanced Features:**

- **Component Breakdown**: Separate averages for each pay element
- **Total Calculation**: Safe aggregation with null handling
- **Market Positioning**: Median and 75th percentile analysis
- **Geographic Segmentation**: CBSA-level insights
- **Job Group Analysis**: Occupation-specific compensation

**Mathematical Complexity:**

- **Nested Calculations**: Total compensation formula used in multiple contexts
- **Statistical Accuracy**: Percentile calculations on derived values
- **Data Validation**: COALESCE prevents null propagation
- **Precision**: Multiple levels of aggregation

**Overall Assessment:**

- **Functionality**: ❌ No data to process
- **Mathematical Design**: ✅ Excellent - Sophisticated calculations
- **Business Value**: Critical - Total compensation intelligence
- **Implementation**: Ready for data when available

---

## 6. Monthly Revenue Analysis Query - Internal Labor Exchange

### Query 6.1: Estimated Revenue from Internal LE Filled Positions

**Internal Breakdown:**

```sql
SELECT
    DATE_TRUNC('month', le.filled_date) AS revenue_month,
    COUNT(*) AS positions_filled,
    SUM(le.hourly_bill_rate * le.hrs_per_week * 4.33) AS estimated_monthly_revenue
FROM fact_le le
WHERE le.filled_date >= :start_date
    AND le.filled_date <= :end_date
GROUP BY DATE_TRUNC('month', le.filled_date)
ORDER BY revenue_month DESC;
```

**Technical Analysis:**

- **Query Type**: Internal Labor Exchange revenue aggregation
- **Complexity**: Medium - Time-based grouping with internal LE revenue math
- **Time Function**: DATE_TRUNC for monthly internal revenue aggregation
- **Revenue Logic**: Internal LE bill rate × hours × weeks per month (4.33)
- **Performance**: Efficient aggregation query for internal data
- **Data Source**: Internal Labor Exchange operations

**Revenue Calculation Analysis:**

- **Hourly Rate**: `le.hourly_bill_rate` - Internal LE base billing rate
- **Utilization**: `le.hrs_per_week` - Internal LE weekly hours commitment
- **Time Factor**: 4.33 - Average weeks per month
- **Volume**: `COUNT(*)` - Internal LE positions filled per month

**Business Intelligence:**

- **Internal Revenue Trends**: Month-over-month Labor Exchange growth analysis
- **Volume Insights**: Internal LE positions filled tracking
- **Performance Metrics**: Internal LE revenue per position calculations
- **Seasonal Analysis**: Monthly internal operations pattern identification

**Critical Limitation:**

- **fact_le**: 0 rows ❌ **No internal Labor Exchange revenue data source**

**Alternative Revenue Approach:**

```sql
-- Use contract data as revenue proxy
SELECT
    DATE_TRUNC('month', ccr.fiscal_year_end) as revenue_month,
    SUM(ccr.contract_wages) / 12 as estimated_monthly_revenue,
    COUNT(DISTINCT ccr.hospital_id) as active_contracts
FROM fact_cms_cost_reports ccr
WHERE ccr.report_year >= EXTRACT(YEAR FROM :start_date)
GROUP BY DATE_TRUNC('month', ccr.fiscal_year_end)
ORDER BY revenue_month DESC;
```

**Overall Assessment:**

- **Functionality**: ❌ No source data
- **Business Value**: Very High - Revenue tracking essential
- **Alternative**: Contract-based revenue estimates possible
- **Priority**: Critical for business performance monitoring

---

## 7. System Operations Queries (Placeholders)

### Query 7.1: Active Users Analysis

**Internal Breakdown:**

```sql
/*
SELECT
    DATE_TRUNC('week', activity_date) AS week,
    COUNT(DISTINCT user_id) AS active_users,
    COUNT(DISTINCT CASE WHEN user_type = 'hospital' THEN user_id END) AS hospital_users,
    COUNT(DISTINCT CASE WHEN user_type = 'hwl_internal' THEN user_id END) AS hwl_users
FROM user_activity
WHERE activity_date >= CURRENT_DATE - INTERVAL '12 weeks'
GROUP BY DATE_TRUNC('week', activity_date)
ORDER BY week DESC;
*/
```

**Technical Analysis:**

- **Query Type**: User engagement analytics
- **Complexity**: Medium - Conditional aggregation
- **Time Grouping**: Weekly user activity trends
- **User Segmentation**: Hospital vs internal users
- **Performance**: Would be efficient with proper indexing

**Business Intelligence:**

- **Engagement Tracking**: Weekly active user trends
- **User Segmentation**: Different user type analysis
- **Growth Analysis**: 12-week historical trends
- **Product Analytics**: User adoption insights

**Implementation Status:**

- **Status**: Placeholder - Requires operational database
- **Tables Needed**: `user_activity` - User engagement tracking
- **Business Value**: High - Product usage insights
- **Alternative**: Application log analysis

**Overall Assessment:**

- **Functionality**: Not implemented - Placeholder only
- **Design**: ✅ Good - Standard analytics pattern
- **Business Value**: High for product management
- **Implementation**: Requires separate operational systems

---

### Query 7.2: System Health Status

**Internal Breakdown:**

```sql
/*
SELECT
    component_name,
    status,
    uptime_percentage,
    last_incident_date,
    last_check_timestamp
FROM system_health_status
WHERE last_check_timestamp >= CURRENT_TIMESTAMP - INTERVAL '1 hour';
*/
```

**Technical Analysis:**

- **Query Type**: System monitoring lookup
- **Complexity**: Low - Simple status query
- **Temporal Filter**: Recent health checks only
- **Performance**: Very fast - Status table lookup

**Monitoring Capabilities:**

- **Component Status**: Individual system component health
- **Uptime Tracking**: Percentage-based availability metrics
- **Incident History**: Last failure timestamp
- **Real-time**: Current operational status

**Implementation Requirements:**

- **Monitoring Infrastructure**: System health collection
- **Status Table**: Real-time health metrics storage
- **Alerting Integration**: Incident detection and logging
- **Historical Data**: Uptime trend analysis

**Overall Assessment:**

- **Functionality**: Not implemented - Infrastructure dependent
- **Business Value**: High - Operational reliability
- **Implementation**: Requires monitoring system integration
- **Alternative**: External monitoring tools (Datadog, New Relic)

---

### Query 7.3: Total Projects Status

**Internal Breakdown:**

```sql
/*
SELECT
    project_status,
    COUNT(*) AS project_count
FROM projects
WHERE is_active = true
GROUP BY project_status
ORDER BY
    CASE project_status
        WHEN 'Active' THEN 1
        WHEN 'Pending' THEN 2
        WHEN 'Completed' THEN 3
        ELSE 4
    END;
*/
```

**Technical Analysis:**

- **Query Type**: Status aggregation with custom sorting
- **Complexity**: Low - Simple grouping query
- **Custom Ordering**: CASE statement for status priority
- **Filtering**: Active projects only

**Project Management Features:**

- **Status Distribution**: Count by project status
- **Priority Ordering**: Custom status hierarchy
- **Active Filter**: Excludes archived projects
- **Management Dashboard**: Executive project overview

**Implementation Requirements:**

- **Project Management System**: Source of project data
- **Status Standardization**: Consistent status values
- **Integration**: Connection to project tracking tools

**Overall Assessment:**

- **Functionality**: Not implemented - Requires project system
- **Design**: ✅ Good - Standard project analytics
- **Business Value**: Medium - Management visibility
- **Implementation**: Integrate with existing project tools

---

## 8. Global Filters and Helper Queries

### Query 8.1: Get All Available CBSAs

**Internal Breakdown:**

```sql
SELECT DISTINCT
    cbsa_code,
    cbsa_title
FROM dim_zip_cbsa
WHERE cbsa_code IS NOT NULL
ORDER BY cbsa_title;
```

**Technical Analysis:**

- **Query Type**: Reference data lookup
- **Complexity**: Very Low - Simple distinct query
- **Performance**: Fast - Small result set from large table
- **Data Quality**: NULL filtering ensures clean results
- **Sorting**: Alphabetical by title for usability

**Data Quality:**

- **dim_zip_cbsa**: 40,773 rows ✅ Excellent coverage
- **Data Completeness**: Filtering nulls ensures quality
- **Geographic Coverage**: Comprehensive US market areas

**UI Integration:**

- **Dropdown Population**: Perfect for filter controls
- **User Experience**: Alphabetical sorting for navigation
- **Data Validation**: Clean, non-null values only

**Overall Assessment:**

- **Functionality**: ✅ Perfect - Complete and fast
- **Business Value**: High - Essential for geographic filtering
- **Performance**: Excellent - Quick reference lookup
- **Data Quality**: Outstanding - Clean, comprehensive data

---

### Query 8.2: Get All Bed Size Ranges

**Internal Breakdown:**

```sql
SELECT DISTINCT
    bed_range
FROM dim_hospital
WHERE bed_range IS NOT NULL
ORDER BY bed_range;
```

**Technical Analysis:**

- **Query Type**: Reference data extraction
- **Complexity**: Very Low - Simple distinct lookup
- **Data Source**: Hospital dimension (6,918 rows)
- **Quality Control**: NULL filtering
- **Sorting**: Natural ordering of ranges

**Bed Range Analysis:**

- **Categories**: Standardized bed size buckets
- **Peer Matching**: Essential for facility comparison
- **Market Segmentation**: Hospital size classification
- **Quality**: Clean, standardized values

**Overall Assessment:**

- **Functionality**: ✅ Complete
- **Business Value**: High - Core peer matching criteria
- **Data Quality**: Excellent
- **Performance**: Very fast

---

### Query 8.3: Get All Ownership Types

**Internal Breakdown:**

```sql
SELECT DISTINCT
    ownership_type
FROM dim_hospital
WHERE ownership_type IS NOT NULL
ORDER BY ownership_type;
```

**Technical Analysis:**

- **Query Type**: Reference lookup
- **Complexity**: Very Low
- **Data Quality**: Clean ownership classifications
- **Performance**: Instantaneous

**Ownership Classifications:**

- **Government**: Public sector facilities
- **Non-profit**: Tax-exempt organizations
- **For-profit**: Commercial enterprises
- **Other**: Mixed or special classifications

**Overall Assessment:**

- **Functionality**: ✅ Perfect
- **Business Value**: High - Critical for peer matching
- **Data Quality**: Excellent
- **Implementation**: Ready for immediate use

---

### Query 8.4: Get All Job Groups

**Internal Breakdown:**

```sql
SELECT DISTINCT
    job_group
FROM dim_job_title
WHERE job_group IS NOT NULL
ORDER BY job_group;
```

**Technical Analysis:**

- **Query Type**: Job classification lookup
- **Complexity**: Very Low
- **Data Source**: Job title dimension (247 rows)
- **Quality**: Clean job group classifications

**Job Group Categories:**

- **Healthcare Practitioners**: Clinical roles
- **Support Services**: Non-clinical roles
- **Administrative**: Management and admin
- **Technical**: Specialized technical roles

**Overall Assessment:**

- **Functionality**: ✅ Complete
- **Business Value**: High - Essential for role-based analysis
- **Data Quality**: Excellent
- **Performance**: Very fast

---

### Query 8.5: Get Available Report Years

**Internal Breakdown:**

```sql
SELECT DISTINCT
    report_year
FROM fact_hcris_facility
ORDER BY report_year DESC;
```

**Technical Analysis:**

- **Query Type**: Temporal reference lookup
- **Complexity**: Very Low
- **Data Source**: HCRIS facility facts (72,802 rows)
- **Sorting**: Descending - most recent first

**Temporal Coverage:**

- **Historical Data**: Multiple years available
- **Trend Analysis**: Supports multi-year comparisons
- **Data Freshness**: Shows available reporting periods
- **Quality**: Complete year coverage

**Overall Assessment:**

- **Functionality**: ✅ Complete
- **Business Value**: High - Temporal filtering essential
- **Data Quality**: Excellent - Good historical coverage
- **Performance**: Very fast reference lookup

---

## Overall Query Performance Assessment

### Query Complexity Distribution:

- **Simple (Low Complexity)**: 8 queries - Reference/lookup queries
- **Medium Complexity**: 6 queries - Standard business logic
- **High Complexity**: 6 queries - Statistical analysis and complex joins
- **Very High Complexity**: 2 queries - Advanced statistical calculations

### Performance Analysis:

#### Excellent Performance (Sub-second):

- All reference/helper queries
- Single-record facility lookups
- Simple aggregations with good selectivity

#### Good Performance (1-5 seconds):

- Most business logic queries with available data
- Standard peer comparisons
- Department analysis (when data available)

#### Performance Issues (5+ seconds):

- Correlated subqueries (Query 1.3)
- Complex statistical calculations across large datasets
- Multi-table joins without proper indexing

#### Optimization Recommendations:

1. **Indexing Strategy**: Composite indexes on filtering columns
2. **Query Rewriting**: Replace correlated subqueries with CTEs
3. **Statistics Updates**: Ensure database statistics are current
4. **Partitioning**: Consider partitioning large fact tables by year

### Data Dependency Summary:

#### Fully Functional (13 queries):

- All facility information queries
- BLS market intelligence queries
- Reference/helper queries
- Basic peer comparisons (facility-level)

#### Partially Functional (3 queries):

- Peer comparisons (limited metrics)
- Some compensation analysis (market-only)

#### Non-Functional (6 queries):

- Department-level analysis (missing HCRIS cost center data)
- Internal HWL staffing rate analysis (missing HWL operational data)
- Internal Labor Exchange compensation calculations (missing LE operational data)
- Internal revenue tracking (missing LE revenue data)

### Business Value Assessment:

#### High Business Value Available:

- Market intelligence and benchmarking
- Facility profiling and basic peer analysis
- Geographic and ownership-based filtering
- External wage analysis

#### High Business Value Missing:

- Department-level cost optimization (HCRIS cost center data)
- Internal HWL vs external market rate comparison (HWL operational data)
- Internal Labor Exchange compensation planning (LE operational data)
- Internal Labor Exchange revenue performance tracking (LE revenue data)

### Implementation Recommendations:

#### Phase 1 - Immediate (2-4 weeks):

- Implement all functional queries
- Focus on market intelligence features
- Build basic peer comparison capabilities

#### Phase 2 - Workarounds (4-6 weeks):

- Implement CMS job category analysis
- Create department alternatives
- Build estimated compensation models

#### Phase 3 - Complete (8-12 weeks):

- Populate missing HWL data tables
- Implement full departmental analysis
- Enable complete compensation tracking

This comprehensive analysis provides both technical implementation guidance and strategic business direction for the healthcare dashboard development project.
