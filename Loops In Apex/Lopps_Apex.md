# Loops in Apex
### For Loop · While Loop · Do-While Loop · SOQL For Loop
#### Comprehensive Notes + 5-Year Experience Interview Q&A (25 Questions)

---

## PART 1 — LOOPS: OVERVIEW & TYPES

### What Are Loops?

A loop is a **control flow statement** that repeatedly executes a block of code as long as a specified condition remains true. In Salesforce Apex, loops are essential for:

- Processing large volumes of records returned from SOQL queries
- Building collections (Lists, Maps) before performing a single DML operation
- Iterating over `Trigger.new` / `Trigger.old` to apply business logic per record
- Automating repetitive computations without writing redundant code

> ⚠️ **Governor Limit awareness:** NEVER place SOQL queries or DML statements inside a loop.
> Apex enforces **100 SOQL queries** and **150 DML operations** per synchronous transaction.
> A loop that runs 200 times with a SOQL inside = 200 SOQL calls = immediate `LimitException`.

---

### ▌ Loop Types in Salesforce Apex

| Loop Type | Syntax Style | Guaranteed First Run? | Best For |
|-----------|-------------|:--------------------:|---------|
| **Traditional For** | `for(init; condition; increment)` | Only if condition is true | Index-based iteration, numeric ranges |
| **Enhanced For (For-Each)** | `for(Type var : collection)` | Only if collection non-empty | Iterating Lists, Sets — clean syntax |
| **SOQL For** | `for(sObj : [SOQL])` | Only if query returns rows | Large datasets — avoids heap limit |
| **While** | `while(condition)` | Only if condition is true | Unknown iteration count, polling logic |
| **Do-While** | `do{} while(condition)` | **YES — always runs once** | When first execution must be guaranteed |

---

## PART 2 — FOR LOOP (All 3 Variants)

Apex supports **three distinct For loop variants**, each suited for different scenarios. Choosing the correct variant is a key indicator of developer maturity.

---

### ▌ Variant 1: Traditional For Loop

```
Syntax: for (init_statement; exit_condition; increment_statement) { ... }
```

- Used when you know the exact number of iterations in advance
- The `i++` is shorthand for `i = i + 1`
- **Performance tip:** Cache `list.size()` in a variable BEFORE the loop — avoids calling `size()` on every iteration

```apex
// Generic syntax
for (Integer i = 0; i < exitCondition; i++) {
    // code block
}

// Fastest pattern — cache size() outside the loop
List<String> fruits = new List<String>{'Apple', 'Banana', 'Mango'};
Integer length = fruits.size();  // cached — not recalculated each iteration

for (Integer i = 0; i < length; i++) {
    String fruitName = fruits.get(i);
    System.debug('Fruit: ' + fruitName);
}

// Iteration trace:
// i=0 → 0<3=true  → fruits.get(0) = Apple
// i=1 → 1<3=true  → fruits.get(1) = Banana
// i=2 → 2<3=true  → fruits.get(2) = Mango
// i=3 → 3<3=false → loop exits
```

> 💡 **Why cache `size()`?** Calling `list.size()` inside the loop condition recalculates on every iteration. Storing it in an `Integer` variable before the loop is a minor performance optimization for large lists.
>
> **Interview Tip:** Interviewers may ask *"what is the fastest for loop pattern?"* — this is the answer.

#### Traditional For with Reverse Iteration

```apex
// Iterating backwards — useful when removing elements while iterating
List<Integer> nums = new List<Integer>{10, 20, 30, 40, 50};

for (Integer i = nums.size() - 1; i >= 0; i--) {
    System.debug('Value at index ' + i + ': ' + nums.get(i));
    if (nums.get(i) == 30) {
        nums.remove(i);  // Safe to remove when iterating backwards
    }
}
```

---

### ▌ Variant 2: Enhanced For Loop (List / Set Iteration)

```
Syntax: for (DataType variable : listOrSet) { ... }
```

- Also called the **'for-each'** loop — cleaner, no index management needed
- Works on any `Iterable`: List, Set, or SOQL results
- **Cannot** access the current index — use Traditional For if you need the index
- **Cannot** modify the collection size (add/remove) while iterating — use Traditional For instead

