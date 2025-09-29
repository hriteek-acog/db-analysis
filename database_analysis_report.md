# Database Query Analysis Report

## Healthcare Dashboard Implementation Analysis

**Project**: Hospital Portal Dashboard - UI/UX Improvement  
**Analysis Date**: September 29, 2025  
**Analyst**: Database Query Analysis Report

---

## Executive Summary

This report provides a comprehensive analysis of database queries designed for a hospital dashboard portal. The analysis reveals significant data availability issues that impact the implementation of key dashboard features, particularly those requiring internal HWL operational data.

### Key Findings:

- **5 critical fact tables are empty** (0 rows), severely limiting functionality
- **External market data is excellent** (BLS OEWS: 159,739 rows)
- **Facility-level data is comprehensive** (72,802 HCRIS facility records)
- **Department-level analysis requires workarounds** due to missing cost center data

---

## Database Schema Overview

### Available Tables (With Data)

| Table                       | Row Count | Purpose                    |
| --------------------------- | --------- | -------------------------- |
| `fact_bls_oews`             | 159,739   | External wage benchmarking |
| `fact_hcris_facility`       | 72,802    | Hospital facility metrics  |
| `fact_cms_oms_job_category` | 45,625    | CMS job category wages     |
| `dim_zip_cbsa`              | 40,773    | Geographic/CBSA mapping    |
| `fact_cms_cost_reports`     | 25,294    | CMS cost report data       |
| `dim_hospital`              | 6,918     | Hospital master data       |
| `dim_date`                  | 4,018     | Date dimension             |
| `fact_market_basket_index`  | 1,040     | Market basket indices      |

### Missing Tables (Empty) - Internal HWL Data

| Table                          | Row Count | Impact                                  |
| ------------------------------ | --------- | --------------------------------------- |
| `fact_hwl`                     | 0         | ❌ Internal HWL staffing rate data      |
| `fact_le`                      | 0         | ❌ Internal Labor Exchange compensation |
| `fact_hcris_cost_center`       | 0         | ❌ Department-level costs               |
| `fact_medicare_case_mix_index` | 0         | ❌ Case mix indexing                    |
| `dim_cms_oms_hwl_le`           | 0         | ❌ HWL-LE job title mapping             |

---

## Task-by-Task Analysis

## 1. Peer Comparison Widget

### **Business Objective**

Show how selected facility compares to peer group using cost per FTE, nursing department spend, and agency spend metrics with defensible peer matching criteria.

### **Query Strategy & Thought Process**

The queries implement a sophisticated peer matching algorithm:

1. **Target Identification**: Extract facility's key characteristics (CBSA, bed size, ownership type, Medicaid payer mix)
2. **Peer Discovery**: Find all facilities matching these exact criteria
3. **Statistical Analysis**: Calculate averages, percentiles (25th, 50th, 75th, 80th) for benchmarking
4. **Trend Analysis**: Provide 12-month sparklines for facility vs peer performance

### **Tables Used & Relevance Analysis**

| Table                    | Rows   | Status       | Role & Importance                                                                                 |
| ------------------------ | ------ | ------------ | ------------------------------------------------------------------------------------------------- |
| `dim_hospital`           | 6,918  | ✅ Available | **Core Foundation** - Hospital identifiers, bed counts, ownership types, geographic references    |
| `fact_hcris_facility`    | 72,802 | ✅ Available | **Essential Metrics** - Total costs, salaries, FTE counts, Medicare percentages for peer matching |
| `dim_zip_cbsa`           | 40,773 | ✅ Available | **Geographic Matching** - CBSA codes and titles for location-based peer grouping                  |
| `dim_ownership_category` | 4      | ✅ Available | **Ownership Classification** - Government, Non-profit, For-profit categorization                  |
| `fact_hcris_cost_center` | 0      | ❌ Missing   | **Critical Gap** - Department-level spending, especially nursing department spend                 |

### **Implementation Status**

- **✅ Partially Functional**: Facility-level peer comparisons work
- **❌ Missing Feature**: Department-specific comparisons (nursing spend, agency spend)
- **⚠️ Limited Metrics**: Only 3 of 5 intended metrics available

