# SOQL vs SQL ANSI: Grammar Analysis (Expert Comparison)

## SQL Standard I Reference Most

I primarily reference **SQL:2003** (with SQL:2011 features) as the "modern baseline":
- SQL-92: Foundation (most widely supported)
- SQL:1999: Added CTEs, basic OLAP
- **SQL:2003**: Window functions, MERGE ⭐ (my typical reference)
- SQL:2011: Temporal data
- PostgreSQL 12+: Reference implementation with excellent ANSI compliance

---

## Major Grammar Differences: SOQL vs SQL

### 1. ❌ **No CASE Expressions** (CRITICAL)

**SQL**:
```sql
SELECT 
  CASE 
    WHEN status = 'active' THEN 1 
    ELSE 0 
  END as active_flag,
  SUM(CASE WHEN status = 'active' THEN amount ELSE 0 END) as active_sum
FROM accounts;
```

**SOQL Grammar**: 
```
NO CASE construct exists anywhere in the grammar
```

**Impact**: Cannot do conditional logic in queries at all. Must use multiple queries or Apex code.

---

### 2. ❌ **No JOIN Syntax** (MASSIVE)

**SQL**:
```sql
SELECT a.name, c.email
FROM accounts a
  INNER JOIN contacts c ON a.id = c.account_id
  LEFT JOIN opportunities o ON a.id = o.account_id
WHERE o.amount > 1000;
```

**SOQL Grammar**:
```antlr
fromNameList
    : fieldName soqlId? (COMMA fieldName soqlId?)*;
```
No JOIN keyword exists. Uses relationship traversal instead:

**SOQL**:
```sql
-- Implicit join via relationship
SELECT Id, Account.Name, Account.Owner.Email 
FROM Contact

-- Can't do: arbitrary joins, outer joins, cross joins, theta joins
```

**Impact**: 
- ✅ Elegant for standard parent-child relationships
- ❌ Cannot join arbitrary tables
- ❌ Cannot do complex multi-table queries
- ❌ No self-joins

---

### 3. ❌ **No Common Table Expressions (CTEs)**

**SQL:2003**:
```sql
WITH regional_sales AS (
  SELECT region, SUM(amount) as total
  FROM orders
  GROUP BY region
),
top_regions AS (
  SELECT region FROM regional_sales WHERE total > 1000000
)
SELECT * FROM orders WHERE region IN (SELECT region FROM top_regions);
```

**SOQL Grammar**:
```antlr
withClause
    : WITH DATA CATEGORY filteringExpression
    | WITH SECURITY_ENFORCED
    | WITH SYSTEM_MODE
    | WITH USER_MODE
    | WITH logicalExpression;
```

**This is NOT a CTE!** `WITH` in SOQL is for:
- Data category filtering
- Security context
- NOT for temporary named result sets

**Impact**: Cannot build complex hierarchical queries or improve readability with CTEs.

---

### 4. ❌ **No Window Functions / OVER Clause**

**SQL:2003**:
```sql
SELECT 
  name,
  amount,
  ROW_NUMBER() OVER (PARTITION BY region ORDER BY amount DESC) as rank,
  SUM(amount) OVER (PARTITION BY region) as region_total,
  LAG(amount) OVER (ORDER BY date) as prev_amount
FROM sales;
```

**SOQL**: Not in grammar. No `OVER`, no `PARTITION BY`, no analytical functions.

**Impact**: Cannot do:
- Running totals
- Row numbering
- Rankings
- Moving averages
- Lead/lag comparisons

---

### 5. ❌ **No Subqueries in FROM (Derived Tables)**

**SQL**:
```sql
SELECT avg_amount 
FROM (
  SELECT AVG(amount) as avg_amount 
  FROM orders 
  GROUP BY customer_id
) AS subquery;
```

**SOQL Grammar**:
```antlr
subQuery
    : SELECT subFieldList
        FROM fromNameList  // <- Just table names, no subqueries here
        whereClause?
        ...
```

Subqueries only allowed in:
- SELECT (relationship queries)
- WHERE with IN

