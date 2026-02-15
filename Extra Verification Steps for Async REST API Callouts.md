# Extra Verification Steps for Async REST API Callouts (Non-Banking)

## Context
- ✅ Good reliability of remote master system
- ✅ Referential data (products, accounts, contacts, etc.)
- ✅ Async pattern chosen for UX performance
- ⚠️ Need verification without over-engineering

---

## 1. Essential Verification Steps (Minimum)

### 1.1 Immediate: Job Status Tracking

```apex
public class AsyncSyncJob implements Queueable, Database.AllowsCallouts {
    private Id recordId;
    
    public void execute(QueueableContext ctx) {
        
        // 1. Mark as PROCESSING
        updateSyncStatus(recordId, 'PROCESSING', ctx.getJobId());
        
        try {
            // 2. Callout
            HttpResponse res = callExternalAPI(recordId);
            
            // 3. Validate response
            if (res.getStatusCode() == 200) {
                Map<String, Object> data = parseResponse(res);
                
                // ✅ CRITICAL: Verify data integrity
                if (validateResponseData(data)) {
                    updateRecord(recordId, data);
                    updateSyncStatus(recordId, 'SYNCED', ctx.getJobId());
                } else {
                    // Data validation failed
                    updateSyncStatus(recordId, 'VALIDATION_FAILED', ctx.getJobId());
                    logValidationError(recordId, data);
                }
            } else {
                updateSyncStatus(recordId, 'API_ERROR', ctx.getJobId());
            }
            
        } catch (Exception e) {
            updateSyncStatus(recordId, 'ERROR', ctx.getJobId());
            logException(recordId, e);
        }
    }
    
    // ✅ Data validation before applying
    private Boolean validateResponseData(Map<String, Object> data) {
        // Check required fields exist
        if (!data.containsKey('id') || !data.containsKey('name')) {
            return false;
        }
        
        // Check data types
        if (!(data.get('id') instanceof String)) {
            return false;
        }
        
        // Check business rules
        String status = (String)data.get('status');
        if (!VALID_STATUSES.contains(status)) {
            return false;
        }
        
        // Version check (prevent overwriting newer data)
        Integer remoteVersion = (Integer)data.get('version');
        Integer localVersion = getCurrentVersion(recordId);
        if (remoteVersion <= localVersion) {
            return false; // Stale data
        }
        
        return true;
    }
}
```

### 1.2 Status Field on Record

```apex
// Account object fields
Account {
    SyncStatus__c          // Picklist: PENDING, PROCESSING, SYNCED, ERROR, VALIDATION_FAILED
    LastSyncAttempt__c     // DateTime
    LastSyncSuccess__c     // DateTime
    SyncJobId__c           // Text(18) - AsyncApexJob ID
    SyncErrorMessage__c    // Text(255)
    ExternalVersion__c     // Number - for optimistic locking
}
```

**User visibility**:
```
Record page layout:
┌────────────────────────────────────┐
│ Account: ACME Corp                 │
│                                    │
│ Sync Status: ⏳ Processing...     │
│ Last Sync: 2 minutes ago           │
│                                    │
│ [Retry Sync] button (if ERROR)    │
└────────────────────────────────────┘
```

---

## 2. Data Integrity Verification

### 2.1 Checksum Validation (Optional but Recommended)

```apex
public class DataIntegrityValidator {
    
    // Generate checksum from critical fields
    public static String calculateChecksum(Map<String, Object> data) {
        // Concatenate critical fields in deterministic order
        String concatenated = 
            String.valueOf(data.get('id')) +
            String.valueOf(data.get('name')) +
            String.valueOf(data.get('email')) +
            String.valueOf(data.get('status'));
        
        // Hash
        Blob hash = Crypto.generateDigest('SHA-256', Blob.valueOf(concatenated));
        return EncodingUtil.convertToHex(hash);
    }
    
    public static Boolean verifyChecksum(Map<String, Object> data, String expectedChecksum) {
        String actualChecksum = calculateChecksum(data);
        return actualChecksum == expectedChecksum;
    }
}
```

