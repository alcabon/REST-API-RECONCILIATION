# REST API Design Prerequisites for Reconciliation

## Critical Requirements for Reconciliation-Friendly APIs

As a REST API design expert, here are the **mandatory** capabilities your Master API must provide:

---

## 1. Individual Resource Retrieval (MUST HAVE)

### Endpoint Pattern
```http
GET /api/v1/accounts/{id}
```

### Required Response Structure
```json
{
  // === IDENTITY ===
  "id": "acc_xyz789",                    // ✅ MUST: Stable, immutable ID
  "external_id": "SF-001ABC",            // ✅ SHOULD: For correlation
  
  // === VERSIONING ===
  "version": 42,                          // ✅ MUST: Monotonic integer
  "etag": "a7f6c3e9d2b1a8f4",            // ✅ SHOULD: Hash for comparison
  
  // === TIMESTAMPS ===
  "created_at": "2024-01-15T10:00:00Z",  // ✅ MUST: ISO 8601 UTC
  "updated_at": "2024-02-15T14:30:00Z",  // ✅ MUST: ISO 8601 UTC
  "deleted_at": null,                     // ✅ MUST: For soft deletes
  
  // === DATA ===
  "name": "ACME Corp",
  "email": "contact@acme.com",
  "status": "active",
  
  // === METADATA ===
  "updated_by": "user_123",               // ✅ SHOULD: Audit trail
  
  // === CHECKSUM (OPTIONAL BUT RECOMMENDED) ===
  "_checksum": "sha256:a7f6c3e9..."      // ✅ SHOULD: Data integrity
}
```

### Why Each Field Matters

| Field | Purpose in Reconciliation | Example Usage |
|-------|---------------------------|---------------|
| **id** | Primary correlation key | Match Salesforce `ExternalId__c` |
| **version** | Detect which is newer | `if (remote.version > local.version)` |
| **updated_at** | Time-based queries | `?updated_after=2024-02-01T00:00:00Z` |
| **deleted_at** | Detect deletions | `if (deleted_at != null) { delete in SF }` |
| **_checksum** | Data integrity check | Verify data not corrupted |

---

## 2. Bulk Retrieval Endpoints (MUST HAVE)

### Why Needed?
Reconciliation needs to compare **thousands** of records. N+1 queries = disaster.

### Option A: List with Filtering (Minimum)

```http
GET /api/v1/accounts?updated_after=2024-02-01T00:00:00Z&limit=100
```

```json
{
  "data": [
    {
      "id": "acc_001",
      "version": 42,
      "updated_at": "2024-02-15T14:30:00Z",
      "name": "ACME Corp",
      // ... full object
    },
    {
      "id": "acc_002",
      // ...
    }
  ],
  "pagination": {
    "limit": 100,
    "next_cursor": "eyJpZCI6MTAwfQ==",
    "has_more": true,
    "total_count": 5847  // ✅ SHOULD: Total for progress tracking
  },
  "_links": {
    "next": "/api/v1/accounts?cursor=eyJpZCI6MTAwfQ==&limit=100"
  }
}
```

**Required Query Parameters**:
```
✅ MUST support:
- limit (pagination size)
- cursor OR offset (pagination mechanism)
- updated_after (time-based filtering)

✅ SHOULD support:
- updated_before (time range)
- created_after, created_before
- ids[] (bulk fetch by IDs)
- include_deleted (show soft-deleted)
```

### Option B: Bulk Fetch by IDs (Highly Recommended)

```http
POST /api/v1/accounts/bulk-get
Content-Type: application/json

{
  "ids": ["acc_001", "acc_002", "acc_003", ..., "acc_200"]
}
```

```json
{
  "data": [
    {
      "id": "acc_001",
      "version": 42,
      // ... full object
    },
    // ... up to 200 objects
  ],
  "errors": [
    {
      "id": "acc_999",
      "error": "not_found",
      "message": "Account not found"
    }
  ]
}
```

**Why This Matters**:
```apex
// ❌ WITHOUT bulk endpoint (N+1 problem)
for (Account acc : salesforceAccounts) {
    HttpResponse res = Http.send('GET /accounts/' + acc.ExternalId__c);
    // 10,000 accounts = 10,000 HTTP requests 😱
}

// ✅ WITH bulk endpoint
List<String> ids = extractIds(salesforceAccounts);
// Batch into chunks of 200
for (List<String> chunk : chunkIds(ids, 200)) {
    HttpResponse res = Http.send('POST /accounts/bulk-get', chunk);
    // 10,000 accounts = 50 HTTP requests ✅
}
```

