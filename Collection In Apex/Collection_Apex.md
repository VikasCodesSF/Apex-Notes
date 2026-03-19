# COLLECTIONS: List, Set & Map
### Comprehensive Notes + 5-Year Experience Interview Questions

---

## PART 1 — OVERVIEW & FUNDAMENTALS

### 1. What Are Collections?

A Collection is a data structure in Apex used to store multiple values of the same (or compatible) data type in a single variable. Instead of declaring dozens of separate variables, collections allow you to work with large datasets efficiently and effectively.

| Collection Type | Key Characteristics |
|----------------|---------------------|
| **List** | Ordered, allows duplicates, index-based access |
| **Set** | Unordered, no duplicates, no index access |
| **Map** | Key-value pairs, unique keys, fast lookup |

> 📌 **IMPORTANT (Official Salesforce Docs):** There is no limit on the number of items a collection can hold. However, the total heap size is limited — **6 MB** for synchronous transactions and **12 MB** for asynchronous transactions.

---

## PART 2 — LIST

### 2. List

#### 2.1 Definition

A List is an **ordered** collection of elements that can be accessed using their index. Lists allow duplicate values and support elements of any data type — primitives, sObjects, custom Apex classes, or even nested collections.

| Property | Detail |
|----------|--------|
| **Order** | Maintained (insertion order preserved) |
| **Duplicates** | Allowed |
| **Index-based access** | Yes — starts at index 0 |
| **Null values** | Allowed |
| **Max nesting** | Up to 8 levels of nested collections |
| **Heap limit** | 6 MB sync / 12 MB async |

---

#### 2.2 Declaration Syntax (3 Ways)

**Method #1 — Declare and allocate memory (empty list):**

```apex
List<String> fruitList = new List<String>();
// Size = 0, can grow dynamically
```

**Method #2 — Declare + allocate + assign values (inline):**

```apex
List<String>  fruitList = new List<String>{'Apple', 'Banana', 'Orange'};
List<Integer> numbers   = new List<Integer>{1, 2, 4, 5, 3};
```

**Method #3 — Declare first, assign later (deferred initialization):**

```apex
List<String> fruitList;                                        // null (no memory allocated)
fruitList = new List<String>();                                // memory allocated, size = 0
fruitList = new List<String>{'Apple', 'Banana', 'Orange'};    // assigned
```

---

#### 2.3 Index Mechanics

Index always starts at `0`. Last index = `size() - 1`.

```
List with 4 elements:
  Index 0 → Apple
  Index 1 → Mango
  Index 2 → Orange
  Index 3 → Banana
  Last Index = 4 - 1 = 3
```

---

#### 2.4 Commonly Used List Methods

| Method | Description & Example |
|--------|----------------------|
| **`add(element)`** | Appends element at end → `fruitList.add("Guava");` |
| **`add(index, element)`** | Inserts at specific position → `fruitList.add(1, "Guava");` |
| **`set(index, element)`** | Replaces element at index → `fruitList.set(1, "Banana");` |
| **`get(index)`** | Returns element at index → `String s = fruitList.get(0);` |
| **`remove(index)`** | Removes element at index → `fruitList.remove(2);` |
| **`size()`** | Returns count of elements → `Integer n = fruitList.size();` |
| **`isEmpty()`** | Returns true if no elements → `Boolean b = fruitList.isEmpty();` |
| **`clear()`** | Removes all elements → `fruitList.clear();` |
| **`clone()`** | Creates a shallow copy → `List<String> copy = fruitList.clone();` |
| **`sort()`** | Sorts elements ascending → `fruitList.sort();` |
| **`contains(element)`** | Checks if element exists (returns Boolean) → `fruitList.contains("Apple");` |
| **`addAll(list/set)`** | Adds all from another collection → `fruitList.addAll(anotherList);` |
| **`indexOf(element)`** | Returns first index of element, or `-1` if not found |
| **`deepClone()`** | Creates deep copy including sObject fields → `list.deepClone();` |
| **`equals(list2)`** | Compares two lists for equality |
| **`hashCode()`** | Returns hash code of the list |
| **`iterator()`** | Returns iterator for the list |

