# Future Apex in Salesforce
### Complete Study Notes + 5-Year Developer Interview Q&A

---

## 1. Asynchronous Apex – Overview

Asynchronous Apex lets Salesforce run code in the background when resources are free — decoupled from the user's transaction. This prevents governor-limit exhaustion and keeps UX fast.

| Type | Best For | Returns Job ID? | Chaining |
|------|----------|:--------------:|:--------:|
| **Future Method** | Callouts, Mixed DML, small sets | ✗ No | ✗ Not supported |
| **Queueable Apex** | Complex jobs, sObject params, chaining | ✓ Yes | ✓ Up to 50 (5 in Dev) |
| **Batch Apex** | Millions of records | ✓ Yes | Via `finish()` → Queueable |
| **Scheduled Apex** | Time-based execution | ✓ Yes | N/A |

---

## 2. Future Method – Deep Dive

### 2.1 What is a Future Method?

A Future Method is a static Apex method annotated with `@future`. It runs **asynchronously** — placed in the Apex Job Queue and executed when server resources are available, **outside the current transaction context**.

### 2.2 When to Use Future Methods

- **Making Callouts from Triggers** — Triggers cannot directly make HTTP callouts; use `@future(callout=true)`.
- **Resolving Mixed DML Errors** — Inserting/updating both Setup Objects (User, Role, Permission Set) and Non-Setup Objects (Account, Lead) in the same transaction throws `MIXED_DML_OPERATION`. Future methods separate the transactions.
- **Long-Running Operations** — Operations that need their own CPU/heap budget.
- **Higher Governor Limits** — Async methods get a fresh set of limits (SOQL: 200, DML: 150, Heap: 12 MB, CPU: 60 s).

### 2.3 Syntax Rules

| # | Rule | Detail |
|---|------|--------|
| 1 | **Annotation** | `@future` (or `@future(callout=true)` for HTTP callouts) |
| 2 | **Access** | Must be `public` or `global` |
| 3 | **Static** | Must be `static` |
| 4 | **Return Type** | Must be `void` — cannot return any value |
| 5 | **Parameters** | Only primitive types: `String`, `Integer`, `Decimal`, `Id`, `Boolean`, `Long`, `Date`, `Datetime`, `List<Id>`, `List<String>`, etc. |
| 6 | **No sObject** | Cannot pass `sObject` or `List<sObject>` as parameter |

### 2.4 Code Structure

```apex
// ✓ CORRECT — No parameters (valid)
@future
public static void myFutureMethod() {
    // business logic here
}

// ✓ CORRECT — Primitive parameters only
@future
public static void processRecords(List<Id> recordIds, String status) {
    List<Account> accs = [SELECT Id, Name FROM Account WHERE Id IN :recordIds];
    for (Account a : accs) { a.Status__c = status; }
    update accs;
}

// ✓ CORRECT — Callout annotation
@future(callout=true)
public static void makeCallout(String endpoint) {
    HttpRequest req = new HttpRequest();
    req.setEndpoint(endpoint);
    req.setMethod('GET');
    Http http = new Http();
    HttpResponse res = http.send(req);
}

// ✗ WRONG — sObject parameter not allowed
@future
public static void badMethod(Account acc) { }  // Compile ERROR

// ✗ WRONG — Non-void return
@future
public static String badReturn() { return 'x'; }  // Compile ERROR
```

---

## 3. Mixed DML Error – Explained

Salesforce raises `MIXED_DML_OPERATION` when you perform DML on both **Setup Objects** and **Non-Setup Objects** within a single transaction.

| Setup Objects *(Metadata-related)* | Non-Setup Objects *(Data-related)* |
|------------------------------------|-------------------------------------|
| User, UserRole, Profile | Account, Contact, Opportunity |
| PermissionSet, PermissionSetAssignment | Lead, Case, Campaign |
| Group, GroupMember | Any Custom Object (e.g., `Hiring_Manager__c`) |
| Layout, WorkflowRule, CustomObject | Contract, Order, Product2 |

