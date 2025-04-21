# Technical Design Document: Organization Quota System

## 1. Overview

This document outlines the design for a quota management system that allows for limiting resources per organization. The system will support quotas for users, messages, storage, and other configurable resources without relying on JSON-based configuration storage.

## 2. Database Design

### 2.1 Tables

#### `org_quota_type`

Stores definitions of different quota types with their defaults.

- `id` (PK): Unique identifier
- `name`: Unique name for the quota type (e.g., "max_users")
- `description`: Human-readable description
- `default_value`: Default limit value

#### `org_quota`

Stores organization-specific quota limits.

- `id` (PK): Unique identifier
- `org_id` (FK): Reference to organization
- `quota_type_id` (FK): Reference to quota type
- `quota_limit`: Organization-specific limit value
- `create_date`: Creation timestamp
- `update_date`: Last update timestamp

#### `org_quota_usage`

Tracks current usage against quotas.

- `id` (PK): Unique identifier
- `org_id` (FK): Reference to organization
- `quota_type_id` (FK): Reference to quota type
- `current_usage`: Current usage amount
- `period_start`: For time-based quotas, start of period
- `period_end`: For time-based quotas, end of period
- `create_date`: Creation timestamp
- `update_date`: Last update timestamp

### 2.2 Indices

- Unique composite index on `org_id, quota_type_id` for both quota and usage tables
- Index on period fields for time-based quota queries
- Standard indices on foreign keys

### 2.3 Default Quota Types

Initial quota types will include:

- `max_users`: Maximum number of users allowed in the organization
- `max_messages_per_day`: Maximum messages per day
- `max_messages_total`: Maximum total messages
- `max_storage_mb`: Maximum storage in MB
- `max_groups`: Maximum number of groups
- `max_users_per_group`: Maximum users per group
- `max_file_size_mb`: Maximum file size in MB

## 3. Application Architecture

### 3.1 Entity Layer

- `OrgQuotaType`: Maps to `org_quota_type` table
- `OrgQuota`: Maps to `org_quota` table
- `OrgQuotaUsage`: Maps to `org_quota_usage` table

### 3.2 Repository Layer

- `OrgQuotaTypeRepository`: CRUD operations for quota types
- `OrgQuotaRepository`: CRUD operations for quota settings
- `OrgQuotaUsageRepository`: CRUD operations for quota usage with locking support

### 3.3 Service Layer

- `OrgQuotaService`: Core service providing quota management functionality
  - Get quota limits
  - Track usage
  - Check if quotas are reached
  - Update quota settings

### 3.4 Controller Layer

- `OrgQuotaController`: REST endpoints for quota management
  - Get all quotas for an organization
  - Update quota limit for a specific quota type

### 3.5 DTO Layer

- `OrgQuotaTypeDto`: Represents quota type definition
- `OrgQuotaDto`: Represents a quota setting
- `OrgQuotaRespDto`: Response containing all quotas for an org
- `OrgQuotaUpdateReqDto`: Request for updating a quota

## 4. Key Functionalities

### 4.1 Quota Checking

```java
// Check if quota limit reached
boolean isQuotaLimitReached(String orgId, String quotaTypeName) {
  Long limit = getQuotaLimit(orgId, quotaTypeName);
  Long usage = getCurrentUsage(orgId, quotaTypeName);
  return usage >= limit;
}

// Check if adding would exceed quota
boolean wouldExceedQuota(String orgId, String quotaTypeName, Long additionalAmount) {
  Long limit = getQuotaLimit(orgId, quotaTypeName);
  Long usage = getCurrentUsage(orgId, quotaTypeName);
  return (usage + additionalAmount) > limit;
}
```

### 4.2 Quota Usage Tracking

```java
// Increment usage with locking
Long incrementUsage(String orgId, String quotaTypeName, Long amount) {
  // Check if would exceed quota
  if (wouldExceedQuota(orgId, quotaTypeName, amount)) {
    throw new ForbiddenExceptionV2(ErrorCodeEnum.FORBIDDEN,
        "Organization quota limit reached for " + quotaTypeName);
  }

  // Acquire lock and update or create usage record
  Optional<OrgQuotaUsage> usageOptional =
      orgQuotaUsageRepository.findWithLockByOrg_idAndQuotaType_name(orgId, quotaTypeName);

  // Update or create usage record
  // Handle time-based quotas (reset periods)
  // Save and return new usage
}
```

