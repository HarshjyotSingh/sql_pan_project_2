# PAN Number Validation Project

## Data Cleaning and Validation Objective

This project focuses on cleaning and validating a dataset containing Permanent Account Numbers (PAN) of Indian nationals. The goal is to ensure that each PAN number adheres to the official format and is categorized as either **Valid** or **Invalid**.

## Project Overview

A comprehensive PostgreSQL-based solution for validating Indian PAN (Permanent Account Number) data with complete data quality checks and cleansing operations. This project processes datasets (originally from Excel files like `PAN Number Validation Dataset.xlsx`) and provides thorough validation against official PAN format requirements.

## Project Tasks Completed

### 1. Data Cleaning and Preprocessing
- **Missing Data Handling**: Identifies and processes missing PAN values appropriately
- **Duplicate Removal**: Detects and handles duplicate PAN numbers in the dataset  
- **Space Handling**: Removes leading/trailing spaces from PAN numbers
- **Case Correction**: Converts all PAN numbers to uppercase format

### 2. PAN Format Validation
The system validates PAN numbers against the official Indian PAN format requirements:

**Valid PAN Format: `AAAAA1234A`**
- Exactly 10 characters long
- **First 5 characters**: Alphabetic (uppercase letters)
  - Adjacent characters cannot be the same (AABCD ❌, AXBCD ✅)
  - Cannot form a complete sequence (ABCDE ❌, ABCDX ✅)
- **Next 4 characters**: Numeric (digits 0-9)
  - Adjacent digits cannot be the same (1123 ❌, 1923 ✅)  
  - Cannot form a complete sequence (1234 ❌, 1243 ✅)
- **Last character**: Alphabetic (uppercase letter)

**Example of Valid PAN**: `AHGVE1276F`

### 3. Categorization
- **Valid PAN**: Matches all format requirements
- **Invalid PAN**: Does not match format, incomplete, or contains non-alphanumeric characters

## Features

- **Complete Data Pipeline**: From raw data import to final validation results
- **Advanced Pattern Detection**: Custom functions for adjacent and sequential character validation
- **Regex Validation**: Pattern matching against official PAN format
- **Comprehensive Reporting**: Detailed summary statistics and categorization

## Database Schema

The project uses a simple table structure:

```sql
CREATE TABLE pan_project
(
    Pan_Numbers TEXT
);
```

## Validation Rules

The system validates PAN numbers against the following criteria:

1. **Format**: Must follow pattern `[A-Z]{5}[0-9]{4}[A-Z]` (5 letters + 4 digits + 1 letter)
2. **No Adjacent Characters**: Consecutive identical characters are not allowed
3. **No Sequential Patterns**: Sequential alphabetic or numeric sequences are invalid
4. **Data Quality**: Must not be null, empty, or contain only whitespace

## Key Functions

### check_adj_char(p_str TEXT)
Detects adjacent identical characters in a string.

```sql
CREATE OR REPLACE Function check_adj_char(p_str TEXT)
RETURNS BOOLEAN
LANGUAGE plpgsql
AS $$
begin
    for i in 1 .. (length(p_str))-1
    loop
        if substring(p_str,i,1)=substring(p_str,i+1,1)
        THEN
            return true;
        end if;
    end loop;
    return false;
end;
$$
```

### check_sqen_char(p_str TEXT)
Identifies sequential character patterns in strings.

```sql
CREATE OR REPLACE FUNCTION check_sqen_char(p_str TEXT)
RETURNS BOOLEAN
LANGUAGE plpgsql
as $$
begin
    for i in 1 .. length(p_str)-1
    loop
        if ascii(substring(p_str,i,1))-ascii(substring(p_str,i+1,1))<>1
        THEN
            return false;
        end if;
    end loop;
    return true;
end;
$$
```

## Complete SQL Implementation

### Data Quality Assessment

**Missing Data Check:**
```sql
-- Check for missing data
SELECT * FROM pan_project WHERE Pan_Numbers IS NULL; --965 records
```

**Duplicate Detection:**
```sql
-- Find duplicates
SELECT Pan_Numbers, count(*)
FROM pan_project
GROUP BY Pan_Numbers
HAVING count(*) > 1; -- 5 duplicates + 965 nulls
```

**Trailing Spaces Check:**
```sql
-- Check for trailing spaces
SELECT * FROM pan_project
WHERE Pan_Numbers <> TRIM(Pan_Numbers); --9 records
```

