## Cron Expression Answers
---

| # | Requirement | Cron Expression | Explanation |
|---|-------------|-----------------|-------------|
| 1 | Daily at 2 AM | 0 2 * * *| nightly warehouse refresh|
| 2 | Hourly at :15 | 15 * * * * | hourly aggregation|
| 3 | Monday 6 AM | 0 6 * * 1 | weekly report|
| 4 | 1st of month midnight |0 0 1 * * | monthly financial close|
| 5 | Every 15 min |*/15 * * * * |near-real-time dashboard update|
| 6 | Weekdays 8 AM | 0 8 * * 1-5 |business-hours alert check |
---

## Part 2: Pipeline Scheduling Design
---

### Pipeline 1: Nightly Warehouse Refresh

**Scheduling Strategy:** Dependency-aware

**Schedule/Trigger:**
- Upstream dependencies :
    - Successful completion of the PostgreSQL source database backup
    - Optional guardrail: latest-start time (e.g., 2:30 AM) to protect downstream SLAs

**Justification:**
1. Why this strategy? 
    - The warehouse refresh depends **on data consistency guaranteed** by the backup, not a fixed clock time.
    - Backup completion varies around 1 AM, so triggering immediately after it finishes maximizes freshness while still meeting the **6 AM BI dashboard SLA.**
    - Moderate data volume (~500K rows) means runtime is predictable; start time is the critical variable.

2. Why not the alternatives? 
    - **Time-based:** A fixed cron (e.g., 2:00 AM) risks starting before the backup completes or waiting unnecessarily long.
    - **Event-based:** Source-level change events are unnecessary for a full reload pattern and add operational complexity

**Failure handling:**
- What if it fails? 
    - Retry automatically with backoff.
    - f still failing, alert the data/on-call team before the 6 AM SLA window
    - Fall back to the previous day’s warehouse snapshot if needed.
- Is it safe to re-run? 
    - Yes. Use truncate-and-reload or load into staging tables followed by an atomic swap, ensuring idempotent re-execution without duplicates.

---

### Pipeline 2: Hourly Clickstream Aggregation

**Scheduling Strategy:** Dependency-aware

**Schedule/Trigger:**
- Upstream dependencies :
    - Arrival of the hourly clickstream files in the Amazon S3 bucket
    - Files written by Amazon Kinesis Firehose for the previous hour
    - Optional guardrail: latest-start cutoff (e.g., :15 past the hour) in case files are delayed

**Justification:**
1. Why this strategy? 
    - The job must wait for **data completeness**, not a fixed clock. Files arrive between :05–:10, which makes a strict cron fragile.
    - Triggering when all expected files for the hour are present ensures correct aggregates and avoids partial-hour sessions.
    - Volume is high (~2M events/hour), so reprocessing due to early starts is costly.

2. Why not the alternatives? 
    - **Time-based:** A cron at :00 or :05 risks missing late files and producing incomplete aggregates.
    - **Event-based:** Per-file triggers would cause multiple partial runs per hour unless heavily coordinated; better to wait on a clear dependency condition.

**Failure handling:**
- What if it fails? 
    - Retry for the same hour window after a short delay.
    - If retries fail, alert the data/on-call team and allow a manual backfill for the affected hour without blocking later hours.

- Is it safe to re-run? 
    - Yes. Partition outputs by hour and write using overwrite/replace semantics (or staging + atomic commit).
    - Deduplicate by (session_id, hour) to guarantee idempotent reprocessing.

---


### Pipeline 3: Financial Close Pipeline

**Scheduling Strategy:** Dependency-aware

**Schedule/Trigger:**
- Upstream dependencies :
    - Sequential task completion: Extract accounts → Validate balances → Calculate interest → Generate report → Email to CFO
    - Initial trigger: start of month (e.g., 1st), with internal task dependencies enforcing order
    - Deadline guardrail: must finish by the 5th of the month

**Justification:**
1. Why this strategy? 
    - This is a multi-step workflow, where each step is logically and technically dependent on the previous one.
    - Correctness matters more than speed; downstream steps are meaningless if upstream validation fails.
    - The SLA is date-based (by the 5th), not tied to a specific hour, making dependency orchestration the natural fit.

