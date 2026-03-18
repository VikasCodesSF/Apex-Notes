# Salesforce Apex — Complete Fundamentals
### OOPs · Data Types · Classes · Collections · Loops · Operators · Interview Q&A

---

## 1. Introduction to Salesforce Apex

Salesforce Apex is a cloud-based, strongly typed, object-oriented programming language that runs on the Salesforce platform. It is used to implement custom business logic that cannot be achieved through configuration alone.

### Why Apex? — Limitations of Configuration

- Cannot arrange fields in more than 2 columns on page layouts
- No de-duplication mechanism out of the box
- Cannot select recipients dynamically at runtime in sharing rules
- Cannot create Rollup Summary between Lookup Relationship objects
- Cannot implement complex conditional logic in workflows

### Key Characteristics of Apex

- **Cloud-based** — no installation required, runs in the browser
- **Object-oriented** — every piece of logic lives in Classes and Objects
- **Strongly typed** — data types are declared and enforced at compile time
- **Not case-sensitive** — code can be written in any character case
- Every statement ends with a semicolon (`;`)
- Compiled by the **Apex Compiler**, executed by the **Apex Runtime Engine**
- Compiled classes stored in Metadata Repository and in the `ApexClass` object

### Ways to Write Apex Code

- Standard Navigation: **Setup → Apex Classes → New**
- Developer Console: **File → New → Apex Class**
- **Execute Anonymous Window** — for quick testing
- **Visual Studio Code** with Salesforce Extension Pack

### Ways to Invoke Apex

- Execute Anonymous Window
- Apex Triggers (before/after insert, update, delete, undelete)
- Batch Apex (`Database.executeBatch`)
- Scheduled Apex (`System.schedule`)
- Process Builder / Flow Builder
- Visualforce Pages and Lightning Web Components
- Web Services & REST/SOAP APIs
- Email Services

---

## 2. Object-Oriented Programming (OOP) Concepts

Apex is purely object-oriented. Every piece of business logic is implemented using Classes and Objects.

### 2.1 Object

Any real-world entity with a **state** (attributes) and **behaviour** (methods) is an Object. It can be physical or logical.

- A Cat has colour, height, weight (state) and speaks, walks, sleeps (behaviour)
- In Apex: objects are instances of a class created using the `new` keyword
- Objects are stored in **Heap Memory**
- Each object has its own memory allocation (dynamic memory)
- Maximum **3,000 instances** can be created in a single transaction

### 2.2 Class

A class is a blueprint/prototype from which multiple individual objects can be created. It is a collection of variables, methods, constructors, and properties.

```apex
// Syntax
[Access Modifier] [with sharing / without sharing] class <ClassName> {
    // Variables (State)
    // Methods (Behaviour)
    // Constructors
    // Properties (Getter/Setter)
}

public class Animal {
    public String name    = 'Max';
    Integer age           = 5;                       // private by default
    public static String address = '123 Main Street';
}
```

---

### 2.3 The Four OOP Pillars

#### ▶ Abstraction — *What the object does*

Hiding internal implementation details and exposing only the functionality. Think of a smartphone — you use apps without knowing the circuit board details.

- Achieved using **Interfaces** and **Abstract classes**
- Provides 'Data Hiding' — users see what is needed, not how it works

#### ▶ Encapsulation — *How the object does it*

Wrapping code and data together into a single unit. A capsule wrapping different medicines is a great analogy. In Salesforce, a class is the perfect example of encapsulation.

- Access modifiers enforce encapsulation
- Private variables with public getter/setter methods is the pattern

#### ▶ Inheritance — *Reusing parent properties*

When a child class acquires all the variables and methods from a parent class. Prevents code redundancy and enables code reuse.

```apex
// Parent class
public class Animal {
    public String name;
    public void speak() { System.debug(name + ' speaks'); }
}

// Child class inherits Animal
public class Dog extends Animal {
    public void bark() { System.debug('Woof!'); }
}

Dog d = new Dog();
d.name  = 'Max';   // inherited from Animal
d.speak();         // inherited method
d.bark();          // own method
```

#### ▶ Polymorphism — *Same name, different behaviour*

When the same task is done in different ways. Apex supports two types:

- **Compile-Time (Early Binding):** Method Overloading and Constructor Overloading — same name, different parameter signatures
- **Runtime (Late Binding):** Method Overriding — child class overrides a parent class method with the `virtual`/`override` keywords