**Impact**: Cannot create temporary result sets for complex analysis.

---

### 6. ❌ **No Set Operations (UNION/INTERSECT/EXCEPT)**

**SQL**:
```sql
SELECT id FROM accounts WHERE status = 'active'
UNION
SELECT id FROM accounts WHERE created_date = TODAY;

SELECT id FROM accounts WHERE region = 'EMEA'
INTERSECT
SELECT id FROM accounts WHERE amount > 10000;
```

**SOQL**: Not in grammar at all.

**Impact**: Cannot combine multiple query results. Must do in Apex code.

---

### 7. ✅ **Date Literals** (SOQL BETTER THAN SQL!)

**SQL** (verbose):
```sql
WHERE created_date >= CURRENT_DATE - INTERVAL '90 days'
WHERE created_date >= DATE_TRUNC('quarter', CURRENT_DATE)
```

**SOQL Grammar** (elegant):
```antlr
dateFormula
    : YESTERDAY | TODAY | TOMORROW
    | LAST_WEEK | THIS_WEEK | NEXT_WEEK
    | LAST_90_DAYS | NEXT_90_DAYS
    | LAST_N_DAYS_N COLON signedInteger
    | THIS_QUARTER | LAST_QUARTER
    | THIS_FISCAL_YEAR | LAST_FISCAL_YEAR
    ...
```

**SOQL**:
```sql
WHERE CreatedDate = LAST_90_DAYS
WHERE CreatedDate = THIS_FISCAL_QUARTER
WHERE CloseDate = NEXT_N_WEEKS_N:4
```

**Impact**: ✅ Much cleaner for business date logic!

---

### 8. ✅ **Relationship Queries** (SOQL UNIQUE)

**SOQL Grammar**:
```antlr
selectEntry
    : ...
    | LPAREN subQuery RPAREN soqlId?  // Parent-to-Child
```

**SOQL**:
```sql
-- Parent-to-child (one-to-many)
SELECT Id, Name, (SELECT FirstName, LastName FROM Contacts) 
FROM Account

-- Child-to-parent (many-to-one) 
SELECT Id, FirstName, Account.Name, Account.Owner.Name
FROM Contact
```

**SQL Equivalent**: Requires explicit JOINs or subqueries.

**Impact**: ✅ Very elegant for Salesforce data model. ❌ Limited to defined relationships only.

---

### 9. ✅ **TYPEOF for Polymorphic Fields** (SOQL UNIQUE)

**SOQL Grammar**:
```antlr
typeOf
    : TYPEOF fieldName whenClause+ elseClause? END;

whenClause
    : WHEN fieldName THEN fieldNameList;
```

**SOQL**:
```sql
SELECT 
  TYPEOF What
    WHEN Account THEN Name, Phone
    WHEN Opportunity THEN Name, Amount
  END
FROM Task
```

**SQL**: No direct equivalent. Would need CASE with type checking.

**Impact**: ✅ Handles polymorphic relationships elegantly.

---

### 10. ⚠️ **Special Operators**

**SOQL Grammar**:
```antlr
comparisonOperator
    : ASSIGN | NOTEQUAL | LT | GT | LT ASSIGN | GT ASSIGN | LESSANDGREATER 
    | LIKE | IN | NOT IN 
    | INCLUDES | EXCLUDES;  // <- SOQL-specific
```

**INCLUDES/EXCLUDES**: For multi-select picklists
```sql
-- SOQL-specific for multi-select
WHERE Skills__c INCLUDES ('Java', 'Python')
WHERE Skills__c EXCLUDES ('COBOL')
```

**SQL**: Would need complex string parsing with `LIKE '%Java%'` patterns.

---

### 11. ❌ **Limited Aggregate Functions**

**SQL:2003 has**:
```sql
VARIANCE, STDDEV, MEDIAN, PERCENTILE_CONT, PERCENTILE_DISC
STRING_AGG, ARRAY_AGG, JSON_AGG
EVERY, BOOL_AND, BOOL_OR
```

