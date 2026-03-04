# WebUntis API - Protocol-Level Documentation

---

## Table of Contents

- [Overview](#overview)
- [Base Configuration](#base-configuration)
- [Data Formats](#data-formats)
- [Authentication](#authentication)
  - [Standard Login (Username/Password)](#1-standard-login-usernamepassword)
  - [OTP/Secret Login](#2-otpsecret-login)
  - [Anonymous Login](#3-anonymous-login)
  - [Logout](#4-logout)
- [Session Management](#session-management)
  - [Cookies](#cookies)
  - [JWT Tokens](#jwt-tokens)
  - [Session Validation](#session-validation)
- [JSON-RPC Endpoints](#json-rpc-endpoints)
  - [getTimetable](#gettimetable)
  - [getSchoolyears](#getschoolyears)
  - [getCurrentSchoolyear](#getcurrentschoolyear)
  - [getSubjects](#getsubjects)
  - [getTeachers](#getteachers)
  - [getStudents](#getstudents)
  - [getRooms](#getrooms)
  - [getKlassen](#getklassen)
  - [getDepartments](#getdepartments)
  - [getHolidays](#getholidays)
  - [getTimegridUnits](#gettimegridunits)
  - [getStatusData](#getstatusdata)
  - [getLatestImportTime](#getlatestimporttime)
- [REST Endpoints](#rest-endpoints)
  - [News Widget](#get-news-widget)
  - [Inbox Messages](#get-inbox-messages)
  - [JWT Token](#get-jwt-token)
  - [Homework](#get-homework)
  - [Exams](#get-exams)
  - [Weekly Timetable](#get-weekly-timetable)
  - [Student Absences](#get-student-absences)
  - [PDF Absence Report](#get-pdf-absence-report)
  - [App Config](#get-app-config)
  - [Day Timetable Config](#get-day-timetable-config)
- [Error Handling](#error-handling)
- [Endpoint Summary](#endpoint-summary)

---

## Overview

The WebUntis API exposes two types of endpoints:

| Type | Base Path | Protocol | Auth |
|------|-----------|----------|------|
| **JSON-RPC 2.0** | `/WebUntis/jsonrpc.do` | POST | Session cookie |
| **JSON-RPC Internal** | `/WebUntis/jsonrpc_intern.do` | POST | OTP token |
| **REST** | `/WebUntis/api/*` | GET | Session cookie or JWT |
| **Reports** | `/WebUntis/reports.do` | GET | Session cookie |

All communication is over HTTPS.

---

## Base Configuration

**Base URL pattern:** `https://{hostname}/`

Example: `https://mese.webuntis.com/`

**Standard headers on all requests:**
```http
Cache-Control: no-cache
Pragma: no-cache
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.79 Safari/537.36
```

**HTTP configuration:**
- Max redirects: 0 (do not follow redirects)
- Accepted status codes: 200-302

---

## Data Formats

### Date Format

**Format:** `yyyyMMdd` (8-digit integer or string)

| Value | Meaning |
|-------|---------|
| `20191113` | November 13, 2019 |
| `20230901` | September 1, 2023 |

Some servers return dates as numbers, others as strings. Handle both.

### Time Format

**Format:** `Hmm` or `HHmm` (integer, 24-hour clock, no leading zeros)

| Value | Meaning |
|-------|---------|
| `830` | 08:30 |
| `1430` | 14:30 |
| `311` | 03:11 |
| `0` | 00:00 |

To parse: pad to 4 digits with leading zeros, then split as `HHmm`.

### ISO Date Format

Used only by the weekly timetable REST endpoint:

**Format:** `yyyy-MM-dd`

| Value | Meaning |
|-------|---------|
| `2023-12-25` | December 25, 2023 |

### Element Types

```
1 = CLASS (Klasse)
2 = TEACHER
3 = SUBJECT
4 = ROOM
5 = STUDENT
```

### Day of Week

```
1 = Sunday
2 = Monday
3 = Tuesday
4 = Wednesday
5 = Thursday
6 = Friday
7 = Saturday
```

---

## Authentication

### 1. Standard Login (Username/Password)

**Endpoint:** `POST /WebUntis/jsonrpc.do?school={school}`

**Request:**
```json
{
  "id": "Awesome",
  "method": "authenticate",
  "params": {
    "user": "username",
    "password": "password",
    "client": "Awesome"
  },
  "jsonrpc": "2.0"
}
```

**Success Response:**
```json
{
  "result": {
    "sessionId": "ABC123DEF456...",
    "personId": 12345,
    "personType": 5,
    "klasseId": 67890
  },
  "jsonrpc": "2.0"
}
```

**Error Response:**
```json
{
  "result": {
    "code": 4010
  },
  "jsonrpc": "2.0"
}
```

After successful login, store `sessionId` for all subsequent requests.

---

### 2. OTP/Secret Login

A multi-step flow used for secret-based and QR-code authentication.

#### Step 1: Authenticate with OTP

**Endpoint:** `POST /WebUntis/jsonrpc_intern.do?m=getUserData2017&school={school}&v=i2.2`

**Request:**
```json
{
  "id": "Awesome",
  "method": "getUserData2017",
  "params": [
    {
      "auth": {
        "clientTime": 1647362400000,
        "user": "username",
        "otp": 123456
      }
    }
  ],
  "jsonrpc": "2.0"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `clientTime` | number | Current Unix timestamp in milliseconds |
| `user` | string | Username |
| `otp` | number | 6-digit TOTP token generated from secret |

**Response:** The `JSESSIONID` is returned in the `Set-Cookie` response header. Extract it:

```
Set-Cookie: JSESSIONID=ABC123...; Path=/WebUntis; ...
```

**Error Response:**
```json
{
  "error": {
    "message": "..."
  },
  "jsonrpc": "2.0"
}
```

#### Step 2: Fetch App Config

**Endpoint:** `GET /WebUntis/api/app/config`

**Headers:** Session cookies (see [Cookies](#cookies))

**Response:**
```json
{
  "data": {
    "loginServiceConfig": {
      "user": {
        "personId": 12345,
        "persons": [
          { "id": 12345, "type": 5 }
        ]
      }
    }
  }
}
```

Extract `personId` and `personType` from the matching person entry.

#### Step 3: Fetch Klasse ID (optional)

**Endpoint:** `GET /WebUntis/api/daytimetable/config`

**Headers:** Session cookies

**Response:**
```json
{
  "data": {
    "klasseId": 67890
  }
}
```

This step may fail; `klasseId` is not critical.

---

### 3. Anonymous Login

For schools that allow public timetable access.

#### Step 1: Get Shared Secret

**Endpoint:** `POST /WebUntis/jsonrpc_intern.do?m=getAppSharedSecret&school={school}&v=i3.5`

**Request:**
```json
{
  "id": "Awesome",
  "method": "getAppSharedSecret",
  "params": [
    {
      "userName": "#anonymous#",
      "password": ""
    }
  ],
  "jsonrpc": "2.0"
}
```

#### Step 2: OTP Login with Fixed Token

Use the OTP login flow (Section 2, Step 1) with:

| Field | Value |
|-------|-------|
| `user` | `#anonymous#` |
| `otp` | `100170` (hardcoded constant) |

Skip Steps 2-3 (no app config fetch needed).

---

### 4. Logout

**Endpoint:** `POST /WebUntis/jsonrpc.do?school={school}`

**Request:**
```json
{
  "id": "Awesome",
  "method": "logout",
  "params": {},
  "jsonrpc": "2.0"
}
```

**Response:**
```json
{
  "result": {},
  "jsonrpc": "2.0"
}
```

---

## Session Management

### Cookies

All authenticated requests require these cookies:

```http
Cookie: JSESSIONID={sessionId}; schoolname=_{base64(school)}
```

| Cookie | Value | Example |
|--------|-------|---------|
| `JSESSIONID` | Session ID from login | `ABC123DEF456` |
| `schoolname` | `_` + Base64 of school name | `_bWVzZQ==` (for "mese") |

The school name is Base64-encoded (standard alphabet `A-Za-z0-9+/=`) and prefixed with underscore `_`.

### JWT Tokens

Some REST endpoints require JWT Bearer authentication (e.g., inbox messages).

**Obtain a token:**
```
GET /WebUntis/api/token/new
Cookie: JSESSIONID=...; schoolname=...
```

**Response:** Plain text JWT string (not JSON).

**Use in requests:**
```http
Authorization: Bearer {jwt_token}
Cookie: JSESSIONID=...; schoolname=...
```

### Session Validation

Sessions expire after ~10 minutes of inactivity. To check validity:

```
POST /WebUntis/jsonrpc.do?school={school}
```
```json
{
  "id": "Awesome",
  "method": "getLatestImportTime",
  "params": {},
  "jsonrpc": "2.0"
}
```

Valid if `response.result` is a number (Unix timestamp).

---

## JSON-RPC Endpoints

All JSON-RPC endpoints use the same base pattern:

```
POST /WebUntis/jsonrpc.do?school={school}
Content-Type: application/json
Cookie: JSESSIONID={sessionId}; schoolname=_{base64(school)}
```

Request body:
```json
{
  "id": "{identity}",
  "method": "{method_name}",
  "params": {},
  "jsonrpc": "2.0"
}
```

Response envelope:
```json
{
  "result": { ... },
  "jsonrpc": "2.0"
}
```

---

### getTimetable

Get timetable for any element (class, teacher, room, student, subject).

**Method:** `getTimetable`

**Params:**
```json
{
  "options": {
    "id": 1647362400000,
    "element": {
      "id": 123,
      "type": 1
    },
    "startDate": "20230901",
    "endDate": "20230901",
    "showLsText": true,
    "showStudentgroup": true,
    "showLsNumber": true,
    "showSubstText": true,
    "showInfo": true,
    "showBooking": true,
    "klasseFields": ["id", "name", "longname", "externalkey"],
    "roomFields": ["id", "name", "longname", "externalkey"],
    "subjectFields": ["id", "name", "longname", "externalkey"],
    "teacherFields": ["id", "name", "longname", "externalkey"]
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `options.id` | number | Timestamp (used as request ID) |
| `options.element.id` | number | Element ID |
| `options.element.type` | number | 1=CLASS, 2=TEACHER, 3=SUBJECT, 4=ROOM, 5=STUDENT |
| `options.startDate` | string | Start date (`yyyyMMdd`) |
| `options.endDate` | string | End date (`yyyyMMdd`) |
| `options.show*` | boolean | Toggle optional fields |
| `options.*Fields` | string[] | Fields to include per entity |

**Response:**
```json
{
  "result": [
    {
      "id": 1001,
      "date": 20230901,
      "startTime": 800,
      "endTime": 850,
      "kl": [
        { "id": 1, "name": "1A", "longname": "Class 1A", "externalkey": "" }
      ],
      "te": [
        { "id": 10, "name": "SMI", "longname": "Smith", "externalkey": "" }
      ],
      "su": [
        { "id": 20, "name": "MA", "longname": "Mathematics", "externalkey": "" }
      ],
      "ro": [
        { "id": 30, "name": "R101", "longname": "Room 101", "externalkey": "" }
      ],
      "lstext": "",
      "lsnumber": 1,
      "activityType": "Unterricht",
      "code": "cancelled",
      "info": "Teacher absent",
      "substText": "Replaced by...",
      "statflags": "",
      "sg": "",
      "bkRemark": "",
      "bkText": ""
    }
  ]
}
```

| Response Field | Type | Description |
|----------------|------|-------------|
| `date` | number | Date in `yyyyMMdd` format |
| `startTime` / `endTime` | number | Time in `Hmm` format |
| `kl` | array | Classes involved |
| `te` | array | Teachers |
| `su` | array | Subjects |
| `ro` | array | Rooms |
| `code` | string | `"cancelled"` or `"irregular"` (if applicable) |
| `info` | string | Additional info text |
| `substText` | string | Substitution text |

---

### getSchoolyears

**Method:** `getSchoolyears`

**Params:** `{}`

**Response:**
```json
{
  "result": [
    {
      "id": 5,
      "name": "2023/2024",
      "startDate": "20230901",
      "endDate": "20240630"
    }
  ]
}
```

> Dates may be returned as numbers or strings depending on the server.

---

### getCurrentSchoolyear

**Method:** `getCurrentSchoolyear`

**Params:** `{}`

**Response:**
```json
{
  "result": {
    "id": 5,
    "name": "2023/2024",
    "startDate": "20230901",
    "endDate": "20240630"
  }
}
```

---

### getSubjects

**Method:** `getSubjects`

**Params:** `{}`

**Response:**
```json
{
  "result": [
    {
      "id": 1,
      "name": "MA",
      "longName": "Mathematics",
      "alternateName": "",
      "active": true,
      "foreColor": "000000",
      "backColor": "FFFFFF"
    }
  ]
}
```

---

### getTeachers

**Method:** `getTeachers`

**Params:** `{}`

**Response:**
```json
{
  "result": [
    {
      "id": 1,
      "name": "SMI",
      "foreName": "John",
      "longName": "Smith",
      "foreColor": "000000",
      "backColor": "FFFFFF"
    }
  ]
}
```

---

### getStudents

**Method:** `getStudents`

**Params:** `{}`

**Response:**
```json
{
  "result": [
    {
      "id": 1,
      "key": 12345,
      "name": "Doe",
      "foreName": "Jane",
      "longName": "Doe Jane",
      "gender": "female"
    }
  ]
}
```

---

### getRooms

**Method:** `getRooms`

**Params:** `{}`

**Response:**
```json
{
  "result": [
    {
      "id": 1,
      "name": "R101",
      "longName": "Room 101",
      "alternateName": "",
      "active": true,
      "foreColor": "000000",
      "backColor": "FFFFFF"
    }
  ]
}
```

---

### getKlassen

**Method:** `getKlassen`

**Params:**
```json
{
  "schoolyearId": 5
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schoolyearId` | number | No | Filter by school year |

**Response:**
```json
{
  "result": [
    {
      "id": 1,
      "name": "1A",
      "longName": "Class 1A",
      "active": true,
      "foreColor": "000000",
      "backColor": "FFFFFF",
      "did": 1,
      "teacher1": 10,
      "teacher2": 11
    }
  ]
}
```

---

### getDepartments

**Method:** `getDepartments`

**Params:** `{}`

**Response:**
```json
{
  "result": [
    {
      "id": 1,
      "name": "SCI",
      "longName": "Science Department"
    }
  ]
}
```

---

### getHolidays

**Method:** `getHolidays`

**Params:** `{}`

**Response:**
```json
{
  "result": [
    {
      "id": 1,
      "name": "Christmas",
      "longName": "Christmas Break",
      "startDate": 20231223,
      "endDate": 20240107
    }
  ]
}
```

---

### getTimegridUnits

**Method:** `getTimegridUnits`

**Params:** `{}`

**Response:**
```json
{
  "result": [
    {
      "day": 2,
      "timeUnits": [
        { "name": "1st Period", "startTime": 800, "endTime": 850 },
        { "name": "2nd Period", "startTime": 855, "endTime": 945 }
      ]
    }
  ]
}
```

`day` values: 1=Sunday through 7=Saturday.

---

### getStatusData

**Method:** `getStatusData`

**Params:** `{}`

**Response:**
```json
{
  "result": {
    "lstypes": [
      {
        "ls": { "foreColor": "000000", "backColor": "FFFFFF" },
        "oh": { "foreColor": "000000", "backColor": "EEEEEE" },
        "sb": null,
        "bs": null,
        "ex": { "foreColor": "FF0000", "backColor": "FFEEEE" }
      }
    ],
    "codes": [
      {
        "cancelled": { "foreColor": "FF0000", "backColor": "FFFFFF" },
        "irregular": { "foreColor": "FF8800", "backColor": "FFFFFF" }
      }
    ]
  }
}
```

| Key | Meaning |
|-----|---------|
| `ls` | Lesson |
| `oh` | Office hours |
| `sb` | Standby |
| `bs` | Base status |
| `ex` | Exam |
| `cancelled` | Cancelled lesson color |
| `irregular` | Irregular lesson color |

---

### getLatestImportTime

**Method:** `getLatestImportTime`

**Params:** `{}`

**Response:**
```json
{
  "result": 1647362400123
}
```

Returns a Unix timestamp (milliseconds) of the last data import.

---

## REST Endpoints

All REST endpoints use GET and require session cookies unless noted otherwise.

**Common headers:**
```http
Cookie: JSESSIONID={sessionId}; schoolname=_{base64(school)}
```

---

### GET News Widget

```
GET /WebUntis/api/public/news/newsWidgetData?date={yyyyMMdd}
```

| Param | Type | Description |
|-------|------|-------------|
| `date` | string | Date in Untis format |

**Response:**
```json
{
  "data": {
    "systemMessage": null,
    "messagesOfDay": [
      {
        "id": 1,
        "subject": "School Event",
        "text": "Tomorrow is sports day...",
        "isExpanded": false,
        "attachments": []
      }
    ],
    "rssUrl": "https://..."
  }
}
```

---

### GET Inbox Messages

```
GET /WebUntis/api/rest/view/v1/messages
Authorization: Bearer {jwt_token}
```

**Requires:** JWT Bearer token (see [JWT Tokens](#jwt-tokens))

**Response:**
```json
{
  "incomingMessages": [
    {
      "id": 1,
      "subject": "Homework reminder",
      "contentPreview": "Please submit your...",
      "sender": {
        "userId": 100,
        "displayName": "Mr. Smith",
        "imageUrl": "",
        "className": "1A"
      },
      "sentDateTime": "2023-09-01T10:30:00",
      "isMessageRead": false,
      "isReply": false,
      "isReplyAllowed": true,
      "hasAttachments": false,
      "allowMessageDeletion": true
    }
  ]
}
```

Not available with anonymous login.

---

### GET JWT Token

```
GET /WebUntis/api/token/new
```

**Response:** Plain text JWT token string (not JSON).

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

### GET Homework

```
GET /WebUntis/api/homeworks/lessons?startDate={yyyyMMdd}&endDate={yyyyMMdd}
```

| Param | Type | Description |
|-------|------|-------------|
| `startDate` | string | Start date in Untis format |
| `endDate` | string | End date in Untis format |

**Response:**
```json
{
  "data": {
    "homeworks": [
      {
        "id": 1,
        "lessonId": 100,
        "date": 20230901,
        "dueDate": 20230908,
        "text": "Complete exercises 1-10",
        "remark": "",
        "completed": false,
        "attachments": []
      }
    ]
  }
}
```

---

### GET Exams

```
GET /WebUntis/api/exams?startDate={yyyyMMdd}&endDate={yyyyMMdd}&klasseId={id}&withGrades={bool}
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `startDate` | string | | Start date in Untis format |
| `endDate` | string | | End date in Untis format |
| `klasseId` | number | `-1` | Class filter (-1 = all) |
| `withGrades` | boolean | `false` | Include grade data |

**Response:**
```json
{
  "data": {
    "exams": [
      {
        "id": 1,
        "examType": "Test",
        "name": "Math Midterm",
        "studentClass": ["1A"],
        "assignedStudents": [
          {
            "id": 50,
            "displayName": "Doe Jane",
            "klasse": { "id": 1, "name": "1A" }
          }
        ],
        "examDate": 20231015,
        "startTime": 800,
        "endTime": 945,
        "subject": "Mathematics",
        "teachers": ["Smith"],
        "rooms": ["R101"],
        "text": "Chapters 1-5",
        "grade": "A"
      }
    ]
  }
}
```

---

### GET Weekly Timetable

```
GET /WebUntis/api/public/timetable/weekly/data?elementType={type}&elementId={id}&date={yyyy-MM-dd}&formatId={format}
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `elementType` | number | | 1=CLASS, 2=TEACHER, 3=SUBJECT, 4=ROOM, 5=STUDENT |
| `elementId` | number | | Element ID |
| `date` | string | | ISO date (`yyyy-MM-dd`) - any day in the target week |
| `formatId` | number | `1` | 1 = include teachers, 2 = omit teachers |

> **Note:** This endpoint uses ISO date format (`yyyy-MM-dd`), unlike all other endpoints.

**Response:**
```json
{
  "data": {
    "result": {
      "data": {
        "elementPeriods": {
          "123": [
            {
              "id": 5001,
              "lessonId": 2001,
              "lessonNumber": 1,
              "lessonCode": "",
              "lessonText": "",
              "periodText": "",
              "hasPeriodText": false,
              "periodInfo": "",
              "periodAttachments": [],
              "substText": "",
              "date": 20230904,
              "startTime": 800,
              "endTime": 850,
              "elements": [
                {
                  "type": 1,
                  "id": 123,
                  "orgId": 123,
                  "missing": false,
                  "state": "REGULAR"
                }
              ],
              "studentGroup": "",
              "hasInfo": false,
              "code": 0,
              "cellState": "STANDARD",
              "priority": 0,
              "is": {
                "standard": true,
                "event": false
              },
              "roomCapacity": 30,
              "studentCount": 25
            }
          ]
        },
        "elements": [
          {
            "type": 1,
            "id": 123,
            "name": "1A",
            "longName": "Class 1A",
            "displayname": "1A",
            "alternatename": "",
            "canViewTimetable": true,
            "externalKey": "",
            "roomCapacity": 0
          }
        ]
      }
    }
  }
}
```

| `cellState` Value | Meaning |
|-------------------|---------|
| `STANDARD` | Normal lesson |
| `SUBSTITUTION` | Teacher substitution |
| `ROOMSUBSTITUTION` | Room change |

| `elements[].state` Value | Meaning |
|--------------------------|---------|
| `REGULAR` | Normal |
| `ABSENT` | Absent |
| `SUBSTITUTED` | Replaced by substitute |

**Error Response:**
```json
{
  "data": {
    "error": {
      "data": {
        "messageKey": "ERR_TTVIEW_NOTALLOWED_ONDATE"
      }
    }
  }
}
```

---

### GET Student Absences

```
GET /WebUntis/api/classreg/absences/students?startDate={yyyyMMdd}&endDate={yyyyMMdd}&studentId={id}&excuseStatusId={statusId}
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `startDate` | string | | Start date in Untis format |
| `endDate` | string | | End date in Untis format |
| `studentId` | number | | Student ID (from session `personId`) |
| `excuseStatusId` | number | `-1` | Excuse status filter (-1 = all) |

Not available with anonymous login.

**Response:**
```json
{
  "data": {
    "absences": [
      {
        "id": 1,
        "startDate": 20230905,
        "endDate": 20230905,
        "startTime": 800,
        "endTime": 1200,
        "createDate": 20230905,
        "lastUpdate": 20230906,
        "createdUser": "admin",
        "updatedUser": "admin",
        "reasonId": 1,
        "reason": "Illness",
        "text": "",
        "interruptions": [],
        "canEdit": true,
        "studentName": "Doe Jane",
        "excuseStatus": "excused",
        "isExcused": true,
        "excuse": {
          "id": 1,
          "text": "Doctor's note provided",
          "excuseDate": 20230906,
          "excuseStatus": "excused",
          "isExcused": true,
          "userId": 100,
          "username": "admin"
        }
      }
    ],
    "absenceReasons": [],
    "excuseStatuses": true,
    "showAbsenceReasonChange": true,
    "showCreateAbsence": false
  }
}
```

---

### GET PDF Absence Report

```
GET /WebUntis/reports.do?name=Excuse&format=pdf&rpt_sd={yyyyMMdd}&rpt_ed={yyyyMMdd}&studentId={id}&excuseStatusId={statusId}&withLateness={bool}&withAbsences={bool}&execuseGroup={groupId}
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | `Excuse` | Report name |
| `format` | string | `pdf` | Output format |
| `rpt_sd` | string | | Report start date |
| `rpt_ed` | string | | Report end date |
| `studentId` | number | | Student ID |
| `excuseStatusId` | number | `-1` | Excuse status filter |
| `withLateness` | boolean | `true` | Include lateness records |
| `withAbsences` | boolean | `true` | Include absence records |
| `execuseGroup` | number | `2` | Excuse group |

Not available with anonymous login.

**Response:**
```json
{
  "data": {
    "error": null,
    "messageId": "msg-123-abc",
    "reportParams": "param1=value1&param2=value2"
  }
}
```

**PDF download URL:** `https://{hostname}/WebUntis/reports.do?msgId={messageId}&{reportParams}`

---

### GET App Config

```
GET /WebUntis/api/app/config
```

Used during OTP login to fetch person information.

**Response:**
```json
{
  "data": {
    "loginServiceConfig": {
      "user": {
        "personId": 12345,
        "persons": [
          { "id": 12345, "type": 5 }
        ]
      }
    }
  }
}
```

---

### GET Day Timetable Config

```
GET /WebUntis/api/daytimetable/config
```

Used during OTP login to fetch klasse ID.

**Response:**
```json
{
  "data": {
    "klasseId": 67890
  }
}
```

---

## Error Handling

### JSON-RPC Error Envelope

**Application error in result:**
```json
{
  "result": {
    "code": 4010
  },
  "jsonrpc": "2.0"
}
```

**Protocol error:**
```json
{
  "error": {
    "message": "invalid method name",
    "code": -32601
  },
  "jsonrpc": "2.0"
}
```

### REST API Error Envelope

```json
{
  "data": {
    "error": {
      "data": {
        "messageKey": "ERR_TTVIEW_NOTALLOWED_ONDATE"
      }
    }
  }
}
```

### Common Error Scenarios

| Scenario | Indicator |
|----------|-----------|
| Invalid credentials | `result.code` is set after authenticate |
| Session expired | JSON-RPC call returns error / non-number result |
| Missing Set-Cookie on OTP login | No `JSESSIONID` in response headers |
| Timetable not allowed | `error.data.messageKey = "ERR_TTVIEW_NOTALLOWED_ONDATE"` |
| Anonymous method restriction | Various auth-required endpoints return errors |

---

## Endpoint Summary

### JSON-RPC (`POST /WebUntis/jsonrpc.do?school={school}`)

| Method | Params | Returns |
|--------|--------|---------|
| `authenticate` | `user`, `password`, `client` | Session info |
| `logout` | (none) | Empty |
| `getLatestImportTime` | (none) | Timestamp |
| `getTimetable` | `options` (element, dates, fields) | Lesson array |
| `getSchoolyears` | (none) | SchoolYear array |
| `getCurrentSchoolyear` | (none) | SchoolYear |
| `getSubjects` | (none) | Subject array |
| `getTeachers` | (none) | Teacher array |
| `getStudents` | (none) | Student array |
| `getRooms` | (none) | Room array |
| `getKlassen` | `schoolyearId` (optional) | Klasse array |
| `getDepartments` | (none) | Department array |
| `getHolidays` | (none) | Holiday array |
| `getTimegridUnits` | (none) | Timegrid array |
| `getStatusData` | (none) | StatusData |

### JSON-RPC Internal (`POST /WebUntis/jsonrpc_intern.do`)

| Method | Query Params | Purpose |
|--------|-------------|---------|
| `getUserData2017` | `m=getUserData2017&school={s}&v=i2.2` | OTP login |
| `getAppSharedSecret` | `m=getAppSharedSecret&school={s}&v=i3.5` | Anonymous secret |

### REST (`GET /WebUntis/api/...`)

| Path | Auth | Purpose |
|------|------|---------|
| `/api/public/news/newsWidgetData` | Cookie | News & messages of day |
| `/api/rest/view/v1/messages` | JWT + Cookie | Inbox messages |
| `/api/token/new` | Cookie | Generate JWT |
| `/api/homeworks/lessons` | Cookie | Homework |
| `/api/exams` | Cookie | Exams |
| `/api/public/timetable/weekly/data` | Cookie | Weekly timetable (rich) |
| `/api/classreg/absences/students` | Cookie | Student absences |
| `/api/app/config` | Cookie | App/login config |
| `/api/daytimetable/config` | Cookie | Timetable config |
| `/reports.do` | Cookie | PDF report generation |
