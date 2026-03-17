# Batch Apex & Async Apex in Salesforce
### Comprehensive Notes + 5-Year Experience Interview Q&A

---

## 1. Synchronous vs Asynchronous Operations

### Synchronous Operations

In synchronous operations, tasks execute sequentially — one by one. No next operation starts until the current one finishes. This wastes both time and resources.

- Validation Rules, Formulas, Sharing Rules, Assignment Rules
- Workflow Rules, Process Builder, Lightning Flows
- Auto-Response Rules, Escalation Rules, Apex Triggers

> 📌 **Salesforce Classic** is purely synchronous. **Salesforce Lightning** is async while maintaining a Queue.

### Asynchronous Operations

In asynchronous operations, tasks are placed in a queue and processed when resources are available. The client does not wait for the response — it follows a **First-Come-First-Served** model.

- Example: Messaging people for a party — if their phone is unreachable, the message is queued and retried.

### ▶ Types of Async Apex in Salesforce

- **Batch Apex** — Long-running processes on millions of records
- **Scheduled Apex** — Execute code at a specific time or interval
- **Flex Queues** — Hold batch jobs waiting for resources
- **Future Methods** — Simple async method for callouts or DML isolation
- **Queueable Apex** — Advanced async with chaining and complex types

---

## 2. Batch Apex — Core Concepts

Batch Apex is used for complex, long-running processes that handle thousands to millions of records — far beyond what a single Apex transaction can process.

### Key Facts

- Implements the `Database.Batchable<sObject>` interface
- Processes up to **50 million records** per job
- Records are broken into chunks (batches) and processed separately
- Each chunk runs in its **own separate transaction** with fresh governor limits
- Batch classes should be declared as `global` (they execute outside the org)

> ⚠️ **Future methods CANNOT be called from Batch Apex.** Only Queueable and another Batch can be called from `finish()`.

### Batch Size Limits

| Limit / Feature | Value | Notes |
|----------------|-------|-------|
| **Max records processed** | 50 Million | Via `Database.QueryLocator` |
| **Default batch size** | 200 records | Can be overridden |
| **Min batch size** | 1 record | — |
| **Max batch size (QueryLocator)** | 2,000 records | — |
| **Max batch size (Iterator)** | No upper limit | Subject to governor limits |
| **Max concurrent batch jobs** | 5 active jobs | Others move to Flex Queue |
| **Max async jobs in 24 hrs** | 250,000 OR licences × 200 | Whichever is greater |
| **Max Flex Queue jobs** | 100 jobs | Holding status |
| **Governor limits per batch** | Reset each `execute()` | Fresh limits per chunk |

---

## 3. The Three Batch Apex Methods

### 3.1 `start()` Method

- Executes only **ONCE** per batch job — even if the query returns 0 records
- Returns a `Database.QueryLocator` or an `Iterable<SObject>`
- Does **NOT** return records — it returns a locator/iterator pointing to records in memory
- `Database.BatchableContext` provides `jobId` and `childJobId` for monitoring

```apex
public Database.QueryLocator start(Database.BatchableContext BC) {
    return Database.getQueryLocator(
        'SELECT Id, Name, Phone FROM Account WHERE Phone = null'
    );
}
```

> 📌 `Database.getQueryLocator` is a **METHOD**. `Database.QueryLocator` is a **CLASS**. Don't confuse them.

### ▶ QueryLocator vs Iterator

| Type | Use Case | Max Batch Size | Max Total Records |
|------|----------|:--------------:|:-----------------:|
| **QueryLocator** | Standard SOQL queries | 2,000 records | 50 million |
| **Iterable** | Custom data processing not possible through SOQL `WHERE` clauses | No upper limit | 50K SOQL row limit applies to underlying query |

---

### 3.2 `execute()` Method

- Executes **ONLY** if the `start()` query returns at least 1 record
- Runs **N times**, where `N = Math.ceil(totalRecords / batchSize)`
- Each execution is a **SEPARATE transaction** with a separate `jobId`
- Governor limits are **RESET** for every `execute()` call
- Use `Database.update(list, false)` instead of plain DML to enable partial processing