### **Workarounds & Solutions**

#### Immediate Solutions:

1. **Use Facility-Level Proxies**:

   ```sql
   -- Use total_salaries as proxy for nursing spend
   SELECT hf.total_salaries as nursing_spend_proxy
   FROM fact_hcris_facility hf
   ```

2. **CMS Job Category Mapping**:
   ```sql
   -- Estimate nursing spend using CMS job category data
   SELECT SUM(wages_paid) as estimated_nursing_spend
   FROM fact_cms_oms_job_category
   WHERE job_category_code IN (SELECT job_category_code
                              FROM dim_cms_oms_jobcategory_soc
                              WHERE job_category_label LIKE '%Nurs%')
   ```

#### Long-term Solutions:

- **Populate HCRIS Cost Center Data**: Import department-level cost data from CMS HCRIS reports
- **Create Nursing Department Mapping**: Develop logic to identify nursing-related cost centers

---

## 2. Top Departments (Spend)

### **Business Objective**

Highlight departments driving highest labor spend with peer median comparisons and trend analysis.

### **Query Strategy & Thought Process**

Multi-layered approach for departmental analysis:

1. **Department Ranking**: Order departments by absolute spend
2. **Peer Benchmarking**: Calculate median spend by department across peer hospitals
3. **Statistical Distribution**: Provide percentile analysis for each department
4. **Historical Trends**: 12-month sparklines showing departmental spend evolution

### **Tables Used & Relevance Analysis**

| Table                    | Rows   | Status                  | Role & Importance                                                       |
| ------------------------ | ------ | ----------------------- | ----------------------------------------------------------------------- |
| `fact_hcris_cost_center` | 0      | ❌ **CRITICAL MISSING** | **Absolutely Essential** - Contains department-level wage and hour data |
| `dim_cost_center`        | 156    | ✅ Available            | **Department Definitions** - Cost center names and classifications      |
| `dim_hospital`           | 6,918  | ✅ Available            | **Hospital Context** - Facility identification for peer matching        |
| `dim_zip_cbsa`           | 40,773 | ✅ Available            | **Peer Grouping** - Geographic matching for peer identification         |

### **Implementation Status**

- **❌ Complete Failure**: Primary data source is empty
- **🔄 Alternative Available**: CMS job category data can substitute

### **Workarounds & Solutions**

#### Primary Alternative - CMS Job Category Analysis:

```sql
-- Replace department analysis with job category analysis
SELECT
    dcjs.job_category_label as department_name,
    SUM(fcoj.wages_paid) as total_spend,
    SUM(fcoj.hours_paid) as total_hours,
    AVG(fcoj.wages_paid) as avg_spend
FROM fact_cms_oms_job_category fcoj
JOIN dim_cms_oms_jobcategory_soc dcjs ON fcoj.job_category_code = dcjs.job_category_code
WHERE fcoj.hospital_ccn = :target_hospital
GROUP BY dcjs.job_category_label
ORDER BY total_spend DESC;
```

#### Mapping Strategy:

- **Available Job Categories**: 7 categories in `dim_cms_oms_jobcategory_soc`
- **Granularity Trade-off**: Less detailed than true cost centers but more actionable than facility-level data
- **Peer Comparison**: Still possible using same job categories across peer facilities

---

## 3. Facility Information

### **Business Objective**

Provide concise facility profile displaying attributes used for peer matching with clickable department chips.

### **Query Strategy & Thought Process**

Straightforward dimensional lookup joining related tables to create comprehensive facility profile with hierarchical geographic and ownership information.

### **Tables Used & Relevance Analysis**

| Table                    | Rows   | Status       | Role & Importance                                                         |
| ------------------------ | ------ | ------------ | ------------------------------------------------------------------------- |
| `dim_hospital`           | 6,918  | ✅ Available | **Primary Profile** - Name, CCN, address, facility type, bed counts       |
| `dim_zip_cbsa`           | 40,773 | ✅ Available | **Geographic Context** - CBSA codes, metropolitan area classification     |
| `dim_state`              | 59     | ✅ Available | **Location Hierarchy** - State names and codes                            |
| `dim_ownership_category` | 4      | ✅ Available | **Ownership Classification** - Government, non-profit, for-profit details |

