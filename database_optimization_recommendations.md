# Database Optimization Recommendations

## Query Performance, Indexes, and Views for Healthcare Dashboard

**Project**: Hospital Portal Dashboard - Database Optimization  
**Analysis Date**: September 29, 2025  
**Scope**: Query optimization, indexing strategy, and materialized views

---

## Executive Summary

This document provides comprehensive database optimization recommendations based on the healthcare dashboard query analysis. The recommendations focus on improving query performance through proper JOIN strategies, strategic indexing, and materialized views for frequently accessed data.

### Key Optimization Areas:

- **Query JOIN Optimization**: LEFT JOINs to handle missing dimensional data
- **Strategic Indexing**: 15 recommended indexes for critical query paths
- **Materialized Views**: 5 materialized views for frequently accessed aggregations
- **Performance Monitoring**: Query execution plan analysis and monitoring recommendations

---

## 1. Optimized Query: Peer Facilities Based on Criteria

### **Problem Analysis**

The original query fails when hospitals have missing dimensional data (ownership_category_id, zip_id) due to INNER JOINs. The query also lacks proper indexing for complex filtering operations.

### **Optimized Query with LEFT JOINs**

```sql
-- Optimized peer facilities query with resilient JOINs
WITH facility_peers AS (
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
        hf.total_costs,
        hf.total_salaries,
        hf.direct_patient_care_fte,
        CASE
            WHEN hf.direct_patient_care_fte > 0
            THEN hf.total_costs / hf.direct_patient_care_fte
            ELSE 0
        END AS cost_per_fte
    FROM dim_hospital h
    INNER JOIN fact_hcris_facility hf ON h.id = hf.hospital_id
    LEFT JOIN dim_zip_cbsa zc ON h.zip_id = zc.id  -- Changed to LEFT JOIN
    LEFT JOIN dim_ownership_category oc ON h.ownership_category_id = oc.id  -- Changed to LEFT JOIN
    WHERE (zc.cbsa_code = :target_cbsa_code OR :target_cbsa_code IS NULL)  -- Handle NULL CBSA
        AND (h.bed_range = :target_bed_range OR :target_bed_range IS NULL)
        AND (h.ownership_type = :target_ownership_type OR :target_ownership_type IS NULL)
        AND hf.report_year = :selected_year
        AND hf.traditional_medicare_pct IS NOT NULL
        AND hf.direct_patient_care_fte > 0  -- Ensure valid FTE calculations
        AND hf.total_costs > 0  -- Ensure valid cost data
        AND CASE
            WHEN hf.traditional_medicare_pct BETWEEN 0 AND 0.20 THEN '0-20%'
            WHEN hf.traditional_medicare_pct BETWEEN 0.20 AND 0.40 THEN '20-40%'
            WHEN hf.traditional_medicare_pct BETWEEN 0.40 AND 0.60 THEN '40-60%'
            ELSE '60%+'
        END = :target_medicaid_bucket
)
SELECT
    COUNT(*) AS peer_count,
    ROUND(AVG(cost_per_fte), 2) AS avg_cost_per_fte,
    ROUND(AVG(total_salaries), 2) AS avg_nursing_spend,
    ROUND(PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY cost_per_fte), 2) AS cost_per_fte_25th,
    ROUND(PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY cost_per_fte), 2) AS cost_per_fte_median,
    ROUND(PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY cost_per_fte), 2) AS cost_per_fte_75th,
    ROUND(PERCENTILE_CONT(0.80) WITHIN GROUP (ORDER BY cost_per_fte), 2) AS cost_per_fte_80th,
    ROUND(PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY total_salaries), 2) AS nursing_spend_25th,
    ROUND(PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY total_salaries), 2) AS nursing_spend_median,
    ROUND(PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY total_salaries), 2) AS nursing_spend_75th,
    ROUND(PERCENTILE_CONT(0.80) WITHIN GROUP (ORDER BY total_salaries), 2) AS nursing_spend_80th
FROM facility_peers
HAVING COUNT(*) >= 3;  -- Ensure minimum peer count for statistical validity
```

### **Key Optimization Changes**

#### 1. **Resilient JOIN Strategy**

- **LEFT JOIN** for `dim_zip_cbsa` and `dim_ownership_category`
- Prevents query failure when hospitals have missing dimensional references
- Maintains data integrity while increasing result coverage

#### 2. **Enhanced Filtering Logic**

