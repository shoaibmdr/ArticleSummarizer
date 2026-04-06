# Broker Account Creation API

## Overview

This endpoint registers a new third-party member (broker) account in the RMS system. It stores the member's credentials in the `THIRD_PARTY_MEMBER` table with a **BCrypt-hashed** password and links the account to an RMS member via `memberCode`.

> **Important:** This is an **administrative / provisioning** endpoint. It is **not** protected by the `@Broker` token filter — no `Authorization` header is required. Access control should be enforced at the network / gateway level.

---

## Endpoint

```
POST /broker/create
```

---

## Headers

| Header         | Value              | Required |
|----------------|--------------------|----------|
| `Content-Type` | `application/json` | Yes      |

---

## Request Body

| Field        | Type      | Required | Default | Description                                                                 |
|--------------|-----------|----------|---------|-----------------------------------------------------------------------------|
| `username`   | `String`  | Yes      | —       | Unique username for the member. Must not already exist in the system.       |
| `password`   | `String`  | Yes      | —       | Plain-text password. Will be hashed with **BCrypt** (cost factor 12) before storage. |
| `memberCode` | `String`  | No       | `null`  | The member code that maps to `RMS_MEMBERS.MEMBER_CODE`. Used during authentication to resolve `memberId` for token claims. |
| `isActive`   | `Boolean` | No       | `true`  | Whether the account is active. Inactive accounts cannot authenticate.       |

### Sample Request

```json
{
  "username": "broker01",
  "password": "secureP@ssword",
  "memberCode": "26",
  "isActive": true
}
```

### Minimal Request (only required fields)

```json
{
  "username": "broker01",
  "password": "secureP@ssword"
}
```

---

## Response

### Success — `200 OK`

Returned when the member account is created successfully.

| Field        | Type      | Description                                  |
|--------------|-----------|----------------------------------------------|
| `id`         | `Integer` | Auto-generated primary key of the new record |
| `username`   | `String`  | The registered username                      |
| `memberCode` | `String`  | The member code (or `null` if not provided)  |
| `isActive`   | `Boolean` | Active status of the account                 |
| `message`    | `String`  | Confirmation message                         |

```json
{
  "id": 1,
  "username": "broker01",
  "memberCode": "26",
  "isActive": true,
  "message": "Broker created successfully"
}
```

### Failure — `400 Bad Request`

**Missing credentials:**

```json
{
  "code": "1",
  "message": "username and password are required"
}
```

**Duplicate username:**

```json
{
  "code": "1",
  "message": "Username already exists"
}
```

### Failure — `500 Internal Server Error`

Returned on unexpected server-side errors.

```json
{
  "code": "1",
  "message": "Error creating broker: <exception detail>"
}
```

---

## Business Logic

1. **Uniqueness check** — The service looks up the `THIRD_PARTY_MEMBER` table by `username`. If a record already exists, the request is rejected with *"Username already exists"*.

2. **Password hashing** — The plain-text password is hashed using **BCrypt** with a salt round cost of **12** before being persisted. The original password is never stored.

3. **Account creation** — A new row is inserted into `THIRD_PARTY_MEMBER` with:
   - `Username` = provided username
   - `Password` = BCrypt hash
   - `MemberCode` = provided member code (nullable)
   - `IsActive` = provided value or defaults to `true`
   - `CreatedDate` = current server timestamp

4. **Member code significance** — When this account later authenticates via `POST /broker/broker-authentication`, the `memberCode` is used to look up `RMS_MEMBERS.ID` to resolve the `memberId` claim in the JWT. If `memberCode` is not set, protected endpoints that require `memberId` will fail validation.

---

## Database Table

**`THIRD_PARTY_MEMBER`**

| Column        | Type              | Nullable | Description                                      |
|---------------|-------------------|----------|--------------------------------------------------|
| `id`          | `INT` (IDENTITY)  | No       | Primary key, auto-incremented                    |
| `Username`    | `NVARCHAR(100)`   | No       | Unique member username                           |
| `Password`    | `NVARCHAR(MAX)`   | No       | BCrypt-hashed password                           |
| `MemberCode`  | `NVARCHAR(255)`   | Yes      | Maps to `RMS_MEMBERS.MEMBER_CODE`                |
| `IsActive`    | `BIT`             | Yes      | `1` = active (default), `0` = disabled           |
| `CreatedDate` | `DATETIME2(7)`    | Yes      | Row creation timestamp (default `GETDATE()`)     |
| `LastLogin`   | `DATETIME2(7)`    | Yes      | Updated on each successful authentication        |

---

## cURL Example

```bash
curl -X POST \
  'https://{host}:{port}/rms-service/api/broker/create' \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "broker01",
    "password": "secureP@ssword",
    "memberCode": "26",
    "isActive": true
  }'
```

---

## Notes

- **No authentication required** — This endpoint does not carry the `@Broker` annotation, so the `BrokerTokenValidator` filter does not intercept it.
- **Password policy** — The API does not currently enforce password complexity rules. Clients should ensure strong passwords are used.
- **Member code** — Should correspond to a valid `MEMBER_CODE` in the `RMS_MEMBERS` table. Without it, downstream margin and trade APIs will not be able to resolve `memberId` from the token and will reject requests.