```apex
public void execute(Database.BatchableContext BC, List<Account> scope) {
    // BC.getJobId()      → Monitor the overall batch job
    // BC.getChildJobId() → Monitor this specific chunk's execution
    // Each execute() runs in a SEPARATE transaction

    for (Account acc : scope) {
        acc.Rating = 'Hot';
    }

    List<Database.SaveResult> results = Database.update(scope, false);

    for (Database.SaveResult sr : results) {
        if (!sr.isSuccess()) {
            // Handle failure — log error, track failed IDs
        }
    }
}
```

> 📌 **Execute count formula:** 5000 records ÷ 200 batch size = **25** `execute()` calls. 110 ÷ 200 = **1**. 250 ÷ 200 = **2**.

---

### 3.3 `finish()` Method

- Always executes — exactly **ONCE** — after all batches are done
- Used for cleanup logic: sending email alerts, calling another batch, logging
- Can query `AsyncApexJob` to get the complete batch job result summary
- Can call another batch class from here (batch chaining)

```apex
public void finish(Database.BatchableContext BC) {
    AsyncApexJob job = [SELECT Id, Status, TotalJobItems,
                               JobItemsProcessed, NumberOfErrors,
                               CreatedBy.Email
                        FROM AsyncApexJob WHERE Id = :BC.getJobId()];

    // Send email notification
    // Call another batch if needed: Database.executeBatch(new NextBatch());
}
```

---

## 4. How to Execute a Batch Job

### Syntax

```apex
// Default batch size (200 records per chunk)
AccountsBatchClass batchJob = new AccountsBatchClass();
Id jobId = Database.executeBatch(batchJob);

// Custom batch size
Integer BATCH_SIZE = 50;
Id jobId = Database.executeBatch(batchJob, BATCH_SIZE);

// Inline shorthand
Database.executeBatch(new AccountsBatchClass(), 100);
```

### Batch Job Statuses

| Status | Meaning |
|--------|---------|
| **Holding** | Submitted to Flex Queue, waiting for resources |
| **Queued** | Waiting for execution in the main queue |
| **Preparing** | The `start()` method is being invoked |
| **Processing** | The `execute()` method is actively running |
| **Aborted** | Manually aborted by user or from Apex code |
| **Completed** | Job finished — with or without record failures |
| **Failed** | Job experienced a system-level failure |

---

## 5. Advanced Interfaces & Patterns

### 5.1 `Database.Stateful`

By default, batch Apex is **STATELESS** — each `execute()` starts fresh and cannot remember values from the previous chunk. `Database.Stateful` enables state retention across chunks.

- **Without Stateful:** variable values reset to initial state before every `execute()`
- **With Stateful:** instance variables are preserved across all `execute()` calls
- Governor limits still reset per chunk — only variables retain their values
- Use case: counting successes/failures across all batches, accumulating totals

```apex
public class ContactsBatchClass
    implements Database.Batchable<sObject>, Database.Stateful {

    private Integer successCount = 0;  // Preserved across all execute() calls
    private Integer failureCount = 0;

    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator(
            'SELECT Id, Phone, Account.Phone FROM Contact LIMIT 10000'
        );
    }

    public void execute(Database.BatchableContext bc, List<Contact> contacts) {
        for (Contact con : contacts) {
            con.Phone = con.Account.Phone;
        }

        List<Database.SaveResult> results = Database.update(contacts, false);

        for (Database.SaveResult sr : results) {
            if (sr.isSuccess()) { successCount++; }
            else { failureCount++; }
        }
    }

    public void finish(Database.BatchableContext bc) {
        System.debug('Success: ' + successCount + ' | Failed: ' + failureCount);
    }
}
```

---

### 5.2 `Database.AllowsCallouts`

By default, batch Apex **cannot** make HTTP callouts. Implement `Database.AllowsCallouts` to enable external API calls from the `execute()` method.

- Callouts must be made in `execute()` — commit results in `finish()`
- Error without interface: `System.LimitException: Too many callouts: 1`
- Batch Apex can work around the 100 callout governor limit per transaction since each chunk is a separate transaction

