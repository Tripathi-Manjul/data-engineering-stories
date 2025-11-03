# Snowflake Cost Optimization - $1K Monthly Savings

## Main Story

"At Karmalife, I reduced Snowflake warehouse costs by about $600–$800 per month (25–30% of compute spend).

**The Problem:**
Legacy queries migrated from PostgreSQL still used SELECT *, and our BI tool auto-generated queries scanning entire tables. On our 50M-row loan transactions table, these ran 100+ times daily, each costing about $0.50 in compute credits.

**My Solution:**

1. **Added Clustering:** Implemented clustering on (client_id, transaction_date). Clustering in Snowflake is like creating a smart index—it physically co-locates related data within micro-partitions based on specified columns. When queries filter on clustered columns, Snowflake uses metadata to prune (skip) irrelevant micro-partitions, scanning only necessary data.

2. **Query Optimization:** Rewrote top 10 expensive queries to select only needed columns instead of SELECT *, reducing data scanned by ~70%.

3. **Result Caching:** Enabled RESULT_SCAN for repetitive aggregations, avoiding redundant computation.

**Results:**
- Typical query cost: $3 → $1 (≈65% reduction)
- Query performance: 45 seconds → 4 seconds (10x faster)
- Monthly savings: ~$700 on warehouse compute
- Pattern applied to 3 other pipelines for additional savings"

---

## Interview Q&A

### Q1: "What exactly is clustering and how does it work?"

**Answer:**
"Clustering in Snowflake is a table property that organizes data within micro-partitions based on specified columns. 

**How it works:**
- Snowflake stores data in immutable 50-500MB micro-partitions
- Each micro-partition has metadata: min/max values for every column
- When you cluster on (client_id, transaction_date), Snowflake co-locates rows with same client_id and similar dates together
- At query time, if you filter WHERE client_id = 123, Snowflake checks metadata and skips micro-partitions that don't contain client 123
- This is called partition pruning—reduces data scanned by 90-95%

**Key difference from indexes:**
- Traditional indexes: separate lookup structure
- Clustering: physical data organization
- Automatic maintenance in background"

---

### Q2: "How did you choose (client_id, transaction_date) as the clustering key?"

**Answer:**
"I analyzed 30 days of query history from QUERY_HISTORY view and found:
- 90% of queries filtered by client_id first
- 85% then filtered by date ranges
- Typical pattern: WHERE client_id = X AND transaction_date >= Y

I tested three options:
1. **Just client_id:** Improved pruning but still scanned months of data per client
2. **Just transaction_date:** Poor pruning when queries filtered by client
3. **(client_id, transaction_date):** Best pruning—reduced partitions scanned from ~1000 to ~50

Verified with SYSTEM$CLUSTERING_INFORMATION() that clustering depth was optimal (2-3 average depth means well-clustered)."

---

### Q3: "Why was SELECT * such a big problem in Snowflake?"

**Answer:**
"Snowflake uses columnar storage—data stored by column, not row.

**The problem:**
- Our loan_transactions table had 40+ columns including large JSON metadata blobs
- Most queries only needed 5-6 columns (client_id, amount, date, status)
- SELECT * forced Snowflake to:
  - Scan 40 columns across micro-partitions
  - Decompress unnecessary data
  - Transfer over network to compute

**Impact:**
- Query scanning 50M rows × 40 columns vs 50M rows × 6 columns
- Data scanned: ~12GB vs ~2GB (6x difference)
- Compute cost proportional to data scanned

**Why it existed:**
- Queries migrated from PostgreSQL row-based storage where SELECT * less costly
- BI tool (Looker) auto-generated queries defaulted to SELECT *
- No one did optimization audit after migration"

---

### Q4: "How did you measure the $1K savings?"

**Answer:**
"I used Snowflake's built-in cost tracking:

**Before optimization (30 days):**
- Exported QUERY_HISTORY for our main warehouse
- Each query ~$0.50 in credits before optimization
- After optimization: ~$0.15 per query
- 120 queries/day × $0.35 savings = ~$40/day
- Monthly: $40 × 30 = ~$1,200/month

🎯 **Why this works:** Shows you actually measured it, and minor correction shows authenticity, not rehearsed.

---

### Q5: "What are the downsides of clustering?"

**Answer:**
"Yes, three main downsides:

**1. Maintenance Cost:**
- Snowflake reclusters in background when data changes
- Reclustering consumes credits (costs money)
- For us: ~$50-100/month in reclustering vs $1K saved (good ROI)

**2. Write Performance:**
- Clustering optimizes reads but can slow writes slightly
- As new data arrives, Snowflake must place it in correct micro-partitions
- Not an issue for our batch load pattern (once daily)

**3. Not Always Beneficial:**
- Helps read-heavy workloads with filtering patterns
- Doesn't help if queries scan entire table (aggregations without WHERE)
- Doesn't help if access pattern is completely random
- Our workload was perfect fit: 90% queries filtered by client and date"

