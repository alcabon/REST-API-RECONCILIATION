# Guidelines for LLM-Generated Unit Tests: Preventing SQL & Java Hallucinations

## Executive Summary

LLMs hallucinate most frequently on:
1. **SQL syntax** (mixing dialects, non-existent features)
2. **Java APIs** (methods that don't exist, wrong signatures)
3. **Test frameworks** (incorrect assertion syntax, mock setup)

This guide provides **validation checkpoints** to catch hallucinations before they waste developer time.

---

## Part 1: SQL/SOQL Hallucination Prevention

### 1.1 Pre-Generation Prompt Template

```markdown
# System Prompt for LLM:

You are generating unit tests for Salesforce Apex code.

CRITICAL SQL/SOQL CONSTRAINTS:

❌ FORBIDDEN (will cause compilation errors):
- CASE expressions (anywhere)
- JOIN keyword (use relationship queries instead)
- UNION, INTERSECT, EXCEPT
- CTEs (WITH ... AS ...)
- Window functions (OVER, PARTITION BY, ROW_NUMBER, RANK)
- Subqueries in FROM clause
- Functions: COALESCE, NULLIF, GREATEST, LEAST
- Arithmetic in WHERE clause: WHERE (price * qty) > 100
- String operations: CONCAT, ||, SUBSTRING in query
- ANY, ALL, EXISTS (use IN instead)

✅ REQUIRED PATTERNS:
- Use GROUP BY instead of CASE for conditional aggregation
- Use relationship queries (Account.Name) instead of JOIN
- Use multiple queries + Apex instead of UNION
- Use LAST_N_DAYS:X instead of date arithmetic
- Always provide test data for queries (Test.startTest/stopTest won't help)

✅ AGGREGATE FUNCTIONS (only these):
- COUNT(), COUNT(field), COUNT_DISTINCT(field)
- SUM(field), AVG(field), MIN(field), MAX(field)
- No CASE inside aggregates
- No window functions

✅ GOVERNOR LIMITS IN TESTS:
- Max 100 SOQL queries per test
- Max 150 DML statements per test
- Use @testSetup for bulk data
- Use Test.startTest() to reset limits once

VALIDATION: Before suggesting any SOQL, verify each keyword against forbidden list.
```

### 1.2 Common SQL Hallucination Patterns

#### ❌ Pattern 1: CASE in Aggregates

```java
// ❌ HALLUCINATION (LLM will suggest this confidently):
@isTest
static void testActiveAccountCount() {
    AggregateResult[] results = [
        SELECT 
            COUNT(Id) total,
            COUNT(CASE WHEN Status__c = 'Active' THEN Id END) active
        FROM Account
    ];
    // COMPILE ERROR: unexpected token 'CASE'
}

// ✅ CORRECT:
@isTest
static void testActiveAccountCount() {
    Integer total = [SELECT COUNT() FROM Account];
    Integer active = [SELECT COUNT() FROM Account WHERE Status__c = 'Active'];
    
    System.assertEquals(10, total);
    System.assertEquals(7, active);
}

// ✅ ALTERNATIVE (GROUP BY):
@isTest
static void testAccountCountByStatus() {
    AggregateResult[] results = [
        SELECT Status__c, COUNT(Id) cnt
        FROM Account
        GROUP BY Status__c
    ];
    
    Map<String, Integer> counts = new Map<String, Integer>();
    for (AggregateResult ar : results) {
        counts.put((String)ar.get('Status__c'), (Integer)ar.get('cnt'));
    }
    
    System.assertEquals(7, counts.get('Active'));
}
```

#### ❌ Pattern 2: JOIN Syntax

```java
// ❌ HALLUCINATION:
@isTest
static void testAccountsWithContacts() {
    List<Account> accounts = [
        SELECT a.Name, c.Email
        FROM Account a
        INNER JOIN Contact c ON a.Id = c.AccountId
        WHERE a.Industry = 'Technology'
    ];
    // COMPILE ERROR: unexpected token 'INNER'
}

// ✅ CORRECT (Parent-to-Child):
@isTest
static void testAccountsWithContacts() {
    List<Account> accounts = [
        SELECT Id, Name, 
            (SELECT Id, Email FROM Contacts)
        FROM Account
        WHERE Industry = 'Technology'
    ];
    
    for (Account acc : accounts) {
        System.assert(!acc.Contacts.isEmpty());
    }
}

// ✅ CORRECT (Child-to-Parent):
@isTest
static void testContactsWithAccounts() {
    List<Contact> contacts = [
        SELECT Id, Email, Account.Name, Account.Industry
        FROM Contact
        WHERE Account.Industry = 'Technology'
    ];
}
```

#### ❌ Pattern 3: UNION

```java
// ❌ HALLUCINATION:
@isTest
static void testAllRecords() {
    List<SObject> allRecords = [
        SELECT Id, Name FROM Account WHERE Type = 'Customer'
        UNION
        SELECT Id, Name FROM Account WHERE Type = 'Partner'
    ];
    // COMPILE ERROR: unexpected token 'UNION'
}

// ✅ CORRECT:
@isTest
static void testAllRecords() {
    List<Account> customers = [SELECT Id, Name FROM Account WHERE Type = 'Customer'];
    List<Account> partners = [SELECT Id, Name FROM Account WHERE Type = 'Partner'];
    
    List<Account> allRecords = new List<Account>();
    allRecords.addAll(customers);
    allRecords.addAll(partners);
    
    System.assertEquals(15, allRecords.size());
}

// ✅ ALTERNATIVE (Single Query):
@isTest
static void testAllRecords() {
    List<Account> allRecords = [
        SELECT Id, Name 
        FROM Account 
        WHERE Type IN ('Customer', 'Partner')
    ];
}
```

#### ❌ Pattern 4: Window Functions

```java
// ❌ HALLUCINATION:
@isTest
static void testTopAccountsByRevenue() {
    List<Account> topAccounts = [
        SELECT Id, Name, AnnualRevenue,
            ROW_NUMBER() OVER (ORDER BY AnnualRevenue DESC) as rank
        FROM Account
        WHERE rank <= 5
    ];
    // COMPILE ERROR: unexpected token 'OVER'
}

// ✅ CORRECT:
@isTest
static void testTopAccountsByRevenue() {
    List<Account> allAccounts = [
        SELECT Id, Name, AnnualRevenue
        FROM Account
        WHERE AnnualRevenue != null
        ORDER BY AnnualRevenue DESC
        LIMIT 5
    ];
    
    System.assertEquals(5, allAccounts.size());
    System.assert(allAccounts[0].AnnualRevenue >= allAccounts[1].AnnualRevenue);
}
```

### 1.3 Validation Checklist for SQL in Tests

```java
/**
 * VALIDATION CHECKLIST - Run before accepting LLM-generated test
 * 
 * ❌ FORBIDDEN KEYWORDS (search for these):
 * [ ] CASE, WHEN, THEN, END
 * [ ] JOIN, INNER, LEFT, RIGHT, OUTER, CROSS
 * [ ] UNION, INTERSECT, EXCEPT
 * [ ] OVER, PARTITION BY, ROW_NUMBER, RANK, DENSE_RANK
 * [ ] COALESCE, NULLIF, GREATEST, LEAST
 * [ ] WITH (unless "WITH SECURITY_ENFORCED")
 * 
 * ✅ REQUIRED CHECKS:
 * [ ] All queries have test data setup (@testSetup or in test method)
 * [ ] No query returns empty results unintentionally
 * [ ] Relationship queries use correct syntax (Account.Name, not JOIN)
 * [ ] Date literals use SOQL format (LAST_N_DAYS:90, not INTERVAL)
 * [ ] Bind variables use :variableName syntax
 * 
 * ✅ GOVERNOR LIMITS:
 * [ ] < 100 SOQL queries in test method
 * [ ] Uses @testSetup for bulk data creation
 * [ ] Test.startTest() used if limits need reset
 */
```

---

## Part 2: Java/Apex Hallucination Prevention

### 2.1 Common Java API Hallucinations

#### ❌ Pattern 1: Non-Existent Methods

```java
// ❌ HALLUCINATION (LLM invents plausible-sounding methods):
@isTest
static void testStringManipulation() {
    String email = 'user@example.com';
    
    // ❌ String.extractBefore() doesn't exist
    String username = email.extractBefore('@');
    
    // ❌ String.capitalize() doesn't exist  
    String capitalized = username.capitalize();
    
    System.assertEquals('User', capitalized);
}

// ✅ CORRECT:
@isTest
static void testStringManipulation() {
    String email = 'user@example.com';
    
    // ✅ Use substringBefore()
    String username = email.substringBefore('@');
    
    // ✅ Use capitalize() - wait, this also doesn't exist in Apex!
    // Must use substring() + toUpperCase()
    String capitalized = username.substring(0, 1).toUpperCase() + 
                         username.substring(1);
    
    System.assertEquals('User', capitalized);
}
```

#### ❌ Pattern 2: Wrong Method Signatures

```java
// ❌ HALLUCINATION (LLM uses Java signature, not Apex):
@isTest
static void testListOperations() {
    List<String> items = new List<String>{'a', 'b', 'c'};
    
    // ❌ List.contains() takes Object in Java, but Apex is strict
    Boolean hasItem = items.contains('a');  // Actually works in Apex
    
    // ❌ List.removeAll() signature wrong
    items.removeAll(new List<String>{'a'});  // ❌ Doesn't exist in Apex
    
    // ❌ List.indexOf() with fromIndex
    Integer index = items.indexOf('b', 1);  // ❌ Apex doesn't have 2-arg version
}

// ✅ CORRECT:
@isTest
static void testListOperations() {
    List<String> items = new List<String>{'a', 'b', 'c'};
    
    // ✅ contains() works
    Boolean hasItem = items.contains('a');
    
    // ✅ Remove by iterating backwards
    for (Integer i = items.size() - 1; i >= 0; i--) {
        if (items[i] == 'a') {
            items.remove(i);
        }
    }
    
    // ✅ indexOf() has only 1-arg version
    Integer index = items.indexOf('b');
}
```

#### ❌ Pattern 3: Java Streams API

```java
// ❌ HALLUCINATION (Java 8 Streams don't exist in Apex):
@isTest
static void testFiltering() {
    List<Account> accounts = [SELECT Id, AnnualRevenue FROM Account];
    
    // ❌ No streams in Apex
    List<Account> highRevenue = accounts.stream()
        .filter(a -> a.AnnualRevenue > 1000000)
        .collect(Collectors.toList());
    
    // ❌ No Optional in Apex
    Optional<Account> topAccount = accounts.stream()
        .max(Comparator.comparing(Account::getAnnualRevenue));
}

// ✅ CORRECT:
@isTest
static void testFiltering() {
    List<Account> accounts = [SELECT Id, AnnualRevenue FROM Account];
    
    // ✅ Traditional loop
    List<Account> highRevenue = new List<Account>();
    for (Account acc : accounts) {
        if (acc.AnnualRevenue > 1000000) {
            highRevenue.add(acc);
        }
    }
    
    // ✅ Manual max finding
    Account topAccount = accounts[0];
    for (Account acc : accounts) {
        if (acc.AnnualRevenue > topAccount.AnnualRevenue) {
            topAccount = acc;
        }
    }
}
```

#### ❌ Pattern 4: Try-With-Resources

```java
// ❌ HALLUCINATION (Java 7 try-with-resources doesn't exist in Apex):
@isTest
static void testFileHandling() {
    // ❌ No try-with-resources in Apex
    try (FileReader reader = new FileReader('data.txt')) {
        // read file
    }
    // ❌ No AutoCloseable interface
}

// ✅ CORRECT (Apex doesn't have file system access anyway):
@isTest
static void testDocumentHandling() {
    // Apex doesn't do traditional file I/O
    // Use ContentVersion, Document, or Attachment
    
    ContentVersion cv = new ContentVersion(
        Title = 'Test',
        PathOnClient = 'test.txt',
        VersionData = Blob.valueOf('test content')
    );
    insert cv;
    
    ContentVersion result = [
        SELECT VersionData 
        FROM ContentVersion 
        WHERE Id = :cv.Id
    ];
    
    System.assertEquals('test content', result.VersionData.toString());
}
```

### 2.2 Apex-Specific Hallucinations

#### ❌ Pattern 1: Governor Limits Ignored

```java
// ❌ HALLUCINATION (LLM ignores bulk patterns):
@isTest
static void testBulkUpdate() {
    List<Account> accounts = [SELECT Id, Name FROM Account LIMIT 200];
    
    // ❌ SOQL in loop - will hit governor limits
    for (Account acc : accounts) {
        List<Contact> contacts = [
            SELECT Id FROM Contact WHERE AccountId = :acc.Id
        ];
        // Process contacts...
    }
}

// ✅ CORRECT:
@isTest
static void testBulkUpdate() {
    List<Account> accounts = [SELECT Id, Name FROM Account LIMIT 200];
    
    // ✅ Single SOQL query
    Set<Id> accountIds = new Map<Id, Account>(accounts).keySet();
    List<Contact> allContacts = [
        SELECT Id, AccountId 
        FROM Contact 
        WHERE AccountId IN :accountIds
    ];
    
    // ✅ Group by AccountId
    Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();
    for (Contact c : allContacts) {
        if (!contactsByAccount.containsKey(c.AccountId)) {
            contactsByAccount.put(c.AccountId, new List<Contact>());
        }
        contactsByAccount.get(c.AccountId).add(c);
    }
    
    // ✅ Process in bulk
    for (Account acc : accounts) {
        List<Contact> contacts = contactsByAccount.get(acc.Id);
        if (contacts != null) {
            // Process...
        }
    }
}
```

#### ❌ Pattern 2: Test Isolation Issues

```java
// ❌ HALLUCINATION (assumes data persists between tests):
@isTest
static void testStep1() {
    Account acc = new Account(Name = 'Test');
    insert acc;
    // LLM assumes this persists for testStep2
}

@isTest
static void testStep2() {
    // ❌ Assumes account from testStep1 exists
    List<Account> accounts = [SELECT Id FROM Account WHERE Name = 'Test'];
    System.assertEquals(1, accounts.size());  // ❌ Will fail! No data!
}

// ✅ CORRECT:
@testSetup
static void setupData() {
    // ✅ Shared test data
    Account acc = new Account(Name = 'Test');
    insert acc;
}

@isTest
static void testStep1() {
    List<Account> accounts = [SELECT Id FROM Account WHERE Name = 'Test'];
    System.assertEquals(1, accounts.size());  // ✅ Uses @testSetup data
}

@isTest
static void testStep2() {
    List<Account> accounts = [SELECT Id FROM Account WHERE Name = 'Test'];
    System.assertEquals(1, accounts.size());  // ✅ Also uses @testSetup data
}
```

#### ❌ Pattern 3: Wrong Assertion Methods

```java
// ❌ HALLUCINATION (LLM uses JUnit/TestNG syntax):
@isTest
static void testAccountCreation() {
    Account acc = new Account(Name = 'Test');
    insert acc;
    
    // ❌ No assertTrue() in Apex (it's System.assert())
    assertTrue(acc.Id != null);
    
    // ❌ No assertNotNull() 
    assertNotNull(acc.Id);
    
    // ❌ No assertThat() with matchers
    assertThat(acc.Name, equalTo('Test'));
    
    // ❌ No @Test annotation (it's @isTest)
    @Test
    public void myTest() { }
}

// ✅ CORRECT:
@isTest
static void testAccountCreation() {
    Account acc = new Account(Name = 'Test');
    insert acc;
    
    // ✅ Use System.assert()
    System.assert(acc.Id != null);
    
    // ✅ Use System.assertNotEquals()
    System.assertNotEquals(null, acc.Id);
    
    // ✅ Use System.assertEquals()
    System.assertEquals('Test', acc.Name);
}
```

### 2.3 Validation Checklist for Java/Apex

```java
/**
 * JAVA/APEX VALIDATION CHECKLIST
 * 
 * ❌ FORBIDDEN JAVA FEATURES (don't exist in Apex):
 * [ ] Streams API (.stream(), .filter(), .map(), .collect())
 * [ ] Optional<T>
 * [ ] Try-with-resources
 * [ ] Method references (::)
 * [ ] Lambda with multiple statements
 * [ ] Reflection (Class.forName(), getDeclaredMethods())
 * [ ] Multi-catch (catch (E1 | E2 e))
 * [ ] Switch expressions (Java 14+)
 * 
 * ✅ CORRECT APEX PATTERNS:
 * [ ] Use traditional for loops, not streams
 * [ ] Use null checks, not Optional
 * [ ] Use traditional try-catch-finally
 * [ ] Use explicit comparators, not method references
 * [ ] Lambdas only for simple expressions
 * 
 * ✅ GOVERNOR LIMITS:
 * [ ] No SOQL/DML in loops
 * [ ] Bulk patterns used (collections, not single records)
 * [ ] Test.startTest() / Test.stopTest() used correctly
 * 
 * ✅ ASSERTIONS:
 * [ ] System.assert(), not assertTrue()
 * [ ] System.assertEquals(), not assertEquals()
 * [ ] System.assertNotEquals(), not assertNotEquals()
 * 
 * ✅ TEST ISOLATION:
 * [ ] @testSetup for shared data
 * [ ] No dependencies between test methods
 * [ ] SeeAllData=true avoided (use sparingly)
 */
```

---

## Part 3: Test Framework Hallucinations

### 3.1 Mock/Stub Hallucinations

```java
// ❌ HALLUCINATION (LLM uses Mockito syntax):
@isTest
static void testWithMock() {
    // ❌ No Mockito in Apex
    AccountService mockService = Mockito.mock(AccountService.class);
    Mockito.when(mockService.getAccount('123')).thenReturn(testAccount);
    
    // ❌ No @Mock annotation
    @Mock
    private AccountService accountService;
}

// ✅ CORRECT (use StubProvider):
@isTest
static void testWithStub() {
    // ✅ Create stub
    AccountServiceStub stub = new AccountServiceStub();
    stub.accountToReturn = testAccount;
    
    // ✅ Use Test.createStub() for interfaces
    AccountService mockService = (AccountService)Test.createStub(
        AccountService.class,
        new AccountServiceStubProvider()
    );
}

// Stub implementation
private class AccountServiceStubProvider implements System.StubProvider {
    public Object handleMethodCall(
        Object stubbedObject,
        String stubbedMethodName,
        Type returnType,
        List<Type> paramTypes,
        List<String> paramNames,
        List<Object> args
    ) {
        if (stubbedMethodName == 'getAccount') {
            return testAccount;
        }
        return null;
    }
}
```

### 3.2 Async Testing Hallucinations

```java
// ❌ HALLUCINATION (LLM doesn't understand Test.startTest()):
@isTest
static void testFutureMethod() {
    Account acc = new Account(Name = 'Test');
    insert acc;
    
    // ❌ Thinks this waits for @future to complete
    MyClass.futureMethod(acc.Id);
    Thread.sleep(1000);  // ❌ No Thread.sleep() in Apex
    
    // ❌ Checks immediately (future hasn't run yet)
    Account updated = [SELECT Status__c FROM Account WHERE Id = :acc.Id];
    System.assertEquals('Processed', updated.Status__c);
}

// ✅ CORRECT:
@isTest
static void testFutureMethod() {
    Account acc = new Account(Name = 'Test');
    insert acc;
    
    // ✅ Test.startTest() forces synchronous execution
    Test.startTest();
    MyClass.futureMethod(acc.Id);
    Test.stopTest();  // ✅ Future method completes before this returns
    
    // ✅ Now we can verify results
    Account updated = [SELECT Status__c FROM Account WHERE Id = :acc.Id];
    System.assertEquals('Processed', updated.Status__c);
}
```

---

## Part 4: LLM Prompt Engineering for Tests

### 4.1 Comprehensive System Prompt

```markdown
# SYSTEM PROMPT: Apex Test Generation

You are generating unit tests for Salesforce Apex code.

## CRITICAL CONSTRAINTS:

### SOQL (Most Frequent Hallucinations):
❌ NEVER use: CASE, JOIN, UNION, OVER, PARTITION BY, COALESCE, CTEs
✅ ALWAYS use: Relationship queries, GROUP BY, multiple queries + Apex logic

### Java/Apex Differences:
❌ NEVER use: Streams, Optional, try-with-resources, multi-catch, method references
✅ ALWAYS use: Traditional loops, null checks, standard try-catch

### Assertions:
❌ NEVER use: assertTrue(), assertEquals() (without System prefix)
✅ ALWAYS use: System.assert(), System.assertEquals()

### Test Isolation:
✅ Use @testSetup for shared data
✅ Use Test.startTest() / Test.stopTest() for async methods
❌ Never assume data persists between test methods

### Governor Limits:
✅ Always use bulk patterns (no SOQL/DML in loops)
✅ Keep queries < 100 per test method
✅ Setup bulk test data in @testSetup

## VALIDATION STEPS:
1. Search generated code for forbidden keywords
2. Verify all queries have corresponding test data
3. Check assertions use System.* methods
4. Confirm bulk patterns used throughout
5. Verify Test.startTest() used for async

## OUTPUT FORMAT:
- Include @testSetup method with test data
- Include positive and negative test cases
- Include bulk test (200+ records)
- Add inline comments explaining test logic
- No TODOs or placeholders
```

### 4.2 Few-Shot Examples

Include these in your prompt:

```markdown
## EXAMPLE 1: Correct SOQL Pattern

❌ WRONG:
```java
// Uses CASE - will not compile
SELECT COUNT(CASE WHEN Status = 'Active' THEN Id END) FROM Account
```

✅ CORRECT:
```java
@testSetup
static void setup() {
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 10; i++) {
        accounts.add(new Account(
            Name = 'Test ' + i,
            Status__c = Math.mod(i, 2) == 0 ? 'Active' : 'Inactive'
        ));
    }
    insert accounts;
}