```apex
// Iterating a List<String>
List<String> fruits = new List<String>{'Apple', 'Banana', 'Mango'};
for (String fruitName : fruits) {
    System.debug('Fruit: ' + fruitName);
}
// Output: Apple, Banana, Mango

// Iterating a Set<Integer> — order is NOT guaranteed
Set<Integer> simpleSet = new Set<Integer>{2, 5, 6, 7, 8, 8};  // 8 is deduped
for (Integer n : simpleSet) {
    System.debug('Number: ' + n);
}
// Output may be in any order: 2, 5, 6, 7, 8

// Iterating a Map — via keySet()
Map<String, Integer> employeesMap = new Map<String, Integer>{
    'Acme'       => 238,
    'Acme Inc'   => 24538,
    'United Oil' => 25538
};

Set<String> keySet = employeesMap.keySet();
for (String key : keySet) {
    System.debug('Key: ' + key + ' | Value: ' + employeesMap.get(key));
}
// Iteration #1: key=Acme,       value=238
// Iteration #2: key=Acme Inc,   value=24538
// Iteration #3: key=United Oil, value=25538
```

> 📌 For Map iteration, always use `keySet()` + enhanced for loop — this is the standard Apex pattern. You can also iterate `map.values()` directly if you only need values and not keys.

---

### ▌ Variant 3: SOQL For Loop

```
Syntax A — Single record:    for (Account acc  : [SELECT Id FROM Account]) { ... }
Syntax B — Batch of 200:     for (List<Account> accs : [SELECT Id FROM Account]) { ... }
```

- Processes records in internal batches — avoids loading all records into heap at once
- This is the **ONLY** loop type that helps avoid heap size limit (6MB sync) when querying >10k records
- *Source: Salesforce Apex Developer Guide — `langCon_apex_loops_for_SOQL`*

```apex
// SOQL For Loop — Single sObject per iteration
for (Account acc : [SELECT Id, Name, Industry FROM Account]) {
    System.debug('Account: ' + acc.Name);
}

// SOQL For Loop — List of sObjects (batch of 200) — preferred for DML
List<Account> toUpdate = new List<Account>();

for (List<Account> accBatch : [SELECT Id, Name, Industry FROM Account WHERE Industry = null]) {
    for (Account acc : accBatch) {
        acc.Industry = 'Technology';
        toUpdate.add(acc);
    }
}

if (!toUpdate.isEmpty()) {
    update toUpdate;  // Single DML after loop
}
```

| Aspect | SOQL in For Loop *(BAD)* | SOQL For Loop *(GOOD)* |
|--------|:------------------------:|:----------------------:|
| **Heap Usage** | All records loaded at once | Internally chunked — avoids heap overflow |
| **Query Count** | 1 query but all rows in memory | 1 query, rows streamed in batches |
| **Governor Risk** | Heap limit exceeded (6MB) | Safe for 50,000+ records |
| **DML inside?** | NEVER — `LimitException` at 151 | Collect to List, DML after loop |
| **Syntax** | `for(Acc a : [SELECT...])` inside outer | `for(Acc a : [SELECT...])` as the loop itself |

> ❌ **CRITICAL** — Never put a SOQL query inside a regular for loop:
> ```apex
> for (Account a : accList) {
>     List<Contact> c = [SELECT Id FROM Contact WHERE AccountId = :a.Id];
> }
> // This fires one SOQL per Account — 200 accounts = 200 SOQL = LimitException!
> ```
> ✅ **Correct approach:** Pre-load a `Map<Id, List<Contact>>` BEFORE the loop using a single SOQL.

---

## PART 3 — WHILE LOOP

The while loop evaluates its condition **before** executing the body. If the condition is false from the start, the body **never executes**.

```
Syntax: while (condition) { // code block }
```

- Condition is a Boolean expression evaluated **BEFORE** each iteration
- Use when the number of iterations is unknown at loop-start time
- **DANGER:** Forgetting the increment (`i++`) causes an infinite loop — Apex will time out

```apex
// Basic while loop — iterates through a List
List<String> fruits = new List<String>{'Apple', 'Banana', 'Mango'};
Integer i = 0;

while (i < fruits.size()) {
    System.debug(fruits.get(i));
    i++;  // CRITICAL — without this, infinite loop!
}
// Output: Apple, Banana, Mango

// Condition is false from start — body NEVER runs
Integer i2 = 0;
while (i2 < 0) {  // 0 < 0 = false immediately
    System.debug('Never printed');
    i2++;
}
System.debug('Loop skipped entirely');
```

