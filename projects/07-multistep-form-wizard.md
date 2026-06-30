# 07 — Project: Multi-Step Form Wizard

> **"Multi-step forms fail in a specific way: each individual step is trivial to build, but the form as a WHOLE accumulates complexity nobody designed for — conditional steps that depend on earlier answers, validation that needs to consider data across multiple steps, partial saves so users don't lose 20 minutes of work, and back-navigation that doesn't discard what they already entered. The wizard's state machine is the actual product; the individual step UIs are just views into it."**

This project guide builds a multi-step form wizard — the kind used for onboarding flows, insurance applications, or checkout processes: conditional step branching, cross-step validation, draft persistence, and a progress indicator that accurately reflects a non-linear flow.

---

## 📚 What You'll Build

A wizard with: multiple steps with conditional branching (some steps only appear based on earlier answers), step-level and cross-step validation, automatic draft saving, a progress indicator, and the ability to navigate back without losing data — built on a state machine rather than ad-hoc step-tracking state.

---

## Requirements

```
FUNCTIONAL:
  - Multiple steps, some of which are conditionally shown/hidden based
    on answers from earlier steps
  - Per-step validation (can't proceed until the current step is valid)
  - Cross-step validation (e.g., "end date must be after start date" where
    start date and end date are on different steps)
  - Draft auto-save (resume an incomplete form later)
  - Accurate progress indicator (handles the fact that the TOTAL number
    of steps can change based on branching)
  - Back navigation preserves previously entered data

NON-FUNCTIONAL:
  - Must not lose data on accidental page refresh
  - Step transitions should feel instant — no waiting for a server
    round-trip just to move between local steps
  - The step configuration should be data-driven enough that adding
    a new step doesn't require touching unrelated step components
```

---

## Architecture Overview

```
COMPONENT TREE:
  <WizardContainer>
    <ProgressIndicator />
    <WizardStepRenderer>          (renders whichever step is active)
      <Step1PersonalInfo />
      <Step2AddressInfo />
      <Step3PaymentInfo />          (conditionally shown)
      <Step4Review />
    <WizardNavigation>            (back / next / submit buttons)

KEY ARCHITECTURE DECISION: model the wizard as an EXPLICIT STATE MACHINE
(steps as states, "next"/"back" as transitions, conditional steps as
guarded transitions) rather than as a simple `currentStepIndex` counter.
A counter cannot express "skip step 3 if the user answered 'no' on step 1"
without scattering conditional logic throughout the navigation code.
```

---

## Step 1 — Modeling the Wizard as a State Machine

```typescript
interface WizardState {
  formData: Record<string, unknown>;
  currentStepId: string;
  visitedSteps: string[]; // for back navigation and progress calculation
}

interface StepDefinition {
  id: string;
  component: React.ComponentType<StepProps>;
  // A step can be conditionally INCLUDED based on current form data
  isApplicable: (formData: Record<string, unknown>) => boolean;
  // Validation specific to this step
  validate: (formData: Record<string, unknown>) => Record<string, string>;
  // Determines the NEXT step id, given current data (supports branching)
  getNextStepId: (formData: Record<string, unknown>) => string | null; // null = end of wizard
}

const wizardSteps: StepDefinition[] = [
  {
    id: "personal-info",
    component: PersonalInfoStep,
    isApplicable: () => true, // always shown
    validate: (data) => {
      const errors: Record<string, string> = {};
      if (!data.fullName) errors.fullName = "Required";
      if (!data.email?.includes("@")) errors.email = "Invalid email";
      return errors;
    },
    getNextStepId: () => "employment-status",
  },
  {
    id: "employment-status",
    component: EmploymentStatusStep,
    isApplicable: () => true,
    validate: (data) => {
      const errors: Record<string, string> = {};
      if (!data.employmentStatus) errors.employmentStatus = "Required";
      return errors;
    },
    // BRANCHING: the next step depends on the answer given in THIS step
    getNextStepId: (data) =>
      data.employmentStatus === "employed" ? "employer-info" : "income-source",
  },
  {
    id: "employer-info", // only reached if employmentStatus === 'employed'
    component: EmployerInfoStep,
    isApplicable: (data) => data.employmentStatus === "employed",
    validate: (data) => {
      const errors: Record<string, string> = {};
      if (!data.employerName) errors.employerName = "Required";
      return errors;
    },
    getNextStepId: () => "review",
  },
  {
    id: "income-source", // only reached if employmentStatus !== 'employed'
    component: IncomeSourceStep,
    isApplicable: (data) => data.employmentStatus !== "employed",
    validate: (data) => {
      const errors: Record<string, string> = {};
      if (!data.incomeSource) errors.incomeSource = "Required";
      return errors;
    },
    getNextStepId: () => "review",
  },
  {
    id: "review",
    component: ReviewStep,
    isApplicable: () => true,
    validate: () => ({}), // cross-step validation happens at submit time
    getNextStepId: () => null, // end of wizard
  },
];
```

