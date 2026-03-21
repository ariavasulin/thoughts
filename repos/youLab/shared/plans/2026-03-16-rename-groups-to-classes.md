# Rename "Groups" to "Classes" in Frontend

## Overview

Rename the user-facing "Groups" concept to "Classes" throughout the Askademia frontend by overriding i18n translation values. No Svelte component changes needed — the backend API, variable names, URLs, and file/directory names all stay the same.

## Current State Analysis

The i18n system (i18next with `returnEmptyString: false`) uses the translation key as display text when the value is `""`. All 21 group-related keys in en-US and en-GB have empty values, so the keys themselves (e.g., `"Groups"`) are what users see.

By setting non-empty values (e.g., `"Groups": "Classes"`), we override the display text while keeping the keys unchanged. This means `$i18n.t('Groups')` returns `"Classes"`.

### Key Discoveries:
- i18n config at `open-webui/src/lib/i18n/index.ts:65` sets `returnEmptyString: false`
- Default fallback locale is `en-US` (line 44)
- en-US and en-GB translation files are identical for all group-related keys
- All user-facing "Group" text goes through `$i18n.t()` — no hardcoded strings in components
- 21 keys total across both files

## Desired End State

All user-facing instances of "Group"/"Groups" display as "Class"/"Classes" in the English UI. Backend API, routes (`/admin/users/groups`), variable names, and non-English locales are untouched.

### Verification:
- Run `make dev-frontend` and navigate to Admin > Users > Groups tab
- Tab label says "Classes"
- Page title says "Classes"
- "New Class" button, "Search Classes" placeholder
- Create/edit/delete modals say "Class" instead of "Group"
- Admin Settings > General shows "Default Class"
- Edit User modal shows "User Classes"
- Channel creation modal shows "Class Channel"

## What We're NOT Doing

- Changing backend API endpoints or database schema
- Changing URL paths (`/admin/users/groups` stays)
- Changing Svelte component file names or variable names
- Updating non-English locale files
- Renaming "users" to "students" in help text

## Implementation Approach

Override the 21 group-related keys in both `en-US/translation.json` and `en-GB/translation.json` by setting their values from `""` to the "Class"/"Classes" equivalent.

## Phase 1: Update en-US translation.json

**File**: `open-webui/src/lib/i18n/locales/en-US/translation.json`

Change these 21 keys from `""` to:

```json
"A discussion channel where access is controlled by groups and permissions": "A discussion channel where access is controlled by classes and permissions",
"Add User Group": "Add User Class",
"Default Group": "Default Class",
"Discussion channel where access is based on groups and permissions": "Discussion channel where access is based on classes and permissions",
"Edit User Group": "Edit User Class",
"Group Channel": "Class Channel",
"Group created successfully": "Class created successfully",
"Group deleted successfully": "Class deleted successfully",
"Group Description": "Class Description",
"Group Name": "Class Name",
"Group updated successfully": "Class updated successfully",
"groups": "classes",
"Groups": "Classes",
"New Group": "New Class",
"No groups found": "No classes found",
"Only select users and groups with permission can access": "Only select users and classes with permission can access",
"Search Groups": "Search Classes",
"Select a group": "Select a class",
"Use groups to organize your users and assign permissions.": "Use classes to organize your users and assign permissions.",
"User Groups": "User Classes",
"Who can share to this group": "Who can share to this class",
```

## Phase 2: Update en-GB translation.json

**File**: `open-webui/src/lib/i18n/locales/en-GB/translation.json`

Apply the identical 21 changes as Phase 1.

### Success Criteria:

#### Automated Verification:
- [ ] `make dev-frontend` builds without errors
- [ ] `grep -c '"": ""' open-webui/src/lib/i18n/locales/en-US/translation.json` still works (we only changed 21 of ~2000 keys)
- [ ] JSON is valid: `python3 -c "import json; json.load(open('open-webui/src/lib/i18n/locales/en-US/translation.json'))"`
- [ ] JSON is valid: `python3 -c "import json; json.load(open('open-webui/src/lib/i18n/locales/en-GB/translation.json'))"`

#### Manual Verification:
- [ ] Admin > Users sidebar tab says "Classes"
- [ ] Groups page title says "Classes"
- [ ] "New Class" button visible
- [ ] Search placeholder says "Search Classes"
- [ ] Empty state says "No classes found" / "Use classes to organize..."
- [ ] Create modal title says "Add User Class"
- [ ] Edit modal title says "Edit User Class"
- [ ] Toast messages say "Class created/updated/deleted successfully"
- [ ] Admin Settings > General shows "Default Class" / "Select a class"
- [ ] Edit User modal shows "User Classes" label
- [ ] Channel modal shows "Class Channel" option
- [ ] Member selector shows "classes" / "Classes" headings

## References

- i18n config: `open-webui/src/lib/i18n/index.ts:40-73`
- en-US translations: `open-webui/src/lib/i18n/locales/en-US/translation.json`
- en-GB translations: `open-webui/src/lib/i18n/locales/en-GB/translation.json`
- Groups page component: `open-webui/src/lib/components/admin/Users/Groups.svelte`
- Groups tab: `open-webui/src/lib/components/admin/Users.svelte:108`
