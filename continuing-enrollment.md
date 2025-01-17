# Continuing Student Enrollment API

This endpoint handles the enrollment process for continuing students across all basic education levels (Elementary, Junior High School, and Senior High School), including validation of academic standing, prerequisites, and schedule management.

**URL**: `/api/v2/enrollment/continuing/process`  
**Method**: `POST`  
**Auth required**: Yes  
**Permissions required**: `STUDENT` or `REGISTRAR`

## Request Body

```json
{
  "studentId": "string",
  "schoolYearId": "string",
  "semesterId": "string",
  "subjectsToEnroll": [
    {
      "subjectId": "string",
      "scheduleId": "string" // Optional
    }
  ],
  "sectionId": "string",
  "processedBy": "string"
}
```

### Fields

| Field | Type | Description | Required |
|-------|------|-------------|-----------|
| `studentId` | string | ID of the continuing student | Yes |
| `schoolYearId` | string | ID of the school year | Yes |
| `semesterId` | string | ID of the semester (required for SHS, optional for others) | Conditional |
| `subjectsToEnroll` | array | List of subjects to enroll in | Conditional* |
| `subjectsToEnroll[].subjectId` | string | ID of the subject | Yes (if using subjects) |
| `subjectsToEnroll[].scheduleId` | string | ID of preferred schedule | No |
| `sectionId` | string | ID of the section for automatic subject enrollment | Conditional* |
| `processedBy` | string | ID of user processing the enrollment | Yes |

\* Either `subjectsToEnroll` or `sectionId` must be provided

## Education Level Specific Rules

### Elementary (Grades 1-6)
- No semester system, full year enrollment
- Automatic subject loading based on grade level
- No prerequisites required
- No GPA requirements
- Maximum failed subjects: 2 per year
- Grade advancement based on:
  - Passing grades in core subjects
  - Completion of remedial classes if needed

### Junior High School (Grades 7-10)
- No semester system, full year enrollment
- Subject-based promotion
- Prerequisites for certain subjects (e.g., Math sequence)
- Minimum GPA requirement: 75
- Maximum failed subjects: 2 per year
- Grade advancement based on:
  - Passing grades in core subjects
  - Completion of remedial classes if needed

### Senior High School (Grades 11-12)
- Semester-based enrollment
- Track/Strand specific subjects
- Strict prerequisites system
- Minimum GPA requirement: 75
- Maximum failed subjects: 2 per semester
- Grade advancement based on:
  - Completion of required units
  - Meeting track/strand requirements

## Validation Rules

### General Validation
- Application ID must exist
- Either `subjectsToEnroll` or `sectionId` must be provided
- Processor ID must exist

### Section Validation (when using sectionId)
- Section must exist and be active
- Grade level must match student's grade level
- For SHS:
  - Strand must match student's strand
  - Track must match student's track
- Section capacity validation
- Schedule availability for all section subjects

### 1. Student Status Validation
- Verifies student record exists
- Checks for any existing active enrollment
- Validates student information including:
  - Grade level
  - Track/Strand (SHS only)
  - Previous academic records

### 2. Academic Standing
Varies by education level:

**Elementary**:
- No GPA requirement
- Core subject completion check
- Attendance requirements check

**Junior High**:
- Minimum GPA: 75
- Maximum yearly failed subjects: 2
- Core subject completion check

**Senior High**:
- Minimum GPA: 75
- Maximum failed subjects per semester: 2
- Track/Strand requirements check

### 3. Prerequisites
**Elementary**:
- Grade level progression only

**Junior High**:
- Basic subject sequencing
- Core subject completion

**Senior High**:
- Strict prerequisite checking
- Track/Strand requirements
- Subject sequencing within specializations

### 4. Unit Load
**Elementary & Junior High**:
- Fixed subject load per grade level

**Senior High**:
- Maximum units per semester: 30
- Minimum units per semester: 15
- Track/Strand specific requirements

### 5. Schedule
- Maximum students per section: 50
- Grade level specific scheduling
- Auto-assigns schedule if not specified
- Considers student level for section assignment

## Success Response

**Code**: `200 OK`