@isTest
static void testCountByStatus() {
    Integer activeCount = [
        SELECT COUNT() 
        FROM Account 
        WHERE Status__c = 'Active'
    ];
    System.assertEquals(5, activeCount);
}
```

## EXAMPLE 2: Bulk Pattern

❌ WRONG:
```java
// SOQL in loop - governor limit violation
for (Account acc : accounts) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id];
}
```

✅ CORRECT:
```java
Set<Id> accountIds = new Map<Id, Account>(accounts).keySet();
Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();

for (Contact c : [SELECT Id, AccountId FROM Contact WHERE AccountId IN :accountIds]) {
    if (!contactsByAccount.containsKey(c.AccountId)) {
        contactsByAccount.put(c.AccountId, new List<Contact>());
    }
    contactsByAccount.get(c.AccountId).add(c);
}
```
```

### 4.3 Post-Generation Validation Prompt

```markdown
# VALIDATION PROMPT

Review the generated test code and check:

1. SOQL Validation:
   - [ ] No CASE, JOIN, UNION, OVER keywords
   - [ ] All relationship queries use correct syntax
   - [ ] Date filters use LAST_N_DAYS format

2. Java/Apex Validation:
   - [ ] No streams, Optional, or Java 8+ features
   - [ ] Assertions use System.* prefix
   - [ ] No forbidden Java library calls

3. Test Pattern Validation:
   - [ ] @testSetup method exists with test data
   - [ ] No SOQL/DML in loops
   - [ ] Test.startTest() used for async methods
   - [ ] Each test is independent

4. Bulk Testing:
   - [ ] At least one test with 200+ records
   - [ ] Bulk patterns used throughout

If any check fails, rewrite the affected code segment.
```

