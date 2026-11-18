---
name: model-driven-bpf-stage-transitions
description: Best-practice pattern for handling Business Process Flow (BPF) stage transitions in a model-driven app form with JavaScript. Use when writing OnLoad / OnPreStageChange / OnStageChange / OnProcessStatusChange logic, validating moves between BPF stages, or adding per-transition handlers. Generic, not tied to any application.
---

# Model-Driven App: BPF Stage Transition Pattern

A registry-based pattern: **two classes -> register stages -> register routes -> register transition handlers -> dispatch on stage change.** Replaces large if/else chains in the stage-change event with small, testable, per-transition functions.

## 1. Two classes

```js
// StageRouter: knows which stages exist and which moves are legal.
class StageRouter {
  constructor() {
    this.stages = new Map();
    this.routes = new Map();
  }
  addStage(stage) {
    if (!stage || !stage.id) {
      console.error("Stage must have an 'id' property");
      return;
    }
    this.stages.set(stage.id, stage);
  }
  addRoute(fromId, toId) {
    if (!this.routes.has(fromId)) this.routes.set(fromId, new Set());
    this.routes.get(fromId).add(toId);
  }
  addBidirectionalRoute(id1, id2) {
    this.addRoute(id1, id2);
    this.addRoute(id2, id1);
  }
  isAllowed(fromId, toId) {
    return this.routes.has(fromId) && this.routes.get(fromId).has(toId);
  }
}

// TransitionHandler: knows what to validate/do for each specific move.
class TransitionHandler {
  constructor() {
    this.handlers = new Map();
  }
  register(fromId, toId, handler) {
    if (typeof handler !== "function") {
      console.error("Transition handler must be a function");
      return;
    }
    this.handlers.set(`${fromId}->${toId}`, handler);
  }
  // ctx = { formContext, eventArgs, fromId, toId } passed explicitly to handlers
  handle(fromId, toId, ctx) {
    const handler = this.handlers.get(`${fromId}->${toId}`);
    if (!handler) {
      console.error(`No transition handler defined from ${fromId} to ${toId}`);
      return false; // fail closed
    }
    return handler(ctx);
  }
}

const stageRouter = new StageRouter();
const transitionHandler = new TransitionHandler();
```

## 2. Add stages

Use the BPF stage GUIDs (`formContext.data.process.getActiveStage().getId()`) held in named constants, never inline.

```js
const STAGE_A = "<stage-guid>";
const STAGE_B = "<stage-guid>";
const STAGE_C = "<stage-guid>";

stageRouter.addStage({ id: STAGE_A, name: "Stage A" });
stageRouter.addStage({ id: STAGE_B, name: "Stage B" });
stageRouter.addStage({ id: STAGE_C, name: "Stage C" });
```

## 3. Add routes

Declare only the moves that are legal. Anything not declared is blocked.

```js
stageRouter.addBidirectionalRoute(STAGE_A, STAGE_B); // forward and back
stageRouter.addRoute(STAGE_B, STAGE_C);              // forward only
```

## 4. Transition logic (one handler per from->to pair)

Handlers receive context, validate, mutate fields, and return a result. Keep each handler to one transition.

```js
transitionHandler.register(STAGE_A, STAGE_B, (ctx) => {
  const { formContext, eventArgs } = ctx;
  if (!formContext.getAttribute("some_field").getValue()) {
    preventTransition(eventArgs, "Please fill in Some Field before continuing.");
    return false;
  }
  // set fields / status for the new stage here
  return true;
});

transitionHandler.register(STAGE_B, STAGE_A, (ctx) => {
  const { formContext, eventArgs } = ctx;
  if (formContext.getAttribute("action").getValue() !== ACTION_RETURN) {
    preventTransition(eventArgs, "Select the Return action to go back.");
    return false;
  }
  return true;
});
```

Shared helper:

```js
function preventTransition(eventArgs, message) {
  eventArgs.preventDefault();
  Xrm.Navigation.openAlertDialog({ title: "Warning", text: message });
}
```

## 5. Dispatch on stage change

Register on form **OnLoad**, passing the execution context.