---

## 3. Change Detection Endpoint (HIGHLY RECOMMENDED)

### Changes Feed API

```http
GET /api/v1/changes/accounts?since=2024-02-01T00:00:00Z&limit=100
```

```json
{
  "changes": [
    {
      "id": "change_001",
      "entity_type": "account",
      "entity_id": "acc_123",
      "operation": "UPDATE",                    // CREATE, UPDATE, DELETE
      "timestamp": "2024-02-15T14:30:00Z",
      "version": 42,
      "data": {
        "id": "acc_123",
        "name": "ACME Corp",
        // ... full object state after change
      }
    },
    {
      "id": "change_002",
      "entity_type": "account",
      "entity_id": "acc_456",
      "operation": "DELETE",
      "timestamp": "2024-02-15T15:00:00Z",
      "data": {
        "id": "acc_456",
        "deleted_at": "2024-02-15T15:00:00Z"
      }
    }
  ],
  "pagination": {
    "next_cursor": "eyJ0aW1lc3RhbXAiOiIyMDI0LTAyLTE1VDE1OjAwOjAwWiJ9"
  }
}
```

**This Enables**:
```apex
// Efficient reconciliation - only fetch what changed
global class SmartReconciliationBatch {
    
    global Database.QueryLocator start(Database.BatchableContext bc) {
        // Get last successful reconciliation time
        DateTime lastRecon = getLastReconciliationTime(); // e.g., 7 days ago
        
        // Call changes feed
        HttpResponse res = Http.send(
            'GET /api/v1/changes/accounts?since=' + lastRecon.formatGmt('yyyy-MM-dd\'T\'HH:mm:ss\'Z\'')
        );
        
        // Only reconcile changed records (not all 100K accounts)
        Set<String> changedIds = parseChangedIds(res);
        
        return Database.getQueryLocator([
            SELECT Id, ExternalId__c, Name, ExternalVersion__c
            FROM Account
            WHERE ExternalId__c IN :changedIds
        ]);
    }
}
```

**Performance Comparison**:
```
Without changes feed:
- Fetch all 100,000 accounts from API
- Compare all 100,000 records
- Time: ~30 minutes
- API calls: 500+

With changes feed:
- Fetch only 247 changed accounts
- Compare only 247 records  
- Time: ~30 seconds ✅
- API calls: 2-3
```

---

## 4. Consistent Field Schema (MUST HAVE)

### Problem: Inconsistent Schemas Break Reconciliation

```json
// ❌ BAD: Different fields in different endpoints
GET /accounts/123
{
  "id": "123",
  "name": "ACME",
  "email": "contact@acme.com"  // Email here
}

GET /accounts?limit=100
{
  "data": [
    {
      "id": "123",
      "name": "ACME",
      "email_address": "contact@acme.com"  // ❌ Different field name!
    }
  ]
}
```

```apex
// Reconciliation breaks
String remoteEmail = (String)remoteData.get('email');
String listEmail = (String)listData.get('email_address');
// Different keys = mismatch detected incorrectly
```

### ✅ Solution: Schema Consistency

**All endpoints return the same structure**:
```
GET /accounts/123        → Account schema v1
GET /accounts            → Array of Account schema v1
POST /accounts           → Returns Account schema v1
PUT /accounts/123        → Returns Account schema v1
GET /changes/accounts    → Returns Account schema v1
```

**Use OpenAPI/JSON Schema**:
```yaml
# openapi.yaml
components:
  schemas:
    Account:
      type: object
      required: [id, name, version, created_at, updated_at]
      properties:
        id:
          type: string
        name:
          type: string
        email:  # ✅ Same name everywhere
          type: string
        version:
          type: integer
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time
```

---

## 5. Soft Deletes (MUST HAVE)

### Why Hard Deletes Break Reconciliation

```apex
// Reconciliation flow
for (Account local : salesforceAccounts) {
    HttpResponse res = callAPI('GET', '/accounts/' + local.ExternalId__c);
    
    if (res.getStatusCode() == 404) {
        // ⚠️ AMBIGUITY:
        // - Was it deleted in master?
        // - Or is ExternalId__c wrong?
        // - Or temporary API issue?
        // Can't tell!
    }
}
```

### ✅ Solution: Soft Delete with `deleted_at`

```http
DELETE /api/v1/accounts/123

# Don't physically delete
# Instead, mark as deleted
```

