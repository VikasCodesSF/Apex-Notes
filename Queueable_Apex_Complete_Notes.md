# Queueable Apex in Salesforce
### Complete Study Notes + 5-Year Developer Interview Q&A

---

## 1. What is Queueable Apex?

Queueable Apex is a more powerful, monitorable, and flexible successor to `@future` methods. It is implemented by creating an Apex class that implements the `System.Queueable` interface and overrides the `execute(QueueableContext ctx)` method. Jobs are submitted using `System.enqueueJob()`, which returns a **Job ID** for tracking.

### Key Advantages over `@future`

- **Accepts sObject and non-primitive parameters** — pass `List<Account>`, custom objects, Maps directly into the constructor.
- **Returns a Job ID** — track execution status via `AsyncApexJob` SOQL or Apex Jobs UI.
- **Supports chaining** — enqueue a child job from within a running job (1 child per parent).
- **Called from Batch `finish()`** — the only async type that can be cleanly invoked post-batch.
- **Can call `@future` methods** — Queueable can invoke future methods (up to 50 per transaction).

---

## 2. Implementation Steps (4-Step Pattern)

| Step | Action | Detail |
|------|--------|--------|
| 1 | **Create the class** | Implement `System.Queueable` interface on a `public`/`global` class |
| 2 | **Override `execute()`** | Implement `public void execute(System.QueueableContext ctx)` — this is the heart of the job |
| 3 | **Enqueue the job** | `Id jobId = System.enqueueJob(new MyQueueable());` — submits to Apex Queue |
| 4 | **Track the job** | `AsyncApexJob job = [SELECT Status, NumberOfErrors FROM AsyncApexJob WHERE Id=:jobId];` |

### 2.1 Basic Syntax

```apex
// Step 1 & 2: Define the Queueable class
public class AccountQueueable implements System.Queueable {

    // Constructor — accepts sObject parameters (advantage over @future)
    private List<Account> accountList;

    public AccountQueueable(List<Account> accounts) {
        this.accountList = accounts;
    }

    // Step 2: The execute() method — heart of the job
    public void execute(System.QueueableContext ctx) {
        // Business logic here
        for (Account acc : accountList) {
            acc.Rating = 'Hot';
        }
        update accountList;
    }
}

// Step 3: Enqueue the job
List<Account> accs = [SELECT Id, Name, Rating FROM Account LIMIT 100];
Id jobId = System.enqueueJob(new AccountQueueable(accs));

// Step 4: Track the job
AsyncApexJob job = [SELECT Id, Status, NumberOfErrors, CreatedDate
                    FROM AsyncApexJob WHERE Id = :jobId];
System.debug('Job Status: ' + job.Status);
```

---

## 3. Job Chaining in Queueable Apex

Chaining means enqueuing a new Queueable job from inside the running `execute()` method. This is used when the output of Job A becomes the input of Job B. **Each parent job can enqueue only one child job.**

```apex
// Parent Queueable: HiringManagerQueueable
public class HiringManagerQueueable implements System.Queueable {

    public void execute(System.QueueableContext ctx) {
        // Insert Hiring Manager (Setup-like custom object)
        Hiring_Manager__c hr = new Hiring_Manager__c();
        hr.Name = 'Bhuvan Kumar';
        hr.Designation__c = 'HR Recruiter';
        insert hr;

        if (hr.Id != null) {
            // Chain: pass the sObject to the child job
            Id childJobId = System.enqueueJob(new PositionQueueable(hr));
            System.debug('Child Job Id: ' + childJobId);
        }
    }
}

// Child Queueable: PositionQueueable — receives sObject from parent
public class PositionQueueable implements System.Queueable {

    private Hiring_Manager__c hrInfo;

    public PositionQueueable(Hiring_Manager__c hrRecord) {
        this.hrInfo = hrRecord;
    }

    public void execute(System.QueueableContext ctx) {
        Position__c pos = new Position__c();
        pos.Name = 'Java Developer';
        pos.Hiring_Manager__c = hrInfo.Id;  // FK from parent
        insert pos;
    }
}

// Execution
Id jobId = System.enqueueJob(new HiringManagerQueueable());
```

> **■ CHAINING RULE**
> Each parent can enqueue **ONLY ONE** child. Attempting to enqueue two child jobs from one `execute()` throws: `"Too many queueable jobs added to the queue: 2"`. Chain depth: **Unlimited** in Production; **max 5** (or 4 children from 1 parent) in Developer/Trial Edition.