```apex
// Method Overloading (Compile-time polymorphism)
public void add(Integer a, Integer b)                      { System.debug(a+b); }
public void add(String s1, String s2)                      { System.debug(s1+s2); }
public Integer add(Integer a, Integer b, Integer c)        { return a+b+c; }
```

---

## 3. Access Modifiers

Access modifiers define the visibility and accessibility of classes, methods, variables, and other components within a Salesforce org.

| Modifier | Description |
|----------|-------------|
| **`private`** | Accessible only within the class where defined. Default for class members if not specified. Test classes can be declared private. |
| **`public`** | Accessible from any class or trigger within the same org. Most common modifier for business logic. |
| **`protected`** | Accessible within the same class AND any subclasses (child classes). Cannot be used on classes themselves — only on members. |
| **`global`** | Accessible from anywhere, including outside the org. Required for: Batch classes, Scheduled classes, WebServices, APIs, Managed Packages. |

> 📌 If no access modifier is specified for a variable inside a class, Apex treats it as **`private`** by default.

### Naming Conventions for Classes

- Use **Pascal Case**: first character uppercase (e.g. `AccountService`, `CustomerHelper`)
- Class name must be a single word — no spaces
- Can include alphanumeric characters and underscores
- Never name a class the same as a Salesforce standard object

---

## 4. Data Types in Apex

### 4.1 Primitive Data Types

Built into the Apex language. They cannot be broken down further. Passed **by value**, not by reference — modifying a copy does NOT affect the original.

| Data Type | Salesforce Field | Description |
|-----------|-----------------|-------------|
| **`String`** | Text, Email, Phone, URL, Picklist | Sequence of characters enclosed in single quotes. Each character = 1 byte. |
| **`Integer`** | Number, Percent, Currency | Whole number, no decimal. 4 bytes. Range: -2^31 to +2^31-1. |
| **`Long`** | Large Number | Larger integer. 8 bytes. Range: -2^63 to +2^63-1. |
| **`Double`** | Number (decimal) | Numbers with decimal digits. Range: -2^63 to +2^63-1. |
| **`Decimal`** | Currency, Percent | Number with decimal, up to 18 digits. More precise than Double. |
| **`Boolean`** | Checkbox | True or false values only. |
| **`Date`** | Date field | Represents a calendar date. |
| **`Time`** | Time field | Represents a specific time (12 or 24 hour). |
| **`DateTime`** | Created/Modified Date | Date + Time combined. |
| **`ID`** | ID, Lookup, Master-Detail | 15 or 18 character Salesforce record ID. |
| **`Blob`** | Document, Attachment | Binary Large Objects — stores images, files, attachments. |

```apex
// Variable Declaration & Assignment
// <DataType> <VariableName> = <Value>;
Integer myAge       = 25;
Decimal mySalary    = 75000.50;
String myName       = 'John Doe';           // Always single quotes
Boolean isActive    = true;
Date joinDate       = Date.today();
DateTime createdAt  = DateTime.now();
ID accountId        = '0012w00001Pz63PAAR';
Blob fileContent    = Blob.valueOf('Hello World');
```

---

### 4.2 Non-Primitive (Complex) Data Types

Non-primitive types are also called complex types. They are passed **by reference** — modifying the parameter DOES affect the original object.

- **`sObject`** — Standard objects (Account, Contact) and Custom objects
- **`List`** — Ordered collection, allows duplicates, indexed access
- **`Set`** — Unordered collection, no duplicates, no index access
- **`Map`** — Key-Value pair collection, keys are unique
- **`Apex Class`** — Custom user-defined types

> ℹ️ **Key difference:** Primitives are passed by **VALUE** (copy). Non-primitives like sObjects and collections are passed by **REFERENCE** (same memory location).

---

## 5. Variables — Instance vs Static vs Constants

### 5.1 Instance (Object) Variables

Declared without the `static` keyword. Memory allocated each time a new object is created. Each object holds its own copy of the value.

```apex
public class Animal {
    public String name = 'Max';  // Instance variable
    Integer age = 5;             // private by default
}

Animal dog = new Animal();       // dog.name = 'Max'
Animal cat = new Animal();       // cat.name = 'Max'
cat.name = 'Lucy';               // only cat is affected
```