### ■ FIX

The fix: Move the DML on the Non-Setup Object into a `@future` method. This separates the two DML operations into different transactions, bypassing the Mixed DML restriction.

```apex
// Triggers MIXED_DML_OPERATION error:
public static void badTransaction() {
    User u = [SELECT Id FROM User WHERE Username='x' LIMIT 1];
    u.IsActive = false;
    update u;                                    // Setup Object DML

    Lead ld = new Lead(LastName='Test', Company='SFDC');
    insert ld;                                   // Non-Setup Object DML ← ERROR
}

// SOLUTION: Separate via @future
public static void goodTransaction() {
    User u = [SELECT Id FROM User WHERE Username='x' LIMIT 1];
    u.IsActive = false;
    update u;                                    // Sync: Setup Object
    createLeadAsync();                           // Async: Non-Setup Object
}

@future()
public static void createLeadAsync() {
    Lead ld = new Lead(LastName='Test', Company='SFDC');
    insert ld;                                   // Runs in separate transaction ✓
}
```

---

## 4. Limitations of Future Methods *(Key for Interviews)*

| # | Limitation | Why / Workaround |
|---|-----------|-----------------|
| 1 | **No sObject parameters** | sObject state may change between call time and execution time. **Workaround:** Pass `List<Id>` and re-query inside the method to get latest data. |
| 2 | **No chaining (Future → Future)** | Cannot call a `@future` from another `@future` — undefined execution order. **Workaround:** Use Queueable Apex for chaining. |
| 3 | **No Job ID returned** | Returns `void`, so you cannot track progress programmatically. **Workaround:** Monitor via Setup → Apex Jobs (Wizard UI). |
| 4 | **Not ideal for bulk records** | Best for small data sets (10–20 records). For bulk, use Batch Apex. |
| 5 | **50 per transaction limit** | Max 50 `@future` calls per single Apex transaction. |
| 6 | **250,000 per 24h org limit** | Org-wide limit (or 200 × number of user licenses, whichever is higher). Shared with Batch, Queueable, Scheduled Apex. |
| 7 | **Cannot call from Batch** | Cannot directly invoke a `@future` method from a Batch class `execute()` or `finish()`. **Workaround:** Call Queueable from Batch `finish()`. |
| 8 | **No getContent / PDF page refs** | `getContent()` and `getContentAsPDFPageReference()` cannot be used inside `@future`. |

---

## 5. Governor Limits Quick Reference

| Limit | Synchronous | Asynchronous (Future/Queueable) |
|-------|:-----------:|:-------------------------------:|
| SOQL Queries | 100 | **200** |
| DML Statements | 150 | 150 |
| DML Rows | 10,000 | 10,000 |
| CPU Time | 10 seconds | **60 seconds** |
| Heap Size | 6 MB | **12 MB** |
| Callouts | 100 | 100 |
| `@future` calls / tx | — | 50 |
| `@future` calls / 24h | — | 250,000 (or 200 × licenses) |

---

## 6. Best Practices

- **Pass IDs, not sObjects.** Always pass `List<Id>` and re-query inside the method — you always get the freshest record state.
- **Design for idempotency.** Async methods can re-run after server failures. Ensure repeated execution produces the same result.
- **Avoid long-running future methods.** Break large work into Batch Apex instead of trying to do it all in one future method.
- **Use Queueable when you need chaining.** If job A must kick off job B, Queueable Apex is the correct tool.
- **Test with `Test.startTest()` / `Test.stopTest()`.** Async methods don't execute in test context unless enclosed in these methods.
- **Add exception handling.** Unhandled exceptions in async methods are hard to surface — log them with a custom logging object or Platform Events.
- **Monitor via Apex Jobs.** Setup → Apex Jobs (or use `AsyncApexJob` SOQL) to check Future method status.
- **Callouts need `callout=true`.** Always annotate `@future(callout=true)` for any method making an HTTP/REST/SOAP callout.
- **Bulkification matters.** Never call `@future` inside a loop in a trigger — you'll hit the 50-per-transaction limit instantly.