```js
function onLoad(executionContext) {
  const formContext = executionContext.getFormContext();
  formContext.data.process.addOnPreStageChange(onPreStageChange);
  formContext.data.process.addOnProcessStatusChange(onProcessStatusChange);
}

function onPreStageChange(executionContext) {
  const formContext = executionContext.getFormContext();
  const eventArgs = executionContext.getEventArgs();

  const fromId = formContext.data.process.getActiveStage().getId();
  // OnPreStageChange event args expose getStage() (the stage being moved TO) and getDirection().
  // There is NO getDestinationStage() in the Client API - calling it throws.
  const toId = eventArgs.getStage().getId();

  // Optional: only handle "next"/"previous" direction if needed
  // const dir = eventArgs.getDirection(); // "Next" | "Previous"

  if (!stageRouter.isAllowed(fromId, toId)) {
    preventTransition(eventArgs, "You cannot move to this stage.");
    return;
  }

  const ok = transitionHandler.handle(fromId, toId, { formContext, eventArgs, fromId, toId });
  if (ok !== true) {
    // Handler already called preventTransition for validation failures.
    // Defensive: make sure a falsy result always blocks the move.
    if (!eventArgs.getDefaultPrevented || !eventArgs.getDefaultPrevented()) {
      eventArgs.preventDefault();
    }
  }
}
```

Finish the process (last stage "Finish") in `onProcessStatusChange`, check `formContext.data.process.getStatus() === "finished"`, validate, and if invalid, call `formContext.data.process.setStatus("active")` to reopen it.

## Best practices

- **Pass context explicitly.** Handlers must receive `formContext` and `eventArgs` as arguments. Never rely on outer-scope variables or globals for them (they will be undefined or stale).
- **Fail closed.** Unknown route or missing handler returns false and blocks the move. A missing handler is a configuration bug, so log it with `console.error`.
- **One registry, one name.** Use a single handler instance consistently (a common bug is registering on one object and dispatching on another).
- **Constants must match.** Route/handler registration and constants must use the same stage constants (watch for near-duplicate names like `STAGE_COMPLETION` vs `STAGE_COMPLETION_REQUEST`).
- **`preventDefault` is only valid inside `OnPreStageChange`** (and OnPreProcessStatusChange). It cannot cancel `OnStageChange`.
- **Async limits.** `OnPreStageChange` cannot await async work to decide cancellation. If a check needs a Web API call (e.g. "does a note exist?"), either prefetch the data on load / on field change and keep it in a variable, or cancel the move first, run the async check, then advance programmatically with `formContext.data.process.moveNext()` / `movePrevious()`. Do not assume a fire-and-forget fetch result is ready before the handler returns.
- **Check security roles/ownership in the handler**, but also enforce it server-side (plug-in or business rule). Client JS is not a security boundary.
- **Reset stage-specific fields** (action choice, etc.) in the handler for the stage you are entering so stale values do not carry over.
- **Return values:** `true` = allowed, `false` = blocked. If you need richer outcomes (e.g. `"rejected"` -> save then close), use a documented set of string/enum results and handle each explicitly in the dispatcher.
- **Delay `form.ui.close()`/navigation** until after save completes (use the promise from `formContext.data.save()`), not an arbitrary `setTimeout`.
- **Never call `formContext.data.save()` inside `OnPreStageChange`.** The BPF saves the stage move itself; an extra save at that moment collides with it and shows a generic "The record could not be saved" error even though the stage move and field values were actually persisted. Inside `OnPreStageChange` only validate, set field values, and set a flag (e.g. `stageTransitionApproved = true`) - then do the save and any navigation/close in an `OnStageChange` handler (`addOnStageChange`), which runs after the BPF has finished its own move:
  ```js
  function onStageChange() {
    if (!stageTransitionApproved) return;
    stageTransitionApproved = false;
    formContext.data.save().then(() => Xrm.Navigation.navigateTo({ pageType: "entitylist", entityName: "<table>" }));
  }
  ```