### 5.2 Static (Class) Variables

Declared with the `static` keyword. Memory allocated only **ONCE** when the class is first loaded. Shared across all objects of the class.

```apex
public class Animal {
    public static String address = '123 Main Street';
}

// Access via class name, not object
Animal.address = '456 New Street';   // changes for ALL objects
System.debug(Animal.address);
```

### 5.3 Constants (`final` keyword)

Constants are declared using the `final` keyword. Their value **cannot** be changed after initialization.

```apex
public class MyConstants {
    public static final Decimal PI_VALUE      = 3.14159;
    public static final String DEFAULT_NAME   = 'John Doe';
    public static final Decimal GRAVITY_VALUE;   // declared, set later in static block
}

// Attempting to reassign will throw a compile error:
// MyConstants.PI_VALUE = 3;  ← ERROR
```

---

## 6. Methods (Behaviours)

A method is a block of code that performs a specific task. Methods can calculate values, manipulate data, or interact with the Salesforce platform.

```apex
// Syntax
<Access Modifier> <Return Type> <Method Name> (<Optional Parameters>) {
    // Method Body
}

// Example: Static method with return
public static Integer calculateSum(Integer a, Integer b) {
    return a + b;
}

// Example: Instance method void (no return)
public void speak() {
    System.debug(name + ' is speaking');
}
```

### Instance vs Static Methods

| Type | Description |
|------|-------------|
| **Instance Method** | Belongs to an object. Called via object reference: `dog.speak()`. Can access instance AND static variables. |
| **Static Method** | Belongs to the class. Called via class name: `Animal.greet()`. Can only access static variables — NOT instance variables. |

### Method Overloading *(same name, different parameters)*

```apex
public void add(String s1, String s2)              { System.debug(s1+' '+s2); }
public Integer add(Integer a, Integer b)           { return a + b; }
public Integer add(Integer a, Integer b, Integer c){ return a+b+c; }

// Same name 'add' — Apex resolves based on parameter count/type
```

---

## 7. Constructors

A constructor is a special method called automatically when an object is created with `new`. It initialises the object's state.

- Constructor name **MUST** match the class name exactly
- No return type (not even `void`)
- Parent class constructor must always be `public`
- If no constructor is defined, Salesforce provides a default no-argument constructor

### Default vs Parameterized Constructors

```apex
public class Laptops {
    public String companyName;
    public String processor;
    public Integer ramSize;

    // Default constructor (no params)
    public Laptops() { }

    // Parameterized — 1 param
    public Laptops(String companyName) {
        this.companyName = companyName;
    }

    // Parameterized — 3 params
    public Laptops(String companyName, String processor, Integer ramSize) {
        this.companyName = companyName;
        this.processor   = processor;
        this.ramSize     = ramSize;
    }
}

// Usage
Laptops apple = new Laptops('Apple');
Laptops dell  = new Laptops('Dell', 'Intel Core Ultra 7', 16);
```

> 📌 The `this` keyword refers to the current object instance. Use `this.variableName` when the constructor parameter name shadows the instance variable name.

### Instance Block vs Static Block

```apex
public class MyClass {
    static String name = 'Apple';

    static {
        // STATIC BLOCK: runs ONCE when class is first loaded into memory
        // Used to initialize static variables
        System.debug('Static block: ' + name);
    }

    {
        // INSTANCE BLOCK: runs every time an object is created
        // Runs BEFORE the constructor
        System.debug('Instance block executed');
    }

    public MyClass() {
        System.debug('Constructor executed');
    }
}

// Order on new MyClass(): Static block → Instance block → Constructor
```

---

## 8. Getter / Setter (Properties)

Getters retrieve the value of a private field. Setters update it. They enforce encapsulation — other classes can't directly modify private data.

```apex
public class AccountDetails {
    private String accountName;

    // Traditional getter/setter
    public String getName()              { return this.accountName; }
    public void setName(String name)     { this.accountName = name; }

    // Shorthand property syntax
    public String rating  { get; private set; }   // public read, private write
    public String partner { get; set; }            // fully public
    public String typex   { private get; set; }   // private read, public write
}

AccountDetails acc = new AccountDetails();
acc.setName('Acme Corp');
String name = acc.getName();
acc.rating = 'Hot';              // ERROR — private set
System.debug(acc.rating);        // OK — public get
```