---

## 4. Hands-On Exercise Solution

**Exercise:** Create a Queueable job to find all Accounts with no Phone, create a Task for each (Subject: "Phone Number is missing", Status: Not Started, Priority: Urgent), and assign it to the Account Owner.

```apex
public class MissingPhoneTaskQueueable implements System.Queueable {

    public void execute(System.QueueableContext ctx) {
        // Query accounts with no phone
        List<Account> accsNoPhone = [SELECT Id, Name, OwnerId
                                     FROM Account
                                     WHERE Phone = null];

        List<Task> tasksToInsert = new List<Task>();

        for (Account acc : accsNoPhone) {
            Task t = new Task();
            t.Subject   = 'Phone Number is missing';
            t.Status    = 'Not Started';
            t.Priority  = 'Urgent';
            t.Description = 'The phone number for the account ' + acc.Name
                          + ' is missing. Please collect the information asap';
            t.OwnerId   = acc.OwnerId;  // Account Owner
            t.WhatId    = acc.Id;       // Related To Account
            tasksToInsert.add(t);
        }

        if (!tasksToInsert.isEmpty()) {
            insert tasksToInsert;
        }
    }
}

// Execute
Id jobId = System.enqueueJob(new MissingPhoneTaskQueueable());
System.debug('Job ID: ' + jobId);
```

---

## 5. Queueable Apex Limits & Governor Limits

| Limit | Value | Notes |
|-------|-------|-------|
| **Jobs enqueued / transaction** | 50 | Same as `@future` per-tx limit |
| **Chaining depth (Production)** | Unlimited | No depth cap in production orgs |
| **Chaining depth (Dev Edition)** | Max 5 | 1 parent + 4 children |
| **Child jobs per parent** | 1 only | Enqueue 2+ from same `execute()` = error |
| **Jobs from `@future` method** | 1 only | Max 1 Queueable from a `@future` |
| **Flex Queue capacity** | 100 jobs max | Holding area before execution queue |
| **Can call `@future` from Queueable?** | Yes (up to 50) | Normal async limits apply |
| **Can call Batch from Queueable?** | Yes | Via `Database.executeBatch()` |
| **Job ID returned?** | Yes | Use for `AsyncApexJob` tracking |
| **sObject parameters?** | Yes | Key advantage over `@future` |

---

## 6. Queueable vs Future — Full Comparison

| Aspect | `@future` Method | Queueable Apex |
|--------|:----------------:|:--------------:|
| **Type** | Static Method | Apex Class (implements `Queueable`) |
| **Annotation / Interface** | `@future` | `implements System.Queueable` |
| **Parameter types** | Primitives only (no sObject) | Primitives + sObject + Collections |
| **Returns Job ID?** | No (`void` only) | Yes (`System.enqueueJob` returns `Id`) |
| **Monitoring** | Apex Jobs UI only (no SOQL) | Apex Jobs UI + `AsyncApexJob` SOQL |
| **Chaining** | Not allowed | 1 child per parent (unlimited depth) |
| **Call from Trigger** | Yes | Yes |
| **Call from Batch `finish()`** | No | Yes |
| **Call from another Future** | No | N/A (Queueable is called via enqueue) |
| **Can call `@future`?** | No (from another future) | Yes (up to 50 per tx) |
| **Callout support** | `@future(callout=true)` | `implements Database.AllowsCallouts` |
| **Data loss risk** | Yes (stale sObject workaround needed) | Lower (pass live sObject) |
| **Best for** | Simple callouts, Mixed DML | Complex jobs, chaining, monitoring |

---

## 7. All 4 Async Types — Full Decision Matrix

| Feature | Future | Queueable | Batch | Scheduled |
|---------|:------:|:---------:|:-----:|:---------:|
| **Structure** | Static Method | Apex Class | Apex Class | Apex Class |
| **Interface / Annotation** | `@future` | `System.Queueable` | `Database.Batchable` | `Schedulable` |
| **sObject params** | No | Yes | N/A (uses query) | N/A |
| **Job ID** | No | Yes | Yes | Yes |
| **Chaining** | No | Yes (1 child) | Via `finish()`+Queueable | No |
| **Max records** | ~10-20 | Limited by heap | 50 million | Depends on logic |
| **Callouts** | `@future(callout=true)` | `+ AllowsCallouts` | `+ AllowsCallouts` | `+ AllowsCallouts` |
| **Trigger-safe callout** | Yes | Yes | Yes | Yes |
| **Mixed DML fix** | Yes | Yes | N/A | N/A |
| **Best for** | Callouts, MixedDML | Complex, chaining | Bulk millions | Scheduled tasks |

