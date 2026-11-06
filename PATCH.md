# Phase 3: permissions and roles module (application code)

Files in this patch mirror the repo layout. Copy `src` and `docs` over the repo root.

## Apply (PowerShell, in claude_nasharat, on branch single-school-baseline)

```powershell
Expand-Archive .\phase3-roles-module.zip -DestinationPath .\_phase3 -Force
Copy-Item .\_phase3\phase3\src  . -Recurse -Force
Copy-Item .\_phase3\phase3\docs . -Recurse -Force
Remove-Item .\_phase3 -Recurse -Force
$env:SWC_NATIVE_BINDING_CACHE="C:\Caches\swc"
npm run verify
```

If `npm run verify` is not the combined command in your package.json, run the three
parts: `npx tsc --noEmit`, `npm run lint`, `npm test`.

## What changed

| Area                                 | Change                                                                                                                                                                                                                                                                                        |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| lib/rbac                             | `hasPermission`, `getEffectivePermissions`, `getSchoolRoles` read the new database functions. The school argument is accepted and ignored so the ~100 existing call sites compile until their modules are migrated. No fallback to a second permission list any more (a failed check throws). |
| lib/rbac/getSchoolRoles              | Reads `name_ar` / `name_en` (display `name` = Arabic name), exposes `is_locked` and `requires_department`, throws on database errors instead of returning an empty list.                                                                                                                      |
| lib/rbac/resolveRoleDisplayName      | `auditor` case removed (role retired).                                                                                                                                                                                                                                                        |
| lib/rbac/permissions.ts + new test   | `permissions.seed.test.ts` fails if the typed catalogue drifts from migration 0009.                                                                                                                                                                                                           |
| actions/rbac                         | Create/update/delete role actions use the new rules: owner only, system roles cannot be deleted, the locked owner role cannot be edited, system role names are never overwritten.                                                                                                             |
| actions/rbac/syncRolePermissions     | Writes only the difference (add first, then remove), so a failure can never leave a role with no permissions.                                                                                                                                                                                 |
| actions/rbac/validateCreateRoleInput | Role keys must match the database check (`^[a-z][a-z0-9_]{1,62}$`).                                                                                                                                                                                                                           |
| lib/school                           | `assignUserRole` no longer takes a school; department rule moved to `resolveAssignmentDepartment` (system role without `requires_department` is school-wide, flagged roles require a department, custom roles keep the department given). Split into small files per the repo rules.          |
| lib/auth                             | `isPlatformOwner`, `getPlatformOwnerUserIds`, `hasAnyPlatformOwner` no longer filter on `school_id`.                                                                                                                                                                                          |
| actions/settings                     | Two call sites edited: `provisionAuthUser.ts`, `updateUserAction.ts` (school argument removed from the role calls only).                                                                                                                                                                      |
| docs                                 | `docs/audits/RBAC_Roles.md` replaced by the new roles report; `docs/design/DATABASE_REDESIGN.md` added.                                                                                                                                                                                       |

No UI file is touched.

## Behaviour changes to be aware of

1. Deleting a system role is now refused (it used to be allowed for the owner).
2. A failed role load or permission check throws instead of returning empty/falling back.
3. Role names: the screens still show one name. New custom roles store it in both
   language columns; system role names come from the message catalogue as before.

## Known state after this phase

Unit tests for the files above are expected to pass. The app as a whole does not run yet:
users, circulars, maintenance, calendar, settings, documents and backup modules still read
`school_id`, `roles.name`, `timezone`, `is_published` and the old function signatures.
They are migrated in the next phases. Tests in those modules that assert the old shapes
(for example `p_school_id` in circular and numbering tests) are updated with their modules.

## Verification limits

Not run on the author's side: tests, ESLint, `next build`. A TypeScript compile of the
changed files overlaid on the full original source showed no new errors against the
original (external packages were not installed, so errors about missing packages were
excluded from the comparison).
