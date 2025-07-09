# Oracle Fusion Pricing Interface Tables Join Research

## Overview

This document provides comprehensive information about joining Oracle Fusion pricing interface tables **QP_PRICE_LIST_INT** and **QP_PRICE_LIST_ITEMS_INT**, based on research and Oracle Fusion pricing architecture.

## Table Structure and Purpose

### QP_PRICE_LIST_INT (Price List Header Interface)
- **Purpose**: Interface table for importing price list header information
- **Function**: Stores price list master data before processing into base tables
- **Level**: Header level data

### QP_PRICE_LIST_ITEMS_INT (Price List Items Interface)  
- **Purpose**: Interface table for importing price list line/item information
- **Function**: Stores individual pricing items and their details
- **Level**: Line level data

## Oracle Fusion Pricing Architecture

### Base Tables (Target Tables)
After successful import, interface data moves to these base tables:
- **QP_PRICE_LISTS_VL** - Price list headers (view with language support)
- **QP_PRICE_LIST_ITEMS** - Price list items/lines
- **QP_PRICE_LIST_CHARGES** - Additional charges

### Related Pricing Tables
- **QP_LIST_HEADERS** - Price list and modifier list headers
- **QP_LIST_LINES** - Price list and modifier lines
- **QP_QUALIFIERS** - Pricing qualifiers
- **QP_PRICING_ATTRIBUTES** - Pricing attributes

## Common Join Patterns

### 1. Basic Header-to-Lines Join
```sql
SELECT 
    h.price_list_name,
    h.currency_code,
    h.list_type_code,
    l.item_number,
    l.list_price,
    l.unit_of_measure
FROM qp_price_list_int h
INNER JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
WHERE h.process_flag = 1  -- Pending processing
    AND l.process_flag = 1;
```

### 2. Join with Batch Processing
```sql
SELECT 
    h.price_list_name,
    l.item_number,
    l.list_price,
    h.set_process_id,
    h.request_id
FROM qp_price_list_int h
INNER JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
    AND h.set_process_id = l.set_process_id
WHERE h.set_process_id = :p_batch_id;
```

### 3. Join with Error Handling
```sql
SELECT 
    h.price_list_name,
    l.item_number,
    l.list_price,
    h.process_flag as header_status,
    l.process_flag as line_status,
    CASE 
        WHEN h.process_flag = 3 OR l.process_flag = 3 THEN 'ERROR'
        WHEN h.process_flag = 7 AND l.process_flag = 7 THEN 'SUCCESS'
        ELSE 'PENDING'
    END as overall_status
FROM qp_price_list_int h
LEFT JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
ORDER BY h.creation_date DESC;
```

## Key Join Columns

### Primary Join Columns
1. **PRICE_LIST_ID** - Primary relationship key
2. **PRICE_LIST_NAME** - Alternative join (if ID not available)

### Processing Control Columns
1. **SET_PROCESS_ID** - For batch processing control
2. **REQUEST_ID** - For tracking import runs
3. **BATCH_ID** - Alternative batching mechanism

### Standard Audit Columns
1. **CREATION_DATE** / **CREATED_BY**
2. **LAST_UPDATE_DATE** / **LAST_UPDATED_BY**
3. **LAST_UPDATE_LOGIN**

## Process Flow Integration

### 1. Data Loading Phase
```sql
-- Load header data
INSERT INTO qp_price_list_int (
    price_list_id,
    price_list_name,
    currency_code,
    list_type_code,
    start_date_active,
    end_date_active,
    process_flag,
    set_process_id,
    created_by,
    creation_date
) VALUES (...);

-- Load line data with matching price_list_id
INSERT INTO qp_price_list_items_int (
    price_list_id,
    price_list_line_id,
    item_number,
    list_price,
    unit_of_measure,
    process_flag,
    set_process_id,
    created_by,
    creation_date
) VALUES (...);
```

