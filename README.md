# ORCHID Organ Donation Data Analysis (PostgreSQL)

## Project Overview

This project analyzes organ donation and procurement data using the ORCHID dataset from PhysioNet. The goal is to explore inefficiencies in the organ procurement process and identify factors influencing successful organ retrieval. The project demonstrates database design, SQL analytics, and query optimization techniques using PostgreSQL.

## Dataset

Organ Retrieval and Collection of Health Information for Donation (ORCHID)

The dataset includes:
- 133,000+ donor referrals
- 9 relational tables
- clinical events and laboratory tests
- organ procurement outcomes

## Tools Used

-PostgreSQL
-SQL
-Database Modeling

## Database Design

A normalized relational schema was designed to store and analyze the dataset efficiently by including
- primary keys
- foreign key relationships
- indexed columns
Relationships were implemented using primary keys and foreign key constraints to maintain data integrity.

## Analysis Performed

- donor demographic analysis
- organ procurement success rates
- lab value analysis
- time-to-procurement metrics
- performance optimization

## Key SQL Concepts Demonstrated

- Complex JOIN operations across multiple healthcare tables
- Aggregation queries for donor and organ retrieval analysis
- Window functions for ranking and analytical calculations
- Common Table Expressions (CTEs) for modular query logic
- Materialized Views for faster analytical queries
- User Defined Functions (UDFs) for reusable calculations
- Function Overloading in PostgreSQL
- Trigger Functions for automated data validation
- Query optimization using indexes and EXPLAIN ANALYZE
- Database normalization and relational schema design

## Example Query

--Function Overloading to find the efficiency of each procured organ based on conditions: all/any the cause_of_death_unos or any causes of donors death (i.e. mechanism_of_death, cause_of_death_opo, cause_of_death_unos, circumstance_of_death)

--Function to find the efficiency of each procured organ based on all/any cause_of_death_unos

CREATE OR REPLACE FUNCTION dbo.get_procurement_efficiency(cod_filter text DEFAULT NULL)

RETURNS TABLE(

    cause_of_death_unos text,
    hearts_efficiency numeric,
    livers_efficiency numeric,
    left_kidneys_efficiency numeric,
    right_kidneys_efficiency numeric,
    left_lungs_efficiency numeric,
    right_lungs_efficiency numeric,
    pancreas_efficiency numeric,
    intestines_efficiency numeric

) AS $$

BEGIN

    RETURN QUERY
    SELECT
        pc.cause_of_death_unos,
        ROUND(100.0 * SUM((o.heart IS NOT NULL)::int) / NULLIF(SUM(1)::int,0), 2) AS hearts_efficiency,
        ROUND(100.0 * SUM((o.liver IS NOT NULL)::int) / NULLIF(SUM(1)::int,0), 2) AS livers_efficiency,
        ROUND(100.0 * SUM((o.left_kidney IS NOT NULL)::int) / NULLIF(SUM(1)::int,0), 2) AS left_kidneys_efficiency,
        ROUND(100.0 * SUM((o.right_kidney IS NOT NULL)::int) / NULLIF(SUM(1)::int,0), 2) AS right_kidneys_efficiency,
        ROUND(100.0 * SUM((o.left_lung IS NOT NULL)::int) / NULLIF(SUM(1)::int,0), 2) AS left_lungs_efficiency,
        ROUND(100.0 * SUM((o.right_lung IS NOT NULL)::int) / NULLIF(SUM(1)::int,0), 2) AS right_lungs_efficiency,
        ROUND(100.0 * SUM((o.pancreas IS NOT NULL)::int) / NULLIF(SUM(1)::int,0), 2) AS pancreas_efficiency,
        ROUND(100.0 * SUM((o.intestine IS NOT NULL)::int) / NULLIF(SUM(1)::int,0), 2) AS intestines_efficiency
    FROM dbo.patient_cod pc
    LEFT JOIN dbo.outcomes o  ON o.patientid = pc.patientid
    WHERE cod_filter IS NULL OR pc.cause_of_death_unos = cod_filter
    GROUP BY pc.cause_of_death_unos
    ORDER BY pc.cause_of_death_unos; 