### **Implementation Status**

- **✅ Fully Functional**: All facility profile queries work perfectly
- **⚠️ Department Chips**: Limited by missing cost center data

### **Department Listing Workaround**:

```sql
-- Use available cost center definitions
SELECT DISTINCT cost_center as available_departments
FROM dim_cost_center
ORDER BY cost_center;

-- Or use job categories as department proxies
SELECT DISTINCT job_category_label as department_equivalent
FROM dim_cms_oms_jobcategory_soc;
```

---

## 4. Pay Rate Analysis

### **Business Objective**

Display facility pay rates against CBSA and peer percentiles using BLS OEWS data with job group filtering.

### **Query Strategy & Thought Process**

Dual-source strategy comparing external market data with internal rates:

1. **External Benchmarking**: BLS OEWS provides robust market data by occupation and geography
2. **Internal Comparison**: HWL rate data for competitive positioning
3. **Statistical Analysis**: Percentile distributions and trend analysis
4. **Geographic Granularity**: CBSA-level market analysis

### **Tables Used & Relevance Analysis**

| Table           | Rows    | Status           | Role & Importance                                                              |
| --------------- | ------- | ---------------- | ------------------------------------------------------------------------------ |
| `fact_bls_oews` | 159,739 | ✅ **Excellent** | **Comprehensive Market Data** - Mean wages, percentiles by occupation and CBSA |
| `dim_soc`       | 168     | ✅ Available     | **Job Classification** - Standard Occupational Classification mapping          |
| `dim_zip_cbsa`  | 40,773  | ✅ Available     | **Geographic Matching** - CBSA to hospital location mapping                    |
| `fact_hwl`      | 0       | ❌ Missing       | **Internal Rates** - HWL historical rate data for competitive analysis         |

### **Implementation Status**

- **✅ Strong External Analysis**: BLS market data provides comprehensive benchmarking
- **❌ Missing Internal Context**: Cannot compare internal rates to market
- **📊 Rich Analytics**: Percentile analysis, trend tracking available for market data

### **Available Analysis Examples**:

```sql
-- Comprehensive market analysis works perfectly
SELECT
    soc.job_title,
    oews.mean_hourly_wage,
    oews.hourly_pct25, oews.hourly_median, oews.hourly_pct75,
    oews.total_employees,
    zc.cbsa_title
FROM fact_bls_oews oews
JOIN dim_soc soc ON oews.soc_id = soc.id
JOIN dim_zip_cbsa zc ON oews.cbsa_code = zc.cbsa_code
WHERE oews.cbsa_code = :target_cbsa
ORDER BY oews.mean_hourly_wage DESC;
```

### **Workarounds for Missing Internal Data**:

1. **Focus on Market Intelligence**: Emphasize external benchmarking capabilities
2. **Use Labor Exchange Data**: Leverage `fact_le` when populated for internal LE rate comparison
3. **Create Rate Input Interface**: Allow manual entry of internal HWL rates for comparison

---

## 5. Total Compensation Analysis

### **Business Objective**

Comprehensive compensation view including base salary, bonuses, equity, stipends with market positioning and stacked visualization.

### **Query Strategy & Thought Process**

Complex multi-component compensation analysis:

1. **Component Aggregation**: Sum base pay, housing stipend, meal stipend, travel allowance
2. **Market Positioning**: Compare total compensation to market percentiles
3. **Trend Analysis**: Year-over-year compensation changes
4. **Geographic Adjustment**: CBSA-specific compensation analysis

### **Tables Used & Relevance Analysis**

| Table                             | Rows  | Status                     | Role & Importance                                                                   |
| --------------------------------- | ----- | -------------------------- | ----------------------------------------------------------------------------------- |
| `fact_le`                         | 0     | ❌ **COMPLETE DEPENDENCY** | **Internal Labor Exchange Data** - LE compensation components, stipends, allowances |
| `dim_specialty_job_title_mapping` | 602   | ✅ Available               | **Job Classification** - Links specialties to job titles for internal systems       |
| `dim_job_title`                   | 247   | ✅ Available               | **Job Definitions** - Job titles and groups for internal compensation analysis      |
| `dim_hospital`                    | 6,918 | ✅ Available               | **Geographic Context** - Hospital location for internal Labor Exchange CBSA mapping |

