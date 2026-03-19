# SOQL — Salesforce Object Query Language
### Comprehensive Notes + 5-Year Experience Interview Questions

---

## PART 1 — SOQL FUNDAMENTALS

### 1. What is SOQL?

SOQL (Salesforce Object Query Language) is a **read-only** query language designed specifically for the Salesforce platform. It is used to retrieve data from Salesforce objects (both standard and custom). SOQL is syntactically similar to SQL, but operates within Salesforce multi-tenant cloud environment and supports features like relationship traversal, polymorphic fields, and aggregate functions.

| Feature | SOQL | SQL (Traditional) |
|---------|:----:|:-----------------:|
| **Platform** | Salesforce only | Any relational DB |
| **Purpose** | Query sObject records | Query tables |
| **Joins** | Via relationship traversal (no traditional JOIN) | Full JOIN syntax |
| **UPDATE / DELETE** | Not supported in SOQL — use DML | Supported |
| **Max rows per query** | 50,000 (governor limit) | No platform limit |
| **Aggregate queries** | Yes — `AggregateResult` | Yes |
| **Polymorphic fields** | `TYPEOF` clause | Not applicable |
| **Bind variables** | Apex `:variable` syntax | Parameterised queries |

> 📌 **KEY RULE:** SOQL is **SELECT-only**. You cannot INSERT, UPDATE, or DELETE using SOQL. All data modifications must use DML statements (`insert`, `update`, `upsert`, `delete`, `undelete`, `merge`).

---

### 1.1 Complete SOQL SELECT Syntax *(Official Reference)*

```soql
SELECT fieldList | (subquery) | typeof_expression
FROM objectType
[WHERE conditionExpression]
[WITH SECURITY_ENFORCED | USER_MODE | SYSTEM_MODE | DATA CATEGORY filterExpression]
[GROUP BY fieldGroupByList | ROLLUP(fieldSubtotal) | CUBE(fieldSubtotal)]
[HAVING conditionExpression]
[ORDER BY fieldOrderByList [ASC|DESC] [NULLS FIRST|NULLS LAST]]
[LIMIT numberOfRowsToReturn]
[OFFSET numberOfRowsToSkip]
[FOR VIEW | REFERENCE | UPDATE | TRACKING]
[UPDATE TRACKING | VIEWSTAT]
```

> ⚡ **PDF GAP:** The PDF listed syntax but did not cover `WITH SECURITY_ENFORCED`, `USER_MODE`, `SYSTEM_MODE`, `OFFSET`, `FOR VIEW/REFERENCE/UPDATE/TRACKING`, or `UPDATE TRACKING/VIEWSTAT`. These are critical for secure, production-grade code.

---

### 1.2 WHERE Clause — Filter Operators

| Operator / Pattern | Syntax Example | Notes |
|-------------------|---------------|-------|
| **Equals** | `WHERE Name = 'Acme'` | Exact match — case-insensitive for strings |
| **Not Equals** | `WHERE Status != 'Closed'` | Also valid: `<>` operator |
| **IN (multi-value)** | `WHERE Name IN ('Acme','XYZ')` | Use `:listVar` for dynamic lists |
| **NOT IN** | `WHERE Id NOT IN :excludedIds` | Excludes set of values |
| **LIKE (contains)** | `WHERE Name LIKE '%tech%'` | `%` = any chars, `_` = single char |
| **LIKE (starts with)** | `WHERE Name LIKE 'Acme%'` | Starting characters fixed |
| **LIKE (ends with)** | `WHERE Name LIKE '%Inc'` | Ending characters fixed |
| **NOT LIKE** | `WHERE Name NOT LIKE 'Test%'` | Negates LIKE pattern |
| **Greater / Less** | `WHERE Amount > 10000` | `>`, `<`, `>=`, `<=` |
| **Date filter** | `WHERE CloseDate > 2024-01-01` | Date literals like `LAST_N_DAYS:30` too |
| **Date literal** | `WHERE CloseDate = LAST_N_DAYS:30` | No quotes around literals |
| **NULL check** | `WHERE Email = null` | `IS NULL` is **NOT** valid SOQL syntax |
| **NOT NULL** | `WHERE Email != null` | Checks field has a value |
| **AND / OR** | `WHERE Industry = 'Tech' AND Rating = 'Hot'` | Combine multiple filters |

> ⚡ **PDF GAP:** The PDF did not cover `NOT IN`, date literals (`TODAY`, `LAST_N_DAYS:n`, `THIS_WEEK`, `LAST_MONTH`, etc.), NULL checks (`= null` vs `!= null`), or combining `AND`/`OR` with parentheses for precedence. **Date literals are heavily tested!**

#### Date Literal Examples — commonly used in real code:

```soql
WHERE CloseDate = TODAY
WHERE CloseDate = YESTERDAY
WHERE CloseDate = THIS_WEEK
WHERE CloseDate = LAST_MONTH
WHERE CloseDate = LAST_N_DAYS:30    -- last 30 days
WHERE CloseDate = NEXT_N_DAYS:7     -- next 7 days
WHERE CreatedDate >= LAST_QUARTER
WHERE CreatedDate = THIS_FISCAL_YEAR
```

---

### 1.3 ORDER BY, LIMIT, OFFSET