> ⚡ **PDF GAP:** The PDF only covered `add`, `set`, `get`, `clear`, `clone`, `addAll`, `remove`, `isEmpty`, `size`, `sort`. Missing methods include: `contains()`, `indexOf()`, `deepClone()`, `equals()`, `add(index, element)`, `hashCode()`, `iterator()`. These are commonly tested in senior interviews.

---

#### 2.5 Hands-On Code *(from PDF — Annotated)*

```apex
List<String> fruits = new List<String>();
fruits.add("Grapes");  // index 0
fruits.add("Mango");   // index 1
fruits.add("Apple");   // index 2
fruits.add("Grapes");  // index 3 ← duplicate allowed in List!
fruits.add("Banana");  // index 4

System.debug(fruits);  // [Grapes, Mango, Apple, Grapes, Banana]

fruits.set(1, "Guava");          // Replace index 1
System.debug(fruits);            // [Grapes, Guava, Apple, Grapes, Banana]

String  val   = fruits.get(2);   // "Apple"
Integer len   = fruits.size();   // 5
Boolean empty = fruits.isEmpty(); // false

fruits.clear();
// ⚠️ fruits.get(2) after clear() throws: List index out of bounds: 2

if (fruits.isEmpty()) {
    System.debug("List is blank");
} else {
    System.debug("List is not blank");
}
```

---

#### 2.6 List with sObjects — Real-World Pattern

```apex
// SOQL result stored in List — most common real-world usage
List<Account> accounts = [SELECT Id, Name, Industry FROM Account WHERE Industry = 'Technology'];

// Create new records and insert in bulk (DML outside loop)
List<Contact> contactsToInsert = new List<Contact>();

for (Account acc : accounts) {
    Contact c = new Contact(LastName = acc.Name + '_Contact', AccountId = acc.Id);
    contactsToInsert.add(c);
}

insert contactsToInsert;  // Single DML — follows governor limit best practice
```

---

#### 2.7 When to Use List

- When order of elements matters
- When duplicate values are needed
- When index-based access is required
- When storing SOQL query results (SOQL always returns `List<sObject>`)
- When creating or updating sObject records in bulk via DML
- When sorting is needed (`sort()` method)
- When multi-dimensional data (matrix/nested lists) is needed

---

## PART 3 — SET

### 3. Set

#### 3.1 Definition

A Set is an **unordered** collection of **unique** elements. Sets do not contain duplicates. If you try to add a duplicate, the set silently ignores it without throwing an error. Elements cannot be accessed by index.

| Property | Detail |
|----------|--------|
| **Order** | Unordered (no guaranteed order) |
| **Duplicates** | NOT allowed — silently ignored |
| **Index-based access** | NOT supported |
| **Use case** | Storing unique identifiers, IDs, emails, de-duplication |
| **Internal mechanism** | Hash-based (like Java `HashSet`) |
| **Key Note** | Iteration order is deterministic but not insertion order |

---

#### 3.2 Declaration Syntax

```apex
Set<String> colors = new Set<String>();                          // Empty set
Set<String> colors = new Set<String>{'Red', 'Blue', 'Green'};   // Inline

Set<String> colors;  // null

// NOTE: If you add a duplicate, it is silently ignored:
Set<String> test = new Set<String>{'A', 'B', 'A'};  // Size = 2, not 3
```

> ⚡ **PDF GAP:** The PDF showed `Set<Integer> colorsSet` with String values in the code — that is a type mismatch error in real code. The correct syntax for a String Set is `Set<String>`. Always match the generic type to the values being stored.

---

#### 3.3 Commonly Used Set Methods

| Method | Description & Example |
|--------|----------------------|
| **`add(element)`** | Adds element — silently ignores if duplicate → `colors.add("Purple");` |
| **`remove(element)`** | Removes specified element (not by index!) → `colors.remove("Red");` |
| **`contains(element)`** | Returns Boolean — most powerful Set method → `colors.contains("Blue");` |
| **`containsAll(collection)`** | Returns true if Set contains all elements of the collection |
| **`addAll(list/set)`** | Adds all elements from another collection → `colors.addAll(anotherSet);` |
| **`removeAll(list/set)`** | Removes all elements that exist in the given collection |
| **`retainAll(list/set)`** | Keeps only elements that exist in both collections (intersection) |
| **`clear()`** | Removes all elements → `colors.clear();` |
| **`size()`** | Returns count of unique elements → `Integer n = colors.size();` |
| **`isEmpty()`** | Returns true if no elements → `Boolean b = colors.isEmpty();` |
| **`clone()`** | Creates shallow copy of the set |
| **`equals(set2)`** | Returns true if both sets contain exactly same elements |
| **`hashCode()`** | Returns hash code of the set |