- **Don't remove and re-add the `OnPreStageChange` handler from inside itself.** Register once on load (call `removeOnPreStageChange` before `addOnPreStageChange` there to avoid duplicates) and leave it registered.
- **Verify control and attribute logical names against the real form before using them.** `getControl()` returns `null` for a name that is not on the form (and control names can differ from the field name, e.g. `xgh_actionpointnumber1`, `Subgrid_stakeholder`). One wrong name in a form-helper object makes the first `ctrl.setDisabled(...)` throw `Cannot read properties of null` and aborts the whole `onLoad`, so the BPF handlers never register and later errors look unrelated. List the real names from the browser console with `Xrm.Page.getControl().forEach(c => console.log(c.getName(), c.getLabel()))` and keep a form-helper that is checked once at load (log every null control) rather than discovering them one error at a time.
- **An error message like `Cannot read properties of undefined (reading 'identifier')` that matches no text in your own code comes from Microsoft's own scripts.** Look for an earlier failure in your `onLoad` or a save/handler lifecycle problem in your code that left the BPF in an inconsistent state, rather than assuming a platform bug.
- **A generic "An error has occurred" save failure does not say what failed.** Get the real cause from the failing request's response body (browser dev tools -> Network) or the Plug-in Trace Log for the dialog's Activity Id. Check whether the data was in fact saved before debugging the server.
- **Use only the public Client API; never private properties.** Read stages with `getActiveStage().getId()` and `eventArgs.getStage().getId()`. Do not use `getActiveStage()._activeStageId` or `eventArgs._stage._stageStep.id`: they are undocumented, can be `undefined` while the BPF is still loading, and break on platform updates.
- **Never busy-wait with `while (x === undefined) { try {...} catch {} }`.** It freezes the browser tab if the value never arrives. Read the stage once, null-check it, and if it is not ready retry with a bounded `setTimeout` (see the rendering-retry rule below), or fall back to a safe default.
- **Normalize every GUID before comparing or using it as a map key.** Strip braces and lower-case: `id.replace(/[{}]/g, "").toLowerCase()`. Apply this to stage ids, record ids and user ids. Stage ids from `getId()`, from a string field and from constants differ in case/braces, and a raw comparison silently fails (route not found, handler not found).
- **If you keep a "current stage" field on the table (a Choice column such as `xgh_stage`), map its numeric option value to the BPF stage GUID with a lookup object.** Do not store or compare a GUID in a Choice column.
  - Keep two named constant sets: the Choice option values (numbers) and the BPF stage GUIDs, plus `CHOICE_TO_BPF_STAGE = { [VALUE]: GUID }`.
  - Write the numeric value with `setValue(STAGE_CHOICE_X)`. Resolve the stage with one helper such as `resolveActiveStage()`: Choice number -> mapped GUID, then a string GUID if present, then `getActiveStage().getId()`.
  - The field is empty on a brand-new record, so the helper must fall back to the active BPF stage. Otherwise all stage logic (required levels, visibility) is skipped on create.
- **BPF header controls render after `OnLoad` runs.** `getControl("header_process_<field>")` can return `null` on first call. Do not assume it is there: collect the controls, and if any is missing retry with a bounded `setTimeout` (for example 300 ms x 20) and then give up quietly. Reset the retry counter once they are found.
- **Show/hide BPF action controls by stage only; express role restrictions with `setDisabled`.** Hide all stage controls first, then show only the control of the active stage. Use `setDisabled(!(hasRole || isAdmin))` for users who may not act, rather than hiding it. Hiding by role makes a stage look empty or broken for everyone else. Also restrict a shared option set per stage (for example only "Submit" in stage 1, only "Complete" or "Return" in the last) with a helper such as `restrictActionOptions(control, allowedValues)`.
- **Register all BPF handlers idempotently on load:** call `removeOnX(handler)` immediately before `addOnX(handler)` for `OnPreStageChange`, `OnStageChange` and `OnProcessStatusChange`. Use the same named function reference for add and remove (not an inline closure), so repeated `onLoad` calls (form refresh, tab switch) never stack duplicates.
- **Do not overwrite a module-level `executionContext` inside event handlers.** Each event gets its own execution context, so use a local `const` (`execContext.getEventArgs()`, `execContext.getFormContext()`) and pass it on. Reassigning the shared global from the stage handler makes later code use a stale or wrong context.
- **Wrap each stage handler in `try/catch` that cancels the move on error.** In the `catch`, call `eventArgs.preventDefault()` and show the error, so an unexpected exception never lets an unvalidated transition through. Show `error.message` only to administrators; give other users a generic message plus a contact hint.
- **Use one `showUserErrorMessage(message, technicalDetails)` helper for every user-facing error.** Call `Xrm.Navigation.openErrorDialog({ message, details })`. Passing `details` enables the dialog's "Download Log File" button. Build the details from a session error log (timestamp, record id, user id, active stage, role names, URL, the current error and its stack, and earlier session errors). This makes support requests self-diagnosing. Always pass `error.stack || error.message` from `catch` blocks.
- **When a transition handler reverts a field, make the rollback explicit.** If a post-stage save fails, tell the user the record was not saved and leave the form open for a retry. Do not navigate away.
- **Namespace the code** (e.g. `MyApp.Form`) and expose only `onLoad` to the form event registration; pass execution context ("Pass execution context as first parameter" checked).
- **Keep handlers small.** Extract shared checks (role check, required-note check) into helpers rather than copying them across handlers.