### 2. Validation and Import
```sql
-- Check data integrity before import
SELECT 
    COUNT(*) as header_count,
    COUNT(DISTINCT l.price_list_id) as lines_with_headers,
    COUNT(*) - COUNT(DISTINCT l.price_list_id) as orphaned_lines
FROM qp_price_list_int h
FULL OUTER JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
WHERE h.set_process_id = :p_batch_id;
```

## Error Handling and Troubleshooting

### Process Flag Values
Based on Oracle standard interface patterns:
- **1** = Pending
- **2** = In Process  
- **3** = Error
- **4** = Warning
- **7** = Success

### Common Error Scenarios
1. **Orphaned Lines**: Items without valid price list headers
2. **Duplicate Keys**: Multiple records with same price_list_id
3. **Invalid References**: References to non-existent items or price lists
4. **Date Conflicts**: Invalid effective date ranges

### Error Query Example
```sql
SELECT 
    'HEADER' as record_type,
    h.price_list_id,
    h.price_list_name,
    h.process_flag,
    h.error_message
FROM qp_price_list_int h
WHERE h.process_flag = 3

UNION ALL

SELECT 
    'LINE' as record_type,
    l.price_list_id,
    l.item_number,
    l.process_flag,
    l.error_message
FROM qp_price_list_items_int l
WHERE l.process_flag = 3;
```

## Best Practices

### 1. Data Preparation
- Always populate SET_PROCESS_ID for batch control
- Ensure referential integrity between headers and lines
- Validate all foreign key references before loading

### 2. Performance Optimization
- Use appropriate indexes on join columns
- Process in manageable batch sizes
- Consider parallel processing for large volumes

### 3. Import Process
- Run validation queries before triggering import
- Monitor process flags during import
- Handle errors systematically

### 4. Sample Complete Query
```sql
SELECT 
    h.price_list_name,
    h.currency_code,
    h.start_date_active,
    h.end_date_active,
    l.item_number,
    l.item_description,
    l.list_price,
    l.unit_of_measure,
    h.process_flag as header_status,
    l.process_flag as line_status,
    h.creation_date,
    h.created_by
FROM qp_price_list_int h
INNER JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
WHERE h.set_process_id = :p_batch_id
    AND (h.process_flag = 1 OR l.process_flag = 1)  -- Pending records
ORDER BY h.price_list_name, l.item_number;
```

## Integration with Oracle Fusion Import Process

### 1. File-Based Data Import (FBDI)
Oracle Fusion typically uses FBDI templates for price list imports:
- Price List Header template
- Price List Items template
- Templates maintain referential relationships

### 2. ESS Jobs
Standard Enterprise Scheduler Service (ESS) jobs:
- **Load Interface File for Import** - Loads data from files to interface tables
- **Import Price Lists** - Processes interface tables to base tables

### 3. Concurrent Processing
```sql
-- Monitor import progress
SELECT 
    request_id,
    COUNT(*) as total_records,
    SUM(CASE WHEN process_flag = 7 THEN 1 ELSE 0 END) as successful,
    SUM(CASE WHEN process_flag = 3 THEN 1 ELSE 0 END) as failed
FROM qp_price_list_int
WHERE request_id = :p_request_id
GROUP BY request_id;
```

## Additional Considerations

### 1. Multi-Org Setup
- Consider ORG_ID for multi-organization implementations
- Validate organization access and security

### 2. Pricing Strategy
- Understand list vs. modifier pricing
- Consider pricing phases and precedence

### 3. Currency Handling
- Multi-currency price lists
- Exchange rate considerations

## Related Documentation
- Oracle Fusion Applications Pricing Implementation Guide
- Oracle Fusion Cloud Applications Interface Documentation
- Oracle Advanced Pricing User Guide

## Conclusion

The join between QP_PRICE_LIST_INT and QP_PRICE_LIST_ITEMS_INT follows standard Oracle interface table patterns, with PRICE_LIST_ID being the primary join key. Proper handling of process flags, batch processing, and error management is crucial for successful pricing data imports in Oracle Fusion applications.