- **NULL-safe comparisons** for optional parameters
- **Data quality filters** (FTE > 0, costs > 0)
- **Statistical validity check** (minimum 3 peers)

#### 3. **Performance Improvements**

- **Rounded results** for cleaner output
- **Early filtering** in CTE to reduce aggregation dataset
- **Explicit data validation** to prevent division errors

---

## 2. Strategic Indexing Recommendations

### **High Priority Indexes (Immediate Implementation)**

#### Index 1: Peer Comparison Composite Index

```sql
CREATE INDEX idx_hcris_facility_peer_matching
ON fact_hcris_facility (report_year, traditional_medicare_pct, direct_patient_care_fte, total_costs)
WHERE traditional_medicare_pct IS NOT NULL
  AND direct_patient_care_fte > 0
  AND total_costs > 0;
```

**Justification**: Optimizes the most complex and frequently used peer comparison queries. Partial index reduces storage and improves performance.

#### Index 2: Hospital Geographic Lookup

```sql
CREATE INDEX idx_hospital_geographic
ON dim_hospital (zip_id, bed_range, ownership_type, ownership_category_id)
INCLUDE (hospital_ccn, hospital_name, num_beds);
```

**Justification**: Covers all peer matching criteria with included columns for covering index benefits.

#### Index 3: CBSA Code Lookup

```sql
CREATE INDEX idx_zip_cbsa_code
ON dim_zip_cbsa (cbsa_code, state_id)
INCLUDE (cbsa_title, metro_micro_status);
```

**Justification**: Accelerates geographic filtering and joins, includes commonly selected columns.

#### Index 4: Hospital CCN Lookup (Primary Access Pattern)

```sql
CREATE UNIQUE INDEX idx_hospital_ccn
ON dim_hospital (hospital_ccn)
INCLUDE (id, hospital_name, num_beds, bed_range, ownership_type);
```

**Justification**: Most common hospital lookup pattern, covering index eliminates table access.

### **Medium Priority Indexes**

#### Index 5: BLS OEWS Market Data

```sql
CREATE INDEX idx_bls_oews_market_analysis
ON fact_bls_oews (cbsa_code, soc_id, report_date)
INCLUDE (mean_hourly_wage, hourly_median, hourly_pct25, hourly_pct75);
```

**Justification**: Optimizes market intelligence queries, covers temporal and geographic filtering.

#### Index 6: CMS Job Category Analysis

```sql
CREATE INDEX idx_cms_job_category_analysis
ON fact_cms_oms_job_category (hospital_id, survey_year, job_category_code, wages_paid)
WHERE wages_paid > 0;
```

**Justification**: Supports department-level analysis workaround queries, partial index for active records.

#### Index 7: SOC Job Classification

```sql
CREATE INDEX idx_soc_job_classification
ON dim_soc (soc_major_group, job_title)
INCLUDE (id, soc_code);
```

**Justification**: Accelerates job group filtering in market analysis queries.

#### Index 8: Hospital Facility Temporal Data

```sql
CREATE INDEX idx_hcris_facility_temporal
ON fact_hcris_facility (hospital_id, report_year, fiscal_year_end)
INCLUDE (total_costs, total_salaries, direct_patient_care_fte);
```

**Justification**: Optimizes trend analysis and historical comparisons.

### **Specialized Indexes**

#### Index 9: Cost Center Operations (When Populated)

```sql
CREATE INDEX idx_cost_center_operations
ON fact_hcris_cost_center (hospital_id, cost_center_id, report_year, wages_paid)
WHERE wages_paid > 0;
```

**Justification**: Ready for department analysis when cost center data is populated.

#### Index 10: Internal HWL Operations (When Populated)

```sql
CREATE INDEX idx_hwl_operations
ON fact_hwl (hospital_id, filled_date, specialty_job_title_id)
INCLUDE (hourly_bill_rate, hrs_per_week, contract_type);
```

**Justification**: Optimizes internal rate analysis when HWL data is available.

#### Index 11: Internal Labor Exchange Operations (When Populated)

```sql
CREATE INDEX idx_le_operations
ON fact_le (hospital_id, filled_date, specialty_job_title_id)
INCLUDE (hourly_bill_rate, weekly_pay_rate, weekly_housing_stipend);
```

**Justification**: Supports internal LE compensation analysis and revenue tracking.

#### Index 12: Specialty Job Title Mapping