**Key decision:** each step's `getNextStepId` function determines branching DECLARATIVELY based on current form data, rather than the navigation component containing a tangle of `if (employmentStatus === 'employed') { goToStep(5) } else { goToStep(7) }` logic. This means adding a new branch, or a new step in the middle of an existing branch, only requires editing that ONE step's definition — not hunting through navigation code scattered across the wizard.

---

## Step 2 — The Wizard Reducer

```typescript
type WizardAction =
  | { type: "UPDATE_FIELD"; field: string; value: unknown }
  | { type: "NEXT_STEP" }
  | { type: "PREVIOUS_STEP" }
  | { type: "JUMP_TO_STEP"; stepId: string } // for "edit" links from the review step
  | { type: "RESTORE_DRAFT"; draft: WizardState };

function wizardReducer(state: WizardState, action: WizardAction): WizardState {
  switch (action.type) {
    case "UPDATE_FIELD":
      return {
        ...state,
        formData: { ...state.formData, [action.field]: action.value },
      };

    case "NEXT_STEP": {
      const currentStep = wizardSteps.find(
        (s) => s.id === state.currentStepId,
      )!;
      const nextStepId = currentStep.getNextStepId(state.formData);
      if (!nextStepId) return state; // already at the end

      return {
        ...state,
        currentStepId: nextStepId,
        visitedSteps: [...state.visitedSteps, nextStepId],
      };
    }

    case "PREVIOUS_STEP": {
      const newVisited = state.visitedSteps.slice(0, -1);
      const previousStepId = newVisited[newVisited.length - 1];
      if (!previousStepId) return state; // already at the first step

      return {
        ...state,
        currentStepId: previousStepId,
        visitedSteps: newVisited,
      };
    }

    case "JUMP_TO_STEP":
      // Used for "edit this section" links from a review/summary step
      return { ...state, currentStepId: action.stepId };

    case "RESTORE_DRAFT":
      return action.draft;

    default:
      return state;
  }
}

function useWizard() {
  const [state, dispatch] = useReducer(wizardReducer, {
    formData: {},
    currentStepId: wizardSteps[0].id,
    visitedSteps: [wizardSteps[0].id],
  });

  const currentStep = wizardSteps.find((s) => s.id === state.currentStepId)!;

  function goNext() {
    const errors = currentStep.validate(state.formData);
    if (Object.keys(errors).length > 0) {
      return { success: false, errors };
    }
    dispatch({ type: "NEXT_STEP" });
    return { success: true };
  }

  return { state, currentStep, dispatch, goNext };
}
```

**Key decision:** `visitedSteps` is tracked as a STACK (array), not just a single `previousStepId` field — this correctly handles the case where the user goes forward through a branch, then back, then forward through a DIFFERENT branch (because they changed an earlier answer). The back button always returns to wherever the user actually came from, not to a hardcoded "previous step in the master list."

---

## Step 3 — Accurate Progress Indicator for a Branching Flow