```apex
// ORDER BY — Sort results
SELECT Id, Name FROM Account ORDER BY Name ASC
SELECT Id, Name FROM Account ORDER BY CreatedDate DESC
SELECT Id, Name FROM Account ORDER BY Name ASC NULLS LAST   // nulls at end
SELECT Id, Name FROM Account ORDER BY Name ASC NULLS FIRST  // nulls first

// LIMIT — Restrict rows returned
SELECT Id, Name FROM Contact LIMIT 10

// OFFSET — Skip rows (pagination)
SELECT Id, Name FROM Account ORDER BY Name LIMIT 20 OFFSET 0   // Page 1
SELECT Id, Name FROM Account ORDER BY Name LIMIT 20 OFFSET 20  // Page 2
SELECT Id, Name FROM Account ORDER BY Name LIMIT 20 OFFSET 40  // Page 3
// OFFSET max = 2000. For >2000 offset, use keyset pagination instead. ⚠️
```

> ⚡ **PDF GAP:** `OFFSET` was not covered in the PDF. `OFFSET` is critical for implementing pagination in LWC/VF pages. Its limit of 2000 and the keyset pagination alternative are common senior interview topics.

---

## PART 2 — AGGREGATE QUERIES & GROUP BY

### 2. Aggregate Functions & GROUP BY

Aggregate queries let you summarize, count, and calculate across records. The return type is always `List<AggregateResult>`. Each `AggregateResult` element's values are accessed via `.get('alias')`.

| Function | Supported Types | Example |
|----------|----------------|---------|
| **`COUNT()`** | Any (counts non-null rows) | `SELECT COUNT() FROM Account` |
| **`COUNT(field)`** | Any field | `SELECT COUNT(Id) FROM Contact` |
| **`SUM(field)`** | Integer, Number, Currency | `SELECT SUM(Amount) FROM Opportunity` |
| **`AVG(field)`** | Integer, Number, Currency | `SELECT AVG(Amount) FROM Opportunity` |
| **`MIN(field)`** | Integer, Number, Currency, Date, DateTime | `SELECT MIN(CloseDate) FROM Opportunity` |
| **`MAX(field)`** | Integer, Number, Currency, Date, DateTime | `SELECT MAX(Amount) FROM Opportunity` |
| **`COUNT_DISTINCT(field)`** | Any | `SELECT COUNT_DISTINCT(Industry) FROM Account` |

---

### 2.1 GROUP BY, HAVING, AggregateResult

```apex
// Basic GROUP BY
SELECT Industry, COUNT(Id) FROM Account GROUP BY Industry

// With alias — required for accessing value in Apex
SELECT Industry, COUNT(Id) cnt, SUM(AnnualRevenue) totalRev
FROM Account
GROUP BY Industry

// HAVING — filter on aggregated results (like WHERE but for aggregates)
SELECT LeadSource, COUNT(Id) total FROM Lead
GROUP BY LeadSource
HAVING COUNT(Id) > 5

// Accessing AggregateResult in Apex:
List<AggregateResult> results = [SELECT Industry, COUNT(Id) cnt FROM Account GROUP BY Industry];
for (AggregateResult ar : results) {
    String  ind = (String)  ar.get('Industry');
    Integer cnt = (Integer) ar.get('cnt');
    System.debug(ind + ' → ' + cnt);
}
```

---

### 2.2 GROUP BY ROLLUP and CUBE

```apex
// ROLLUP — subtotals along a hierarchy
SELECT LeadSource, Rating, COUNT(Id)
FROM Lead
GROUP BY ROLLUP(LeadSource, Rating)
// Returns rows for each LeadSource+Rating combination,
// subtotals per LeadSource, and a grand total row.

// CUBE — all possible subtotal combinations
SELECT Type, BillingCountry, COUNT(Id)
FROM Account
GROUP BY CUBE(Type, BillingCountry)
// Returns every combination: each pair, each individual field, and grand total.

// GROUPING() — identifies which rows are subtotal rows in ROLLUP/CUBE
SELECT LeadSource, GROUPING(LeadSource) isSubtotal, COUNT(Id)
FROM Lead
GROUP BY ROLLUP(LeadSource)
```

> ⚡ **PDF GAP:** `COUNT_DISTINCT()`, `AVG()`, `GROUPING()` function, aliases on aggregate fields, and accessing `AggregateResult` values via `.get()` were all missing from the PDF. These come up in every senior data-access interview.

---

## PART 3 — RELATIONSHIP QUERIES

### 3. Relationship Queries

Salesforce supports two types of relationship queries: **Child-to-Parent** (dot notation to traverse up) and **Parent-to-Child** (subquery using the child relationship name in brackets). These replace SQL JOINs.

---

### 3.1 Child-to-Parent (Dot Notation)

Traverse from a child record **UP** to its parent using the relationship field name.

```apex
// Contact → Account (standard relationship)
SELECT Id, Name, Email, Account.Name, Account.Phone, Account.Industry
FROM Contact

// Opportunity → Account
SELECT Id, Name, Amount, Account.Name, Account.BillingCity
FROM Opportunity

// Case → Account & Contact
SELECT Id, CaseNumber, Account.Name, Contact.Email
FROM Case

// Custom object → Account (uses __r relationship suffix)
SELECT Id, Name, Account__r.Name, Account__r.Industry
FROM Invoice__c

// Multi-level traversal (up to 5 levels)
SELECT Id, Name, Account__r.Name, Account__r.Parent.Name
FROM Invoice__c
// Invoice → Account → Parent Account (3 levels)
```

> ⚡ **PDF GAP:** Custom object relationship traversal uses `__r` (not `__c`). Multi-level traversal (up to 5 levels for standard, 5 for custom) was not covered. Mixing standard and custom relationship hops in the same query is a common interview scenario.

