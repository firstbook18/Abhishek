# Troubleshooting Oracle Fusion Pricing Table Join Issues

## Problem: Join Returns Too Many Records

When joining QP_PRICE_LIST_INT and QP_PRICE_LIST_ITEMS_INT returns unexpectedly high record counts, here's how to diagnose and fix it:

## Step 1: Analyze Data Distribution

### Check Record Counts in Each Table
```sql
-- Count records in header table
SELECT 
    COUNT(*) as total_headers,
    COUNT(DISTINCT price_list_id) as unique_price_lists,
    COUNT(DISTINCT set_process_id) as unique_batches
FROM qp_price_list_int;

-- Count records in items table  
SELECT 
    COUNT(*) as total_items,
    COUNT(DISTINCT price_list_id) as unique_price_lists,
    COUNT(DISTINCT set_process_id) as unique_batches
FROM qp_price_list_items_int;
```

### Identify Duplicate Headers
```sql
-- Find duplicate price_list_ids in header table
SELECT 
    price_list_id,
    COUNT(*) as duplicate_count,
    MIN(creation_date) as first_created,
    MAX(creation_date) as last_created
FROM qp_price_list_int
GROUP BY price_list_id
HAVING COUNT(*) > 1
ORDER BY duplicate_count DESC;
```

## Step 2: Check for Cartesian Product Issues

### Verify Join Multiplier Effect
```sql
-- Analyze join multiplication
WITH header_counts AS (
    SELECT price_list_id, COUNT(*) as header_count
    FROM qp_price_list_int
    GROUP BY price_list_id
),
item_counts AS (
    SELECT price_list_id, COUNT(*) as item_count  
    FROM qp_price_list_items_int
    GROUP BY price_list_id
)
SELECT 
    h.price_list_id,
    h.header_count,
    i.item_count,
    (h.header_count * i.item_count) as expected_join_records
FROM header_counts h
FULL OUTER JOIN item_counts i ON h.price_list_id = i.price_list_id
WHERE h.header_count > 1 OR i.item_count > 10  -- Highlight potential issues
ORDER BY expected_join_records DESC;
```

## Step 3: Add Proper Filtering

### Filter by Process Status
```sql
-- Only join current/pending records
SELECT 
    h.price_list_name,
    l.item_number,
    l.list_price
FROM qp_price_list_int h
INNER JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
WHERE h.process_flag = 1  -- Pending only
    AND l.process_flag = 1  -- Pending only
    AND h.set_process_id = l.set_process_id;  -- Same batch
```

### Filter by Batch/Request ID
```sql
-- Filter by specific batch
SELECT 
    h.price_list_name,
    l.item_number,
    l.list_price
FROM qp_price_list_int h
INNER JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
WHERE h.set_process_id = :p_batch_id  -- Specific batch only
    AND l.set_process_id = :p_batch_id
    AND h.request_id = l.request_id;    -- Same request
```

## Step 4: Use Analytical Approach

### Identify Most Recent Records Only
```sql
-- Get only latest version of each price list
WITH latest_headers AS (
    SELECT 
        price_list_id,
        price_list_name,
        currency_code,
        ROW_NUMBER() OVER (
            PARTITION BY price_list_id 
            ORDER BY creation_date DESC, last_update_date DESC
        ) as rn
    FROM qp_price_list_int
    WHERE process_flag IN (1, 7)  -- Pending or Success
)
SELECT 
    h.price_list_name,
    h.currency_code,
    l.item_number,
    l.list_price
FROM latest_headers h
INNER JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
WHERE h.rn = 1  -- Only latest header version
    AND l.process_flag IN (1, 7);
```

## Step 5: Check for Historical Data

### Clean Up Old Interface Records
```sql
-- Check for old processed records
SELECT 
    process_flag,
    COUNT(*) as record_count,
    MIN(creation_date) as oldest_record,
    MAX(creation_date) as newest_record
FROM qp_price_list_int
GROUP BY process_flag
ORDER BY process_flag;

-- Consider filtering out successfully processed records
SELECT 
    h.price_list_name,
    l.item_number,
    l.list_price
FROM qp_price_list_int h
INNER JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
WHERE h.process_flag = 1  -- Only pending
    AND l.process_flag = 1
    AND h.creation_date >= SYSDATE - 7;  -- Only recent records
```