> ⚡ **PDF GAP:** Missing Set methods — `containsAll()`, `removeAll()`, `retainAll()`. These are used in advanced de-duplication, trigger optimization, and bulk processing scenarios. Senior developers are expected to know these.

---

#### 3.4 Hands-On Code *(from PDF — Annotated)*

```apex
Set<String> colors = new Set<String>{'black', 'Red', 'Black', 'Blue', 'Green', 'Black'};
// Duplicate "Black" variants silently ignored → size = 5 (black, Red, Black, Blue, Green)
// Note: "black" and "Black" are DIFFERENT — Set is case-sensitive!

System.debug(colors);
System.debug(colors.contains("Black"));  // true

Set<String> fruits = new Set<String>();
fruits.add("Apple");
fruits.add("Banana");
fruits.add("Mango");
colors.addAll(fruits);  // Merges fruits into colors
System.debug(colors);

// Set with sObjects → used in triggers frequently
Account a1 = new Account(); a1.Name = "Demo1";
Account a2 = new Account(); a2.Name = "Demo1";  // same name but DIFFERENT object ref
Account a3 = new Account(); a3.Name = "Demo2";

Set<Account> accSet = new Set<Account>();
accSet.add(a1);
accSet.add(a2);  // Added! Apex uses field hash — a1 & a2 have same fields so DUPLICATE
accSet.add(a3);  // Added — different Name field

List<Account> accList = new List<Account>();
accList.addAll(accSet);
insert accList;
```

---

#### 3.5 When to Use Set

- When storing unique record IDs (`Id` type) — most common trigger pattern
- When de-duplicating data (emails, phone numbers, external IDs)
- When order does not matter
- When using `contains()` for fast O(1) lookup vs List's O(n) iteration
- When building a collection of IDs to pass into a SOQL `WHERE IN` clause

```apex
// CLASSIC trigger pattern using Set for SOQL optimization:
Set<Id> accountIds = new Set<Id>();
for (Contact c : Trigger.new) {
    accountIds.add(c.AccountId);
}

// ONE SOQL query outside loop — governor limit friendly
List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :accountIds];
```

---

## PART 4 — MAP

### 4. Map

#### 4.1 Definition

A Map is a collection of **key-value pairs** where each unique key maps to a single value. Keys must be unique; values can be duplicated. Keys can be any primitive data type (and sObject IDs); values can be any data type including List, Set, or another Map. Map iteration order is deterministic.

| Property | Detail |
|----------|--------|
| **Structure** | Key → Value pairs |
| **Keys** | MUST be unique. Primitives, IDs, sObjects (using field hash) |
| **Values** | Can be duplicated. Any data type including nested collections |
| **Iteration order** | Deterministic — same order on repeated execution |
| **Internal mechanism** | Hash-based (like Java `HashMap`) |
| **Max nesting** | Up to 5 levels of nested collections in Map values |
| **Key type note** | Changing a field on an sObject key changes its hash → treat as new key! |

---

#### 4.2 Declaration Syntax

```apex
// Method 1 — Empty Map
Map<String, Integer> accountMap = new Map<String, Integer>();

// Method 2 — Inline initialization
Map<String, Integer> accountMap = new Map<String, Integer>{'Acme' => 10000, 'XYZ' => 5000};

// Method 3 — Declare first
Map<String, Integer> accountMap;

// Method 4 — From SOQL (ID → sObject) — MOST COMMON in real code
Map<Id, Account> accountMap = new Map<Id, Account>([SELECT Id, Name FROM Account]);
// Automatically uses the Id field as the key!
```

> ⚡ **PDF GAP — CRITICAL PATTERN:** `Map<Id, sObject>(SOQL)` is the most powerful and widely used Map pattern in Salesforce development. Not covered in the PDF. **This is tested in almost every senior interview!**

---

#### 4.3 Map Key-Value Type Combinations