### ▌ Real-World While Loop Pattern

```apex
// Polling / retry pattern with while loop
Integer retryCount = 0;
Integer maxRetries = 3;
Boolean success    = false;

while (!success && retryCount < maxRetries) {
    try {
        // attempt callout or operation
        success = true;
    } catch (Exception e) {
        retryCount++;
        System.debug('Retry attempt: ' + retryCount);
    }
}
```

### ▌ When to Use While Loop

- When number of iterations is not known in advance
- Retry / polling logic where you loop until success
- Processing a queue until it's empty: `while(!queue.isEmpty())`
- Reading paginated data where next-page availability is the condition

---

## PART 4 — DO-WHILE LOOP

The do-while loop evaluates its condition **AFTER** executing the body — guaranteeing the body executes **at least once**, regardless of the condition.

```
Syntax: do { // code block } while (condition);
```

- The **semicolon** after `while(condition)` is MANDATORY — a common syntax mistake
- Body runs at least once — condition checked **AFTER** execution
- Key difference from while: do-while always executes first, while may never execute

```apex
// do-while always runs once, even when condition is false from start
List<String> fruits = new List<String>{'Apple', 'Banana', 'Mango'};
Integer i = 0;

// While loop — body SKIPPED because 0 < 0 is false
while (i < 0) {
    System.debug('This never prints — while');
    i++;
}

// Do-While loop — body RUNS ONCE even though 0 < 0 is false
do {
    System.debug(fruits.get(i));  // Prints: Apple
    i++;
} while (i < 0);  // Checked AFTER — 1<0=false, stops

// Standard do-while iterating a list
Integer j = 0;
do {
    System.debug('Fruit: ' + fruits.get(j));
    j++;
} while (j < fruits.size());
// Output: Apple, Banana, Mango
```

### ▌ While vs Do-While — Side-by-Side

| Feature | While Loop | Do-While Loop |
|---------|:----------:|:-------------:|
| **Condition checked** | BEFORE body executes | AFTER body executes |
| **Min executions** | 0 (body may never run) | 1 (always runs at least once) |
| **Syntax ending** | Closing `}` | `} while(condition);` ← note semicolon |
| **Use when** | Condition may be false initially | Must execute body at least once |
| **Common Use** | Processing lists, queues | First-time initialization, menus |
| **Risk if no increment** | Infinite loop | Infinite loop |

---

## PART 5 — BREAK & CONTINUE

> These control keywords are **frequently tested in interviews** and used in real Apex code.
> *Source: Salesforce Apex Developer Guide*

### ▌ `break` — Exit the Loop Immediately

- `break` exits the innermost loop entirely — no further iterations
- Works in: `for`, `for-each`, `while`, `do-while`, and SOQL for loops
- In SOQL for loop with `List<sObject>` batch format, `break` exits the entire loop

```apex
// break — stop as soon as we find the target
List<String> fruits = new List<String>{'Apple', 'Banana', 'Mango', 'Guava'};
String target = 'Mango';

for (String fruit : fruits) {
    if (fruit == target) {
        System.debug('Found: ' + fruit);
        break;  // Exit immediately — Guava is never processed
    }
    System.debug('Checking: ' + fruit);
}
// Output: Checking: Apple, Checking: Banana, Found: Mango
```

---

### ▌ `continue` — Skip Current Iteration, Continue Loop

- `continue` skips the rest of the current iteration and moves to the NEXT one
- In SOQL for loop with `List<sObject>` batch format, `continue` skips to the **next batch of 200**
- Works in: `for`, `for-each`, `while`, `do-while`, and SOQL for loops

```apex
// continue — skip null/invalid values
List<String> names = new List<String>{'Alice', null, 'Bob', '', 'Carol'};

for (String name : names) {
    if (String.isBlank(name)) {
        continue;  // Skip blank/null entries
    }
    System.debug('Processing: ' + name);
}
// Output: Processing: Alice, Processing: Bob, Processing: Carol

// In SOQL for loop (List batch format), continue skips to next 200-record batch
for (List<Account> accBatch : [SELECT Id, Name FROM Account]) {
    if (accBatch.isEmpty()) continue;  // Skip empty batches
    // process batch
}
```

---

## PART 6 — GOVERNOR LIMITS & LOOP BEST PRACTICES