---

### Q6: "How did you get buy-in from leadership?"

**Answer:**
"I created a one-page proposal:

**Current State:**
- Monthly Snowflake cost: ~$3K
- Top 10 queries consuming 30% of compute
- Queries slow (45 seconds average)

**Proposed Solution:**
- Add clustering (2 days implementation)
- Optimize queries (3 days)
- Enable result caching (1 day)
- Total effort: 1 week

**Expected Impact:**
- Cost reduction: ~$1K/month ($12K annually)
- Performance improvement: 10x faster queries
- Risk: Zero—clustering transparent to existing queries
- Rollback: Can drop clustering if issues

**ROI:** $12K annual savings for 1 week of work = immediate approval."

---

### Q7: "Did you consider other optimization approaches?"

**Answer:**
"Yes, I evaluated four approaches:

**1. Clustering (Chose this):**
- Pro: Automatic partition pruning, helps all queries
- Con: Maintenance cost
- Decision: Best ROI for our access pattern

**2. Materialized Views:**
- Pro: Pre-computed results, zero query-time cost
- Con: Storage cost, staleness issues, maintenance
- Decision: Saved for future if needed

**3. Search Optimization Service:**
- Pro: Snowflake feature for point lookups
- Con: High cost ($$ per table), overkill for our case
- Decision: Not cost-effective

**4. Separate Smaller Tables:**
- Pro: Naturally smaller scans
- Con: Query complexity, join overhead, data duplication
- Decision: Too disruptive to existing queries

Clustering gave us 80% of benefit for 20% of effort—clear winner."

---

### Q8: "How do you monitor ongoing savings?"

**Answer:**
"I set up three monitoring mechanisms:

**1. Weekly Cost Dashboard:**
- Query ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
- Track credits consumed per warehouse per day
- Alert if weekly cost exceeds baseline + 20%

**2. Query Performance Tracking:**
- Average execution time for top 10 queries
- Partitions scanned vs partitions total ratio
- Alert if ratio degrades (indicates poor clustering)

**3. Clustering Health:**
- Monthly check with SYSTEM$CLUSTERING_INFORMATION()
- Average clustering depth should stay < 5
- Reclustering automatically maintains this

**Outcome:** Savings sustained over 6+ months, no degradation."

---

### Q9: "What would you do differently if you did this again?"

**Answer:**
"Three improvements:

**1. Automate Analysis:**
- Build script to automatically identify expensive queries
- Run weekly instead of one-time manual analysis
- Would have caught issues faster

**2. Testing Environment:**
- Clone production table to dev for testing clustering strategies
- We tested on production (low risk but not ideal)
- Would reduce risk further

**3. Documentation:**
- I documented after the fact
- Should have documented while building
- Would help team understand decision-making process better

**But overall:** Approach was solid—systematic analysis, measured results, sustained impact."

---

### Q10: "Can you show me the actual queries?"

**Answer:**
"I can't share exact queries due to confidentiality, but here's the pattern:

**Before:**
```sql
SELECT * 
FROM loan_transactions
WHERE status = 'approved'
  AND created_date >= '2024-01-01'
GROUP BY client_id;

-- Scanned: 50M rows × 40 columns = ~12GB
-- Partitions: 1000/1000 (no pruning)
-- Time: 45 seconds
-- Cost: ~$0.50
```

**After:**
```sql
SELECT 
    client_id,
    COUNT(*) as loan_count,
    SUM(loan_amount) as total_amount,
    AVG(interest_rate) as avg_rate
FROM loan_transactions
WHERE client_id IN (SELECT id FROM active_clients WHERE region = 'India')
  AND transaction_date >= CURRENT_DATE - 90
  AND status = 'approved'
GROUP BY client_id;

-- Scanned: 2M rows × 4 columns = ~400MB (clustering pruned 48M rows)
-- Partitions: 45/1000 (95% pruning)
-- Time: 4 seconds
-- Cost: ~$0.15
```

**Key changes:**
1. Specific columns instead of SELECT *
2. Added client_id filter for clustering pruning
3. Date range narrowed to 90 days (business requirement)
4. Clustering on (client_id, transaction_date) enabled partition pruning"

---

## Technical Deep Dive

### Snowflake Micro-Partitions
- Immutable, compressed, columnar
- 50-500MB uncompressed (typically 16MB compressed)
- Stored in cloud storage (S3/Azure Blob/GCS)
- Metadata: min/max/distinct/null count for ALL columns

### Clustering Metadata
- Clustering depth: average overlap of micro-partitions
- Lower = better (1 = perfect, 5+ = poor)
- Automatic reclustering maintains depth

### Cost Model
- Compute: Charged per second of warehouse runtime
- Credits consumed based on warehouse size
- Our X-Small warehouse: ~4 credits/hour
- Query cost = (data scanned + computation) × time

---