**API should return checksum**:
```json
PUT /api/accounts/123

Response:
{
  "id": "123",
  "name": "ACME Corp",
  "email": "contact@acme.com",
  "status": "active",
  "version": 42,
  "_checksum": "a7f6c3e9d2b1a8f4e5c6d7b8a9f0e1d2"
}
```

**Verification in Apex**:
```apex
Map<String, Object> data = parseResponse(res);
String expectedChecksum = (String)data.remove('_checksum');

if (!DataIntegrityValidator.verifyChecksum(data, expectedChecksum)) {
    throw new DataCorruptionException('Checksum mismatch - data may be corrupted');
}
```

### 2.2 Field-Level Validation

```apex
public class FieldValidator {
    
    public static List<String> validateAccount(Map<String, Object> data) {
        List<String> errors = new List<String>();
        
        // Email format
        String email = (String)data.get('email');
        if (email != null && !Pattern.matches('[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}', email)) {
            errors.add('Invalid email format: ' + email);
        }
        
        // Phone format (if applicable)
        String phone = (String)data.get('phone');
        if (phone != null && !Pattern.matches('\\+?[0-9]{10,15}', phone)) {
            errors.add('Invalid phone format: ' + phone);
        }
        
        // Status whitelist
        String status = (String)data.get('status');
        if (!VALID_STATUSES.contains(status)) {
            errors.add('Invalid status: ' + status);
        }
        
        // Required fields
        if (String.isBlank((String)data.get('name'))) {
            errors.add('Name is required');
        }
        
        return errors;
    }
}
```

---

## 3. Version Conflict Detection

### 3.1 Optimistic Locking Pattern

```apex
public class VersionConflictHandler {
    
    public static Boolean checkAndUpdate(Id recordId, Map<String, Object> remoteData) {
        
        Integer remoteVersion = (Integer)remoteData.get('version');
        
        // Get current local version
        Account local = [
            SELECT ExternalVersion__c, LastModifiedDate 
            FROM Account 
            WHERE Id = :recordId
            LIMIT 1
        ];
        
        // ✅ Check 1: Version comparison
        if (local.ExternalVersion__c >= remoteVersion) {
            // Local is newer or same - skip update
            logSkippedUpdate(recordId, 'Stale remote data', local.ExternalVersion__c, remoteVersion);
            return false;
        }
        
        // ✅ Check 2: Recent local modifications
        DateTime lastModified = local.LastModifiedDate;
        DateTime syncStarted = (DateTime)remoteData.get('sync_initiated_at');
        
        if (lastModified > syncStarted) {
            // User modified locally AFTER sync started
            logConflict(recordId, 'Concurrent modification detected');
            
            // Mark as CONFLICT for manual review
            local.SyncStatus__c = 'CONFLICT';
            local.ConflictReason__c = 'Modified locally during sync';
            update local;
            
            return false;
        }
        
        // ✅ Safe to update
        return true;
    }
}
```

### 3.2 Conflict Resolution UI

```apex
@AuraEnabled
public static ConflictData getConflict(Id recordId) {
    Account local = [SELECT Name, Email__c, Phone FROM Account WHERE Id = :recordId];
    
    // Fetch remote version
    HttpResponse res = callAPI('GET', '/accounts/' + local.ExternalId__c);
    Map<String, Object> remote = parseResponse(res);
    
    return new ConflictData(local, remote);
}

@AuraEnabled
public static void resolveConflict(Id recordId, String resolution) {
    // resolution: 'USE_LOCAL' or 'USE_REMOTE'
    
    if (resolution == 'USE_REMOTE') {
        // Re-trigger sync
        System.enqueueJob(new AsyncSyncJob(recordId));
    } else {
        // Push local to remote
        System.enqueueJob(new PushToRemoteJob(recordId));
    }
}
```

---

## 4. Automated Health Checks

