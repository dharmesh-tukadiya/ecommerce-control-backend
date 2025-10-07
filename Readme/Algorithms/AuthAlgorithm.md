# **Tech Stack**

* PHP 8.4
* Laravel 12.5
* Laravel Stacks: Sanctum (API authentication), Fortify (login/registration/2FA/password reset)
* Stancl Tenancy for multi-tenant handling

---

# **1 — Domains & System Layout**

| Frontend    | Domain                           | Description                                                          |
| ----------- | -------------------------------- | -------------------------------------------------------------------- |
| Admin SPA   | `admin.ecommercecontrol.local`   | Central admin UI; manages tenants.                                   |
| Tenant SPA  | `app.ecommercecontrol.local`     | Shared domain for tenants; tenant auto-identified after login.       |
| Backend API | `backend.ecommercecontrol.local` | Single API for auth, tenant/admin requests, third-party integration. |

**Database Layout:**

* **Central DB:**

  * `users` → admin users
  * `tenants` → Stancl tenants
  * `tenant_links_to` → maps tenant emails → tenant_id

* **Tenant DBs:** each tenant has a `users` table with Fortify fields

**Rules:**

* Context is detected from request host.
* Tenant DB connections initialized on login/API request; terminated after request.
* No path-based tenant identification; mapping via `tenant_links_to`.

---

# **2 — Context Detection & Domain Identification**

* Detect context (`admin` or `tenant`) using request host.
* Store `context` as a request attribute for internal usage.
* Tenant initialization occurs **after login** based on `tenant_links_to`.
* Middleware required:

  * `DetectAppContext` → sets context in request
  * `TerminateTenancy` → closes tenant DB connection post-request

**Default Tech Stack Use:**

* Only Fortify + Stancl Tenancy methods; no custom tenant status checks are needed yet.

---

# **3 — Auth Flow Overview**

### **Admin Flow**

1. SPA sends login request.
2. Backend detects `admin` context.
3. Fortify authenticates against central `users`.
4. Optional 2FA (Fortify-provided).
5. Sanctum issues token (SPA or API).
6. Admin uses token for subsequent requests.

### **Tenant Flow**

1. SPA or Chrome extension sends login request.
2. Backend detects `tenant` context.
3. Lookup `tenant_links_to` for email; get tenant ID.
4. Initialize tenancy (`tenancy()->initialize($tenant)`).
5. Fortify authenticates against tenant DB `users`.
6. Optional 2FA challenge (Fortify).
7. Sanctum issues token for SPA session or extension.

**Default Tech Stack Use:**

* Fortify handles login, registration, password reset, and 2FA.
* Sanctum handles token creation and cookie/session management.
* Chrome extension uses Bearer token; SPA can use cookies.

---

# **4 — Token Strategy**

* SPA: cookie-based Sanctum session (default).
* Chrome extension: Sanctum token via API request.
* No custom token rotation or refresh required; Fortify + Sanctum handle session expiration.

**Rules:**

* Tokens include `abilities` metadata if needed for roles/permissions.
* Token revocation occurs automatically on password change via Sanctum.

---

# **5 — Fortify + Tenancy Integration**

* Use `Fortify::authenticateUsing()` closure:

  * Detect context.
  * Admin → central DB auth.
  * Tenant → initialize tenancy → tenant DB auth.
* 2FA handled via Fortify standard flow.
* Transactions and DB isolation handled by Stancl tenancy package.

---

# **6 — Registration & Mapping**

* Admin creates tenant: central DB adds tenant; tenant DB seeded.
* `tenant_links_to` updated for new tenant users.
* Tenant users can register using Fortify registration flow.

**Default Tech Stack Use:**

* No extra validation; Fortify handles user creation, email verification, and password hashing.

---

# **7 — Password Reset & 2FA**

* Fortify default reset flow used for admin and tenant users.
* 2FA optional; enforced per Fortify configuration.
* Password policies follow Fortify defaults (can be customized in future).

---

# **8 — Middleware & Routing**

* `DetectAppContext` middleware sets context.
* `TerminateTenancy` middleware closes tenant DB connection.
* Route groups for admin vs tenant APIs.
* Sanctum middleware (`auth:sanctum`) protects all routes.

**Default Tech Stack Use:**

* Fortify + Sanctum handle authentication, password reset, and 2FA.
* Stancl Tenancy ensures tenant DB scoping.

---

# **9 — Security & Production Hardening**

* HTTPS enforced in `.env` and server config.
* Fortify + Sanctum provide CSRF, session management, and 2FA security.
* Rate limiting via Laravel throttle middleware.
* Audit logging optional; can use default Laravel logging.

---

# **10 — SPA & Chrome Extension Integration**

* SPA: uses cookie-based sessions; token automatically included in requests.
* Chrome extension: uses Sanctum Bearer token.
* Tenant API requests scoped via Stancl tenancy.

---

# **11 — Scalability & Future Proof**

* Central DB holds tenants/admin metadata; tenants initialized on demand.
* Adding new tenants requires no code change; only admin UI provisioning.
* Roles/permissions handled via Sanctum token abilities.
* Future upgrades: Laravel auth packages, Fortify policies, password rules, etc.

---

# **12 — Error Handling**

* Standard Laravel/Fortify error responses used.
* Generic messages prevent account enumeration.
* Tenant lookup failure handled gracefully with Laravel exceptions.

---

# **13 — Production Checklist**

* HTTPS enabled.
* Sanctum cookies configured for stateful domains.
* Database connections secure; backups for central DB (`tenant_links_to`).
* Rate limiting on all auth endpoints.
* Tokens expire per Sanctum default.

---

# **14 — Flow Diagrams (Textual)**

**Tenant Login:**

```
SPA/Extension → Backend → Detect context → Lookup tenant_links_to → Initialize tenancy → Fortify login → 2FA (optional) → Sanctum token → API calls
```

**Admin Login:**

```
SPA → Backend → Detect context → Fortify login → 2FA (optional) → Sanctum token → Admin API calls
```