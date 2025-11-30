# Timesheet API Specification

## 1. Core Data Models

These are the objects the API should know about.

### 1.1 Employee / User

Used for auth and to know who can see whose timesheet.

```jsonc
// Employee
{
  "id": "emp_123",
  "fullName": "Alice Doe",
  "email": "alice@example.com",
  "role": "employee",           // "employee" | "manager" | "admin"
  "managerId": "emp_001",       // null if top level
  "permissions": {
    "canViewTeamTimesheets": true,
    "canViewProjectNames": ["proj_100", "proj_200"], // or "*" for all
    "canApproveTimesheets": true
  }
}
```

### 1.2 Project

Only partially visible depending on permissions.

```jsonc
// Project
{
  "id": "proj_100",
  "code": "P-ALPHA",
  "name": "Alpha Plant Expansion",
  "isConfidential": true
}
```

### 1.3 Task

Each timesheet entry is linked to a task.

```jsonc
// Task
{
  "id": "task_5001",
  "code": "T-PL-001",
  "title": "P&ID Review R-101",
  "projectId": "proj_100",
  "assignees": ["emp_123", "emp_200"],
  "status": "open",  // open | done | on_hold

  // Location/time constraints (optional)
  "allowedLocations": [
    { "type": "site", "siteId": "site_1", "name": "Head Office" },
    { "type": "geo_fence", "lat": 32.6123, "lng": 51.6789, "radiusMeters": 200 }
  ],
  "allowedTimeWindows": [
    { "dayOfWeek": "MONDAY", "start": "08:00", "end": "17:00" }
  ]
}
```

> The API can hide `projectId`/`projectName` if the user is not allowed to see it (see outputs below).

### 1.4 Timesheet Day + Timesheet Entry

Minimal structure to cover all of the timesheet types.

```jsonc
// Timesheet for a day (summary)
{
  "employeeId": "emp_123",
  "date": "2025-11-30",
  "status": "submitted",  // draft | submitted | approved | rejected
  "totalHours": 7.5
}
```

```jsonc
// Timesheet entry (line)
{
  "id": "ts_789",
  "employeeId": "emp_123",
  "date": "2025-11-30",

  "taskId": "task_5001",
  "projectId": "proj_100",       // may be null or hidden if not allowed
  "projectName": "Alpha Plant Expansion", // may be hidden

  "startTime": "09:00",
  "endTime": "12:00",
  "hours": 3.0,

  "location": {                  // where the work was done (optional)
    "type": "site",
    "siteId": "site_1",
    "name": "Head Office",
    "lat": 32.6123,
    "lng": 51.6789
  },

  "notes": "Reviewed P&ID for reactor",
  "createdAt": "2025-11-30T07:10:00Z",
  "updatedAt": "2025-11-30T07:20:00Z"
}
```

---

## 2. Minimal API Endpoints

Using REST + JSON. For each endpoint: **Goal → Input → Output**.

### 2.1 Authentication & Identity

#### 2.1.1 Login

**Goal:** Get a token & base info to know who we are and what we can see.

`POST /auth/login`

**Input (body):**

```json
{
  "username": "alice",
  "password": "password123"
}
```

**Output (200):**

```json
{
  "token": "jwt-or-session-token",
  "user": {
    "id": "emp_123",
    "fullName": "Alice Doe",
    "role": "manager"
  }
}
```

#### 2.1.2 Current User Profile

**Goal:** Get permissions + subordinates for “see under-control employees”.

`GET /me`

**Output:**

```jsonc
{
  "id": "emp_123",
  "fullName": "Alice Doe",
  "role": "manager",
  "managerId": "emp_001",
  "permissions": {
    "canViewTeamTimesheets": true,
    "canApproveTimesheets": true,
    "canViewProjectNames": "*"
  },
  "directReports": [
    { "id": "emp_200", "fullName": "Bob Smith" },
    { "id": "emp_201", "fullName": "Charlie Lee" }
  ]
}
```

---

### 2.2 Tasks API

Your app needs:

- List of tasks assigned to the current user  
- For each task: allowed location/time, possibly project info (if allowed)

#### 2.2.1 List My Tasks

`GET /tasks/my`

**Query params (optional):**

- `status` (e.g., `open`)
- `date=2025-11-30` (if tasks are date-specific)

**Output** (API automatically filters project info based on permissions):

```jsonc
[
  {
    "id": "task_5001",
    "code": "T-PL-001",
    "title": "P&ID Review R-101",
    "project": {
      "id": "proj_100",
      "code": "P-ALPHA",
      "name": "Alpha Plant Expansion",  // or null if not allowed
      "visible": true                  // false if you hide it
    },
    "status": "open",
    "allowedLocations": [
      { "type": "site", "siteId": "site_1", "name": "Head Office" }
    ],
    "allowedTimeWindows": [
      { "dayOfWeek": "SUNDAY", "start": "08:00", "end": "17:00" }
    ]
  },
  {
    "id": "task_5002",
    "code": "T-SEC-002",
    "title": "Confidential Task",
    "project": {
      "id": null,
      "code": null,
      "name": null,
      "visible": false
    },
    "status": "open",
    "allowedLocations": [],
    "allowedTimeWindows": []
  }
]
```