---

### 3.2 Parent-to-Child (Subquery)

Query a parent and retrieve all related child records in a nested list using the child's **relationship name** (NOT the API name).

```apex
// Account → Contacts (Contacts = relationship name, plural)
SELECT Id, Name, (SELECT Id, FirstName, LastName, Email FROM Contacts)
FROM Account

// Account → Contacts, Opportunities, Cases (multiple subqueries)
SELECT Id, Name,
    (SELECT Id, Name, Email FROM Contacts),
    (SELECT Id, Name, Amount FROM Opportunities),
    (SELECT Id, CaseNumber, Status FROM Cases)
FROM Account

// Custom relationship — uses plural of relationship label
SELECT Id, Name, (SELECT Id, Name FROM Invoice__r)
FROM Account__c

// Accessing in Apex:
List<Account> accs = [SELECT Id, Name, (SELECT Id, Email FROM Contacts) FROM Account];
for (Account a : accs) {
    List<Contact> cons = a.Contacts;  // child list
    for (Contact c : cons) {
        System.debug(c.Email);
    }
}
// IMPORTANT: a.Contacts returns null if no children (not an empty list!) ⚠️
// Always null-check before iterating
```

---

### 3.3 Relationship Query Rules & Limits

| Rule | Detail |
|------|--------|
| **Child-to-Parent max levels** | 5 hops for standard, 5 for custom (combined max 5) |
| **Parent-to-Child subqueries** | Max 1 level deep (no nested subqueries within subqueries) |
| **Max subqueries per query** | 20 |
| **Standard child relationship name** | Plural of the child object — `Contacts`, `Opportunities`, `Cases` |
| **Custom child relationship name** | Plural of the relationship label defined in the lookup field |
| **Child list null check** | `a.Contacts` is `null` (not empty list) when no children exist |
| **Subquery row limit** | 200 child rows per parent in a subquery |

---

## PART 4 — POLYMORPHIC QUERIES (TYPEOF)

### 4. Polymorphic Relationships & TYPEOF

A polymorphic relationship field can reference multiple different object types. The most common examples are the "What" and "Who" fields on Activity objects (Task, Event).

| Field | Object | Can Reference |
|-------|--------|--------------|
| **`Who` (WhoId)** | Task, Event | Contact OR Lead (Person objects) |
| **`What` (WhatId)** | Task, Event | Any standard or custom object EXCEPT Lead and Contact |
| **`Owner` (OwnerId)** | Most objects | User OR Group (Queue) |

---

### 4.1 TYPEOF Syntax

```apex
// Full TYPEOF structure
SELECT Id, Subject,
    TYPEOF What
        WHEN Account     THEN Id, Name, Industry, Phone
        WHEN Opportunity THEN Id, Name, Amount, StageName
        WHEN Case        THEN Id, CaseNumber, Status, ContactId
        ELSE Name
    END
FROM Event
WHERE What.Type IN ('Account', 'Opportunity')

// TYPEOF OWNER — for queue vs user assignment
SELECT Id, CaseNumber, Subject,
    TYPEOF Owner
        WHEN User  THEN Id, Name, Email
        WHEN Group THEN Name
    END
FROM Case
WHERE Owner.Type IN ('User', 'Queue')
```

---

### 4.2 `instanceof` Operator in Apex

```apex
List<Event> events = [SELECT Id, Subject,
    TYPEOF What
        WHEN Account     THEN Id, Name, Industry, Phone
        WHEN Opportunity THEN Id, Name, Amount, StageName
        WHEN Case        THEN Id, CaseNumber, Status
        ELSE Name
    END
    FROM Event WHERE What.Type IN ('Account','Opportunity','Case')];

for (Event evt : events) {
    if (evt.What instanceof Account) {
        Account acc = (Account) evt.What;
        System.debug('Account: ' + acc.Name + ' Industry: ' + acc.Industry);
    } else if (evt.What instanceof Opportunity) {
        Opportunity opp = (Opportunity) evt.What;
        System.debug('Opp Amount: ' + opp.Amount);
    } else if (evt.What instanceof Case) {
        Case c = (Case) evt.What;
        System.debug('Case #' + c.CaseNumber);
    }
}
```

---

## PART 5 — DYNAMIC SOQL & SECURITY

### 5. Dynamic SOQL

Dynamic SOQL builds query strings at runtime using Apex strings. It is used when the fields, objects, or conditions are not known at compile time (e.g., user-driven search, metadata-driven apps).

| Method | Use Case |
|--------|---------|
| **`Database.query(queryString)`** | Standard dynamic SOQL — executes string as SOQL |
| **`Database.queryWithBinds(q, bindMap, mode)`** | Spring '23+ — pass bind variables via Map (safer) |
| **`Database.getQueryLocator(q)`** | For Batch Apex `start()` method — cursor-based 50M rows |
| **`Database.getQueryLocatorWithBinds(q, bindMap, mode)`** | Spring '23+ version for Batch Apex locator |
| **`Database.countQuery(q)`** | Returns `Integer` count from a `COUNT()` SOQL string |
| **`Database.countQueryWithBinds(q, bindMap, mode)`** | Spring '23+ bind-safe count query |

---

### 5.1 Static SOQL vs Dynamic SOQL — When to Use Each

