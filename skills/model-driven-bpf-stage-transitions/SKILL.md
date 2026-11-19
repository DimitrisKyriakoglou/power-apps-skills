---
name: model-driven-bpf-stage-transitions
description: Pattern for planning and writing Business Process Flow (BPF) stage transition logic in a model-driven app form with JavaScript, using one declarative transition table plus one dispatcher. Use when planning or documenting BPF business rules, writing OnLoad / OnPreStageChange / OnStageChange / OnProcessStatusChange logic, validating or blocking moves between stages, adding or changing a transition, or explaining how a BPF works. Generic, not tied to any application.
---

# Model-Driven App: BPF Stage Transitions

**One table describes every allowed move. One dispatcher enforces it.** A move that is not in the table is blocked. Business people can read and approve the table; developers implement it row by row.

Use this skill for every BPF: planning the rules, writing the code, reviewing it, and explaining it to someone new.

---

## 1. How a BPF stage move works (plain words)

A **Business Process Flow** is the chevron bar at the top of a form. Each chevron is a **stage**. A **transition** is a move from one stage to another, forward ("Next Stage") or back ("Previous Stage").

When the user clicks, the platform fires events in this order:

| Order | Event | When | What you can do |
|---|---|---|---|
| 1 | `OnPreStageChange` | **Before** the move | Validate, set field values, **cancel** the move (`preventDefault`). Do not save here. |
| 2 | (platform) | | The BPF performs and saves its own stage move |
| 3 | `OnStageChange` | **After** the move | Save the record, navigate, refresh UI. Cannot cancel. |
| 4 | `OnProcessStatusChange` | When the whole process finishes or is abandoned | Final validation, write the "completed" status |

Every stage is identified by a **GUID**. The code compares GUIDs, never stage names.

---

## 2. Step 1: Plan the transitions (before any code)

Write one row per direction. Forward and return moves are separate rows. Every row answers the same questions, in business language, so non-developers can review it:

| # | From | To | Who may do it | Action to select first | Validations (all must pass) | Fields updated |
|---|---|---|---|---|---|---|
| 1 | New | Review | Requestor, Admin | Submit for Review | progress not "Not Started"; due date set | stage, status, submitted-on date |
| 2 | Review | Completion | Responsible, Admin | Submit for Completion | evidence note exists | stage, status |
| 3 | Review | New | Responsible, Admin | Return for Info | return comment filled | stage, status, clear action |
| 4 | Completion | Review | Requestor, Admin | Return for Info | none | stage, status |

Rules for the plan:

- Rows can have different numbers of validations and different updated fields. Do not force them to match.
- Any move not listed is **not allowed**. Confirm this with the business owner explicitly.
- List who may act per transition, not per stage. The same stage can be left by different roles in different directions.
- Keep this table in the project documentation. The code table in step 4 mirrors it one-to-one.
- Also note what changes **inside** each stage (which fields are locked, required or hidden). That is stage behaviour, covered in section 6, not a transition.

---

## 3. Step 2: Constants

Name every value once. Never use raw GUIDs or numbers inline.

```js
// BPF stage ids (lower-case, no braces)
const BPF_STAGE_A = "<stage-guid>";
const BPF_STAGE_B = "<stage-guid>";

// Choice option values (numbers) for the stage and status columns
const STAGE_CHOICE_A = 100000000;
const STAGE_CHOICE_B = 100000001;
const STATUS_SUBMITTED = 100000000;

// Choice option values for the BPF action control
const ACTION_SUBMIT = 100000000;
const ACTION_RETURN = 100000001;
```

If the table has a "current stage" Choice column, keep a lookup from its numeric value to the BPF stage GUID (see section 7, Stages and GUIDs).

---

## 4. Step 3: The transition table

```js
// Every allowed stage transition. Key = "fromStageId->toStageId". A move NOT listed here is blocked.
// canMove is a FUNCTION: role flags (isRequestor, ...) are set in onLoad, after this table is created.
const TRANSITIONS = {
  // 1. A -> B
  [`${BPF_STAGE_A}->${BPF_STAGE_B}`]: {
    canMove: () => isRequestor || isSystemAdmin,
    requiredAction: ACTION_SUBMIT,
    validations: [
      validateProgressNotStarted,
      requireField("xgh_duedate", "Due Date"),
    ],
    updates: {
      xgh_stage: STAGE_CHOICE_B,
      xgh_status: STATUS_SUBMITTED,
      xgh_submittedon: () => new Date(), // a function value is computed at transition time
    },
  },
  // 2. B -> A (return)
  [`${BPF_STAGE_B}->${BPF_STAGE_A}`]: {
    canMove: () => isResponsible || isSystemAdmin,
    requiredAction: ACTION_RETURN,
    validations: [requireField("xgh_returncomment", "Return Comment")],
    updates: {
      xgh_stage: STAGE_CHOICE_A,
      xgh_status: STATUS_RETURNED,
      xgh_action: null, // clear the action for the stage being entered
    },
  },
};
```