---

## 8. Interview Q&A — 5-Year Salesforce Developer Level

> **INTERVIEW GUIDE**
> These questions are calibrated for developers with ~5 years Salesforce experience. Interviewers expect trade-off awareness, design reasoning, governor-limit precision, and real-world scenario thinking — not just textbook definitions.

---

### Part A — Core Queueable Concepts

---

**Q1. What is Queueable Apex? How is it different from a `@future` method?**

**Ans:** Queueable Apex is an Apex class that implements `System.Queueable` and overrides `execute(QueueableContext)`. Unlike `@future` (which is a static method):
1. Queueable can accept sObject and non-primitive parameters in its constructor.
2. `System.enqueueJob()` returns a Job ID for monitoring.
3. Queueable supports job chaining — one child per parent.
4. It can be called from Batch `finish()`.

Use `@future` for simple fire-and-forget callouts or Mixed DML. Use Queueable when you need sObject params, chaining, or monitoring.

> **💡 5-YR TIP:** Frame it as: "Queueable solves the top 3 limitations of `@future`" — no tracking, no sObjects, no chaining.

---

**Q2. How does `System.enqueueJob()` work? What does it return and how do you use the return value?**

**Ans:** `System.enqueueJob(new MyQueueable())` places the job in the Apex **Flex Queue** (which holds up to 100 pending jobs) and returns an `Id` (the `AsyncApexJob` record ID). You use this ID to query job status:

```apex
AsyncApexJob job = [SELECT Status, NumberOfErrors, JobItemsProcessed, TotalJobItems
                    FROM AsyncApexJob WHERE Id = :jobId];
```

Status values: `Queued → Preparing → Processing → Completed / Failed / Aborted`. You can also monitor from **Setup → Apex Jobs**.

> **💡 5-YR TIP:** Mention the **Flex Queue** (100 capacity) — shows deeper platform knowledge.

---

**Q3. How do you pass an sObject to a Queueable class? Write the pattern.**

**Ans:** Pass sObjects through the constructor — unlike `@future`, Queueable constructors have no parameter type restrictions. Pattern: define member variables, set them in the constructor, use them in `execute()`.

```apex
public class MyQueueable implements System.Queueable {
    private List<Account> accounts;

    public MyQueueable(List<Account> accounts) {
        this.accounts = accounts;
    }

    public void execute(QueueableContext ctx) {
        /* use accounts */
    }
}

Id jobId = System.enqueueJob(new MyQueueable([SELECT Id FROM Account LIMIT 50]));
```

This is the **#1 reason** to prefer Queueable over `@future` when the job needs live record data.

---

**Q4. Explain job chaining in Queueable. What are the limits?**

**Ans:** Chaining = calling `System.enqueueJob()` from inside `execute()` of a running Queueable job. This creates a parent→child relationship where the child receives the parent's output. Limits:
1. Only **ONE** child job can be enqueued per parent `execute()` — enqueuing 2 throws `"Too many queueable jobs: 2"`.
2. In **Production**: no depth limit — chains can be arbitrarily long.
3. In **Developer/Trial Edition**: max depth is 5 (i.e., 1 parent + 4 children in chain).

Use case: Job A processes Accounts → chains Job B to process related Contacts → chains Job C for notifications.

> **💡 5-YR TIP:** Always distinguish Production (unlimited) vs Developer Edition (max 5) — interviewers test this.

---

**Q5. How do you implement callouts in a Queueable class?**

**Ans:** Add the `Database.AllowsCallouts` marker interface alongside `System.Queueable`:

```apex
public class MyCalloutQueueable implements System.Queueable, Database.AllowsCallouts {

    public void execute(QueueableContext ctx) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('https://api.example.com/sync');
        req.setMethod('POST');
        Http h = new Http();
        HttpResponse res = h.send(req);
    }
}
```

Without `Database.AllowsCallouts`, the callout throws a runtime error. Unlike `@future` which uses `@future(callout=true)`, Queueable uses an **interface**.

> **💡 5-YR TIP:** The marker interface distinction vs `@future` annotation is a common interview gotcha.

---

### Part B — Advanced & Cross-Topic Questions

---

**Q6. Can you call a `@future` method from Queueable? What is the practical limit?**