### 4.1 Post-Sync Verification Query

```apex
public class SyncHealthChecker {
    
    @future
    public static void verifyRecentSync(Id recordId) {
        // Wait 10 seconds after sync to verify
        // (Give time for eventual consistency)
        
        Account local = [
            SELECT ExternalId__c, Name, Email__c, ExternalVersion__c
            FROM Account 
            WHERE Id = :recordId
        ];
        
        // Fetch from remote to compare
        HttpResponse res = callAPI('GET', '/accounts/' + local.ExternalId__c);
        Map<String, Object> remote = parseResponse(res);
        
        // Compare critical fields
        List<String> discrepancies = new List<String>();
        
        if (local.Name != (String)remote.get('name')) {
            discrepancies.add('Name mismatch');
        }
        
        if (local.Email__c != (String)remote.get('email')) {
            discrepancies.add('Email mismatch');
        }
        
        if (local.ExternalVersion__c != (Integer)remote.get('version')) {
            discrepancies.add('Version mismatch');
        }
        
        if (!discrepancies.isEmpty()) {
            // Create alert
            insert new Data_Discrepancy_Alert__c(
                RecordId__c = recordId,
                Discrepancies__c = String.join(discrepancies, ', '),
                DetectedAt__c = System.now()
            );
        }
    }
}
```

**Trigger verification automatically**:
```apex
public void execute(QueueableContext ctx) {
    // ... sync logic ...
    
    if (syncSuccess) {
        // ✅ Schedule verification 10 seconds later
        System.scheduleBatch(
            new VerificationBatch(recordId),
            'Verify Sync',
            0,  // 0 minutes delay
            1   // Batch size 1
        );
    }
}
```

### 4.2 Sample Verification (Random Audit)

```apex
global class RandomSyncAuditBatch implements Database.Batchable<SObject>, Database.AllowsCallouts {
    
    global Database.QueryLocator start(Database.BatchableContext bc) {
        // Audit 5% of recently synced records
        return Database.getQueryLocator([
            SELECT Id, ExternalId__c, Name, Email__c, ExternalVersion__c
            FROM Account
            WHERE LastSyncSuccess__c = TODAY
            AND RAND() < 0.05  // 5% sample
        ]);
    }
    
    global void execute(Database.BatchableContext bc, List<Account> scope) {
        for (Account local : scope) {
            // Fetch remote
            HttpResponse res = callAPI('GET', '/accounts/' + local.ExternalId__c);
            Map<String, Object> remote = parseResponse(res);
            
            // Compare
            if (!dataMatches(local, remote)) {
                logDiscrepancy(local, remote);
            }
        }
    }
    
    global void finish(Database.BatchableContext bc) {
        sendAuditReport();
    }
}

// Schedule daily
System.schedule('Daily Sync Audit', '0 0 3 * * ?', new RandomSyncAuditSchedulable());
```

---

## 5. Error Detection & Alerting

### 5.1 Pattern Detection

```apex
public class SyncAnomalyDetector {
    
    public static void checkForAnomalies() {
        
        // ✅ Check 1: High error rate
        AggregateResult[] errorRate = [
            SELECT COUNT(Id) errors
            FROM Account
            WHERE SyncStatus__c = 'ERROR'
            AND LastSyncAttempt__c = LAST_N_HOURS:1
        ];
        
        Integer errors = (Integer)errorRate[0].get('errors');
        if (errors > 50) {  // Threshold
            sendAlert('HIGH_ERROR_RATE', errors + ' sync errors in last hour');
        }
        
        // ✅ Check 2: Long-pending syncs
        List<Account> stale = [
            SELECT Id, Name
            FROM Account
            WHERE SyncStatus__c = 'PENDING'
            AND LastSyncAttempt__c < :System.now().addHours(-2)
        ];
        
        if (!stale.isEmpty()) {
            sendAlert('STALE_SYNCS', stale.size() + ' records pending >2 hours');
        }
        
        // ✅ Check 3: Version drift
        List<Account> versionDrift = [
            SELECT Id, Name, ExternalVersion__c
            FROM Account
            WHERE ExternalVersion__c != null
            AND LastSyncSuccess__c < :System.now().addDays(-7)
        ];
        
        if (!versionDrift.isEmpty()) {
            sendAlert('VERSION_DRIFT', versionDrift.size() + ' records not synced in 7 days');
        }
    }
}

// Run every hour
System.schedule('Hourly Sync Anomaly Check', '0 0 * * * ?', new SyncAnomalyDetectorSchedulable());
```