---

## 7. Interview Q&A — 5-Year Salesforce Developer Level

> **INTERVIEW GUIDE**
> These questions reflect real interview scenarios for a developer with ~5 years of Salesforce experience. Interviewers expect not just the "what" but the "why", trade-offs, and workarounds.

---

### Part A — Core Concepts

---

**Q1. What is a Future Method? When would you use it over synchronous Apex?**

**Ans:** A Future Method is a static Apex method annotated with `@future` that executes asynchronously in the background when server resources are free. I use it when:
1. I need to make a callout from a trigger — triggers cannot call callouts synchronously.
2. I need to resolve a Mixed DML Operation — updating a Setup Object and a Non-Setup Object in the same transaction throws `MIXED_DML_OPERATION`.
3. A long-running operation needs a higher CPU/heap budget than what synchronous limits allow.

> **💡 5-YR TIP:** Mention the specific error name `MIXED_DML_OPERATION` — interviewers notice this level of precision.

---

**Q2. Why can't you pass an sObject or List as a parameter to a Future Method?**

**Ans:** Salesforce explicitly documented this: because future methods execute at an indeterminate time in the future, the sObject's state may have changed by the time the method runs — leading to stale data overwrites and potential data loss. The workaround is to pass a `List<Id>` instead, then re-query the records at the start of the future method to always work with the freshest data.

> **💡 5-YR TIP:** Always follow up with the workaround — just saying "it's a limitation" is not a 5-year answer.

---

**Q3. What is Mixed DML Operation? How does `@future` resolve it? Give a real scenario.**

**Ans:** `MIXED_DML_OPERATION` is thrown when your Apex code performs DML on both Setup Objects (e.g., `User`, `UserRole`, `Profile`, `PermissionSetAssignment`) and Non-Setup Objects (e.g., `Account`, `Lead`, Custom Objects) within a single transaction. **Real scenario:** In a community user provisioning flow, when a new user (Setup Object) is created with a Role and simultaneously a related Contact (Non-Setup) is updated — this fails. The fix: perform the Setup Object DML synchronously, then call a `@future` method to handle the Non-Setup Object DML in a separate transaction.

> **💡 5-YR TIP:** Give a real-world scenario like user provisioning, not just a textbook answer.

---

**Q4. What are all the limitations of Future Methods? *(Classic interview question)***

**Ans:**
1. Cannot pass `sObject` or `List<sObject>` as parameters — use `List<Id>` workaround.
2. Cannot chain — a `@future` method cannot call another `@future` method; use Queueable for chaining.
3. No Job ID returned — cannot track execution programmatically; monitor via Apex Jobs UI.
4. Not suitable for bulk records — best for 10-20 records; use Batch Apex for large datasets.
5. Max 50 `@future` calls per transaction.
6. Max 250,000 calls per 24 hours org-wide.
7. Cannot be called directly from Batch Apex.
8. Cannot use `getContent()` or `getContentAsPDFPageReference()` inside a future method.
9. Incompatible with Visualforce controller getter/setter methods.

> **💡 5-YR TIP:** Memorise all 9 points — this is frequently asked as "list all limitations".

---

**Q5. Can you call a Future method from another Future method? What happens?**

**Ans:** No. Salesforce prevents this at runtime — you will not get a compile-time error, but at execution time you get:
```
System.AsyncException: Future method cannot be called from a future or batch method.
```
The reason: since neither future method's execution time is guaranteed, chaining them creates unpredictable dependency chains and resource contention. The correct solution is **Queueable Apex**, which supports controlled chaining (up to 50 jobs in production, 5 in Developer Edition).

---

### Part B — Advanced Scenarios

---

**Q6. Can a Future method be called from a Batch class? What is the recommended approach?**