### **Implementation Status**

- **❌ Complete Failure**: Primary data source is completely empty
- **🔄 Supporting Infrastructure**: Job classification tables available for future implementation

### **Comprehensive Workaround Strategy**:

#### Alternative Data Sources:

1. **CMS Wage Data**: Use `fact_cms_oms_job_category` for base wage estimates
2. **BLS Market Data**: Leverage `fact_bls_oews` for market benchmarking
3. **GSA Per Diem**: Calculate housing/meal stipends using GSA rates by location

```sql
-- Estimated compensation using available data
SELECT
    jt.job_group,
    AVG(coj.avg_hourly_wage * 40 * 52) as estimated_annual_base,
    -- Add GSA per diem rates for stipends
    (SELECT mean_annual_wage FROM fact_bls_oews bls
     JOIN dim_soc soc ON bls.soc_id = soc.id
     WHERE soc.job_title SIMILAR TO jt.job_title
     AND bls.cbsa_code = :target_cbsa) as market_benchmark
FROM fact_cms_oms_job_category coj
JOIN dim_hospital h ON coj.hospital_id = h.id
JOIN dim_job_title jt ON /* mapping logic */
```

---

## 6. Monthly Revenue Analysis

### **Business Objective**

Track monthly revenue from internal Labor Exchange operations and filled positions with month-over-month change analysis.

### **Query Strategy & Thought Process**

Revenue calculation based on internal LE billing rates and utilization:

1. **Time Aggregation**: Monthly revenue rollups from internal LE filled positions
2. **Rate Calculation**: Internal LE hourly bill rate × hours per week × utilization
3. **Trend Analysis**: Month-over-month and year-over-year comparisons of internal operations
4. **Volume Analysis**: Internal LE position count trends alongside revenue

### **Tables Used & Relevance Analysis**

| Table     | Rows | Status                 | Role & Importance                                                                           |
| --------- | ---- | ---------------------- | ------------------------------------------------------------------------------------------- |
| `fact_le` | 0    | ❌ **SOLE DEPENDENCY** | **Internal LE Revenue Source** - Internal Labor Exchange billing rates, hours, filled dates |

### **Implementation Status**

- **❌ No Internal Data Available**: Cannot calculate actual revenue without internal Labor Exchange filled position data
- **📊 Framework Ready**: Query logic is sound for future internal LE implementation

### **Alternative Revenue Approaches**:

#### Estimated Revenue Models:

1. **Contract-Based**: Use `fact_cms_cost_reports` contract spending as revenue proxy
2. **Market-Based**: Estimate revenue using market rates × estimated volume
3. **Capacity-Based**: Hospital bed count × estimated revenue per bed

```sql
-- Revenue estimation using contract data
SELECT
    DATE_TRUNC('month', ccr.fiscal_year_end) as revenue_month,
    SUM(ccr.contract_wages / 12) as estimated_monthly_revenue,
    COUNT(DISTINCT ccr.hospital_id) as active_facilities
FROM fact_cms_cost_reports ccr
WHERE ccr.report_year = :current_year
GROUP BY DATE_TRUNC('month', ccr.fiscal_year_end)
ORDER BY revenue_month DESC;
```

---

## 7. System Health, Active Users, Total Projects

### **Business Objective**

Operational dashboards for system monitoring, user engagement tracking, and project management oversight.

### **Implementation Status**

- **📝 Placeholder Queries**: Marked as requiring separate operational databases
- **🔄 Out of Scope**: Not part of healthcare data schema
- **💡 Recommendation**: Implement using application-specific logging and project management systems

### **Alternative Approaches**:

1. **Application Logs**: Query web application databases for user activity
2. **System Monitoring**: Integrate with monitoring tools (Datadog, New Relic, etc.)
3. **Project Management**: Connect to project tracking systems (Jira, Asana, etc.)

---

## Critical Data Gaps Analysis

### **High Priority Missing Data**

#### 1. Department-Level Cost Analysis (`fact_hcris_cost_center`)