END;
$$ LANGUAGE plpgsql;

-- -- a. based on all the cause_of_death_unos 
-- SELECT * FROM dbo.get_procurement_efficiency();

-- --b.based on any cause_of_death_unos
-- SELECT * FROM dbo.get_procurement_efficiency('Anoxia');

--c.Function to find the efficiency of each procured organ based on any causes of donors death ( i.e. mechanism_of_death, cause_of_death_opo, cause_of_death_unos, circumstance_of_death) ?

CREATE OR REPLACE FUNCTION dbo.get_procurement_efficiency(f_value TEXT,f_column INTEGER) 

RETURNS TABLE(

    filter_value TEXT,
    hearts_efficiency NUMERIC,
    livers_efficiency NUMERIC,
    kidneys_efficiency NUMERIC,
    lungs_efficiency NUMERIC,
    pancreas_efficiency NUMERIC,
    intestine_efficiency NUMERIC

) AS $$

DECLARE
    col_name TEXT;
    sql TEXT;

BEGIN

    CASE f_column
        WHEN 1 THEN col_name := 'mechanism_of_death';
        WHEN 2 THEN col_name := 'cause_of_death_opo';
        WHEN 3 THEN col_name := 'cause_of_death_unos';
        WHEN 4 THEN col_name := 'circumstances_of_death';
        ELSE RAISE EXCEPTION 'Invalid filter_column value';
    END CASE;
    sql := format($f$
        SELECT
            %L AS filter_value,
            ROUND(100.0 * SUM((o.heart IS NOT NULL)::int) / NULLIF(COUNT(*),0),2) AS hearts_efficiency,
            ROUND(100.0 * SUM((o.liver IS NOT NULL)::int) / NULLIF(COUNT(*),0),2) AS livers_efficiency,
            ROUND(100.0 * SUM((o.left_kidney IS NOT NULL OR o.right_kidney IS NOT NULL)::int) / NULLIF(COUNT(*),0),2) AS kidneys_efficiency,
            ROUND(100.0 * SUM((o.left_lung IS NOT NULL OR o.right_lung IS NOT NULL)::int) / NULLIF(COUNT(*),0),2) AS lungs_efficiency,
            ROUND(100.0 * SUM((o.pancreas IS NOT NULL)::int) / NULLIF(COUNT(*),0),2) AS pancreas_efficiency,
            ROUND(100.0 * SUM((o.intestine IS NOT NULL)::int) / NULLIF(COUNT(*),0),2) AS intestine_efficiency
        FROM dbo.patient_cod pc
        LEFT JOIN dbo.outcomes o ON o.patientid = pc.patientid
        WHERE %I = %L
    $f$, f_value, col_name, f_value);
    RETURN QUERY EXECUTE sql;

END;
$$ LANGUAGE plpgsql;

-- --c.Based on any causes of donors death ( i.e. mechanism_of_death, cause_of_death_opo, cause_of_death_unos, circumstance_of_death)
-- SELECT * FROM dbo.get_procurement_efficiency('Asphyxiation', 1); -- 1 = mechanism_of_death

## Insights

- Identified variation in procurement rates across OPOs
- Measured the contribution of each age to the procurement of each organ
- Analysed the procurement status rate (i.e. Transplanted, Recovered for Research or Recovered but Not Transplanted) of any organ categorized by the cause of death
- Find the efficiency of each procured organ based on all/any the cause_of_death_unos or any causes of donors death
- Observed organ procurement outcomes that can be standardized into a longitudinal format to enable consistent, high-performance analysis of transplant success, recovery, and research utilization across all organs
- Measured time delays in procurement workflows