**Ans:** No, you cannot directly call a `@future` method from a Batch class `execute()` or `finish()` method — both are already asynchronous contexts and Salesforce prohibits nesting async calls. **Recommended approach:** From the Batch `finish()` method, enqueue a Queueable job. That Queueable job can then call one `@future` method if absolutely necessary (limit: 1 Queueable job from a future). However, the cleaner design is to keep all async work inside Queueable or another Batch rather than mixing.

> **💡 5-YR TIP:** Knowing the Batch → Queueable → Future chain shows architectural depth.

---

**Q7. Can you call a Queueable Apex from a Future method? What is the limit?**

**Ans:** Yes, but with a restriction: you can enqueue only **1 Queueable job** from within a `@future` method. Attempting to enqueue a second job throws:
```
Too many queueable jobs added to the queue: 2
```
This is Salesforce's way of discouraging improper mixing of async contexts — `@future` is meant for isolated, fire-and-forget operations.

---

**Q8. How would you handle a scenario in a Trigger where you need to make an HTTP callout?**

**Ans:** Triggers cannot make synchronous callouts — Salesforce throws `"Callout from triggers are currently not supported."` **Solution:** Create a `@future(callout=true)` method in a helper class and call it from the trigger. **Example:** In an `AfterInsert` trigger on Account, collect account IDs, pass them to a `@future(callout=true)` method, which then re-queries the accounts, constructs the HTTP request, and processes the response. **Important:** Never call the future method inside a loop — collect all IDs first, pass the full list.

> **💡 5-YR TIP:** The "never call inside a loop" part is critical — shows you understand bulkification.

---

**Q9. How do you test a Future method in Apex test classes?**

**Ans:** Future methods do **not** execute synchronously in test context by default. To force their execution, wrap the method call between `Test.startTest()` and `Test.stopTest()`. The `Test.stopTest()` call synchronously executes all queued async operations. **Example pattern:**

```apex
Test.startTest();
    MyClass.myFutureMethod(new List<Id>{accId});
Test.stopTest();
// Assert results here — future has executed by now
```

Also annotate the test method with `@isTest(SeeAllData=false)` and use `Test.loadData()` or test factories for data.

> **💡 5-YR TIP:** Always mention `Test.startTest/stopTest` — a 5-year developer must know this cold.

---

**Q10. What is the workaround if you need to pass complex data (e.g., a Map of records) to a Future method?**

**Ans:** Since `@future` only accepts primitive types, there are two approaches:

1. **Serialize to JSON:** Convert the complex object to JSON string using `JSON.serialize()`, pass the `String`, then `JSON.deserialize()` inside the future method.
2. **Pass IDs and re-query:** Pass `List<Id>` and rebuild the map/object inside the future method from fresh queries.

**Option 2 is preferred** — it avoids stale data. Option 1 is useful when metadata/config needs to travel with the call and is not stored in the database.

> **💡 5-YR TIP:** Option 2 is best practice; option 1 shows broader knowledge.

---

**Q11. Future method vs Queueable Apex — when would you choose each?**

**Ans:**

**Use `@future` when:**
- The operation is simple, fire-and-forget.
- You need callout from trigger.
- You need to resolve Mixed DML quickly with minimal code.
- Data set is small.

**Use Queueable when:**
- You need to pass sObject types as parameters.
- You need chaining (job A triggers job B).
- You need a Job ID to monitor progress.
- You need to work with complex data structures.
- You want to call it from Batch `finish()`.

**Rule of thumb:** Queueable is a more powerful, monitorable replacement for `@future`. For greenfield development, prefer Queueable unless the use case is a simple callout from trigger.

---

**Q12. How would you debug a Future method that is silently failing?**

**Ans:** Since `@future` runs outside the user transaction, exceptions are not surfaced to the calling code. **Debugging steps:**

1. Wrap the entire future method body in a `try-catch` and insert an `Error_Log__c` record (custom object) with the exception details.
2. Check **Setup → Apex Jobs** — failed future jobs show in the UI with error messages.
3. Query `AsyncApexJob`:
```apex
SELECT Status, ExtendedStatus FROM AsyncApexJob
WHERE JobType='Future'
ORDER BY CreatedDate DESC LIMIT 10
```
4. Use **Salesforce Debug Logs** — enable a trace flag for the user/class and reproduce.
5. Consider using **Platform Events** to emit errors that can be subscribed to by monitoring flows.