```json
HTTP 200 OK
{
  "id": "acc_123",
  "name": "ACME Corp",
  "deleted": true,
  "deleted_at": "2024-02-15T14:30:00Z",
  "deleted_by": "user_456",
  "version": 43  // ✅ Version still increments
}
```

**GET still works**:
```http
GET /api/v1/accounts/123

HTTP 200 OK
{
  "id": "acc_123",
  "deleted": true,
  "deleted_at": "2024-02-15T14:30:00Z"
}
```

**List excludes by default**:
```http
GET /api/v1/accounts
# Returns only non-deleted accounts

GET /api/v1/accounts?include_deleted=true
# Returns all including deleted
```

**Reconciliation can handle properly**:
```apex
Map<String, Object> remote = parseResponse(res);

if ((Boolean)remote.get('deleted') == true) {
    // ✅ Clear: deleted in master
    delete [SELECT Id FROM Account WHERE ExternalId__c = :entityId];
} else {
    // ✅ Clear: still active, update it
    upsertAccount(remote);
}
```

---

## 6. Idempotent GET (MUST HAVE)

### GET Must Be Side-Effect Free

```http
# ❌ BAD: GET with side effects
GET /api/v1/accounts/123
# On server: incrementViewCount(123)
# Problem: Reconciliation calling GET 1000x inflates view count

# ✅ GOOD: GET is pure read
GET /api/v1/accounts/123
# No side effects, safe to call multiple times
```

**REST Principle**: GET, HEAD, OPTIONS must be **idempotent** and **safe**.

---

## 7. Rate Limiting Headers (SHOULD HAVE)

### Why: Reconciliation Makes Many Requests

```http
HTTP 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1708012800

{
  "data": [...]
}
```

```apex
// Reconciliation respects rate limits
global void execute(Database.BatchableContext bc, List<Account> scope) {
    
    HttpResponse res = callAPI('POST', '/accounts/bulk-get', scope);
    
    Integer remaining = Integer.valueOf(res.getHeader('X-RateLimit-Remaining'));
    
    if (remaining < 100) {
        // Slow down
        Integer resetTime = Integer.valueOf(res.getHeader('X-RateLimit-Reset'));
        Integer waitSeconds = resetTime - (System.now().getTime() / 1000);
        
        // Pause this batch job
        System.abortJob(bc.getJobId());
        
        // Reschedule for after reset
        System.scheduleBatch(
            new ReconciliationBatch(),
            'Reconciliation Resume',
            waitSeconds / 60  // Convert to minutes
        );
    }
}
```

---

## 8. Filtering & Sorting (MUST HAVE)

### Required Query Parameters

```http
# Time-based filtering (CRITICAL for reconciliation)
GET /api/v1/accounts?updated_after=2024-02-01T00:00:00Z
GET /api/v1/accounts?updated_before=2024-02-15T23:59:59Z
GET /api/v1/accounts?created_after=2024-01-01T00:00:00Z

# Status filtering
GET /api/v1/accounts?status=active
GET /api/v1/accounts?status=active,pending

# Sorting (for pagination consistency)
GET /api/v1/accounts?sort=updated_at
GET /api/v1/accounts?sort=-created_at  # Descending

# Deleted records
GET /api/v1/accounts?include_deleted=true
GET /api/v1/accounts?deleted_only=true
```

**Why This Matters**:
```apex
// Incremental reconciliation
DateTime lastRecon = System.now().addDays(-7);

// Only fetch records updated since last reconciliation
HttpResponse res = callAPI(
    'GET',
    '/accounts?updated_after=' + lastRecon.formatGmt('yyyy-MM-dd\'T\'HH:mm:ss\'Z\'') +
    '&limit=200'
);

// Without updated_after: Must fetch ALL 100K records every time 😱
// With updated_after: Fetch only ~500 changed records ✅
```

---

## 9. API Versioning (MUST HAVE)

### URL-Based Versioning

```http
GET /api/v1/accounts/123  # Current production
GET /api/v2/accounts/123  # New version (breaking changes)
```

### Header-Based Versioning (Alternative)

```http
GET /api/accounts/123
Accept: application/vnd.myapi.v1+json
```

### Why Critical for Reconciliation

```apex
// Salesforce code written for v1 schema
Map<String, Object> data = parseResponse(res);
String email = (String)data.get('email');  // v1 field name

// If API changes to v2 with different schema:
// v2 uses 'email_address' instead of 'email'
// Reconciliation breaks

// Solution: Pin to v1
HttpRequest req = new HttpRequest();
req.setEndpoint('https://api.example.com/api/v1/accounts/123');  // ✅ Explicit v1
```