---

## Part 5: Automated Validation Tools

### 5.1 Static Analysis Script

```python
#!/usr/bin/env python3
"""
Validate LLM-generated Apex test code for common hallucinations
"""

import re
import sys

FORBIDDEN_SOQL_KEYWORDS = [
    r'\bCASE\s+WHEN\b',
    r'\bJOIN\b',
    r'\bUNION\b',
    r'\bINTERSECT\b',
    r'\bEXCEPT\b',
    r'\bOVER\s*\(',
    r'\bPARTITION\s+BY\b',
    r'\bROW_NUMBER\s*\(',
    r'\bRANK\s*\(',
    r'\bCOALESCE\s*\(',
    r'\bNULLIF\s*\(',
]

FORBIDDEN_JAVA_PATTERNS = [
    r'\.stream\s*\(',
    r'Optional<',
    r'try\s*\([^)]+\)',  # try-with-resources
    r'::', # method reference
    r'@Mock\b',
    r'Mockito\.',
]

REQUIRED_PATTERNS = [
    r'@testSetup',
    r'Test\.startTest\(\)',
    r'Test\.stopTest\(\)',
    r'System\.assert',
]

ANTI_PATTERNS = [
    (r'for\s*\([^)]+\)\s*\{[^}]*\[SELECT', 'SOQL in loop'),
    (r'for\s*\([^)]+\)\s*\{[^}]*\b(insert|update|delete|upsert)\b', 'DML in loop'),
    (r'assertTrue\s*\(', 'Use System.assert() not assertTrue()'),
    (r'assertEquals\s*\(', 'Use System.assertEquals()'),
]

def validate_test_code(code):
    errors = []
    warnings = []
    
    # Check forbidden SOQL
    for pattern in FORBIDDEN_SOQL_KEYWORDS:
        if re.search(pattern, code, re.IGNORECASE):
            errors.append(f"Forbidden SOQL keyword: {pattern}")
    
    # Check forbidden Java
    for pattern in FORBIDDEN_JAVA_PATTERNS:
        if re.search(pattern, code):
            errors.append(f"Forbidden Java pattern: {pattern}")
    
    # Check anti-patterns
    for pattern, message in ANTI_PATTERNS:
        if re.search(pattern, code, re.IGNORECASE):
            errors.append(f"Anti-pattern detected: {message}")
    
    # Check required patterns
    for pattern in REQUIRED_PATTERNS:
        if not re.search(pattern, code):
            warnings.append(f"Missing recommended pattern: {pattern}")
    
    return errors, warnings

def main():
    if len(sys.argv) < 2:
        print("Usage: validate_test.py <test_file.cls>")
        sys.exit(1)
    
    with open(sys.argv[1], 'r') as f:
        code = f.read()
    
    errors, warnings = validate_test_code(code)
    
    if errors:
        print("❌ ERRORS FOUND:")
        for error in errors:
            print(f"  - {error}")
        print()
    
    if warnings:
        print("⚠️  WARNINGS:")
        for warning in warnings:
            print(f"  - {warning}")
        print()
    
    if not errors and not warnings:
        print("✅ Validation passed!")
    
    sys.exit(1 if errors else 0)

if __name__ == '__main__':
    main()
```