Salesforce enforces **hard runtime limits** in a multi-tenant environment. Violations throw a `System.LimitException` that **cannot be caught**. Loops are the **#1 place** where limits are accidentally exceeded.

### Governor Limits Related to Loops

| Governor Limit | Sync Limit | Async Limit | What Triggers It in a Loop |
|---------------|:----------:|:-----------:|---------------------------|
| **SOQL Queries** | 100 | 200 | SOQL inside a loop body |
| **SOQL Query Rows** | 50,000 | 50,000 | Large queries — use SOQL for loop |
| **DML Statements** | 150 | 150 | DML inside a loop body |
| **DML Rows** | 10,000 | 10,000 | DML on too many records |
| **Heap Size** | 6 MB | 12 MB | Loading all query results at once |
| **CPU Time** | 10,000 ms | 60,000 ms | Nested loops over large datasets |
| **Callouts** | 100 | 100 | HTTP callout inside a loop |

---

### ▌ The Golden Rules of Loops in Apex

> **RULE 1** — No SOQL inside a loop: Use `Map<Id, sObject>` pre-loaded BEFORE the loop.
>
> **RULE 2** — No DML inside a loop: Collect records in a List, then DML once AFTER the loop.
>
> **RULE 3** — No callouts inside a loop: Callouts inside loops are a design smell — use Queueable.
>
> **RULE 4** — Cache `list.size()` outside the loop condition for large list iterations.
>
> **RULE 5** — Use SOQL For Loop when querying potentially large record sets (>10k rows).

---

### ▌ Anti-Pattern vs Best Practice

#### ✗ BAD — SOQL and DML Inside Loop

```apex
// WRONG — will hit governor limits immediately
for (Account acc : [SELECT Id FROM Account]) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id];  // SOQL in loop!
    for (Contact c : contacts) {
        c.Description = 'Updated';
        update c;  // DML in loop!
    }
}
```

#### ✓ GOOD — Collections + Single DML

```apex
// CORRECT — bulkified pattern

// Step 1: One SOQL to get accounts
List<Account> accounts  = [SELECT Id FROM Account];
Set<Id> accountIds      = new Map<Id,Account>(accounts).keySet();

// Step 2: One SOQL to get all related contacts
Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();
for (Contact c : [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds]) {
    if (!contactsByAccount.containsKey(c.AccountId)) {
        contactsByAccount.put(c.AccountId, new List<Contact>());
    }
    contactsByAccount.get(c.AccountId).add(c);
}

// Step 3: Prepare updates in a loop — NO SOQL, NO DML
List<Contact> toUpdate = new List<Contact>();
for (Account acc : accounts) {
    List<Contact> relatedContacts = contactsByAccount.get(acc.Id);
    if (relatedContacts != null) {
        for (Contact c : relatedContacts) {
            c.Description = 'Updated';
            toUpdate.add(c);
        }
    }
}

// Step 4: ONE DML after loop
if (!toUpdate.isEmpty()) {
    update toUpdate;
}
```

---

## PART 7 — QUICK REFERENCE CHEAT SHEET

### Loop Comparison — Full Reference

| Loop | Syntax | Runs 0 times? | Use Case | Avoid When |
|------|--------|:------------:|---------|-----------|
| **Traditional For** | `for(init;cond;incr){}` | Yes | Known iterations, index access | Iterating collections without index |
| **Enhanced For** | `for(T v : collection){}` | Yes (empty coll) | Lists, Sets, clean iteration | You need to modify collection size |
| **SOQL For** | `for(sObj:[SOQL]){}` | Yes (0 results) | Large SOQL result sets | Small datasets (overhead not worth it) |
| **While** | `while(cond){}` | Yes | Unknown iteration count | You always need at least one execution |
| **Do-While** | `do{}while(cond);` | **NO — runs once** | Guaranteed first execution | Condition is always true (infinite) |

---

### ▌ When to Use Which Loop — Decision Guide

| Scenario | Recommended Loop | Reason |
|----------|:----------------:|--------|
| Iterate `Trigger.new` records | Enhanced For | Clean, no index needed, auto-handles all records |
| Process SOQL result >50k rows | SOQL For (List batch) | Prevents heap limit — processes in 200-record batches |
| Build a Map from a List | Enhanced For | Clean key-value mapping without index |
| Remove items while iterating | Traditional For (reverse) | Reverse iteration safe for removal — `list.remove(i)` |
| Retry logic with unknown count | While | Loop continues until success condition met |
| Menu prompt / initialization | Do-While | Guarantees at least one execution before checking |
| Numeric countdown | Traditional For (reverse) | `i--` countdown pattern |
| Iterate Map by key | Enhanced For on `keySet()` | Standard Apex Map iteration pattern |