```typescript
// Naive approach (WRONG for branching wizards): "step 3 of 7" based on
// a FIXED total step count doesn't work when the actual path length
// varies depending on answers (the 'employed' branch has 4 steps total,
// the 'not employed' branch also has 4 steps, but a different wizard
// might have branches of different LENGTHS)

function useWizardProgress(state: WizardState) {
  // Compute the LIKELY total steps for the path the user is currently on,
  // by simulating forward from the current step using current form data
  const pathStepIds = useMemo(() => {
    const path: string[] = [];
    let stepId: string | null = wizardSteps[0].id;
    const visited = new Set<string>();

    while (stepId && !visited.has(stepId)) {
      visited.add(stepId);
      path.push(stepId);
      const step = wizardSteps.find(s => s.id === stepId)!;
      stepId = step.getNextStepId(state.formData); // uses CURRENT answers to predict the path
    }

    return path;
  }, [state.formData]);

  const currentIndex = pathStepIds.indexOf(state.currentStepId);

  return {
    currentStepNumber: currentIndex + 1,
    totalSteps: pathStepIds.length,
    percentage: ((currentIndex + 1) / pathStepIds.length) * 100,
  };
}

function ProgressIndicator({ state }) {
  const { currentStepNumber, totalSteps, percentage } = useWizardProgress(state);

  return (
    <div className="wizard-progress">
      <div className="progress-bar" style={{ width: `${percentage}%` }} />
      <span>Step {currentStepNumber} of {totalSteps}</span>
    </div>
  );
}
```

**Key decision:** the progress indicator RE-SIMULATES the likely path on every render, using current form data — this means if the user is on step 2 and hasn't yet answered the question that determines branching, the progress indicator shows a "best guess" total that may shift slightly once they answer. This is preferable to either a fixed, inaccurate total OR no progress indicator at all — most users intuitively understand that progress indicators in branching flows are approximate.

---

## Step 4 — Draft Auto-Save and Resume

```typescript
function useWizardDraftPersistence(
  state: WizardState,
  dispatch: React.Dispatch<WizardAction>,
) {
  const draftKey = "wizard-draft-v1"; // versioned, in case the schema changes later

  // Restore draft on mount
  useEffect(() => {
    try {
      const saved = localStorage.getItem(draftKey);
      if (saved) {
        const draft = JSON.parse(saved) as WizardState;
        dispatch({ type: "RESTORE_DRAFT", draft });
      }
    } catch {
      // Corrupted draft — ignore and start fresh
      localStorage.removeItem(draftKey);
    }
  }, []); // eslint-disable-line react-hooks/exhaustive-deps -- intentionally once

  // Save draft on every change, debounced
  useEffect(() => {
    const timer = setTimeout(() => {
      localStorage.setItem(draftKey, JSON.stringify(state));
    }, 500);
    return () => clearTimeout(timer);
  }, [state]);

  function clearDraft() {
    localStorage.removeItem(draftKey);
  }

  return { clearDraft };
}

// On successful final submission: clear the draft so the user doesn't
// see "resume your incomplete application" for a form they already submitted
function WizardContainer() {
  const { state, dispatch, goNext } = useWizard();
  const { clearDraft } = useWizardDraftPersistence(state, dispatch);

  async function handleSubmit() {
    await submitApplication(state.formData);
    clearDraft();
  }

  // ...
}
```

**Key decision:** the draft storage key is VERSIONED (`wizard-draft-v1`) — if the wizard's step structure or field names change in a future deployment, a stale draft from an old version won't be incorrectly restored into a wizard that no longer matches its shape (which could otherwise produce confusing partial states or runtime errors trying to render a step that no longer exists). Bumping the version effectively invalidates old drafts cleanly.

---

## Step 5 — Cross-Step Validation at Submit Time

