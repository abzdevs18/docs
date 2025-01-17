# Basic Education Admission Process API

## Overview
This endpoint handles the processing of new student admissions for basic education (Elementary, Junior High School, and Senior High School). It manages the entire workflow from application approval to student enrollment, including account creation, section assignment, and subject enrollment.

## Endpoint Details
- **URL**: `/api/v2/applications/basic-education/process`
- **Method**: `POST`
- **Authentication**: Required (Admin/Registrar access)

## Request Format

### Headers
```json
{
  "Content-Type": "application/json",
  "x-tenant-tag": "string"
}
```

### Body
```json
{
  "applicationId": "string",     // Application ID or Application Number
  "status": "string",           // PENDING_FEE_CONFIGURATION, APPROVE, REJECT
  "remarks": "string",          // Optional remarks for the action
  "processedBy": "string",      // User ID of the processor
  "sectionId": "string"         // Required when status is APPROVE
}
```

## Processing Steps

### 1. PENDING_FEE_CONFIGURATION Status
When setting status to `PENDING_FEE_CONFIGURATION`, the system:
1. Creates a new user account with:
   - Generated unique username
   - Temporary password
   - QR hash for identification
2. Creates student record with:
   - Basic information from application
   - Grade level assignment
   - Strand/Track assignment (for SHS)
3. Updates application status
4. Notifies finance department

### 2. APPROVE Status
When setting status to `APPROVE`, the system:

#### Pre-enrollment Validation
1. Verifies application status (must be `PENDING_PAYMENT` or `PENDING_FEE_CONFIGURATION`)
2. Validates section capacity
3. Checks grade level compatibility
4. For SHS:
   - Validates active semester
   - Verifies strand/track alignment

#### Section Assignment
1. Assigns student to specified section
2. Updates section student count
3. Validates maximum capacity
4. Handles rollback if capacity exceeded

#### Subject Enrollment
1. Retrieves appropriate subjects based on:
   - Grade level
   - Strand/Track (for SHS)
   - Active semester (for SHS)
2. For each subject:
   - Creates section-subject relationship
   - Generates or assigns schedule
   - Validates schedule capacity
   - Creates subject enrollment record

#### Schedule Management
1. Creates or finds existing schedules
2. Validates schedule capacity
3. Updates enrolled student count
4. Handles schedule conflicts

### 3. REJECT Status
When setting status to `REJECT`:
1. Updates application status
2. Records rejection reason
3. Sends notification to applicant

## Response Format

### Success Response
```json
{
  "success": true,
  "message": "Student enrollment approved and processed successfully",
  "data": {
    "student": {
      "id": "string",
      "userId": "string",
      "gradeLevelId": "string",
      "sectionId": "string"
    },
    "enrollment": {
      "id": "string",
      "status": "ENROLLED",
      "type": "K12",
      "schoolYearId": "string",
      "semesterId": "string"      // For SHS only
    },
    "subjects": [
      {
        "id": "string",
        "name": "string",
        "code": "string",
        "schedule": {
          "days": ["string"],
          "timeStart": "string",
          "timeEnd": "string",
          "room": "string",
          "capacity": "number",
          "enrolled": "number"
        }
      }
    ]
  }
}
```

### Error Response
```json
{
  "success": false,
  "error": "Error message description"
}
```

## Validation Rules

### General Validation
- Application ID must exist
- Status must be valid
- Processor ID must exist
- Section ID required for approval

### Section Validation
- Maximum capacity check
- Grade level compatibility
- Strand/Track compatibility (for SHS)
- Current student count verification

### Subject Enrollment Validation
1. **Elementary/Junior High**
   - All grade level subjects must be available
   - Core subjects must be included
   - No semester validation required

2. **Senior High School**
   - Semester validation required
   - Subjects must match active semester
   - Strand/Track specific subjects
   - Core subjects must be included
   - Unit load validation

### Schedule Validation
- Maximum capacity per schedule
- Schedule availability
- Room capacity
- Time slot conflicts

