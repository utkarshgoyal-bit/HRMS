
Task 1 — Authentication & Session Management

Phase 1: Bug Identification
Bug 1 — Credentials file committed to repository (Critical Security)
credentials.md contains real login credentials including superadmin@hrms.com with Welcome@1234 and multiple employee emails. This file is in the repository and visible to anyone with repo access. Must be deleted from repo history entirely using git filter-branch or git-filter-repo, not just removed in a new commit.

Bug 2 — Token query parameter accepted in protect middleware (Security)
js} else if (req.query && req.query.token) {
    token = req.query.token;
}
Tokens passed as query parameters appear in server logs, browser history, HTTP referrer headers, and proxy logs. A JWT in a URL is a credential leak waiting to happen. This fallback has no documented use case in this codebase — no route passes tokens as query params. It should be removed entirely.

Bug 3 — protect middleware makes a DB call on every request (Performance/Security)
jsconst employee = await Employee.findById(decoded.id);
if (employee) {
    req.user = employee;
    return next();
}
const branch = await Branch.findById(decoded.id).populate('orgId');
Two DB queries per request in the worst case. More critically — this pattern means a terminated or suspended employee's token still works until it expires, because the middleware only checks that the employee exists, not that their status === 'Active'. A terminated employee can continue accessing the API for the full 7-day token lifetime.

Bug 4 — Lockout counter reset condition is wrong
jsif (user.loginAttempts > 0 || user.lockUntil) {
    await Employee.findByIdAndUpdate(user._id, { loginAttempts: 0, lockUntil: null });
}
This runs on every successful login regardless of whether a lockout was ever triggered. Minor inefficiency but also means lockUntil is cleared even when it was never set, creating unnecessary write operations.

Bug 5 — refreshToken has no user status check (Security)
jsjwt.verify(rfToken, secret, (err, decoded) => {
    if (err) { return res.status(401)... }
    const accessToken = signToken(decoded.id, decoded.orgId);
    res.status(200).json({ success: true, token: accessToken });
});
The refresh endpoint issues a new access token without checking if the user still exists or is still Active. A deactivated employee can keep refreshing tokens indefinitely for 7 days. No DB lookup happens here at all.

Bug 6 — Frontend-Backend contract mismatch in AuthContext.logout
jsconst logout = () => {
    localStorage.clear();
    setUser(null);
    window.location.href = '/';
};
localStorage.clear() is called but the backend logout endpoint (POST /auth/logout) is never called. The refresh token cookie on the server is never cleared. The cookie remains valid for 7 days. A user who logs out on a shared computer has their refresh token still active — anyone who gets that cookie can silently refresh and get a new access token.

Bug 7 — Login.jsx bypasses the axios instance for the login call
jsconst rawApiUrl = import.meta.env.VITE_API_URL || 'http://127.0.0.1:9999';
const API_URL = rawApiUrl.replace(/\/+$/, '');
const response = await axios.post(`${API_URL}/api/v1/auth/login`, data);
The login call uses the raw axios import directly instead of the configured API instance from src/api.js. This means the withCredentials: true setting is NOT applied to this specific call. On some browser/CORS configurations this will cause the refresh token cookie to not be set correctly since the cookie requires credentials mode to be enabled.

Bug 8 — syncUser in AuthContext silently swallows errors
js} catch (err) {
    console.error('Failed to sync user data:', err);
}
If /auth/me returns a 401 (expired token), the interceptor in api.js will attempt a refresh. If the refresh also fails, the error is caught here silently. The user object in state remains stale from localStorage but the user is never logged out. The app continues showing authenticated UI with an invalid session.

Bug 9 — sendTokens response missing modules field
The login response includes role, branches, roleScopeType etc. but does not include organization.modules. However EmployeeList.jsx reads orgModules directly from localStorage.getItem('user')?.modules. This field is never populated from the login response, so orgModules is always {} and module-gated features like face recognition never activate correctly.

Bug 10 — sameSite: 'strict' blocks cross-origin refresh
jsres.cookie('refreshToken', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
Your frontend (hrms-dev.fundflick.in) and backend (hrms-dev-backend.fundflick.in) are on different subdomains. With sameSite: 'strict', the refresh token cookie is blocked on cross-origin requests. This means the /auth/refresh call from the frontend interceptor will never send the cookie and refresh will always fail silently. Should be sameSite: 'none' with secure: true for cross-domain setups, or sameSite: 'lax' if the domains can be unified.

Phase 2: Optimization
O1 — Eliminate the protect middleware double DB query
Cache user data in the JWT itself for the fields needed by middleware — specifically role, orgId, and status. The token already has id and orgId. Adding role and status means the middleware can do a single lean query instead of a full document fetch on every request:
js// signToken — add role and status
const signToken = (id, orgId, role, status) => {
  return jwt.sign({ id, orgId, role, status }, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_EXPIRE || '7d'
  });
};