---

## 9. Collections — List, Set, Map

### 9.1 List

An ordered collection of elements accessible by index. Allows duplicates. Most commonly used for storing SOQL query results.

```apex
// Declaration
List<String> fruitList = new List<String>();
List<String> fruitList = new List<String>{'Apple','Mango','Banana'};

// Key Methods
fruitList.add('Orange');          // add to end
fruitList.set(1, 'Guava');        // replace at index 1
String f = fruitList.get(0);      // get at index 0 = 'Apple'
fruitList.remove(2);              // remove at index 2
Integer sz = fruitList.size();    // total elements
Boolean empty = fruitList.isEmpty();
fruitList.sort();                 // sort alphabetically
fruitList.clear();                // remove all

// Index formula: Last Index = size() - 1
```

> ✓ **Use List when:** ordering matters, index access needed, storing SOQL results, or creating/updating multiple sObject records with DML.

---

### 9.2 Set

An unordered collection that automatically eliminates duplicates. No index access. Faster for membership checks than List.

```apex
Set<String> colorsSet = new Set<String>{'Red','Blue','Green','Red'};
// Result: {Red, Blue, Green} — duplicate 'Red' removed

// Key Methods
colorsSet.add('Yellow');
Boolean has = colorsSet.contains('Red');     // true
colorsSet.remove('Blue');
colorsSet.addAll(anotherSetOrList);
Integer sz = colorsSet.size();
```

> ✓ **Use Set when:** uniqueness matters (e.g. storing record IDs, emails, phone numbers), or when you need fast duplicate checking.

---

### 9.3 Map

Stores data as key-value pairs. Keys are unique (like a Set). Values can be duplicates. Extremely powerful for Apex development.

```apex
// Simple Map
Map<String, String> countryCurrencies = new Map<String, String>();
countryCurrencies.put('India', 'INR');
countryCurrencies.put('Japan', 'YEN');
countryCurrencies.put('USA', 'USD');

String currency   = countryCurrencies.get('India');         // 'INR'
Boolean exists    = countryCurrencies.containsKey('USA');   // true
Set<String> keys  = countryCurrencies.keySet();
List<String> vals = countryCurrencies.values();

// Complex Map (Map of Lists)
Map<String, List<String>> countryStates = new Map<String, List<String>>();
List<String> indiaStates = new List<String>{'UP','Delhi','MH','MP'};
countryStates.put('India', indiaStates);
```

### Collection Comparison Table

| Feature | List | Set | Map |
|---------|:----:|:---:|:---:|
| **Ordering** | Ordered (indexed) | Unordered | Unordered |
| **Duplicates** | Allowed | Not allowed | Keys: No, Values: Yes |
| **Index access** | Yes (`get`/`set` by index) | No | By key (`get`/`containsKey`) |
| **DML use** | Required for insert/update | No (convert to List) | No (get values as List) |
| **Best for** | SOQL results, ordered data | Unique IDs, emails | Key-value lookups |

---

## 10. Conditional Statements

### 10.1 `if` / `else if` / `else`

```apex
String fruit = 'banana';

if (fruit == 'apple') {
    System.debug('This is an apple');
} else if (fruit == 'orange') {
    System.debug('This is an orange');
} else {
    System.debug('Other fruit: ' + fruit);
}
```

### 10.2 `switch on`

```apex
String fruit = 'banana';

switch on fruit {
    when 'apple'  { System.debug('Apple'); }
    when 'orange' { System.debug('Orange'); }
    when 'banana' { System.debug('Banana'); }
    when else     { System.debug('Unknown fruit'); }
}
```

### 10.3 Ternary Operator

```apex
Integer i = 5;
String msg  = i > 5 ? 'Greater than 5' : 'Less than or equal to 5';

// Nested ternary
String msg2 = i > 5 ? 'Greater' : i == 5 ? 'Equal' : 'Less';
```

---

## 11. Loops

### 11.1 Traditional For Loop

```apex
List<String> fruits = new List<String>{'Apple','Banana','Mango'};

for (Integer i = 0; i < fruits.size(); i++) {
    System.debug(fruits.get(i));
}

// Fastest approach: store size() before loop to avoid recalculation
Integer len = fruits.size();
for (Integer i = 0; i < len; i++) { System.debug(fruits.get(i)); }
```