2. Why not the alternatives? 
    - **Time-based:** A single cron cannot safely coordinate multiple interdependent steps or handle partial failures.
    - **Event-based:** There’s no external event stream; the process is internally driven and stateful.

**Failure handling:**
- What if it fails? 
    - Fail fast at the step where the error occurs and block downstream steps.
    - Alert finance/data engineering immediately with context (which step, which accounts)
    - Allow fixes and resume from the failed step without restarting the entire pipeline.
    - Escalate if completion is at risk of missing the 5th-of-month deadline.

- Is it safe to re-run? 
    - Yes, if each step is **checkpointed:**
        - Extracts are versioned by month.
        - Validations and calculations are deterministic.
        - Reports are regenerated with overwrite semantics.
    - Email step should be explicitly controlled (manual approval or “send-once” flag) to avoid duplicate sends.

---


### Pipeline 4: Partner File Ingestion

**Scheduling Strategy:** Event-based

**Schedule/Trigger:**
- Trigger condition :
    - Detection of a **new CSV file** arriving on the partner **SFTP server**
    - Triggered via:
        - SFTP directory polling with change detection, or
        - A partner-sent notification (if available)

**Justification:**
1. Why this strategy? 
    - File arrivals are unpredictable, so time-based scheduling would either introduce unnecessary latency or waste resources polling too frequently.
    - Processing should start as soon as the file exists, which aligns naturally with an event-based trigger.
    - Volume is modest (10K–50K rows), so immediate processing is efficient and keeps business operations data fresh.

2. Why not the alternatives? 
    - **Time-based:** Requires guessing arrival windows and risks delayed ingestion or empty runs.
    - **Dependency-aware:** There is no internal upstream job—only an external file arrival—so a full dependency graph is unnecessary.

**Failure handling:**
- What if it fails? 
    - Quarantine the file and log the failure reason.
    - Alert the business operations/data team with file name and error details.
    - Allow reprocessing once the issue is corrected (e.g., schema fix, data correction).

- Is it safe to re-run? 
    - Yes. Use **file-level idempotency:**
        - Track processed files by checksum or filename + timestamp.
        - Load into staging tables, then merge or replace atomically.
        - Prevent duplicate ingestion if the same file is retried or re-uploaded.

---

### Pipeline 5: ML Feature Pipeline

**Scheduling Strategy:** Dependency-aware

**Schedule/Trigger:**
- upstream dependencies
    - Successful completion of Pipeline 1 (Nightly Warehouse Refresh)
    - Must complete before 7:00 AM, when ML scoring jobs begin

**Justification:**
1. Why this strategy? 
    - Feature generation depends on fresh, fully loaded warehouse data; running before Pipeline 1 completes would produce stale or inconsistent features.
    - The scoring job has a hard start time (7 AM), so coordinating completion order is critical.
    - Dependency-aware scheduling ensures features are built as soon as data is ready, maximizing buffer time before scoring.

2. Why not the alternatives?
    - **Time-based:** A fixed cron (e.g., 5 AM) risks racing the warehouse refresh on slow days.
    - **Event-based:** There’s no external event; the trigger is an internal pipeline dependency, not data arrival at the source level.

**Failure handling:**
- What if it fails? 
    - Retry automatically within the available window before 7 AM.
    - If retries fail, alert ML/data engineering immediately.
    - Optionally fall back to previous day’s feature set to avoid blocking scoring (with reduced model freshness).

- Is it safe to re-run? 
    - Yes. Features should be:
        - Built partitioned by date.
        - Written using overwrite or atomic swap semantics.
        - Derived deterministically from the warehouse snapshot to ensure idempotent re-execution.

---

### Pipeline 6: Data Quality Checks

**Scheduling Strategy:** Dependency-aware

**Schedule/Trigger:**
- upstream dependencies 
   - Successful (or completed) execution of each upstream pipeline:
   - Triggered immediately after each pipeline finishes, with an SLA of ≤ 30 minutes to complete checks

**Justification:**
1. Why this strategy?
    - Data quality checks are post-conditions, not standalone jobs.
    - Running them immediately after each pipeline ensures issues are caught close to the point of data creation, when context and logs are still fresh.
    - Different pipelines run on different cadences (hourly, daily, monthly, event-driven), making a single time-based schedule impractical.