| Map Type | Real-World Usage |
|----------|-----------------|
| **`Map<String, String>`** | Country → Currency, Config key → Value |
| **`Map<String, Integer>`** | Account Name → Revenue |
| **`Map<Id, Account>`** | Auto-keyed from SOQL — fast record lookup in triggers |
| **`Map<Id, List<Contact>>`** | Account ID → Its contacts (parent-child grouping) |
| **`Map<String, List<String>>`** | Country → List of states |
| **`Map<String, Object>`** | Mixed value types — flexible but needs casting |
| **`Map<Id, SObject>`** | Generic sObject map — works with any standard/custom object |

---

#### 4.4 Commonly Used Map Methods

| Method | Description & Example |
|--------|----------------------|
| **`put(key, value)`** | Adds/updates key-value pair → `accountMap.put("Acme", 10000);` |
| **`get(key)`** | Returns value for key (null if not found) → `Integer v = accountMap.get("Acme");` |
| **`containsKey(key)`** | Checks if key exists → `Boolean b = accountMap.containsKey("XYZ");` |
| **`keySet()`** | Returns `Set<KeyType>` of all keys → `Set<String> keys = accountMap.keySet();` |
| **`values()`** | Returns `List<ValueType>` of all values → `List<Integer> vals = accountMap.values();` |
| **`remove(key)`** | Removes key-value pair → `accountMap.remove("Acme");` |
| **`putAll(map)`** | Merges another map → `accountMap.putAll(anotherMap);` |
| **`clear()`** | Removes all entries → `accountMap.clear();` |
| **`size()`** | Count of key-value pairs → `Integer n = accountMap.size();` |
| **`isEmpty()`** | True if no entries → `Boolean b = accountMap.isEmpty();` |
| **`clone()`** | Shallow copy of map |
| **`deepClone()`** | Deep copy — important for sObject maps |
| **`equals(map2)`** | Returns true if both maps have identical key-value pairs |
| **`hashCode()`** | Returns hash code of the map |

---

#### 4.5 Hands-On Code *(from PDF — Annotated + Extended)*

```apex
// Simple Map — Country → Currency
Map<String, String> countryMap = new Map<String, String>();
countryMap.put("India", "INR");
countryMap.put("Japan", "YEN");

if (!countryMap.containsKey("USA")) {
    countryMap.put("USA", "US Dollars");
} else {
    System.debug("Key already exists");
}

String       japanCurrency = countryMap.get("Japan");    // "YEN"
Set<String>  keys          = countryMap.keySet();        // {India, Japan, USA}
List<String> values        = countryMap.values();        // [INR, YEN, US Dollars]

// Nested Map — Country → List of States
Map<String, List<String>> countryStateMap = new Map<String, List<String>>();
List<String> indiaStates = new List<String>{'UP', 'Delhi', 'MP', 'MH'};
countryStateMap.put("India", indiaStates);

// Adding a new state to existing country
if (countryStateMap.containsKey("India")) {
    List<String> states = countryStateMap.get("India");
    states.add("GJ");
    countryStateMap.put("India", states);  // Step 4: update map
}
```

---

#### 4.6 Map from SOQL — The Power Pattern

```apex
// Most important Map pattern in Salesforce:
Map<Id, Account> accountMap = new Map<Id, Account>([SELECT Id, Name, Industry FROM Account]);

// Usage in trigger — get related Account for each Contact (without SOQL in loop!)
trigger ContactTrigger on Contact (before insert) {

    Set<Id> accountIds = new Set<Id>();
    for (Contact c : Trigger.new) { accountIds.add(c.AccountId); }

    Map<Id, Account> accMap = new Map<Id, Account>(
        [SELECT Id, Name FROM Account WHERE Id IN :accountIds]
    );

    for (Contact c : Trigger.new) {
        Account relatedAcc = accMap.get(c.AccountId);
        if (relatedAcc != null) {
            c.Description = "Account: " + relatedAcc.Name;
        }
    }
}
```

---

#### 4.7 When to Use Map

- When you need to look up a value quickly using a key (O(1) lookup)
- When you need to associate IDs with records (trigger pattern)
- When grouping child records by parent ID
- When building configuration/settings key-value stores
- When you need to check existence of a key (`containsKey`) efficiently
- When replacing SOQL inside loops — query once, store in Map, iterate

---

## PART 5 — COLLECTIONS COMPARISON & GOVERNOR LIMITS