---

## PART 8 — INTERVIEW QUESTIONS: 5-YEAR EXPERIENCE LEVEL

> **INTERVIEW GUIDE**
> The following 25 questions are crafted for candidates with **4–6 years** of Salesforce experience. Expect scenario-based, governor-limit, and code-review style questions.

---

### ▌ Section A — Core Loop Fundamentals

---

**Q1. How many types of loops does Apex support? Explain each briefly.**

**Ans:** Apex supports **5 loop constructs** — 3 variants of for, plus while and do-while:

- **Traditional For:** `for(Integer i=0; i<n; i++)` — best when iteration count is known and index access is needed.
- **Enhanced For (For-Each):** `for(String s : list)` — cleanest way to iterate Lists and Sets.
- **SOQL For:** `for(Account a : [SELECT Id FROM Account])` — processes SOQL results in internal batches to avoid heap limits.
- **While:** `while(condition){}` — runs as long as condition is true, may run 0 times.
- **Do-While:** `do{} while(condition);` — always runs at least once; condition checked after body.

---

**Q2. What is the key difference between while and do-while? Give a real-world use case for do-while.**

**Ans:** While evaluates the condition **BEFORE** the body — body may never execute if condition is initially false. Do-while evaluates **AFTER** — guarantees the body executes at least once. Real use case: displaying a menu and asking for input — the menu must always appear once before checking if the user wants to continue.

---

**Q3. What happens if you access `list.get(list.size())` in Apex?**

**Ans:** It throws `System.ListException: List index out of bounds` at runtime. The valid last index is always `size()-1`. This is a common bug when the exit condition is written as `i <= length` instead of `i < length`.

---

**Q4. Can you iterate a Set using a traditional for loop with an index?**

**Ans:** No. Sets are unordered and do not support index-based access. You cannot call `set.get(0)`. You must use the enhanced for loop: `for(String s : mySet)`. If you need indexed access, convert the Set to a List first using `List<String> l = new List<String>(mySet);` and then iterate with index.

---

**Q5. What does the enhanced for loop do when iterating a Map?**

**Ans:** You **cannot** directly iterate a Map — you must iterate its `keySet()` (returns `Set<KeyType>`) or `values()` (returns `List<ValueType>`). Pattern:

```apex
for (String key : myMap.keySet()) {
    String val = myMap.get(key);
}
```

This is the standard and most common Apex Map iteration pattern.

---

### ▌ Section B — SOQL For Loop (High-Value Questions)

---

**Q6. What is the advantage of using a SOQL for loop over assigning SOQL results to a List?**

**Ans:** A standard `List<Account> accs = [SELECT...]` loads all records into heap memory at once — if records exceed ~50,000 or heap hits 6MB, you get a `LimitException`. A SOQL for loop `for(Account a : [SELECT...])` processes records in internal batches, **streaming** data without loading everything at once. This makes it the preferred approach for large datasets.

---

**Q7. What is the difference between the single-sObject and List-of-sObject formats of a SOQL for loop?**

**Ans:** Single format: `for(Account a : [SELECT...])` — executes the body once per record individually. List format: `for(List<Account> batch : [SELECT...])` — executes the body once per 200-record batch. The batch format is preferred when you are performing DML, as you can collect the batch and DML once per batch rather than after the entire loop.

```apex
// Single sObject — one per iteration
for (Account a : [SELECT Id, Name FROM Account]) {
    System.debug(a.Name);
}

// List<sObject> batch — 200 per iteration
List<Account> toUpdate = new List<Account>();
for (List<Account> batch : [SELECT Id FROM Account WHERE Industry = null]) {
    for (Account a : batch) { a.Industry = 'Tech'; toUpdate.add(a); }
}
update toUpdate;
```

---

**Q8. Can you use `break` and `continue` inside a SOQL for loop?**

**Ans:** Yes. In the single sObject format, `break` and `continue` behave as in any other for loop — `break` exits the loop, `continue` skips to the next record. In the `List<sObject>` batch format, `continue` skips to the **next 200-record batch**, and `break` exits the entire loop. This is documented in the official Salesforce Apex Developer Guide.