```json
{
  "success": true,
  "message": "Continuing student enrollment processed successfully",
  "data": {
    "enrollment": {
      "id": "string",
      "status": "ENROLLED",
      "type": "K12",
      "educationLevel": "ELEMENTARY|JUNIOR_HIGH|SENIOR_HIGH",
      // ... other enrollment details
    },
    "student": {
      // Student information
    },
    "summary": {
      "studentInfo": {
        "id": "string",
        "name": "string",
        "email": "string",
        "gradeLevel": "string",
        "strand": "string", // SHS only
        "track": "string"   // SHS only
      },
      "enrollmentInfo": {
        "id": "string",
        "status": "string",
        "type": "CONTINUING",
        "totalUnits": "number",
        "totalSubjects": "number",
        "semester": "string" // SHS only
      },
      "subjects": [
        {
          "code": "string",
          "name": "string",
          "units": "number",
          "schedule": {
            "days": ["string"],
            "timeStart": "string",
            "timeEnd": "string",
            "room": "string"
          }
        }
      ]
    }
  }
}
```

## Error Responses

### Invalid Request
**Code**: `400 BAD REQUEST`
```json
{
  "success": false,
  "error": "Missing required fields"
}
```

### Student Not Eligible
**Code**: `400 BAD REQUEST`
```json
{
  "success": false,
  "error": "Student has {n} failed subjects, exceeding the maximum allowed"
}
```
```json
{
  "success": false,
  "error": "Student's GPA ({gpa}) is below the required minimum of {min}"
}
```
```json
{
  "success": false,
  "error": "Core subjects from previous level not completed"
}
```

### Prerequisite Not Met
**Code**: `400 BAD REQUEST`
```json
{
  "success": false,
  "error": "Missing prerequisites for {subject}: {prerequisite1}, {prerequisite2}"
}
```

### Schedule Full
**Code**: `400 BAD REQUEST`
```json
{
  "success": false,
  "error": "Schedule is already full"
}
```

### Server Error
**Code**: `500 INTERNAL SERVER ERROR`
```json
{
  "success": false,
  "error": "Failed to process enrollment"
}
```

## Notes

1. **Education Level Specific Processing**
   - System automatically detects education level from student record
   - Applies appropriate validation rules and requirements
   - Handles subject loading based on curriculum level

2. **Notifications**
   - System sends notifications to student/parent and registrar
   - Email confirmation sent to primary contact
   - Level-specific enrollment details included

3. **Grade Level Advancement**
   - Automated promotion based on level-specific requirements
   - Different criteria for each education level
   - Considers remedial classes and summer programs

4. **Schedule Assignment**
   - Grade level appropriate section assignment
   - Considers student age and previous performance
   - Maintains optimal class sizes

5. **Transaction Handling**
   - All database operations are wrapped in a transaction
   - Ensures data consistency across all operations
   - Handles rollback for failed enrollments

## Example Usage

### 1. Manual Subject Selection Example

#### Elementary School (Grade 4)
```http
POST /api/v2/enrollment/continuing/process
Content-Type: application/json
x-tenant-tag: sample-school

{
  "studentId": "ELEM123",
  "schoolYearId": "SY2024-2025",
  "subjectsToEnroll": [
    { "subjectId": "MATH4" },
    { "subjectId": "ENG4" },
    { "subjectId": "SCI4" },
    { "subjectId": "FIL4" },
    { "subjectId": "AP4" },
    { 
      "subjectId": "COMP4", 
      "scheduleId": "SCH456"  // With preferred schedule
    }
  ],
  "processedBy": "REGISTRAR_001"
}
```

### 2. Section-Based Enrollment Examples

#### Elementary School (Grade 4)
```http
POST /api/v2/enrollment/continuing/process
Content-Type: application/json
x-tenant-tag: sample-school

{
  "studentId": "ELEM123",
  "schoolYearId": "SY2024-2025",
  "sectionId": "SEC-G4-A",        // Section-based enrollment
  "processedBy": "REGISTRAR_001"
}
```

#### Senior High School (Grade 11 - STEM)
```http
POST /api/v2/enrollment/continuing/process
Content-Type: application/json
x-tenant-tag: sample-school

{
  "studentId": "SHS123",
  "schoolYearId": "SY2024-2025",
  "semesterId": "SEM1",          // Required for SHS
  "sectionId": "SEC-G11-STEM-A", // Section-based enrollment
  "processedBy": "REGISTRAR_001"
}
```

### General API Call Example
```typescript
const response = await fetch('/api/v2/enrollment/continuing/process', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer {token}'
  },
  body: JSON.stringify(seniorHighEnrollment) // or elementaryEnrollment or juniorHighEnrollment
});

const result = await response.json();
``` 