- **Impact**: Eliminates department spending analysis and nursing spend benchmarking
- **Business Value**: High - Critical for departmental budgeting and resource allocation
- **Workaround Viability**: Medium - CMS job category data provides alternative

#### 2. Internal HWL Rate Data (`fact_hwl`)

- **Impact**: No internal vs external rate comparison capabilities
- **Business Value**: High - Essential for competitive positioning
- **Data Type**: Internal HWL staffing operations
- **Workaround Viability**: Low - Market data alone provides limited actionability

#### 3. Internal Labor Exchange Compensation (`fact_le`)

- **Impact**: No total compensation analysis or revenue tracking from internal LE operations
- **Business Value**: Very High - Core to HWL business model
- **Data Type**: Internal Labor Exchange (LE) compensation and revenue data
- **Workaround Viability**: Medium - Can estimate using market data and contract information

### **Data Quality Assessment**

#### Excellent Data Quality:

- **External Market Data**: BLS OEWS data is comprehensive and current
- **Facility Characteristics**: Hospital dimension data is complete and well-structured
- **Geographic Data**: CBSA and state mappings are comprehensive

#### Adequate Data Quality:

- **CMS Reporting Data**: Good coverage for basic facility metrics
- **Job Classifications**: Sufficient for basic analysis

#### Missing Data:

- **Internal HWL Operations**: No HWL staffing operational data
- **Internal Labor Exchange Operations**: No LE compensation and revenue data
- **Real-time Metrics**: No current period operational data for internal systems

---

## Implementation Recommendations

### **Phase 1: Immediate Implementation (Available Data)**

**Timeline**: 2-4 weeks
**Scope**: Leverage existing robust datasets

#### High-Value Features:

1. **Market Intelligence Dashboard**:

   - BLS wage benchmarking by occupation and geography
   - Market trend analysis using 159,739 wage records
   - CBSA-level compensation analysis

2. **Facility Profile & Basic Peer Comparison**:

   - Complete facility information display
   - Basic peer matching on facility-level metrics
   - Geographic and ownership-based grouping

3. **Basic Financial Analysis**:
   - Facility-level cost analysis using HCRIS data
   - Basic peer financial benchmarking

#### Implementation Priority:

```sql
-- Phase 1 Query Example: Market Intelligence
SELECT
    soc.job_title,
    soc.soc_major_group,
    AVG(oews.mean_hourly_wage) as avg_market_rate,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY oews.mean_hourly_wage) as median_rate,
    COUNT(*) as market_data_points
FROM fact_bls_oews oews
JOIN dim_soc soc ON oews.soc_id = soc.id
WHERE oews.cbsa_code = :target_cbsa
GROUP BY soc.job_title, soc.soc_major_group
ORDER BY avg_market_rate DESC;
```

### **Phase 2: Workaround Implementation (Alternative Data)**

**Timeline**: 4-6 weeks
**Scope**: Implement department analysis using CMS job category data

#### Enhanced Features:

1. **Job Category Analysis**:

   - Department-level spending using CMS job categories
   - Peer comparison by job category
   - Trend analysis using available historical data

2. **Estimated Compensation Analysis**:
   - Base wage analysis using CMS data
   - Market-based stipend calculations
   - Total compensation estimates

#### Key Implementation:

```sql
-- Phase 2 Query Example: Department Analysis Alternative
WITH job_category_spend AS (
    SELECT
        dcjs.job_category_label,
        fcoj.hospital_ccn,
        SUM(fcoj.wages_paid) as category_spend,
        SUM(fcoj.hours_paid) as category_hours
    FROM fact_cms_oms_job_category fcoj
    JOIN dim_cms_oms_jobcategory_soc dcjs ON fcoj.job_category_code = dcjs.job_category_code
    WHERE fcoj.survey_year = :current_year
    GROUP BY dcjs.job_category_label, fcoj.hospital_ccn
)
SELECT
    job_category_label,
    AVG(category_spend) as avg_peer_spend,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY category_spend) as median_spend
FROM job_category_spend
GROUP BY job_category_label
ORDER BY avg_peer_spend DESC;
```

### **Phase 3: Full Implementation (Complete Data)**