### 4.3 Time-based Quotas

```java
// For time-based quotas like daily messages
if ("max_messages_per_day".equals(quotaTypeName) && usage.getPeriodEnd() != null) {
  // If current time is outside the period, reset the counter and update period
  if (now.isAfter(usage.getPeriodEnd())) {
    long startOfDay = now.toEpochMilli() / 86400000L * 86400000L;
    long endOfDay = startOfDay + 86400000L - 1;
    usage.setPeriodStart(Instant.ofEpochMilli(startOfDay));
    usage.setPeriodEnd(Instant.ofEpochMilli(endOfDay));
    usage.setCurrentUsage(amount); // Reset counter
  } else {
    // Within same period, just increment
    usage.setCurrentUsage(usage.getCurrentUsage() + amount);
  }
}
```

### 4.4 Quota Management API

```
GET /v1/org/{orgId}/quota
PUT /v1/org/{orgId}/quota
```

## 5. Integration Points

### 5.1 User Creation

```java
// Check before creating user
if (orgQuotaService.isQuotaLimitReached(orgId, "max_users")) {
  throw new ForbiddenExceptionV2(ErrorCodeEnum.FORBIDDEN,
      "Organization has reached maximum user limit");
}

// After successfully creating user
orgQuotaService.incrementUsage(orgId, "max_users", 1L);
```

### 5.2 Message Sending

```java
// Check before sending message
public void checkMessageQuota(String orgId) {
  if (orgQuotaService.isQuotaLimitReached(orgId, "max_messages_per_day")) {
    throw new ForbiddenExceptionV2(ErrorCodeEnum.FORBIDDEN,
        "Daily message quota reached for this organization");
  }

  if (orgQuotaService.isQuotaLimitReached(orgId, "max_messages_total")) {
    throw new ForbiddenExceptionV2(ErrorCodeEnum.FORBIDDEN,
        "Total message quota reached for this organization");
  }
}

// After sending message
public void incrementMessageCount(String orgId) {
  orgQuotaService.incrementUsage(orgId, "max_messages_per_day", 1L);
  orgQuotaService.incrementUsage(orgId, "max_messages_total", 1L);
}
```

## 6. Caching Strategy

### 6.1 Quota Limits Caching

```java
@Cacheable(value = "default", key = "'org_quota:' + #orgId")
public OrgQuotaRespDto getAllQuotaLimits(String orgId) {
  // Implementation
}

@CacheEvict(value = "default", key = "'org_quota:' + #orgId")
public OrgQuotaRespDto updateQuotaLimit(String orgId, OrgQuotaUpdateReqDto dto) {
  // Implementation
}
```

## 7. Security Considerations

### 7.1 Access Control

- Quota management endpoints restricted to ADMIN role
- Appropriate authorization checks on all endpoints
- Proper error handling to avoid leaking quota information

### 7.2 Concurrency

- Pessimistic locking for usage updates
- Consistent transaction boundaries

## 8. Performance Considerations

### 8.1 Database Load

- Quota limit checks happen frequently
- Heavy caching recommended
- Batch usage updates where possible

### 8.2 Locking Strategy

- Short-lived locks during usage increments
- Transaction timeouts to prevent deadlocks

## 9. Monitoring and Alerting

### 9.1 Key Metrics

- Quota utilization percentage
- Quota limit reached events
- Usage growth trends

### 9.2 Alerts

- Organizations approaching quota limits (e.g., 80% threshold)
- Unusually rapid quota consumption

## 10. Future Enhancements

### 10.1 Additional Quota Types

- API call rate limits
- Storage quotas by content type
- Bandwidth quotas

### 10.2 Quota Plans

- Tiered quota templates (Basic, Premium, Enterprise)
- Organization-specific quota overrides

### 10.3 Analytics

- Usage reports and trends
- Predictive quota limit recommendations