```apex
public class MyBatch implements Database.Batchable<sObject>,
                                Database.AllowsCallouts {
    // Now external HTTP callouts are allowed in execute()
}
```

---

### 5.3 Scheduled Apex

Scheduled Apex executes code at a specified time or recurring interval — daily, weekly, hourly, monthly, etc. It implements `System.Schedulable` and is commonly used to trigger batch jobs automatically.

```apex
public class ContactsBatchSchedule implements System.Schedulable {
    public void execute(SchedulableContext SC) {
        ContactsBatchClass batchClass = new ContactsBatchClass();
        Database.executeBatch(batchClass);
    }
}

// Schedule via Apex (Anonymous window or code):
// Cron: 'Seconds Minutes Hours Day Month Weekday Year'
String cron = '0 0 2 * * ?';  // Every day at 2 AM
System.schedule('Daily Batch Job', cron, new ContactsBatchSchedule());
```

---

## 6. Async Apex Comparison Table

| Feature | Future Method | Queueable Apex | Batch Apex | Scheduled Apex |
|---------|:-------------:|:--------------:|:----------:|:--------------:|
| **Records** | Small set | Medium (50K) | 50 Million | Depends on job |
| **Chaining** | No | Yes (1 level) | Yes (finish method) | No |
| **Callouts** | `@future(callout=true)` | `Database.AllowsCallouts` | `Database.AllowsCallouts` | Via batch |
| **Monitor** | No | Yes (jobId) | Yes (AsyncApexJob) | Yes |
| **State** | Stateless | Stateless | Stateful (opt-in) | N/A |
| **Chunk processing** | No | No | Yes | No |
| **Governor limits** | Separate TX | Separate TX | Reset per chunk | Same as sync |

> **Rule of thumb:** Future → simple one-off async. Queueable → complex/chained async. Batch → massive data volumes. Scheduled → time-based triggers.

---

## 7. Common Q&A from Session *(Day 11.1 – 11.3)*

**Q: Can we call a batch from batch Apex?**
A: Yes — but **ONLY** from the `finish()` method. You cannot call a batch from `start()` or `execute()`.

---

**Q: Can we call a Queueable Apex from batch Apex?**
A: Yes — you can enqueue a Queueable from the `execute()` or `finish()` method of a batch class.

---

**Q: Can we call a batch from Queueable Apex?**
A: Yes — `Database.executeBatch()` can be called from within a Queueable's `execute()` method.

---

**Q: Can we call a Future method from batch Apex?**
A: **No** — calling `@future` methods from batch Apex is NOT allowed. This is a known limitation.

---

**Q: Can we call batch Apex from a Future method?**
A: Yes — `Database.executeBatch()` can be invoked from within a `@future` method.

---

**Q: What is the maximum number of async methods in 24 hours?**
A: **250,000 OR (total Salesforce licences × 200)** — whichever is greater, per 24-hour period across all async Apex types.

---

**Q: Can we make callouts from batch Apex?**
A: Yes — implement `Database.AllowsCallouts` interface. Without it: `System.LimitException: Too many callouts: 1`.

---

**Q: Can we make callouts from Future methods?**
A: Yes — annotate with `@future(callout=true)`. Without it: `'Callouts are not allowed from the future method'`.

---

**Q: Can we make callouts from Queueable Apex?**
A: Yes — implement `Database.AllowsCallouts` interface. Same as batch Apex.

---

## 8. Interview Questions — 5 Years Experience Level

> **INTERVIEW GUIDE**
> The following questions test deep understanding expected from a mid-to-senior Salesforce developer. Expect these at 4–6 year experience interviews.

---

### Conceptual & Architecture Questions

---

**Q1. What is Batch Apex and when would you use it over a Data Loader?**

**Ans:** Batch Apex is used when the data to be updated requires runtime custom calculations, relationship queries, external callouts, or complex logic that cannot be driven by a static Excel/CSV file. Data Loader is appropriate only for static, predefined data updates. Examples requiring Batch Apex: updating parent records based on child aggregation, syncing data with external APIs per record, applying complex business rules at scale.

---