## Error Codes
- `400`: Bad Request (Invalid parameters)
- `404`: Resource Not Found
- `409`: Conflict (Capacity/Schedule conflicts)
- `422`: Unprocessable Entity (Validation failures)
- `500`: Internal Server Error

## Notes
1. **Transaction Management**
   - All database operations are wrapped in a transaction
   - Automatic rollback on failure
   - Capacity checks are atomic

2. **Notifications**
   - Email notifications for account creation
   - SMS notifications if phone number available
   - System notifications for relevant departments

3. **Caching**
   - Subject load details are cached
   - Cache invalidation on schedule updates
   - Section capacity cache updates

4. **Security**
   - Role-based access control
   - Tenant isolation
   - Audit logging of all actions

5. **Special Considerations for SHS**
   - Semester-based enrollment
   - Strand/Track specific validation
   - Different unit load requirements
   - Semester-specific subject availability 

## Sample Usage

### 1. PENDING_FEE_CONFIGURATION Status Example

#### Request
```http
POST /api/v2/applications/basic-education/process
Content-Type: application/json
x-tenant-tag: sample-school

{
  "applicationId": "APP-2024-0001",
  "status": "PENDING_FEE_CONFIGURATION",
  "remarks": "Application verified. Proceeding with account creation.",
  "processedBy": "USR-ADMIN-001"
}
```

#### Success Response
```json
{
  "success": true,
  "message": "Application processed successfully. Account created and pending fee configuration.",
  "data": {
    "application": {
      "id": "APP-2024-0001",
      "status": "PENDING_FEE_CONFIGURATION",
      "student": {
        "id": "STD-2024-0001",
        "userId": "USR-2024-0001",
        "credentials": {
          "username": "john.doe24",
          "temporaryPassword": "TempPass123!"
        }
      },
      "timeline": {
        "action": "PENDING_FEE_CONFIGURATION",
        "timestamp": "2024-01-12T08:30:00Z",
        "performedBy": "USR-ADMIN-001",
        "remarks": "Application verified. Proceeding with account creation."
      }
    }
  }
}
```

### 2. APPROVE Status Examples

#### Example 1: Elementary Student Approval

##### Request
```http
POST /api/v2/applications/basic-education/process
Content-Type: application/json
x-tenant-tag: sample-school

{
  "applicationId": "APP-2024-0001",
  "status": "APPROVE",
  "remarks": "All requirements complete. Enrollment approved.",
  "processedBy": "USR-ADMIN-001",
  "sectionId": "SEC-G3-A"
}
```

##### Success Response
```json
{
  "success": true,
  "message": "Student enrollment approved and processed successfully",
  "data": {
    "student": {
      "id": "STD-2024-0001",
      "userId": "USR-2024-0001",
      "gradeLevelId": "GL-G3",
      "sectionId": "SEC-G3-A",
      "name": {
        "first": "John",
        "middle": "Michael",
        "last": "Doe"
      }
    },
    "enrollment": {
      "id": "ENR-2024-0001",
      "status": "ENROLLED",
      "type": "K12",
      "schoolYearId": "SY-2024-2025"
    },
    "subjects": [
      {
        "id": "SUBJ-MATH3",
        "name": "Mathematics 3",
        "code": "MATH3",
        "schedule": {
          "days": ["Monday", "Wednesday"],
          "timeStart": "08:00",
          "timeEnd": "09:00",
          "room": "Room 301",
          "capacity": 40,
          "enrolled": 25
        }
      },
      {
        "id": "SUBJ-ENG3",
        "name": "English 3",
        "code": "ENG3",
        "schedule": {
          "days": ["Tuesday", "Thursday"],
          "timeStart": "09:00",
          "timeEnd": "10:00",
          "room": "Room 301",
          "capacity": 40,
          "enrolled": 25
        }
      }
      // ... other subjects
    ]
  }
}
```

#### Example 2: Senior High School Student Approval