### Row fields

| Field | Required | Meaning |
|---|---|---|
| `canMove` | yes | Function returning true if the current user may make this move |
| `requiredAction` | no | Option the user must select in the BPF first. Omit if none |
| `validations` | no | Array of synchronous checks. Omit or `[]` for none |
| `updates` | no | `{ logicalName: value }`. Any number of fields. A function value is evaluated at transition time (dates, current user) |
| `apply` | no | `(ctx) => {}` run after `updates` for logic a field map cannot express. Use rarely |
| `label` | no | Destination name for messages |

### Validations

A validation returns `null` when it passes, or a message string when it fails. It never shows UI itself, so the dispatcher can collect every failure and show them in one message.

```js
function validateProgressNotStarted() {
  const v = form.progressStatusAttr.getValue();
  return v === null || v === PROGRESS_NOT_STARTED ? "The Progress Status cannot be 'Not Started'." : null;
}

// Reusable factory: write a common check once, use it in many rows
function requireField(attributeName, label) {
  return () => {
    const attr = formContext.getAttribute(attributeName);
    const v = attr && attr.getValue();
    return v === null || v === undefined || v === "" ? `${label} is required.` : null;
  };
}
```

Validations must be synchronous. If a check needs a Web API call, prefetch the data on load or on field change and read it from a variable (see Async limits in section 8).

---

## 5. Step 4: The dispatcher (the only place the table is read)

Register on form `OnLoad`. Use named functions and remove before add, so a repeated `onLoad` never stacks duplicates.

```js
function onLoad(executionContext) {
  formContext = executionContext.getFormContext();
  formContext.data.process.removeOnPreStageChange(onPreStageChange);
  formContext.data.process.addOnPreStageChange(onPreStageChange);
  formContext.data.process.removeOnStageChange(onStageChange);
  formContext.data.process.addOnStageChange(onStageChange);
  formContext.data.process.removeOnProcessStatusChange(onProcessStatusChange);
  formContext.data.process.addOnProcessStatusChange(onProcessStatusChange);
}

function preventStageTransition(eventArgs, message) {
  eventArgs.preventDefault();
  showUserErrorMessage(message);
}

function onPreStageChange(execContext) {
  const eventArgs = execContext.getEventArgs();
  try {
    const fromId = getActiveStageId();
    // getStage() is the stage being moved TO. There is no getDestinationStage() in the Client API.
    const destination = eventArgs.getStage();
    const toId = destination ? normalizeGuid(destination.getId()) : null;

    if (!fromId || !toId) {
      preventStageTransition(eventArgs, "The current stage could not be determined. Please refresh the page.");
      return;
    }

    const rule = TRANSITIONS[`${fromId}->${toId}`];

    // 1. Is the move allowed at all? Not in the table = blocked.
    if (!rule) {
      preventStageTransition(eventArgs, "You cannot move to this stage.");
      return;
    }
    // 2. Is this user allowed?
    if (!rule.canMove()) {
      preventStageTransition(eventArgs, "You are not authorized to move to this stage.");
      return;
    }
    // 3. Was the required action selected?
    if (rule.requiredAction !== undefined && form.actionAttr.getValue() !== rule.requiredAction) {
      preventStageTransition(
        eventArgs,
        `To make this move, please first select the action "${form.actionAttr.getOption(rule.requiredAction).text}".`
      );
      return;
    }
    // 4. Run ALL validations and report every failure in one message.
    const errors = (rule.validations || []).map((check) => check()).filter(Boolean);
    if (errors.length > 0) {
      preventStageTransition(eventArgs, errors.join("\n"));
      return;
    }
    // 5. Everything passed: apply updates LAST, so a blocked move leaves the form untouched.
    //    setValue only marks fields dirty. The save happens in onStageChange.
    Object.entries(rule.updates || {}).forEach(([name, value]) => {
      const attr = formContext.getAttribute(name);
      if (!attr) { console.error(`updates: attribute "${name}" is not on the form`); return; }
      attr.setValue(typeof value === "function" ? value() : value);
    });
    if (rule.apply) rule.apply({ formContext, eventArgs });

    // Do NOT save here. Flag the move; onStageChange saves.
    stageTransitionApproved = true;
  } catch (error) {
    eventArgs.preventDefault(); // an exception must never let an unvalidated move through
    showUserErrorMessage(
      isSystemAdmin
        ? `Stage transition error: ${error.message}`
        : "An unexpected error occurred during stage transition. Please contact your administrator.",
      error.stack || error.message
    );
  }
}

// Runs after the BPF has completed its own move: save once, then navigate.
function onStageChange() {
  if (!stageTransitionApproved) return;
  stageTransitionApproved = false;
  formContext.data.save().then(
    () => Xrm.Navigation.navigateTo({ pageType: "entitylist", entityName: "<table>" }),
    (error) => showUserErrorMessage(
      "The record could not be saved. Please try again or contact your administrator.",
      (error && (error.message || error.stack)) || String(error)
    ) // leave the form open for a retry; do not navigate away
  );
}
```

