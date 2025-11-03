# Snowflake Batch Load Optimization - 10× Throughput Improvement

## Main Story

"At Karmalife, I optimized our Snowflake data ingestion pipeline to achieve 6× faster throughput—reducing load time from 30 minutes to ~5 minutes—by implementing batch writes using `executemany()` and `write_pandas()`, while also building resilience against schema drift and invalid records."

---

## The Problem

**Context:**
Our ingestion jobs pulled partner and loan transaction data from REST APIs every 30 minutes and wrote to Snowflake. The legacy implementation used row-by-row inserts via `cursor.execute()` in a loop.

**Three Critical Issues:**

**1. Performance Bottleneck:**
- Each record triggered individual round-trip to Snowflake
- As data grew from thousands to millions of rows, load time exploded: 8 minutes → 30 minutes
- CPU utilization spiked, network latency became bottleneck
- Approaching our 10-minute ingestion window—risking data freshness SLA

**2. Schema Drift Failures:**
- Upstream partner APIs frequently added or renamed fields without notice
- When schema mismatched target table, entire batch failed
- Manual investigation and schema updates caused 2-3 hour delays
- Lost data during downtime

**3. Partial Failure Problem:**
- Single malformed record (null in NOT NULL column, data type mismatch) failed entire batch
- No visibility into which record caused failure
- Manual error identification was time-consuming and reactive

---

## My Solution

I redesigned the ingestion layer with three core optimizations:

### 1. Batch Processing with executemany() and write_pandas()

**For smaller batches (<100K rows):**
Replaced row-by-row `cursor.execute()` with `executemany()`:
```python
# BEFORE (Slow - Row by Row)
for record in records:
    cursor.execute(
        "INSERT INTO loans (id, amount, status) VALUES (?, ?, ?)",
        (record['id'], record['amount'], record['status'])
    )
conn.commit()
# Result: 30 minutes for 1M records

# AFTER (Fast - Batch Insert)
values = [(r['id'], r['amount'], r['status']) for r in records]
cursor.executemany(
    "INSERT INTO loans (id, amount, status) VALUES (?, ?, ?)",
    values
)
conn.commit()
# Result: 5 minutes for 1M records (6× improvement)
```

**For larger batches (>100K rows):**
Used Snowflake's `write_pandas()` for optimal performance:
```python
import pandas as pd
from snowflake.connector.pandas_tools import write_pandas

# Convert to DataFrame
df = pd.DataFrame(records)

# Batch write to Snowflake
success, nchunks, nrows, _ = write_pandas(
    conn=conn,
    df=df,
    table_name='LOANS',
    database='PROD_DB',
    schema='RAW',
    chunk_size=100000,  # Process in 100K row chunks
    compression='gzip',
    quote_identifiers=False  # Avoid identifier quoting overhead
)
# Result: 3.5 minutes for 1M records (10× improvement)
```

---

### 2. Schema Drift Handling

Implemented automatic schema reconciliation:
```python
def handle_schema_drift(df, table_name):
    """
    Compare DataFrame columns with Snowflake table schema.
    Add missing columns automatically.
    """
    # Get current table schema
    cursor.execute(f"DESC TABLE {table_name}")
    existing_columns = {row[0].upper() for row in cursor.fetchall()}
    
    # Get DataFrame columns
    df_columns = {col.upper() for col in df.columns}
    
    # Find new columns
    new_columns = df_columns - existing_columns
    
    if new_columns:
        for col in new_columns:
            # Infer data type from DataFrame
            dtype = df[col].dtype
            sql_type = 'VARCHAR' if dtype == 'object' else 'NUMBER'
            
            # Add column to table
            cursor.execute(f"ALTER TABLE {table_name} ADD COLUMN {col} {sql_type}")
            logger.info(f"Added new column {col} to {table_name}")
    
    return df
```

**Result:** Zero downtime from schema changes; automatic adaptation

---

### 3. Resilient Error Handling

Implemented record-level error isolation:
```python
def write_with_error_handling(df, table_name):
    """
    Attempt batch write. On failure, identify and isolate bad records.
    """
    try:
        # Attempt full batch write
        write_pandas(conn, df, table_name)
        return len(df), []
    
    except Exception as e:
        logger.warning(f"Batch write failed: {e}. Attempting row-level validation...")
        
        good_records = []
        bad_records = []
        
        # Validate each record
        for idx, row in df.iterrows():
            try:
                # Test insert single row
                single_df = pd.DataFrame([row])
                write_pandas(conn, single_df, table_name)
                good_records.append(row)
            except Exception as row_error:
                bad_records.append({
                    'index': idx,
                    'data': row.to_dict(),
                    'error': str(row_error)
                })
        
        # Write good records in batch
        if good_records:
            good_df = pd.DataFrame(good_records)
            write_pandas(conn, good_df, table_name)
        
        # Log bad records for investigation
        if bad_records:
            log_failed_records(bad_records, table_name)
        
        return len(good_records), bad_records
```