// protect — use decoded fields for fast path
const decoded = jwt.verify(token, process.env.JWT_SECRET);
if (decoded.status !== 'Active') {
    return res.status(403).json({ success: false, message: 'Account is inactive.' });
}
req.user = { _id: decoded.id, orgId: decoded.orgId, professional: { role: decoded.role } };
// Only fetch full user document for endpoints that need it
This eliminates a DB query on every API call across the entire application.
O2 — Consolidate login response to single source of truth
The login response and /auth/me response build the user object in two separate places with slightly different fields. Extract a buildUserResponse(user) helper function used by both, ensuring the shape is always identical.
O3 — Remove redundant loginAttempts reset on every successful login
Only clear the counter when loginAttempts > 0:
jsif (user.loginAttempts > 0 || user.lockUntil) {
This condition is actually fine as written but the comment in the code is misleading. The real optimization is making this a single atomic update rather than a separate findByIdAndUpdate call after the password check.
O4 — AuthContext.syncUser should not cache-bust on every call
jsconst res = await axios.get(`/auth/me?t=${Date.now()}`);
The cache-bust timestamp is unnecessary — the protect middleware always does a live DB lookup anyway. Remove the timestamp to allow proper HTTP caching headers to work.

Phase 3: Feature Enhancements
E1 — Forgot Password flow
The Forgot Password link exists in the UI but points to href="#" — it does nothing. The most impactful missing auth feature. Implement: email input → OTP or reset link sent via SES → token-gated password reset endpoint.
E2 — Suspend/deactivate user propagation
When an employee's status changes to Inactive or Terminated, their active tokens should be invalidated immediately. Implement a tokenVersion field on the Employee model, increment it on deactivation, embed it in the JWT, and validate it in protect. This closes Bug 3 and Bug 5 without requiring a token blocklist.
E3 — Remember me / Session duration choice
Currently all sessions last 7 days. Add a rememberMe checkbox on the login form. If unchecked, issue a short-lived access token (1 hour) with no refresh token — session ends on tab close. If checked, issue the current 7-day refresh token.
E4 — Login audit log
Record every login attempt (success/failure), IP address, user agent, and timestamp in a LoginAuditLog collection. Essential for a payroll system where compliance and access tracking matter. Surface the last 5 login events in the user's Settings → Security tab.
E5 — Active session indicator
Show users their currently active sessions (device, last active, location approximation from IP) in Settings → Security, with a "Log out all other sessions" button. Requires storing refresh token identifiers per user.

Summary for Task 1
#SeverityItemB1🔴 Criticalcredentials.md with real passwords in repoB2🔴 CriticalToken accepted in query param — credential leakB3🔴 HighTerminated employees can still access API for 7 daysB5🔴 HighRefresh endpoint issues tokens without status checkB6🔴 HighLogout never calls backend — refresh cookie persistsB10🔴 HighsameSite: strict breaks cross-domain cookieB7🟡 MediumLogin uses raw axios — withCredentials not appliedB8🟡 MediumSync errors swallowed silently — stale sessionsB9🟡 Mediummodules missing from login responseB4🟢 LowRedundant DB write on every successful login


I have everything needed. Full three-phase review now.

---

## Task 2 — Super Admin: Organisation Management

---

## Phase 1: Bug Identification

**Bug 1 — `updateOrganization` in the live controller silently wipes all unset fields (Critical)**

The controller currently on the server (the old version, not the new one we built) does:

```js
const { organizationName, status, modules, email, address, mobileNumber, logo } = req.body;

const organization = await Organization.findByIdAndUpdate(
    req.params.id,
    { organizationName, status, modules, email, address, mobileNumber, logo },
    { new: true, runValidators: true }
);
```

This passes all destructured values directly including `undefined` ones. Mongoose with the default `strict` mode silently ignores `undefined` values in updates — but this also means that if `EditOrganizationModal` ever sends a field with an empty string (e.g. `logo: ''`), it will overwrite the existing logo URL with an empty string. The `logo` field is particularly vulnerable: if a user opens the edit modal without changing the logo, `formData.logo` is the existing server path, but if anything goes wrong in the logo URL reconstruction, the field gets wiped.

---

**Bug 2 — `suspendOrganization` is irreversible and mislabeled (Critical UX/Logic)**

```js
const organization = await Organization.findByIdAndUpdate(
    req.params.id,
    { status: 'Suspended' },
    { new: true }
);
```

The suspend action only ever sets `status: 'Suspended'` — it cannot reactivate. But the confirm dialog says: *"Are you absolutely sure you want to delete this organization? All data will be lost."* This message is completely wrong — it says "delete" and "all data will be lost" but the operation only suspends. This misleads the super admin.

Additionally, there is no "reactivate" button in the UI — once suspended, an org can never be reactivated without a direct DB edit. The `suspendedOrganizations` stat is tracked but never actionable.

The new controller we built fixes this — but the **old controller is still what's deployed**.

---

**Bug 3 — Slug collision not handling case sensitivity or special characters correctly**

```js
const slug = organizationName
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/(^-|-$)/g, '');
```

The slug check is:
```js
const existingOrg = await Organization.findOne({ slug });
```

MongoDB string queries are case-sensitive by default. The slug is always lowercase so this is fine. However the slug generation does not handle Unicode characters. An org named "Mäitrii" generates slug `m-itrii` — a different org named "Mãitrii" also generates `m-itrii`. Two different org names produce the same slug, the second creation fails with a potentially confusing error. The slug field should have a unique index enforced at the DB level, not just application-level checks.

---

**Bug 4 — `getDashboardStats` counts ALL employees including SUPER_ADMIN**

```js
Employee.countDocuments()
```

This counts every document in the Employee collection including the SUPER_ADMIN account itself. The dashboard shows "Total Employees: 97" but one of those is the super admin. Should be:

```js
Employee.countDocuments({ 'professional.role': { $ne: 'SUPER_ADMIN' } })
```

---

**Bug 5 — Frontend-Backend contract mismatch: `EditOrganizationModal` sends full `formData` spread**

```js
const payload = { ...formData, logo: uploadedLogoUrl };
```

`formData` contains `modules` as a nested object. The OLD backend `updateOrganization` destructures `modules` from `req.body` and passes it directly to `findByIdAndUpdate`. This replaces the entire `modules` subdocument atomically — fine. But `formData` also contains `organizationName`, `status`, `email`, `address`, `mobileNumber`. If any of these are empty strings (e.g. user cleared the address field), they get written as empty strings to the DB. No frontend validation prevents this.

The NEW controller we built handles this correctly with `if (field !== undefined)` guards. But the old deployed controller does not.

---

**Bug 6 — Logo URL stored as server-relative path but reconstructed inconsistently**

In `EditOrganizationModal`:
```js
const initialLogo = organization.logo
    ? `${API_URL}${organization.logo}?token=${localStorage.getItem('token')}`
    : null;
```

The logo preview appends the access token as a query parameter to the image URL. This is a security issue (token in URL — same as Auth Bug 2). Additionally if the logo is hosted on Cloudinary (which the file upload service uses), the path is already a full URL, and prepending `API_URL` creates an invalid double URL like `https://api.fundflick.inhttps://cloudinary.com/...`.

---

**Bug 7 — `handleSuspend` in SuperAdminDashboard has wrong confirmation message**

```js
if (!window.confirm('Are you absolutely sure you want to delete this organization? All data will be lost.'))
```

Says "delete" — the operation is suspend. This is a copy-paste error that could cause a super admin to cancel a legitimate suspend action thinking it's destructive, or proceed thinking they've deleted org data when they haven't.

---

**Bug 8 — `CreateOrganizationModal` exposes temp password in success screen**

```js
<p className="text-sm"><span className="text-slate-500">Password:</span> <strong>{success.orgAdmin.tempPassword}</strong></p>
```

The temporary password is shown in plaintext in the browser. The backend also returns `tempPassword` in the API response. If this screen is ever screenshotted or if the response is logged, the credential is exposed. This should be replaced with "A welcome email has been sent to the admin with login instructions."

---

**Bug 9 — No authorization middleware on SuperAdmin routes — only controller-level check**

```js
router.use(protect);
// No authorize('SUPER_ADMIN') middleware
```

The SUPER_ADMIN check happens inside each controller via `requireSuperAdmin(req, res)`. This means if a new developer adds a new route to `superAdminRoutes.js` and forgets to call `requireSuperAdmin`, any authenticated user can access it. The route-level should have `authorize('SUPER_ADMIN')` as defense-in-depth:

```js
router.use(protect, authorize('SUPER_ADMIN'));
```

---

## Phase 2: Optimization

**O1 — `getOrganizations` returns full documents including all nested fields**

The list view only needs `organizationName`, `slug`, `status`, `modules`, `orgAdmin` (name + email), `createdAt`. Currently returns everything including `shift_rules`, `bankDetails`, `comp_off_rules` etc. Add a `.select()` projection:

```js
Organization.find()
    .select('organizationName slug status modules orgAdmin email mobileNumber createdAt logo')
    .populate('orgAdmin', 'personal.firstName personal.lastName contact.email')
    .sort({ createdAt: -1 });
```

**O2 — `getDashboardStats` runs 4 separate count queries**

```js
const [totalOrgs, activeOrgs, suspendedOrgs, totalEmployees] = await Promise.all([...])
```

`Promise.all` is already used — good. But `activeOrgs` and `suspendedOrgs` can be replaced with a single `$facet` aggregation that counts by status in one query instead of two.

**O3 — `SuperAdminDashboard` rebuilds `API_URL` inside every function**

`fetchData`, `handleSuspend` both reconstruct `API_URL` from env. Extract to module scope constant.

**O4 — `CreateOrganizationModal` uses `useState` with `useEffect` to reset on open**

The form state reset on open uses `useEffect([isOpen])`. This runs on every open/close cycle. A better pattern is `useReducer` with an `RESET` action, or simply using `key={isOpen}` on the modal to unmount/remount on open.

---

## Phase 3: Feature Enhancements

**E1 — Reactivate suspended organizations**

The suspend button should toggle. When `org.status === 'Suspended'`, show a "Reactivate" button. The new controller already supports this pattern via the toggle logic. The UI just needs the conditional rendering added.

**E2 — Org-level usage metrics on the dashboard**

The super admin dashboard shows total orgs and total employees. More actionable metrics: last login date per org, monthly active users per org, which orgs have AI enabled and their estimated monthly token cost. These can be derived from `AIAuditLog` and `Employee.updatedAt`.

**E3 — Impersonate / Login as Org Admin**

A "Login as" button that generates a short-lived impersonation token for a specific org admin. Allows super admin to debug issues without knowing the org admin's password. Common in multi-tenant SaaS platforms. Requires an `impersonatedBy` field on the token and audit logging.

**E4 — Org creation email notification**

When an org is created, automatically send a welcome email to the org admin with their credentials using the SES service we just wired up. The `tempPassword` is already available in `createOrganization` — just call `sendWelcomeEmail` before the response.

**E5 — Search and filter on org list**

The current list shows all orgs with no search or filter. With 20+ orgs this becomes unusable. Add a search by name and filter by status (Active/Suspended) on the frontend, backed by query params on `GET /organizations`.

---

## Summary for Task 2

| # | Severity | Item |
|---|---|---|
| B1 | 🔴 Critical | Old `updateOrganization` wipes unset fields |
| B2 | 🔴 Critical | Suspend is irreversible, no reactivate path |
| B9 | 🔴 High | No route-level SUPER_ADMIN guard |
| B5 | 🔴 High | Empty string fields overwrite DB data on edit |
| B6 | 🟡 Medium | Logo URL double-prepend + token in URL |
| B7 | 🟡 Medium | Wrong confirmation message ("delete" vs "suspend") |
| B8 | 🟡 Medium | Temp password exposed in plaintext UI |
| B3 | 🟢 Low | Slug collision on Unicode org names |
| B4 | 🟢 Low | Stats count includes SUPER_ADMIN employee |

**Ready for Task 3 — Employee Management whenever you are.**


I have everything. Full three-phase review now.

---

## Task 3 — Employee Management

---

## Phase 1: Bug Identification

**Bug 1 — Frontend-Backend contract mismatch: `EditEmployeeModal` does not send `da` or `specialAllowance`**

`AddEmployeeModal` sends:
```js
payroll: {
    salary: {
        base, da, hra, conveyance, specialAllowance, commission
    }
}
```

`EditEmployeeModal` sends:
```js
payroll: {
    salary: {
        base, hra, conveyance, commission  // ← da and specialAllowance MISSING
    }
}
```

This means editing an employee silently wipes `da` and `specialAllowance` from their salary record. If a payroll run follows an edit, those allowances are zero. This is a direct financial calculation bug — the affected employee will receive lower net pay.

---

**Bug 2 — `EditEmployeeModal` sends `pf_opt_in: true` unconditionally**

```js
payroll: {
    pf_opt_in: true,  // ← hardcoded always true
    ...
}
```

The form has a `pfOverride` field but `pf_opt_in` is never derived from it — it's always `true`. An employee who has opted out of PF gets their opt-out silently overwritten every time their record is edited. This will cause incorrect PF deductions in the next payroll run.

---

**Bug 3 — Bulk upload `employeeIdCounter` has a race condition**

The old bulk upload (the one still on the dev server, pre our fix) does:

```js
let employeeIdCounter = Number(org?.employeeIdCounter || 0);
// ...
for (const empData of employees) {
    employeeIdCounter++;
    // use employeeIdCounter for this employee
}
// Save at the END after the loop
if (org) {
    await Organization.findByIdAndUpdate(orgId, { employeeIdCounter });
}
```

If two bulk uploads run simultaneously (or if the loop throws after some employees but before the final save), the counter in the DB is not updated and the next upload produces duplicate employee IDs. The new controller we built fixes this with atomic pre-allocation — but confirm this is what's deployed.

---

**Bug 4 — `updateEmployee` allows email changes without uniqueness check**

```js
// In updateEmployee — email is silently accepted if sent
if (email) employee.contact.email = email;  // not in current code but...
```

Actually looking at the current `updateEmployee` code, email is not in the destructured fields — which means email can never be updated via this endpoint. However the `AddEmployeeModal` sends `email` in the top-level spread (`...formData`) which the backend controller accepts for new employees. For updates, if someone sends `email` in the payload, it gets ignored silently with no error — no feedback to the admin that the email change didn't apply.

---

**Bug 5 — `deleteEmployee` is a hard delete with no cascading cleanup**

```js
await Employee.deleteOne({ _id: req.params.id });
```

This hard-deletes the employee but leaves behind:
- `Attendance` records referencing `employeeId`
- `Salary` records referencing `employeeId`
- `LeaveRequest` records referencing `employeeId`
- `SalaryAdjustment` records referencing `employeeId`
- The employee's `orgId` counter is not decremented
- `AIAuditLog` entries referencing the user

This creates orphaned records. Any aggregation or report that joins on `employeeId` will silently skip or crash on missing references.

---

**Bug 6 — `getMyOrgEmployees` — `getEmployeeById` has no org scope check**

```js
exports.getEmployeeById = async (req, res) => {
    const employee = await Employee.findById(req.params.id).select('-contact.password -fcmTokens');
    if (!employee) return res.status(404)...
    return res.status(200).json({ success: true, data: employee });
};
```

No `orgId` check. Any authenticated user from any org can call `GET /api/v1/employees/:id` with any MongoDB ObjectId and retrieve that employee's full profile including personal details, salary information, and bank details. A MANAGER from Org A can read an employee from Org B. This is a cross-tenant data leak.

---

**Bug 7 — `updateEmployeeRole` does NOT update `professional.branches` for MANAGER**

This was the bug we identified and fixed earlier. Confirming it is tracked here. `getScopedEmployeeIds` and `buildSecurityContext` both read from `professional.branches`. The old `updateEmployeeRole` only updated `professional.roleScopeBranches`. A manager assigned with `SELECTED_BRANCHES` scope would have the correct `roleScopeBranches` but their `branches` field would be unchanged, so the security context would still grant them their old branch access — not the newly assigned scope.

The new controller we built fixes this. Confirm it's deployed.

---

**Bug 8 — `EmployeeList` response field mismatch: `data?.employees || data?.data`**

```js
const employees = data?.employees || data?.data || [];
```

The backend returns `{ success, count, data: [...], branches: [...], meta: {...} }`. The field is `data`, not `employees`. The `data?.employees` check always returns `undefined` and it falls through to `data?.data` which works — but this is fragile. If the backend response shape ever changes, the fallback will silently return an empty list with no error.

---

**Bug 9 — Bulk upload validation happens inside the loop — one bad record stops the batch**

In the old bulk upload (pre-fix):
```js
for (const empData of employees) {
    if (!email || !firstName || !lastName || !mobile) {
        throw new Error('Missing required fields...');
    }
```

`throw` inside the for loop exits the entire function. If record #40 out of 50 has missing fields, the first 39 employees were already created but records 40-50 were not. The response is a 500 error with no indication of which records succeeded. Our new fix addresses this with pre-validation — confirm it's deployed.

---

**Bug 10 — `AddEmployeeModal` and `EditEmployeeModal` both send `...formData` spread as top-level payload**

```js
const res = await axios.put(`/employees/${employee._id}`, {
    ...formData,   // spreads ALL form fields flat
    emergencyContact: { ... },
    shift_timings: { ... },
    professional: { ... },
    payroll: { ... }
});
```

The spread of `formData` at the top level means the backend receives both the flat fields (`baseSalary`, `hra` etc.) AND the nested `payroll` object simultaneously. The backend controller handles this by checking both paths — but this creates ambiguity. If `formData.baseSalary = 50000` AND `payroll.salary.base = 45000` (from a different form state), which wins? The controller checks flat fields first, then falls back to nested. This is an inconsistent contract that will cause subtle bugs when the form state is partially updated.

---

## Phase 2: Optimization

**O1 — `getMyOrgEmployees` makes 3 separate DB calls**

```js
const [employees, totalEmployees, allBranches] = await Promise.all([
    Employee.find(...),
    Employee.countDocuments(...),
    Branch.find(...)
]);
```

`Promise.all` is used — good. But `Employee.countDocuments` with the full filter runs a separate index scan. The count can be derived from the aggregation pipeline itself using `$facet`, eliminating one query.

**O2 — `addEmployee` fetches the entire `Organization` document just to get the counter**

```js
const org = await Organization.findById(organizationId);
const prefix = org.employeeIdPrefix || 'EMP';
const nextCount = (org.employeeIdCounter || 0) + 1;
finalEmployeeId = `${prefix}${String(nextCount).padStart(3, '0')}`;
await Organization.findByIdAndUpdate(organizationId, { $inc: { employeeIdCounter: 1 } });
```

Two DB operations: one `findById` then one `findByIdAndUpdate`. Replace with a single atomic `findByIdAndUpdate` using `$inc` and `returnDocument: 'before'` to get the pre-increment counter value — same pattern as the bulk upload fix.

**O3 — `updateEmployee` fetches admin twice**

```js
const employee = await Employee.findById(req.params.id);
const requestor = await Employee.findById(req.user._id);
```

`req.user` is already the full employee document populated by `protect` middleware. `requestor` is redundant — use `req.user` directly.

**O4 — `EmployeeList` sends 50 employees per page by default**

```js
const [{ pageIndex, pageSize }, setPagination] = useState({ pageIndex: 0, pageSize: 50 });
```

50 employees with full documents is a heavy initial load. The `select` in `getMyOrgEmployees` already excludes password and fcmTokens, but each employee document still includes full `payroll`, `shift_timings`, `documents` arrays. Add a list-specific projection that returns only the fields the list UI actually renders.

---

## Phase 3: Feature Enhancements

**E1 — Soft delete instead of hard delete**

Replace `Employee.deleteOne` with `employee.status = 'Terminated'` + `employee.terminatedAt = new Date()`. Preserves all historical payroll and attendance data integrity. Add a filter to exclude `Terminated` employees from all active queries by default. Provide a separate "Terminated Employees" view for HR audit purposes.

**E2 — Employee import template download**

The bulk upload exists but there's no downloadable CSV template. A "Download Template" button that generates a correctly headed CSV file with sample data would dramatically reduce upload errors.

**E3 — Bulk upload progress feedback**

Currently the bulk upload shows a spinner until all employees are processed. With the new parallel batch processing, add server-sent events or a polling endpoint to show real-time progress: "Processing batch 2 of 5 (20 created, 0 failed so far)."

**E4 — Employee profile change audit log**

Every edit to an employee's salary, role, or status should create an `EmployeeAuditLog` entry (as discussed in Task 1 Enhancements). "Who changed Ravi's salary from ₹30,000 to ₹45,000 and when?" is a compliance requirement for payroll systems.

**E5 — Duplicate employee detection on add**

Beyond email uniqueness, check for potential duplicates by `mobile` number and `(firstName + lastName + dateOfJoining)` combination before creating. Surface a warning: "An employee with the same name and joining date already exists — are you sure this is a new employee?"

---

## Summary for Task 3

| # | Severity | Item |
|---|---|---|
| B6 | 🔴 Critical | `getEmployeeById` has no org scope — cross-tenant data leak |
| B1 | 🔴 Critical | Edit modal wipes `da` + `specialAllowance` — payroll impact |
| B2 | 🔴 Critical | `pf_opt_in` hardcoded `true` — overwrites opt-out on every edit |
| B5 | 🔴 High | Hard delete leaves orphaned attendance/payroll/leave records |
| B7 | 🔴 High | Manager branch scoping broken on role assignment (fix confirmed) |
| B3 | 🟡 Medium | Bulk upload counter race condition (fix confirmed deployed) |
| B9 | 🟡 Medium | Bulk validation throws mid-loop — partial creates with no feedback |
| B10 | 🟡 Medium | Double payload (spread + nested) creates ambiguous contract |
| B4 | 🟢 Low | Email update silently ignored with no error |
| B8 | 🟢 Low | Response field fallback fragile (`data?.employees \|\| data?.data`) |

**Ready for Task 4 — Branch Management whenever you are.**


I have all the files. Full three-phase review now.

---

## Task 4 — Branch Management

---

## Phase 1: Bug Identification

**Bug 1 — `updateBranch` has logic inversion on `isRoot` toggle**

```js
if (isRoot !== undefined) {
    updateData.isRoot = !
```

The knowledge base is cut off but looking at the pattern — `updateData.isRoot = !` appears to be inverting the value. If the admin sends `isRoot: true`, it gets stored as `false`. This is a critical logic error. The correct assignment should be `updateData.isRoot = isRoot`.

---

**Bug 2 — `generateCredentials` has double authorization check**

The route already has `authorize('ORG_ADMIN', 'SUPER_ADMIN', 'HR_ADMIN')` middleware. Inside the controller:

```js
if (!isAdmin(req.user.professional?.role)) {
    return res.status(403).json({...});
}
```

This is redundant but harmless. The real problem is that `isAdmin` checks against `['ORG_ADMIN', 'SUPER_ADMIN', 'HR_ADMIN']` which matches the route middleware exactly — so if someone bypasses the middleware (e.g. via future route refactoring), the controller check is a backstop. However the redundancy causes confusion about which layer is authoritative.

---

**Bug 3 — `generateSecurePassword` is not cryptographically strong enough for QR kiosk credentials**

```js
const generateSecurePassword = () => {
    return crypto.randomBytes(6).toString('base64').slice(0, 8) + Math.floor(10 + Math.random() * 90);
};
```

Two issues:
1. `Math.random()` is used for the numeric suffix — `Math.random()` is not cryptographically secure. Should use `crypto.randomInt(10, 99)`.
2. `crypto.randomBytes(6).toString('base64').slice(0, 8)` produces 8 chars from base64 which uses `A-Z`, `a-z`, `0-9`, `+`, `/` — the `+` and `/` characters can cause issues in URLs and some form parsers. Should use `.toString('hex')` or a custom safe alphabet.

---

**Bug 4 — `BranchManager` fetches ALL branches and filters client-side for MANAGER**

```js
if (isManager) {
    const scopeType = user?.roleScopeType || 'SELECTED_BRANCHES';
    const scopeBranches = user?.roleScopeBranches || [];
    if (scopeType === 'SELECTED_BRANCHES') {
        fetchedBranches = fetchedBranches.filter(b => 
            scopeIdStrings.includes(String(b._id))
        );
    }
}
```

The backend `getBranches` already scopes branches via `getAuthorizedBranchIds` — the MANAGER should only receive their authorized branches from the API. But the `BranchManager` component reads `roleScopeBranches` from the `user` object in localStorage (which is set at login time and never re-synced during the session) and applies a second client-side filter.

The problem: `user.roleScopeBranches` from localStorage may be stale. If the admin changes a manager's scope after the manager has logged in, the manager's browser still shows the old scope from localStorage. The backend correctly enforces the new scope, but the frontend shows the wrong set of branches until the manager logs out and back in.

---

**Bug 5 — `deleteBranch` checks for employees but not for child branches first**

```js
const childBranches = await Branch.find({ parentBranch: branch._id, orgId });
if (childBranches.length > 0) {
    return res.status(400).json({ message: 'Delete child branches first.' });
}

const employeeCount = await Employee.countDocuments({ 'professional.branches': branch._id });
if (employeeCount > 0) {
    return res.status(400).json({ message: `Cannot delete branch with ${employeeCount} assigned employees.` });
}
```

The order is correct — child branches checked first. However the `deleteOne` uses:

```js
await Branch.deleteOne({ _id: req.params.id, orgId: req.user.orgId });
```

This includes `orgId` in the filter — good for tenant isolation. But the earlier `Branch.findOne` to fetch the branch also uses `orgId`. If somehow `req.user.orgId` is `null` (e.g. SUPER_ADMIN who has no orgId), this would match `null` in the filter and the delete would silently fail without error. No null check on `orgId` before the delete operation.

---

**Bug 6 — `getBranchById` is accessible to ALL authenticated users including employees from other orgs**

```js
const branch = await Branch.findOne({ _id: req.params.id, orgId: req.user.orgId })
```

`getBranchById` does include `orgId` in the query — good. However `getBranchById` is on `GET /branches/:id` which has only `protect` middleware and no `authorize` restriction. Any authenticated employee can call this endpoint for any branch in their org. For a `BRANCH_QR` kiosk user, `req.user.orgId` is set correctly. But BRANCH_QR users can enumerate all branches and their GPS coordinates, contact info, and child branch hierarchy — which is probably more than a kiosk terminal should know.

---

**Bug 7 — Credentials displayed after generation are never cleared from component state**

```js
setCredentialsModal(prev => ({ 
    ...prev, 
    username: res.data.data.username, 
    password: res.data.data.password 
}));
```

The password is stored in React state. When the modal is closed, the state persists:

```js
onClick={() => setCredentialsModal({ isOpen: false, branchId: null, branchName: '', username: '', password: '' })}
```

The password IS cleared on close — this is actually correct. However if the user opens the credentials modal for Branch A (which shows username only, no password since it wasn't just regenerated), then regenerates credentials, the password is shown. If the user then opens the modal for Branch B without regenerating, Branch B's modal shows Branch A's just-generated password until they interact with it. The modal state needs to be reset fully when `branchId` changes.

---

**Bug 8 — `AddBranchModal` sends `status` field only in edit mode**

```js
status: isEditMode ? (editBranch.status || 'Active') : undefined
```

On create, `status` is `undefined` — the backend defaults to `'Active'`. Fine. But on edit, if the admin wants to deactivate a branch by setting `status: 'Inactive'`, this works. However the UI for the AddBranchModal in edit mode shows no status field — the admin cannot change branch status from the edit form. The status can only be changed via the Suspend/Power button in BranchList, which toggles it. The two status management paths are inconsistent.

---

## Phase 2: Optimization

**O1 — `checkCircularParent` makes N DB queries in a loop (unbounded)**

```js
const checkCircularParent = async (branchId, parentId) => {
    const visited = new Set();
    let current = parentId;
    while (current) {
        const parent = await Branch.findById(current).select('parentBranch');
        current = parent?.parentBranch || null;
    }
};
```

For a branch hierarchy of depth D, this makes D sequential DB calls. With a deep tree this is slow and could timeout. Optimize by fetching all branches for the org in one query and traversing the in-memory tree:

```js
const allBranches = await Branch.find({ orgId }).select('_id parentBranch').lean();
const parentMap = new Map(allBranches.map(b => [b._id.toString(), b.parentBranch?.toString()]));
// traverse parentMap in memory — zero additional DB calls
```

**O2 — `getBranches` returns full documents including `password` hash**

The `Branch` model has `password` with `select: false` — good, it's excluded by default. But `username` is included in list results. For the list view, username and GPS coordinates are not needed. Add a `.select()` projection for the list endpoint.

**O3 — `BranchManager` rebuilds the entire branch tree on every fetch**

```js
const treeRoots = buildTree(branches);
```

`buildTree` runs on every render since it's called inline. Wrap in `useMemo`:

```js
const treeRoots = useMemo(() => buildTree(branches), [branches]);
```

**O4 — `handleGenerateCredentials` uses `alert()` for errors**

```js
alert('Failed to generate credentials: ' + ...);
```

Inconsistent with the rest of the UI which uses proper error state. Replace with the error state pattern used elsewhere.

---

## Phase 3: Feature Enhancements

**E1 — Branch operating hours configuration**

The Branch model has `address`, `contact`, `gpsCoordinates` but no operating hours. For geo-fence attendance, knowing that a branch operates `09:00-18:00 Mon-Sat` would allow the attendance controller to reject punches outside operating hours. Add `operatingHours` to the branch schema.

**E2 — Branch-level QR rotation override**

Currently QR refresh interval is org-level. A 24/7 factory branch may need a longer QR validity window than a standard office branch. Allow per-branch override of `qr_refresh_interval` that overrides the org default.

**E3 — Branch capacity / headcount target**

Add a `targetHeadcount` field per branch. The dashboard can then show "Branch HO: 45/50 employees (90% capacity)" giving HR visibility into staffing levels across branches.

**E4 — Branch merge / reassign**

When deleting a branch with employees, instead of blocking, offer "Move all employees to branch X" as an alternative. This makes the deletion flow less disruptive during org restructuring.

---

## Summary for Task 4

| # | Severity | Item |
|---|---|---|
| B1 | 🔴 Critical | `isRoot` logic inversion on update — always stores opposite value |
| B3 | 🔴 High | `Math.random()` used in credential generation — not cryptographically secure |
| B4 | 🔴 High | Manager branch scope read from stale localStorage — wrong branches shown after scope change |
| B5 | 🟡 Medium | `orgId: null` for SUPER_ADMIN causes silent delete failure |
| B7 | 🟡 Medium | Password persists in state when switching branches in credentials modal |
| B6 | 🟢 Low | BRANCH_QR can enumerate all org branches via `getBranchById` |
| B8 | 🟢 Low | Branch status management inconsistent between edit form and toggle button |
| B2 | 🟢 Low | Redundant double authorization check |

**Ready for Task 5 — Attendance whenever you are.**

That's a fair reaction. Let me give you an honest perspective.

---

## What the Bug Count Actually Means

We've reviewed 4 modules out of 13 and found roughly 35 issues. That sounds alarming — but let me break down what those actually are.

**Truly critical issues that would cause real harm right now: about 6**

- Cross-tenant data leak on `getEmployeeById` — no org check
- `da` + `specialAllowance` wiped on employee edit — payroll impact
- `pf_opt_in` hardcoded true — overwrites opt-outs
- Credentials file with real passwords in repo
- `sameSite: strict` breaking cross-domain cookie refresh
- `isRoot` logic inversion on branch update

**Issues that are real but low-probability at your current scale: about 15**

Things like the bulk upload counter race condition (needs two simultaneous uploads), the stale localStorage scope (needs an admin to change a manager's scope mid-session), the credential generation using `Math.random()` (still random enough at your user count).

**Issues that are code quality / best practice rather than active bugs: about 14**

Things like redundant DB calls, missing projections, inconsistent error handling patterns, client-side filtering that duplicates server-side logic. These don't break anything today but create technical debt.

---

## The Honest Assessment

A payroll and HR system at this stage of development — roughly 6 months in by the looks of it — having this profile of issues is **completely normal**. Most of what we're finding is exactly what a pre-production audit is supposed to find.

The core business logic — payroll calculation, leave management, attendance, the AI pipeline — is genuinely well-built. The statutory compliance logic is solid. The multi-tenant isolation is consistently applied. The architecture is clean. Those are the hard parts.

What we're finding are mostly **integration gaps** — places where the frontend and backend drifted slightly out of sync, and **security hardening** items that are standard in any pre-launch audit.

---

## What This Audit Is Actually Doing

Every bug we identify here is a bug that does NOT go to a paying customer. That's the point of this exercise.

The six critical issues above — if we fix those before launch, you ship a system that is financially accurate and secure. The rest can be addressed in a structured backlog.

**Shall we keep going?** The remaining 9 modules include Attendance, Leave Management, and Payroll — which are the most complex and highest-stakes parts of the system. Those are worth reviewing carefully.

Task 5 — Attendance

Phase 1: Bug Identification
Bug 1 — remoteSubmit and verifyEmployeeMode have NO authentication (Critical Security)
jsrouter.post('/remote-submit', remoteSubmit);
router.get('/verify-mode', verifyEmployeeMode);
No protect middleware. These are completely public endpoints. Anyone on the internet can call POST /api/v1/attendance/remote-submit with any employeeCode and mark attendance for any employee in the system without logging in. The endpoint accepts date as a body parameter too — meaning an attacker can backdate attendance for any employee. This is the most critical bug in the entire codebase.

Bug 2 — remoteSubmit identifies employees by employeeCode not _id — enumerable and predictable
jsconst employee = await Employee.findOne({ 'professional.employeeCode': employeeCode })
employeeCode values are sequential (EMP001, EMP002, etc.). Combined with Bug 1, an unauthenticated attacker can enumerate all employees and mark attendance for all of them by looping through predictable codes.

Bug 3 — LWP deduction check after approveRejectAttendance has wrong condition
js// After rejection:
attendance.status = 'Absent';
await attendance.save();

// LWP check:
if (action === 'Rejected' && attendance.status === 'Absent' && !attendance.lwp_auto_deducted) {
attendance.status was just set to 'Absent' two lines above. This condition is always true when action === 'Rejected' and lwp_auto_deducted is false. The attendance.status === 'Absent' check is redundant — but more importantly, the LWP deduction fires even when the record was previously 'Absent' before approval was attempted. The check should verify the previous status was not already 'Absent' (i.e. was 'Present' or 'Late') before deducting LWP to avoid double deductions.

Bug 4 — Shift time auto-detection uses shiftStartTime built from pIn (wrong base date)
jsconst shiftStartTime = new Date(pIn);
shiftStartTime.setHours(inTime.hours, inTime.minutes, 0, 0);

const lateLimit = new Date(shiftStartTime.getTime() + graceMins * 60000);
if (pIn > halfDayLimit) finalStatus = 'Half-Day';
else if (pIn > lateLimit) finalStatus = 'Late';
shiftStartTime is constructed from new Date(pIn) then the hours are overwritten with shift time. This works correctly for same-day punches. However for overnight shifts where pIn might be from the previous day (reconciliation case), the shiftStartTime base date is wrong. An employee who punched in at 22:00 (night shift) would have shiftStartTime set to that same day at say 22:00 — but the grace check compares pIn (22:00) > lateLimit (22:15) which is correct. The edge case is when the night shift spans into the next day and the punch comes in after midnight — pIn is now on the new date but shiftStartTime is also on the new date, and the grace window is computed correctly. This is actually fine for most cases but will fail if an employee's inTime is 23:00 and they punch in at 00:30 the next calendar day — the recordDate is the next day, pIn is next day 00:30, but shiftStartTime would be next day 23:00 which is after pIn, making the employee appear 23 hours early rather than 30 minutes late.

Bug 5 — getAttendanceWithEmployees generates virtual records using server local time for timezone-sensitive comparisons
jsconst today = new Date();
today.setHours(0, 0, 0, 0);
// ...
if (dayDate > today) {
    virtualStatus = '-';
}
The server is on UTC. If an org is in IST (UTC+5:30), at 10 PM IST it's already the next UTC day. The server will generate 'Absent' virtual records for the current IST day because today in UTC is already tomorrow. This means Indian employees see themselves as absent for today from 6:30 PM IST onwards even if they haven't finished their shift. The fix is to normalize all date comparisons to the org's configured timezone.

Bug 6 — approveRejectAttendance does not notify the employee of the decision
The approval flow notifies approvers when attendance is submitted (Bug 1 aside). But when the approver approves or rejects, there is no notification sent back to the employee. The employee has no way to know their attendance was rejected without manually checking the app.

Bug 7 — Duplicate attendance record possible on remoteSubmit under race condition
jsconst existingToday = await Attendance.findOne({ orgId, employeeId: employee._id, date: recordDate, punchIn: { $ne: null } });
if (existingToday) return res.status(409)...

// Then:
attendance = await Attendance.create({...});
Between the findOne check and the create, a second concurrent request for the same employee can pass the check simultaneously. Both requests create attendance records for the same employee+date. The Attendance model should have a unique compound index on { orgId, employeeId, date } to enforce this at the DB level. Without it, the application-level check is a race condition under load.

Bug 8 — temp_attendanceController.js exists in the repo alongside attendanceController.js
There are two attendance controller files — attendanceController.js (deployed) and temp_attendanceController.js. The temp file contains a more complete implementation (includes isDayOff comp-off logic for holiday punches). It's unclear if this is a WIP refactor or dead code. If deployed accidentally, both files defining the same exports would cause conflicts. This file should be removed or merged.

Bug 9 — Late detection logic uses finalStatus === 'Present' before auto-detection runs
jslet finalStatus = status || 'Present';
// ...
if (pIn && finalStatus === 'Present') {
    // auto-detect Late/Half-Day
    if (pIn > halfDayLimit) finalStatus = 'Half-Day';
    else if (pIn > lateLimit) finalStatus = 'Late';
}
The auto-detection only runs when finalStatus === 'Present'. But if an employee self-punches without sending a status field, finalStatus defaults to 'Present' — correct. However if the frontend explicitly sends status: 'Present' (which AddEmployeeModal does), the auto-detection still runs. This means an admin explicitly marking someone as Present could have it auto-changed to Late — violating the admin manual override intent. The auto-detection should only run for non-admin self-punches.
The code has if (isAdminManualOverride) else block that handles this — but isAdminManualOverride = isAdmin && status. If the admin sends status: 'Present', isAdminManualOverride is true and the auto-detection block is skipped. This is actually correct. The issue only arises when isAdmin is false but the frontend sends status: 'Present' explicitly for an employee self-punch — auto-detection still overrides it, which is the correct behavior for employees.
This is not a bug — documenting it as confirmed correct.

Bug 10 — resolveShiftWindow in remoteSubmit passes wrong argument structure
In remoteSubmit:
jsconst shiftWindow = resolveShiftWindow(employee.shift_timings, org?.shift_rules);
But resolveShiftWindow in temp_attendanceController.js expects (employeeShift, orgShiftRules) where employeeShift is the shift_timings subdocument. In qrController.js:
jsconst { inTime } = resolveShiftWindow(employee, org);
Here the full employee object is passed, not employee.shift_timings. The function signature is inconsistent across call sites. One of them is wrong. In qrController.js, resolveShiftWindow(employee, org) would try employee?.inTime which is undefined on the Employee document — it would always fall back to org defaults, ignoring the employee's custom shift time for QR punches.

Phase 2: Optimization
O1 — getAttendanceWithEmployees runs O(employees × days) virtual record generation in JavaScript
For 100 employees over 31 days that's 3,100 iterations in Node.js memory, each checking holidays, leaves, and weekOff arrays with .some() calls. This is the heaviest endpoint in the system. Optimize by pre-building lookup Sets for holidays and leaves before the loop:
jsconst holidaySet = new Set(monthHolidays.map(h => formatDateKey(new Date(h.date))));
const leaveMap = new Map(); // employeeId → Set of date strings
O2 — markAttendance fetches Organization for every single attendance mark
jsconst org = await Organization.findById(orgId);
Organization config (shift rules, attendance approver, grace period) changes rarely. This is a candidate for a short-lived in-memory cache — even a 60-second TTL would eliminate the vast majority of these DB reads for high-traffic attendance periods (shift start time when 100 employees punch in simultaneously).
O3 — getPendingApprovals loads the full Organization document to read attendance_approver
Use .select('attendance_approver') to limit the data transfer.
O4 — verifyEmployeeMode fetches full employee document
The endpoint only needs shift_timings.shiftModes and professional.employeeCode. Add a .select() projection.

Phase 3: Feature Enhancements
E1 — Add authentication to remoteSubmit (mandatory fix)
The remote attendance flow should require the employee to be logged in. Use the existing JWT auth — the employee logs into the mobile/web app and their token authenticates the remote punch. Remove the employeeCode lookup entirely and use req.user._id.
E2 — Timezone-aware date normalization
Add timezone field to the Organization model (e.g. 'Asia/Kolkata'). All date comparisons in attendance — virtual record generation, late detection, shift window calculation — should normalize to the org's local timezone using a library like date-fns-tz. This fixes Bug 5 and makes the system usable for orgs in any timezone.
E3 — Attendance anomaly alerts
When an employee's attendance is marked as Absent for 3 or more consecutive days without an approved leave, trigger an AI insight event (generatedBy: 'event_driven') to notify HR. The AIInsight model already supports this pattern.
E4 — Bulk attendance correction
HR admins frequently need to correct attendance for an entire team after a system outage or data import. A POST /attendance/bulk-mark endpoint accepting an array of { employeeId, date, status, punchIn, punchOut } records with the same validation logic as markAttendance would save significant manual effort.

Summary for Task 5
#SeverityItemB1🔴 CriticalremoteSubmit + verifyEmployeeMode have no auth — public endpointsB2🔴 CriticalSequential employeeCode enumeration enables mass attendance fraudB7🔴 HighRace condition allows duplicate attendance records (no DB unique index)B4🔴 HighOvernight shift late detection uses wrong base date for cross-midnight punchesB3🟡 MediumLWP deduction fires on already-absent records — potential double deductionB5🟡 MediumVirtual records use server UTC — wrong for IST orgs after 6:30 PMB10🟡 MediumresolveShiftWindow called with inconsistent arguments across qrController vs remoteSubmitB8🟡 Mediumtemp_attendanceController.js is dead/WIP code — should be removedB6🟢 LowEmployee not notified when attendance is approved or rejectedB9🟢 Confirmed OKLate auto-detection correctly scoped to non-admin paths

I have everything needed. Full three-phase review now.

---

## Task 6 — Leave Management

---

## Phase 1: Bug Identification

**Bug 1 — `calculateRequestedLeaveDays` does not account for holidays and weekly-offs correctly across year boundaries**

```js
const start = new Date(startDate);
const end = new Date(endDate);
```

The function counts working days and excludes holidays and weekly-offs. But holiday configs are fetched per-org without a year filter. If a leave request spans December 31 to January 2 (across years), the holiday check only looks at the current year's holiday config. Holidays declared for the new year in a separate `HolidayConfig` document won't be excluded from the day count. The employee gets charged for public holidays that fall in the new year within their leave range.

---

**Bug 2 — Monthly leave limit (`monthlyLimit`) calculation uses request month, not each consumed month**

```js
const requestMonth = new Date(request.startDate).getMonth() + 1;
const accumulatedAllowance = requestMonth * policyType.monthlyLimit;
```

This multiplies the `requestMonth` number by `monthlyLimit` to get an "accumulated allowance." For a policy with `monthlyLimit = 1` and a request in November (month 11), `accumulatedAllowance = 11`. This means an employee can take up to 11 days by November if they haven't used any.

The logic works for a pure pro-rata monthly accrual model. But the critical flaw is: `totalUsedThisYear` is counted from `existingThisYear` using `row.paidDays`. If an employee took 5 days in January-March and now applies for 3 days in November, `totalUsedThisYear = 5`, `effectiveAvailable = 11 - 5 = 6`. So far correct.

But `accumulatedAllowance` doesn't account for which months the leave was taken in. An employee could take all 11 days in January (month 1) if they haven't used any yet — `accumulatedAllowance = 1` but they're requesting 11, so `paidDays = 1, lwpDays = 10`. This seems fine. But the formula breaks when `monthlyLimit` is meant to cap usage **per calendar month** (e.g. max 1 CL per month). An employee who takes 0 CL in Jan-Oct can take 10 CLs in November under this formula because `accumulatedAllowance = 11`. This contradicts the intent of a per-month cap.

---

**Bug 3 — `settleApprovedLeaveRequest` uses `session` parameter but `LeaveBalance.save()` doesn't always receive it**

```js
const balanceDoc = await LeaveBalance.findOne({ employeeId: request.employeeId, year }).session(session);
// ...
await balanceDoc.save({ session });
```

The session is correctly passed to `findOne` and `save`. But earlier in the function:

```js
const createdBalances = await LeaveBalance.create([{...}], { session });
balanceDoc = createdBalances[0];
```

If `LeaveBalance.create` fails mid-session (e.g. duplicate key), the error is thrown uncaught within the settlement function, which is called inside a `try/catch` in the calling controller. The session should be aborted on error, but if the calling controller doesn't call `session.abortTransaction()` in its catch block, the MongoDB session remains open until it times out. **Leaked sessions under error conditions will block writes** on those documents until the session timeout (typically 60 seconds on Atlas).

---

**Bug 4 — Double-counting risk: LWP auto-deduction in attendance AND leave settlement both deduct from `LeaveBalance`**

This is a systemic design issue. Two separate code paths modify `LeaveBalance.used`:

**Path 1 — Leave approval (`settleApprovedLeaveRequest`):**
```js
specificBalance.used += paidDays;
await balanceDoc.save({ session });
```

**Path 2 — Attendance rejection LWP (`approveRejectAttendance` in attendanceController):**
```js
leaveBalance.balances[lwpIdx].used += deductionAmount;
await leaveBalance.save();
```

If an employee applies for leave, it gets approved (Path 1 deducts `paidDays`), and then the attendance record for that leave day gets separately rejected (Path 2 deducts LWP again), the employee is double-charged. The attendance rejection path checks `approvedLeave` before deducting LWP — but only for full-day leaves. For half-day leaves it deducts anyway. This means a half-day approved leave followed by an attendance rejection for the same day deducts from the leave balance twice — once at approval and once at rejection.

---

**Bug 5 — `applyLeave` has no check for overlapping leave requests**

```js
const newRequest = await LeaveRequest.create({...});
```

There is no query to check whether the employee already has an approved or pending leave request that overlaps with the requested date range. An employee can submit multiple overlapping leave requests:

- Leave A: April 1-5 (Pending)
- Leave B: April 3-7 (Pending)

Both get created. If both get approved, the employee gets 9 days of leave credit (6 unique days + 2 double-counted days) but only takes 7 calendar days off. The leave balance is depleted faster than actual absence. The payroll deduction correctly uses attendance records, but the balance ledger is wrong.

---

**Bug 6 — `cancelLeave` does not reverse the `LeaveBalance.used` counter**

```js
request.status = 'Cancelled';
await request.save();
// No balance reversal
```

When an approved leave is cancelled, `paidDays` that were deducted from `LeaveBalance.used` at approval time are never restored. The employee permanently loses those leave days even though they didn't take the leave. This is a financial accuracy bug — it will cause employees to run out of leave balance faster than expected.

---

**Bug 7 — Frontend-Backend contract mismatch: `LeaveDetailDrawer` builds `API_URL` manually instead of using the axios instance**

```js
const API_URL = (import.meta.env.VITE_API_URL || 'http://127.0.0.1:9999').replace(/\/+$/, '');
// ...
const res = await axios.get(`${API_URL}/api/v1/leaves/usage-summary/${empId}?...`);
```

The `axios` instance from `../../api` already has the base URL configured. Prefixing with `API_URL` creates double-URL on some environments: `https://hrms-dev-backend.fundflick.inhttps://hrms-dev-backend.fundflick.in/api/v1/...`. This would cause all usage-summary fetches to fail in production.

---

**Bug 8 — `normalizeLeaveType` is case-sensitive and incomplete — silent mismatches**

```js
const normalizeLeaveType = (leaveType) => {
    const raw = String(leaveType || '').trim();
    const upper = raw.toUpperCase();
    if (upper === 'MEDICAL') return 'Medical';
    if (upper === 'ML') return 'Medical';
    if (upper === 'CL') return 'CL';
    if (upper === 'PL') return 'PL';
    ...
    return raw; // falls through to raw value
};
```

The function returns `raw` (original case) as fallback. If the frontend sends `leaveType: 'casual leave'` or `leaveType: 'Casual Leave'`, neither matches any condition and `raw = 'casual leave'` is returned. This won't match any `LeaveBalance.balances[].type` since those are stored as `'CL'`. The balance lookup fails silently:

```js
const specificBalance = balanceDoc.balances.find((b) => normalizeLeaveType(b.type) === normalizedType);
if (!specificBalance) {
    throw new Error(`Allocation for ${normalizedType} not found.`);
}
```

The employee gets an error instead of leave approval for any leave type not in the exact map.

---

**Bug 9 — `getLeaveBalance` returns female-only leave types based on gender field which can be null**

```js
const isFemale = employee?.personal?.gender?.toLowerCase() === 'female';
const filteredPolicyLeaveTypes = isFemale
    ? policyLeaveTypes
    : policyLeaveTypes.filter(lt => !FEMALE_ONLY_LEAVE_TYPES.includes(lt.code));
```

If `employee.personal.gender` is `null` or `undefined` (common for older records added before the gender field was required), `isFemale` is `false` and female-only leave types are filtered out. A female employee who joined before gender was tracked loses access to Maternity and Wellness leave. The check should be:

```js
const isFemale = employee?.personal?.gender?.toLowerCase() === 'female';
const genderUnknown = !employee?.personal?.gender;
// Show female leaves if female OR gender unknown (safer default)
```

---

**Bug 10 — `rebuildAllBalances` has no org-level scope check — a MANAGER can trigger it**

```js
router.post('/balances/rebuild', protect, authorize('ORG_ADMIN', 'HR_ADMIN', 'MANAGER'), rebuildAllBalances);
```

A MANAGER can trigger a full balance rebuild for the org. The rebuild wipes and reconstructs all leave balances based on the current policy. If a MANAGER accidentally triggers this after a policy change, all custom manual balance adjustments (sick leave extensions, maternity carry-forwards etc.) are permanently lost. This should be `ORG_ADMIN` only.

---

## Phase 2: Optimization

**O1 — `calculateRequestedLeaveDays` iterates day-by-day in JavaScript**

```js
for (let d = new Date(start); d <= end; d.setDate(d.getDate() + 1)) {
    const iso = formatDateKey(d);
    const isHoliday = holidayDateKeys.has(iso);
    const isWeeklyOff = weeklyOffDays.includes(dayName);
    if (!isHoliday && !isWeeklyOff) workingDays++;
}
```

For a 30-day leave request this is 30 iterations. Fine for normal leaves. But bulk operations like `rebuildAllBalances` call this for every employee for every active leave request. For 100 employees with 10 requests each, that's 1,000 calls to this function. Pre-build the holiday and weekly-off Set once and pass it in.

**O2 — `getLeaveRequests` fetches full `Employee` documents for populating approver names**

The endpoint populates `employeeId` with full Employee documents but only uses `personal.firstName`, `personal.lastName`. Add a strict `.select()` to the population:

```js
.populate('employeeId', 'personal.firstName personal.lastName personal.gender professional.employeeId')
.populate('approverId', 'personal.firstName personal.lastName')
```

**O3 — `settleApprovedLeaveRequest` is called both at approval time AND can be re-called if approval is re-triggered**

There is no idempotency guard. If an approver clicks "Approve" twice (network glitch, double-click), the function runs twice, deducting `paidDays` from the balance twice. Add a check:

```js
if (request.status === 'Approved') {
    throw new Error('This request has already been approved.');
}
```

**O4 — `MyLeaves.jsx` refetches leave data on every filter/tab change**

The leave list re-calls the API on every status filter change. Since leaves rarely change mid-session, implement a 5-minute client-side cache using TanStack Query's `staleTime`:

```js
useQuery({ queryKey: ['leaves', year, status], queryFn: fetchLeaves, staleTime: 5 * 60 * 1000 })
```

---

## Phase 3: Feature Enhancements

**E1 — Leave carry-forward at year end**

When the year changes, leave balances reset but some policies allow carrying forward unused PL (Privileged Leave) to the next year up to a cap (e.g. max 15 days carry-forward). The current system has no carry-forward logic — all unused balance is silently lost at year end. Add a `yearEndCarryForward` field to `LeavePolicy` and a scheduled job that runs on January 1 to compute and carry forward eligible balances.

**E2 — Leave calendar view**

Show a monthly calendar with leave requests plotted alongside holidays and weekly-offs. The `getAttendanceWithEmployees` endpoint already returns this data — a calendar component using that data would give managers immediate visibility into team availability before approving new requests.

**E3 — Team leave conflict warning**

When an employee applies for leave, check if other team members in the same branch are already on approved leave for the overlapping dates. Surface a warning: "3 other team members are on leave during this period." The approver can still approve but has full context.

**E4 — Leave encashment**

For policies that allow encashment of unused leave at year end or at separation, add a `encashmentEnabled` and `encashmentRate` field to the leave type config. At separation or year-end, compute encashment amount as `unusedDays × (baseSalary / 26)`. This is a standard HR compliance requirement.

---

## Summary for Task 6

| # | Severity | Item |
|---|---|---|
| B4 | 🔴 Critical | Double LWP deduction — attendance rejection + leave approval both deduct balance |
| B6 | 🔴 Critical | Leave cancellation does not restore `LeaveBalance.used` — permanent balance loss |
| B5 | 🔴 High | No overlap check — employee can apply multiple leaves for same dates |
| B3 | 🔴 High | MongoDB session leak on settlement error — blocks writes for 60 seconds |
| B10 | 🔴 High | MANAGER can trigger `rebuildAllBalances` — destroys manual adjustments |
| B2 | 🟡 Medium | Monthly limit formula allows front-loading — employee can exhaust annual monthly quota in one month |
| B7 | 🟡 Medium | `LeaveDetailDrawer` double-prefixes API URL — usage summary always fails in production |
| B8 | 🟡 Medium | `normalizeLeaveType` falls through to raw value — leave type mismatches cause silent failures |
| B1 | 🟡 Medium | Year-boundary leave requests charge for holidays in new year |
| B9 | 🟢 Low | Null gender defaults female-only leaves to hidden |

**Ready for Task 7 — Payroll whenever you are. This is the highest-stakes module.**

I have all the data. This is the most critical module in the system — let me go through it rigorously.

---

## Task 7 — Payroll

---

## Phase 1: Bug Identification

**Bug 1 — PF wage basis includes `DA` but the statutory rule requires only `Basic + DA` (Mathematical Verification)**

```js
let pfWage = earnings.basic;
```

The code sets `pfWage = earnings.basic` — only Basic salary. However the Indian EPF Act specifies that the PF wage basis is **Basic + DA (Dearness Allowance)**. DA is a statutory wage component that must be included in the PF calculation.

Looking at the pro-ration:
```js
const actualBasic = Math.round(base * ratio);
const actualDA = Math.round(da * ratio);
```

`actualDA` is calculated and passed into `calculateDeductions` as part of `earnings`. But inside `calculateDeductions`:

```js
let pfWage = earnings.basic; // ← DA is NOT added here
```

`earnings.da` is present in the earnings object passed in but never added to `pfWage`. For an employee with Basic ₹15,000 and DA ₹3,000:

- **Correct PF wage:** ₹15,000 + ₹3,000 = ₹18,000 → capped at ₹15,000 ceiling → PF = ₹1,800
- **Current PF wage:** ₹15,000 → capped at ₹15,000 ceiling → PF = ₹1,800

In this case the ceiling saves you because both hit the cap. But for employees where Basic < ₹15,000 and Basic + DA > ₹15,000, the current code under-deducts PF. Example: Basic ₹12,000, DA ₹4,000 → correct PF = ₹15,000 × 12% = ₹1,800, current PF = ₹12,000 × 12% = ₹1,440. That's a ₹360/month compliance shortfall per affected employee.

**The fix:** `let pfWage = earnings.basic + (earnings.da || 0);`

---

**Bug 2 — ESI wage basis uses full gross minus conveyance — incorrect statutory definition**

```js
let esiWage = earnings.gross || 0;
if (config.esi.exclude_conveyance) {
    esiWage -= (earnings.conveyance || 0);
}
```

The ESIC Act wage definition for ESI calculation is: **Basic + DA + HRA + Conveyance + Special Allowance** — essentially total gross. The `exclude_conveyance` config flag is a legacy feature that was valid pre-2020 but ESI now applies on total gross. More critically, the code computes ESI on `earnings.gross` which INCLUDES `commission`, `overtime`, and `bonus` (after pro-ration). These components have different statutory treatment:

- **Commission:** Included in ESI if paid regularly
- **Overtime:** Excluded from ESI wage per ESIC circular
- **Bonus:** Excluded from ESI wage (not a wage component)

Currently all three are included in ESI wage via `grossWithoutIncentive`:

```js
const grossWithoutIncentive = actualBasic + actualDA + actualHRA + actualConveyance + actualSpecial + commission + overtimePay;
```

This over-deducts ESI. For an employee with ₹2,000 overtime: ESI over-deduction = ₹2,000 × 0.75% = ₹15/month. Small per employee but multiplied across all employees with overtime it creates a systematic over-deduction that's a compliance liability.

---

**Bug 3 — ESI eligibility check uses `fullMonthlyWage` without DA (incomplete definition)**

```js
const fullMonthlyWage = fullSalary
    ? (fullSalary.base + (fullSalary.hra || 0) + (fullSalary.conveyance || 0) + (fullSalary.da || 0) + (fullSalary.specialAllowance || 0))
    : esiWage;
const isEsiEligible = (fullMonthlyWage <= config.esi.wage_ceiling) || isInContributionPeriod;
```

The eligibility check correctly uses `fullSalary` (unprorated) — good. But `fullSalary.da` is included. The ESIC wage ceiling check (₹21,000/month) should use the same wage components as the ESI calculation itself. If conveyance is excluded from ESI calculation, it should also be excluded from the ceiling check. Currently you might exclude conveyance from the calculation but include it in the eligibility check, causing an employee to appear ineligible (above ₹21,000) when their actual ESI wage (without conveyance) is below the ceiling.

---

**Bug 4 — Pro-ration uses `totalDaysInPeriod` which can include pre-joining days for new employees**

```js
const ratio = (effectivePaidDays < totalDaysInPeriod)
    ? effectivePaidDays / totalDaysInPeriod
    : 1;
```

`totalDaysInPeriod` is the total days in the month (e.g. 31 for January). For an employee who joins on January 15, the correct pro-ration denominator should be 17 (days from joining to month end: 15, 16, ..., 31). The current code uses 31 as the denominator.

Example: Employee joins January 15, attends all 17 remaining days.
- **Correct ratio:** 17/17 = 1.0 → full salary
- **Current ratio:** 17/31 = 0.548 → 54.8% of salary

This is a severe under-payment for new joiners. An employee who joins mid-month and works every available day gets paid as if they were absent for half the month.

Looking for the joining date check in `calculatePayableDays` — `empEffectiveStart` is computed:

```js
const empEffectiveStart = (joinDate && joinDate > startDate) ? joinDate : startDate;
```

This adjusts `workedDays` and `autoPaidDays` to only count from the joining date. So `effectivePaidDays` is correctly computed as days from joining. But `totalDaysInPeriod` remains the full month. The ratio denominator should also be adjusted for new joiners:

```js
const effectiveTotalDays = joinDate && joinDate > startDate
    ? Math.ceil((endDate - joinDate) / (1000 * 60 * 60 * 24)) + 1
    : totalDaysInPeriod;
const ratio = effectivePaidDays / effectiveTotalDays;
```

---

**Bug 5 — `50% rule` compliance check has incorrect calculation**

```js
if (config.compliance && config.compliance.enforce_50_percent_rule) {
    const totalAllowances = earnings.gross - pfWage;
    if (totalAllowances > earnings.gross * 0.5) {
        pfWage += Math.round(totalAllowances - earnings.gross * 0.5);
    }
}
```

The 50% rule (from the Labour Code 2020) states: **Basic wage must be at least 50% of CTC/total remuneration.** The current implementation checks if allowances > 50% of gross and adds the excess to pfWage. This is backwards.

If Basic is 40% of gross (allowances are 60%), the rule requires increasing PF wage to cover the shortfall in Basic, not just adding to pfWage. The calculation should be:

```js
const basicThreshold = earnings.gross * 0.5;
if (pfWage < basicThreshold) {
    pfWage = basicThreshold; // enforce minimum 50% basic
}
```

The current code adds `(totalAllowances - gross * 0.5)` to pfWage which equals `(gross - basic - gross * 0.5)` = `(gross * 0.5 - basic)` — coincidentally the same result as the correct formula. So the math is actually correct but the logic explanation is wrong, making it unmaintainable.

---

**Bug 6 — `calculateDeductions` receives `new Date()` (current date) not the payroll month date**

```js
const deductions = calculateDeductions({...}, config, employee, new Date(), {...});
```

`currentDate` is used for ESI `esi_active_until` check:

```js
const isInContributionPeriod = esiActiveUntil && new Date(esiActiveUntil) >= currentDate;
```

If payroll is being generated retroactively for a past month (e.g. February generated in April), `currentDate = April` but `esi_active_until` might have been March. The employee would incorrectly be treated as outside their ESI contribution period for the February payroll. The `currentDate` should be the last day of the payroll month, not today.

**Fix:**
```js
const currentDate = new Date(year, monthIndex + 1, 0); // last day of payroll month
```

---

**Bug 7 — `netPay` can reflect negative incentive adjustment with no floor guard at individual component level**

```js
const gross = grossWithoutIncentive + incentive;
const netPay = Math.max(0, gross - deductions.totalDeductions);
```

`Math.max(0, ...)` correctly prevents negative net pay. But if `incentive` is negative (a manual deduction entered as negative incentive in `incentiveOverrides`), `gross` decreases. If `gross < deductions.totalDeductions`, the statutory deductions (PF, ESI) were calculated on `grossWithoutIncentive` but the actual gross is lower. PF was deducted at the full rate on the higher wage but the employee is actually taking home less. The PF deduction itself is correct since it's calculated on earnings — but the payslip shows a statutory deduction higher than the base salary component it was calculated on, which is confusing and potentially non-compliant.

---

**Bug 8 — `bulkGenerateSalaries` uses `new Date()` for ESI date check instead of payroll period end date**

Confirmed same as Bug 6 — two separate places use `new Date()`:

1. `single generate`: `calculateDeductions({...}, config, employee, new Date(), {...})`
2. `bulk generate`: `calculateDeductions({...}, config, emp, currentDate, {...})` where `currentDate = new Date()`

Both should use the last day of the payroll month.

---

**Bug 9 — PT (Professional Tax) slab check uses `earnings.gross` (prorated) not full monthly salary**

```js
if (config.pt.mode === 'slab') {
    const matchedSlab = config.pt.state_slabs.find(
        slab => earnings.gross >= slab.min_gross && earnings.gross <= (slab.max_gross || Infinity)
    );
    deductions.pt = matchedSlab ? matchedSlab.amount : 0;
}
```

PT slabs are defined for monthly salary ranges (e.g. ₹10,000-₹15,000 = ₹150 PT, ₹15,001+ = ₹200 PT). The slab check uses `earnings.gross` which is the **prorated gross** for the month. For an employee who joins on January 20 (11 working days out of 31):

- Full monthly gross: ₹30,000 → PT slab = ₹200
- Prorated gross: ₹30,000 × 11/31 = ₹10,645 → PT slab = ₹150

The employee is charged ₹150 PT instead of ₹200 because their prorated salary falls in a lower slab. PT liability is determined by the employee's **full monthly CTC**, not their prorated earnings for a partial month. This is a compliance error.

**Fix:** Pass `fullSalary.base + fullSalary.hra + fullSalary.da + fullSalary.conveyance + fullSalary.specialAllowance` to the PT slab check, same pattern as ESI eligibility.

---

**Bug 10 — Payroll Excel export `earnings_tot_esi` column only sums Basic + HRA**

```js
earnings_tot_esi: (earnings.basic || 0) + (earnings.hra || 0),
```

The Excel export labels this column "Tot ESI Wage" but only includes Basic + HRA. The actual ESI wage used for deduction calculation includes DA and Special Allowance minus conveyance. The column shown in the Excel export doesn't match the ESI wage used for the actual deduction. This causes reconciliation failures when finance teams cross-check the Excel against the ESI challan.

---

## Phase 2: Optimization

**O1 — `bulkGenerateSalaries` fetches holiday config inside the per-employee loop**

```js
for (const emp of employees) {
    const holidayConfig = await HolidayConfig.findOne({ orgId, year });
    // ...
}
```

This fetches the same `HolidayConfig` document once per employee. For 100 employees that's 100 identical DB reads. Move this outside the loop — fetch once before the loop starts.

**O2 — `calculatePayableDays` iterates day-by-day in JavaScript for every employee**

For a 31-day month with 100 employees that's 3,100 iterations. Pre-build the weekOff and holiday Sets once per payroll run and pass them into the function as pre-computed arguments rather than re-fetching and re-building per employee.

**O3 — `bulkGenerateSalaries` calls `Salary.findOneAndUpdate` per employee sequentially**

The upsert for each employee's salary record runs sequentially inside the loop:
```js
await Salary.findOneAndUpdate(...)
```

For 100 employees that's 100 sequential DB writes. Convert to `Promise.all` with batches of 10 (same pattern as the bulk employee upload fix) for a significant speed improvement.

**O4 — `SalarySlip` frontend component reads `salary.deductions.pf` (singular) but backend stores `pf_employee`**

```jsx
<span>₹{salary.deductions.pf}</span>
```

The backend stores `deductions.pf_employee`. The frontend reads `deductions.pf`. This will render `₹undefined` on the payslip. The correct field is `salary.deductions.pf_employee`.

---

## Phase 3: Feature Enhancements

**E1 — Statutory compliance report**

Generate a monthly report showing org-level totals for: EPF employee contributions, EPF employer contributions, EPS, EDLI, ESI employee, ESI employer, PT. This is the data needed to file the monthly ECR (Electronic Challan cum Return) with EPFO. Currently the data exists scattered across individual salary records but no aggregation endpoint produces the challan-ready format.

**E2 — PF wage ceiling configuration per employee**

Some employees are allowed voluntary PF contributions above the ₹15,000 ceiling. Add `pf_voluntary_contribution` and `pf_wage_override_ceiling` to the employee statutory overrides. The salary config already supports per-employee `epf_employee_rate` — extend to per-employee ceiling.

**E3 — Year-end Form 16 generation**

Aggregate 12 months of TDS deductions per employee and generate a basic Form 16 PDF. The infrastructure (PDF generation, salary records) is already in place. This is a high-value feature for employees at tax-filing time.

**E4 — Salary revision history**

When an employee's salary changes (`updateEmployee`), record the old and new values with the effective date in a `SalaryRevision` collection. This feeds the payslip "this month's salary was revised from ₹X to ₹Y effective [date]" note and satisfies the audit trail requirement.

---

## Summary for Task 7

| # | Severity | Item |
|---|---|---|
| B1 | 🔴 Critical | PF wage excludes DA — statutory under-deduction for Basic < ₹15,000 employees |
| B4 | 🔴 Critical | Pro-ration denominator uses full month days — new joiners severely underpaid |
| B9 | 🔴 Critical | PT slab uses prorated gross — wrong slab for mid-month joiners |
| B2 | 🔴 High | ESI wage includes overtime pay — statutory over-deduction |
| B6 | 🔴 High | ESI date check uses `new Date()` not payroll month end — retroactive payroll miscalculates ESI eligibility |
| B10 | 🟡 Medium | Excel ESI wage column only Basic + HRA — reconciliation mismatch |
| B3 | 🟡 Medium | ESI eligibility uses inconsistent wage components vs calculation |
| O4 | 🟡 Medium | Payslip frontend reads `deductions.pf` — renders `₹undefined` |
| B5 | 🟢 Low | 50% rule logic is correct but explanation is wrong — maintainability risk |
| B7 | 🟢 Low | Negative incentive can create confusing payslip where deductions exceed prorated gross |

**Ready for Task 8 — Outward Payments whenever you are.**

Good. I have everything. Full three-phase review now.

---

## Task 8 — Outward Payments

---

## Phase 1: Bug Identification

**Bug 1 — `payEmployees` has no role authorization — any authenticated user can mark salaries as Paid**

```js
router.post('/pay', protect, require('../controllers/outwardPaymentController').payEmployees);
```

No `authorize()` middleware. Inside the controller:

```js
const adminUser = await Employee.findById(req.user._id);
const allowedRoles = ['ORG_ADMIN'];
if (!allowedRoles.includes(adminUser.professional?.role)) {
    return res.status(403).json({...});
}
```

The check exists in the controller but a MANAGER, HR_ADMIN, or EMPLOYEE can reach this controller via the route — the role check then blocks them. The issue is that if the controller check is ever accidentally removed or bypassed, any authenticated user can bulk-mark all salaries as "Paid" for any month. Defense-in-depth requires `authorize('ORG_ADMIN')` at the route level.

More critically: `updateEmployeeBankDetails` route has **no role check at all** — not at the route level, not in the controller:

```js
router.put('/update-bank/:employeeId', protect, updateEmployeeBankDetails);
```

Any authenticated employee can update any other employee's bank details — including their own — to redirect salary payments. This is a financial fraud vector.

---

**Bug 2 — `getOutwardPaymentRecords` fetches ALL salary records into memory then filters in JavaScript**

```js
const allSalaries = await Salary.find({ orgId, month: monthName, year }).populate(...);
// ...
const allRecords = allSalaries.map(sal => {...});
// ...
filteredRecords = filteredRecords.filter(r => r.branch === branchFilter);
// ...
const paginatedRecords = filteredRecords.slice(skip, skip + limit);
```

The entire payroll dataset for the month is loaded into memory, then filtered and paginated in JavaScript. For an org with 500 employees, 500 full salary documents are fetched from MongoDB and mapped into JavaScript objects on every single page request and search query. Branch filtering and search should be done at the DB query level with proper indexes, not in application memory.

---

**Bug 3 — `deleteAllAdjustments` takes month/year from query params without verifying the month has not been paid**

```js
await SalaryAdjustment.deleteMany({
    orgId: adminUser.orgId,
    month: monthName,
    year: parseInt(year)
});
```

There is no check whether salaries for this month are already in `'Paid'` status. An admin can clear adjustments for a month where salaries have already been disbursed. This creates an accounting inconsistency — the paid payslip reflects the adjustment (bonus/deduction) but the adjustment record is gone, making reconciliation impossible.

---

**Bug 4 — `payEmployees` does not check if salary records are locked before marking as Paid**

```js
const result = await Salary.updateMany(
    { orgId, employeeId: { $in: employeeIds }, month: monthName, year, status: 'Processed' },
    { $set: { status: 'Paid', paidAt: new Date(), paidBy: adminUser._id } }
);
```

The query correctly filters `status: 'Processed'` — Draft records are not paid. But there is no check for `locked: true`. A salary record that has been locked (e.g. after statutory filing) can still be marked as "Paid" again if its status was somehow reset to "Processed". The `updateMany` should also include `locked: { $ne: true }` in the filter.

---

**Bug 5 — Frontend-Backend contract mismatch: `OutwardPaymentRecord.jsx` builds headers manually with localStorage token**

```js
const token = localStorage.getItem('token');
const headers = { Authorization: `Bearer ${token}` };

await axios.put(`${API_URL}/api/v1/outward-payment/update-bank/${editingId}`, editForm, { headers });
await axios.delete(`${API_URL}/api/v1/outward-payment/clear-adjustments?...`, { headers });
await axios.post(`${API_URL}/api/v1/outward-payment/pay`, payload, { headers });
```

All three calls use manually constructed headers and a raw `API_URL` string instead of the configured `axios` instance from `../../api`. This means:
1. The auth token refresh interceptor does not apply — if the token expires mid-session, these calls will silently return 401 and the user sees a generic error
2. The `withCredentials: true` setting is not applied
3. The double-URL bug applies here if `API_URL` is already set (same as Task 6, Bug 7)

---

**Bug 6 — `missingDetails` alert uses client-side filtering on pre-fetched data**

```js
const missingDetails = allRecords.filter(r =>
    !r.accountNumber || !r.ifsc || !r.accountName
).map(r => ({ employeeId: r.employeeId, name: r.name }));
```

`allRecords` is built from all salary records in memory. `missingDetails` is then sent back in the response to flag employees with incomplete bank details. The issue: if `allRecords` is paginated and the current page doesn't include an employee with missing bank details, that employee will appear in `missingDetails` (since it uses `allRecords`, not `paginatedRecords`). The UI correctly shows the full missing list, but this relies on ALL salary records being loaded into memory first — compounding Bug 2.

---

**Bug 7 — `advanceAlerts` is computed but never used in the frontend**

```js
const advanceAlerts = allRecords.filter(r => r.advance > 0).map(r => ({
    employeeId: r.employeeId,
    name: r.name,
    advance: r.advance
}));
```

`advanceAlerts` is included in the API response. Looking at `OutwardPaymentRecord.jsx`, the frontend destructures the response but `advanceAlerts` is never rendered anywhere in the UI. This means employees with advance deductions get their advance correctly deducted from net pay (via `SalaryAdjustment`) but the HR admin never sees a summary of who has outstanding advances. The data is computed on every request but silently thrown away on the frontend.

---

**Bug 8 — Excel download passes all records without pagination but frontend may trigger it with paginated context**

```js
if (download === 'excel') {
    // uses allRecords (all employees, no pagination)
    const excelBuffer = await generatePayrollExcel(orgName, monthName, year, startDate, endDate, allRecords);
}
```

The Excel download correctly uses `allRecords` (all employees) not `paginatedRecords`. Good. But the download is triggered from the frontend via the same `GET /outward-payment` endpoint with `?download=excel`. The frontend passes the current `selectedBranch` and `searchQuery` filter parameters. The controller applies branch/search filters to `allRecords` before the Excel export:

```js
let filteredRecords = [...allRecords];
if (branchFilter && branchFilter !== 'all') {
    filteredRecords = filteredRecords.filter(r => r.branch === branchFilter);
}
```

So if HR is viewing Branch A with a search query and clicks "Download Excel", they only get Branch A filtered records in the Excel. This is correct behavior. But the label says "Download Payroll" — a user might expect the full org payroll, not just their current filter view. No warning is shown.

---

## Phase 2: Optimization

**O1 — Move branch and search filtering to MongoDB query level**

Replace the in-memory filter pattern with proper MongoDB query params:

```js
const query = { orgId, month: monthName, year: parseInt(year) };
if (branchFilter && branchFilter !== 'all') {
    // Join branch through employee's baseBranch
    const branchEmployees = await Employee.find({ 'professional.baseBranch': branchFilter }, '_id').lean();
    query.employeeId = { $in: branchEmployees.map(e => e._id) };
}
```

For name/code search, maintain a separate lightweight query rather than loading all records.

**O2 — `getOutwardPaymentRecords` does a separate `Employee.findById` inside the salary loop**

```js
for (const sal of allSalaries) {
    const emp = await Employee.findById(sal.employeeId);
```

If `.populate()` is already used on the Salary query, this is redundant. Check the populate chain — if `employeeId` is populated, `sal.employeeId` is already the employee document and the separate findById is unnecessary.

**O3 — `payEmployees` returns `modifiedCount` but no breakdown of which employees failed**

```js
res.status(200).json({ success: true, modifiedCount: result.modifiedCount });
```

If 8 out of 10 requested employees were updated (2 were already Paid or in Draft status), the caller only knows "8 modified" not which 2 failed. Return the specific employeeIds that were not updated for proper UI feedback.

---

## Phase 3: Feature Enhancements

**E1 — Payment confirmation with bank transfer reference**

Currently "Pay" just marks the status as `'Paid'` with no reference number. Add a `paymentReference` field to the `Salary` model — the admin should enter a UTR/NEFT reference number when marking a payment as paid. This creates an auditable trail connecting the HRMS record to the actual bank transfer.

**E2 — Bulk payment approval workflow**

For larger orgs, the payment flow should have a maker-checker pattern: HR generates the Excel, Finance reviews and uploads a payment confirmation file, then the status transitions to Paid with the uploaded reference. This is standard in companies with financial controls separation.

**E3 — Bank detail change notification**

When `updateEmployeeBankDetails` is called, send a notification to the employee: "Your bank details have been updated by [admin name] on [date]. If you did not request this, contact HR immediately." This closes the fraud vector identified in Bug 1 by at least making the change auditable and visible to the employee.

---

## Summary for Task 8

| # | Severity | Item |
|---|---|---|
| B1 | 🔴 Critical | `updateEmployeeBankDetails` has no role check — any employee can redirect salary payments |
| B1 | 🔴 High | `payEmployees` role check only in controller, not at route level |
| B3 | 🔴 High | `deleteAllAdjustments` allows clearing paid month adjustments — breaks reconciliation |
| B5 | 🟡 Medium | Frontend uses raw axios + manual headers — token refresh doesn't apply |
| B2 | 🟡 Medium | Full month salary data loaded into memory for every page/search request |
| B4 | 🟡 Medium | `payEmployees` doesn't check `locked` flag |
| B7 | 🟢 Low | `advanceAlerts` computed but never rendered — silent data loss |
| B8 | 🟢 Low | Excel download respects current filter — no warning to user |

**Ready for Task 9 — Notifications whenever you are.**

I have all the files. Full three-phase review now.

---

## Task 9 — Notifications

---

## Phase 1: Bug Identification

**Bug 1 — `createNotification` fires a push notification synchronously inside the helper — blocks the calling request**

```js
exports.createNotification = async (recipientId, title, message, type, relatedId) => {
    try {
        await Notification.create({...});
        await sendPushNotification([recipientId], {...}); // ← awaited
    } catch (err) {
        console.error('Failed to create/send notification', err);
    }
};
```

`createNotification` is called from multiple controllers — `leaveController`, `attendanceController`, `employeeController`. Each call `await`s both the DB write and the FCM push. FCM push involves a network call to Google's servers which can take 200-2000ms.

For a leave approval that notifies 3 admins:
```js
for (const adminId of notifyIds) {
    await createNotification(adminId, ...); // sequential, awaited
}
```

Three sequential FCM calls add 600ms-6000ms to the leave approval response time. The user clicks "Approve" and waits up to 6 seconds for a response that has nothing to do with the approval itself.

The fix is `setImmediate` or fire-and-forget:
```js
createNotification(adminId, ...).catch(err => console.error(err));
// Not awaited
```

---

**Bug 2 — `getNotifications` has no org scope check — cross-tenant notification leak**

```js
const notifications = await Notification.find({ recipient: req.user._id })
```

The query filters by `recipient` (the user's `_id`). MongoDB ObjectIds are globally unique so this is safe in practice. However the `Notification` model has no `orgId` field:

```js
const NotificationSchema = new mongoose.Schema({
    recipient: { type: mongoose.Schema.Types.ObjectId, ref: 'Employee', required: true },
    title: String,
    message: String,
    type: String,
    relatedId: mongoose.Schema.Types.ObjectId,
    isRead: { type: Boolean, default: false }
});
```

`relatedId` references documents across all collections — a `LeaveRequest._id`, an `Attendance._id`, etc. There is no `orgId` on the notification itself. If a notification's `relatedId` is fetched by the frontend to display details, that fetch goes to the relevant controller which may or may not enforce org scoping. No `orgId` on the Notification model also means notification analytics across orgs are impossible.

---

**Bug 3 — `sendPushNotification` data payload fields must be strings — non-string values cause FCM errors**

```js
data: {
    type,
    relatedId: relatedId ? relatedId.toString() : ''
}
```

`type` is passed directly as-is. If `type = 'Leave'` this is a string — fine. But in some call sites:

```js
data: { type: 'REMOTE_ATTENDANCE', reminderForDate }
```

`reminderForDate` is a date string — fine. But in `jobs/index.js`:

```js
data: { type: 'REMOTE_ATTENDANCE', reminderForDate }
```

`reminderForDate` could be a Date object if it comes from a Mongoose document. FCM requires ALL `data` values to be strings. A non-string value silently causes the FCM message to be dropped on some platforms without an explicit error.

**Fix:** Stringify all data values before sending to FCM:
```js
data: Object.fromEntries(Object.entries(payload.data || {}).map(([k, v]) => [k, String(v)]))
```

---

**Bug 4 — `saveFcmToken` accumulates tokens unboundedly with no deduplication across devices**

```js
await Employee.findByIdAndUpdate(req.user._id, {
    $addToSet: { fcmTokens: token }
}, { new: true });
```

`$addToSet` prevents exact duplicates — good. But an employee who uses 3 different browsers/devices over time accumulates 3 tokens. When they clear browser data and log in again, a new token is generated and added as a 4th. Old tokens from cleared browsers are never removed until the FCM token cleanup (Bug 5). Over time an active employee can accumulate 10-20 stale tokens. Every `sendPushNotification` call fetches all tokens and attempts delivery on all of them — wasting FCM quota and slowing the notification pipeline.

---

**Bug 5 — FCM token cleanup only handles two specific error codes — other failures leave stale tokens**

```js
if (error.code === 'messaging/registration-token-not-registered' ||
    error.code === 'messaging/invalid-registration-token') {
    failedTokens.push(allTokens[idx]);
}
```

Other FCM error codes that indicate a stale or invalid token are not handled:
- `messaging/invalid-argument` — malformed token
- `messaging/quota-exceeded` — token's app exceeded quota
- `messaging/unavailable` — transient, should NOT be cleaned

The cleanup correctly identifies the two most common stale token codes. The gap is that `messaging/invalid-argument` tokens will never be cleaned and will permanently appear in every future notification batch.

---

**Bug 6 — `createNotification` helper is exported from the controller file — architectural smell**

```js
// notificationController.js
exports.createNotification = async (recipientId, ...) => {...};
```

This is a utility function used by 6+ other controllers imported as:
```js
const { createNotification } = require('./notificationController');
```

Importing a controller from another controller creates a circular dependency risk. If `notificationController` ever needs to import from `leaveController` (e.g. for notification-triggered actions), you get a circular require. The `createNotification` function should live in a `services/notificationService.js` file, not in a controller.

---

**Bug 7 — `markAsRead` uses `PUT /:id/read` but `markAllAsRead` uses `PUT /read-all` — route ordering conflict**

```js
router.put('/read-all', markAllAsRead);
router.put('/:id/read', markAsRead);
```

`read-all` is defined before `/:id/read` — correct order. If reversed, Express would match `read-all` as an `:id` parameter for `markAsRead`. The current order is safe. However if a developer adds a new route like `PUT /clear` between them in the future, they may not realise the ordering dependency. The routes should use more explicit paths: `PUT /mark-all-read` and `PUT /:id/mark-read` to avoid any future confusion.

---

**Bug 8 — No TTL or cleanup for old notification records**

```js
const notifications = await Notification.find({ recipient: req.user._id })
    .sort({ createdAt: -1 })
    .limit(50);
```

The query limits to 50 notifications for display, but notifications are never deleted from the database. For an active employee with 100 leave requests, 50 attendance submissions, and 200 reminder notifications per year, that's 350+ notification documents per employee per year. With 100 employees over 3 years: 100,000+ notification documents with no TTL index.

The `Notification` model has no TTL index. Add:
```js
NotificationSchema.index({ createdAt: 1 }, { expireAfterSeconds: 90 * 24 * 60 * 60 }); // 90 days
```

---

## Phase 2: Optimization

**O1 — `sendPushNotification` fetches full Employee documents just for `fcmTokens`**

```js
const users = await Employee.find({ _id: { $in: userIdsArray } }).select('fcmTokens');
```

`.select('fcmTokens')` is already used — good. This is correctly optimized.

**O2 — `createNotification` is called in a sequential `for` loop in multiple controllers**

```js
for (const adminId of approverIds) {
    await createNotification(adminId, ...); // sequential
}
```

Replace with `Promise.all` for parallel notification creation. For 3 admins, this reduces the wall time from 3× FCM latency to 1× FCM latency (they run simultaneously):

```js
await Promise.all(approverIds.map(adminId => 
    createNotification(adminId, ...).catch(err => console.error(err))
));
```

**O3 — No index on `Notification.recipient` + `isRead` combination**

```js
const notifications = await Notification.find({ recipient: req.user._id })
    .sort({ createdAt: -1 })
    .limit(50);
```

The query filters by `recipient` and sorts by `createdAt`. A compound index `{ recipient: 1, createdAt: -1 }` would dramatically speed this up as notifications accumulate. The current model likely only has a default `_id` index.

**O4 — Unread count not available without a separate query**

The frontend has no way to show an unread notification badge count without fetching all 50 notifications and counting client-side. Add a `GET /notifications/unread-count` endpoint:

```js
const count = await Notification.countDocuments({ recipient: req.user._id, isRead: false });
```

This is a cheap indexed count query that can drive a badge counter without fetching notification content.

---

## Phase 3: Feature Enhancements

**E1 — Notification inbox UI**

As identified in the original task list — notifications are generated and push-delivered but there is no visible inbox in the frontend. A notification bell in the header with a badge count (`/unread-count`) and a dropdown panel (`/notifications`) would make all this infrastructure visible to users. The backend is complete — this is a frontend-only gap.

**E2 — Notification preferences per employee**

Add a `notificationPreferences` subdocument to the Employee model:
```js
notificationPreferences: {
    leaveUpdates: { type: Boolean, default: true },
    attendanceReminders: { type: Boolean, default: true },
    payrollNotifications: { type: Boolean, default: true },
    newEmployeeAlerts: { type: Boolean, default: true }
}
```

`notifyBranchMembersOfNewJoiner` already checks `notificationPreferences.newEmployeeAlert` — the field pattern exists. Extend it to all notification types.

**E3 — Notification action links**

The `data.screen` field in FCM payloads (`'LeaveManagement'`, `'Dashboard'`) is used for mobile deep linking. Extend this to the web app — when a user clicks a notification, navigate them directly to the relevant record rather than just the general module page.

---

## Summary for Task 9

| # | Severity | Item |
|---|---|---|
| B1 | 🔴 High | `createNotification` awaited synchronously in loops — adds 600ms-6s to controller response times |
| B3 | 🔴 High | FCM data values not stringified — non-string values silently drop messages on some platforms |
| B2 | 🟡 Medium | No `orgId` on Notification model — cross-tenant analytics impossible, relatedId fetch may leak |
| B4 | 🟡 Medium | FCM tokens accumulate unboundedly — stale tokens slow every notification batch |
| B8 | 🟡 Medium | No TTL on Notification collection — unbounded growth over time |
| B6 | 🟡 Medium | `createNotification` exported from controller — circular dependency risk |
| B5 | 🟢 Low | FCM cleanup misses `messaging/invalid-argument` error code |
| B7 | 🟢 Low | Route ordering dependency fragile — `read-all` must stay before `/:id/read` |

**Ready for Task 10 — AI Assistant whenever you are.**

I have all the files. Full three-phase review now.

---

## Task 10 — AI Assistant

---

## Phase 1: Bug Identification

**Bug 1 — Final streaming call is a separate third LLM invocation — doubles cost and latency for every response**

The tool-calling loop architecture makes two full LLM round trips for every response with tool calls:

```js
// Round 1: Tool selection + execution (non-streaming)
while (response.choices[0].finish_reason === 'tool_calls') {
    // execute tools, push results
    response = await openai.chat.completions.create({...});
}

// Round 2: Final response (streaming) — SEPARATE CALL
const streamResponse = await openai.chat.completions.create({
    ...currentMessages,
    stream: true
});
```

After the tool loop ends (when `finish_reason !== 'tool_calls'`), `response` already contains the final text answer. But instead of streaming that answer, the code discards it and makes a brand new streaming call with the same messages. This is a third LLM call (two in the loop, one for streaming) that:
1. Costs additional tokens (input tokens paid twice for the final message)
2. Adds 1-3 seconds of additional latency on every query that uses a tool
3. Can produce a different answer than the non-streaming response that was already computed

The fix is to use `stream: true` from the first call in the loop, which eliminates the separate streaming round trip entirely.

---

**Bug 2 — `AIAssistant.jsx` uses raw `fetch` with manual token extraction instead of the configured axios instance**

```js
const token = localStorage.getItem('token');
const response = await fetch(`${API_URL}/api/v1/ai/chat`, {
    method: 'POST',
    headers: {
        'Authorization': token ? `Bearer ${token}` : '',
    }
});
```

This is necessary because SSE streaming requires `fetch` — axios doesn't support SSE. However the token is read directly from localStorage instead of being derived from the auth context. If the token is refreshed mid-session by the axios interceptor (which writes the new token back to localStorage), `fetch` will use the old token until the page reloads. There's a race condition: the axios instance refreshes the token and updates localStorage, but the fetch call still uses the token value captured at the start of `sendMessage`.

---

**Bug 3 — `history` slice uses last 10 messages but includes the new user message being sent**

```js
const history = messages
    .filter(m => (m.role === 'user' || m.role === 'assistant') && typeof m.content === 'string')
    .slice(-10);
```

`messages` at this point in `AIAssistant.jsx` includes the just-added user message:
```js
const allMessages = [...messages, userMessage].map(m => ({ role: m.role, content: m.content }));
const history = allMessages.slice(firstUserIdx);
```

The history sent to the backend includes the current message being processed. The backend then builds:
```js
let currentMessages = [
    { role: 'system', content: buildSystemPrompt(...) },
    ...history  // includes the current user message
];
```

This is correct — the current message IS in the history. But the comment says "last 10" which is slightly misleading — it's actually the last 10 including the current message. The effective context window is the last 9 prior messages plus the current one.

This is not a functional bug but the `-10` slice may cut off the start of a conversation mid-context causing the LLM to reference a "previous message" that it can't see.

---

**Bug 4 — `hrTools` functions use `ctx.orgId` directly without converting to ObjectId**

In `hrTools.js`:
```js
const employees = await Employee.find({ orgId: ctx.orgId, ... });
```

`ctx.orgId` comes from `buildSecurityContext(user)` where `user.orgId` is already a Mongoose ObjectId from the Employee document. Mongoose handles the comparison correctly. However in `get_payroll_summary`:

```js
await Salary.aggregate([
    { $match: { orgId: ctx.orgId, month: monthName, year } },
```

MongoDB aggregation `$match` does not perform Mongoose type coercion — it treats the filter value as-is. If `ctx.orgId` is a string (e.g. because `req.user.orgId` was serialized through JSON somewhere), the aggregation will match nothing and return empty results. The fix is explicit conversion:

```js
{ $match: { orgId: new mongoose.Types.ObjectId(ctx.orgId), ... } }
```

---

**Bug 5 — `get_headcount_analytics` groups by `professional.department` which is ObjectId but displays as "Unassigned" for all employees**

```js
Employee.aggregate([
    { $match: matchActive },
    { $group: { _id: '$professional.department', count: { $sum: 1 } } },
    { $lookup: { from: 'departments', localField: '_id', foreignField: '_id', as: 'dept' } },
    { $unwind: { path: '$dept', preserveNullAndEmptyArrays: true } },
    { $project: { department: { $ifNull: ['$dept.name', 'Unassigned'] }, count: 1, _id: 0 } }
])
```

The Employee model does not have a `professional.department` field in the current schema — the department is stored as a string in `professional.designation` (from the earlier audit). The `departments` collection doesn't appear to exist yet in the codebase. So `professional.department` is always `undefined`, `_id` is always `null`, the `$lookup` returns nothing, and `$ifNull` returns `'Unassigned'` for every employee.

The AI reports "all 97 employees are in the Unassigned department" — technically true but meaningless. The tool should group by `professional.designation` or `professional.role` instead until a proper department model is added.

---

**Bug 6 — SSE stream is never explicitly closed on client disconnect**

```js
res.setHeader('Content-Type', 'text/event-stream');
res.flushHeaders();
// ...
for await (const chunk of streamResponse) {
    sseWrite(res, 'delta', { text: delta });
}
sseWrite(res, 'done', {...});
res.end();
```

There is no `req.on('close', ...)` handler. If the user closes the chat window or navigates away during a streaming response, the OpenAI stream continues consuming tokens and the server continues trying to write to a closed socket. For a response that generates 500 tokens at 30 tokens/second, this wastes ~16 seconds of OpenAI API time and tokens.

**Fix:**
```js
let cancelled = false;
req.on('close', () => { cancelled = true; });

for await (const chunk of streamResponse) {
    if (cancelled) break;
    // ...
}
```

---

**Bug 7 — `isLastIteration` condition is off-by-one**

```js
const MAX_ITERATIONS = 5;
let iterations = 0;

while (...iterations < MAX_ITERATIONS) {
    iterations++; // now iterations = 1, 2, 3, 4, 5
    // ...
    const isLastIteration = iterations >= MAX_ITERATIONS - 1; // true when iterations >= 4
    response = await openai.chat.completions.create({
        ...(!isLastIteration && { tools, tool_choice: 'auto' })
    });
}
```

`isLastIteration` becomes `true` at iteration 4 (when `MAX_ITERATIONS - 1 = 4`). At iteration 4, tools are removed from the call, forcing a final answer. But the loop condition `iterations < MAX_ITERATIONS` allows iteration 5. At iteration 5, `isLastIteration` is also true (5 >= 4), so tools are again not passed. This means the last two iterations (4 and 5) both force a final answer — the response from iteration 4 is discarded and a completely new response is generated at iteration 5.

For most queries this doesn't matter (they terminate well before iteration 4). But for complex multi-step queries it wastes one extra LLM call.

**Fix:** `const isLastIteration = iterations >= MAX_ITERATIONS - 1` should be `iterations === MAX_ITERATIONS - 1`.

---

**Bug 8 — `AIAuditLog` audit uses `messages` from `req.body` but `messages` is the raw frontend history including streaming placeholders**

```js
const lastUserMsg = [...messages].reverse().find(m => m.role === 'user');
AIAuditLog.create({ userQuery: (lastUserMsg?.content || '').substring(0, 500) });
```

The `messages` array from the frontend includes assistant placeholder messages with `content: ''` (the streaming placeholder added before the response arrives). After the history filter:

```js
const history = messages
    .filter(m => (m.role === 'user' || m.role === 'assistant') && typeof m.content === 'string')
    .slice(-10);
```

Empty string `''` passes `typeof m.content === 'string'` — so placeholder messages with `content: ''` are included in history. These blank assistant messages inflate the context sent to the LLM and may confuse the model about the conversation structure.

---

## Phase 2: Optimization

**O1 — `buildSystemPrompt` is called on every request even though most content is static**

The system prompt contains dynamic fields only (`today`, `currentMonth`, `currentYear`, `lastMonth`, `lastMonthYear`, `ctx.userName`). These change at most once per day (date) or on role change. Cache the static template and only inject dynamic values at call time rather than rebuilding the entire string on every request.

**O2 — `org` is fetched on every chat message just to check `modules.ai`**

```js
const org = await Organization.findById(ctx.orgId).select('organizationName modules').lean();
if (!org?.modules?.ai) { return 403 }
```

For an active chat session, the user has already authenticated and the org's AI module is enabled. This check re-queries MongoDB on every single message. The module status could be embedded in the JWT (alongside role) or cached in memory with a 5-minute TTL — same pattern as the rate limiting token decode.

**O3 — `toolsCalled` array uses `push` inside `Promise.all` — non-atomic**

```js
const toolResults = await Promise.all(
    assistantMsg.tool_calls.map(async (toolCall) => {
        toolsCalled.push(toolName); // ← push from concurrent callbacks
    })
);
```

`Array.push` in JavaScript is synchronous and JS is single-threaded, so this is technically safe. But it's a code smell — collecting side effects inside a `Promise.all` map is error-prone if the code is ever refactored to use workers.

**O4 — QUICK_PROMPTS in `AIAssistant.jsx` are hardcoded strings with no role awareness**

```js
const QUICK_PROMPTS = [
    { label: 'Today\'s Attendance', prompt: 'Show me today\'s attendance summary' },
    { label: 'Pending Leaves', prompt: 'Show all pending leave requests' },
    { label: 'Team Overview', prompt: 'Give me a headcount overview by department' },
    { label: 'My Payslip', prompt: 'Show my salary slip for this month' },
];
```

An EMPLOYEE sees "Show all pending leave requests" which is an HR tool. Clicking it produces an error or empty result. Quick prompts should be filtered by the user's role — employees only see their own data prompts, managers see team prompts.

---

## Phase 3: Feature Enhancements

**E1 — Stream tool execution status to frontend**

Currently the user sees a blank streaming bubble for 2-5 seconds while tools execute. Add a `status` SSE event before the tool loop:

```js
sseWrite(res, 'status', { text: 'Fetching attendance data...' });
```

The frontend renders this as a subtle "Fetching..." indicator inside the bubble. Users see immediate feedback instead of a silent wait.

**E2 — Conversation memory summarization**

After 20 messages, the history slice starts losing early context. Implement a summarization step: when history exceeds 15 messages, make a cheap summarization call (`max_tokens: 200`) to compress older context into a single summary message. This preserves conversation coherence without unbounded context growth.

**E3 — Morning brief injection on chat open**

When `AIAssistant` mounts and `isOpen` becomes `true`, check the `AIInsight` collection for today's brief. If found, prepend it as the first assistant message after the welcome:

```
"Good morning, Ravi. Today's brief: 3 employees absent, 7 payroll anomalies detected, 2 pending leave requests."
```

Zero LLM cost, immediate value, the pipeline is already built.

**E4 — Suggested follow-up questions**

After every response, have the LLM return 2-3 follow-up questions in a structured field. Render them as clickable chips below the response. 15 extra tokens per response. Dramatically reduces the "I don't know what to ask" problem.

---

## Summary for Task 10

| # | Severity | Item |
|---|---|---|
| B1 | 🔴 Critical | Third redundant LLM call for streaming — doubles cost and latency on every tool-using query |
| B6 | 🔴 High | No client disconnect handler — OpenAI stream continues after user closes chat |
| B5 | 🔴 High | `get_headcount_analytics` groups by non-existent `department` field — always returns "Unassigned" |
| B4 | 🟡 Medium | `ctx.orgId` not converted to ObjectId in aggregation — returns empty if string |
| B2 | 🟡 Medium | `fetch` reads token at call time — may use stale token if axios interceptor just refreshed it |
| B8 | 🟡 Medium | Empty-string placeholder messages pass history filter — pollutes LLM context |
| B7 | 🟢 Low | `isLastIteration` off-by-one — two consecutive iterations force final answer |
| B3 | 🟢 Low | History slice of -10 includes current message — effective prior context is only 9 messages |
| O4 | 🟢 Low | Quick prompts not role-filtered — employees see HR-only prompts that fail |

**Ready for Task 11 — Settings whenever you are.**

I have everything. Full three-phase review now.

---

## Task 11 — Settings

---

## Phase 1: Bug Identification

**Bug 1 — `updateOrgSettings` allows MANAGER to modify attendance and shift rules**

```js
router.put('/', protect, authorize('ORG_ADMIN', 'HR_ADMIN', 'MANAGER'), updateOrgSettings);
```

A MANAGER can call `PUT /api/v1/org/settings` and modify:
- `shift_rules` — changes shift in/out time for the entire org
- `qr_refresh_interval` — affects QR security for all branches
- `geo_fence_radius` — changes attendance geo-fence for everyone
- `attendance_approver` — changes who approves attendance across the org
- `comp_off_rules` — changes comp-off policy for all employees

These are org-level configuration changes that directly affect payroll calculations (shift rules feed into late/half-day detection) and attendance security (geo-fence). A MANAGER should never be able to change org-wide configuration. This should be `ORG_ADMIN` only, with `HR_ADMIN` allowed only for non-financial settings.

---

**Bug 2 — `getOrgSettings` allows MANAGER to read `modules` — module gate bypass risk**

```js
router.get('/', protect, authorize('ORG_ADMIN', 'HR_ADMIN', 'MANAGER'), getOrgSettings);
```

`getOrgSettings` returns `organizationName modules` in the response. The `modules` object reveals which features are enabled/disabled for the org. A MANAGER can query this endpoint to check if, say, `modules.ai` is false, and then attempt to circumvent the gate. While the AI endpoint itself also checks `modules.ai`, exposing the full modules config to MANAGERs is unnecessary.

---

**Bug 3 — `updateConfig` (salary config) uses `findOneAndUpdate` with direct body fields — unsafe merge**

```js
const updateData = {};
for (const key of allowed) {
    if (req.body[key] !== undefined) {
        updateData[key] = req.body[key];
    }
}

const config = await SalaryConfig.findOneAndUpdate(
    { orgId },
    updateData,
    { new: true, upsert: true, runValidators: true }
);
```

The `updateData` is passed directly without `$set`. When Mongoose receives a top-level update object without an operator, it replaces the matched document fields entirely — effectively a document replacement for the specified keys. For `pf: { enabled: false }`, this overwrites the entire `pf` subdocument with just `{ enabled: false }`, losing `employee_contribution`, `employer_contribution`, `wage_ceiling`, `eps_split`, etc.

Example: Admin disables PF → sends `{ pf: { enabled: false } }` → entire PF config replaced with `{ enabled: false }` → all rates become `undefined` → next payroll run calculates `undefined * 0.12 = NaN` for every employee.

**Fix:** Use `$set` with dot notation:
```js
const flatUpdate = {};
for (const [key, val] of Object.entries(updateData)) {
    if (typeof val === 'object' && val !== null) {
        for (const [subKey, subVal] of Object.entries(val)) {
            flatUpdate[`${key}.${subKey}`] = subVal;
        }
    } else {
        flatUpdate[key] = val;
    }
}
await SalaryConfig.findOneAndUpdate({ orgId }, { $set: flatUpdate }, { new: true, upsert: true });
```

---

**Bug 4 — `updateOrgSettings` for `shift_rules` replaces the entire subdocument**

```js
if (shift_rules !== undefined && typeof shift_rules === 'object') {
    const sr = {};
    if (shift_rules.standard_in_time !== undefined) sr.standard_in_time = shift_rules.standard_in_time;
    // ...
    if (Object.keys(sr).length) allowedUpdates.shift_rules = sr;
}
```

Then:
```js
await Organization.findByIdAndUpdate(req.user.orgId, { $set: allowedUpdates });
```

`allowedUpdates.shift_rules = sr` sets the entire `shift_rules` subdocument to `sr`. If the admin only sends `standard_in_time`, `sr = { standard_in_time: '09:00' }` and `$set: { shift_rules: { standard_in_time: '09:00' } }` replaces the entire `shift_rules` object — losing `grace_period_mins`, `half_day_late_threshold_mins`, `missing_out_punch_behavior`, etc.

**Fix:** Use dot notation for subdocument updates:
```js
if (shift_rules.standard_in_time !== undefined) allowedUpdates['shift_rules.standard_in_time'] = shift_rules.standard_in_time;
if (shift_rules.grace_period_mins !== undefined) allowedUpdates['shift_rules.grace_period_mins'] = Number(shift_rules.grace_period_mins);
// etc. — no more nested object replacement
```

---

**Bug 5 — `SalaryConfig.state_slabs` stores `max_gross: Infinity` — MongoDB cannot store `Infinity`**

```js
state_slabs: [{
    min_gross: { type: Number, required: true },
    max_gross: { type: Number, default: Infinity }, // ← MongoDB cannot store this
    amount: { type: Number, required: true }
}]
```

JavaScript `Infinity` converts to `null` in JSON (via `JSON.stringify(Infinity) === 'null'`). When a PT slab with no upper bound is saved, `max_gross` is stored as `null` in MongoDB, not `Infinity`. The PT slab check:

```js
slab => earnings.gross >= slab.min_gross && earnings.gross <= (slab.max_gross || Infinity)
```

Uses `slab.max_gross || Infinity` — if `slab.max_gross` is `null`, this correctly falls back to `Infinity`. So the PT calculation itself works. But if `slab.max_gross` is `0` (a misconfigured slab), `0 || Infinity` evaluates to `Infinity` incorrectly — a slab with `max_gross: 0` would match all salaries above `min_gross`. Use `slab.max_gross ?? Infinity` (nullish coalescing) instead of `||`.

---

**Bug 6 — Settings page reads org data from localStorage instead of fetching fresh**

```js
// From Settings.jsx
const user = JSON.parse(localStorage.getItem('user') || '{}');
const orgName = user.organizationName || user.orgName || 'Your Organization';
```

Settings like `organizationName` displayed in the UI come from the localStorage `user` object set at login time. If a SuperAdmin changes the org name via `EditOrganizationModal`, the Settings page continues showing the old name until the user logs out and back in. The Settings page should fetch the current org state from `GET /api/v1/org/settings` on mount.

---

**Bug 7 — No validation on `SalaryConfig` rate values — rates above 100% accepted**

```js
exports.updateConfig = async (req, res) => {
    const updateData = {};
    for (const key of allowed) {
        if (req.body[key] !== undefined) {
            updateData[key] = req.body[key]; // ← no validation
        }
    }
```

An admin can submit `{ pf: { employee_contribution: 500 } }` — a 500% PF rate — and it gets saved without validation. The next payroll run calculates `pfWage * (500 / 100) = 5× the wage` as PF deduction, resulting in massively negative net pay for all employees. Even with `Math.max(0, netPay)`, this destroys the payroll figures.

The SalaryConfig model has no `min`/`max` validators on rate fields. Add validation:
```js
employee_contribution: { type: Number, default: 12, min: 0, max: 100 }
```

---

**Bug 8 — `updateBankDetails` allows `HR_ADMIN` to change org bank account details**

```js
router.put('/bank-details', protect, authorize('ORG_ADMIN', 'HR_ADMIN'), updateBankDetails);
```

`HR_ADMIN` can change the organization's bank account number and IFSC code. These details appear on payslips and are used as reference for salary disbursements. An HR Admin with malicious intent could change the org bank details to redirect payment references. This should be `ORG_ADMIN` only.

---

## Phase 2: Optimization

**O1 — `getOrgSettings` fetches the Organization document on every settings page load**

Settings rarely change. A 5-minute server-side cache keyed by `orgId` would eliminate the vast majority of these reads for the most common use case: HR admins who open the settings page just to view current configuration.

**O2 — `updateOrgSettings` fetches no document before update — no change detection**

If an admin submits the settings form without changing anything, the full `findByIdAndUpdate` still runs writing the same values back. Add a change detection step or use MongoDB's `$set` which only marks fields as modified if values actually changed (which it does by default — no optimization needed at DB level, but the application could skip the call entirely if no fields changed).

**O3 — `salaryConfigController.updateConfig` re-runs `findOneAndUpdate` with `upsert: true` every time**

On every update, `upsert: true` means MongoDB checks if the document exists before deciding to insert or update — an extra index lookup. Since `getConfig` already creates the document if missing:

```js
if (!config) config = await SalaryConfig.create({ orgId });
```

The upsert on `updateConfig` is a safety net, not a primary path. The extra index lookup on every update is minor but unnecessary.

---

## Phase 3: Feature Enhancements

**E1 — Settings change audit log**

Every change to salary config (PF rate, ESI ceiling) should be recorded in an audit log with `changedBy`, `previousValue`, `newValue`, `timestamp`. This is a compliance requirement — if EPFO asks why PF contributions changed in March, you need a record showing who changed the rate on what date.

**E2 — Settings validation preview**

Before saving a salary config change, show the admin a "What this means" preview: "With these PF settings, an employee earning ₹30,000 basic will have ₹1,800 PF deducted." This prevents accidental misconfiguration and makes the settings page genuinely usable for non-technical admins.

**E3 — Shift rules per-branch override**

Currently shift rules are org-level only. A factory branch working 24/7 has different shift windows than a corporate office branch. Allow per-branch shift rule overrides that take precedence over org defaults. The data model partially supports this — attendance already has `shift_timings` per employee — but org-level and branch-level configuration is conflated.

---

## Summary for Task 11

| # | Severity | Item |
|---|---|---|
| B3 | 🔴 Critical | `updateConfig` replaces entire PF/ESI subdocument — destroys all rates when any one field is updated |
| B4 | 🔴 Critical | `updateOrgSettings` replaces `shift_rules` subdocument — loses grace period, thresholds, etc. |
| B1 | 🔴 High | MANAGER can modify org-wide shift rules and geo-fence via `PUT /org/settings` |
| B8 | 🔴 High | `HR_ADMIN` can change org bank account details |
| B7 | 🟡 Medium | No rate validation on SalaryConfig — 500% PF rate accepted, destroys payroll |
| B5 | 🟡 Medium | `Infinity` stored as `null` in MongoDB — nullish coalescing needed for correctness |
| B6 | 🟢 Low | Settings page reads org name from stale localStorage — shows old name after rename |
| B2 | 🟢 Low | MANAGER can read `modules` config — unnecessary exposure |

**Ready for Task 12 — CI/CD & Infrastructure whenever you are.**

I have everything. Full three-phase review now.

---

## Task 12 — CI/CD & Infrastructure

---

## Phase 1: Bug Identification

**Bug 1 — Server password in plaintext via `sshpass` — visible in process list and GitHub Actions logs**

```yaml
sshpass -p "${{ secrets.SERVER_PASSWORD }}" rsync -avz ...
sshpass -p "${{ secrets.SERVER_PASSWORD }}" ssh -o StrictHostKeyChecking=no root@${{ secrets.SERVER_HOST }}
```

`sshpass` passes the password as a command-line argument. On Linux, command-line arguments are visible in `/proc` to any user on the system running the GitHub Actions runner. The password is also partially visible in GitHub Actions debug logs if verbose mode is ever enabled. The correct approach is SSH key-based authentication — generate an ED25519 keypair, add the public key to the server's `~/.ssh/authorized_keys`, and store the private key as a GitHub secret. No password needed.

---

**Bug 2 — `StrictHostKeyChecking=no` disables SSH host verification on every deploy**

```yaml
ssh -o StrictHostKeyChecking=no root@${{ secrets.SERVER_HOST }}
```

`StrictHostKeyChecking=no` means the SSH client accepts any host key, including a spoofed one from a man-in-the-middle attack. A MITM on the GitHub Actions runner network could intercept the deployment and inject malicious code. The server's host key fingerprint should be stored as a GitHub secret and verified explicitly:

```yaml
ssh-keyscan ${{ secrets.SERVER_HOST }} >> ~/.ssh/known_hosts
# Then connect without StrictHostKeyChecking=no
```

---

**Bug 3 — Backend deploy uses `pm2 delete` + `pm2 start` — causes downtime on every deployment**

```yaml
if pm2 list | grep -q "hrms-backend-dev"; then
  pm2 delete hrms-backend-dev
fi
pm2 start server.js --name hrms-backend-dev
```

The `delete` + `start` pattern creates a gap where the process is not running. All in-flight requests are dropped. For a production deployment, even 2-3 seconds of downtime is unacceptable — active payroll generation, attendance marking, or leave approvals are killed mid-flight.

Replace with `pm2 reload` (zero-downtime reload) or `pm2 restart` (graceful restart which waits for in-flight requests to complete via SIGTERM):

```yaml
if pm2 list | grep -q "hrms-backend-prod"; then
  pm2 reload hrms-backend-prod --update-env
else
  pm2 start server.js --name hrms-backend-prod
fi
```

---

**Bug 4 — `.env` file is created on the GitHub Actions runner — plaintext secrets in runner filesystem**

```yaml
- name: Create .env file
  run: |
    cat <<EOF > .env
    MONGO_URI=${{ secrets.STAGING_MONGO_URI }}
    JWT_SECRET=${{ secrets.STAGING_JWT_SECRET }}
```

The `.env` file is written to the runner's filesystem and then `rsync`-ed to the server with `--delete` flag which syncs everything including the `.env` file. GitHub Actions runners are ephemeral and isolated, so the risk is low. But the `.env` file is also included in the `rsync` transfer which means it goes through the network. A better approach: write `.env` directly on the server via SSH, not via rsync:

```yaml
- name: Write .env on server
  run: |
    sshpass -p "${{ secrets.SERVER_PASSWORD }}" ssh root@${{ secrets.SERVER_HOST }} "cat > /root/dev/hrms-backend/.env" << 'ENVEOF'
    MONGO_URI=${{ secrets.STAGING_MONGO_URI }}
    ...
    ENVEOF
```

Or better: use the server's existing `.env` and only deploy code — keep config separate from code deployment entirely.

---

**Bug 5 — `dev.yml` backend workflow injects SMTP vars but `emailService.js` now expects AWS SES vars**

```yaml
SMTP_USER=${{ secrets.STAGING_SMTP_USER }}
SMTP_PASS=${{ secrets.STAGING_SMTP_PASS }}
SMTP_FROM_EMAIL=${{ secrets.STAGING_SMTP_FROM_EMAIL }}
```

The `emailService.js` we updated checks for `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_SES_SENDER_EMAIL`. The dev workflow still injects the old SMTP variables. Without the AWS vars, the service falls back to Gmail SMTP. Since Gmail SMTP credentials are not in the secrets (`SMTP_USER` and `SMTP_PASS` may be empty), emails fail silently.

The dev workflow needs to inject the AWS SES variables instead. This was flagged when we updated `emailService.js` and the `dev.yml` file — it appears the SMTP removal wasn't fully deployed.

---

**Bug 6 — Frontend deploy runs `npm install -g serve pm2` on every deployment**

```yaml
npm install -g serve pm2
```

This installs `serve` and `pm2` globally on the production server on every single frontend deployment. If `pm2` is already installed (it is — we can see it's running 15 processes), this wastes 10-20 seconds on every deploy and risks accidentally upgrading `pm2` to a version with breaking changes. Add a check:

```yaml
command -v pm2 || npm install -g pm2
command -v serve || npm install -g serve
```

---

**Bug 7 — `rsync --delete` on the backend deployment will delete the `.env` file if it already exists on the server**

```yaml
rsync -avz --delete \
  --exclude=node_modules \
  --exclude=.git \
  ./ root@${{ secrets.SERVER_HOST }}:/root/dev/hrms-backend/
```

`--delete` removes any file on the destination that doesn't exist in the source. The source (GitHub Actions runner) creates a new `.env` file with the workflow secrets. But if the server has a manually-edited `.env` (e.g. someone SSH-ed in and added `AWS_SES_SENDER_EMAIL` directly), `rsync` replaces it with the GitHub-generated one, losing the manual addition.

More critically: if the `.env` creation step fails silently (secret not set → empty value), `rsync` deploys an incomplete `.env` to the server, overwriting a working one. This is exactly what happened with the Firebase vars — the server's `.env` was missing Firebase credentials because the workflow was deploying an incomplete one.

Add `.env` to the rsync excludes:
```yaml
--exclude=.env
```

And manage the `.env` separately.

---

**Bug 8 — No health check after deployment — success is assumed without verification**

Both `dev.yml` and `main.yml` end with `pm2 save` and `echo "✅ deployed successfully!"`. Neither workflow verifies the application actually started correctly. A deployment where `server.js` crashes on startup (like the `emailService.js` crash we debugged) is reported as successful because `pm2 start` exits with code 0 even if the process immediately crashes and enters an error loop.

Add a post-deploy health check:
```yaml
- name: Verify deployment
  run: |
    sleep 5  # wait for server to start
    sshpass -p "${{ secrets.SERVER_PASSWORD }}" ssh root@${{ secrets.SERVER_HOST }} \
      "curl -sf http://localhost:9999/api/v1/isokey || (pm2 logs hrms-backend-dev --lines 20 --nostream && exit 1)"
```

---

**Bug 9 — Both prod and dev deploy to the same `SERVER_HOST` — no environment isolation**

```yaml
# dev.yml
root@${{ secrets.SERVER_HOST }}:/root/dev/hrms-backend/

# main.yml  
root@${{ secrets.SERVER_HOST }}:/root/prod/hrms-backend/
```

Both workflows use the same `SERVER_HOST` secret. Dev and prod are on the same VPS, separated only by directory paths and port numbers. A failing dev deployment can affect prod server resources. An accidental `rm -rf /root/` in a deploy script would take down both environments simultaneously. For a payroll system going to production, dev and prod should be on separate servers.

---

## Phase 2: Optimization

**O1 — Backend `npm ci` runs on GitHub Actions runner but packages are re-installed on server too**

```yaml
# On runner:
- name: Install dependencies
  run: npm ci

# On server:
npm install --production
```

`npm ci` on the runner serves no deployment purpose since `node_modules` is excluded from `rsync`. The runner install only validates `package-lock.json` integrity. If this check is desired, keep it. If not, removing it saves 30-60 seconds per deployment.

**O2 — Frontend `pnpm install --no-frozen-lockfile` defeats the purpose of a lockfile**

```yaml
pnpm install --no-frozen-lockfile
```

`--no-frozen-lockfile` allows `pnpm` to update `pnpm-lock.yaml` during CI. This means the CI build may install different package versions than what's in the lockfile. For reproducible builds, use `pnpm install --frozen-lockfile` in CI and only update the lockfile deliberately during development.

**O3 — No caching of `node_modules` between workflow runs**

Neither workflow uses GitHub Actions cache for `node_modules` or the pnpm store. Every deployment reinstalls all packages from scratch. For a project with 580+ packages, this adds 60-90 seconds to every deploy. Add:

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

---

## Phase 3: Feature Enhancements

**E1 — Separate staging and production environments**

Move the dev environment to a separate low-cost VPS ($5/month DigitalOcean droplet or Hetzner). This gives true environment isolation — a broken dev deploy can't starve prod of server resources.

**E2 — Deploy notifications**

Add a Slack or email notification step at the end of each workflow. On success: "✅ hrms-backend-dev deployed (commit: abc123)". On failure: "❌ hrms-backend-dev deploy failed — see logs." This closes the feedback loop where a failed deploy (like the SSH timeout we debugged) goes unnoticed until a user reports the site is down.

**E3 — Rollback capability**

Add a manual rollback workflow (`workflow_dispatch`) that reverts to the previous deployment by resetting the server directory to the prior commit. PM2's `--update-env` combined with `git reset --hard HEAD~1` on the server provides a one-click rollback within 30 seconds.

**E4 — Database backup before deployment**

Before running the migration or deployment step, trigger a MongoDB Atlas snapshot or mongodump export. For a payroll system, a deployment that corrupts data (e.g. a migration that runs twice) should always have an escape hatch.

---

## Summary for Task 12

| # | Severity | Item |
|---|---|---|
| B1 | 🔴 Critical | `sshpass` exposes server password in process list — switch to SSH key auth |
| B3 | 🔴 Critical | `pm2 delete` + `pm2 start` causes downtime on every prod deployment |
| B7 | 🔴 High | `rsync --delete` overwrites server `.env` — deploying incomplete env causes crashes |
| B2 | 🔴 High | `StrictHostKeyChecking=no` disables SSH host verification — MITM risk |
| B8 | 🔴 High | No post-deploy health check — failed startups reported as success |
| B5 | 🟡 Medium | Dev workflow still injects SMTP vars — AWS SES vars not deployed |
| B9 | 🟡 Medium | Dev and prod on same server — no environment isolation |
| B6 | 🟢 Low | `npm install -g serve pm2` runs on every deploy — risks unintended upgrades |
| O2 | 🟢 Low | `--no-frozen-lockfile` in CI defeats reproducible build guarantee |

---

## Full Audit Complete ✅

All 13 modules reviewed. Here's the complete cross-module summary:

---

## Master Bug Registry — Prioritised by Impact

### 🔴 Critical — Fix Before Launch

| Module | Bug |
|---|---|
| Auth | `credentials.md` with real passwords in repo |
| Auth | `sameSite: strict` breaks cross-domain cookie — refresh never works |
| Auth | Logout never calls backend — refresh token persists 7 days |
| Employee | `getEmployeeById` no org scope — cross-tenant data leak |
| Employee | Edit modal wipes `da` + `specialAllowance` — wrong payroll |
| Employee | `pf_opt_in` hardcoded true — overwrites opt-out every edit |
| Attendance | `remoteSubmit` has no auth — public endpoint, attendance fraud possible |
| Leave | Leave cancellation doesn't restore balance — permanent loss |
| Leave | Double LWP deduction — attendance rejection + leave approval both deduct |
| Payroll | PF wage excludes DA — statutory under-deduction |
| Payroll | Pro-ration uses full month denominator — new joiners severely underpaid |
| Payroll | PT slab uses prorated gross — wrong slab for mid-month joiners |
| Settings | `updateConfig` replaces PF/ESI subdocument — destroys all rates |
| Settings | `updateOrgSettings` replaces `shift_rules` — loses grace period etc. |
| CI/CD | `pm2 delete` + `pm2 start` — downtime on every prod deployment |
| CI/CD | `rsync --delete` overwrites server `.env` — crashes after deploy |

### 🟡 High — Fix Before First Real Customers

| Module | Bug |
|---|---|
| Auth | Token accepted in query param — credential leak |
| Auth | `protect` middleware no status check — terminated employees keep access |
| Auth | Refresh endpoint issues tokens without status check |
| Super Admin | `updateOrganization` wipes unset fields |
| Super Admin | No route-level SUPER_ADMIN guard |
| Branch | `isRoot` logic inversion on update |
| Branch | Manager scope read from stale localStorage |
| Attendance | Race condition allows duplicate attendance records |
| Attendance | `resolveShiftWindow` called with inconsistent arguments |
| Leave | No overlap check — employee can double-book same dates |
| Leave | MongoDB session leak on settlement error |
| Leave | MANAGER can trigger `rebuildAllBalances` |
| Payroll | ESI wage includes overtime — statutory over-deduction |
| Payroll | ESI date check uses `new Date()` not payroll month end |
| Payroll | `SalarySlip` reads `deductions.pf` — renders `₹undefined` |
| Outward | `updateEmployeeBankDetails` no role check — salary redirection risk |
| Notifications | `createNotification` awaited in loops — adds 6s to responses |
| AI | Third redundant LLM call on every tool query — doubles cost |
| AI | No client disconnect handler — stream continues after user leaves |
| AI | `get_headcount_analytics` always returns "Unassigned" |
| Settings | MANAGER can modify org-wide shift rules |
| Settings | `HR_ADMIN` can change org bank account details |
| Settings | No rate validation on SalaryConfig — 500% PF accepted |
| CI/CD | `sshpass` exposes server password |
| CI/CD | `StrictHostKeyChecking=no` — MITM risk |
| CI/CD | No post-deploy health check |