### 11.2 Enhanced For Loop *(List/Set Iteration)*

```apex
List<String> fruits = new List<String>{'Apple','Banana','Mango'};

for (String fruit : fruits) {
    System.debug(fruit);
}

// Iterating over a Map
Map<String, Integer> emp = new Map<String, Integer>{'Acme'=>238, 'XYZ'=>500};

for (String key : emp.keySet()) {
    System.debug(key + ' => ' + emp.get(key));
}
```

### 11.3 While Loop

```apex
Integer i = 0;

while (i < fruits.size()) {
    System.debug(fruits.get(i));
    i++;   // CRITICAL: without increment, infinite loop!
}
```

### 11.4 Do-While Loop

```apex
// Executes body AT LEAST ONCE, checks condition after
Integer i = 0;

do {
    System.debug(fruits.get(i));
    i++;
} while (i < fruits.size());
```

---

## 12. Operators in Apex

| Category | Examples & Description |
|----------|----------------------|
| **Arithmetic** | `+`, `-`, `*`, `/`, `Math.Mod()`. Example: `'Welcome'+25000` = `'Welcome25000'` |
| **Comparison** | `==`, `!=`, `<`, `>`, `<=`, `>=`. Note: `==` is text comparison; `.equals()` is binary comparison |
| **Logical** | `&&` (AND), `\|\|` (OR), `!` (NOT) |
| **Assignment** | `=`, `+=`, `-=`, `*=`, `/=`. Example: `x += 5` is same as `x = x + 5` |
| **Increment/Decrement** | `++` (add 1), `--` (subtract 1). Example: `customerAge++` → 33 if was 32 |
| **String Concat** | `+` joins strings. `+=` concatenates and assigns |
| **Ternary** | `condition ? valueIfTrue : valueIfFalse` |
| **Safe Navigation** | `?.` — skips null pointer exceptions: `account?.Contact?.Name` |

---

## 13. Data Type Conversion

Apex does **not** do implicit type conversion. You must explicitly cast or use static methods on wrapper classes.

```apex
// String → Integer
String numStr  = '8998';
Integer num    = Integer.valueOf(numStr);

// Integer → String
Integer age    = 25;
String ageStr  = String.valueOf(age);

// String → Boolean
String flag    = 'true';
Boolean b      = Boolean.valueOf(flag);

// String → Decimal
Decimal d      = Decimal.valueOf('99.5');

// Date operations
Date today     = Date.today();
DateTime now   = DateTime.now();
```

---

## 14. Memory — Heap, Stack & Call Stack

| Memory Type | Description |
|-------------|-------------|
| **Heap Memory** | Stores variables and objects. All instances created with `new` keyword go here. Sync limit: 6 MB, Async limit: 12 MB. |
| **Stack Memory** | Stores method frames during execution. LIFO (Last In, First Out). Each method call pushes a frame. |

When a method is called, a frame is pushed onto the stack with its arguments and local variables. When the method returns, the frame is popped and execution resumes from where it left off.

```apex
// Call Stack Order: A → B → C → D → E
// Stack: [A frame][B frame][C frame][D frame][E frame]
// E executes, returns → D resumes → C resumes → B resumes → A resumes

CallStackDemo demo = new CallStackDemo();
demo.methodA();   // calls B → calls C → calls D → calls E
// Output: E, D, C, B, A (LIFO)
```

---

## 15. Inner Class (Wrapper Class)

A class defined inside another class. Used to group related data together — very common in Apex for building structured objects to pass between LWC/VF and Apex.

```apex
public class AccountDetails {
    // Outer class variables...

    // Inner (wrapper) class — private
    private class AccountWrapper {
        private String accountName;
        private Integer noOfEmployees;
    }

    public void testWrapper() {
        AccountDetails.AccountWrapper w = new AccountDetails.AccountWrapper();
        w.accountName = 'Acme Corp';
        System.debug(w);
    }
}

// Accessing inner class from outside
AccountDetails acc = new AccountDetails();
acc.testWrapper();
```

---

## 16. Interview Questions — 5 Years Experience Level

> **INTERVIEW GUIDE**
> These questions test depth expected from a mid-to-senior Salesforce Apex developer at 4–6 years of experience.

---

### OOP & Class Design