### 5. Collections Comparison Matrix

| Feature | List | Set | Map |
|---------|:----:|:---:|:---:|
| **Order maintained** | Yes | No | No (deterministic) |
| **Duplicates allowed** | Yes | No | Keys: No, Values: Yes |
| **Index access** | Yes (0-based) | No | By key |
| **Null values** | Allowed | One null | Key: not allowed, Value: yes |
| **Keyword** | `List<T>` | `Set<T>` | `Map<K,V>` |
| **SOQL result type** | Default return type | No | Via constructor |
| **DML operations** | Yes (`insert`/`update` list) | Convert to List first | Use `.values()` |
| **Common use case** | Records, ordered data | Unique IDs, dedup | Key-value lookup, triggers |
| **Lookup speed** | O(n) — linear | O(1) — hash | O(1) — hash |

---

### 6. Governor Limits Related to Collections

| Limit | Synchronous | Asynchronous |
|-------|:-----------:|:------------:|
| **Heap size** | 6 MB | 12 MB |
| **SOQL queries** | 100 | 200 |
| **DML statements** | 150 | 150 |
| **DML rows** | 10,000 | 10,000 |
| **CPU time** | 10 seconds | 60 seconds |
| **Max nested collection levels** | 8 (List/Set), 5 (Map) | 8 (List/Set), 5 (Map) |

> ⚡ **PDF GAP — GOVERNOR LIMITS:** The PDF only mentions "6MB Sync, 12MB Async" as a comment in code. Collections are the primary heap consumers. A 5-year developer must know: heap size, DML row limit (10,000), SOQL limit (100 sync), and how to monitor with `Limits` class.

```apex
// Monitor heap usage — important for large collection operations
System.debug("Heap used: " + Limits.getHeapSize()       + " / " + Limits.getLimitHeapSize());
System.debug("SOQL used: " + Limits.getQueries()        + " / " + Limits.getLimitQueries());
System.debug("DML used:  " + Limits.getDmlStatements()  + " / " + Limits.getLimitDmlStatements());
```

---

## PART 6 — GAPS IN PDF vs OFFICIAL SALESFORCE DOCS

### 7. What the PDF Missed *(Official Salesforce Docs Reference)*

| Gap / Missing Topic | Why It Matters for 5-Year Developers |
|--------------------|--------------------------------------|
| **`Map<Id, sObject>(SOQL)` pattern** | Most common trigger & bulk pattern — tested in every senior interview |
| **`contains()` on List** | Exists but O(n) — important to know vs `Set.contains()` which is O(1) |
| **`deepClone()` vs `clone()`** | `clone()` does NOT copy sObject field values — `deepClone()` does. Trap question! |
| **`List.sort()` with `Comparable`** | Custom sort order for sObjects using `Comparable` interface |
| **Set type mismatch in PDF** | PDF showed `Set<Integer>` with String values — compile error in reality |
| **Map iteration order** | Deterministic but NOT insertion order — important for data processing |
| **sObject as Map/Set key — hash risk** | Changing a field on sObject key changes its hash → dangerous bug! |
| **`retainAll()` / `removeAll()`** | Powerful set operations for intersection/difference — missing from PDF |
| **Iterator pattern on collections** | `Iterator<T>` interface — used in custom iteration scenarios |
| **`transient` keyword for heap** | Use `transient` in VF controllers to exclude large collections from view state |
| **SOQL for-loop to avoid heap** | `for(Account a : [SELECT...])` processes 200 at a time, avoids 6MB limit |
| **Collection nesting limits** | List/Set: up to 8 levels; Map: up to 5 levels — PDF not mentioned |
| **List array notation** | `String[] names = new String[5];` is equivalent to `List<String>(5)` — alternate syntax |

---

#### 7.1 `deepClone()` vs `clone()` — Important Distinction

```apex
List<Account> original = [SELECT Id, Name FROM Account LIMIT 3];

// clone() — shallow copy. Does NOT copy field values from sObjects
List<Account> shallowCopy = original.clone();
// shallowCopy[0].Name changes DO affect original[0].Name — same reference!

// deepClone() — full copy including field values and optionally IDs
List<Account> deepCopy = original.deepClone(false, true, true);
// Parameters: preserveId, preserveReadonlyTimestamps, preserveAutonumber

// deepCopy[0].Name changes do NOT affect original — independent object
// Common usage: prepare modified copy for insert without changing original
```

