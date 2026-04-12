# CaulfieldLife API Documentation

This is unofficial, reverse-engineered documentation for the CaulfieldLife GraphQL API used by Caulfield Grammar School. It was discovered by inspecting network requests made by the CaulfieldLife website at [caulfieldlife.com.au](https://caulfieldlife.com.au).

---

## Base URL

```
https://life-api.caulfieldlife.com.au/
```

All requests are `POST` to this single endpoint. The API is a **GraphQL** API — the operation is determined by the request body, not the URL path.

---

## Authentication

The API uses a two-token system.

### Token 1 — Microsoft JWT (`x-community-token`)

This is a standard Microsoft Azure AD JWT issued by the Caulfield Grammar identity provider at `identity.caulfieldgs.vic.edu.au`. It is generated when the user logs in via Microsoft SSO.

- It is stored in `localStorage` under the key `jwt` on `caulfieldlife.com.au`
- It expires approximately **1 hour** after issuance (`exp` field in the JWT payload)
- It is sent as the `x-community-token` request header on **all** API requests
- It encodes the user's email, name, Microsoft OID, preferred username, and tenant ID

### Token 2 — xhqToken

An internal CaulfieldLife session token. It is a shorter HS256 JWT containing:

- `tokenType`: `"MEMBER"`
- `id`: The internal member identifier (e.g. `Member-XXXX`)
- `communityName`: e.g. `community-cgs-prod`
- `iat`: Issued-at timestamp

This token is returned by the `GetSessionByJwt` mutation and stored in `localStorage` under the key `xhqToken`. Note: in testing this token appeared to be long-lived and potentially cached server-side, meaning repeated calls to `GetSessionByJwt` may return the same token even after clearing local storage.

### Required Request Headers

Every request must include all of the following headers:

```
Content-Type: application/json
authorization: Bearer X-ae97a16a-3e28-4365:Y-53e74b38-8a53-4ef7
origin: https://caulfieldlife.com.au
referer: https://caulfieldlife.com.au/
x-community-token: <Microsoft JWT>
```

The `authorization` header appears to be a static API key shared across all clients. The `origin` and `referer` headers are checked server-side — omitting them results in CORS or authorization errors. The API returns `access-control-allow-origin: *` in responses.

---

## Session Exchange

Before making data requests, the Microsoft JWT must be exchanged for a session containing the user's internal member ID and xhqToken.

### `GetSessionByJwt`

```graphql
query GetSessionByJwt($jwtToken: String!) {
  sessionByJwt(jwtIdToken: $jwtToken) {
    id
    processedAt
    isImpersonating
    xhqToken
    member {
      id
      firstName
      lastName
      preferredName
      roles {
        title
        __typename
      }
      __typename
    }
    sessionSyncCheckpoints {
      success
      type
      error
      __typename
    }
    __typename
  }
}
```

**Variables:**
```json
{
  "jwtToken": "<Microsoft JWT>"
}
```

**Example response:**
```json
{
  "data": {
    "sessionByJwt": {
      "id": "session_id_here",
      "processedAt": "2026-04-12T00:00:00.000Z",
      "isImpersonating": false,
      "xhqToken": "eyJhbGci...",
      "member": {
        "id": "cm2mkjjy...",
        "firstName": "First",
        "lastName": "Last",
        "preferredName": "First",
        "roles": [{ "title": "STUDENT" }]
      },
      "sessionSyncCheckpoints": [
        { "success": true, "type": "MEMBER_DETAILS", "error": null },
        { "success": true, "type": "XHQ_AUTH", "error": null },
        { "success": true, "type": "ROLES", "error": null },
        { "success": true, "type": "CONSENTS", "error": null },
        { "success": true, "type": "RELATIVES", "error": null },
        { "success": true, "type": "SESSION_SYNC", "error": null }
      ]
    }
  }
}
```

**Key fields:**
- `member.id` — the internal member ID, required as `memberId` in subsequent queries
- `xhqToken` — used as `x-community-token` for some downstream requests (behaviour varies)

---

## Timetable / Classes

### `ClassData`

Returns the user's scheduled classes for a given time range. The `x-community-token` for this request must be the **Microsoft JWT** (not the xhqToken).

```graphql
fragment ClassFields on Class {
  id
  title
  description
  startTime
  endTime
  dayOrder
  periodOrder
  periodName
  colour
  room
  teacherName
  __typename
}

query ClassData($memberId: ID!, $startTime: DateTime!, $endTime: DateTime) {
  classes(
    where: { startTime: $startTime, endTime: $endTime, memberId: $memberId }
    orderBy: { property: "startTime", sort: ASC }
  ) {
    ...ClassFields
  }
}
```

**Variables:**
```json
{
  "memberId": "cm2mkjjy...",
  "startTime": "2026-04-11T13:00:00.000Z",
  "endTime": "2026-04-12T12:59:59.999Z"
}
```

> **Important:** Times are in **UTC**. Melbourne is UTC+11 (AEDT) in summer and UTC+10 (AEST) in winter. A Melbourne school day runs from midnight to midnight local time, which in UTC is approximately `T13:00:00Z` to next day `T12:59:59Z` (AEDT) or `T14:00:00Z` to `T13:59:59Z` (AEST). Use the `Intl` API or `toLocaleString` with `timeZone: 'Australia/Melbourne'` to calculate this dynamically.

**Example response:**
```json
{
  "data": {
    "classes": [
      {
        "id": "232241-20220523",
        "title": "W12SDECA0",
        "description": "SOFTWARE DEVELOPMENT - UNIT 3/4 W12",
        "startTime": "2026-04-11T22:40:00.000Z",
        "endTime": "2026-04-11T23:30:00.000Z",
        "dayOrder": 1,
        "periodOrder": 3,
        "periodName": "1",
        "colour": "#8D6CAB",
        "room": "CAD",
        "teacherName": "Ms Jane Smith",
        "__typename": "Class"
      }
    ]
  }
}
```

**Field reference:**

| Field | Type | Description |
|---|---|---|
| `id` | String | Unique class instance ID, format `classId-YYYYMMDD` |
| `title` | String | Short code identifier (e.g. `W12SDECA0`) |
| `description` | String | Human-readable subject name |
| `startTime` | DateTime | Class start time in UTC ISO 8601 |
| `endTime` | DateTime | Class end time in UTC ISO 8601 |
| `dayOrder` | Int | Day number within the school cycle |
| `periodOrder` | Int | Period slot within the day |
| `periodName` | String | Display period label (e.g. `"1"`, `"2"`) |
| `colour` | String | Hex colour for the subject (e.g. `"#8D6CAB"`) |
| `room` | String | Room code where class is held |
| `teacherName` | String | Full name of the teacher |

---

## Events / Daily Planner

### `DailyPlannerData`

Returns events and calendar events for a given time range and member.

```graphql
fragment EventFields on Event {
  id
  category
  title
  type
  sportEventType
  musicEventType
  organisers {
    id
    firstName
    lastName
    avatar
    roles { id title capabilities slug __typename }
    __typename
  }
  attendees {
    id
    member { id avatar __typename }
    __typename
  }
  location { id details mapLink lat long createdAt updatedAt __typename }
  description
  urls { id text url note createdAt updatedAt __typename }
  transportVehicles {
    id type description departureTime returnTime vehicleId __typename
  }
  startTime
  endTime
  createdAt
  updatedAt
  __typename
}

fragment CalendarEventFields on CalendarEvent {
  id
  title
  description
  startTime
  endTime
  __typename
}

query DailyPlannerData(
  $attendeeId: ID,
  $organiserId: ID,
  $startTime: DateTime,
  $endTime: DateTime
) {
  events(
    where: {
      startTime_gte: $startTime,
      endTime_lte: $endTime,
      OR: [
        { attendeeMemberId: $attendeeId, OR: [{ isDeleted: false }, { isDeleted: null }, { deletedAt: null }] },
        { organiserId: $organiserId, OR: [{ isDeleted: false }, { isDeleted: null }, { deletedAt: null }] }
      ],
      isArchived: false,
      isPublishedToPortal: true
    }
    orderBy: { property: "startTime", sort: ASC }
  ) {
    ...EventFields
    __typename
  }
  calendarEvents(where: { startTime_gt: $startTime, endTime_lt: $endTime }) {
    ...CalendarEventFields
    __typename
  }
}
```

**Variables:**
```json
{
  "attendeeId": "cm2mkjjy...",
  "organiserId": "cm2mkjjy...",
  "startTime": "2026-04-11T13:00:00.000Z",
  "endTime": "2026-04-12T12:59:59.999Z"
}
```

Both `attendeeId` and `organiserId` should be set to the member's ID to retrieve all events they are involved in. The same UTC date range convention applies as for `ClassData`.

---

## Error Handling

When authentication fails the API returns HTTP 200 with an errors array:

```json
{
  "errors": [{
    "message": "Access not allowed. You can't perform this action",
    "name": "Unauthorized",
    "time_thrown": "2026-04-12T00:00:00.000Z",
    "data": {
      "message": "User not logged with the token: eyJ..."
    }
  }],
  "data": { "classes": null }
}
```

Common causes:
- The Microsoft JWT has expired (1 hour TTL)
- The `origin` or `authorization` headers are missing
- The xhqToken is stale (server-side session cache issue)

---

## LocalStorage Keys (caulfieldlife.com.au)

| Key | Value |
|---|---|
| `jwt` | Microsoft Azure AD JWT (expires ~1 hour) |
| `xhqToken` | Internal CaulfieldLife session token |

---

## Notes

- This API is **undocumented and unofficial**. It may change without notice.
- The static `authorization` bearer token is embedded in the CaulfieldLife web app and is the same for all users.
- The API uses **GraphQL** — all operations go to the same endpoint with different query bodies.
- There is no public registration or API key issuance. Access requires a valid Caulfield Grammar School Microsoft account.
- The timetable `ClassData` query only returns results when both `startTime` and `endTime` are supplied and fall within a valid school timetable period. Querying future dates outside the timetable cycle returns an empty array.