---

**Q9. What is the governor limit for SOQL query rows? How does a SOQL for loop help?**

**Ans:** The limit is **50,000 rows** per synchronous transaction. A standard assignment (`List<Account> accs = [SELECT...]`) loads all rows into heap. A SOQL for loop streams them in batches, preventing heap overflow — but the total row count processed still counts toward the 50,000 row limit. For datasets beyond 50,000 rows, you need **Batch Apex** (`Database.Batchable`).

---

### ▌ Section C — Governor Limits & Bulkification

---

**Q10. What happens if you put a SOQL query inside a for loop? What is the fix?**

**Ans:** Apex allows up to 100 SOQL queries per synchronous transaction. If a for loop runs 101+ times with a SOQL inside, a `System.LimitException: Too many SOQL queries: 101` is thrown. Fix: Move the SOQL outside the loop and use a `Map<Id, sObject>` for O(1) lookup inside the loop.

```apex
// BAD: SOQL inside loop
for (Account a : accountList) {
    List<Contact> c = [SELECT Id FROM Contact WHERE AccountId = :a.Id];  // DANGER
}

// GOOD: Pre-load Map before loop
Map<Id, List<Contact>> contactMap = new Map<Id, List<Contact>>();
for (Contact c : [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds]) {
    if (!contactMap.containsKey(c.AccountId))
        contactMap.put(c.AccountId, new List<Contact>());
    contactMap.get(c.AccountId).add(c);
}

for (Account a : accountList) {
    List<Contact> related = contactMap.get(a.Id);  // O(1) — no SOQL
}
```

---

**Q11. What happens if you put a DML statement inside a for loop?**

**Ans:** DML operations are limited to **150 per synchronous transaction**. A loop with `insert`/`update`/`delete` inside will hit `LimitException: Too many DML statements: 151` after the 151st iteration. Always collect records in a List inside the loop and perform a **single DML after the loop** exits.

---

**Q12. How does caching `list.size()` before a traditional for loop improve performance?**

**Ans:** When using `for(Integer i=0; i<list.size(); i++)`, Apex calls `list.size()` on every iteration — this is a method call overhead. By caching: `Integer len = list.size(); for(Integer i=0; i<len; i++)`, the size is calculated only once. While the difference is minor for small lists, it's a best practice indicator that interviewers look for, especially in large-volume trigger contexts.

---

**Q13. In a Trigger, you iterate `Trigger.new`. What loop type do you use and why?**

**Ans:** **Enhanced For:** `for(Account acc : Trigger.new)`. `Trigger.new` is a `List<sObject>`, so the for-each loop is the cleanest approach. You do NOT use a SOQL for loop here — `Trigger.new` is already in memory. You do NOT use Traditional For unless you need the index (rare). Inside the loop, you read from a pre-loaded Map using `Trigger.oldMap` for comparison.

---

### ▌ Section D — Scenario & Code Review Questions

---

**Q14. Review this code. What is wrong? How would you fix it?**

```apex
trigger AccountTrigger on Account (after insert) {
    for (Account acc : Trigger.new) {
        List<Contact> contacts = [SELECT Id, Email FROM Contact
                                   WHERE AccountId = :acc.Id];
        for (Contact c : contacts) {
            c.Description = 'Linked';
            update c;
        }
    }
}
```

**Ans:** Two critical issues: (1) **SOQL inside the outer loop** — fires one query per account, hits 101 SOQL limit. (2) **DML inside inner loop** — fires one update per contact, hits 151 DML limit. Fix: Use one SOQL to get all contacts using `WHERE AccountId IN :accountIds`, build a Map, loop without SOQL/DML, collect updates in a List, and call `update` once after the loop.

---

**Q15. What is the output of this code?**

```apex
Integer i = 0;
do {
    System.debug('Value: ' + i);
    i++;
} while (i < 0);
```

**Ans:** Output: `Value: 0`. The do-while body executes once (printing `i=0`), then increments `i` to 1. The condition (`1<0`) is false, so the loop exits. This demonstrates the **guaranteed-one-execution** behavior of do-while.

---

**Q16. When iterating a List to remove elements, which loop type is safest? Why?**