---

#### 7.2 sObject as Map/Set Key — The Hash Trap

```apex
// WARNING: Changing a field on an sObject used as a key changes its hash!
Contact c = new Contact(LastName = "Smith");
Map<Contact, String> myMap = new Map<Contact, String>();
myMap.put(c, "Test");

c.LastName = "Jones";  // ← This changes the hash of c!
// Now myMap.get(c) returns NULL — the old key no longer matches!

// BEST PRACTICE: Always use Id (or a stable primitive) as Map/Set key
Map<Id, Account> safeMap = new Map<Id, Account>([SELECT Id, Name FROM Account]);
```

---

## PART 7 — INTERVIEW QUESTIONS (5 YEARS EXPERIENCE)

### 8. Interview Questions — Conceptual

> These questions test fundamental understanding. A 5-year developer should answer with real-world context, not just definitions.

---

**Q1. What are the three types of collections in Salesforce Apex and when would you use each?**

**▶** **List:** ordered, allows duplicates → store SOQL results, bulk DML. **Set:** unique elements, O(1) lookup → store record IDs for SOQL `IN` clause. **Map:** key-value pairs, O(1) lookup → associate records by ID for trigger optimization.

---

**Q2. What is the difference between `clone()` and `deepClone()` in Apex collections?**

**▶** `clone()` creates a shallow copy — sObject field values are still references to originals. `deepClone()` creates independent copies of sObject field values. For sObjects, always use `deepClone()` to avoid unintended mutations. `deepClone()` accepts parameters for preserving ID, timestamps, and autonumber fields.

---

**Q3. Can a Set contain null values? Can a Map have null as a key?**

**▶** Set **can** contain null as an element, but only one null since sets reject duplicates. Map **CANNOT** have null as a key — it throws a `NullPointerException`. Map **values** CAN be null.

---

**Q4. What is the maximum number of levels for nesting collections?**

**▶** Lists and Sets can be nested up to **8 levels** deep. Maps can only contain up to **5 levels** of nested collections. This is a Salesforce platform constraint, not a JVM/memory constraint.

---

**Q5. Explain the iteration order of Set and Map in Apex.**

**▶** Both Set and Map in Apex have **deterministic** iteration order — meaning the same code will always produce elements in the same order on repeated execution. However, this order is **NOT** the insertion order. This differs from Java's `LinkedHashSet`/`LinkedHashMap`. This is stated explicitly in Salesforce official docs.

---

### 9. Interview Questions — Governor Limits & Performance

> These are the most common scenario-based questions for experienced developers.

---

**Q6. How do collections relate to the heap size governor limit?**

**▶** Collections are stored in heap memory. The heap limit is **6 MB** (sync) and **12 MB** (async). Large `List<SObject>` collections with many fields consume significant heap. Best practices: query only needed fields (avoid `SELECT *`), use SOQL for loops for 50k+ records (processes 200 at a time using cursor), use `Map<Id, SObject>` instead of nested objects, and monitor with `Limits.getHeapSize()`.

---

**Q7. Why should you never query inside a loop, and how do collections solve this?**

**▶** Each SOQL call inside a loop counts against the 100 query limit (sync). If you have 200 contacts in a trigger, that's 200 SOQL calls — instant governor limit exception. The solution: collect all needed IDs into a `Set<Id>`, run ONE query outside the loop, store results in a `Map<Id, SObject>`, then iterate and use `Map.get()` for O(1) lookups inside the loop.

---

**Q8. How do you handle more than 200 child records in a parent-child query?**

**▶** Direct assignment (`List<Contact> cons = acc.Contacts`) throws `"Aggregate query has too many rows"` if child count exceeds 200. Use a nested SOQL for loop:

```apex
for (Account acc : [SELECT Id, (SELECT Id FROM Contacts) FROM Account]) {
    for (Contact c : acc.Contacts) { ... }
}
```

Each inner list chunk uses cursor-based processing.

---

**Q9. If you have 50,000 Account records to process, how do you handle collections to avoid heap limit?**

**▶** Use SOQL for loop:

```apex
for (Account acc : [SELECT Id, Name FROM Account]) { ... }
```