Click-to-saved-record sequence: checks pass -> `updates` applied (dirty, unsaved) -> `stageTransitionApproved = true` -> BPF completes its move -> `onStageChange` saves once -> navigate after the save promise resolves.

### Finishing the process

Handle the last stage's "Finish" in `onProcessStatusChange`: check `formContext.data.process.getStatus() === "finished"`, validate, and if invalid call `formContext.data.process.setStatus("active")` to reopen. A "completed" status written when the whole process finishes belongs here, not in the transition table.

---

## 6. Stage behaviour (separate from transitions)

Transitions decide *whether you may leave* a stage. **Stage behaviour** decides how the form looks *while in* a stage: locked fields, required fields, visible controls, allowed action options. Keep it in one function, `applyStageBehaviour()`, with one `switch` on the active stage, called on load and after each stage change.

- Show or hide BPF action controls **by stage only**. Hide all first, then show the active stage's control.
- Express role limits with `setDisabled(!(hasRole || isAdmin))`, not by hiding. Hiding by role makes a stage look empty to everyone else.
- Restrict a shared option set per stage with a helper such as `restrictActionOptions(control, allowedValues)`.
- BPF header controls render after `OnLoad`. `getControl("header_process_<field>")` can be `null` at first. Retry with a bounded `setTimeout` (for example 300 ms x 20), then give up quietly; reset the counter once found.

---

## 7. Stages and GUIDs

- **Normalize every GUID** before comparing or using it as a key: `id.replace(/[{}]/g, "").toLowerCase()`. Apply this to stage ids, record ids and user ids. Raw GUIDs differ in case and braces, and a raw comparison silently fails (the transition is "not found").
- **Use only the public Client API.** Read stages with `getActiveStage().getId()` and `eventArgs.getStage().getId()`. Never use `_activeStageId` or `_stage._stageStep.id`: they are undocumented, can be `undefined` while the BPF loads, and break on platform updates.
- **Current-stage Choice column** (e.g. `xgh_stage`): map its numeric option value to the BPF stage GUID with a lookup object. Never store or compare a GUID in a Choice column.
  - Keep two constant sets (Choice numbers and BPF GUIDs) plus `CHOICE_TO_BPF_STAGE = { [VALUE]: GUID }`.
  - Resolve the stage with one helper, `resolveActiveStage()`: Choice number -> mapped GUID, then a string GUID if present, then `getActiveStage().getId()`.
  - The column is empty on a new record, so the helper must fall back to the active BPF stage; otherwise all stage behaviour is skipped on create.
- **Never busy-wait** (`while (x === undefined) {...}`): it freezes the tab. Read once, null-check, and retry with a bounded `setTimeout` or use a safe default.

---

## 8. Rules that prevent the common failures

### Correctness
- **Fail closed.** Not in `TRANSITIONS` means blocked. Never add a "default allow".
- **`canMove` is a function, not a value.** A value would freeze role flags as `false` at definition time.
- **Constants must match.** Watch near-duplicates such as `STAGE_COMPLETION` vs `STAGE_COMPLETION_REQUEST`. A typo silently makes a move "not allowed".
- **Apply `updates` only after every check passes.**
- **Reset stage-specific fields** (action choice, etc.) for the stage being entered, in `updates`, so stale values do not carry over.
- **Client JS is not a security boundary.** Check roles in `canMove`, and enforce the same rule server-side (plug-in or business rule).