> **💡 5-YR TIP:** Mentioning `AsyncApexJob` SOQL and Platform Events shows senior-level thinking.

---

### Part C — Scenario / Design Questions

---

**Q13. A trigger is calling `@future` inside a `for` loop. What is wrong and how do you fix it?**

**Ans:** This is a classic **anti-pattern**. The problem: if the trigger fires on 100 records, you invoke 100 future calls in a single transaction, instantly hitting the 50-per-transaction governor limit and throwing a `LimitException`.

**Fix:** Bulkify the future call. Collect all record IDs **outside** the loop, then call the future method **once** with the full `List<Id>`.

```apex
// ✗ BAD — Future called inside loop
for (Account a : Trigger.new) {
    MyClass.futureMethod(a.Id);
}

// ✓ GOOD — Bulkified, called once
Set<Id> ids = Trigger.newMap.keySet();
MyClass.futureMethod(new List<Id>(ids));
```

> **💡 5-YR TIP:** This is one of the most common Salesforce interview gotchas — know it perfectly.

---

**Q14. Design a solution: On Account insert, sync data to an external ERP system via REST API.**

**Ans:** Design:

1. **Trigger (AfterInsert)** on Account — collect all new Account IDs: `List<Id> accIds = Trigger.newMap.keySet()`.
2. Call `@future(callout=true)` method **once**, passing `accIds`.
3. **Inside the future method:** re-query accounts for required fields, build HTTP POST request for each, send to ERP endpoint, handle response codes.
4. **For error handling:** log failures to an `Integration_Log__c` object.
5. **For retries:** consider Queueable with re-enqueue logic, or a Platform Event + subscriber flow.

**Key considerations:** Named Credentials for endpoint auth, bulkification, idempotency (check if record already synced via external ID).

---

**Q15. Why does Salesforce not allow sObjects in Future method parameters? What data loss risk exists?**

**Ans:** **Official Salesforce reasoning:** The `@future` method is queued and executes at an unknown future time. If you pass an sObject at time T, but the method runs at time T+5min, another user might have updated that record. If your future method then DMLs the stale sObject, it **overwrites the more recent changes — silent data loss**. By forcing developers to pass only IDs and re-query, Salesforce ensures the method always operates on current data. This is also why the `JSON.serialize()` workaround, while technically possible, carries the same stale-data risk and should be used only for config/metadata, not live record data.

> **💡 5-YR TIP:** Explaining the data loss risk shows you truly understand the design, not just memorised the rule.

---

## 8. Quick Cheat Sheet

| Item | Value |
|------|-------|
| **Annotation** | `@future` / `@future(callout=true)` |
| **Access** | `public` or `global` |
| **Return type** | `void` only |
| **Allowed params** | Primitive, `String`, `Id`, `List<Id>`, `List<String>`, etc. |
| **Blocked params** | `sObject`, `List<sObject>`, `Map<String,sObject>` |
| **Chaining** | NOT allowed (Future → Future) |
| **Job ID** | NOT returned |
| **Callout flag** | `@future(callout=true)` required for HTTP |
| **Limit / tx** | 50 calls per transaction |
| **Limit / 24h** | 250,000 (or 200 × licenses, whichever is higher) |
| **Test approach** | `Test.startTest()` … call … `Test.stopTest()` |
| **Monitor** | Setup → Apex Jobs OR `SELECT FROM AsyncApexJob` |
| **sObject workaround** | Pass `List<Id>`, re-query inside method |
| **Chaining workaround** | Queueable Apex |
| **Bulk workaround** | Batch Apex |

---

> **⏭️ NEXT UP**
> Next topic in Async Apex series: **Queueable Apex** — which solves chaining, sObject params, and monitoring limitations of `@future`.