### 5.2 Alert Configuration

```apex
public class SyncAlertManager {
    
    public static void sendAlert(String alertType, String message) {
        
        // Check alert throttling (don't spam)
        if (recentAlertExists(alertType, 30)) {
            return;  // Already alerted in last 30 minutes
        }
        
        // Log alert
        insert new Sync_Alert__c(
            Type__c = alertType,
            Message__c = message,
            Severity__c = getSeverity(alertType),
            AlertedAt__c = System.now()
        );
        
        // Send notification based on severity
        if (getSeverity(alertType) == 'HIGH') {
            sendSlackNotification(message);
            sendEmailToAdmins(message);
        } else {
            // Log only for LOW/MEDIUM
        }
    }
    
    private static String getSeverity(String alertType) {
        switch on alertType {
            when 'HIGH_ERROR_RATE', 'API_DOWN' {
                return 'HIGH';
            }
            when 'STALE_SYNCS', 'VERSION_DRIFT' {
                return 'MEDIUM';
            }
            when else {
                return 'LOW';
            }
        }
    }
}
```

---

## 6. Dashboard & Monitoring

### 6.1 Real-Time Metrics

```apex
@AuraEnabled(cacheable=true)
public static Map<String, Object> getSyncMetrics() {
    Map<String, Object> metrics = new Map<String, Object>();
    
    // Success rate (last 24h)
    AggregateResult[] stats = [
        SELECT 
            COUNT(Id) total,
            COUNT_DISTINCT(CASE WHEN SyncStatus__c = 'SYNCED' THEN Id END) successes
        FROM Account
        WHERE LastSyncAttempt__c = LAST_N_DAYS:1
    ];
    
    Integer total = (Integer)stats[0].get('total');
    Integer successes = (Integer)stats[0].get('successes');
    Decimal successRate = total > 0 ? (successes * 100.0 / total) : 100;
    
    metrics.put('success_rate', successRate);
    metrics.put('total_syncs', total);
    
    // Current status breakdown
    AggregateResult[] statusBreakdown = [
        SELECT SyncStatus__c, COUNT(Id) count
        FROM Account
        WHERE SyncStatus__c != null
        GROUP BY SyncStatus__c
    ];
    
    Map<String, Integer> breakdown = new Map<String, Integer>();
    for (AggregateResult ar : statusBreakdown) {
        breakdown.put((String)ar.get('SyncStatus__c'), (Integer)ar.get('count'));
    }
    
    metrics.put('status_breakdown', breakdown);
    
    // Average sync time
    AggregateResult[] avgTime = [
        SELECT AVG(SyncDuration__c) avgDuration
        FROM Account
        WHERE SyncStatus__c = 'SYNCED'
        AND LastSyncSuccess__c = LAST_N_HOURS:1
    ];
    
    metrics.put('avg_sync_duration_ms', avgTime[0].get('avgDuration'));
    
    return metrics;
}
```

### 6.2 LWC Dashboard Component

```javascript
// syncMonitorDashboard.js
import { LightningElement, wire } from 'lwc';
import getSyncMetrics from '@salesforce/apex/SyncMonitorController.getSyncMetrics';

export default class SyncMonitorDashboard extends LightningElement {
    
    @wire(getSyncMetrics)
    wiredMetrics({ data, error }) {
        if (data) {
            this.successRate = data.success_rate;
            this.totalSyncs = data.total_syncs;
            this.breakdown = data.status_breakdown;
            this.avgDuration = data.avg_sync_duration_ms;
        }
    }
    
    get healthColor() {
        if (this.successRate >= 95) return 'green';
        if (this.successRate >= 90) return 'yellow';
        return 'red';
    }
}
```