---

## 10. Complete API Design Checklist

### Minimum Viable Reconciliation API

| Feature | Endpoint | Required? | Purpose |
|---------|----------|-----------|---------|
| **Individual GET** | `GET /accounts/{id}` | ✅ MUST | Fetch single record |
| **List with pagination** | `GET /accounts?limit=100` | ✅ MUST | Fetch multiple |
| **Time filtering** | `?updated_after=<timestamp>` | ✅ MUST | Incremental sync |
| **Versioning** | `version` field in response | ✅ MUST | Conflict detection |
| **Timestamps** | `updated_at`, `created_at` | ✅ MUST | Time-based queries |
| **Soft deletes** | `deleted_at` field | ✅ MUST | Deletion detection |
| **Bulk fetch** | `POST /accounts/bulk-get` | ✅ SHOULD | Performance |
| **Changes feed** | `GET /changes/accounts` | ✅ SHOULD | Efficiency |
| **Rate limit headers** | `X-RateLimit-*` | ✅ SHOULD | Avoid throttling |
| **Checksum** | `_checksum` field | ⚠️ OPTIONAL | Data integrity |

---

## 11. Example: Well-Designed Reconciliation API

### Complete API Contract

```yaml
openapi: 3.0.0
info:
  title: Master Data API
  version: 1.0.0

paths:
  /api/v1/accounts:
    get:
      summary: List accounts
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            default: 100
            maximum: 200
        - name: cursor
          in: query
          schema:
            type: string
        - name: updated_after
          in: query
          schema:
            type: string
            format: date-time
        - name: updated_before
          in: query
          schema:
            type: string
            format: date-time
        - name: include_deleted
          in: query
          schema:
            type: boolean
            default: false
      responses:
        '200':
          description: Success
          headers:
            X-RateLimit-Limit:
              schema:
                type: integer
            X-RateLimit-Remaining:
              schema:
                type: integer
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Account'
                  pagination:
                    $ref: '#/components/schemas/Pagination'

  /api/v1/accounts/{id}:
    get:
      summary: Get single account
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Account'
        '404':
          description: Not found (only for invalid IDs, not deletions)

  /api/v1/accounts/bulk-get:
    post:
      summary: Bulk fetch accounts
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                ids:
                  type: array
                  items:
                    type: string
                  maxItems: 200
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Account'
                  errors:
                    type: array
                    items:
                      type: object

  /api/v1/changes/accounts:
    get:
      summary: Get account changes
      parameters:
        - name: since
          in: query
          required: true
          schema:
            type: string
            format: date-time
        - name: limit
          in: query
          schema:
            type: integer
            default: 100
      responses:
        '200':
          description: Success

components:
  schemas:
    Account:
      type: object
      required:
        - id
        - version
        - created_at
        - updated_at
      properties:
        id:
          type: string
          example: "acc_xyz789"
        name:
          type: string
        email:
          type: string
          format: email
        status:
          type: string
          enum: [active, inactive, pending]
        version:
          type: integer
          minimum: 1
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time
        deleted_at:
          type: string
          format: date-time
          nullable: true
        _checksum:
          type: string
          example: "sha256:a7f6c3e9..."
    
    Pagination:
      type: object
      properties:
        limit:
          type: integer
        next_cursor:
          type: string
        has_more:
          type: boolean
        total_count:
          type: integer
```

---

## Summary: API Requirements Priority

### P0 - CRITICAL (Reconciliation Impossible Without)
```
✅ GET /accounts/{id} with full object
✅ version field (integer, monotonic)
✅ updated_at timestamp (ISO 8601 UTC)
✅ Soft deletes (deleted_at field)
✅ Pagination (cursor or offset)
```

### P1 - HIGH (Major Performance Impact)
```
✅ Bulk fetch endpoint (POST /accounts/bulk-get)
✅ Time-based filtering (?updated_after)
✅ Rate limit headers
✅ Consistent schema across endpoints
```

### P2 - MEDIUM (Quality of Life)
```
✅ Changes feed (GET /changes/accounts)
✅ Checksum field for validation
✅ Total count in pagination
✅ Explicit API versioning
```

If your Master API has **P0 + P1**, you can build a **production-grade reconciliation system**. Without P0, reconciliation is either **impossible** or **extremely inefficient**. 🎯
