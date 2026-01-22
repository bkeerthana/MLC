# Credential Leakage Demo Dataset (SQLite `.db`)

This dataset is a **synthetic** (classroom-safe) SQLite database designed to teach:
- How to list tables in a database
- How to load tables into Pandas DataFrames
- How to inspect DataFrames (shape, dtypes, missing values, describe)
- How to access/filter records (`loc`, `iloc`, boolean filters, joins)
- How to reason about credential leakage and related risks (sessions, API keys, reset tokens, audit logs)


# Tables included (overview)

The database models a typical web application authentication/authorization stack:

| Table | What it represents |
|------|---------------------|
| `users` | User identities + auth configuration (role, MFA) |
| `sessions` | Session records created after login |
| `api_keys` | API keys (secrets) and scopes (permissions) |
| `password_resets` | Password reset requests and reset tokens |
| `audit_log` | Security/activity events for monitoring & forensics |

---

## Table schemas (columns and meanings)

### A) `users` — user accounts
Stores account identity and authentication-related attributes.

| Column | Meaning | Typical target dtype (Pandas) |
|---|---|---|
| `user_id` | Unique user identifier | `int64` |
| `username` | Login name | `string`/`object` |
| `email` | Email address | `string`/`object` |
| `password_hash` | Password hash (not plaintext) | `string`/`object` |
| `password_salt` | Salt used for hashing | `string`/`object` |
| `role` | Role (student/staff/admin/auditor) | `category` |
| `mfa_enabled` | MFA enabled flag (`"1"`/`"0"`) | `bool` |
| `created_utc` | Account creation timestamp (UTC) | `datetime64[ns, UTC]` |
| `last_login_utc` | Most recent successful login (UTC); may be null | `datetime64[ns, UTC]` |

Security notes:
- Use `role` + `mfa_enabled` to identify high-risk privileged accounts.
- `created_utc` vs `last_login_utc` supports dormant-account analysis.

---

### `sessions` — login sessions
Represents session tokens issued after authentication.

| Column | Meaning | Typical target dtype (Pandas) |
|---|---|---|
| `session_id` | Unique session token/identifier | `string`/`object` |
| `user_id` | Owner user id (FK to `users.user_id`) | `int64` |
| `created_utc` | Session creation time (UTC) | `datetime64[ns, UTC]` |
| `expires_utc` | Session expiry time (UTC) | `datetime64[ns, UTC]` |
| `ip_address` | Source IP for the session | `string`/`object` |
| `user_agent` | Client identifier (browser/curl/Postman) | `category` or `string` |
| `is_active` | Active flag (`"1"`/`"0"`) | `bool` |

Security notes:
- Active sessions for privileged users are higher risk in a leak.
- `ip_address` and `user_agent` help identify anomalies.

---

###  `api_keys` — API authentication secrets
Represents long-lived secrets used to access APIs.

| Column | Meaning | Typical target dtype (Pandas) |
|---|---|---|
| `key_id` | Unique API key record id | `int64` |
| `user_id` | Owner user id (FK to `users.user_id`) | `int64` |
| `api_key` | API key secret/token | `string`/`object` |
| `scope` | Permissions for the key | `category` |
| `created_utc` | Key creation time (UTC) | `datetime64[ns, UTC]` |
| `last_used_utc` | Last usage time (UTC); may be null | `datetime64[ns, UTC]` |
| `is_revoked` | Revocation flag (`"1"`/`"0"`) | `bool` |

Security notes:
- Keys with broad scopes (e.g., admin) raise risk.
- Keys not revoked after long inactivity are a common control gap.

---

###  `password_resets` — password reset workflow
Tracks reset requests and tokens.

| Column | Meaning | Typical target dtype (Pandas) |
|---|---|---|
| `reset_id` | Unique reset request id | `int64` |
| `user_id` | User requesting reset (FK to `users.user_id`) | `int64` |
| `reset_token` | Reset token secret | `string`/`object` |
| `requested_utc` | When reset was requested (UTC) | `datetime64[ns, UTC]` |
| `used_utc` | When token was used; null if unused | `datetime64[ns, UTC]` |
| `ip_address` | IP address that requested the reset | `string`/`object` |

Security notes:
- Unused reset tokens can indicate attempted or incomplete ATO flows.
- Multiple requests per account can indicate brute-force or abuse.

---

### `audit_log` — activity & security logging
Captures events useful for monitoring and forensics.

| Column | Meaning | Typical target dtype (Pandas) |
|---|---|---|
| `event_id` | Unique audit event id | `int64` |
| `event_utc` | When the event occurred (UTC) | `datetime64[ns, UTC]` |
| `actor_user_id` | User performing action; may be null | `int64` (nullable) |
| `action` | Event type (LOGIN, LOGIN_FAIL, etc.) | `category` |
| `resource` | Target resource/endpoint | `category` or `string` |
| `outcome` | Result (SUCCESS/FAIL/BLOCKED) | `category` |
| `ip_address` | IP associated with event | `string`/`object` |
| `details` | Optional extra info; may include synthetic debug fragments | `string`/`object` |

Security notes:
- Use this table for timeline analysis (resampling by minute/day).
- Search `details` for leakage-like strings (e.g., token fragments).

---

##  Loading tables into DataFrames

```python
import sqlite3
import pandas as pd

conn = sqlite3.connect("credential_leak_demo_object.db")

df_users = pd.read_sql_query("SELECT * FROM users;", conn)
df_sessions = pd.read_sql_query("SELECT * FROM sessions;", conn)
df_api_keys = pd.read_sql_query("SELECT * FROM api_keys;", conn)
df_resets = pd.read_sql_query("SELECT * FROM password_resets;", conn)
df_audit = pd.read_sql_query("SELECT * FROM audit_log;", conn)

conn.close()
```

---

## 5) Sample solutions : datatype conversion exercise (example for `users`)


df_users["user_id"] = df_users["user_id"].astype("int64")
df_users["role"] = df_users["role"].astype("category")

df_users["mfa_enabled"] = df_users["mfa_enabled"].map({"1": True, "0": False})

df_users["created_utc"] = pd.to_datetime(df_users["created_utc"], utc=True)
df_users["last_login_utc"] = pd.to_datetime(df_users["last_login_utc"], utc=True, errors="coerce")

df_users.dtypes