**Ans:** **Traditional For loop iterating BACKWARDS** (from `size()-1` down to `0`). When you remove an element at index `i`, all elements after it shift left by one. Iterating forward after removal causes elements to be skipped. Reverse iteration avoids this issue since removing index `i` doesn't affect indices `0` to `i-1`, which haven't been processed yet.

```apex
List<Integer> nums = new List<Integer>{1,2,3,4,5};
for (Integer i = nums.size()-1; i >= 0; i--) {
    if (nums.get(i) % 2 == 0) {
        nums.remove(i);  // safe — iterating backwards
    }
}
System.debug(nums);  // (1, 3, 5)
```

---

**Q17. What is the output of this code?**

```apex
List<Integer> nums = new List<Integer>{10,20,30,40,50};
for (Integer i = 0; i < nums.size(); i++) {
    if (nums.get(i) == 30) {
        System.debug('Found 30');
        break;
    }
    System.debug('Processed: ' + nums.get(i));
}
```

**Ans:** Output: `Processed: 10`, `Processed: 20`, `Found 30`. When `i=2`, `nums.get(2)=30` matches, so we debug `'Found 30'` and `break`. The values 40 and 50 are never processed.

---

**Q18. What is the difference between `break` in a nested loop vs a single loop?**

**Ans:** `break` only exits the **INNERMOST** loop it is placed in. If you have a nested loop (for inside a for), `break` inside the inner loop exits the inner loop but the outer loop continues. To exit both loops you need a flag variable: `Boolean breakOuter = false;` then set it and check it in the outer loop condition.

---

### ▌ Section E — Advanced & Architecture Level

---

**Q19. How do you iterate a `Map<String, List<String>>` and print all key-value pairs?**

```apex
Map<String, List<String>> roleMap = new Map<String, List<String>>{
    'Developer' => new List<String>{'Platform Developer I', 'Platform Developer II'},
    'Admin'     => new List<String>{'Administrator', 'Advanced Administrator'}
};

for (String role : roleMap.keySet()) {
    List<String> certs = roleMap.get(role);
    System.debug('Role: ' + role);
    for (String cert : certs) {
        System.debug('  Certification: ' + cert);
    }
}
```

**Ans:** Use nested enhanced for loops: outer iterates `keySet()`, inner iterates the `List<String>` value. This is the standard pattern for `Map<String, List<T>>` iteration.

---

**Q20. You need to process 500,000 Account records. Which Apex mechanism and loop type would you use?**

**Ans:** Use **Batch Apex** (`Database.Batchable`). The `execute()` method receives chunks of up to 200 records (configurable via scope). Inside `execute()`, use an enhanced for loop to process each record in the chunk, collect updates in a List, and DML once per batch. Never use a SOQL for loop to process 500k records in a single transaction — governor limits cap rows at **50,000 per transaction**.

---

**Q21. What is the risk of using a `while(true)` loop in Apex? How would you safely use it?**

**Ans:** A `while(true)` loop is an infinite loop — Apex will throw a **CPU Time Limit Exceeded** exception (10,000ms sync) before it truly loops forever. It's generally an anti-pattern in Apex. Safe usage requires a guaranteed exit: always combine with a `break` on a condition, or a maximum counter:

```apex
Integer maxIter = 1000;
while (true && maxIter-- > 0) {
    if (done) break;
}
```

---

**Q22. Explain how `continue` behaves differently in a SOQL for loop vs a regular for loop.**

**Ans:** In a regular for or enhanced for, `continue` skips to the next record/element. In a SOQL for loop using the `List<sObject>` batch format, `continue` skips the rest of the **current 200-record batch** and moves to the NEXT batch — not just the next individual record. This is documented in the Salesforce Apex Developer Guide and is a subtle but important distinction.

---

**Q23. A trigger processes 200 records. You need to group contacts by AccountId and update each contact's description with the Account name. Write efficient Apex.**

```apex
trigger ContactTrigger on Contact (before insert) {

    // Step 1: collect Account Ids
    Set<Id> accountIds = new Set<Id>();
    for (Contact c : Trigger.new) {
        if (c.AccountId != null) accountIds.add(c.AccountId);
    }

    // Step 2: one SOQL — Map<Id, Account>
    Map<Id, Account> accMap = new Map<Id, Account>(
        [SELECT Id, Name FROM Account WHERE Id IN :accountIds]
    );

    // Step 3: loop — no SOQL, no DML
    for (Contact c : Trigger.new) {
        if (accMap.containsKey(c.AccountId)) {
            c.Description = 'Linked to: ' + accMap.get(c.AccountId).Name;
        }
    }

    // Before trigger — no DML needed; Trigger.new is modified in-memory
}
```