### Events and saving
- **`preventDefault` works only in `OnPreStageChange`** (and `OnPreProcessStatusChange`). It cannot cancel `OnStageChange`.
- **Never call `formContext.data.save()` inside `OnPreStageChange`.** The BPF saves its own move; a second save collides and shows a generic "The record could not be saved" error even though the data was persisted. Validate, set values, set the flag, then save in `onStageChange`.
- **Wait for the save promise** before `form.ui.close()` or navigation. Never use an arbitrary `setTimeout`.
- **Register once, idempotently.** Same named function for `removeOnX` and `addOnX`; never register or unregister from inside the handler itself.
- **Async limits.** `OnPreStageChange` cannot await. For a Web API check, either prefetch on load / field change and keep the result in a variable, or cancel the move, run the async check, then advance with `formContext.data.process.moveNext()` / `movePrevious()`. A fire-and-forget fetch is not ready when the handler returns.
- **Richer outcomes** than allow/block (for example "rejected" -> save then close): use a documented set of enum results and handle each explicitly in the dispatcher.

### Context
- **Use each event's own execution context.** Take `getEventArgs()` and `getFormContext()` from the `execContext` argument. Never overwrite a shared module-level `executionContext` inside event handlers; it goes stale.
- **Namespace the code** (e.g. `MyApp.Form`) and register only `onLoad` on the form, with "Pass execution context as first parameter" checked.

### Errors and support
- **Wrap each stage event in `try/catch` that cancels the move** (`preventDefault`) and shows the error. Show `error.message` to administrators only; others get a generic message plus a contact hint.
- **One `showUserErrorMessage(message, technicalDetails)` helper** for every user-facing error, calling `Xrm.Navigation.openErrorDialog({ message, details })`. Passing `details` enables "Download Log File". Build details from a session error log: timestamp, record id, user id, active stage, role names, URL, current error and stack, earlier session errors. Always pass `error.stack || error.message` from `catch`.
- **Failed post-stage save:** tell the user the record was not saved, leave the form open, do not navigate.

### Troubleshooting
- **Verify control and attribute logical names against the real form.** `getControl()` returns `null` for a wrong name, and control names can differ from field names (e.g. `xgh_actionpointnumber1`, `Subgrid_stakeholder`). One wrong name makes the first `ctrl.setDisabled(...)` throw `Cannot read properties of null`, aborts `onLoad`, and the BPF handlers never register, so later errors look unrelated. List real names in the browser console: `Xrm.Page.getControl().forEach(c => console.log(c.getName(), c.getLabel()))`. Keep a form helper that checks them once at load and logs every null.
- **`Cannot read properties of undefined (reading 'identifier')`** with no matching text in your code comes from Microsoft's scripts. Look for an earlier failure in your `onLoad` or a save/handler lifecycle problem that left the BPF inconsistent; do not assume a platform bug.
- **A generic "An error has occurred" on save** does not say what failed. Read the failing request's response body (dev tools -> Network) or the Plug-in Trace Log for the Activity Id. Check whether the data was in fact saved before debugging the server.

---

## 9. Recipes

### Add a new transition
1. Add a row to the plan table (section 2) and get it approved.
2. Add the BPF stage constants if the stage is new.
3. Add one entry to `TRANSITIONS` mirroring the row. Reuse existing validation functions and factories.
4. Add any new validation as a function returning `null` or a message.
5. Run the test matrix below for the new row.

### Change a rule
Edit the one row. Nothing else should need to change; if it does, the logic is in the wrong place.

---

## 10. Test matrix

Test every row of the table, plus the blocked cases:

| Case | Expected |
|---|---|
| Each listed move, correct role, correct action, all validations pass | Allowed; `updates` applied; saved once; navigates |
| Each listed move, wrong role | Blocked: not authorized; no field changed |
| Each listed move, wrong or missing action | Blocked: names the required action; no field changed |
| Each listed move, one validation failing | Blocked; message shows that failure |
| Each listed move, several validations failing | Blocked; **all** failures shown in one message |
| A move not in the table (e.g. skip a stage) | Blocked: cannot move to this stage |
| Administrator | Allowed where `canMove` includes admin |
| Save fails after an allowed move | Error shown with details; form stays open |
| Reload the form twice | Handlers fire once per move (no duplicates) |
| Brand-new record | Stage behaviour resolves from the active BPF stage; no errors |

---

## 11. Review checklist

- Table matches the approved plan, row for row.
- Every key uses constants; GUIDs normalized.
- `canMove` is a function in every row.
- Validations return messages and show no UI; none is async.
- `updates` field names exist on the form.
- No `save()` in `OnPreStageChange`; save and navigation in `OnStageChange`.
- Handlers registered with remove-then-add, named functions.
- `try/catch` cancels the move; errors go through `showUserErrorMessage` with details.
- Roles also enforced server-side.