```apex
// ── STATIC SOQL (preferred when possible) ──
// Compile-time validation — errors caught before deployment
List<Account> accs = [SELECT Id, Name FROM Account WHERE Industry = :ind];

// ── DYNAMIC SOQL — use when structure is unknown at compile time ──
String query = 'SELECT Id, Name FROM Account WHERE Industry = \'' + ind + '\'';
// DANGEROUS — SOQL Injection risk if ind comes from user input! ⚠️

// ── SAFE DYNAMIC SOQL — always escape user input ──
String safeInd = String.escapeSingleQuotes(ind);
String query   = 'SELECT Id, Name FROM Account WHERE Industry = \'' + safeInd + '\'';
List<Account> accs = Database.query(query);

// ── SAFEST — Use bind variable in dynamic SOQL ──
String query   = 'SELECT Id, Name FROM Account WHERE Industry = :ind';
List<Account> accs = Database.query(query);
// Note: bind variable :ind must be in scope — resolved from current Apex context
```

---

### 5.2 `Database.queryWithBinds` — Spring '23+ Best Practice

```apex
// Safest dynamic SOQL — bind variables resolved from Map, not scope
Map<String, Object> bindMap = new Map<String, Object>{
    'industryKey' => 'Technology',
    'minAmount'   => 50000
};

String query = 'SELECT Id, Name FROM Account WHERE Industry = :industryKey';
List<Account> results = Database.queryWithBinds(query, bindMap, AccessLevel.USER_MODE);

// AccessLevel options:
// AccessLevel.USER_MODE   — respects FLS, sharing rules, object permissions
// AccessLevel.SYSTEM_MODE — ignores all security (like WITH SYSTEM_MODE)
```

---

### 5.3 `WITH SECURITY_ENFORCED` vs `USER_MODE` vs `SYSTEM_MODE`

> ⚡ **CRITICAL PDF GAP:** Security keywords in SOQL were completely absent from the PDF. These are **mandatory** in modern Salesforce code reviews and heavily tested for senior developers.

```apex
// WITH SECURITY_ENFORCED (older approach — deprecated in favor of USER_MODE)
// Throws exception if user lacks access to any field in SELECT
List<Account> accs = [SELECT Id, Name, AnnualRevenue
                      FROM Account WITH SECURITY_ENFORCED];

// WITH USER_MODE (preferred — Spring'22+)
// Silently strips inaccessible fields; respects FLS, sharing, CRUD
List<Account> accs = [SELECT Id, Name FROM Account WITH USER_MODE];

// WITH SYSTEM_MODE (runs in system context regardless of sharing)
// Use only when deliberately bypassing security (e.g., system jobs)
List<Account> accs = [SELECT Id, Name FROM Account WITH SYSTEM_MODE];

// ── Difference from without sharing class ──
// WITHOUT SHARING class: ignores sharing rules but still enforces FLS
// WITH SYSTEM_MODE:       ignores BOTH sharing rules AND FLS
```

| Keyword | FLS Enforced? | Sharing Rules? | When to Use |
|---------|:------------:|:--------------:|-------------|
| **Default (no keyword)** | No | Class-defined | Simple internal queries |
| **`WITH SECURITY_ENFORCED`** | Yes (throws error) | Class-defined | Legacy enforcement |
| **`WITH USER_MODE`** | Yes (strips fields) | Enforced | Modern best practice |
| **`WITH SYSTEM_MODE`** | No | Bypassed | System jobs, setup objects |

---

## PART 6 — SOQL IN APEX: PATTERNS & BEST PRACTICES

### 6. SOQL in Apex — Inline, Bind Variables, For Loop

#### 6.1 Inline SOQL & Bind Variables

```apex
// Inline SOQL — most common form
List<Account> accs = [SELECT Id, Name FROM Account WHERE Name = 'Acme'];

// Bind variable (:) — resolves Apex variable value into SOQL
String accountName = 'Genepoint';
List<Account> accs = [SELECT Id, Name FROM Account WHERE Name =: accountName LIMIT 50000];

// Bind with List/Set (IN clause)
List<String> industries = new List<String>{'Education','Banking','Consulting'};
List<Account> accs = [SELECT Id, Name FROM Account WHERE Industry IN: industries];

// Bind with Set<Id> — most common trigger pattern
Set<Id> accIds = new Set<Id>();
// ... populate accIds ...
List<Account> accs = [SELECT Id, Name FROM Account WHERE Id IN: accIds];

// Dynamic LIKE using bind
String keyword   = 'tech';
String likeParam = '%' + keyword + '%';  // build pattern first
List<Account> accs = [SELECT Id, Name FROM Account WHERE Name LIKE: likeParam];
```

---

#### 6.2 SOQL For Loop — Heap-Safe Large Data Processing

> ⚡ **PDF GAP (partial):** The PDF showed SOQL For Loop in code but did not explain **WHY** it is critical. The SOQL For Loop processes 200 records at a time using a server-side cursor — this is the **ONLY** way to safely process 50k+ records without hitting the 6MB heap limit.

```apex
// ── WRONG: loads ALL records into heap at once ──
List<Contact> allCons = [SELECT Id, Name FROM Contact LIMIT 50000];
for (Contact c : allCons) { /* process */ }  // 50k objects in heap!

// ── CORRECT: SOQL For Loop — server-side cursor, 200 at a time ──
for (Contact c : [SELECT Id, Name, AccountId FROM Contact WHERE Email != null]) {
    // Each iteration: 1 record from the chunk of 200
    // Total heap: ~200 Contact objects max, not 50,000
}

// ── BATCH version (200 per chunk) — use for DML inside loop ──
for (List<Contact> conBatch : [SELECT Id, Name FROM Contact LIMIT 50000]) {
    update conBatch;  // DML on batch of 200 — still governed by DML limits!
}
// DML inside SOQL For Loop still counts toward 150 DML statement limit ⚠️
```