**Result:** 99% of records loaded successfully; bad records logged for analysis

---

## Results

**Performance:**
- Load time: 30 minutes → 5 minutes (6× improvement)
- Throughput: 28K records/min → 285K records/min
- Network round-trips: 1M → 10 (chunking with 100K rows)
- CPU utilization: 85% → 30%

**Reliability:**
- Zero downtime from schema changes (automatic adaptation)
- Batch failure rate: 100% (all-or-nothing) → 0.2% (bad records isolated)
- Mean time to recovery: 2-3 hours → 0 (automatic handling)

**Business Impact:**
- Met 10-minute ingestion SLA consistently
- Enabled real-time dashboards for business users
- Reduced operational burden—no manual intervention needed
- Pattern adopted for 6 other ingestion pipelines

---

## Interview Q&A

### Q1: "Why was row-by-row insert so slow?"

**Answer:**
"Each `cursor.execute()` in the loop triggered a separate network round-trip to Snowflake:

**Per-record overhead:**
1. Python serializes parameters
2. Sends INSERT over network (HTTP request)
3. Snowflake parses SQL
4. Executes insert
5. Returns acknowledgment
6. Python processes response

For 1M records: 1M round-trips × ~2ms each = 30 minutes in network overhead alone.

**Batch approach:**
- `executemany()`: Single network call with all data
- `write_pandas()`: Uses Snowflake's internal staging (PUT/COPY)
- Network overhead: 10 calls (chunked) × 2ms = 20ms total

This is why we got 10× improvement—eliminated network bottleneck."

---

### Q2: "Why write_pandas() instead of executemany() for large batches?"

**Answer:**
"`write_pandas()` uses Snowflake's optimized bulk loading mechanism:

