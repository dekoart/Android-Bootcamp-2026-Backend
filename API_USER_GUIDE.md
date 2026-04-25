# Bunch API — User Guide

> **Base URL:** `http://localhost:8080`  
> **OpenAPI spec:** [`openapi.yaml`](./openapi.yaml)  
> **Interactive UI** (when the server is running): `http://localhost:8080/swagger-ui.html`

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
   - [Register](#register)
   - [Login](#login)
   - [Refresh tokens](#refresh-tokens)
   - [Logout](#logout)
3. [Meetings](#meetings)
   - [List meetings](#list-meetings)
   - [Get a meeting](#get-a-meeting)
   - [Create a meeting](#create-a-meeting)
   - [Cancel a meeting](#cancel-a-meeting)
   - [Delete a meeting](#delete-a-meeting)
   - [Find free time slots](#find-free-time-slots)
4. [Invitations](#invitations)
   - [List invitations](#list-invitations)
   - [Get invitation details](#get-invitation-details)
   - [Respond to an invitation](#respond-to-an-invitation)
5. [Profile](#profile)
   - [Get profile](#get-profile)
   - [Update profile](#update-profile)
   - [Update avatar](#update-avatar)
   - [Reset password](#reset-password)
   - [List all users](#list-all-users)
6. [Health check](#health-check)
7. [Error handling](#error-handling)
8. [Enumerations](#enumerations)

---

## Overview

**Bunch** is a meeting-scheduling service.  
Users can create meetings, invite other participants, and respond to incoming invitations.

All data is exchanged as JSON (`Content-Type: application/json`).  
Timestamps follow the [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) format in UTC, e.g. `"2026-05-01T10:00:00Z"`.

---

## Authentication

The API uses **JWT Bearer tokens**.

| Token        | Lifetime | Purpose                                  |
|--------------|----------|------------------------------------------|
| Access token | 5 min    | Authenticates every protected request   |
| Refresh token| 7 days   | Obtains a new access token when it expires |

Include the access token in **every protected request**:

```
Authorization: Bearer <accessToken>
```

### Register

Create a new account.

**`POST /api/v1/auth/register`** — no authentication required

```json
// Request body
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "secret123"
}
```

| Field      | Type   | Rules                          |
|------------|--------|--------------------------------|
| `username` | string | 2–50 characters, required      |
| `email`    | string | valid e-mail, ≤ 255 chars, required |
| `password` | string | 6–255 characters, required     |

```json
// 200 OK — response body
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "username": "john_doe",
  "email": "john@example.com",
  "accessToken": "eyJhbGci...",
  "refreshToken": "eyJhbGci...",
  "accessTokenExpiresAt": "2026-04-25T10:05:00Z",
  "refreshTokenExpiresAt": "2026-05-02T10:00:00Z"
}
```

### Login

Authenticate with existing credentials.

**`POST /api/v1/auth/login`** — no authentication required

```json
// Request body
{
  "email": "john@example.com",
  "password": "secret123"
}
```

Response body is identical to `/register`.

### Refresh tokens

When the access token expires, exchange the **refresh token** for a new pair without logging in again.

**`POST /api/v1/auth/refresh`**  
Pass the **refresh token** (not the access token) in the `Authorization` header:

```
Authorization: Bearer <refreshToken>
```

```json
// 200 OK — response body (new token pair)
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "accessToken": "eyJhbGci...",
  "refreshToken": "eyJhbGci...",
  "accessTokenExpiresAt": "2026-04-25T10:10:00Z",
  "refreshTokenExpiresAt": "2026-05-02T10:05:00Z"
}
```

> After a successful refresh, the old refresh token is invalidated. Store the new one.

### Logout

**`POST /api/v1/auth/logout`** — requires access token

```
Authorization: Bearer <accessToken>
```

Returns **`204 No Content`**.  
The server invalidates the stored refresh token; discard both tokens on the client side.

---

## Meetings

### List meetings

**`GET /api/v1/meetings`** — requires authentication

Returns a paginated list of meetings where the authenticated user is either the organizer or a participant.

**Query parameters:**

| Parameter | Type              | Default | Description                        |
|-----------|-------------------|---------|------------------------------------|
| `status`  | `MeetingStatus`   | —       | Filter: `SCHEDULED`, `CANCELLED`, `COMPLETED` |
| `page`    | integer           | `0`     | Zero-based page index              |
| `size`    | integer           | `20`    | Items per page (max 100)           |
| `sort`    | string            | —       | e.g. `startTime,desc`              |

```
GET /api/v1/meetings?status=SCHEDULED&page=0&size=10&sort=startTime,asc
Authorization: Bearer <accessToken>
```

```json
// 200 OK
{
  "content": [ /* MeetingResponse objects */ ],
  "totalElements": 42,
  "totalPages": 5,
  "size": 10,
  "number": 0
}
```

### Get a meeting

**`GET /api/v1/meetings/{meetingId}`** — requires authentication

```
GET /api/v1/meetings/a3bb189e-8bf9-3888-9912-ace4e6543002
Authorization: Bearer <accessToken>
```

```json
// 200 OK
{
  "id": "a3bb189e-8bf9-3888-9912-ace4e6543002",
  "organizerId": "550e8400-e29b-41d4-a716-446655440000",
  "organizerUsername": "john_doe",
  "title": "Weekly sync",
  "description": "Discuss sprint goals",
  "location": "Conference room A",
  "startTime": "2026-05-01T10:00:00Z",
  "endTime": "2026-05-01T11:00:00Z",
  "status": "SCHEDULED",
  "createdAt": "2026-04-25T09:00:00Z",
  "updatedAt": "2026-04-25T09:00:00Z",
  "participants": [
    {
      "userId": "550e8400-e29b-41d4-a716-446655440001",
      "username": "jane_doe",
      "status": "CONFIRMED"
    }
  ]
}
```

### Create a meeting

**`POST /api/v1/meetings`** — requires authentication

The authenticated user becomes the **organizer**. Each UUID in `participantIds` receives a `PENDING` invitation.

```json
// Request body
{
  "title": "Weekly sync",
  "description": "Discuss sprint goals",
  "location": "Conference room A",
  "startTime": "2026-05-01T10:00:00Z",
  "endTime": "2026-05-01T11:00:00Z",
  "participantIds": [
    "550e8400-e29b-41d4-a716-446655440001",
    "550e8400-e29b-41d4-a716-446655440002"
  ]
}
```

| Field            | Type          | Rules                             |
|------------------|---------------|-----------------------------------|
| `title`          | string        | required, ≤ 50 chars              |
| `description`    | string        | optional, ≤ 2 048 chars           |
| `location`       | string        | optional, ≤ 255 chars             |
| `startTime`      | date-time     | required                          |
| `endTime`        | date-time     | required                          |
| `participantIds` | UUID array    | required, ≥ 1 item                |

Returns **`201 Created`** with the `MeetingResponse` body.  
Returns **`409 Conflict`** if the time slot overlaps an existing meeting for any participant.

### Cancel a meeting

**`PUT /api/v1/meetings/{meetingId}/cancel`** — requires authentication, organizer only

```
PUT /api/v1/meetings/a3bb189e-8bf9-3888-9912-ace4e6543002/cancel
Authorization: Bearer <accessToken>
```

Returns **`200 OK`** with the updated `MeetingResponse` (status becomes `CANCELLED`).

### Delete a meeting

**`DELETE /api/v1/meetings/{meetingId}`** — requires authentication, organizer only

```
DELETE /api/v1/meetings/a3bb189e-8bf9-3888-9912-ace4e6543002
Authorization: Bearer <accessToken>
```

Returns **`204 No Content`** on success.

### Find free time slots

**`POST /api/v1/meetings/freeTime`** — requires authentication

Finds **60-minute** windows during which all specified users have no scheduled meetings.

```json
// Request body
{
  "userIds": [
    "550e8400-e29b-41d4-a716-446655440000",
    "550e8400-e29b-41d4-a716-446655440001"
  ]
}
```

```json
// 200 OK
{
  "startEndTime": [
    {
      "startTime": "2026-05-01T08:00:00Z",
      "endTime": "2026-05-01T09:00:00Z"
    },
    {
      "startTime": "2026-05-01T12:00:00Z",
      "endTime": "2026-05-01T13:00:00Z"
    }
  ]
}
```

---

## Invitations

When a meeting is created, every participant listed in `participantIds` automatically receives an invitation with status `PENDING`.

### List invitations

**`GET /api/v1/invitations`** — requires authentication

Returns all invitations for the authenticated user.

```
GET /api/v1/invitations
Authorization: Bearer <accessToken>
```

```json
// 200 OK
[
  {
    "id": "b2cc290f-9cg0-4999-0023-bdf5f7654113",
    "meetingId": "a3bb189e-8bf9-3888-9912-ace4e6543002",
    "meetingTitle": "Weekly sync",
    "meetingDescription": "Discuss sprint goals",
    "meetingLocation": "Conference room A",
    "meetingStartTime": "2026-05-01T10:00:00Z",
    "meetingEndTime": "2026-05-01T11:00:00Z",
    "organizerUsername": "john_doe",
    "status": "PENDING",
    "createdAt": "2026-04-25T09:00:00Z"
  }
]
```

### Get invitation details

**`GET /api/v1/invitations/{meetingId}`** — requires authentication

Returns the authenticated user's invitation for the given meeting.

```
GET /api/v1/invitations/a3bb189e-8bf9-3888-9912-ace4e6543002
Authorization: Bearer <accessToken>
```

### Respond to an invitation

**`PUT /api/v1/invitations/{meetingId}/respond`** — requires authentication

Accept or decline a meeting invitation.

```json
// Request body
{
  "status": "CONFIRMED"
}
```

| `status` value | Meaning              |
|----------------|----------------------|
| `CONFIRMED`    | Accept the invitation |
| `DECLINED`     | Decline the invitation |

Returns **`200 OK`** with the updated `InvitationResponse`.

---

## Profile

### Get profile

**`GET /api/v1/profile`** — requires authentication

```
GET /api/v1/profile
Authorization: Bearer <accessToken>
```

```json
// 200 OK
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "username": "john_doe",
  "email": "john@example.com",
  "avatarUrl": "https://cdn.example.com/avatars/john.png"
}
```

### Update profile

**`PUT /api/v1/profile`** — requires authentication

All fields are optional; only the provided ones are updated.

```json
// Request body
{
  "username": "johnny",
  "avatarUrl": "https://cdn.example.com/avatars/johnny_new.png"
}
```

Returns **`200 OK`** with the updated `UserProfileResponse`.

### Update avatar

**`PUT /api/v1/profile/avatar`** — requires authentication

Updates only the avatar URL.

```json
// Request body
{
  "avatarUrl": "https://cdn.example.com/avatars/new.png"
}
```

Returns **`200 OK`** with the updated `UserProfileResponse`.

### Reset password

**`PUT /api/v1/profile/reset-password`** — requires authentication

> ⚠️ **This endpoint is not yet implemented.** It always returns `501 Not Implemented`.

```json
// Request body (planned)
{
  "currentPassword": "oldSecret123",
  "newPassword": "newSecret456",
  "confirmPassword": "newSecret456"
}
```

### List all users

**`GET /api/v1/profile/public/all`** — requires authentication

Returns a paginated list of all registered users (excluding the current user).
Useful for selecting participants when creating a meeting.

**Query parameters:** `page`, `size`, `sort` (same as [List meetings](#list-meetings))

```
GET /api/v1/profile/public/all?page=0&size=20
Authorization: Bearer <accessToken>
```

```json
// 200 OK
{
  "content": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "username": "jane_doe",
      "email": "jane@example.com",
      "avatarUrl": null
    }
  ],
  "totalElements": 15,
  "totalPages": 1,
  "size": 20,
  "number": 0
}
```

---

## Health check

**`GET /api/v1/health`** — no authentication required

```
GET /api/v1/health
```

```json
// 200 OK
{
  "status": "UP",
  "timestamp": "1745575807000"
}
```

---

## Error handling

All error responses share a common JSON structure:

```json
{
  "timestamp": "2026-04-25T10:00:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed for field 'email': must be a valid email"
}
```

| HTTP Status | Meaning                                                    |
|-------------|------------------------------------------------------------|
| `400`       | Validation error or malformed request body                  |
| `401`       | Missing, expired, or invalid JWT token                     |
| `403`       | Authenticated but not authorised (e.g. not the organizer)  |
| `404`       | Resource not found (meeting, invitation, user)             |
| `409`       | Conflict — scheduling overlap                              |
| `501`       | Not implemented (reset-password)                           |

---

## Enumerations

### `MeetingStatus`

| Value       | Description                              |
|-------------|------------------------------------------|
| `SCHEDULED` | Meeting is upcoming and active           |
| `CANCELLED` | Meeting was cancelled by the organizer   |
| `COMPLETED` | Meeting has taken place                  |

### `ParticipantStatus`

| Value       | Description                                  |
|-------------|----------------------------------------------|
| `PENDING`   | Invitation sent, awaiting response           |
| `CONFIRMED` | Participant accepted the invitation          |
| `DECLINED`  | Participant declined the invitation          |
