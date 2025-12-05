# STAR Story: Cross-Team Collaboration - API Breaking Change

## The STAR Story

### Situation
> "At my previous company, we provided micro-loans to gig workers — delivery partners, drivers, etc. A critical part of our loan approval pipeline was fetching credit history from a third-party credit bureau API. One morning, our daily loan eligibility pipeline started failing. The on-call alert fired around 6 AM, and I picked it up.

> The pipeline was responsible for pulling credit scores and repayment history for loan applicants, transforming the data, and loading it into Snowflake for our risk models. Without this data, our loan disbursement for the day was blocked — we were looking at ~500 loan applications stuck in queue."

**Key Details to Remember:**
- Credit bureau = third-party vendor (think CIBIL, Experian equivalent)
- ~500 applications/day was realistic volume
- Pipeline ran daily at 5 AM via Airflow

---

### Task
> "My immediate task was to restore the pipeline so loan disbursements could resume. But I also realized this wasn't just a one-time fix — we had no visibility into upstream API changes, and this could happen again. So I took ownership of both the immediate fix and proposing a longer-term solution to prevent recurrence."

---

### Action

> "First, I investigated the root cause. The Airflow logs showed a `KeyError` in our parsing logic. I pulled a sample API response and compared it with our existing schema — turned out the credit bureau had renamed two fields (`credit_score` → `cibil_score`, `outstanding_amount` → `total_outstanding`) and added a new required field (`bureau_version`).
>
> **Immediate Fix (Same Day):**
> 1. I updated the Python transformation script to handle both old and new field names using defensive parsing — essentially checking for both keys and mapping to our internal schema.
> 2. Added a fallback for the new required field with a sensible default, and flagged those records for manual review.
> 3. Tested with sample payloads, deployed to staging, validated, and pushed to production.
> 4. Reran the failed DAG, and loan processing resumed by evening.
>
> **Preventive Measures (Next 2 Weeks):**
> 1. I reached out to the credit bureau's integration team to understand their release process — learned they push updates quarterly but don't notify all clients.
> 2. I added **schema validation** at the ingestion layer using `jsonschema` in Python — if any unexpected field changes occur, the pipeline fails fast with a clear error instead of silently breaking downstream.
> 3. Set up a **Slack alert** that triggers when schema validation fails, with a diff of expected vs received schema.
> 4. Documented the API contract in our internal wiki and added it to our quarterly vendor review checklist.
> 5. Shared the incident in our engineering sync and proposed that all vendor API integrations add similar schema validation — this was adopted for 2 other pipelines (payment gateway, KYC provider)."

**Technical Depth:**

```python
# Defensive parsing example (be ready to explain)
def parse_credit_response(response: dict) -> dict:
    """Handle both old and new API schema"""
    return {
        "credit_score": response.get("cibil_score") or response.get("credit_score"),
        "outstanding": response.get("total_outstanding") or response.get("outstanding_amount"),
        "bureau_version": response.get("bureau_version", "v1_legacy"),  # default for old responses
        "fetch_timestamp": datetime.utcnow().isoformat()
    }
```

```python
# Schema validation example
from jsonschema import validate, ValidationError

EXPECTED_SCHEMA = {
    "type": "object",
    "required": ["cibil_score", "total_outstanding", "bureau_version"],
    "properties": {
        "cibil_score": {"type": "integer"},
        "total_outstanding": {"type": "number"},
        "bureau_version": {"type": "string"}
    }
}

def validate_api_response(response: dict):
    try:
        validate(instance=response, schema=EXPECTED_SCHEMA)
    except ValidationError as e:
        send_slack_alert(f"Schema mismatch: {e.message}")
        raise
```

---

### Result

> "The pipeline was restored within the same day, and we cleared the backlog of 500+ loan applications by that evening. More importantly, the schema validation approach I introduced became a standard practice — we implemented it across 3 critical vendor integrations (credit bureau, payment gateway, KYC provider).
>
> Over the next 6 months, the schema validation caught 2 more unannounced API changes before they caused production failures.
>
> The incident also led to us adding vendor API reviews to our quarterly checklist, which improved our relationship with the credit bureau team — they now notify us before major releases."

**Quantified Impact:**
| Metric | Value |
|--------|-------|
| Time to restore | ~12 hours (same day) |
| Applications unblocked | 500+ |
| Pipelines protected | 3 |
| Future incidents prevented | 2 (caught by schema validation) |

---

## Follow-Up Questions & Answers

### Q1: "How did you identify that it was a schema change and not something else?"

> "My first instinct was to check the Airflow logs. I saw a `KeyError: 'credit_score'` in the transformation task. That immediately told me the key was missing from the response.
>
> I then pulled a raw API response from our staging environment and compared it with what our code expected. The diff was clear — two fields were renamed, one new field was added.
>
> I also checked if the issue was on our side — verified our API credentials hadn't expired, checked if there was a network timeout, and confirmed the API endpoint was reachable. Once I ruled those out, the schema change was the only explanation."

---

### Q2: "Why defensive parsing instead of just updating to the new schema?"