**Q2. What is `Database.Stateful`? When do you need it?**

**Ans:** By default, batch Apex is stateless — each `execute()` transaction resets all instance variables. `Database.Stateful` makes the batch retain variable values across all `execute()` calls. Governor limits still reset per chunk, but class-level variables persist. Use case: tracking success/failure counts across all batches, accumulating totals like revenue, collecting failed record IDs to reprocess. Without Stateful, you cannot reliably identify which specific records failed across different batches.

---

**Q3. How does the partial processing mechanism work in Batch Apex?**

**Ans:** Each `execute()` runs in an isolated transaction. If records in one batch fail, that batch is rolled back but the remaining batches continue processing. To enable partial processing within a single `execute()` call itself, use `Database.update(list, false)` instead of plain DML. The second parameter `false` means "do not throw exception on failure" — it returns a `List<Database.SaveResult>` you can inspect per record.

---

**Q4. What is the difference between `Database.QueryLocator` and `Iterable` in the `start()` method?**

**Ans:** `Database.QueryLocator`: Used for standard SOQL queries. Bypasses the 50K SOQL row governor limit (supports up to 50 million records). Max batch size is 2,000. Most commonly used approach. `Iterable<SObject>`: Used when you need custom data processing logic that cannot be expressed in a SOQL `WHERE` clause (e.g., filtering in-memory, reading from custom data structures). The 50K SOQL row limit still applies to any underlying query. No upper limit on batch size.

---

**Q5. How do you identify and reprocess failed records in a batch?**

**Ans:** You must implement `Database.Stateful` and maintain a collection (`List` or `Set`) of failed record IDs as instance variables. In `execute()`, use `Database.update(list, false)`, iterate over the `SaveResult` list, and add the IDs of failed records to your collection. In `finish()`, you can then pass those IDs to another batch for retry processing. Without `Database.Stateful`, the failed IDs collected in one `execute()` are lost before the next one runs.

---

**Q6. How do you monitor a batch job programmatically?**

**Ans:** Query the `AsyncApexJob` standard object using the job ID from `Database.executeBatch()` or `BC.getJobId()`. Key fields: `Status`, `TotalJobItems`, `JobItemsProcessed`, `NumberOfErrors`, `CreatedBy.Email`. The `BC.getChildJobId()` inside `execute()` gives the ID for that specific chunk's transaction. You can also monitor from **Setup → Apex Jobs** in the Salesforce UI.

---

**Q7. What happens if an exception is thrown inside `execute()` without `Database.update(list, false)`?**

**Ans:** If you use a plain DML statement (`update list`) and any record fails, an exception is thrown that rolls back the entire current batch chunk. The batch job continues with the next chunk, but all records in the failed chunk are not processed. Using `Database.update(list, false)` prevents exceptions from propagating — instead it returns `SaveResult` objects so you can handle failures gracefully per record.

---

**Q8. Can you explain batch chaining and why it's needed?**

**Ans:** Batch chaining means calling a new batch from the `finish()` method of the current batch. Use case: when you have more than 50 million records, you can split across chained batches. Or when different processing logic is needed for different subsets of records. Up to 5 batch jobs can run concurrently; chaining is sequential so it does not count towards concurrency limits while the current job is still running.

---

**Q9. Why would you use Queueable Apex instead of Batch Apex for certain scenarios?**

**Ans:** Queueable Apex is better when: (1) you need to pass complex object types (non-primitive) as parameters, (2) you want to chain jobs where each depends on the result of the previous, (3) the data volume is moderate (under 50K), (4) you need a job ID immediately for tracking. Batch Apex is overkill for small datasets and has more overhead. Queueable is lighter and more flexible for complex workflows.

---

**Q10. If your `start()` returns 2,000 records and batch size is 10, how many callouts can you make?**

**Ans:** The `execute()` method runs **200 times** (2000 ÷ 10). Each `execute()` is a separate transaction with a fresh governor limit of 100 callouts per transaction. So theoretically **200 × 100 = 20,000 callouts** total. However, Salesforce also has org-wide daily callout limits. The key insight: batch Apex helps work around the per-transaction 100 callout limit by distributing callouts across multiple transactions.

