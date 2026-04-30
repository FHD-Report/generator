# Date of Birth Field — Design Spec

**Date:** 2026-04-29  
**Status:** Approved

---

## Overview

Add a Date of Birth (`f-dob`) field to the patient demographics section. When DOB is entered, Age is automatically calculated from DOB and the scan date, and locked as read-only. Clearing DOB unlocks Age for manual entry. DOB is included in all exported reports and cleared during pseudo-anonymisation.

---

## Form Field

- Element: `<input type="date" id="f-dob">`
- Label: `Date of Birth`
- Placement: full-width field inserted between Patient ID and the existing `[Sex | Age]` row
- No existing rows are restructured

---

## Age Calculation Logic

Function `recalcAgeFromDOB()`:

1. Read `f-dob`. If blank, remove `readonly` from `f-age`, hide the DOB hint, and return.
2. Get reference date from `f-scanDate`; fall back to today if blank.
3. Calculate whole years using birthday-boundary method (subtract 1 if birthday hasn't occurred yet in the reference year).
4. Write result to `f-age`, apply `readonly`, show hint `"Calculated from date of birth"` beneath Age field.

**Scan date reactivity:** `recalcAgeFromDOB()` is added to `runMainContentSoftValidation()`. Because the `#mainContent` delegated listener already calls `scheduleMainContentSoftValidation()` on every `input` event, changing the scan date automatically recalculates Age whenever DOB is set — no additional wiring needed.

**DOB field oninput:** `recalcAgeFromDOB()` is called directly via `oninput` on `f-dob` for immediate response without waiting for the debounce.

---

## Visual State

- When DOB is set: `f-age` gains `readonly` attribute; a `calc-hint` element with text `"Calculated from date of birth"` appears beneath it (matching the styling of other calc hints in the form).
- When DOB is cleared: `readonly` removed, hint hidden, Age value cleared so the user can enter it manually.

---

## Exports

DOB is added to all export paths, formatted as `dd-mm-yyyy` using the existing `formatStudyDateTime(dateStr, '')` helper. It appears immediately before Age in the demographics table/row list.

Affected functions:
- `buildEdecPrintHTML()` — added to `demoRows` array next to Age
- `buildEdecPlainText()` — added to `demoRows` array next to Age
- `buildExportRows()` — added as a `row()` call adjacent to Age

If DOB is blank, it is omitted from all exports (consistent with other optional fields).

---

## Pseudo-anonymisation

**Before anonymising** (`captureDemographicWidgetSnapshot`): save `f-dob` value in the snapshot so it can be restored.

**When anonymising** (`applyPseudoAnonymisation`): the existing age-banding logic reads `f-age` to derive the decade band. Because DOB drives `f-age` whenever DOB is set, the DOB-calculated integer is already in `f-age` at this point and is automatically used as the basis for the band — no extra logic needed. After the band is written to `f-age`, clear `f-dob` and remove `readonly` from `f-age` (order matters: clear DOB after the band is applied, so `recalcAgeFromDOB` cannot overwrite the band value).

**When restoring** (`restoreNativeDemographicInputsAfterClear`): restore `f-dob` from snapshot, restore `f-age` type back to `number`, and call `recalcAgeFromDOB()` to re-lock Age if DOB was present.

---

## Draft / Session Save

No changes required. `collectSessionDraftPayload()` already scans all `input` elements inside `#mainContent` by ID, so `f-dob` is captured and restored automatically.

---

## Out of Scope

- DOB is not used for any other calculations (BSA, reference ranges, etc.)
- No validation warning for implausible DOB values beyond the browser's native date-input constraints
- No age band for DOB itself during anonymisation (only Age is banded; DOB is simply cleared)