**Ans:** Pattern: Set for unique IDs → one SOQL into Map → enhanced for loop on `Trigger.new` using `Map.get()` — **zero SOQL or DML inside any loop**. This is the gold-standard bulkified trigger pattern.

---

**Q24. What is the CPU time governor limit and how can nested loops cause it?**

**Ans:** Salesforce enforces **10,000ms CPU time** per synchronous transaction and **60,000ms** asynchronous. Nested loops with O(n²) or O(n³) complexity on large datasets can exceed this. Example: two nested for loops each over 1,000 records = 1,000,000 iterations. Mitigation: replace inner loops with **Map lookups (O(1))**, use aggregate SOQL instead of looping computations, or move heavy processing to Batch Apex.

---

**Q25. When would you choose a traditional for loop over an enhanced for loop for a List? Give an example.**

**Ans:** Use traditional for when you need: (1) The index of the current element, (2) To modify the list size during iteration (add/remove — must iterate backwards), or (3) To iterate only a subset of elements (e.g., every other element). Enhanced for is preferred for readability when you just need to process each element without caring about its position.

```apex
// Traditional For — needed for index access and safe removal
List<String> items = new List<String>{'A', 'B', 'C', 'D'};

for (Integer i = items.size()-1; i >= 0; i--) {
    if (items.get(i) == 'B' || items.get(i) == 'D') {
        System.debug('Removing at index ' + i + ': ' + items.get(i));
        items.remove(i);
    }
}
System.debug(items);  // (A, C)
```

---

## PART 9 — ASSIGNMENT SOLUTIONS

The following code solves all 5 assignment tasks from the PDF, incorporating Salesforce certifications as the data and demonstrating all loop types.

```apex
// ── TASK 1: List of Salesforce Certifications ──────────────────────────
List<String> certList = new List<String>{
    'Salesforce Administrator',
    'Platform Developer I',
    'Platform Developer II',
    'JavaScript Developer I',
    'System Architect',
    'Application Architect',
    'Integration Architect',
    'B2C Commerce Developer'
};

// ── TASK 2: Set of Developer Certifications ────────────────────────────
Set<String> devCertSet = new Set<String>{
    'Platform Developer I',
    'Platform Developer II',
    'JavaScript Developer I',
    'B2C Commerce Developer',
    'Platform Developer I'   // duplicate — ignored by Set
};

// ── TASK 3: Map<Role, List<Certifications>> ────────────────────────────
Map<String, List<String>> certMap = new Map<String, List<String>>();

certMap.put('Developer', new List<String>{
    'Platform Developer I', 'Platform Developer II',
    'JavaScript Developer I', 'B2C Commerce Developer'
});
certMap.put('Admin', new List<String>{
    'Administrator', 'Advanced Administrator', 'Platform App Builder'
});
certMap.put('Architect', new List<String>{
    'System Architect', 'Application Architect',
    'Integration Architect', 'Data Architect', 'B2C Solution Architect'
});

// ── TASK 4a: For Loop — iterate certList ──────────────────────────────
System.debug('=== For Loop: All Certifications ===');
Integer length = certList.size();
for (Integer i = 0; i < length; i++) {
    System.debug('Cert [' + i + ']: ' + certList.get(i));
}

// ── TASK 4b: Enhanced For Loop — iterate devCertSet ───────────────────
System.debug('=== Enhanced For: Developer Set ===');
for (String cert : devCertSet) {
    System.debug('Dev Cert: ' + cert);
}

// ── TASK 4c: Enhanced For Loop — iterate certMap ──────────────────────
System.debug('=== Enhanced For: Map by Role ===');
for (String role : certMap.keySet()) {
    System.debug('Role: ' + role);
    for (String cert : certMap.get(role)) {
        System.debug('  - ' + cert);
    }
}

// ── TASK 4d: Do-While Loop — iterate certList ─────────────────────────
System.debug('=== Do-While Loop: All Certs ===');
Integer j = 0;
do {
    System.debug('Do-While Cert: ' + certList.get(j));
    j++;
} while (j < certList.size());

// ── TASK 5: All output goes to Debug Log via System.debug() above ──────
```

---