##### Request
```http
POST /api/v2/applications/basic-education/process
Content-Type: application/json
x-tenant-tag: sample-school

{
  "applicationId": "APP-2024-0002",
  "status": "APPROVE",
  "remarks": "SHS requirements complete. First semester enrollment approved.",
  "processedBy": "USR-ADMIN-001",
  "sectionId": "SEC-G11-STEM-A"
}
```

##### Success Response
```json
{
  "success": true,
  "message": "Student enrollment approved and processed successfully",
  "data": {
    "student": {
      "id": "STD-2024-0002",
      "userId": "USR-2024-0002",
      "gradeLevelId": "GL-G11",
      "sectionId": "SEC-G11-STEM-A",
      "name": {
        "first": "Jane",
        "middle": "Anne",
        "last": "Smith"
      },
      "strand": "STEM",
      "track": "Academic"
    },
    "enrollment": {
      "id": "ENR-2024-0002",
      "status": "ENROLLED",
      "type": "K12",
      "schoolYearId": "SY-2024-2025",
      "semesterId": "SEM-2024-1ST"
    },
    "subjects": [
      {
        "id": "SUBJ-PRECAL",
        "name": "Pre-Calculus",
        "code": "PRECAL",
        "type": "Specialized",
        "units": 2,
        "schedule": {
          "days": ["Monday", "Wednesday", "Friday"],
          "timeStart": "07:30",
          "timeEnd": "08:30",
          "room": "SCI-LAB-1",
          "capacity": 35,
          "enrolled": 20
        }
      },
      {
        "id": "SUBJ-GENMATH",
        "name": "General Mathematics",
        "code": "GENMATH",
        "type": "Core",
        "units": 2,
        "schedule": {
          "days": ["Tuesday", "Thursday"],
          "timeStart": "09:00",
          "timeEnd": "10:30",
          "room": "Room 401",
          "capacity": 35,
          "enrolled": 20
        }
      }
      // ... other subjects
    ]
  }
}
```

### 3. REJECT Status Example

#### Request
```http
POST /api/v2/applications/basic-education/process
Content-Type: application/json
x-tenant-tag: sample-school

{
  "applicationId": "APP-2024-0003",
  "status": "REJECT",
  "remarks": "Incomplete requirements. Missing previous school records.",
  "processedBy": "USR-ADMIN-001"
}
```

#### Success Response
```json
{
  "success": true,
  "message": "Application rejected successfully",
  "data": {
    "application": {
      "id": "APP-2024-0003",
      "status": "REJECTED",
      "timeline": {
        "action": "REJECT",
        "timestamp": "2024-01-12T10:15:00Z",
        "performedBy": "USR-ADMIN-001",
        "remarks": "Incomplete requirements. Missing previous school records."
      }
    }
  }
}
```

## Error Response Examples

### 1. Invalid Application ID
```json
{
  "success": false,
  "error": "Application not found",
  "code": 404,
  "details": {
    "applicationId": "APP-2024-9999"
  }
}
```

### 2. Section Capacity Exceeded
```json
{
  "success": false,
  "error": "Section capacity exceeded",
  "code": 409,
  "details": {
    "sectionId": "SEC-G11-STEM-A",
    "currentCount": 40,
    "maxCapacity": 40
  }
}
```

### 3. Invalid Semester for SHS
```json
{
  "success": false,
  "error": "Invalid or inactive semester for Senior High School enrollment",
  "code": 422,
  "details": {
    "gradeLevelId": "GL-G11",
    "currentSemester": null,
    "requiresSemester": true
  }
}
```

### 4. Schedule Conflict
```json
{
  "success": false,
  "error": "Schedule conflict detected",
  "code": 409,
  "details": {
    "subject": "Pre-Calculus",
    "conflictingSchedule": {
      "days": ["Monday", "Wednesday"],
      "timeStart": "07:30",
      "timeEnd": "08:30"
    },
    "existingSchedule": {
      "subject": "General Mathematics",
      "days": ["Monday", "Wednesday"],
      "timeStart": "07:00",
      "timeEnd": "08:00"
    }
  }
}
``` 