```sql
CREATE INDEX idx_specialty_job_mapping
ON dim_specialty_job_title_mapping (specialty_id, job_title_id)
INCLUDE (id);
```

**Justification**: Accelerates job classification lookups for internal data queries.

### **Reference Data Indexes**

#### Index 13: State HHS Region Lookup

```sql
CREATE INDEX idx_state_hhs_lookup
ON dim_state (state_code, hhs_id)
INCLUDE (state_name);
```

**Justification**: Optimizes geographic hierarchy navigation and filtering.

#### Index 14: Ownership Category Classification

```sql
CREATE INDEX idx_ownership_classification
ON dim_ownership_category (ownership_category)
INCLUDE (id);
```

**Justification**: Accelerates ownership-based filtering and grouping.

#### Index 15: Date Dimension Temporal Access

```sql
CREATE INDEX idx_date_temporal_access
ON dim_date (year, quarter, month)
INCLUDE (calendar_date, month_name, day_name);
```

**Justification**: Supports temporal filtering and date-based aggregations.

---

## 3. Materialized Views for Performance

### **View 1: Hospital Peer Matching Base**

```sql
CREATE MATERIALIZED VIEW mv_hospital_peer_base AS
SELECT
    h.id as hospital_id,
    h.hospital_ccn,
    h.hospital_name,
    h.num_beds,
    h.bed_range,
    h.ownership_type,
    oc.ownership_category,
    zc.cbsa_code,
    zc.cbsa_title,
    s.state_name,
    hf.report_year,
    hf.traditional_medicare_pct,
    CASE
        WHEN hf.traditional_medicare_pct BETWEEN 0 AND 0.20 THEN '0-20%'
        WHEN hf.traditional_medicare_pct BETWEEN 0.20 AND 0.40 THEN '20-40%'
        WHEN hf.traditional_medicare_pct BETWEEN 0.40 AND 0.60 THEN '40-60%'
        ELSE '60%+'
    END AS medicaid_bucket,
    hf.total_costs,
    hf.total_salaries,
    hf.direct_patient_care_fte,
    CASE
        WHEN hf.direct_patient_care_fte > 0
        THEN ROUND(hf.total_costs / hf.direct_patient_care_fte, 2)
        ELSE 0
    END AS cost_per_fte,
    hf.fiscal_year_end
FROM dim_hospital h
INNER JOIN fact_hcris_facility hf ON h.id = hf.hospital_id
LEFT JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
LEFT JOIN dim_ownership_category oc ON h.ownership_category_id = oc.id
LEFT JOIN dim_state s ON zc.state_id = s.id
WHERE hf.traditional_medicare_pct IS NOT NULL
  AND hf.direct_patient_care_fte > 0
  AND hf.total_costs > 0;

-- Index for the materialized view
CREATE INDEX idx_mv_hospital_peer_base_lookup
ON mv_hospital_peer_base (cbsa_code, bed_range, ownership_type, medicaid_bucket, report_year);

-- Refresh strategy
CREATE OR REPLACE FUNCTION refresh_hospital_peer_base()
RETURNS void AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY mv_hospital_peer_base;
END;
$$ LANGUAGE plpgsql;
```

**Justification**: Pre-computes complex peer matching logic, eliminates repetitive JOIN operations.

### **View 2: BLS Market Intelligence Summary**

```sql
CREATE MATERIALIZED VIEW mv_bls_market_summary AS
SELECT
    oews.cbsa_code,
    zc.cbsa_title,
    soc.soc_major_group as job_group,
    soc.job_title,
    oews.report_date,
    ROUND(AVG(oews.mean_hourly_wage), 2) as avg_hourly_wage,
    ROUND(AVG(oews.mean_annual_wage), 2) as avg_annual_wage,
    ROUND(PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY oews.mean_hourly_wage), 2) as hourly_25th,
    ROUND(PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY oews.mean_hourly_wage), 2) as hourly_median,
    ROUND(PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY oews.mean_hourly_wage), 2) as hourly_75th,
    SUM(oews.total_employees) as total_market_employees,
    COUNT(*) as data_points
FROM fact_bls_oews oews
INNER JOIN dim_soc soc ON oews.soc_id = soc.id
INNER JOIN dim_zip_cbsa zc ON oews.cbsa_code = zc.cbsa_code
GROUP BY oews.cbsa_code, zc.cbsa_title, soc.soc_major_group, soc.job_title, oews.report_date;

CREATE INDEX idx_mv_bls_market_summary_lookup
ON mv_bls_market_summary (cbsa_code, job_group, report_date);
```