## Step 6: Use Proper Join Strategy

### One-to-Many Relationship (Correct Approach)
```sql
-- This is the CORRECT join for one header to many items
SELECT 
    h.price_list_id,
    h.price_list_name,
    h.currency_code,
    COUNT(l.price_list_line_id) as item_count,
    MIN(l.list_price) as min_price,
    MAX(l.list_price) as max_price,
    AVG(l.list_price) as avg_price
FROM qp_price_list_int h
INNER JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
WHERE h.process_flag = 1
    AND l.process_flag = 1
    AND h.set_process_id = l.set_process_id
GROUP BY h.price_list_id, h.price_list_name, h.currency_code
ORDER BY h.price_list_name;
```

### If You Need Unique Headers Only
```sql
-- If you only want to see each header once with sample item data
SELECT DISTINCT
    h.price_list_id,
    h.price_list_name,
    h.currency_code,
    h.start_date_active,
    h.end_date_active
FROM qp_price_list_int h
WHERE EXISTS (
    SELECT 1 
    FROM qp_price_list_items_int l
    WHERE l.price_list_id = h.price_list_id
        AND l.process_flag = 1
)
AND h.process_flag = 1;
```

## Step 7: Diagnostic Query

### Complete Diagnostic Analysis
```sql
-- Comprehensive diagnostic query
SELECT 
    'Analysis' as metric,
    'Headers Total' as description,
    COUNT(*) as count_value
FROM qp_price_list_int

UNION ALL

SELECT 
    'Analysis',
    'Items Total',
    COUNT(*)
FROM qp_price_list_items_int

UNION ALL

SELECT 
    'Analysis',
    'Join Result Count',
    COUNT(*)
FROM qp_price_list_int h
INNER JOIN qp_price_list_items_int l ON h.price_list_id = l.price_list_id

UNION ALL

SELECT 
    'Analysis',
    'Expected Max (Headers × Max Items per Header)',
    h.total_headers * i.max_items_per_header
FROM (
    SELECT COUNT(*) as total_headers FROM qp_price_list_int
) h,
(
    SELECT MAX(item_count) as max_items_per_header
    FROM (
        SELECT price_list_id, COUNT(*) as item_count
        FROM qp_price_list_items_int
        GROUP BY price_list_id
    )
) i;
```

## Recommended Solutions

### Solution 1: Most Restrictive Join
```sql
SELECT 
    h.price_list_name,
    l.item_number,
    l.list_price,
    l.unit_of_measure
FROM qp_price_list_int h
INNER JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
    AND h.set_process_id = l.set_process_id
    AND h.request_id = l.request_id
WHERE h.process_flag = 1
    AND l.process_flag = 1
    AND h.creation_date >= TRUNC(SYSDATE) - 1  -- Only recent
ORDER BY h.price_list_name, l.item_number;
```

### Solution 2: Aggregated View
```sql
SELECT 
    h.price_list_name,
    h.currency_code,
    COUNT(l.price_list_line_id) as total_items,
    MIN(l.list_price) as min_price,
    MAX(l.list_price) as max_price
FROM qp_price_list_int h
LEFT JOIN qp_price_list_items_int l
    ON h.price_list_id = l.price_list_id
    AND l.process_flag = 1
WHERE h.process_flag = 1
GROUP BY h.price_list_id, h.price_list_name, h.currency_code
ORDER BY h.price_list_name;
```

## Prevention Tips

1. **Always filter by process_flag** to avoid old records
2. **Use set_process_id** for batch-specific queries  
3. **Include request_id** when tracking specific imports
4. **Consider date filters** to limit scope
5. **Use DISTINCT** when appropriate for header-only queries
6. **Aggregate when needed** instead of showing all combinations

## Next Steps

1. Run the diagnostic queries above
2. Identify which scenario matches your situation
3. Apply the appropriate solution
4. Always validate record counts before and after