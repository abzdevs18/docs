# Schedule Dissolution API

This endpoint handles the dissolution of a schedule/offering, including the management of enrolled students and notifications.

## Endpoint

```
POST /api/v2/schedule/dissolve
```

## Authentication

Authentication is required for this endpoint.

## Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| scheduleId | string | Yes | The ID of the schedule to dissolve |
| reason | string | No | The reason for dissolution |
| forceDissolve | boolean | No | Whether to force dissolution even with enrolled students |
| transferTargets | TransferTarget[] | No | Array of transfer targets for enrolled students |

### TransferTarget Object

```typescript
{
  scheduleId: string;    // ID of the target schedule
  studentIds: string[];  // Array of student IDs to transfer
}
```

## Response

### Success Response

```typescript
{
  success: true,
  message: "Schedule dissolved successfully",
  data: Schedule,        // The updated schedule object
  affectedStudents: number  // Number of affected students
}
```

### Error Responses

#### 400 Bad Request
- When scheduleId is missing
```typescript
{
  error: "Schedule ID is required"
}
```

- When attempting to dissolve a schedule with enrolled students without transfer plans
```typescript
{
  error: "Cannot dissolve schedule with enrolled students",
  requiresTransfer: true,
  enrolledStudents: [
    {
      studentId: string,
      userId: string,
      name: string
    }
  ]
}
```

#### 404 Not Found
```typescript
{
  error: "Schedule not found"
}
```

#### 500 Internal Server Error
```typescript
{
  error: "Failed to dissolve schedule",
  details: string
}
```

## Features

### 1. Validation
- Verifies schedule existence
- Checks for enrolled students
- Validates transfer target capacities
- Ensures data consistency through transactions

### 2. Student Transfer Management
- Supports bulk student transfers
- Validates target schedule capacity
- Updates enrollment records
- Sends transfer notifications to affected students

### 3. Notification System
- Automatic notifications for:
  - Schedule dissolution
  - Student transfers
- High-priority modal notifications
- Includes relevant context (schedule, professor, reason)

### 4. Data Preservation
- Maintains historical data
- Records dissolution metadata:
  - Timestamp
  - Reason
  - Previous status
  - Transfer details
  - Force dissolution flag

## Example Usage

### Basic Dissolution (No Students)
```typescript
const response = await fetch('/api/v2/schedule/dissolve', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    scheduleId: "schedule-123",
    reason: "Schedule optimization"
  })
});
```

### Dissolution with Student Transfers
```typescript
const response = await fetch('/api/v2/schedule/dissolve', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    scheduleId: "schedule-123",
    reason: "Schedule optimization",
    transferTargets: [
      {
        scheduleId: "schedule-456",
        studentIds: ["student-1", "student-2"]
      },
      {
        scheduleId: "schedule-789",
        studentIds: ["student-3"]
      }
    ]
  })
});
```

### Force Dissolution
```typescript
const response = await fetch('/api/v2/schedule/dissolve', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    scheduleId: "schedule-123",
    reason: "Emergency closure",
    forceDissolve: true
  })
});
```

## Sample Responses

### Successful Dissolution (No Students)
```typescript
{
  "success": true,
  "message": "Schedule dissolved successfully",
  "data": {
    "id": "schedule-123",
    "status": "DISSOLVED",
    "dissolvedAt": "2024-01-20T08:30:00Z",
    "dissolvedReason": "Schedule optimization",
    "subjectId": "subject-456",
    "days": ["MON", "WED"],
    "timeStart": "08:00",
    "timeEnd": "09:30",
    "room": "Room 301",
    "professorId": "prof-789",
    "capacity": 40,
    "schoolYearId": "sy-2024"
  },
  "affectedStudents": 0
}
```

### Successful Dissolution (With Transfers)
```typescript
{
  "success": true,
  "message": "Schedule dissolved and students transferred successfully",
  "data": {
    "id": "schedule-123",
    "status": "DISSOLVED",
    "dissolvedAt": "2024-01-20T08:30:00Z",
    "dissolvedReason": "Schedule optimization",
    // ... other schedule fields ...
  },
  "affectedStudents": 3,
  "transfers": [
    {
      "targetScheduleId": "schedule-456",
      "transferredStudents": 2
    },
    {
      "targetScheduleId": "schedule-789",
      "transferredStudents": 1
    }
  ]
}
```

### Error: Enrolled Students Without Transfer Plan
```typescript
{
  "error": "Cannot dissolve schedule with enrolled students",
  "requiresTransfer": true,
  "enrolledStudents": [
    {
      "studentId": "student-1",
      "userId": "user-1",
      "name": "John Doe"
    },
    {
      "studentId": "student-2",
      "userId": "user-2",
      "name": "Jane Smith"
    }
  ]
}
```

### Error: Invalid Transfer Target
```typescript
{
  "error": "Invalid transfer target",
  "details": {
    "scheduleId": "schedule-456",
    "reason": "Insufficient capacity",
    "currentEnrollment": 38,
    "capacity": 40,
    "requestedTransfers": 3
  }
}
```

### Error: Schedule Not Found
```typescript
{
  "error": "Schedule not found",
  "details": "No schedule exists with ID: schedule-999"
}
```

## Error Handling

The endpoint includes comprehensive error handling:

1. **Input Validation**
   - Validates required fields
   - Checks schedule existence
   - Verifies transfer target validity

2. **Capacity Checks**
   - Ensures target schedules have sufficient capacity
   - Prevents oversubscription

3. **Transaction Safety**
   - All operations are wrapped in a transaction
   - Rolls back on any failure
   - Maintains data consistency

4. **Notification Reliability**
   - Creates notifications within the transaction
   - Ensures notification delivery for all affected students

## Notes

1. **Force Dissolution**
   - Use `forceDissolve` with caution
   - Will leave students without a schedule
   - Sends urgent notifications to affected students

2. **Transfer Targets**
   - Can specify multiple target schedules
   - Students can be distributed across different schedules
   - Capacity is checked before transfer

3. **Notifications**
   - Transfer notifications are INFO level
   - Dissolution notifications are WARNING level
   - All use modal display for visibility 