> The hiding of project name is handled by the backend based on the current user.

#### 2.2.2 Task Details

`GET /tasks/{taskId}`

Same structure as above, but for a single task.

---

### 2.3 Timesheet APIs

Needs to support:

- Employee entering their timesheet for a day  
- Viewing own timesheet (current & previous days)  
- Manager viewing team timesheets (current & previous days)

#### 2.3.1 Get My Timesheet for a Day

`GET /timesheets/my?date=2025-11-30`

**Output:**

```jsonc
{
  "employeeId": "emp_123",
  "date": "2025-11-30",
  "status": "draft",
  "entries": [
    {
      "id": "ts_789",
      "taskId": "task_5001",
      "taskTitle": "P&ID Review R-101",
      "projectName": "Alpha Plant Expansion",   // null if not allowed
      "startTime": "09:00",
      "endTime": "12:00",
      "hours": 3.0,
      "location": {
        "type": "site",
        "siteId": "site_1",
        "name": "Head Office"
      },
      "notes": "Reviewed P&ID for R-101."
    }
  ],
  "totalHours": 3.0
}
```

#### 2.3.2 Save / Update My Timesheet for a Day

You can do **upsert** style: send entire day in one shot.

`PUT /timesheets/my?date=2025-11-30`

**Input (body):**

```jsonc
{
  "status": "draft", // or "submitted"
  "entries": [
    {
      // if updating existing entry, provide id; if new, omit or use client temp id
      "id": "ts_789",
      "taskId": "task_5001",
      "startTime": "09:00",
      "endTime": "12:00",
      "location": {
        "type": "geo",
        "lat": 32.6123,
        "lng": 51.6789,
        "accuracyMeters": 15
      },
      "notes": "Reviewed R-101 docs."
    },
    {
      "taskId": "task_5002",
      "startTime": "13:00",
      "endTime": "16:30",
      "location": { "type": "site", "siteId": "site_2" },
      "notes": "Site visit."
    }
  ]
}
```

**Output (200):**

```jsonc
{
  "employeeId": "emp_123",
  "date": "2025-11-30",
  "status": "submitted",
  "totalHours": 6.5,
  "entries": [
    { "id": "ts_789", "taskId": "task_5001", "hours": 3.0 },
    { "id": "ts_790", "taskId": "task_5002", "hours": 3.5 }
  ]
}
```

> The backend will validate:
> - Sum of hours vs allowed daily limit.  
> - Location/time against task constraints (if you want strict mode).  
> - Whether user is allowed to log against that task.

#### 2.3.3 List My Timesheet History

`GET /timesheets/my/history?from=2025-11-01&to=2025-11-30`

**Output (summary list):**

```jsonc
[
  { "date": "2025-11-28", "status": "approved", "totalHours": 8.0 },
  { "date": "2025-11-29", "status": "submitted", "totalHours": 7.0 },
  { "date": "2025-11-30", "status": "draft",    "totalHours": 6.5 }
]
```

#### 2.3.4 Manager: View Team Timesheets (Summary)

`GET /timesheets/team?date=2025-11-30`

**Output:**

```jsonc
[
  {
    "employeeId": "emp_200",
    "employeeName": "Bob Smith",
    "date": "2025-11-30",
    "status": "submitted",
    "totalHours": 8.0
  },
  {
    "employeeId": "emp_201",
    "employeeName": "Charlie Lee",
    "date": "2025-11-30",
    "status": "draft",
    "totalHours": 4.0
  }
]
```

> Backend enforces that the requester can only see **their own subordinates**.

#### 2.3.5 Manager: View Detailed Timesheet of a Subordinate

`GET /timesheets/{employeeId}?date=2025-11-30`

**Output** (similar to `/timesheets/my`):

```jsonc
{
  "employeeId": "emp_200",
  "employeeName": "Bob Smith",
  "date": "2025-11-30",
  "status": "submitted",
  "entries": [
    {
      "id": "ts_800",
      "taskId": "task_5003",
      "taskTitle": "Site Survey",
      "projectName": "Alpha Plant Expansion",
      "startTime": "08:00",
      "endTime": "16:00",
      "hours": 8.0,
      "location": { "type": "site", "siteId": "site_2", "name": "Construction Site" },
      "notes": ""
    }
  ],
  "totalHours": 8.0
}
```

#### 2.3.6 Manager: Approve / Reject Timesheet (Optional)

`POST /timesheets/{employeeId}/{date}/status`

**Input:**

```json
{
  "status": "approved",  // or "rejected"
  "comment": "Looks good."
}
```

---

### 2.4 Helper Endpoints (Optional but Useful)

To keep the API minimal, these can be added later, but they’re handy:

- `GET /projects/{id}` – full project info (restricted by permissions).  
- `GET /locations` – list of known sites with IDs/coords.  
- `GET /employees/{id}/hierarchy` – manager’s org tree (if you need deeper structure than `/me`).