```html
<!-- syncMonitorDashboard.html -->
<template>
    <lightning-card title="Sync Health Dashboard">
        
        <!-- Success Rate -->
        <div class="slds-p-around_medium">
            <div class="metric">
                <h2>Success Rate (24h)</h2>
                <div class={healthColor}>
                    {successRate}%
                </div>
            </div>
            
            <!-- Status Breakdown -->
            <div class="breakdown">
                <div>✅ Synced: {breakdown.SYNCED}</div>
                <div>⏳ Pending: {breakdown.PENDING}</div>
                <div>🔄 Processing: {breakdown.PROCESSING}</div>
                <div>❌ Error: {breakdown.ERROR}</div>
            </div>
            
            <!-- Avg Duration -->
            <div class="metric">
                <h2>Avg Sync Time</h2>
                <div>{avgDuration} ms</div>
            </div>
        </div>
        
    </lightning-card>
</template>
```

---

## 7. Recommended Verification Strategy (Non-Banking)

### 7.1 Tiered Approach

| Priority | Verification | Frequency | Action on Failure |
|----------|--------------|-----------|-------------------|
| **P0 - Critical** | Job status tracking | Every sync | Immediate retry |
| **P0 - Critical** | Response validation | Every sync | Log error, alert |
| **P1 - High** | Version conflict check | Every sync | Mark conflict |
| **P2 - Medium** | Checksum validation | Every sync | Log warning |
| **P3 - Low** | Post-sync verification | 5% sample | Log discrepancy |
| **P3 - Low** | Anomaly detection | Hourly | Alert if threshold |
| **P4 - Audit** | Full reconciliation | Weekly | Report + cleanup |

### 7.2 Minimal Implementation (Recommended Starting Point)

```apex
// Bare minimum for production
public class AsyncSyncJob implements Queueable, Database.AllowsCallouts {
    
    public void execute(QueueableContext ctx) {
        
        // 1️⃣ MUST: Track status
        updateStatus(recordId, 'PROCESSING');
        
        try {
            HttpResponse res = callAPI();
            
            // 2️⃣ MUST: Validate HTTP response
            if (res.getStatusCode() != 200) {
                throw new CalloutException('API returned ' + res.getStatusCode());
            }
            
            Map<String, Object> data = parseResponse(res);
            
            // 3️⃣ MUST: Check version (prevent stale data)
            if (!isNewerVersion(data)) {
                updateStatus(recordId, 'SKIPPED');
                return;
            }
            
            // 4️⃣ SHOULD: Validate critical fields
            if (!hasRequiredFields(data)) {
                throw new ValidationException('Missing required fields');
            }
            
            // 5️⃣ Apply update
            updateRecord(recordId, data);
            updateStatus(recordId, 'SYNCED');
            
        } catch (Exception e) {
            // 6️⃣ MUST: Log errors with context
            logError(recordId, e);
            updateStatus(recordId, 'ERROR');
            
            // 7️⃣ SHOULD: Auto-retry for transient errors
            if (isRetryable(e)) {
                retryLater(recordId);
            }
        }
    }
}
```

### 7.3 Enhanced Implementation (For Critical Data)

Add on top of minimal:
- ✅ Checksum validation
- ✅ Post-sync verification (sample)
- ✅ Conflict detection & resolution UI
- ✅ Anomaly detection
- ✅ Real-time dashboard

---

## 8. Reconciliation Job (Safety Net)

### 8.1 Weekly Full Reconciliation

