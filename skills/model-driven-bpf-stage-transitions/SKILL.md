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
  const toId = eventArgs.getDestinationStage().getId(); // NOT the target stage's id from getStage()

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
- **Namespace the code** (e.g. `MyApp.Form`) and expose only `onLoad` to the form event registration; pass execution context ("Pass execution context as first parameter" checked).
- **Keep handlers small.** Extract shared checks (role check, required-note check) into helpers rather than copying them across handlers.