```typescript
// Validation that depends on fields from MULTIPLE steps can't live in
// any single step's `validate` function — it needs to run against the
// FULL accumulated form data, typically right before final submission

function validateCrossStepRules(formData: Record<string, unknown>): Record<string, string> {
  const errors: Record<string, string> = {};

  // Example: start date (step 2) must be before end date (step 4)
  if (formData.startDate && formData.endDate) {
    if (new Date(formData.startDate as string) >= new Date(formData.endDate as string)) {
      errors.endDate = 'End date must be after start date';
    }
  }

  // Example: total allocation percentages (entered across multiple steps)
  // must sum to 100%
  const allocations = formData.allocations as Record<string, number> | undefined;
  if (allocations) {
    const total = Object.values(allocations).reduce((sum, v) => sum + v, 0);
    if (total !== 100) {
      errors.allocations = `Allocations must sum to 100% (currently ${total}%)`;
    }
  }

  return errors;
}

function ReviewStep({ formData, onEditStep }) {
  const crossStepErrors = useMemo(() => validateCrossStepRules(formData), [formData]);
  const hasErrors = Object.keys(crossStepErrors).length > 0;

  return (
    <div>
      <ReviewSummary formData={formData} onEditSection={onEditStep} />
      {hasErrors && (
        <div className="cross-step-errors">
          {Object.entries(crossStepErrors).map(([field, message]) => (
            <p key={field}>{message}</p>
          ))}
        </div>
      )}
      <button disabled={hasErrors} onClick={handleSubmit}>Submit Application</button>
    </div>
  );
}
```

**Key decision:** cross-step validation runs on the REVIEW step (the last step before submission), where the full accumulated `formData` is available — and any errors found here link back to the SPECIFIC earlier step that needs correction (`onEditStep`), rather than just blocking submission with a generic error. This "review and edit" pattern is standard for multi-step forms because it gives users a clear path to fix cross-cutting issues without restarting the entire flow.

---

## Step 6 — Preventing Accidental Data Loss

```typescript
function useUnsavedChangesWarning(hasUnsavedChanges: boolean) {
  useEffect(() => {
    function handleBeforeUnload(e: BeforeUnloadEvent) {
      if (hasUnsavedChanges) {
        e.preventDefault();
        e.returnValue = "";
      }
    }
    window.addEventListener("beforeunload", handleBeforeUnload);
    return () => window.removeEventListener("beforeunload", handleBeforeUnload);
  }, [hasUnsavedChanges]);
}

// Note: since draft auto-save (Step 4) already persists to localStorage,
// the user technically WON'T lose data on accidental refresh — but the
// warning is still valuable because:
// 1. It prevents confusion ("did my data save?")
// 2. It catches cases where localStorage is unavailable (private browsing
//    with storage disabled, storage quota exceeded)
// 3. It signals intent clearly rather than relying silently on a
//    recovery mechanism the user doesn't know exists
```

---

## State Management Checklist

```
☐ Wizard modeled as an explicit state machine (steps + transitions),
  not a raw currentStepIndex counter
☐ Branching logic lives in step definitions (getNextStepId), not scattered
  through navigation components
☐ Back navigation uses a visited-steps stack, correctly handling
  branch changes after going back
☐ Progress indicator accounts for variable path length in branching flows
☐ Draft persistence is versioned to handle schema changes safely
☐ Cross-step validation runs against full accumulated data before submit
☐ "Edit this section" links from the review step jump directly to the
  relevant step (JUMP_TO_STEP), not forcing a full re-walk from step 1
```

---

## Extension Ideas

```
- Save-and-resume via account (not just localStorage) — sync drafts
  across devices for logged-in users
- Step-level analytics (track drop-off rate per step to identify friction)
- A/B testable step ordering or copy
- Conditional field-level validation (not just step-level)
- File upload steps with progress tracking (see networking patterns)
- Multi-language support with step content driven by i18n keys
```

---

## 🔗 Related Topics

- [`patterns/04-controlled-uncontrolled.md`](../patterns/04-controlled-uncontrolled.md) — Form field patterns used within each step
- [`exercises/02-react-exercises.md`](../exercises/02-react-exercises.md) — Exercise 4.1 (simpler reducer-based form)
- [`security/04-auth-patterns.md`](../security/04-auth-patterns.md) — If the wizard requires authentication mid-flow

---

<div align="center">

**Next:** [`projects/08-authentication-system.md`](./08-authentication-system.md) →

</div>