> "Two reasons:
>
> First, **backward compatibility**. We had historical data in Snowflake with the old field names. If I changed everything to the new schema immediately, I'd need to backfill or transform existing data. Defensive parsing let me normalize both formats to our internal schema without touching historical data.
>
> Second, **safety during transition**. I wasn't 100% sure if the API change was rolled out to all environments or if it was gradual. Defensive parsing ensured we could handle mixed responses during any rollout period.
>
> Once we confirmed the old schema was fully deprecated (after 2 weeks), I cleaned up the defensive code and moved to the new schema exclusively."

---

### Q3: "How did you test your fix before deploying to production?"

> "I followed a staged approach:
>
> 1. **Unit tests**: Wrote test cases with both old and new API response formats, verified the parsing logic handled both.
>
> 2. **Staging environment**: We had a staging Airflow instance. I deployed there first and ran the DAG with a sample of real (anonymized) applicant IDs.
>
> 3. **Dry run in production**: Before full execution, I ran the pipeline on a small batch (~50 records) with a flag that parsed but didn't write to the final table — wrote to a temp table instead. Validated the output manually.
>
> 4. **Full production run**: Once validated, I triggered the full backfill for the day's applications.
>
> The whole testing cycle took about 3-4 hours, which is why restoration took the full day."

---

### Q4: "What was the business impact of the delay?"

> "The direct impact was that ~500 loan applications were stuck for about 12 hours. For our users — gig workers — this mattered because many of them rely on quick disbursement for daily expenses.
>
> However, we communicated proactively. I informed our ops team within the first hour, and they sent notifications to applicants about the delay. We also prioritized the backlog once restored, so no application was delayed beyond that day.
>
> There was no financial loss since we didn't reject any valid applications — just delayed processing. But it reinforced why we needed schema validation to prevent future delays."

---

### Q5: "You mentioned you reached out to the vendor team. How did that conversation go?"

> "I sent an email to our integration contact at the credit bureau, explaining the incident and asking about their release process.
>
> They were apologetic — apparently, the change was in their release notes, but it was buried in a PDF that went to a generic email alias on our side that nobody monitored.
>
> From that conversation, I learned:
> - They release quarterly
> - They have a sandbox environment we weren't using
> - They can add us to a direct notification list
>
> I made sure we were added to that list and started using their sandbox to test against upcoming releases before they hit production."

---

### Q6: "How did you convince the team to adopt schema validation for other pipelines?"

> "I didn't push hard — I let the incident speak for itself.
>
> In our next engineering sync, I shared a quick post-mortem: what broke, how I fixed it, and the schema validation approach. I showed the code — it was simple, maybe 20 lines.
>
> Two other engineers who owned the payment gateway and KYC integrations immediately saw the value. They'd had similar scares before. One of them said, 'We should have done this ages ago.'
>
> I offered to pair with them to implement it, and within two weeks, both pipelines had schema validation. It wasn't a top-down mandate — it was peer-driven adoption because the value was obvious."

---

### Q7: "What would you do differently if this happened again?"

> "Honestly, the response was solid. But if I could improve:
>
> 1. **Proactive monitoring**: Instead of waiting for pipeline failure, I'd set up a lightweight daily job that just fetches one sample response and validates schema — a canary check. This would catch changes before the main pipeline runs.
>
> 2. **Contract testing with vendor**: Some mature organizations do contract testing where both parties agree on a schema and any change requires a version bump. We didn't have that leverage with this vendor, but for future integrations, I'd push for it during procurement.
>
> 3. **Faster rollback**: We didn't have a quick rollback mechanism. If the fix had failed, I'd have been stuck. I'd now advocate for feature flags or versioned transformation logic that can be toggled without deployment."

---

### Q8: "How did you prioritize the immediate fix vs the long-term solution?"

> "Clear priority: **restore business first**.
>
> The loan disbursement was blocked — that's revenue and user trust. I timebox the investigation to 2 hours. If I couldn't figure it out in 2 hours, I'd have escalated.
>
> Luckily, the root cause was clear within 30 minutes. The fix took another 4-5 hours including testing.
>
> The long-term solution (schema validation, vendor outreach) happened over the next 2 weeks — after the fire was out. I carved out time in my sprint for it and framed it as tech debt reduction to my manager.
>
> The key is not letting the urgent crowd out the important — but also not letting the important delay the urgent."

---

### Q9: "What tools or frameworks did you use for schema validation?"

> "I used Python's `jsonschema` library — it's lightweight, well-documented, and sufficient for our needs.
>
> For more complex scenarios, I'd consider:
> - **Great Expectations**: For data quality validation at scale
> - **Pydantic**: If we were doing this in a FastAPI context
> - **dbt tests**: If the validation was post-load in the warehouse
>
> But for API response validation at ingestion, `jsonschema` was the right tool — minimal overhead, easy to maintain, and team was already familiar with JSON schema syntax."

---

### Q10: "How do you handle vendor dependencies in general now?"

> "After this incident, I follow a few principles:
>
> 1. **Never trust external data**: Always validate schema and data types at ingestion.
>
> 2. **Isolate vendor logic**: Keep vendor-specific parsing in separate modules. If the vendor changes, I only touch one file.
>
> 3. **Monitor aggressively**: Schema validation, response time monitoring, and alerting on anomalies.
>
> 4. **Build relationships**: Know who to call when things break. A 5-minute call can save hours of debugging.
>
> 5. **Document contracts**: Even if informal, write down what you expect from the API. It helps during incidents and onboarding new team members."

---