---

**Q11. What is the difference between `BC.getJobId()` and `BC.getChildJobId()`?**

**Ans:** `BC.getJobId()` returns the ID of the **overall parent batch job** — same value in `start()`, all `execute()` calls, and `finish()`. `BC.getChildJobId()` is only available inside `execute()` and returns the **unique ID for that specific chunk's transaction** — useful for granular monitoring or debugging a specific failing chunk.

---

**Q12. How would you implement a batch that sends an email after completion with job statistics?**

**Ans:** In the `finish()` method: query `AsyncApexJob` using `BC.getJobId()` to get `TotalJobItems`, `JobItemsProcessed`, `NumberOfErrors`, and `CreatedBy.Email`. Construct a `Messaging.SingleEmailMessage`, set the `to`/`subject`/`htmlBody` fields, and call `Messaging.sendEmail()`. If you're also tracking custom counts (success/failure per record), implement `Database.Stateful` to preserve those variables and include them in the email body.

---

**Q13. What is the Flex Queue and how does it relate to batch jobs?**

**Ans:** When more than 5 batch jobs are submitted simultaneously, the excess jobs go into the **Flex Queue** (`Holding` status) instead of the main Apex Job Queue (`Queued` status). The Flex Queue holds up to **100 jobs**. Jobs automatically move from `Holding → Queued` as active jobs complete. You can also programmatically reorder jobs in the Flex Queue using `FlexQueue.moveBeforeJob()` / `moveAfterJob()`.

---

**Q14. Can a batch Apex class be `global` or `public`? Does it matter?**

**Ans:** Batch Apex classes should be declared as `global`. The reason is that batch jobs execute **outside the organization's transaction boundary** — they run as background jobs managed by Salesforce's async framework. For the framework to invoke the `start()`, `execute()`, and `finish()` methods from outside the class's defining namespace, the class and its methods must be globally accessible.

---

**Q15. How would you test a Batch Apex class?**

**Ans:** Use `Test.startTest()` and `Test.stopTest()` to force async execution synchronously in tests. Insert test data before `Test.startTest()`. Execute the batch inside `Test.startTest()` with a small batch size (e.g., 1–5 records). After `Test.stopTest()`, all async jobs are synchronously completed. Then assert the expected changes on the records. Use `Database.executeBatch(new MyBatch(), 1)` to test `execute()` isolation. Use `Test.isRunningTest()` inside the batch if you need to skip certain logic during tests.

---

## 9. Quick Reference Cheat Sheet

### Complete Batch Apex Template

```apex
global class MyBatchClass
    implements Database.Batchable<sObject>,
               Database.Stateful,
               Database.AllowsCallouts {

    global Integer successCount = 0;
    global Integer failureCount = 0;

    global Database.QueryLocator start(Database.BatchableContext BC) {
        return Database.getQueryLocator(
            'SELECT Id, Name FROM Account WHERE Phone = null'
        );
    }

    global void execute(Database.BatchableContext BC, List<SObject> scope) {
        List<Account> accs = (List<Account>) scope;

        // Business logic here
        List<Database.SaveResult> results = Database.update(accs, false);

        for (Database.SaveResult sr : results) {
            if (sr.isSuccess()) successCount++;
            else failureCount++;
        }
    }

    global void finish(Database.BatchableContext BC) {
        AsyncApexJob job = [SELECT Status, TotalJobItems,
                                   JobItemsProcessed, NumberOfErrors
                            FROM AsyncApexJob WHERE Id = :BC.getJobId()];

        System.debug('Done. Success: ' + successCount + ' | Failed: ' + failureCount);

        // Database.executeBatch(new NextBatchClass()); // chaining
    }
}

// Execute:
Database.executeBatch(new MyBatchClass(), 200);
```

---

### Key Numbers to Remember

- **50 Million** — Max records via QueryLocator
- **200** — Default batch size
- **2,000** — Max batch size with QueryLocator
- **5** — Max concurrent active batch jobs
- **100** — Max Flex Queue jobs (Holding status)
- **250,000 or licences × 200** — Max async jobs per 24 hours

---