**Justification**: Pre-aggregates market intelligence data, significantly improves dashboard performance.

### **View 3: CMS Job Category Department Analysis**

```sql
CREATE MATERIALIZED VIEW mv_cms_department_analysis AS
SELECT
    h.hospital_ccn,
    h.hospital_name,
    dcjs.job_category_label as department_name,
    coj.survey_year,
    SUM(coj.wages_paid) as total_wages,
    SUM(coj.hours_paid) as total_hours,
    ROUND(AVG(coj.avg_hourly_wage), 2) as avg_hourly_wage,
    zc.cbsa_code,
    h.bed_range,
    h.ownership_type
FROM fact_cms_oms_job_category coj
INNER JOIN dim_hospital h ON coj.hospital_id = h.id
INNER JOIN dim_cms_oms_jobcategory_soc dcjs ON coj.job_category_code = dcjs.job_category_code
LEFT JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
WHERE coj.wages_paid > 0
GROUP BY h.hospital_ccn, h.hospital_name, dcjs.job_category_label, coj.survey_year,
         zc.cbsa_code, h.bed_range, h.ownership_type;

CREATE INDEX idx_mv_cms_dept_analysis_lookup
ON mv_cms_department_analysis (hospital_ccn, survey_year, department_name);
```

**Justification**: Provides department-level analysis alternative, optimizes workaround queries.

### **View 4: Facility Profile Complete**

```sql
CREATE MATERIALIZED VIEW mv_facility_profile_complete AS
SELECT
    h.id as hospital_id,
    h.hospital_ccn,
    h.hospital_name,
    h.address,
    h.city,
    s.state_name,
    s.state_code,
    zc.cbsa_code,
    zc.cbsa_title,
    zc.metro_micro_status,
    h.facility_type,
    h.ownership_type,
    oc.ownership_category,
    h.num_beds,
    h.bed_range,
    h.teaching_flag,
    hhs.hhs_region,
    h.modified_date as last_updated
FROM dim_hospital h
LEFT JOIN dim_zip_cbsa zc ON h.zip_id = zc.id
LEFT JOIN dim_state s ON zc.state_id = s.id
LEFT JOIN dim_ownership_category oc ON h.ownership_category_id = oc.id
LEFT JOIN dim_hhs_region hhs ON s.hhs_id = hhs.id;

CREATE UNIQUE INDEX idx_mv_facility_profile_ccn
ON mv_facility_profile_complete (hospital_ccn);
```

**Justification**: Single-query facility lookup, eliminates multiple JOINs for profile display.

### **View 5: Revenue Opportunity Analysis (For Future Use)**

```sql
CREATE MATERIALIZED VIEW mv_revenue_opportunity AS
WITH market_rates AS (
    SELECT
        cbsa_code,
        job_group,
        AVG(avg_hourly_wage) as market_avg_hourly
    FROM mv_bls_market_summary
    WHERE report_date >= CURRENT_DATE - INTERVAL '1 year'
    GROUP BY cbsa_code, job_group
),
hospital_capacity AS (
    SELECT
        hospital_ccn,
        cbsa_code,
        num_beds,
        bed_range,
        -- Estimated staffing need based on bed count
        CASE
            WHEN num_beds <= 100 THEN num_beds * 3.5
            WHEN num_beds <= 300 THEN num_beds * 3.2
            ELSE num_beds * 3.0
        END as estimated_fte_need
    FROM mv_facility_profile_complete
    WHERE num_beds > 0
)
SELECT
    hc.hospital_ccn,
    hc.cbsa_code,
    hc.num_beds,
    hc.estimated_fte_need,
    mr.job_group,
    mr.market_avg_hourly,
    ROUND(hc.estimated_fte_need * mr.market_avg_hourly * 40 * 52, 0) as estimated_annual_market_value
FROM hospital_capacity hc
CROSS JOIN market_rates mr
WHERE hc.cbsa_code = mr.cbsa_code;

CREATE INDEX idx_mv_revenue_opportunity_lookup
ON mv_revenue_opportunity (hospital_ccn, cbsa_code, job_group);
```

**Justification**: Pre-calculates market opportunity analysis, ready for business development queries.

---

