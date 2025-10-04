---

# **1 — Domains & System Layout**

| Frontend    | Domain                           | Description                                                                    |
| ----------- | -------------------------------- | ------------------------------------------------------------------------------ |
| Admin SPA   | `admin.ecommercecontrol.local`   | Central admin UI. Manages tenants.                                             |
| Tenant SPA  | `app.ecommercecontrol.local`     | All tenants share this domain. Domain auto-identifies tenant after login.      |
| Backend API | `backend.ecommercecontrol.local` | Single API handling all auth, tenant & admin requests, and third-party access. |

**DB layout:**

* **Central DB**:

  * `users` → admin users
  * `tenants` → Stancl tenants
  * `tenant_links_to` → maps tenant emails to tenant_id
* **Tenant DBs**: each tenant has a full `users` table with Fortify fields.

---

# **2 — Context Detection & Domain Identification**

* Detect context **purely via PHP backend** using request host:

  ```php
  $host = $request->getHost();
  if ($host === env('ADMIN_DOMAIN')) $context='admin';
  elseif ($host === env('TENANT_DOMAIN')) $context='tenant';
  else return 400;
  ```
* **Store `context` in request attributes** using values from `.env` (`ADMIN_DOMAIN`, `TENANT_DOMAIN`) for added security; prevents spoofing.
* No path-based tenant identification required. The tenant is auto-resolved **after login** using `tenant_links_to` (email → tenant_id) when the user attempts login.

---

# **3 — Auth Flow Overview**

### **Admin Flow**

1. SPA sends login request to backend.
2. Backend detects context (`admin`) based on host.
3. Fortify authenticates against central `users` table.
4. Optional 2FA: enabled per-user. If enabled, return 2FA challenge before token issuance.
5. On success: issue Sanctum token for SPA or external API clients (optional).
6. Admin continues using SPA or API with token/session.

### **Tenant Flow**

1. SPA or Chrome extension sends login request to backend.
2. Backend detects context (`tenant`) based on host.
3. Lookup `tenant_links_to` in central DB for email:

   * If found → get `tenant_id` → initialize tenancy.
   * If not found → generic auth error (do not leak info).
4. Initialize tenancy (`tenancy()->initialize($tenant)`), now all tenant DB operations are scoped.
5. Fortify authenticates against tenant `users` table.
6. Optional 2FA:

   * If user enabled 2FA → challenge returned first.
   * If not enabled → issue Sanctum token immediately.
7. SPA or Chrome extension stores token for API calls.

**Key:** Tenant identification happens **after login attempt**, no path needed. First successful auth maps tenant automatically via `tenant_links_to`.

---

# **4 — Token Strategy for SPA & Third-party**

* **Sanctum token issuance**:

  * Admin → central DB, optional SPA or API usage.
  * Tenant → tenant DB, SPA, Chrome extension, or other external tools.
* Tokens **include metadata**:

  * `tenant_id` (if tenant)
  * `abilities` for role or plan-based access control
* SPA uses cookie-based session or token, Chrome extension uses Bearer token.
* 2FA enforced before token creation if enabled for the user.

**Flow**:

```
Login → Credentials verified → (If 2FA enabled: 2FA challenge) → Token issued → API calls with token → Authenticated
```

---

# **5 — Fortify + Tenancy Integration**

* Use `Fortify::authenticateUsing()` closure:

  1. Detect context (admin/tenant via host).
  2. Admin → central DB → standard Fortify authentication.
  3. Tenant → resolve tenant via `tenant_links_to` → initialize tenancy → authenticate tenant DB `users`.
  4. 2FA applied **after credential check** if enabled for that account.
* Wrap login in **transaction** for tenant DB to prevent race conditions.
* Use same Fortify flows for:

  * Registration (admin creates tenants and their users)
  * Forgot password
  * Reset password
  * Email verification
  * Two-factor authentication

---

# **6 — Registration & Mapping**

* Admin creates tenant user:

  1. Central DB: create tenant via Stancl.
  2. Tenant DB: seed users.
  3. Update `tenant_links_to`:

     ```php
     DB::connection('central')->table('tenant_links_to')
       ->updateOrInsert(['tenant_id'=>$tenant->id,'email'=>$user->email],['updated_at'=>now()]);
     ```
* Tenant can create additional users (if allowed):

  * On `Registered` event, write mapping to central `tenant_links_to`.

---

# **7 — Password Reset & 2FA**

* **Admin password reset:** Fortify + central DB.
* **Tenant password reset:** detect tenant via `tenant_links_to`, initialize tenancy, run Fortify tenant reset flow.
* **2FA:** optional for admins, enabled per-user. Tenant users can also enable 2FA. Enforcement happens before token issuance.

---

# **8 — Middleware & Routing**

* Global middleware:

  * `DetectAppContext` → sets context (`admin`/`tenant`) from request host.
* Route groups:

  ```php
  Route::middleware(['api','DetectAppContext'])->group(function(){
      Route::post('/auth/login',[AuthController::class,'login']);
      Route::post('/auth/register',[AuthController::class,'register']);
      Route::post('/auth/forgot-password',[AuthController::class,'forgotPassword']);
      // Tenant SPA APIs
      Route::middleware(['auth:sanctum'])->group(...);
      // Admin APIs
      Route::middleware(['auth:admin'])->group(...);
  });
  ```

---

# **9 — Security & Production Hardening**

1. **Environment separation**:

   * `.env.local` → local development.
   * `.env.production` → production config. Laravel handles via `APP_ENV`.
2. **Rate-limiting**:

   * Apply throttle middleware per IP/email.
3. **Tokens**:

   * Issue restricted abilities, revoke on password change.
4. **2FA enforcement**:

   * Optional per user (admin) / per plan (tenant).
5. **Audit logging**:

   * Log all login attempts, 2FA attempts, token creation.
6. **No account enumeration**:

   * Always generic messages on failed login or reset.
7. **Tenancy isolation**:

   * Each tenant DB separate; token cannot cross tenants.

---

# **10 — SPA & Chrome Extension Integration**

* SPA:

  * Stores Sanctum token in memory or secure cookie (SPA config).
  * Makes API calls to backend with token.
* Chrome Extension:

  * Sends Bearer token to backend.
  * Backend resolves tenant using token → tenant DB → perform requested operations.
* All tenant API calls validated against tenant DB; central DB never touched for tenant data.

---

# **11 — Scalability & Future Proof**

* Central DB handles all tenant metadata and admin users.
* Tenants auto-initialized only on login or API access.
* Additional frontends (mobile apps, external tools) integrate via same API endpoints using tokens.
* Adding new tenants requires no code change; only provisioning via admin UI.
* Ability to enforce different plans, abilities, or 2FA policies per tenant.

---

# **12 — Final Flow Diagram**

**Login Flow (Tenant)**:

```
SPA/Extension → backend → Detect context(host) → lookup tenant_links_to → initialize tenancy → Fortify credentials check → 2FA (if enabled) → issue Sanctum token → SPA/extension stores token → API calls
```

**Login Flow (Admin)**:

```
SPA → backend → Detect context(host) → Fortify central users → optional 2FA → issue Sanctum token → SPA stores token → Admin API calls
```

**Token usage for APIs**:

* SPA → token in header (Bearer or cookie)
* Chrome extension → token in header (Bearer)
* Backend always validates token against appropriate DB (central vs tenant).

---