```apex
global class WeeklyReconciliationBatch implements Database.Batchable<SObject>, Database.AllowsCallouts {
    
    global Database.QueryLocator start(Database.BatchableContext bc) {
        // All accounts with external IDs
        return Database.getQueryLocator([
            SELECT Id, ExternalId__c, Name, Email__c, ExternalVersion__c
            FROM Account
            WHERE ExternalId__c != null
        ]);
    }
    
    global void execute(Database.BatchableContext bc, List<Account> scope) {
        List<ReconciliationResult> results = new List<ReconciliationResult>();
        
        for (Account local : scope) {
            // Fetch remote version
            HttpResponse res = callAPI('GET', '/accounts/' + local.ExternalId__c);
            
            if (res.getStatusCode() == 404) {
                // Missing in remote
                results.add(new ReconciliationResult(
                    local.Id,
                    'MISSING_REMOTE',
                    'Exists in SF but not in master'
                ));
                continue;
            }
            
            Map<String, Object> remote = parseResponse(res);
            
            // Compare
            if (local.ExternalVersion__c != (Integer)remote.get('version')) {
                results.add(new ReconciliationResult(
                    local.Id,
                    'VERSION_MISMATCH',
                    'Local: ' + local.ExternalVersion__c + ', Remote: ' + remote.get('version')
                ));
            }
            
            if (local.Name != (String)remote.get('name')) {
                results.add(new ReconciliationResult(
                    local.Id,
                    'DATA_MISMATCH',
                    'Name differs'
                ));
            }
        }
        
        // Log discrepancies
        logReconciliationResults(results);
    }
    
    global void finish(Database.BatchableContext bc) {
        sendReconciliationReport();
    }
}

// Schedule weekly
System.schedule('Weekly Reconciliation', '0 0 2 ? * SUN', new WeeklyReconciliationSchedulable());
```

---

## 9. Summary: Recommended Checks by Reliability Level

### High Reliability System (Your Case)

```
✅ MUST HAVE:
1. Job status tracking (PENDING → PROCESSING → SYNCED/ERROR)
2. HTTP response validation (status code, body structure)
3. Version conflict detection
4. Error logging with context
5. Basic retry logic

✅ SHOULD HAVE:
6. Field validation (required fields, formats)
7. Anomaly detection (error rate, stale syncs)
8. Weekly reconciliation batch

⚠️ OPTIONAL (if budget allows):
9. Checksum validation
10. Post-sync verification (sample)
11. Real-time monitoring dashboard
12. Conflict resolution UI
```

### Medium Reliability System

Add to above:
- Daily reconciliation (instead of weekly)
- Higher sample rate for verification (10-20%)
- More aggressive alerting

### Low Reliability System

Go synchronous or add all checks.

---

## Final Recommendation

**For your use case (good reliability, non-banking, referential data)**:

```apex
// Production-ready async pattern
public class ProductionAsyncSync implements Queueable, Database.AllowsCallouts {
    
    public void execute(QueueableContext ctx) {
        
        // ✅ 1. Status tracking
        updateStatus('PROCESSING', ctx.getJobId());
        
        try {
            // ✅ 2. API call
            HttpResponse res = callAPI();
            
            // ✅ 3. Response validation
            validateResponse(res);
            
            Map<String, Object> data = parseResponse(res);
            
            // ✅ 4. Data validation
            validateData(data);
            
            // ✅ 5. Version check
            checkVersion(data);
            
            // ✅ 6. Update
            applyUpdate(data);
            
            // ✅ 7. Success
            updateStatus('SYNCED', ctx.getJobId());
            
        } catch (Exception e) {
            handleError(e, ctx.getJobId());
        }
    }
}

// ✅ 8. Scheduled reconciliation (weekly)
// ✅ 9. Anomaly detection (hourly)
// ✅ 10. Monitoring dashboard
```

**This gives you**:
- 95%+ reliability
- Fast failure detection
- Automatic recovery for transient errors
- Weekly safety net
- Manageable complexity

**Cost**: ~500 lines of code, minimal compute overhead, high confidence in data integrity.