---

**Q1. What are the four pillars of OOP and how does Apex implement each?**

**Ans:** **Encapsulation:** Apex uses access modifiers (`private`/`public`/`protected`/`global`) and getter-setter patterns to wrap data and behaviour in classes. **Abstraction:** Achieved via interfaces and abstract classes — callers see method signatures, not implementation. **Inheritance:** `extends` keyword allows a child class to acquire all parent class variables and methods. **Polymorphism:** Method overloading (same name, different params) is compile-time polymorphism; method overriding with `virtual`/`override` is runtime polymorphism.

---

**Q2. What is the difference between `private`, `public`, `protected`, and `global` access modifiers?**

**Ans:** `private`: accessible only within the same class. Default for class members. `public`: accessible across all classes in the org. `protected`: accessible within the class and its subclasses — cannot be applied to the class itself. `global`: accessible from anywhere including outside the org — required for Batch, Scheduled, WebService, and API classes.

---

**Q3. What is the difference between instance variables and static variables?**

**Ans:** **Instance variables:** memory allocated each time a new object is created using `new`. Each object has its own copy. Accessed via object reference (`dog.name`). **Static variables:** memory allocated only ONCE when the class is loaded. Shared across all objects. Accessed via class name (`Animal.address`). Changing a static variable through one object changes it for ALL.

---

**Q4. When would you use Constructor Overloading in Apex?**

**Ans:** Constructor overloading lets callers create objects in different ways depending on what data they have available. For example, a `PaymentProcessor` class might have `PaymentProcessor()` for default setup, `PaymentProcessor(String cardType)` for card payments, and `PaymentProcessor(String cardType, Decimal amount)` for a fully initialised processor. It also enforces that mandatory fields must be provided — if a constructor requires certain params, the caller cannot skip them.

---

**Q5. What is the execution order when a new object is created?**

**Ans:** Order: (1) **Static block** — runs ONCE when the class is first loaded into memory. (2) **Instance block** — runs every time a new object is created, before the constructor. (3) **Constructor** — runs after the instance block. So: `Static block → Instance block → Constructor`.

---

### Data Types & Variables

---

**Q6. What is the difference between primitive and non-primitive data types in Apex?**

**Ans:** Primitives (`Integer`, `String`, `Boolean`, `Date`, etc.) are passed by **VALUE** — a copy is made. Modifying the parameter inside a method does NOT affect the original. Non-primitives (sObjects, `List`, `Set`, `Map`, Apex classes) are passed by **REFERENCE** — they point to the same memory. Modifying them inside a method DOES affect the original object.

---

**Q7. What is the difference between `Integer`, `Long`, `Double`, and `Decimal`?**

**Ans:** `Integer`: 4 bytes, no decimal, range ±2^31. `Long`: 8 bytes, no decimal, range ±2^63 — for very large numbers. `Double`: 8 bytes, with decimal digits — imprecise floating point. `Decimal`: up to 18 digits with decimal — used for currency and financial calculations because it is more precise than Double. In Salesforce, Currency and Percent fields map to `Decimal`.

---

**Q8. What does the `final` keyword do in Apex?**

**Ans:** The `final` keyword declares a constant — a variable whose value cannot be changed after initialization. Typically combined with `static` for class-level constants: `public static final Decimal PI = 3.14159`. Attempting to reassign a `final` variable throws a compile error. Use constants to avoid magic numbers, ensure consistency, and make code more readable and maintainable.

---

### Collections

---

**Q9. When would you choose a `List` over a `Set` or `Map`?**

**Ans:** Use `List` when: ordering matters, you need index-based access (`get(0)`), you need to store SOQL results (SOQL always returns a List), or you need to perform DML on multiple records. Use `Set` when: you need uniqueness guaranteed (e.g. a collection of Account IDs to avoid duplicates in SOQL `WHERE IN` clause). Use `Map` when: you need fast O(1) lookup by key — e.g. `Map<Id, Account>` to avoid nested SOQL loops inside triggers.

---

**Q10. How is a Map used to avoid SOQL inside a loop (governor limit violation)?**

**Ans:** The pattern: query once before the loop and store results in a `Map<Id, SObject>`. Then inside the loop, use `map.get(id)` instead of a SOQL query.