---

#### 6.3 Map from SOQL — The Power Pattern

```apex
// Auto-key by Id — most common trigger pattern
Map<Id, Account> accountMap = new Map<Id, Account>([SELECT Id, Name, Industry FROM Account]);

// Use in trigger: look up parent for each child (zero SOQL in loop)
trigger ContactTrigger on Contact (before insert) {
    Set<Id> accIds = new Set<Id>();
    for (Contact c : Trigger.new) accIds.add(c.AccountId);

    Map<Id, Account> accMap = new Map<Id, Account>(
        [SELECT Id, Name, Industry FROM Account WHERE Id IN: accIds]
    );

    for (Contact c : Trigger.new) {
        Account relAcc = accMap.get(c.AccountId);
        if (relAcc != null) c.Description = 'Industry: ' + relAcc.Industry;
    }
}
```

---

#### 6.4 Null-Safe SOQL Patterns

```apex
// Check if result is empty (prefer isEmpty over size > 0)
List<Account> accs = [SELECT Id FROM Account WHERE Name = 'Unknown'];
if (!accs.isEmpty()) {
    // Process
}

// Single record — guard against empty list with getSObject
Account a = Database.query('SELECT Id, Name FROM Account LIMIT 1');
// Throws exception if query returns 0 rows! ⚠️

// Safer:
List<Account> results = [SELECT Id FROM Account WHERE Name = 'X' LIMIT 1];
if (!results.isEmpty()) {
    Account a = results[0];
}
```

---

## PART 7 — SOQL GOVERNOR LIMITS & OPTIMIZATION

### 7. Governor Limits Related to SOQL

| Limit | Synchronous | Asynchronous | Notes |
|-------|:-----------:|:------------:|-------|
| **SOQL queries per transaction** | 100 | 200 | Most critical limit |
| **Records returned per transaction (total)** | 50,000 | 50,000 | Across ALL queries combined |
| **Records returned by `Database.getQueryLocator`** | 10,000 | 50,000,000 | Used in Batch Apex `start()` |
| **SOSL queries per transaction** | 20 | 20 | Different limit from SOQL |
| **Records returned per SOSL** | 2,000 | 2,000 | Much lower than SOQL |
| **Subqueries per SOQL** | 20 | 20 | Per SELECT statement |
| **Child rows per subquery** | 200 | 200 | Per parent record |
| **OFFSET maximum** | 2,000 | 2,000 | Use keyset pagination beyond |

---

### 7.1 Best Practices to Stay Within Limits

- Never put SOQL inside a `for` loop — collect IDs first, query once outside
- Use `Set<Id>` to collect IDs → single `WHERE Id IN: idSet` query
- Use `Map<Id, sObject>(SOQL)` for zero-loop lookups
- Query only needed fields — never `SELECT *` (not even valid in SOQL)
- Use indexed fields in `WHERE` clause for selective queries
- Add `LIMIT` clause in queries that may return large volumes
- Use **SOQL For Loop** for 50k+ record processing to avoid heap limit
- Use **Batch Apex** for 50k+ record processing with DML
- Use `Database.getQueryLocator` in Batch Apex for up to 50M records
- Monitor usage: `Limits.getQueries()` / `Limits.getLimitQueries()`

---

### 7.2 SOQL Injection — Critical Security Topic

> ⚡ **PDF GAP:** SOQL Injection was not mentioned at all. This is a **top security concern** and tested in every senior developer interview.

```apex
// ── VULNERABLE — user input directly concatenated ──
String name = ApexPages.currentPage().getParameters().get('name');
// If user enters: ' OR '1'='1 → returns ALL accounts!
String q   = 'SELECT Id FROM Account WHERE Name = \'' + name + '\'';
List<Account> accs = Database.query(q);  // SOQL injection possible!

// ── SAFE Method 1: String.escapeSingleQuotes() ──
String safeName = String.escapeSingleQuotes(name);
String q        = 'SELECT Id FROM Account WHERE Name = \'' + safeName + '\'';

// ── SAFE Method 2: Static SOQL with bind variable (BEST) ──
List<Account> accs = [SELECT Id FROM Account WHERE Name =: name];
// Bind variables are NEVER vulnerable to injection

// ── SAFE Method 3: Database.queryWithBinds (Spring'23+) ──
Map<String,Object> binds = new Map<String,Object>{'n' => name};
List<Account> accs = Database.queryWithBinds(
    'SELECT Id FROM Account WHERE Name = :n',
    binds,
    AccessLevel.USER_MODE
);
```

---

## PART 8 — COMPLETE PDF GAPS vs OFFICIAL DOCS

### 8. What the PDF Missed *(Referenced from Salesforce Docs)*