**Timeline**: 8-12 weeks
**Scope**: Populate missing tables and implement full feature set

#### Data Collection Priorities:

1. **HCRIS Cost Center Data**:

   - Import department-level cost data from CMS HCRIS reports
   - Establish automated data pipeline for ongoing updates

2. **Internal HWL Operational Data**:

   - Populate `fact_hwl` with historical HWL staffing rate data
   - Populate `fact_le` with Labor Exchange compensation and revenue data
   - Establish data pipelines for both internal systems

3. **Operational Systems Integration**:
   - Connect to HWL internal systems for real-time data
   - Implement user activity and project tracking

#### Full Feature Enablement:

- Complete department spending analysis
- Comprehensive internal vs external rate comparison
- Total compensation analysis with all components
- Revenue tracking and forecasting
- Real-time operational dashboards

---

## Business Impact Assessment

### **Available Capabilities (Immediate Value)**

- **Market Intelligence**: Comprehensive external wage benchmarking
- **Peer Analysis**: Facility-level comparative analysis
- **Geographic Analysis**: CBSA-level market analysis
- **Financial Benchmarking**: Basic facility cost comparison

**Business Value**: **High** - Provides immediate competitive intelligence and market positioning

### **Limited Capabilities (Workaround Value)**

- **Department Analysis**: Job category-based spending analysis
- **Estimated Compensation**: Market-based compensation estimates
- **Trend Analysis**: Historical facility-level trends

**Business Value**: **Medium** - Provides operational insights with some limitations

### **Missing Capabilities (Future Value)**

- **Internal Rate Optimization**: Cannot optimize internal rates vs market
- **Department-Level Budgeting**: Limited departmental cost management
- **Revenue Management**: No revenue tracking or optimization
- **Real-time Operations**: No current operational dashboards

**Business Value Impact**: **High** - Significant operational optimization opportunities lost

### **ROI Calculation**

#### Phase 1 Implementation:

- **Development Cost**: Low (2-4 weeks)
- **Business Value**: High market intelligence and peer benchmarking
- **ROI**: **Immediate positive** - Market positioning and competitive analysis

#### Phase 2 Implementation:

- **Development Cost**: Medium (4-6 weeks additional)
- **Business Value**: Operational insights and departmental analysis
- **ROI**: **Positive within 3 months** - Departmental optimization opportunities

#### Phase 3 Implementation:

- **Development Cost**: High (8-12 weeks additional)
- **Business Value**: Complete operational optimization
- **ROI**: **High within 6 months** - Full revenue optimization and cost management

---

## Technical Architecture Considerations

### **Database Performance**

- **Large Tables**: `fact_bls_oews` (159,739 rows) - Consider indexing on cbsa_code, soc_id
- **Join Optimization**: Complex multi-table joins require proper indexing strategy
- **Aggregation Performance**: Percentile calculations on large datasets need optimization

### **Data Pipeline Requirements**

1. **External Data**: Automated BLS OEWS data updates (annual)
2. **CMS Data**: Regular HCRIS and OMS data synchronization
3. **Internal Data**: HWL operational system integration

### **Scalability Planning**

- **Query Caching**: Implement for frequently accessed peer comparisons
- **Data Partitioning**: Consider partitioning fact tables by year/geography
- **API Design**: RESTful APIs for dashboard frontend integration

---

## Conclusion

This analysis reveals a healthcare data environment with **excellent external market intelligence capabilities** but **significant gaps in internal operational data**. The database schema is well-designed and the available data is high-quality, but critical HWL-specific tables are unpopulated.

### **Key Success Factors**:

1. **Start with strengths**: Implement market intelligence features immediately
2. **Creative workarounds**: Use CMS job category data for departmental insights
3. **Phased approach**: Build value incrementally while addressing data gaps
4. **Business alignment**: Focus on high-ROI features that work with available data

### **Strategic Recommendation**:

**Proceed with implementation using available data while simultaneously investing in populating missing tables**. The external market data alone provides significant business value and competitive advantage, while the workarounds for department-level analysis provide operational insights that justify continued investment in complete data collection.

This approach ensures immediate business value while building toward a comprehensive healthcare analytics platform.