**SOQL Grammar has**:
```antlr
soqlFunction
    : AVG | COUNT | COUNT_DISTINCT | MIN | MAX | SUM
    | TOLABEL | FORMAT
    | GROUPING | CONVERT_CURRENCY
    | CALENDAR_MONTH | FISCAL_QUARTER | ...
```

**Impact**: 
- ❌ No statistical functions
- ❌ No array/JSON aggregation
- ✅ But has fiscal calendar functions (useful!)

---

### 12. ❌ **No Arithmetic in WHERE**

**SQL**:
```sql
WHERE (price * quantity * (1 - discount)) > 1000
WHERE SQRT(x*x + y*y) < 100
```

**SOQL Grammar**:
```antlr
fieldExpression
    : fieldName comparisonOperator value
    | soqlFunction comparisonOperator value;
```

Very restrictive! Only `field OPERATOR value`, no complex expressions.

**Impact**: Cannot do calculations in WHERE. Must use formula fields or filter in Apex.

---

### 13. ⚠️ **Bound Variables**

**SQL** (various syntaxes):
```sql
-- JDBC
SELECT * FROM accounts WHERE id = ?

-- PostgreSQL
SELECT * FROM accounts WHERE id = $1

-- Named parameters (some systems)
SELECT * FROM accounts WHERE id = :id
```

**SOQL Grammar**:
```antlr
boundExpression
    : COLON expression;
```

**SOQL**:
```apex
String accountName = 'ACME';
List<Account> accounts = [SELECT Id FROM Account WHERE Name = :accountName];
```

**Impact**: ✅ Cleaner syntax with named binds. Similar to Oracle SQL.

---

### 14. ✅ **Geolocation** (SOQL BETTER)

**SOQL Grammar**:
```antlr
| DISTANCE LPAREN locationValue COMMA locationValue COMMA StringLiteral RPAREN
...
locationValue
    : fieldName
    | boundExpression
    | GEOLOCATION LPAREN coordinateValue COMMA coordinateValue RPAREN
```

**SOQL**:
```sql
SELECT Name, 
  DISTANCE(Location__c, GEOLOCATION(37.775, -122.418), 'mi') dist
FROM Account
WHERE DISTANCE(Location__c, GEOLOCATION(37.775, -122.418), 'mi') < 10
ORDER BY dist
```

**SQL**: Requires PostGIS or similar extensions. Not in ANSI standard.

**Impact**: ✅ Built-in geospatial is convenient.

---

### 15. ✅ **Security Context** (SOQL UNIQUE)

**SOQL Grammar**:
```antlr
withClause
    : ...
    | WITH SECURITY_ENFORCED
    | WITH SYSTEM_MODE
    | WITH USER_MODE
```

**SOQL**:
```sql
SELECT Id FROM Account WITH SECURITY_ENFORCED
SELECT Id FROM Account WITH USER_MODE
```

**SQL**: No equivalent. Security is database-level, not query-level.

**Impact**: ✅ Excellent for multi-tenant security.

---

### 16. ⚠️ **GROUP BY Restrictions**

**SQL**:
```sql
-- Can group by complex expressions
SELECT EXTRACT(YEAR FROM date), UPPER(region), COUNT(*)
FROM sales
GROUP BY EXTRACT(YEAR FROM date), UPPER(region);

-- Can reference aliases
SELECT region as r, COUNT(*) 
FROM sales 
GROUP BY r;
```

**SOQL Grammar**:
```antlr
groupByClause
    : GROUP BY selectList (HAVING logicalExpression)?
    | GROUP BY ROLLUP LPAREN fieldName (COMMA fieldName)* RPAREN
    | GROUP BY CUBE LPAREN fieldName (COMMA fieldName)* RPAREN;
```

**SOQL**:
```sql
-- Can only group by fields, not expressions
SELECT Status__c, COUNT(Id)
FROM Account
GROUP BY Status__c

-- ❌ Cannot: GROUP BY CALENDAR_YEAR(CreatedDate) directly
-- Must use in SELECT:
SELECT CALENDAR_YEAR(CreatedDate) year, COUNT(Id)
FROM Account
GROUP BY CALENDAR_YEAR(CreatedDate)
```