**Under the hood:**
1. Writes data to internal stage (cloud storage: S3/Azure)
2. Uses COPY INTO command (Snowflake's fastest load method)
3. Parallel processing across multiple compute nodes
4. No SQL parsing overhead

**Performance comparison (1M rows):**
- `cursor.execute()` loop: 30 minutes
- `executemany()`: 5 minutes (6× faster)
- `write_pandas()`: 3.5 minutes (10× faster, 30% better than executemany)

**Trade-off:**
- `write_pandas()` requires pandas dependency and more memory (DataFrame in memory)
- For <50K rows, `executemany()` is simpler and sufficient
- For >100K rows, `write_pandas()` is clearly superior"

---

### Q3: "How did you handle schema drift without breaking existing columns?"

**Answer:**
"Key was to only ADD columns, never modify or delete:

**Safety checks:**
1. Only add new columns (missing in table but present in data)
2. Never remove columns (old data might reference them)
3. Default data type: VARCHAR (safest, can always cast later)
4. Add columns as nullable (existing rows need default)

**Example scenario:**
- API adds new field `partner_reference_id`
- My code detects it's missing in table
- Executes: `ALTER TABLE loans ADD COLUMN partner_reference_id VARCHAR`
- Historical rows have NULL for this column
- New rows populate it
- No disruption to existing queries

**What I didn't handle:** Column renames (would require mapping logic) or type changes (would need data migration). Those required manual intervention."

---

### Q4: "The record-level validation seems expensive. Didn't it slow things down?"

**Answer:**
"Great observation! Record-level validation is expensive, but it's a **fallback mechanism**, not the primary path.

**Happy path (99.8% of batches):**
- Batch write succeeds → No validation → 3.5 minutes
- Fast and efficient

**Error path (0.2% of batches):**
- Batch write fails → Triggers row-by-row validation → ~15 minutes
- Slow but provides visibility into bad records

**Why this design:**
- Most batches are clean—optimized for common case
- When batch fails, we need to identify the problem (better than failing silently)
- 15 minutes once in 500 runs is acceptable vs. 30 minutes every run

**Alternative considered:**
- Pre-validation of all records (expensive every time)
- Rejected: Would slow down happy path

**Better long-term solution:**
- Added data quality checks at API ingestion layer
- Reduced bad record rate from 0.2% to 0.02%"

---

### Q5: "Did you use transactions? What about data consistency?"

**Answer:**
"Yes, transactions were critical:

**With executemany():**
```python
try:
    cursor.executemany(insert_sql, values)
    conn.commit()  # All succeed or none
except Exception as e:
    conn.rollback()  # Atomic rollback
```

**With write_pandas():**
- Uses Snowflake's COPY INTO internally
- COPY INTO is atomic by default
- Either all rows load or none

**Idempotency:**
- Each batch has unique `batch_id` and `ingestion_timestamp`
- If rerun due to failure, we check: `WHERE batch_id NOT EXISTS`
- Prevents duplicates

**At-least-once delivery:**
- API → Snowflake staging → Snowflake table
- If staging succeeds but table load fails, retry loads from staging
- No data loss"

---

### Q6: "How did you test this before production?"

**Answer:**
"Three-phase testing:

**Phase 1: Dev Environment**
- Cloned production table schema to dev
- Generated synthetic data (1M rows) mimicking production patterns
- Tested both happy path and error scenarios
- Verified: Performance, error handling, schema drift handling

**Phase 2: Staging with Real Data**
- Ran new code in shadow mode (parallel to old pipeline)
- Compared results: row counts, checksums, sample validation
- Monitored for 3 days—zero discrepancies

**Phase 3: Production Rollout**
- Blue-green deployment: New pipeline runs alongside old
- Switched 10% of traffic → 50% → 100% over 1 week
- Rollback plan: Switch traffic back to old pipeline
- Monitored: Load times, error rates, data quality

**Post-deployment:**
- Kept old pipeline code for 2 weeks (safety net)
- Decommissioned after validation period"

---

### Q7: "What was the biggest challenge?"

**Answer:**
"Convincing stakeholders to invest time in this refactor.

**The challenge:**
- System was 'working' (albeit slowly)
- Stakeholders prioritized new features over infrastructure
- Concerns: Risk of breaking production, development time

**My approach:**
1. **Quantified pain:** Showed we'd miss SLA within 2 months at current growth
2. **Demonstrated POC:** Built proof-of-concept in 2 days showing 10× improvement
3. **Minimal risk:** Proposed shadow mode deployment (no production impact)
4. **Business value:** Faster loads → real-time dashboards → better decisions

**Result:** Got 2-week sprint allocation. Delivered in 10 days with comprehensive testing."

---

### Q8: "Did you consider other solutions?"

**Answer:**
"Yes, evaluated three alternatives:

**1. Snowpipe (Snowflake's streaming ingestion):**
- Pro: Near real-time, auto-scaling
- Con: Higher cost, requires S3/Azure staging setup, overkill for 10-min batch
- Decision: Future consideration for true real-time needs

**2. Spark for ETL:**
- Pro: Handles massive scale, complex transformations
- Con: Infrastructure overhead (EMR/Databricks), team unfamiliar with Spark
- Decision: Not justified for our scale (1M rows is small for Spark)

**3. DBT for transformations:**
- Pro: SQL-based, version control, testing framework
- Con: Doesn't solve ingestion performance (happens after load)
- Decision: Implemented later for transformations, separate concern

**Why batch optimization won:**
- Smallest change (same infrastructure)
- Lowest risk (incremental improvement)
- Immediate impact (10× faster)
- Taught team: Optimize before adding complexity"

---

### Q9: "How do you monitor this in production?"

**Answer:**
"Four-layer monitoring:

**1. Performance Metrics:**
```python
metrics = {
    'load_time_seconds': time.time() - start_time,
    'records_processed': len(df),
    'records_failed': len(bad_records),
    'throughput_per_min': len(df) / (load_time / 60)
}
# Send to CloudWatch
```
Alert if load_time > 5 minutes or throughput < 200K/min

**2. Data Quality Checks:**
- Row count validation (source vs. destination)
- Checksum validation for critical columns
- Schema drift alerts (new columns detected)

**3. Error Tracking:**
- Bad records logged to S3 for analysis
- Daily summary: error patterns, affected partners
- Alert if error rate > 1%

**4. Business Metrics:**
- Data freshness (last successful load timestamp)
- Dashboard showing: load trends, error trends, SLA compliance
- Accessible to business stakeholders"

---

### Q10: "What would you improve if you had more time?"

**Answer:**
"Three improvements:

**1. Parallel Processing:**
- Current: Sequential batch processing
- Future: Process multiple API sources in parallel (ThreadPoolExecutor)
- Expected: 2× additional speedup

**2. Incremental Loading:**
- Current: Full extract every 10 minutes
- Future: Incremental (only changed records since last load)
- Requires: API versioning and change tracking
- Expected: 50% reduction in data volume

**3. Automated Schema Migration:**
- Current: Add new columns automatically
- Future: Detect type changes, handle renames with mapping logic
- Requires: Schema registry and migration framework

**But honestly:** Current solution meets all requirements. These are optimizations for future scale, not urgent needs."

---

## Technical Deep Dive

### Why write_pandas() is Fast

**Snowflake's Internal Process:**
1. `write_pandas()` serializes DataFrame to Parquet
2. Uploads Parquet to internal stage (PUT command)
3. Executes COPY INTO from stage to table
4. COPY INTO uses:
   - Parallel processing across compute nodes
   - Optimized Parquet reader
   - Direct columnar write (no row-by-row processing)

**Key Parameters:**
```python
write_pandas(
    conn=conn,
    df=df,
    table_name='LOANS',
    chunk_size=100000,      # Process in chunks (memory management)
    compression='gzip',      # Reduce network transfer
    quote_identifiers=False  # Avoid case-sensitivity overhead
)
```

---