2. Why not the alternatives?
    - **Time-based:** Would either miss runs, run too early, or require many fragile cron schedules.
    - **Event-based:** External data arrival events are irrelevant; the trigger is internal pipeline completion.

**Failure handling:**
- What if it fails? 
    - Fail the data quality task without blocking downstream consumers unless the check is marked critical.
    - Optionally auto-create an incident or ticket for critical failures.

- Is it safe to re-run? 
    - Yes. Data quality checks are read-only:
        - Queries are deterministic and side-effect free.
        - Re-runs simply re-evaluate the same data snapshot.
    - Store results with timestamps to track trends without duplication.

---

## Anti-Pattern Identification
---
### Anti-Pattern : Time-Based Chaining (Clock-Driven Dependencies)

**What's wrong:** 
Pipelines B and C are scheduled based on assumed completion times of upstream pipelines rather than on actual completion. The schedule encodes dependencies in wall-clock time instead of explicitly modeling them.

**Risk:** 
- If Pipeline A runs late or fails, Pipeline B may start with incomplete or missing data.
- Delays accumulate silently, producing partial loads or corrupt downstream outputs.
- Failures can go unnoticed until consumers see bad data.
- Manual re-runs are painful because cron schedules don’t encode execution order.

**Fix:** 
- Make the workflow **dependency-aware:**
    - Trigger Pipeline B on successful completion of Pipeline A.
    - Trigger Pipeline C on successful completion of Pipeline B.
- Optionally keep a time-based kickoff only for Pipeline A (e.g., 1:00 AM), with downstream pipelines driven by dependencies.
- Use an orchestrator (e.g., Airflow, Dagster, Prefect) or explicit success markers to enforce execution order.

---

### Anti-Pattern : Over-Polling / Wasted Cron

**What's wrong:** 
The pipeline is scheduled to check every 5 minutes, even though the partner uploads files only once per week. This results in many unnecessary runs that mostly do nothing.

**Risk:** 
- Wastes compute resources and increases operational cost.
- Generates unnecessary logs/alerts, making it harder to detect real issues.
- Can create unnecessary load on the SFTP server or downstream systems.

**Fix:** 
- Switch to an **event-based approach:**
   -  Trigger the pipeline only when a new file arrives (via partner notification or SFTP directory watch).
   - If event notifications are not possible, reduce polling frequency to match expected file arrival (e.g., once per day) and combine with file existence check.
- Implement idempotency so reprocessing the same file is safe.
---

### Anti-Pattern : Time-Based Trigger Ignoring Upstream Dependency

**What's wrong:** 
The warehouse refresh is scheduled at **midnight**, but the **source database backup is still running (11 PM – 1 AM).** This ignores the actual dependency on backup completion.

**Risk:** 
- Refresh may start before backup finishes, causing incomplete or inconsistent data.
- Downstream consumers (BI dashboards) may receive wrong or partial data, breaking SLAs.
- Manual intervention may be required to fix or re-run the job.

**Fix:** 
- Make the pipeline **dependency-aware:**
    - Trigger the refresh **only after the backup completes successfully.**
    - Optionally add a **latest-start guardrail (e.g., 1:30 AM)** to ensure BI SLA is met even if backup is delayed.
- Use a workflow orchestrator or a **completion signal from the backup process** rather than relying on a fixed cron.

---
## Pipeline Dependency DAG
---
```
[Pipeline 1: Warehouse Refresh]
        |
        ├──→ [Pipeline 5: ML Features]
        |           |
        |           └──→ [ML Scoring Job (external)]
        |
        └──→ [Pipeline 6: Data Quality Checks]

[Pipeline 2: Clickstream Aggregation]
        |
        └──→ [Pipeline 6: Data Quality Checks]

[Pipeline 3: Financial Close]
    (Step 1) → (Step 2) → (Step 3) → (Step 4) → (Step 5)
        |
        └──→ [Pipeline 6: Data Quality Checks]

[Pipeline 4: Partner Ingestion]
        |
        └──→ [Pipeline 6: Data Quality Checks]
```
---



