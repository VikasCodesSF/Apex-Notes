# Scheduled Apex — Complete Notes
### CRON Expressions · Implementation · Limits · Interview Q&A

---

## 1. What is Scheduled Apex?

Scheduled Apex allows you to execute Apex code automatically at a specified time or at recurring intervals — daily, weekly, hourly, monthly, or any custom schedule. It implements the `System.Schedulable` interface.

### When to Use Scheduled Apex

- Run data cleanup or archiving jobs every night
- Trigger a weekly report-generation batch
- Sync Salesforce data with an external system at regular intervals
- Send automated email alerts on a schedule
- Kick off a nightly Batch Apex job to process large data volumes

### Key Facts

- Implements `System.Schedulable` interface
- Must provide `execute(SchedulableContext ctx)` — the only required method
- Class should be declared `global` (executes outside the org's standard transaction)
- Can be scheduled via Apex code (`System.schedule`) or via the Setup UI
- `CronTrigger` object tracks each scheduled job instance
- `CronJobDetail` is the parent object — tracks the class-level scheduling info
- Max **100** active scheduled jobs per org (all types)

> ⚠️ You **CANNOT** modify a Scheduled Apex class if there are active jobs running for it. Abort/delete those jobs first, save the class, then reschedule.

---

## 2. Implementation — Step by Step

### Step 1 — Create the Schedulable Class

Create a `global` class that implements `System.Schedulable`:

```apex
global class MyScheduledJob implements System.Schedulable {

    global void execute(SchedulableContext SC) {
        // Place your logic here
        // Typically: invoke a Batch, Queueable, or regular Apex class
    }
}
```

---

### Step 2 — Implement the `execute()` Method

The `execute()` method receives a `SchedulableContext` which gives you access to the job ID via `SC.getTriggerId()`. Inside `execute()` you can: invoke a batch job, enqueue a Queueable, run SOQL/DML, or call any Apex method.

```apex
global class CalculateAccountRevenueSchedule implements System.Schedulable {

    global void execute(System.SchedulableContext sContext) {
        // Trigger a batch job
        CalculateTotalAccountRevenue revenueBatch = new CalculateTotalAccountRevenue();
        Database.executeBatch(revenueBatch, 5);

        // Can also call a normal Apex class method
        DatabaseHelper.ExportAllAccounts();
    }
}
```

---

### Step 3 — Schedule the Job

#### ▶ Option A: Programmatically *(Anonymous Window / Apex)*

```apex
// System.schedule(uniqueJobName, cronExpression, schedulableInstance)
String cron = '0 0 2 * * ?';  // Every day at 2:00 AM
System.schedule('Nightly Revenue Job', cron, new CalculateAccountRevenueSchedule());

// Or inline:
System.schedule('Contact Batch Daily', '0 0 1 * * ?', new ContactsBatchSchedule());
```

> 📌 Jobs scheduled via Apex code (`System.schedule`) do **NOT** show a `'Manage'` option in the Scheduled Jobs UI. Only jobs scheduled via the Setup UI get a Manage link.

#### ▶ Option B: Via Setup UI *(Admin Path)*

1. Go to **Setup** → search `'Apex Classes'` in QuickFind
2. Click the **'Schedule Apex'** button
3. Enter a unique **Schedule Job Name**
4. Select the Schedule Class (`global` Schedulable class) via Lookup
5. Choose Frequency: Daily / Weekly / Monthly
6. Set Preferred Start Time (e.g. 2:00 AM)
7. Set Start Date and End Date
8. Click **Save**
9. To monitor: **Setup** → search `'Scheduled Jobs'` → view all active schedules

---

### Step 4 — Abort / Delete a Scheduled Job

#### ▶ Delete *(UI)*

Go to **Setup → Scheduled Jobs** → click **Del** next to the job. This simply removes the job entry.

#### ▶ Abort *(Programmatic)*

```apex
// Step 1: Find the CronTrigger ID
CronTrigger ct = [SELECT Id, CronJobDetail.Name
                  FROM CronTrigger
                  WHERE CronJobDetail.Name = 'Nightly Revenue Job'
                  LIMIT 1];

// Step 2: Abort it
System.abortJob(ct.Id);

// Query all CronTrigger records
SELECT Id, CronJobDetail.Id, CronJobDetail.Name, CronJobDetail.JobType
FROM CronTrigger;
```

---

## 3. CRON Expression — Complete Guide

CRON expressions define when a scheduled job runs. Salesforce uses a **7-field CRON format** (slightly different from Unix CRON which uses 5 fields).

### 3.1 CRON Field Reference

| Position | Field | Allowed Values | Special Chars |
|:--------:|-------|----------------|---------------|
| **1** | Seconds | 0–59 | `*` (any) |
| **2** | Minutes | 0–59 | `*` (any) |
| **3** | Hours | 0–23 | `*` (any) |
| **4** | Day of Month | 1–31 | `* / ? / L / W` |
| **5** | Month | 1–12 or JAN–DEC | `*` (any) |
| **6** | Day of Week | 1–7 or SUN–SAT | `* / ? / L / #` |
| **7** | Year (optional) | 1970–2099 | `*` (any) |

> ℹ️ **CRITICAL RULE:** `Day_of_Month` and `Day_of_Week` **CANNOT** both have values. Whichever one you are NOT using must be set to `'?'` (question mark). Setting both causes an error.

---

### 3.2 Special Characters

- **`*` (asterisk)** — every value. Example: `*` in Hours = every hour
- **`?` (question mark)** — no specific value. Used to exclude one of `Day_of_Month` or `Day_of_Week`
- **`-` (hyphen)** — range. Example: `MON-FRI` = Monday through Friday
- **`/` (slash)** — increment. Example: `0/15` in minutes = 0, 15, 30, 45
- **`L`** — last. Example: `L` in `Day_of_Month` = last day of the month
- **`W`** — nearest weekday. Example: `15W` = nearest weekday to the 15th
- **`#`** — Nth weekday. Example: `2#1` in `Day_of_Week` = 1st Monday of the month

---

### 3.3 Common CRON Examples

| CRON Expression | Meaning |
|----------------|---------|
| `0 0 2 * * ?` | Every day at 2:00 AM |
| `0 0 8 ? * MON` | Every Monday at 8:00 AM |
| `0 0 * * * ?` | Every hour (at minute 0) |
| `0 0 0 1 * ?` | First day of every month at midnight |
| `0 0 0 L * ?` | Last day of every month at midnight |
| `0 0 9 ? * MON-FRI` | Every weekday at 9:00 AM |
| `0 30 12 * * ?` | Every day at 12:30 PM |
| `0 0 23 31 12 ?` | Last day of year at 11 PM (31st Dec) |
| `0 0 0 ? * 1` | Every Sunday at midnight |

---

## 4. Sub-Hourly Scheduling *(Every 5 / 15 / 30 Minutes)*

Salesforce does **not** support a single CRON job running more frequently than once per hour. To schedule a job every N minutes, you must create multiple `System.schedule()` calls with unique names — one for each minute offset.

### 4.1 Every 5 Minutes *(12 jobs)*

```apex
// Salesforce requires 12 separate scheduled jobs for every-5-minute intervals
System.schedule('YourScheduler 1',  '0 00 * * * ?', new YourScheduler());
System.schedule('YourScheduler 2',  '0 05 * * * ?', new YourScheduler());
System.schedule('YourScheduler 3',  '0 10 * * * ?', new YourScheduler());
System.schedule('YourScheduler 4',  '0 15 * * * ?', new YourScheduler());
System.schedule('YourScheduler 5',  '0 20 * * * ?', new YourScheduler());
System.schedule('YourScheduler 6',  '0 25 * * * ?', new YourScheduler());
System.schedule('YourScheduler 7',  '0 30 * * * ?', new YourScheduler());
System.schedule('YourScheduler 8',  '0 35 * * * ?', new YourScheduler());
System.schedule('YourScheduler 9',  '0 40 * * * ?', new YourScheduler());
System.schedule('YourScheduler 10', '0 45 * * * ?', new YourScheduler());
System.schedule('YourScheduler 11', '0 50 * * * ?', new YourScheduler());
System.schedule('YourScheduler 12', '0 55 * * * ?', new YourScheduler());

// Uses 12 out of 100 scheduled job slots
```

### 4.2 Every 15 Minutes *(4 jobs)*

```apex
SchedulableClass obj = new SchedulableClass();
System.schedule('Every 0 Minute',  '0 0  * * * ?', obj);
System.schedule('Every 15 Minute', '0 15 * * * ?', obj);
System.schedule('Every 30 Minute', '0 30 * * * ?', obj);
System.schedule('Every 45 Minute', '0 45 * * * ?', obj);

// Uses 4 out of 100 scheduled job slots
```

### 4.3 Every 30 Minutes *(2 jobs)*

```apex
SchedulableClass obj = new SchedulableClass();
System.schedule('Every 0 Minute',  '0 0  * * * ?', obj);
System.schedule('Every 30 Minute', '0 30 * * * ?', obj);

// Uses 2 out of 100 scheduled job slots
```

> ⚠️ Each of these scheduled jobs counts toward the **100-job org limit**. Scheduling every 5 minutes uses 12 slots. Plan carefully to avoid hitting the limit.

---

## 5. Scheduled Apex — Limits & Constraints

| Limit / Feature | Value / Detail |
|----------------|----------------|
| **Max total active scheduled jobs per org** | 100 (all types combined) |
| **Max Apex scheduled jobs (`JobType = '7'`)** | 100 |
| **Same job name — reuse allowed?** | No — name must be unique across active jobs |
| **Same class — multiple schedules?** | Yes — with DIFFERENT unique job names |
| **`Database.AllowsCallouts` on Schedulable?** | Not directly — delegate to Batch or Queueable |
| **`Database.Stateful` on Schedulable?** | Supported but limited (`execute()` runs once) |
| **Programmatic scheduling method** | `System.schedule(name, cron, instance)` |
| **Abort programmatically** | `System.abortJob(cronTriggerId)` |
| **Min CRON interval (single job)** | 1 hour (Salesforce enforces hourly minimum) |
| **Sub-hourly workaround** | Multiple `System.schedule()` calls with unique names |
| **Query job count** | `SELECT COUNT() FROM CronJobDetail WHERE JobType = '7'` |

---

## 6. Scheduled Apex — Code Patterns

### 6.1 Schedule + Batch *(Most Common Pattern)*

```apex
public class ContactsBatchSchedule implements System.Schedulable {

    public void execute(SchedulableContext SC) {
        ContactsBatchClass batchClass = new ContactsBatchClass();
        Database.executeBatch(batchClass);  // Default batch size 200
    }
}

// Schedule it:
String cron = '0 0 2 * * ?';
System.schedule('Nightly Contact Batch', cron, new ContactsBatchSchedule());
```

---

### 6.2 Schedule + Batch with Custom Size + Callouts

```apex
public class ContactsBatchClassSchedulable
    implements System.Schedulable, Database.AllowsCallouts {

    public void execute(System.SchedulableContext sc) {
        ContactsBatchClass batch = new ContactsBatchClass();
        Id jobId = Database.executeBatch(batch, 200);

        // Note: AllowsCallouts here is for informational purposes.
        // The actual callout must happen in the Batch execute() method.
    }
}
```

> 📌 `Database.AllowsCallouts` on a Schedulable class allows it to call methods that make callouts — but the actual HTTP callout must be in the Batch or Queueable class, not the Schedulable itself.

---

### 6.3 Schedule + Queueable

```apex
global class QueueableScheduler implements System.Schedulable {

    global void execute(SchedulableContext SC) {
        Set<Id> accountIds = new Set<Id>();

        // Build the ID set from a query
        for (Account a : [SELECT Id FROM Account WHERE Phone = null]) {
            accountIds.add(a.Id);
        }

        System.enqueueJob(new AccountQueueable(accountIds));
    }
}

System.schedule('Weekly Queue Job', '0 0 1 ? * MON', new QueueableScheduler());
```

---

### 6.4 Check Current Job Count

```apex
// Count active Scheduled Apex jobs (JobType '7' = Scheduled Apex)
Integer scheduledJobCount = [SELECT COUNT() FROM CronJobDetail WHERE JobType = '7'];
System.debug('Active Scheduled Jobs: ' + scheduledJobCount);

// List all active jobs with names:
List<CronTrigger> jobs = [SELECT Id, CronJobDetail.Name, CronExpression,
                                  NextFireTime, State
                           FROM CronTrigger
                           WHERE CronJobDetail.JobType = '7'];
```

---

## 7. `CronTrigger` vs `CronJobDetail` — Deep Dive

### CronJobDetail

- **Parent object** — represents the Scheduled Apex class configuration
- Stores: `Name` (job name), `JobType` (`'7'` for Scheduled Apex)
- One `CronJobDetail` record per unique scheduled class configuration
- Accessible via: `[SELECT Id, Name FROM CronJobDetail WHERE JobType = '7']`

### CronTrigger

- **Child object** — represents a single specific scheduled job instance
- Stores: `CronExpression`, `NextFireTime`, `PreviousFireTime`, `State`, `TimesTriggered`
- Multiple `CronTrigger` records can point to the same `CronJobDetail`
- Has a lookup field `CronJobDetail` that links to `CronJobDetail`
- **State values:** `WAITING`, `ACQUIRED`, `EXECUTING`, `COMPLETE`, `ERROR`, `DELETED`, `PAUSED`

```apex
// Full query combining both objects:
SELECT Id, CronJobDetail.Id, CronJobDetail.Name, CronJobDetail.JobType,
       CronExpression, NextFireTime, PreviousFireTime, State, TimesTriggered
FROM CronTrigger
WHERE CronJobDetail.JobType = '7';
```

---

## 8. Async Apex — Full Comparison

| Feature | Scheduled | Future | Queueable | Batch |
|---------|:---------:|:------:|:---------:|:-----:|
| **Interface** | `System.Schedulable` | `@future` annotation | `System.Queueable` | `Database.Batchable` |
| **Trigger mechanism** | Time / CRON | Apex call | Apex call | Apex call |
| **Callouts** | Via delegate only | `@future(callout=true)` | `AllowsCallouts` | `AllowsCallouts` |
| **Monitoring** | `CronTrigger` obj | Not possible | `AsyncApexJob` | `AsyncApexJob` |
| **Chaining** | Triggers Batch/Queue | None | Yes (1 child) | From `finish()` |
| **Recurring** | Yes — CRON | No | No | No |
| **Max active limit** | 100 jobs | 50/tx | 50/tx | 5 concurrent |

---

## 9. Session Q&A — Quick Reference

**Q: Why can't I save changes to my Scheduled Apex class?**
A: Active scheduled jobs exist for that class. Salesforce locks the class from changes while jobs are active to prevent corruption. Solution: go to **Setup → Scheduled Jobs** → Abort all jobs for that class → save the class → reschedule.

---

**Q: What is the difference between Delete and Abort for a scheduled job?**
A: **Delete** simply removes the job entry from the UI — it's a UI-only action. **Abort** calls `System.abortJob(triggerId)` programmatically — it terminates the job in the queue and removes it. Both stop execution, but Abort is the programmatic approach used in deployments.

---

**Q: Can we schedule the same Apex class multiple times?**
A: Yes — you can schedule the same class multiple times as long as each job has a **UNIQUE name**. For example, scheduling every 15 minutes requires 4 separate `System.schedule()` calls with 4 different names. The total org limit is **100** active scheduled jobs.

---

**Q: Can we schedule two different instances of the same class with the same job name?**
A: No. Job names must be unique across all active scheduled jobs regardless of which class they reference. Attempting to schedule with a duplicate name throws an error: `'There is already a job with the same name'`.

---

**Q: What is `CronTrigger` and `CronJobDetail`?**
A: `CronJobDetail` is the parent — it stores the class-level schedule info (name, job type). `CronTrigger` is the child — it stores the per-instance schedule details (CRON expression, next fire time, state). When a class is scheduled multiple times with different names, each creates a separate `CronTrigger` record pointing to the same `CronJobDetail`.

---

**Q: Can Scheduled Apex make HTTP callouts directly?**
A: Not directly. Scheduled Apex does not support `Database.AllowsCallouts`. To make callouts on a schedule, the Schedulable `execute()` must invoke a Batch Apex class (implementing `Database.AllowsCallouts`) or a Queueable class (implementing `Database.AllowsCallouts`). The callout happens in those async classes, not in the Schedulable itself.

---

**Q: How do you check if you're close to the 100 scheduled job limit?**
A: Query: `SELECT COUNT() FROM CronJobDetail WHERE JobType = '7'` — `JobType '7'` represents Scheduled Apex. Or go to **Setup → Scheduled Jobs** and create a custom view filtered by Type = Scheduled Apex.

---

**Q: `Day_of_Month` and `Day_of_Week` — can both have values in a CRON?**
A: No. They are mutually exclusive in Salesforce CRON. If you set `Day_of_Month` to a value, `Day_of_Week` must be `'?'`. If you set `Day_of_Week` to a value (e.g. `MON`), `Day_of_Month` must be `'?'`. Setting both causes a runtime error.

---

## 10. Interview Questions — 5 Years Experience Level

> **INTERVIEW GUIDE**
> These questions test production-level knowledge of Scheduled Apex expected from mid-to-senior Salesforce developers.

---

**Q1. What is Scheduled Apex and how does it work internally?**

**Ans:** Scheduled Apex implements `System.Schedulable` and uses a CRON expression to define when the `execute()` method runs. When you call `System.schedule()`, Salesforce creates a `CronTrigger` record (and a linked `CronJobDetail` record) that the platform's scheduling engine monitors. When the scheduled time arrives, Salesforce invokes the `execute(SchedulableContext SC)` method in a **separate transaction** with fresh governor limits. The job runs asynchronously and doesn't block user activity.

---

**Q2. What are the different ways to schedule an Apex job and what is the key difference?**

**Ans:** Two ways: (1) **Programmatically** via `System.schedule(name, cronExpr, instance)` in the Anonymous Window or Apex class. (2) **Via Setup UI:** Apex Classes → Schedule Apex button → fill the form. Key difference: jobs scheduled programmatically do **NOT** show a `'Manage'` link in the Scheduled Jobs UI, only a Delete option. Jobs scheduled via UI show both Manage and Delete. Programmatic scheduling is preferred for deployments and CI/CD pipelines.

---

**Q3. You have a requirement to run a job every 15 minutes. How would you implement it?**

**Ans:** Since Salesforce does not support a single CRON expression running more frequently than hourly, you must create **4 separate** `System.schedule()` calls with unique names and different minute offsets: `'0 0 * * * ?'` for :00, `'0 15 * * * ?'` for :15, `'0 30 * * * ?'` for :30, `'0 45 * * * ?'` for :45. Each uses the same Schedulable class but a different job name. This consumes **4 of the 100** available scheduled job slots. For every 5 minutes you would need 12 jobs.

---

**Q4. What happens if your organization already has 100 scheduled jobs and you try to add one more?**

**Ans:** Salesforce throws a `System.AsyncException: 'Maximum number of Apex scheduled jobs has been reached.'` To resolve: identify and abort any stale or completed jobs (query `CronTrigger` for `State = 'COMPLETE'` or `'DELETED'`), delete them from Setup, or consolidate multiple schedules into a single Schedulable class that handles multiple tasks in one `execute()` call.

---

**Q5. How would you build a deployment script that safely replaces a running scheduled job?**

**Ans:** Step 1: Query `CronTrigger` for the job by name. Step 2: If found, call `System.abortJob(ct.Id)` for each. Step 3: Save and deploy the updated Schedulable class. Step 4: Re-schedule using `System.schedule()` with the same or new CRON expression. This is often done in a post-install script or an Execute Anonymous block in a deployment pipeline. **Never assume the job doesn't exist — always query first.**

---

**Q6. Can a Scheduled Apex class implement both `System.Schedulable` and `Database.Batchable`?**

**Ans:** Yes — a class can implement multiple interfaces. However, mixing Schedulable and Batchable in one class is generally **not recommended** for maintainability. The common pattern is to have a separate Schedulable class that calls `Database.executeBatch()` on a separate Batchable class. This separation of concerns makes each class independently testable and maintainable. Implementing both in one class is technically valid but architecturally messy.

---

**Q7. How do you test Scheduled Apex in a unit test?**

**Ans:** Wrap the `System.schedule()` call between `Test.startTest()` and `Test.stopTest()`. `Test.stopTest()` forces all scheduled jobs to run synchronously within the test context. After `Test.stopTest()`, assert the expected data changes.

```apex
Test.startTest();
    System.schedule('Test Job', '0 0 1 * * ?', new MySchedulable());
Test.stopTest();
// Assert expected changes

// Verify the CronTrigger was created:
System.assertEquals(1, [SELECT COUNT() FROM CronTrigger
                         WHERE CronJobDetail.Name = 'Test Job']);
```

---

**Q8. What is `SchedulableContext` and what can you extract from it?**

**Ans:** `SchedulableContext` is passed to `execute()` and provides `SC.getTriggerId()` — which returns the `CronTrigger` ID for the current running job. You can use this ID to query `CronTrigger` for the job's next fire time, state, or CRON expression. It is the Scheduled Apex equivalent of `Database.BatchableContext`.

---

**Q9. What is the difference between deleting and aborting a scheduled job?**

**Ans:** **Delete** (via UI) removes the `CronTrigger` record from Setup but only works if the job is in a deletable state. **Abort** (`System.abortJob`) programmatically terminates and removes the job regardless of current state — it's more reliable for automation. In deployments, always use `System.abortJob()` because you need programmatic control. After aborting, the `CronTrigger` record moves to `'DELETED'` state.

---

**Q10. Can you explain a real-world scenario where Scheduled Apex is the right choice over other async types?**

**Ans:** Example: An organization syncs 500K Account records with an external ERP every night at 2 AM. The sync involves querying all accounts, updating status fields, and triggering REST callouts per account. Solution: a Scheduled Apex class runs nightly (CRON: `'0 0 2 * * ?'`) and invokes a Batch Apex class implementing `Database.AllowsCallouts`. The batch handles 500K records in 200-record chunks, each chunk making callouts to the ERP. The Schedulable is just the trigger — the heavy lifting is done by the Batch. Neither Future nor Queueable can handle 500K records reliably.

---

**Q11. What is the governor limit impact on a Scheduled Apex job?**

**Ans:** Each execution of `execute()` runs in a separate Apex transaction with a full set of fresh governor limits (SOQL queries, DML rows, CPU time, heap, etc.). This means the Schedulable `execute()` itself gets fresh limits. If you invoke a Batch from `execute()`, each Batch chunk **also** gets fresh limits — so the effective capacity is much higher than a single synchronous transaction. However, the overall org-level 24-hour async limit (250,000 async calls or licences × 200) still applies across all async jobs.

---

**Q12. How would you handle a situation where a scheduled job's CRON needs to change dynamically based on business rules?**

**Ans:** You cannot modify a CRON expression of an existing job — you must abort it and reschedule. Build a Schedulable class that reads its next CRON from a **Custom Setting** or **Custom Metadata Type**. In the `execute()` method, after running the business logic, read the configured next CRON, abort the current job (`System.abortJob(SC.getTriggerId())`), and re-schedule itself with the new CRON: `System.schedule('Dynamic Job', newCron, new MySchedulable())`. This allows runtime CRON reconfiguration without code deployment.

---

**Q13. How does the 'same name' constraint work for scheduled jobs — and what are the edge cases?**

**Ans:** The job name must be unique among all **ACTIVE** scheduled jobs. If a job has been aborted/deleted (state = `DELETED`), its name becomes available again. Edge cases: (1) Two different Schedulable classes cannot share the same job name. (2) Two instances of the same class cannot share the same name. (3) In test context, `Test.stopTest()` completes the job synchronously — the job name is released after the test. (4) In a deployment, if the old job wasn't properly aborted, the new `System.schedule()` with the same name fails.

---

**Q14. Can a scheduled job reschedule itself? What are the use cases?**

**Ans:** Yes — inside `execute()`, call `System.abortJob(SC.getTriggerId())` to cancel the current schedule, then call `System.schedule()` with a new CRON to re-register the job with a different timing. Use cases: (1) **Self-adjusting retry logic** — if the job found no data to process, schedule itself to retry in 10 minutes. (2) **Dynamic schedule changes** — shift from peak to off-peak times based on org load. (3) **One-time jobs with cleanup** — schedule once, run, then abort without rescheduling.

---

**Q15. How would you monitor and alert if a scheduled job fails?**

**Ans:** Salesforce does not natively send alerts when a Scheduled Apex job fails — it only records the failure in the `CronTrigger State` field. Production monitoring approaches: (1) Query `CronTrigger` for `State = 'ERROR'` in a separate monitoring job. (2) Wrap the `execute()` body in a `try-catch`, and in the catch block, send an email using `Messaging.sendEmail()` or insert a custom log record. (3) Use **Platform Events** from the catch block to trigger a notification flow. (4) Set up a Salesforce Health Cloud or third-party monitoring tool that polls `AsyncApexJob` and `CronTrigger` for errors.

---

## 11. Quick Reference Cheat Sheet

### Complete Scheduled Apex Template

```apex
// ── Schedulable class ──────────────────────────────────────────────────
global class MyScheduledJob implements System.Schedulable {

    global void execute(SchedulableContext SC) {
        // Option A: Trigger Batch Apex
        Database.executeBatch(new MyBatchClass(), 200);

        // Option B: Trigger Queueable Apex
        // System.enqueueJob(new MyQueueable(idSet));

        // Option C: Call regular Apex
        // MyHelper.runLogic();
    }
}

// ── Schedule it ────────────────────────────────────────────────────────
String cron = '0 0 2 * * ?';  // Every day at 2:00 AM
System.schedule('My Daily Job', cron, new MyScheduledJob());

// ── Abort it ───────────────────────────────────────────────────────────
CronTrigger ct = [SELECT Id FROM CronTrigger
                  WHERE CronJobDetail.Name = 'My Daily Job' LIMIT 1];
System.abortJob(ct.Id);

// ── Check count ────────────────────────────────────────────────────────
Integer n = [SELECT COUNT() FROM CronJobDetail WHERE JobType = '7'];
```

---

### Key Numbers to Remember

- **100** — Max active scheduled jobs per org
- **`'7'`** — `JobType` value for Scheduled Apex in `CronJobDetail`
- **1 hour** — Minimum interval for a single scheduled job
- **12 jobs** — Required for every-5-minutes scheduling
- **4 jobs** — Required for every-15-minutes scheduling
- **2 jobs** — Required for every-30-minutes scheduling
- **`SC.getTriggerId()`** — Returns `CronTrigger` ID in `execute()`

---

### CRON Quick Reference

- Every day at midnight: `0 0 0 * * ?`
- Every day at 2 AM: `0 0 2 * * ?`
- Every Monday at 8 AM: `0 0 8 ? * MON`
- Every hour: `0 0 * * * ?`
- Every 30 minutes: `0 0/30 * * * ?` *(use 2 separate jobs)*
- First of every month at midnight: `0 0 0 1 * ?`
- Last day of month: `0 0 0 L * ?`
- Every weekday at 9 AM: `0 0 9 ? * MON-FRI`

---

### Useful Resources

- CRON expression generator: <https://crontab.cronhub.io/>
- Panther Schools — Schedule every 5 mins: <https://www.pantherschools.com/how-to-schedule-batch-apex-for-every-5-minutes/>
- Panther Schools — Schedule Batch Apex: <https://www.pantherschools.com/how-to-schedule-batch-apex-in-salesforce/>
- Salesforce Docs — Schedulable Interface: <https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_scheduler.htm>

---

> *SwiftNotes | Scheduled Apex Comprehensive Notes | Salesforce Developer Series — Day 11.3*