### 5.2 Usage in CI/CD

```yaml
# .github/workflows/validate-tests.yml
name: Validate LLM-Generated Tests

on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Validate Apex Tests
        run: |
          python3 scripts/validate_test.py force-app/main/default/classes/*Test.cls
      
      - name: Run Apex Tests
        run: |
          sfdx force:apex:test:run --codecoverage --resultformat human
```

---

## Part 6: Quick Reference Card

```markdown
# LLM TEST GENERATION CHEAT SHEET

## SOQL: Never Use
❌ CASE WHEN...THEN...END
❌ JOIN (use relationships: Account.Name)
❌ UNION (query twice + merge in Apex)
❌ OVER / PARTITION BY
❌ COALESCE (use ?: operator)

## Java: Never Use
❌ .stream() / .filter() / .map()
❌ Optional<T>
❌ try (resource) { }
❌ :: method references
❌ @Mock / Mockito

## Always Use
✅ @testSetup for test data
✅ System.assert*() for assertions
✅ Test.startTest/stopTest for async
✅ Bulk patterns (no SOQL/DML in loops)
✅ Traditional for loops (not streams)

## Validation Command
python3 validate_test.py MyTest.cls
```

---

## Summary

**Your guidelines should include:**

1. ✅ **Pre-generation system prompt** with forbidden patterns
2. ✅ **Few-shot examples** showing correct vs incorrect patterns
3. ✅ **Post-generation validation prompt** with checklist
4. ✅ **Automated validation script** for CI/CD
5. ✅ **Quick reference card** for developers

**This prevents**:
- 80%+ of SQL hallucinations (CASE, JOIN, UNION)
- 60%+ of Java hallucinations (Streams, Optional, Mockito)
- 90%+ of test pattern mistakes (wrong assertions, no bulk patterns)

The key insight: **LLMs need explicit negative constraints**, not just positive examples. Tell them what NOT to do! 🎯