**Ans:** Yes — a Queueable `execute()` can call `@future` methods. The documented limit is **50 `@future` calls per transaction**, which applies inside Queueable as well. Practically, calling multiple `@future` methods from one Queueable is allowed and works. The reverse (calling Queueable from `@future`) is limited to **1 Queueable job** only. Design note: mixing async types unnecessarily complicates debugging — prefer keeping async work in one type where possible.

---

**Q7. Can you call Queueable from a Batch class? How?**

**Ans:** Yes — from the Batch `finish()` method **only** (not `execute()`). This is a standard pattern: Batch processes large data → `finish()` chains a Queueable for post-processing or notifications.

```apex
public void finish(Database.BatchableContext bc) {
    System.enqueueJob(new PostProcessQueueable(bc.getJobId()));
}
```

You cannot reliably call Queueable from Batch `execute()` because `execute()` can run in parallel across chunks.

> **💡 5-YR TIP:** The `finish()`-only rule is frequently asked — shows understanding of Batch execution model.

---

**Q8. What happens if you enqueue two child jobs from a single Queueable `execute()` method?**

**Ans:** You get a runtime exception:
```
System.LimitException: Too many queueable jobs added to the queue: 2
```
This is **not** a compile-time error — the code saves and deploys fine but fails at runtime. Root cause: Salesforce enforces that parent-child chaining is strictly linear (one-to-one), to prevent exponential fan-out of async jobs that could overwhelm the platform queue. **Workaround:** If you truly need parallel processing, use **Batch Apex** (which processes chunks in parallel).

---

**Q9. How do you test a Queueable Apex class? What is the key test pattern?**

**Ans:** Wrap the `enqueueJob` call in `Test.startTest()` / `Test.stopTest()` — the `stopTest()` forces all queued async jobs to execute synchronously before assertions run.

```apex
@isTest
static void testQueueable() {
    Account acc = new Account(Name='Test');
    insert acc;

    Test.startTest();
        System.enqueueJob(new MyQueueable(new List<Account>{acc}));
    Test.stopTest();

    // Assert post-execution state here
    Account updated = [SELECT Rating FROM Account WHERE Id=:acc.Id];
    System.assertEquals('Hot', updated.Rating);
}
```

For chaining: only the **first-level** job executes in tests — child jobs **do NOT run**. Use conditional logic (`Test.isRunningTest()`) to skip chaining in test context if needed.

> **💡 5-YR TIP:** Knowing that child chaining does NOT execute in tests is a senior-level detail interviewers love.

---

**Q10. What is the Apex Flex Queue? How does it relate to Queueable?**

**Ans:** The Apex Flex Queue is a **holding area** that can hold up to **100 pending** Queueable jobs in `"Holding"` status. When server resources become available, jobs move from Flex Queue to the Processing Queue (which can hold up to 100 active jobs). `System.enqueueJob()` places the job in the Flex Queue first. You can view Flex Queue contents via: **Setup → Apex Flex Queue**.

Job status flow: `Holding → Queued → Preparing → Processing → Completed/Failed/Aborted`

The Flex Queue allows Salesforce to buffer large numbers of async requests without immediately consuming resources.

---

**Q11. Future method vs Queueable vs Batch — when do you choose each?**

**Ans:**

**`@future`:** Use for simple, fire-and-forget operations — trigger callouts, Mixed DML fix, small data sets. Minimal code overhead.

**Queueable:** Use when you need sObject parameters, job chaining, monitoring, or to be called from Batch. Also preferred for moderate complexity where `@future` falls short but full Batch is overkill.

**Batch:** Use for very large data volumes (thousands to 50 million records), or when you need chunked, recoverable processing with per-chunk error isolation.

**Decision tree:**
- Callout from trigger? → `@future(callout=true)`
- sObjects/chaining needed? → Queueable
- Large volume? → Batch
- Time-based? → Scheduled

> **💡 5-YR TIP:** A clean decision tree shows architectural thinking, not just feature knowledge.

---

**Q12. Can you call a Schedulable class from a Queueable? What is the use case?**

**Ans:** Yes — `System.schedule()` can be called from within `execute()`. Use case: a Queueable job that dynamically schedules a follow-up batch job at a future time. Limit: you can schedule up to **100 jobs** total in the org at any time. Example: after processing records, schedule a nightly cleanup job:

```apex
System.schedule('Nightly Cleanup', '0 0 2 * * ?', new CleanupSchedulable());
```

Practical note: mixing async types like this should be intentional — always document the job chain clearly for maintainability.

---