**Case Inconsistency Check:**
```sql
-- Check for incorrect letter case
SELECT * FROM pan_project
WHERE Pan_Numbers <> UPPER(Pan_Numbers); --990 records
```

### Data Cleaning Query

```sql
-- Clean and standardize data
SELECT DISTINCT UPPER(TRIM(Pan_Numbers)) 
FROM pan_project
WHERE Pan_Numbers IS NOT NULL
AND TRIM(Pan_Numbers) <> '';
```

### Pattern Validation

```sql
-- Validate PAN format using regex
SELECT *
FROM pan_project
WHERE pan_numbers ~ '^[A-Z]{5}[0-9]{4}[A-Z]$';
```

### Complete Validation View

```sql
-- Create comprehensive validation view
CREATE OR REPLACE VIEW vw_valid_invalid_pans AS
WITH cte_cleaned_pan (pan_number) AS (
    SELECT DISTINCT UPPER(TRIM(Pan_Numbers)) AS pan_number 
    FROM pan_project
    WHERE Pan_Numbers IS NOT NULL
    AND TRIM(Pan_Numbers) <> ''
),
cte_valid_pan (pan_number) AS (
    SELECT pan_number 
    FROM cte_cleaned_pan
    WHERE check_adj_char(pan_number) = false
    AND check_sqen_char(substring(pan_number,1,5)) = false
    AND check_sqen_char(substring(pan_number,6,4)) = false
    AND pan_number ~ '^[A-Z]{5}[0-9]{4}[A-Z]$'
)
SELECT cln.pan_number,
    CASE
        WHEN vld.pan_number IS NOT NULL THEN 'Valid Pan' 
        ELSE 'Invalid Pan' 
    END AS status
FROM cte_cleaned_pan cln
LEFT JOIN cte_valid_pan vld ON cln.pan_number = vld.pan_number;
```

### Summary Report

```sql
-- Generate comprehensive summary report
WITH cte AS (
    SELECT 
        (SELECT COUNT(*) FROM pan_project) AS total_processed_records,
        COUNT(*) FILTER (WHERE vw.status = 'Valid Pan') AS total_valid_pans,
        COUNT(*) FILTER (WHERE vw.status = 'Invalid Pan') AS total_invalid_pans
    FROM vw_valid_invalid_pans vw
)
SELECT 
    total_processed_records, 
    total_valid_pans, 
    total_invalid_pans,
    total_processed_records - (total_valid_pans + total_invalid_pans) AS missing_incomplete_pans
FROM cte;
```

## Installation and Setup

1. **Prerequisites**
   - PostgreSQL database server
   - Appropriate database permissions for creating tables, functions, and views

2. **Database Setup**
   ```sql
   -- Create the main table
   CREATE TABLE pan_project (Pan_Numbers TEXT);
   
   -- Import your data from Excel file
   -- Run all function definitions and view creation queries
   ```

3. **Data Import**
   - Load your PAN number data from `PAN Number Validation Dataset.xlsx`
   - Ensure data is imported into the `Pan_Numbers` column

## Usage Examples

**View All Results:**
```sql
SELECT * FROM vw_valid_invalid_pans ORDER BY pan_number;
```

**Count Valid vs Invalid:**
```sql
SELECT status, COUNT(*) as count
FROM vw_valid_invalid_pans 
GROUP BY status;
```

**Test Custom Functions:**
```sql
-- Test adjacent character detection
SELECT check_adj_char('AABCD'); -- Returns true (invalid)
SELECT check_adj_char('AXBCD'); -- Returns false (valid)

-- Test sequential character detection
SELECT check_sqen_char('ABCDE'); -- Returns true (invalid sequence)
SELECT check_sqen_char('ABCDX'); -- Returns false (valid)
```

## Data Quality Issues Identified

- **Missing Data**: *965 null records* identified and handled
- **Duplicates**: *5 duplicate entries* plus null values detected
- **Formatting Issues**: *9 records* with trailing/leading spaces
- **Case Inconsistencies**: *990 records* with incorrect letter casing

## Technical Requirements

- **PostgreSQL 9.1+** (for plpgsql functions)
- Database user with **CREATE permissions** for functions and views
- Excel file processing capability for initial data import

## Project Results

The system successfully processes the entire dataset and provides:
- **Complete data quality assessment**
- **Automated data cleansing**
- **Comprehensive PAN validation**
- **Detailed categorization** (Valid/Invalid)
- **Statistical summary reports**

## Contributing

Feel free to submit issues, feature requests, or pull requests to improve the validation logic or add new data quality checks.

## License

This project is available under the **MIT License**.