```apex
Map<Id, Account> accMap = new Map<Id, Account>(
    [SELECT Id, Name FROM Account WHERE Id IN :idSet]
);

for (Contact c : contacts) {
    Account acc = accMap.get(c.AccountId);
}
```

This reduces 1000 SOQL queries inside a loop to just **1**.

---

**Q11. What is the maximum number of elements a `List` can hold in Apex?**

**Ans:** A List can hold up to **50,000 elements** in a single Apex transaction (based on the heap size governor limit). However, the SOQL query governor limit of 50,000 rows applies when populating a List from a query. Using `Database.QueryLocator` in Batch Apex bypasses this to allow up to 50 million records.

---

### OOP Design & Advanced

---

**Q12. What is a Wrapper Class and when do you use it in Salesforce?**

**Ans:** A wrapper class (inner class) is a custom class defined inside another class to bundle multiple fields together into a single object. Common use cases: (1) **LWC-to-Apex data transfer** — an `@AuraEnabled` method returning a wrapper with multiple related fields. (2) **Displaying mixed data** in a VF page table (e.g. a row with an Account and a related Contact). (3) **Batch processing** — passing structured data between `execute()` chunks.

```apex
public class AccountWrapper {
    public Account acc;
    public Boolean isSelected;
}
```

---

**Q13. What is method overriding and how do you implement it in Apex?**

**Ans:** Method overriding is runtime polymorphism — a child class provides its own implementation of a method defined in the parent class. In Apex: the parent method must be declared with the `virtual` keyword; the child class uses the `override` keyword.

```apex
// Parent:
public virtual void speak()  { System.debug('Generic animal'); }

// Child:
public override void speak() { System.debug('Dog barks'); }
```

Without `virtual`/`override` keywords, Apex does not allow overriding.

---

**Q14. What is the difference between `==` and `.equals()` in Apex?**

**Ans:** `==` performs a text (value) comparison for Strings and primitives. `.equals()` performs a binary comparison. For Strings in Apex, both usually give the same result, but `.equals()` is more precise for case-sensitive binary matching. For sObjects and collections, `==` checks if they reference the same object in memory, while `.equals()` checks value equality. Best practice: use `.equals()` for String comparisons to be explicit.

---

**Q15. Explain Heap Memory vs Stack Memory and their governor limits.**

**Ans:** **Heap Memory** stores variables, objects, and collections created during execution. Synchronous transactions have a **6 MB** heap limit; asynchronous (Batch, Future, Queueable) have a **12 MB** limit. Exceeding this throws `'System.LimitException: Apex heap size too large'`. **Stack Memory** stores method call frames in a LIFO structure. Salesforce limits the call stack to prevent infinite recursion — maximum call stack depth is around **1,000 levels**. Exceeding it throws `'Maximum stack depth reached'`.

---

## 17. Quick Reference Cheat Sheet

### Complete Class Template

```apex
public class MyService {

    // Static (class) variable — shared across all objects
    public static final String VERSION  = '1.0';
    private static Integer instanceCount = 0;

    // Instance variables
    private String name;
    public Integer age { get; private set; }   // shorthand property

    // Static block — runs once when class loads
    static { System.debug('Class loaded: ' + VERSION); }

    // Instance block — runs before constructor
    { instanceCount++; }

    // Default constructor
    public MyService() { this.name = 'Default'; }

    // Parameterized constructor
    public MyService(String name, Integer age) {
        this.name = name;
        this.age  = age;
    }

    // Instance method
    public String getName()         { return this.name; }
    public void setName(String n)   { this.name = n; }

    // Static method
    public static Integer getCount() { return instanceCount; }

    // Inner wrapper class
    public class ResultWrapper {
        public Boolean isSuccess;
        public String message;
    }
}
```

---

### Key Governor Limits to Remember

| Limit | Value |
|-------|-------|
| **SOQL queries per transaction** | 100 (sync), 200 (async) |
| **SOQL rows returned** | 50,000 |
| **DML statements per transaction** | 150 |
| **DML rows processed** | 10,000 |
| **Heap memory (sync)** | 6 MB |
| **Heap memory (async)** | 12 MB |
| **CPU time (sync)** | 10,000 ms |
| **CPU time (async)** | 60,000 ms |
| **Max call stack depth** | ~1,000 levels |
| **Max object instantiations** | 3,000 per transaction |

---