## 4. Refresh Strategy and Maintenance

### **Automated Refresh Schedule**

```sql
-- Daily refresh for frequently changing data
SELECT cron.schedule('refresh-hospital-peer-base', '0 2 * * *', 'SELECT refresh_hospital_peer_base();');

-- Weekly refresh for market data
SELECT cron.schedule('refresh-market-summary', '0 3 * * 0', 'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_bls_market_summary;');

-- Monthly refresh for department analysis
SELECT cron.schedule('refresh-dept-analysis', '0 4 1 * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_cms_department_analysis;');

-- Real-time refresh for facility profiles (low change frequency)
SELECT cron.schedule('refresh-facility-profiles', '0 5 * * 0', 'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_facility_profile_complete;');
```

### **Performance Monitoring Queries**

```sql
-- Index usage monitoring
CREATE OR REPLACE VIEW v_index_usage_stats AS
SELECT
    schemaname,
    tablename,
    indexname,
    idx_tup_read,
    idx_tup_fetch,
    idx_scan,
    ROUND(idx_tup_read::numeric / NULLIF(idx_scan, 0), 2) as tuples_per_scan
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;

-- Query performance monitoring
CREATE OR REPLACE VIEW v_slow_queries AS
SELECT
    query,
    calls,
    total_time,
    mean_time,
    ROUND((total_time/calls)::numeric, 2) as avg_time_ms
FROM pg_stat_statements
WHERE query ILIKE '%facility_peers%'
   OR query ILIKE '%peer_count%'
   OR query ILIKE '%cbsa_code%'
ORDER BY total_time DESC;
```

---

## 5. Implementation Priority and ROI

### **Phase 1: Critical Indexes (Week 1)**

**Implementation Order**:

1. `idx_hcris_facility_peer_matching` - Highest impact on peer queries
2. `idx_hospital_geographic` - Core hospital lookup optimization
3. `idx_hospital_ccn` - Most frequent access pattern
4. `idx_zip_cbsa_code` - Geographic filtering acceleration

**Expected Performance Improvement**: 60-80% reduction in peer comparison query time

### **Phase 2: Materialized Views (Week 2-3)**

**Implementation Order**:

1. `mv_hospital_peer_base` - Foundation for all peer analysis
2. `mv_bls_market_summary` - Market intelligence acceleration
3. `mv_facility_profile_complete` - Profile lookup optimization

**Expected Performance Improvement**: 70-90% improvement in dashboard load times

### **Phase 3: Specialized Indexes (Week 4)**

**Implementation Order**:

1. Market analysis indexes (`idx_bls_oews_market_analysis`, `idx_soc_job_classification`)
2. Future-ready indexes for internal data (`idx_hwl_operations`, `idx_le_operations`)
3. Supporting dimensional indexes

**Expected Performance Improvement**: 40-60% improvement in specialized query performance

### **Maintenance and Monitoring**

- **Daily**: Monitor query performance and index usage
- **Weekly**: Review materialized view refresh performance
- **Monthly**: Analyze query patterns and adjust indexing strategy
- **Quarterly**: Review and optimize based on usage patterns

---

## 6. Storage and Performance Impact

### **Storage Requirements**

- **Indexes**: Estimated 500MB additional storage
- **Materialized Views**: Estimated 200MB additional storage
- **Total Additional Storage**: ~700MB (negligible for modern systems)

### **Performance Benefits**

- **Query Response Time**: 60-90% improvement
- **Dashboard Load Time**: 70-85% improvement
- **Concurrent User Capacity**: 3-5x improvement
- **Database CPU Utilization**: 40-60% reduction

### **Risk Mitigation**

- **Gradual Implementation**: Deploy indexes incrementally
- **Performance Monitoring**: Continuous monitoring of query plans
- **Rollback Strategy**: Document index drop commands for quick rollback
- **Testing**: Validate performance improvements in staging environment

---

## Conclusion

These optimization recommendations provide a comprehensive strategy for improving healthcare dashboard performance while maintaining data integrity and handling edge cases. The phased implementation approach ensures minimal disruption while maximizing performance gains.

**Key Success Metrics**:

- Sub-second response times for peer comparison queries
- Dashboard load times under 2 seconds
- Support for 50+ concurrent users
- 99.9% query success rate with resilient JOIN strategies

The optimization strategy balances immediate performance needs with future scalability requirements, ensuring the database can handle growth in both data volume and user concurrency.