**Q13. How do you debug a Queueable job that is failing silently?**

**Ans:** Multiple approaches:

1. **Query `AsyncApexJob`:**
```apex
SELECT Status, ExtendedStatus, NumberOfErrors
FROM AsyncApexJob
WHERE JobType='Queueable'
ORDER BY CreatedDate DESC LIMIT 10
```
`ExtendedStatus` shows the exception message.

2. **Setup → Apex Jobs** — shows status, errors, and timestamps visually.
3. Wrap `execute()` in `try-catch` and insert to a custom `Error_Log__c` object with exception type, message, and stack trace.
4. Enable **Debug Logs** for the running user/class — trace the execution.
5. **Platform Events** — emit an error event that a Flow or subscriber can capture in real time.

Unlike `@future`, Queueable's Job ID means you can always trace back to the exact failed job record.

> **💡 5-YR TIP:** Mentioning `ExtendedStatus` field on `AsyncApexJob` is a very senior-level detail.

---

### Part C — Scenario / Design Questions

---

**Q14. Design: On Opportunity close-won, sync data to an external CRM and then create a follow-up Task. The sync may take time. How do you architect this?**

**Ans:** Architecture:

1. **Trigger (AfterUpdate)** on Opportunity — filter for Stage changed to `"Closed Won"`.
2. Collect Opportunity IDs, call `System.enqueueJob(new CRMSyncQueueable(oppIds))`.
3. `CRMSyncQueueable` implements `System.Queueable, Database.AllowsCallouts`:
   - Re-query Opportunities with required fields.
   - Build HTTP POST payload, call external CRM API.
   - On success, chain: `System.enqueueJob(new TaskCreationQueueable(oppIds))`.
4. `TaskCreationQueueable` creates follow-up Tasks for each Opportunity.

**Why Queueable over `@future`:** sObject data needed, chaining required, monitoring desired.
**Resilience:** add Platform Event-based retry for failed callouts, log errors in `Integration_Log__c`.

> **💡 5-YR TIP:** A two-job chain showing the real power of Queueable over `@future` — this is what 5-year level looks like.

---

**Q15. A Queueable job processes 100 accounts, and you need to ensure it does not reprocess already-handled records if it runs again (idempotency). How do you design this?**

**Ans:** Idempotency strategies:

1. **Custom status field** (`Sync_Status__c = "Processed"`) — query only `WHERE Sync_Status__c != "Processed"` and set it to `"Processed"` after DML. Wrap in `try-catch` so failures leave the flag unset for retry.
2. **Custom Timestamp field** (`Last_Synced__c`) — only process records where `Last_Synced__c = null` or `Last_Synced__c < some threshold`.
3. **External ID matching** — if syncing to an external system, check external ID presence before insert/update. Also consider `Database.upsert()` with an external ID field instead of `insert()` to make the DML itself idempotent.

This is critical for async jobs because they can re-run after server failures without warning.

> **💡 5-YR TIP:** Idempotency is a senior architecture concern — most junior devs miss it entirely.

---

## 9. Quick Cheat Sheet

| Item | Queueable Value |
|------|----------------|
| **Interface** | `implements System.Queueable` |
| **Method to implement** | `public void execute(System.QueueableContext ctx)` |
| **Submit job** | `Id jobId = System.enqueueJob(new MyQueueable());` |
| **Track job** | `SELECT Status, ExtendedStatus FROM AsyncApexJob WHERE Id=:jobId` |
| **sObject params?** | Yes — via constructor |
| **Job ID returned?** | Yes — by `System.enqueueJob()` |
| **Chaining** | 1 child per parent; unlimited depth (Prod); max 5 (Dev Ed) |
| **Callout support** | `implements Database.AllowsCallouts` |
| **Max jobs / transaction** | 50 |
| **Flex Queue capacity** | 100 pending jobs |
| **Call from Batch?** | Yes — from `finish()` only |
| **Call `@future` from Queueable?** | Yes — up to 50 per tx |
| **Test pattern** | `Test.startTest()` → `enqueueJob()` → `Test.stopTest()` → assert |
| **Chaining in test** | Only first-level job runs; child jobs do **NOT** execute |
| **Job status flow** | `Holding → Queued → Preparing → Processing → Completed/Failed` |

---

> **⏭️ NEXT UP: Batch Apex (Day 11.1)**
> Next topic in Async Apex series: **Batch Apex** — `Database.Batchable` interface, Start/Execute/Finish methods, 50 million record processing, `Database.executeBatch()`.