**Impact**: Less flexible than SQL.

---

### 17. ✅ **FIELDS() Function** (SOQL UNIQUE)

**SOQL Grammar**:
```antlr
| FIELDS LPAREN soqlFieldsParameter RPAREN

soqlFieldsParameter
    : ALL | CUSTOM | STANDARD;
```

**SOQL**:
```sql
SELECT FIELDS(ALL) FROM Account LIMIT 200
SELECT FIELDS(CUSTOM) FROM Account
SELECT FIELDS(STANDARD) FROM Account
```

**SQL**: Must list all columns explicitly or use `*`.

**Impact**: ✅ Convenient for development/debugging.

---

### 18. ✅ **ALL ROWS** (SOQL UNIQUE)

**SOQL Grammar**:
```antlr
allRowsClause
    : ALL ROWS;
```

**SOQL**:
```sql
SELECT Id FROM Account WHERE IsDeleted = true ALL ROWS
```

Includes deleted/archived records in query.

**SQL**: Would use different tables or schemas for soft-deleted data.

---

## Summary Table: Feature Comparison

| Feature | SQL:2003 | SOQL | Winner |
|---------|----------|------|--------|
| **CASE expressions** | ✅ | ❌ | SQL |
| **JOINs** | ✅ All types | ❌ Relationship only | SQL |
| **CTEs (WITH)** | ✅ | ❌ | SQL |
| **Window functions** | ✅ | ❌ | SQL |
| **Subquery in FROM** | ✅ | ❌ | SQL |
| **UNION/INTERSECT** | ✅ | ❌ | SQL |
| **Arithmetic expressions** | ✅ | ❌ Limited | SQL |
| **Statistical functions** | ✅ | ❌ | SQL |
| **Date literals** | ⚠️ Verbose | ✅ Elegant | **SOQL** |
| **Relationship queries** | ⚠️ Manual JOIN | ✅ Built-in | **SOQL** |
| **TYPEOF (polymorphic)** | ❌ | ✅ | **SOQL** |
| **Geolocation** | ⚠️ Extension | ✅ Built-in | **SOQL** |
| **Security context** | ❌ | ✅ | **SOQL** |
| **Multi-select operators** | ❌ | ✅ INCLUDES/EXCLUDES | **SOQL** |
| **Fiscal calendar** | ❌ | ✅ | **SOQL** |

---

## Philosophy Difference

### SQL (ANSI)
- **General-purpose** query language
- **Set-based** operations (relational algebra)
- **Turing-complete** with extensions (PL/SQL, PL/pgSQL)
- **Flexible** - many ways to solve problems

### SOQL
- **Domain-specific** for Salesforce object model
- **Relationship-based** (object graph traversal)
- **Query-only** (no DML in SOQL)
- **Constrained** - optimized for multi-tenant governor limits

---

## Why These Restrictions?

**Governor Limits**: Salesforce multi-tenant architecture requires:
1. ❌ No JOINs → Prevents Cartesian explosion
2. ❌ No CTEs → Prevents recursive queries consuming resources
3. ❌ No complex WHERE → CPU time limits
4. ✅ Relationship queries → Optimized for object model
5. ✅ Date formulas → Common business patterns, pre-optimized

---

## Conclusion

**SOQL is approximately SQL-89 level** with:
- ✅ Modern conveniences (date formulas, relationships, geolocation)
- ❌ No SQL-92 features (JOINs, subqueries in FROM)
- ❌ No SQL:1999 features (CTEs)
- ❌ No SQL:2003 features (window functions, MERGE)

**SOQL = "SQL Lite" + Salesforce-specific superpowers**

It's **not inferior** - it's **purpose-built** for:
- Multi-tenant scalability
- Object-relational mapping
- Governor limit compliance
- Developer productivity (date formulas, relationships)

But for **complex analytical queries**, you must work around limitations in Apex, which is why we ended up with reconciliation frameworks doing multiple queries! 🎯