| Gap / Missing Topic | Why It Matters at 5 Years Experience |
|--------------------|--------------------------------------|
| **`WITH USER_MODE` / `SECURITY_ENFORCED` / `SYSTEM_MODE`** | Required for all production-grade SOQL — security compliance |
| **`OFFSET` for pagination** | Used in every search/list UI implementation |
| **Date Literals (`TODAY`, `LAST_N_DAYS:n`, etc.)** | Used in dashboards, reports, scheduled queries constantly |
| **SOQL Injection & `String.escapeSingleQuotes()`** | Critical security — tested in every senior interview |
| **`Database.queryWithBinds` (Spring'23+)** | Modern best practice for safe dynamic SOQL |
| **`Database.getQueryLocator`** | Needed to understand Batch Apex `start()` method |
| **`NOT IN` clause** | Common pattern for exclusion queries |
| **NULL semantics (`= null` vs `!= null`)** | SOQL null handling differs from SQL `IS NULL` |
| **`COUNT_DISTINCT()`, `AVG()`** | Missing aggregate functions — tested in data work |
| **`GROUPING()` function** | Used with `ROLLUP`/`CUBE` — identifies subtotal rows |
| **`AggregateResult .get()` access** | How to extract values from aggregate queries in Apex |
| **Child list returns `null` (not empty)** | Classic NPE bug — parent-to-child returns null with no children |
| **Subquery 200-row limit per parent** | Pagination needed for parents with >200 children |
| **`OFFSET` max = 2000 / keyset pagination** | Important for scalable list UIs |
| **`FOR VIEW` / `FOR UPDATE` / `FOR REFERENCE`** | Locking, tracking — used in advanced concurrent update scenarios |
| **Custom relationship `__r` notation** | Required for all custom object relationship queries |
| **Multi-level traversal depth (5 levels max)** | Governs how deep relationship queries can go |

---

## PART 9 — INTERVIEW QUESTIONS (5 YEARS EXPERIENCE)

### 9. Conceptual Interview Questions

> Expect these in phone screens and first rounds. Answer with real-world context, not definitions alone.

---

**Q1. What is the difference between SOQL and SQL? What can SOQL NOT do that SQL can?**

**▶** SOQL is **SELECT-only** — no `INSERT`, `UPDATE`, `DELETE`. No traditional JOINs across unrelated objects. Only relationship-based traversal. No subquery nesting beyond 1 level in parent-to-child. Max 50,000 records per transaction. No wildcard `SELECT *` — must name fields explicitly. SOQL is platform-specific to Salesforce; SQL works on any RDBMS.

---

**Q2. What is the difference between SOQL and SOSL?**

**▶** SOQL queries a **single object** (or related objects via relationships) with precise conditions — like SQL SELECT. SOSL performs **full-text search** across multiple objects simultaneously. Limits: SOQL = 100 queries/50k rows (sync); SOSL = 20 queries/2,000 rows. Use SOQL when you know which object and fields to query. Use SOSL when searching a keyword across Account + Contact + Lead in one shot.

---

**Q3. What is a bind variable in SOQL and why should you use it?**

**▶** A bind variable uses the `:` syntax to inject an Apex variable's value into a SOQL `WHERE` clause — e.g., `WHERE Id IN: mySet`. Benefits: (1) **Prevents SOQL injection** — bound values are never interpreted as SOQL code. (2) Cleaner than string concatenation. (3) Works with collections (Set, List) directly. Always prefer bind variables over string concatenation in dynamic SOQL.

---

**Q4. What is the difference between HAVING and WHERE in SOQL?**

**▶** `WHERE` filters individual records **BEFORE** aggregation — you cannot use aggregate functions in `WHERE`. `HAVING` filters the aggregated results **AFTER** `GROUP BY` — you CAN use aggregate functions. Example: `SELECT LeadSource, COUNT(Id) FROM Lead GROUP BY LeadSource HAVING COUNT(Id) > 5`. If you tried `WHERE COUNT(Id) > 5`, it would throw a compile error.

---

**Q5. Explain the difference between `COUNT()`, `COUNT(Id)`, and `COUNT(fieldName)`.**

**▶** `COUNT()` with no argument counts **ALL rows** returned — does not count nulls per field. `COUNT(Id)` counts rows where `Id` is not null (effectively same as `COUNT()` since Id is always populated). `COUNT(fieldName)` counts rows where the specified field has a **non-null value** — so it may return fewer rows than `COUNT()`. `COUNT_DISTINCT(fieldName)` counts **unique** non-null values.

---

### 10. Governor Limits & Performance Questions

> These are the most important questions for 5+ year developers — expect these in every round.

---

**Q6. What are the key SOQL governor limits in synchronous vs asynchronous Apex?**

**▶** Synchronous: **100 SOQL queries**, **50,000 total rows** across all queries. Asynchronous: **200 SOQL queries**, **50,000 total rows**. `Database.getQueryLocator` allows up to **50 MILLION rows** in Batch Apex. SOSL: 20 queries, 2,000 rows — same for both. Hitting these limits throws an uncatchable `System.LimitException` at runtime.

---

**Q7. Why should you never write SOQL inside a for loop? Show a bad and good example.**

**▶** SOQL inside a loop uses one query per iteration. With 200 contacts in a trigger, that's 200 queries — instantly hits the 100-query limit.

```apex
// BAD: SOQL inside loop
for (Contact c : Trigger.new) {
    Account a = [SELECT Id FROM Account WHERE Id = :c.AccountId];  // 1 query per contact!
}

// GOOD: collect Set<Id>, query once outside loop into Map, use accMap.get() inside
Set<Id> accIds = new Set<Id>();
for (Contact c : Trigger.new) accIds.add(c.AccountId);
Map<Id, Account> accMap = new Map<Id, Account>(
    [SELECT Id, Name FROM Account WHERE Id IN: accIds]
);
for (Contact c : Trigger.new) {
    Account a = accMap.get(c.AccountId);  // O(1) lookup — 1 query total
}
```

---

**Q8. What is the difference between SOQL For Loop and regular for loop with SOQL? When do you use each?**

**▶** Regular `for` loop loads all records into heap at once (up to 50k objects). SOQL For Loop processes **200 records at a time** using a server-side cursor — never loads the full dataset into heap. Use **SOQL For Loop** when processing 50k+ records to avoid the 6MB heap limit. Use List assignment when you need all records in memory for Map building or cross-referencing.

---

**Q9. You have a trigger on Contact and need to query parent Account data for 10,000 contacts. How do you avoid governor limit issues?**

**▶** Step 1: Collect `AccountId`s into a `Set<Id>` in one pass over `Trigger.new`. Step 2: Run ONE SOQL outside the loop: `Map<Id, Account> accMap = new Map<Id, Account>([SELECT Id, Name FROM Account WHERE Id IN: accIds])`. Step 3: Loop `Trigger.new` again, use `accMap.get(c.AccountId)` — no additional queries. Result: **1 query total**, O(1) Map lookup, handles 10k+ records safely.

---

**Q10. What is the maximum number of records that can be processed in Batch Apex and how does SOQL enable this?**

**▶** `Database.getQueryLocator` in Batch Apex's `start()` method can return up to **50 MILLION records** — far beyond the 50k limit of regular SOQL. The locator returns a cursor reference, not actual records. Salesforce processes records in chunks (default 200, max 2000) per `execute()` call. This is the **ONLY** way to process millions of records in Salesforce without a callout.

---

### 11. Scenario-Based Questions

> Write the actual SOQL query as part of your answer — interviewers often ask you to code on the spot.

---

**Q11. Write a SOQL query to find all Contacts without an associated Account.**

**▶**
```apex
SELECT Id, FirstName, LastName, Email FROM Contact WHERE AccountId = null
```
Note: SOQL uses `= null`, NOT `IS NULL` (SQL syntax). This returns contacts with no `AccountId` populated.

---

**Q12. Write a SOQL query to find all Accounts that have at least one Contact.**

**▶**
```apex
SELECT Id, Name FROM Account WHERE Id IN (SELECT AccountId FROM Contact WHERE AccountId != null)
```
Uses a **semi-join** (subquery in WHERE) to filter Accounts that appear in the Contact table.

---

**Q13. Write a SOQL query to find duplicate Contacts by Name and Email.**

**▶**
```apex
SELECT Name, Email, COUNT(Id) cnt FROM Contact GROUP BY Name, Email HAVING COUNT(Id) > 1
```
Groups by both fields, `HAVING` filters to groups with more than one record. Returns `AggregateResult` — use `.get('cnt')` to access count.

---

**Q14. Write a SOQL query to show the MAX Opportunity Amount per StageName.**

**▶**
```apex
SELECT StageName, MAX(Amount) maxAmt FROM Opportunity GROUP BY StageName
```
Returns `List<AggregateResult>`. Access with `ar.get('StageName')` and `ar.get('maxAmt')` after casting.

---

**Q15. Write a SOQL query that queries an Invoice, its related Account, and the Account's Parent Account.**

**▶**
```apex
SELECT Id, Name, Account__r.Name, Account__r.ParentId, Account__r.Parent.Name
FROM Invoice__c
```
This traverses 3 levels: Invoice → Account → Parent Account. Note: `__r` notation for custom lookup, `.Parent` for standard self-relationship on Account.

---

**Q16. How do you paginate SOQL results in a Lightning component?**

**▶** Use `OFFSET` and `LIMIT`: Page 1 = `LIMIT 20 OFFSET 0`, Page 2 = `LIMIT 20 OFFSET 20`, etc. Limit: `OFFSET` max is **2000**. For >2000 rows, switch to **keyset pagination** — store the last record's `OrderBy` field value, then filter: `WHERE Name > :lastSeenName ORDER BY Name LIMIT 20`. Keyset is also more performant as it uses indexes.

---

**Q17. Explain `WITH USER_MODE` and when you would use it versus not using it.**

**▶** `WITH USER_MODE` enforces Field Level Security and Sharing Rules for the running user — fields the user cannot read are **silently stripped** from results. Use it in any Apex that runs in user context (Apex triggered by user actions, LWC/Aura controllers). Omit it (or use `WITH SYSTEM_MODE`) only in system-context jobs like Batch Apex, scheduled jobs, or when you explicitly need to bypass sharing for admin-only operations. Not using security keywords in user-facing code can expose data to unauthorized users — a security vulnerability.

---

**Q18. A developer wrote `WHERE Name LIKE '%' + userInput + '%'` in a dynamic SOQL query. What is the problem and how do you fix it?**

**▶** This is a **SOQL Injection vulnerability**. If `userInput = 'test' OR Name LIKE '%'`, the WHERE clause evaluates to true for every record, returning the entire org's data. Fix:

```apex
// (1) String.escapeSingleQuotes() — sanitizes before concatenation
String.escapeSingleQuotes(userInput)

// (2) Better: use a bind variable — bind variables are NEVER vulnerable
String likeParam = '%' + userInput + '%';
[WHERE Name LIKE: likeParam]

// (3) Best for dynamic queries: Database.queryWithBinds() with AccessLevel.USER_MODE
```

---

### 12. Tricky / Output Questions

> Quick-fire questions — these catch developers who know SOQL but haven't written enough real code.

---

**Q19. What is returned by `List<AggregateResult> r = [SELECT SUM(Amount) s FROM Opportunity];`? How do you get the value?**

**▶** Returns a `List` with **one** `AggregateResult` element. Access:
```apex
Decimal total = (Decimal) r[0].get('s');
```
If no alias (`SUM(Amount)` without `'s'`), the auto-generated key is `'expr0'`. Always use aliases for readable code.

---

**Q20. What happens if you call `a.Contacts` on an Account that has no related Contacts?**

**▶** In a parent-to-child query result, `a.Contacts` returns **`null`** — NOT an empty List. Always null-check:
```apex
List<Contact> cons = a.Contacts;
if (cons != null) {
    for (Contact c : cons) { ... }
}
```
This is a classic NPE source in production code.

---

**Q21. Can you use SOQL in a static block of an Apex class?**

**▶** Yes, but it is **highly discouraged**. Static blocks run when the class is first loaded — if the class is loaded in any trigger or batch, the SOQL counts toward the transaction limits. It also makes the class impossible to mock in unit tests. Always use lazy initialization instead.

---

**Q22. What is the difference between `ORDER BY Name NULLS FIRST` and `NULLS LAST`?**

**▶** `NULLS FIRST` places records with null `Name` at the **top** of results. `NULLS LAST` places them at the **bottom**. The default when not specified: `NULLS FIRST` for `ASC` order, `NULLS LAST` for `DESC` order. Always specify explicitly to avoid environment-dependent sort behavior.

---

**Q23. Can you query a field with `WITH SECURITY_ENFORCED` if the user has read access to the object but not the field?**

**▶** No — `WITH SECURITY_ENFORCED` throws a `QueryException` at runtime if the user lacks access to **ANY** field in the `SELECT` list or `WHERE` clause. `WITH USER_MODE` (preferred) **silently strips** inaccessible fields instead of throwing. This difference — **exception vs silent strip** — is a key interview distinction.

---

## PART 10 — SOQL QUICK CHEAT SHEET

### 13. Complete Query Cheat Sheet

```apex
// ── BASIC ────────────────────────────────────────────────────────────────
SELECT Id, Name FROM Account WHERE Industry = 'Tech' ORDER BY Name ASC LIMIT 50

// ── BIND VARIABLE ─────────────────────────────────────────────────────────
String ind = 'Technology';
List<Account> a = [SELECT Id FROM Account WHERE Industry =: ind];

// ── LIKE + BIND ───────────────────────────────────────────────────────────
String key = '%tech%';
List<Account> a = [SELECT Id FROM Account WHERE Name LIKE: key];

// ── NULL CHECK ────────────────────────────────────────────────────────────
WHERE Email = null    // no Email
WHERE Email != null   // has Email

// ── DATE LITERALS ─────────────────────────────────────────────────────────
WHERE CreatedDate = TODAY
WHERE CloseDate   = LAST_N_DAYS:30

// ── AGGREGATE ─────────────────────────────────────────────────────────────
List<AggregateResult> r = [
    SELECT Industry, COUNT(Id) cnt, SUM(AnnualRevenue) rev
    FROM Account
    GROUP BY Industry
    HAVING COUNT(Id) > 2
];
for (AggregateResult ar : r) {
    System.debug(ar.get('Industry') + ' → ' + ar.get('cnt'));
}

// ── CHILD-TO-PARENT ───────────────────────────────────────────────────────
SELECT Id, Name, Account.Name, Account.Industry FROM Contact
SELECT Id, Name, Account__r.Name FROM Invoice__c

// ── PARENT-TO-CHILD ───────────────────────────────────────────────────────
SELECT Id, Name,
    (SELECT Id, Email   FROM Contacts),
    (SELECT Id, Amount  FROM Opportunities)
FROM Account

// ── MAP FROM SOQL ─────────────────────────────────────────────────────────
Map<Id, Account> m = new Map<Id, Account>([SELECT Id, Name FROM Account]);

// ── SOQL FOR LOOP ─────────────────────────────────────────────────────────
for (Contact c : [SELECT Id FROM Contact LIMIT 50000]) { /* 200 at a time */ }

// ── SECURITY ──────────────────────────────────────────────────────────────
SELECT Id FROM Account WITH USER_MODE
SELECT Id FROM Account WITH SECURITY_ENFORCED

// ── DYNAMIC ───────────────────────────────────────────────────────────────
Map<String,Object> binds = new Map<String,Object>{'n' => name};
Database.queryWithBinds('SELECT Id FROM Account WHERE Name = :n', binds, AccessLevel.USER_MODE);

// ── PAGINATION ────────────────────────────────────────────────────────────
SELECT Id FROM Account ORDER BY Name LIMIT 20 OFFSET 40  // page 3

// ── TYPEOF ────────────────────────────────────────────────────────────────
SELECT Id, Subject,
    TYPEOF What
        WHEN Account     THEN Name, Phone
        WHEN Opportunity THEN Amount
    END
FROM Event WHERE What.Type IN ('Account','Opportunity')
```

---

### 14. Official Salesforce Documentation

- **SOQL SELECT Syntax:** <https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_select.htm>
- **Aggregate Functions:** <https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_select_agg_functions.htm>
- **SELECT Examples:** <https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/sforce_api_calls_soql_select_examples.htm>
- **Polymorphic Relationships:** <https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/langCon_apex_SOQL_polymorphic_relationships.htm>
- **Dynamic SOQL:** <https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_dynamic_soql.htm>
- **Governor Limits:** <https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm>

---