This processes 200 records at a time using a server-side cursor, never loading all 50k into heap simultaneously. The alternative — storing all 50k in a `List<Account>` — would require ~100MB (50k × 2KB), far exceeding the 6MB sync limit.

---

**Q10. How do you use the `Limits` class to monitor collection-related resource usage?**

**▶** Use `Limits.getHeapSize()` / `Limits.getLimitHeapSize()` to monitor heap. You can add conditional logic:

```apex
if (Limits.getHeapSize() > Limits.getLimitHeapSize() * 0.8) {
    // warn or branch to async
}
```

Also useful: `Limits.getQueries()`, `Limits.getDmlStatements()`, `Limits.getDmlRows()`.

---

### 10. Interview Questions — Scenario Based

> Real-world code scenarios — think through the answer before reading. These come up in whiteboard rounds.

---

**Q11. A trigger on Contact needs to update the parent Account's Description field. How would you use a Map here?**

**▶** Step 1: Collect Account IDs from `Trigger.new` into a `Set<Id>`. Step 2: Query accounts — `Map<Id, Account> accMap = new Map<Id, Account>([SELECT Id, Description FROM Account WHERE Id IN :accIds])`. Step 3: Loop `Trigger.new`, use `accMap.get(c.AccountId)` to fetch account. Step 4: Update accounts in `accMap`. Step 5: `update accMap.values()`. This is the standard bulkified trigger pattern.

---

**Q12. You have a `List<Account>` with 10,000 records. How do you remove duplicates?**

**▶** **Option 1:** Convert to Set — `Set<Account> uniqueAccs = new Set<Account>(accList);` — but this uses field-hash comparison, which may not match your business dedup logic. **Option 2 (better):** Use a `Map<String, Account>` keyed on the unique field (e.g., `Name` or `External_Id__c`), put all records in — `Map.put()` naturally overwrites duplicates. Then extract: `accList = new List<Account>(accMap.values())`.

---

**Q13. How would you build a Map grouping Contacts by their AccountId from a trigger context?**

**▶**

```apex
Map<Id, List<Contact>> accContactMap = new Map<Id, List<Contact>>();

for (Contact c : Trigger.new) {
    if (!accContactMap.containsKey(c.AccountId)) {
        accContactMap.put(c.AccountId, new List<Contact>());
    }
    accContactMap.get(c.AccountId).add(c);
}
```

This is the classic parent-child grouping pattern used in every complex trigger.

---

**Q14. When would you use `List.sort()` vs a Map for ordering data? What is the `Comparable` interface?**

**▶** `List.sort()` sorts primitives natively. To sort sObjects or custom objects, implement the `Comparable` interface (`compareTo()` method). Map does not sort — it is hash-based. If you need sorted output of sObjects, put them in a List and implement `Comparable`:

```apex
public Integer compareTo(Object o) {
    Account other = (Account)o;
    return this.Name.compareTo(other.Name);
}
```

---

**Q15. What is the difference between `Set.contains()` and `List.contains()` in terms of performance?**

**▶** `Set.contains()` is **O(1)** — constant time using hash lookup. `List.contains()` is **O(n)** — linear scan of all elements. For a list of 50,000 elements, `Set.contains()` is dramatically faster (milliseconds vs seconds). In high-volume trigger scenarios with large ID collections, always use Set for existence checks.

---

**Q16. Explain how you would use `Map<String, Object>` and what the risks are.**

**▶** `Map<String, Object>` stores mixed value types. It is useful in dynamic Apex, REST responses, or flexible configurations. The risk: every value retrieval requires explicit type casting, and a wrong cast throws a `TypeException` at runtime. Best practice: use typed Maps (`Map<String, String>`) whenever possible. Use `Map<String, Object>` only when the value type is genuinely dynamic, and always validate before casting.

---

**Q17. A developer used an sObject as a Map key, and after modifying the object, `Map.get()` returns null. Why?**

**▶** Apex uses a hash of the sObject's field values as the map key. When you modify a field on the sObject after putting it in the map, the hash changes. The map still holds the old hash as key, so the new hash does not match — `Map.get()` returns null. **Fix:** always use the record's `Id` (a stable primitive) as the Map key, not the sObject itself.

---

**Q18. How do you convert between List, Set, and Map?**

**▶**

