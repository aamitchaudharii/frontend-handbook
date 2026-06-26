# 03 — Debugging Strategies

> **"Debugging is not searching for a bug — it's generating and testing hypotheses. Every developer who stares at code hoping to spot the problem is guessing. Every developer who says 'if this assumption is true, I expect to see X' and then checks for X is doing science. The scientific method is the fastest way to debug."**

Debugging is a skill separate from writing code — one that can be systematically improved. Experienced developers debug faster not because they're luckier or more intuitive, but because they follow a process: isolate the problem, form a hypothesis, test it, update the hypothesis. This document covers the debugging mental model, systematic methodologies, specific techniques for frontend and React bugs, rubber duck debugging, and the discipline of eliminating possibilities efficiently.

---

## 📚 Table of Contents

1. [The Debugging Mental Model](#1-the-debugging-mental-model)
2. [The Scientific Debugging Method](#2-the-scientific-debugging-method)
3. [Isolation Strategies](#3-isolation-strategies)
4. [Binary Search Debugging (Bisection)](#4-binary-search-debugging-bisection)
5. [Common React Debugging Patterns](#5-common-react-debugging-patterns)
6. [Network and API Debugging](#6-network-and-api-debugging)
7. [CSS Debugging Strategies](#7-css-debugging-strategies)
8. [The Rubber Duck Method](#8-the-rubber-duck-method)
9. [Reading Error Messages Effectively](#9-reading-error-messages-effectively)
10. [Debugging with console Effectively](#10-debugging-with-console-effectively)
11. [Good Practices](#11-good-practices)
12. [Common Debugging Anti-Patterns](#12-common-debugging-anti-patterns)
13. [Interview-Level Explanation](#13-interview-level-explanation)

---

## 1. The Debugging Mental Model

```
THE THREE QUESTIONS:

1. WHAT IS ACTUALLY HAPPENING?
   (Not what you think is happening — what the program ACTUALLY does)

2. WHAT SHOULD BE HAPPENING?
   (What behavior is expected)

3. WHERE IS THE FIRST POINT OF DIVERGENCE?
   (The gap between actual and expected — finding THIS is the bug)

THE COMMON MISTAKE:
  Jumping from "there's a bug" to "I'll fix this line I'm looking at"
  without going through all three questions.

  Experienced debuggers spend more time on question 3 (isolation)
  than on question 2 (fixing) because finding the gap precisely
  makes the fix obvious. Bad debuggers fix symptoms without finding
  the root cause — the bug comes back.
```

---

## 2. The Scientific Debugging Method

```
STEPS:

1. REPRODUCE reliably
   A bug you can't reproduce consistently is nearly impossible to fix.
   Before anything else: find the exact steps to reproduce.
   "It happens sometimes" → find what condition makes "sometimes" become "always"

2. GATHER EVIDENCE
   Read the error message carefully (not just the first line)
   Check the browser console for errors AND warnings
   Check the network requests for failures
   Check React DevTools for unexpected state
   Do NOT start fixing until you understand the evidence

3. FORM A HYPOTHESIS
   "I believe the bug is caused by X because of evidence Y"
   Be specific: "the function is being called with undefined because..."
   not: "something is wrong with the data"

4. TEST THE HYPOTHESIS
   Design a minimal test that would CONFIRM OR REFUTE your hypothesis
   Add a breakpoint exactly where your hypothesis predicts the problem
   OR: add an assertion/console.log that would expose if your hypothesis is wrong

5. REVISE OR FIX
   CONFIRMED: fix the root cause
   REFUTED: eliminate this hypothesis, form a new one from new evidence
   The refuted hypothesis is NOT wasted time — it's eliminated possibility space

REPEAT until found.
```

---

## 3. Isolation Strategies

### Isolate the Component

```jsx
// When a bug exists in a complex page: isolate the component in a minimal environment
// Create a test file or Storybook story that renders ONLY the buggy component

// ❌ Debugging in context of the full app
// Full app → many providers, many side effects, hard to tell what's causing what

// ✅ Isolate:
function DebugEnvironment() {
  return (
    <TestProviders>
      {" "}
      {/* minimal providers: just what the component needs */}
      <BuggyComponent
        prop1="literal value"
        prop2={42}
        onAction={() => console.log("action called")}
      />
    </TestProviders>
  );
}

// If the bug disappears in isolation: the issue is in the context (providers, siblings)
// If the bug persists in isolation: the issue is in the component itself
// This single determination cuts the search space in half
```

### Isolate the Data

```javascript
// When the bug only appears with certain data: find the minimal data that reproduces it
const fullDataset = /* 1000 items */;

// Binary search for the problematic data:
// Try with first 500 items → bug? → try 250 → no bug? → try 375 → etc.
// Find the exact item(s) that trigger the bug

// Once found: inspect the problematic data
const buggyItem = { id: null, name: 'Test', price: -1 }; // id is null!
// Now you know: the bug is triggered by null id → validate/handle null ids
```

### Isolate the Timing

```javascript
// When a bug happens "sometimes" or "after a few seconds":
// Is it a race condition? A timing issue? A state accumulation problem?

// Add timestamps to isolate:
console.log(`[${performance.now().toFixed(0)}ms] fetchUser called`, userId);

// Sequence logging: give each operation a unique id
const reqId = Math.random().toString(36).slice(2, 6);
console.log(`[${reqId}] fetch start`);
fetch(url).then(() => console.log(`[${reqId}] fetch complete`));
// If two operations race: you can see which completes first via the reqId
```

---

## 4. Binary Search Debugging (Bisection)

When a bug exists somewhere in a large codebase or a sequence of changes, binary search finds it in O(log n) steps:

```
BISECTION FOR GIT COMMITS (finding when a bug was introduced):

git bisect start
git bisect bad                  # current commit: bug present
git bisect good v1.2.0          # v1.2.0: bug not present
# Git checks out a commit halfway between good and bad
# → Test: does the bug exist here?
git bisect good  # or: git bisect bad
# Git checks out next midpoint — repeat until the first bad commit is found
# For 100 commits: found in 7 steps (log2(100) ≈ 7)

AUTOMATED BISECT:
git bisect start
git bisect bad
git bisect good v1.2.0
git bisect run npm test          # git automatically runs tests at each midpoint
```

### Bisection for Code

```javascript
// When a 500-line function has a bug: comment out the second half
// If bug disappears: bug is in the second half
// Uncomment second half, comment out first half of the second half
// Repeat until you've narrowed to ~5 lines

// Comment-based bisection:
function processData(data) {
  const step1 = transform(data);
  const step2 = validate(step1);
  // ↓ COMMENTING OUT SECOND HALF TO BISECT
  // const step3 = format(step2);
  // const step4 = enrich(step3);
  // const step5 = finalize(step4);
  console.log("after step2:", step2); // inspect at bisection point
  return step2; // temporary
}

// Bug in step2 or earlier:
// Comment out step2, inspect step1
// etc. until narrowed down
```

---

## 5. Common React Debugging Patterns

### "My state update doesn't seem to work"

```javascript
// SYSTEMATIC APPROACH:
// 1. Verify setState IS being called
//    Add: console.log('setState called with:', newValue);
//    OR: breakpoint right before the setState call

// 2. Verify the new value is correct
//    console.log before setState: is the value you're setting correct?

// 3. Verify the component re-renders
//    Add: console.log('component rendered, count =', count);
//    If this doesn't log: the setState call is being missed

// 4. Verify React is not batching away your update
//    In React 18, ALL updates are batched — this is usually fine
//    If using flushSync: is it needed? Is it being applied correctly?

// 5. Verify you're not MUTATING state
//    const newItems = [...items];
//    newItems.push(item);
//    setItems(newItems); // ✅ new array reference
//
//    items.push(item);
//    setItems(items);   // ❌ same reference — React won't see the change!
```

### "My useEffect isn't running when I expect"

```javascript
// CHECKLIST:
// 1. Does it have a deps array? No array = runs every render
// 2. What ARE the dependencies? Are they what you think?
// 3. Are the dep values actually changing? Add console.log inside the effect
// 4. Are you comparing objects/arrays? Reference equality, not deep equality
//    (new object every render even if same content → effect runs every render)
// 5. Is the component actually re-rendering? Add a log at the top of the component

// Diagnostic effect: add temporarily to see when/why it runs
useEffect(() => {
  console.log("Effect ran. Deps:", { userId, filter }); // trace which dep changed
}, [userId, filter]);
```

### "My component renders with the wrong data"

```javascript
// TRACE THE DATA FLOW BACKWARD:
// Start at the consumer (the component showing wrong data)
// → Where does this prop come from? (parent → grandparent → ...)
// → At each level: is the data correct here? (console.log or DevTools)
// → The first level where data is WRONG is where the bug is

// React DevTools: click the component → check Props section
// Compare actual props vs what you expect
// Click the parent → check its state/props
// Repeat until you find the level where data first becomes wrong

// Often revealed: the data is correct in state but wrong when transformed:
const displayName = user.firstName + user.lastName; // missing space!
// Correct in DevTools (user object fine), wrong in the UI (formatting bug)
```

---

## 6. Network and API Debugging

```javascript
// SYSTEMATIC NETWORK DEBUGGING:

// 1. Check the request WAS sent
//    Network panel → XHR/Fetch filter → is the request there?
//    Not there: the fetch/axios call isn't being reached
//    → Breakpoint at the API call site

// 2. Check the request is correct
//    Click the request → Headers tab
//    URL: correct? Query params correct?
//    Request headers: Authorization present? Content-Type correct?
//    Request body: correct format and content?

// 3. Check the response
//    Status code: 200? 401? 500?
//    Response tab: full response body
//    Preview tab: formatted view (JSON parsed)

// 4. Check the response handling
//    Is the response being parsed correctly?
//    Is the data being stored in the right state?
//    Are you handling errors (non-200 responses)?

// COMMON ISSUES:
//   401: missing/expired auth token
//   403: user doesn't have permission for this operation
//   404: URL is wrong (typo, wrong version, wrong path)
//   422: request body invalid (check the error details in response body)
//   500: server bug — look at server logs
//   CORS error: cross-origin request blocked (see networking/04-cors.md)
//   net::ERR_FAILED: network down, or request aborted (AbortController)
```

### Intercepting Requests for Debugging

```javascript
// Temporarily wrap fetch to log all requests/responses
const originalFetch = window.fetch;
window.fetch = async function debugFetch(...args) {
  const [url, options] = args;
  console.group(`Fetch: ${options?.method ?? "GET"} ${url}`);
  console.log("Options:", options);

  try {
    const response = await originalFetch.apply(this, args);
    const clone = response.clone();
    const body = await clone.json().catch(() => clone.text());
    console.log(`Response ${response.status}:`, body);
    console.groupEnd();
    return response;
  } catch (err) {
    console.error("Fetch error:", err);
    console.groupEnd();
    throw err;
  }
};

// Remember to restore: window.fetch = originalFetch;
// Or use a browser extension like DevTools' Network tab (which is better)
```

---

## 7. CSS Debugging Strategies

```css
/* CSS bug patterns and systematic fixes */

/* STRATEGY 1: Add a bright border to suspect elements */
.suspicious-element {
  outline: 2px solid red !important; /* doesn't affect layout, always visible */
}
/* Shows: is the element even rendering? Is it in the right place? */

/* STRATEGY 2: Use DevTools live editing */
/* Elements panel → Styles → click any value → change it */
/* See immediate effect without file change */
/* Once found: copy the working value to your source */

/* STRATEGY 3: Check computed styles */
/* Elements → Computed tab → find the property */
/* Shows: FINAL computed value (after cascade, inheritance) */
/* Source: shows WHICH rule is providing the value */
/* Helps: "why is this 32px when I set 16px?" → Computed shows the winner */

/* STRATEGY 4: Specificity issues */
/* Styles panel → hover a struck-through property → shows which rule wins */
/* Fix: increase your rule's specificity, or move it later in the stylesheet */

/* STRATEGY 5: Layout debugging */
/* Add: * { outline: 1px solid rgba(255, 0, 0, 0.2); } */
/* Shows all element boundaries — great for spacing/alignment issues */
```

---

## 8. The Rubber Duck Method

```
WHAT IT IS:
  Explain your code/problem to an inanimate object (literally a rubber duck)
  The act of articulating the problem forces you to think precisely

WHY IT WORKS:
  Most bugs exist because we hold a wrong assumption silently
  When you EXPLAIN code out loud, you must state assumptions explicitly
  Often, you hear yourself say "and here it does X... wait, that's wrong"

HOW TO DO IT:
  1. Pick up the rubber duck (or explain to a colleague, a blank notepad, yourself)
  2. "This component is supposed to show the user's name"
  3. "It gets the name from the `user` prop"
  4. "The `user` prop comes from the parent's state"
  5. "The parent state is set when... wait, the parent state is set in a useEffect
     that has userId in deps, and userId comes from the URL params, and I'm using
     useParams... oh. I forgot to add the React Router v6 useParams import.
     That's why user is undefined."

  The duck didn't say anything. The explanation revealed the bug.

WHEN TO USE:
  After 15-20 minutes of being stuck
  Before asking a colleague (often solves it before you reach them)
  When reading the code for the 5th time without seeing anything
```

---

## 9. Reading Error Messages Effectively

```javascript
// ANATOMY OF A USEFUL ERROR MESSAGE:
//
// TypeError: Cannot read properties of undefined (reading 'name')
//   at UserProfile (UserProfile.jsx:42:18)        ← exact file and line
//   at renderWithHooks (react-dom.development.js) ← React internals (ignore)
//   at updateFunctionComponent (react-dom.development.js)
//   at updateElement (react-dom.development.js)
//   at reconcileChildren (react-dom.development.js)
//   at updateHostRoot (react-dom.development.js)
//
// READING:
// - Error type: TypeError
// - Error message: "Cannot read properties of undefined (reading 'name')"
//   → Something is undefined, and we're trying to access `.name` on it
// - Ignore the React internal frames (react-dom.development.js)
// - Your code is the FIRST frame that's YOUR file: UserProfile.jsx:42:18
// - Line 42, column 18: go there
//
// WHAT TO DO:
// 1. Go to the exact file:line
// 2. Look at what's at column 18 — likely: user.name, where `user` is undefined
// 3. Trace backward: where does `user` come from? Is it a prop? State?
// 4. Add: console.log('user value:', user); before line 42

// COMMON PATTERNS:
// "Cannot read properties of undefined (reading 'X')"
//   → obj.X where obj is undefined → obj is not initialized yet or prop is missing

// "Objects are not valid as a React child"
//   → You're rendering an object directly: {user} instead of {user.name}
//   → Or: Promise rendered instead of awaited

// "Each child in a list should have a unique 'key' prop"
//   → Add key={item.id} to the outermost element in .map()

// "Too many re-renders"
//   → setState being called inside render or in effect without deps
//   → OR: setState called during render (not in an event handler or effect)
```

---

## 10. Debugging with console Effectively

```javascript
// BEYOND console.log:

// 1. LABELED LOGGING
console.log("userId:", userId, "user:", user); // add context to values
// vs: console.log(userId, user); // which value is which?

// 2. CONSOLE.TABLE for arrays/objects
console.table(users); // formatted table with columns for each property
console.table(users, ["id", "name"]); // only show specific columns

// 3. CONSOLE.GROUP for related logs
console.group("UserProfile render");
console.log("props:", props);
console.log("state:", { count, isOpen });
console.log("computed:", { displayName, isAdmin });
console.groupEnd();
// Collapsible in console — keeps related logs together

// 4. CONSOLE.TIME for performance
console.time("transform");
const result = expensiveTransform(data);
console.timeEnd("transform"); // "transform: 47.3ms"

// 5. CONSOLE.TRACE to see call stack without breakpoint
function suspiciousFunction() {
  console.trace("called from"); // prints full call stack at this point
}

// 6. CONSOLE.COUNT to track invocations
function frequentlyCalledFn() {
  console.count("frequentlyCalledFn"); // logs "frequentlyCalledFn: 1", "...2", etc.
}

// 7. CONSOLE.ASSERT for conditional debugging
console.assert(user.id !== undefined, "BUG: user.id is undefined", user);
// Only logs if condition is FALSE — zero noise when things are working

// 8. CONDITIONAL LOGGING without removing logs
const DEBUG = process.env.NODE_ENV === "development";
if (DEBUG) console.log("debug:", value);
// Or use a debug module: import debug from 'debug'; const log = debug('app:user');
```

---

## 11. Good Practices

### ✅ Reproduce before investigating

```
Never spend more than 2 minutes guessing at a bug you can't reproduce.
Invest the time to find reliable reproduction steps first.
A reliably reproducible bug is a solvable bug.
```

### ✅ Use the simplest possible reproduction case

```javascript
// When a bug is in the full app: strip it down
// Create an isolated sandbox (CodeSandbox, local test file)
// Add back complexity until the bug appears
// The minimum reproduction reveals the actual cause
```

### ✅ Document your debugging process for future self

```javascript
// When finding a particularly tricky bug: write a comment about it
// The "why" of a fix is more valuable than the "what"

// ❌ Silent fix
if (user?.id) {
  // ...
}

// ✅ Documented fix
// user can be undefined on first render before the auth hook initializes.
// Without optional chaining, this throws before the spinner is shown.
// See: https://github.com/org/repo/issues/123
if (user?.id) {
  // ...
}
```

---

## 12. Common Debugging Anti-Patterns

### ❌ Random Code Changes

```
Approach: "I'll just change a few things and see if it fixes it"
Problem:  If it works: you don't know WHY it works — the real bug may still be there
          If it doesn't: you've added noise, harder to read, may break other things
          May introduce new bugs while "fixing" the apparent bug

Alternative: Hypothesis → Test → Confirm → Fix
```

### ❌ Fixing Symptoms Instead of Root Causes

```javascript
// ❌ Fixing the symptom
function UserProfile({ user }) {
  if (!user.name) return <span>Unknown</span>; // symptom fix
  return <span>{user.name}</span>;
}
// Root cause: user.name is undefined because the API response shape changed
// Fix: handle the API change properly, or update the type

// ❌ Catching errors silently
try {
  processData(data);
} catch (e) {
  // silently swallow the error (the bug is now invisible)
}
```

### ❌ Debugging Under Pressure (Rushing)

```
Pressure to fix quickly → skip the reproduction step
→ spend 3 hours in the wrong part of the codebase
→ fix the wrong thing
→ bug reappears or new bug introduced

Better: 5 minutes spent reliably reproducing → 30 minutes to fix
vs: 3 hours of frustrated guessing
```

---

## 13. Interview-Level Explanation

> **"How do you approach debugging a complex frontend bug?"**

**Strong answer:**

> "I follow a systematic process rather than intuition or random changes, because systematic debugging is reliably faster.
>
> The first step is reliable reproduction. I don't start investigating until I can reproduce the bug on demand. If it's intermittent, I find the condition that makes it consistent. A bug I can trigger reliably is much faster to debug than one that 'sometimes happens.'
>
> With a reliable reproduction, I gather evidence first: the exact error message (including stack trace), the network request that failed (if any), the React DevTools state of the component at the moment of the bug. I'm reading, not guessing.
>
> Then I form a specific, testable hypothesis. Not 'something is wrong with the data' but 'the user object is undefined at line 42 because the useEffect that fetches it hasn't completed before the component tries to render user.name.' I design a minimal test for that: a breakpoint at line 42, a console.log of user before that line.
>
> If my hypothesis is confirmed: fix the root cause. If it's refuted: I've still narrowed the problem — I know what it ISN'T, and I form a new hypothesis from what I learned.
>
> For React specifically: I use React DevTools Components panel to inspect actual props and state at the moment of the bug. The 'Why did this render?' in the Profiler tells me exactly what triggered a re-render. For state updates that don't seem to take effect, I check whether state is being mutated directly (same reference → React won't detect the change).
>
> For intermittent or hard-to-find bugs, I use bisection: comment out half the suspicious code (or use git bisect for regression), check if the bug persists, narrow to the other half. Log2(500) is about 9 steps to isolate a bug in 500 lines. This is dramatically faster than reading 500 lines looking for the problem.
>
> The common mistake I avoid is fixing symptoms instead of root causes. If I add a null check and move on, the real cause is still there — it'll surface in a different form. I always ask 'why is this value null? Where does it come from? What assumption is wrong?'"