```apex
// List → Set
new Set<String>(myList)

// Set → List
new List<String>(mySet)

// List → Map (by Id)
new Map<Id, Account>(soqlList)

// Map → List (values)
new List<Account>(myMap.values())

// Map → Set (keys)
myMap.keySet()
// ⚠️ Note: keySet() returns a DIRECT reference, not a copy. Modifying it modifies the map!
```

---

### 11. Interview Questions — Code Output & Tricky Questions

> These are the **"gotcha"** questions — designed to catch developers who only know the basics.

---

**Q19. What is the output?**
```apex
Set<String> s = new Set<String>{'A','B','A'};
System.debug(s.size());
```

**▶** `"2"` — not 3. Set silently ignores the duplicate `"A"`. The second `"A"` add is a no-op.

---

**Q20. What happens when you call `Map.keySet()` and then modify the returned Set?**

**▶** `Map.keySet()` returns a **DIRECT reference** to the map's internal key set — not a copy. Adding to or removing from it directly modifies the Map. This is a common source of bugs. Always clone if you need to modify it safely:

```apex
Set<String> keys = myMap.keySet().clone();
```

---

**Q21. What is the difference between `List<sObject>` and `List<Account>`?**

**▶** `List<sObject>` is a generic polymorphic list that can hold any sObject type. `List<Account>` is strongly typed — you get compile-time type checking and can directly access Account fields without casting. You can assign `List<Account>` to `List<sObject>` but **NOT** vice versa without explicit casting.

---

**Q22. Can you call DML directly on a Set? What is the correct approach?**

**▶** No — DML operations (`insert`, `update`, `delete`) only work on Lists or single sObjects. If you have `Set<Account>`, convert first:

```apex
List<Account> accList = new List<Account>(accSet);
insert accList;
```

---

**Q23. What happens if you call `List.get(index)` on an empty list?**

**▶** Throws: `System.ListException: List index out of bounds: <index>`. Always check `isEmpty()` or `size()` before accessing by index. This is a very common runtime exception in production code.

---

## PART 8 — QUICK REFERENCE CHEAT SHEET

### 12. Collections Quick Cheat Sheet

```apex
// ══════════ LIST ══════════════════════════════════════════════════════
List<String> l = new List<String>();
l.add("A"); l.add("B"); l.add("A");  // Duplicates OK
l.set(0, "Z");                        // Replace at index
l.get(0);                             // "Z"
l.remove(1);                          // Remove index 1
l.size(); l.isEmpty(); l.clear();
l.contains("Z"); l.indexOf("Z");
l.sort(); l.clone(); l.deepClone();

// ══════════ SET ═══════════════════════════════════════════════════════
Set<String> s = new Set<String>();
s.add("A"); s.add("A");               // Second "A" ignored
s.contains("A");                      // true — O(1) lookup
s.remove("A");                        // By VALUE not index
s.size(); s.isEmpty(); s.clear();
s.addAll(anotherList);
s.containsAll(anotherSet);
s.retainAll(anotherSet);              // Intersection
s.removeAll(anotherSet);              // Difference

// ══════════ MAP ═══════════════════════════════════════════════════════
Map<String, Integer> m = new Map<String, Integer>();
m.put("A", 10);                       // Add/update
m.get("A");                           // 10
m.containsKey("A");                   // true
m.keySet();                           // Set<String>
m.values();                           // List<Integer>
m.remove("A");
m.putAll(anotherMap);
m.size(); m.isEmpty(); m.clear();

// Power pattern — Map from SOQL:
Map<Id, Account> accMap = new Map<Id, Account>([SELECT Id, Name FROM Account]);
```

---

### 13. Official Documentation References

- **Collections Overview:** <https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/langCon_apex_collections.htm>
- **List Methods:** <https://developer.salesforce.com/docs/atlas.en-us.apexref.meta/apexref/apex_methods_system_list.htm>
- **Set Methods:** <https://developer.salesforce.com/docs/atlas.en-us.apexref.meta/apexref/apex_methods_system_set.htm>
- **Map Methods:** <https://developer.salesforce.com/docs/atlas.en-us.apexref.meta/apexref/apex_methods_system_map.htm>
- **Governor Limits:** <https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm>
- **Certifications Reference:** <https://trailhead.salesforce.com/en/credentials/administratoroverview/>

---
