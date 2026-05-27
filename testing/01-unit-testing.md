# 01 — Unit Testing

> **"A unit test is a specification written in code. The best unit tests don't just catch bugs — they document exactly what a function is supposed to do, in every case that matters, and tell you immediately when it stops doing that."**

Unit tests are the foundation of a healthy test suite. They're the fastest, cheapest, and most surgical tests you can write. When a unit test fails, you know exactly which function broke and why — no browser, no network, no ambiguity. This document covers unit testing philosophy, Jest and Vitest setup, the full range of mocking techniques, testing async code, property-based testing, and the thinking behind what to unit test and what not to.

---

## 📚 Table of Contents

1. [What Unit Testing Is](#1-what-unit-testing-is)
2. [Test Structure — Arrange, Act, Assert](#2-test-structure--arrange-act-assert)
3. [Jest and Vitest — Setup and Configuration](#3-jest-and-vitest--setup-and-configuration)
4. [Matchers — The Full Reference](#4-matchers--the-full-reference)
5. [Mocking — Strategies and Techniques](#5-mocking---strategies-and-techniques)
6. [Mocking Modules](#6-mocking-modules)
7. [Testing Async Functions](#7-testing-async-functions)
8. [Testing with Fake Timers](#8-testing-with-fake-timers)
9. [Testing Error Cases](#9-testing-error-cases)
10. [Testing Pure Functions](#10-testing-pure-functions)
11. [Test Coverage — What It Means and Doesn't](#11-test-coverage--what-it-means-and-doesnt)
12. [Property-Based Testing](#12-property-based-testing)
13. [Test Organization and Naming](#13-test-organization-and-naming)
14. [Good Practices](#14-good-practices)
15. [Bad Practices](#15-bad-practices)
16. [Common Mistakes](#16-common-mistakes)
17. [Interview-Level Explanation](#17-interview-level-explanation)
18. [Exercises](#18-exercises)

---

## 1. What Unit Testing Is

A **unit test** verifies the behavior of the smallest meaningful unit of code — typically a single function, class, or module — in complete isolation from its dependencies.

```
Unit under test: formatPrice(amount, currency)

Unit tests ask:
  formatPrice(29.99, 'USD') → '$29.99'
  formatPrice(0, 'USD')     → '$0.00'
  formatPrice(-5, 'USD')    → '-$5.00'
  formatPrice(1000, 'EUR')  → '€1,000.00'
  formatPrice(NaN, 'USD')   → throws TypeError
  formatPrice(29.99, 'XXX') → throws RangeError (unknown currency)

Each test: one input → expected output
No network, no DOM, no other modules
Runs in < 1ms
```

### The Unit Test Contract

A good unit test acts as a specification:
- **What** the function does (stated in the test name)
- **Given** these inputs (the arrange phase)
- **When** this action happens (the act phase)
- **Then** this is the expected result (the assert phase)

A test suite that reads cleanly is one where you can determine the function's full contract just by reading the test names — without looking at the implementation.

---

## 2. Test Structure — Arrange, Act, Assert

Every test follows the same three-phase structure:

```javascript
test('formatPrice returns correctly formatted USD price', () => {
  // ARRANGE — set up the inputs and expected outputs
  const amount   = 29.99;
  const currency = 'USD';

  // ACT — call the function under test
  const result = formatPrice(amount, currency);

  // ASSERT — verify the output matches expectation
  expect(result).toBe('$29.99');
});
```

### Given-When-Then Variant

```javascript
// Given-When-Then is a more explicit style, useful for complex scenarios
test('given an invalid currency code, when formatting price, then throws RangeError', () => {
  // Given
  const amount   = 29.99;
  const currency = 'INVALID';

  // When / Then
  expect(() => formatPrice(amount, currency)).toThrow(RangeError);
});
```

### describe Blocks for Organization

```javascript
describe('formatPrice', () => {
  describe('USD formatting', () => {
    test('formats positive amounts with $ prefix', () => {
      expect(formatPrice(29.99, 'USD')).toBe('$29.99');
    });

    test('formats zero with two decimal places', () => {
      expect(formatPrice(0, 'USD')).toBe('$0.00');
    });

    test('formats negative amounts correctly', () => {
      expect(formatPrice(-5.5, 'USD')).toBe('-$5.50');
    });

    test('adds thousands separator for large amounts', () => {
      expect(formatPrice(1_234_567.89, 'USD')).toBe('$1,234,567.89');
    });
  });

  describe('error handling', () => {
    test('throws TypeError for NaN amount', () => {
      expect(() => formatPrice(NaN, 'USD')).toThrow(TypeError);
    });

    test('throws RangeError for unknown currency code', () => {
      expect(() => formatPrice(10, 'XXX')).toThrow(RangeError);
    });
  });
});
```

---

## 3. Jest and Vitest — Setup and Configuration

### Jest (Node.js / legacy projects)

```bash
npm install --save-dev jest @types/jest ts-jest
```

```javascript
// jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  preset: 'ts-jest',            // TypeScript support
  testEnvironment: 'node',      // 'jsdom' for DOM-dependent code
  collectCoverageFrom: [
    'src/**/*.{ts,js}',
    '!src/**/*.d.ts',
    '!src/index.ts',            // entry points rarely need direct tests
  ],
  coverageThresholds: {
    global: {
      branches:   80,
      functions:  80,
      lines:      80,
      statements: 80,
    },
  },
  // Optional: display individual test results
  verbose: true,
};
```

### Vitest (Vite projects — recommended for modern setups)

```bash
npm install --save-dev vitest @vitest/coverage-v8
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',         // or 'jsdom', 'happy-dom'
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      thresholds: {
        branches:   80,
        functions:  80,
        lines:      80,
        statements: 80,
      },
    },
    globals: true,               // no need to import describe, test, expect
  },
});
```

### Key Differences: Jest vs Vitest

| Feature | Jest | Vitest |
|---|---|---|
| Speed | Moderate | 2-10× faster (native ESM) |
| Config | Separate `jest.config.js` | In `vite.config.ts` |
| TypeScript | Needs `ts-jest` or Babel | Native |
| ESM support | Requires workarounds | Native |
| Watch mode | `jest --watch` | `vitest --watch` (faster HMR) |
| API compatibility | Original | ~100% compatible |

---

## 4. Matchers — The Full Reference

```javascript
// EQUALITY
expect(value).toBe(exact);           // Object.is comparison (strict)
expect(value).toEqual(deep);         // deep equality for objects/arrays
expect(value).toStrictEqual(deep);   // toEqual + checks undefined properties

// TRUTHINESS
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();
expect(value).toBeNaN();

// NUMBERS
expect(num).toBeGreaterThan(n);
expect(num).toBeGreaterThanOrEqual(n);
expect(num).toBeLessThan(n);
expect(num).toBeLessThanOrEqual(n);
expect(num).toBeCloseTo(n, numDigits); // floating-point comparison

// STRINGS
expect(str).toMatch('substring');
expect(str).toMatch(/regex/);
expect(str).toContain('substring'); // also works for arrays

// ARRAYS
expect(arr).toHaveLength(n);
expect(arr).toContain(item);         // item present (reference equality)
expect(arr).toContainEqual(obj);     // deep equal item present
expect(arr).toEqual(expect.arrayContaining([a, b])); // contains at least these

// OBJECTS
expect(obj).toHaveProperty('key');
expect(obj).toHaveProperty('key', value);
expect(obj).toHaveProperty('a.b.c'); // nested path
expect(obj).toMatchObject({ partial: 'match' }); // subset match

// FUNCTIONS / ERRORS
expect(fn).toThrow();
expect(fn).toThrow(Error);
expect(fn).toThrow('message');
expect(fn).toThrow(/regex/);

// ASYNC
await expect(promise).resolves.toBe(value);
await expect(promise).rejects.toThrow('error message');

// FUNCTION CALLS (mocks/spies)
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(n);
expect(mockFn).toHaveBeenCalledWith(arg1, arg2);
expect(mockFn).toHaveBeenLastCalledWith(arg1, arg2);
expect(mockFn).toHaveBeenNthCalledWith(n, arg1, arg2);
expect(mockFn).toHaveReturnedWith(value);
expect(mockFn).toHaveReturnedTimes(n);

// NEGATION (any matcher can be negated)
expect(value).not.toBe(other);
expect(fn).not.toThrow();
expect(arr).not.toContain(item);
```

### Asymmetric Matchers (for Partial Matching)

```javascript
// When you don't care about exact value
expect(result).toEqual({
  id:        expect.any(Number),            // any number
  name:      expect.any(String),            // any string
  createdAt: expect.any(Date),              // any Date
  userId:    expect.stringMatching(/^u_/),  // string matching regex
  tags:      expect.arrayContaining(['a']), // array containing at least 'a'
});

// Useful for objects with generated/timestamp fields
expect(createUser('Alice')).toEqual({
  name: 'Alice',
  id:   expect.any(String),       // UUID — we don't know the exact value
  createdAt: expect.any(Number),  // timestamp
});
```

---

## 5. Mocking — Strategies and Techniques

Mocking replaces real dependencies with controlled substitutes, enabling isolation.

### Mock Functions (Spies)

```javascript
// jest.fn() creates a mock function that records calls
const mockCallback = jest.fn();

// Call it
mockCallback('hello', 42);
mockCallback('world');

// Assert on calls
expect(mockCallback).toHaveBeenCalledTimes(2);
expect(mockCallback).toHaveBeenCalledWith('hello', 42);
expect(mockCallback).toHaveBeenLastCalledWith('world');

// Access call data directly
console.log(mockCallback.mock.calls);
// [['hello', 42], ['world']]

console.log(mockCallback.mock.results);
// [{ type: 'return', value: undefined }, ...]
```

### Controlling Return Values

```javascript
// Return a fixed value
const mockGetUser = jest.fn().mockReturnValue({ id: 1, name: 'Alice' });
mockGetUser(); // → { id: 1, name: 'Alice' }
mockGetUser(); // → { id: 1, name: 'Alice' } (same every time)

// Return different values on successive calls
const mockIterator = jest.fn()
  .mockReturnValueOnce('first')
  .mockReturnValueOnce('second')
  .mockReturnValue('default'); // fallback for all subsequent calls
mockIterator(); // 'first'
mockIterator(); // 'second'
mockIterator(); // 'default'
mockIterator(); // 'default'

// Return a resolved/rejected Promise
const mockFetch = jest.fn()
  .mockResolvedValue({ data: 'success' });
  // same as: .mockReturnValue(Promise.resolve({ data: 'success' }))

const mockFetchFail = jest.fn()
  .mockRejectedValue(new Error('Network error'));

// Custom implementation
const mockProcess = jest.fn().mockImplementation((input) => {
  if (input > 0) return input * 2;
  throw new Error('Input must be positive');
});
```

### Spying on Real Methods

```javascript
// Spy on a method without replacing it
const spy = jest.spyOn(console, 'log');

doSomethingThatLogsToConsole();

expect(spy).toHaveBeenCalledWith('expected log message');

// Restore original implementation after test
spy.mockRestore();

// Or replace implementation temporarily
const spy = jest.spyOn(mathLib, 'random').mockReturnValue(0.5);
expect(generateId()).toBe('id_0.5'); // deterministic with mocked random
spy.mockRestore();
```

---

## 6. Mocking Modules

### Mocking an Entire Module

```javascript
// math.ts
export function add(a, b) { return a + b; }
export function multiply(a, b) { return a * b; }

// calculator.test.ts
import { calculator } from './calculator';
import * as math from './math';

// Auto-mock: replaces all exports with jest.fn()
jest.mock('./math');
// Now: math.add is jest.fn(), math.multiply is jest.fn()

test('calculator calls add correctly', () => {
  (math.add as jest.Mock).mockReturnValue(5);

  const result = calculator.addNumbers(2, 3);

  expect(math.add).toHaveBeenCalledWith(2, 3);
  expect(result).toBe(5);
});
```

### Manual Module Mocks

```javascript
// __mocks__/fs.ts  (placed next to node_modules for node modules)
// or src/__mocks__/api.ts for local modules

// __mocks__/api.ts
export const fetchUser = jest.fn().mockResolvedValue({
  id: '1',
  name: 'Alice',
  email: 'alice@example.com',
});

export const updateUser = jest.fn().mockResolvedValue({ success: true });
```

```javascript
// In test file: activate the manual mock
jest.mock('./api'); // uses __mocks__/api.ts automatically

import { fetchUser } from './api';

test('loads user data', async () => {
  const user = await fetchUser('1');
  expect(user.name).toBe('Alice'); // from mock
});
```

### Partial Mocks

```javascript
// Mock only some exports, keep the rest real
jest.mock('./utils', () => ({
  ...jest.requireActual('./utils'), // import real implementations
  generateId: jest.fn().mockReturnValue('test-id'), // mock only this one
}));
```

### Mocking Third-Party Libraries

```javascript
// Mock axios
jest.mock('axios', () => ({
  get: jest.fn().mockResolvedValue({ data: { users: [] } }),
  post: jest.fn().mockResolvedValue({ data: { id: 'new-id' } }),
}));

// Mock date-fns
jest.mock('date-fns', () => ({
  ...jest.requireActual('date-fns'),
  format: jest.fn().mockReturnValue('2024-01-15'),
}));
```

---

## 7. Testing Async Functions

### Async/Await in Tests

```javascript
// Always mark test as async when testing async code
test('fetchUser returns user data', async () => {
  const user = await fetchUser('42');

  expect(user).toEqual({
    id:   '42',
    name: 'Alice',
  });
});

// Test rejection
test('fetchUser rejects with unknown id', async () => {
  await expect(fetchUser('unknown')).rejects.toThrow('User not found');
  // or:
  await expect(fetchUser('unknown')).rejects.toThrowError(/not found/i);
});
```

### Testing with Promises (alternative style)

```javascript
// Return the promise — Jest waits for it
test('fetchUser returns user data', () => {
  return fetchUser('42').then(user => {
    expect(user.name).toBe('Alice');
  });
});

// Using .resolves / .rejects
test('fetchUser resolves with user', async () => {
  await expect(fetchUser('42')).resolves.toMatchObject({ name: 'Alice' });
});

test('fetchUser rejects for unknown id', async () => {
  await expect(fetchUser('unknown')).rejects.toThrow('User not found');
});
```

### Testing Race Conditions

```javascript
test('only last request wins (debounce)', async () => {
  const results = [];

  const debounced = debounce(async (value) => {
    const result = await fetchSearchResults(value);
    results.push(result);
  }, 300);

  jest.useFakeTimers();

  debounced('a');
  debounced('ab');
  debounced('abc');  // only this one should actually fire

  jest.advanceTimersByTime(300);
  await Promise.resolve(); // flush microtasks

  expect(fetchSearchResults).toHaveBeenCalledTimes(1);
  expect(fetchSearchResults).toHaveBeenCalledWith('abc');
});
```

---

## 8. Testing with Fake Timers

Real timers (`setTimeout`, `setInterval`, `Date`) make tests slow and non-deterministic. Jest provides fake timers that you control.

### Basic Fake Timer Usage

```javascript
test('debounce delays execution', () => {
  jest.useFakeTimers();
  const mockFn = jest.fn();

  const debounced = debounce(mockFn, 500);

  debounced('first call');
  debounced('second call');
  debounced('third call');

  // Not called yet (timer not advanced)
  expect(mockFn).not.toHaveBeenCalled();

  // Advance time past the debounce delay
  jest.advanceTimersByTime(500);

  // Now called once with the last value
  expect(mockFn).toHaveBeenCalledTimes(1);
  expect(mockFn).toHaveBeenCalledWith('third call');

  jest.useRealTimers(); // restore after test
});
```

### Fake Timers with Cleanup

```javascript
describe('timer-based functionality', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.clearAllTimers();  // cancel pending timers
    jest.useRealTimers();   // restore real timers
  });

  test('throttle executes at most once per interval', () => {
    const mockFn = jest.fn();
    const throttled = throttle(mockFn, 1000);

    throttled(); // fires immediately
    throttled(); // ignored (within interval)
    throttled(); // ignored

    expect(mockFn).toHaveBeenCalledTimes(1);

    jest.advanceTimersByTime(1000);
    throttled(); // fires again (interval passed)

    expect(mockFn).toHaveBeenCalledTimes(2);
  });

  test('setInterval executes callback repeatedly', () => {
    const callback = jest.fn();
    setInterval(callback, 1000);

    jest.advanceTimersByTime(3500);

    expect(callback).toHaveBeenCalledTimes(3);
  });
});
```

### Mocking `Date`

```javascript
// Option 1: fake timers (also mocks Date.now())
jest.useFakeTimers();
jest.setSystemTime(new Date('2024-01-15T10:00:00Z'));

const timestamp = getTimestamp();
expect(timestamp).toBe('2024-01-15T10:00:00.000Z');

jest.useRealTimers();

// Option 2: spy on Date
jest.spyOn(global, 'Date').mockImplementation(
  () => new Date('2024-01-15T10:00:00Z') as unknown as string
);

// Option 3: inject date as parameter (better design — no mocking needed)
function getFormattedDate(date = new Date()) {
  return date.toISOString().split('T')[0];
}

test('formats date correctly', () => {
  // No mocking needed — inject the date
  expect(getFormattedDate(new Date('2024-01-15'))).toBe('2024-01-15');
});
```

---

## 9. Testing Error Cases

Error testing is often neglected but critical for robust software.

```javascript
// Synchronous errors
test('divideBy throws on zero divisor', () => {
  expect(() => divideBy(10, 0)).toThrow();
  expect(() => divideBy(10, 0)).toThrow(Error);
  expect(() => divideBy(10, 0)).toThrow('Cannot divide by zero');
  expect(() => divideBy(10, 0)).toThrow(/divide by zero/i);
});

// Check error type specifically
test('throws RangeError for out-of-bounds index', () => {
  expect(() => getElement(arr, 100)).toThrowError(RangeError);
});

// Custom error classes
class ValidationError extends Error {
  constructor(message: string, public field: string) {
    super(message);
    this.name = 'ValidationError';
  }
}

test('throws ValidationError with field information', () => {
  let thrownError;
  try {
    validateForm({ email: 'invalid' });
  } catch (err) {
    thrownError = err;
  }
  expect(thrownError).toBeInstanceOf(ValidationError);
  expect(thrownError.field).toBe('email');
  expect(thrownError.message).toMatch(/invalid email/i);
});

// Async errors
test('async function rejects with specific error', async () => {
  await expect(fetchUser('not-found')).rejects.toThrow('User not found');
  await expect(fetchUser('not-found')).rejects.toBeInstanceOf(NotFoundError);
});
```

---

## 10. Testing Pure Functions

Pure functions (same input → same output, no side effects) are the easiest to unit test. Test the full contract: happy paths, edge cases, and error cases.

```javascript
// The function under test
function calculateDiscount(price, discountPercent, minOrderValue = 0) {
  if (typeof price !== 'number' || typeof discountPercent !== 'number') {
    throw new TypeError('price and discountPercent must be numbers');
  }
  if (price < 0 || discountPercent < 0 || discountPercent > 100) {
    throw new RangeError('Invalid price or discount range');
  }
  if (price < minOrderValue) return 0;
  return Math.round(price * (discountPercent / 100) * 100) / 100;
}

describe('calculateDiscount', () => {
  // Happy path
  test('calculates 10% discount on $100', () => {
    expect(calculateDiscount(100, 10)).toBe(10);
  });

  test('calculates 25% discount with rounding', () => {
    expect(calculateDiscount(33.33, 25)).toBe(8.33);
  });

  // Edge cases
  test('returns 0 for 0% discount', () => {
    expect(calculateDiscount(100, 0)).toBe(0);
  });

  test('returns full price for 100% discount', () => {
    expect(calculateDiscount(100, 100)).toBe(100);
  });

  test('returns 0 when price is below minimum order value', () => {
    expect(calculateDiscount(50, 10, 100)).toBe(0);
  });

  test('applies discount when price meets minimum order value', () => {
    expect(calculateDiscount(100, 10, 100)).toBe(10);
  });

  // Error cases
  test('throws TypeError for non-numeric price', () => {
    expect(() => calculateDiscount('100', 10)).toThrow(TypeError);
  });

  test('throws RangeError for negative price', () => {
    expect(() => calculateDiscount(-10, 10)).toThrow(RangeError);
  });

  test('throws RangeError for discount > 100', () => {
    expect(() => calculateDiscount(100, 110)).toThrow(RangeError);
  });
});
```

---

## 11. Test Coverage — What It Means and Doesn't

### Types of Coverage

```
Statement coverage: % of statements executed by tests
Branch coverage:    % of if/else branches taken
Function coverage:  % of functions called
Line coverage:      % of lines executed

Example:
function example(x) {
  if (x > 0) {        ← branch: x > 0
    return 'positive'; ← statement
  } else {
    return 'non-positive'; ← statement
  }
}

With only: test('positive', () => expect(example(1)).toBe('positive'))
  Statement coverage: 50% (else branch not reached)
  Branch coverage:    50% (else branch not taken)
  Function coverage:  100% (function was called)
  Line coverage:      75%
```

### What Coverage Tells You

```
High coverage (> 80%) means:
  ✓ Most code is exercised during tests
  ✓ Likely to catch obvious regressions

High coverage does NOT mean:
  ✗ Tests are meaningful (can have 100% coverage with no assertions)
  ✗ Edge cases are handled
  ✗ Error cases are tested
  ✗ The code does the right thing (tests might have wrong expectations)
```

### Coverage Configuration

```javascript
// jest.config.js
module.exports = {
  coverageThresholds: {
    global: {
      branches:   70, // lower for branches — they're hard to cover fully
      functions:  80,
      lines:      80,
      statements: 80,
    },
    // File-level thresholds
    './src/utils/': {
      branches:  90, // critical utilities need higher coverage
      functions: 95,
    },
  },
  // Exclude from coverage
  coveragePathIgnorePatterns: [
    '/node_modules/',
    '/dist/',
    'index.ts',            // re-export files
    '\\.stories\\.tsx?$',  // Storybook stories
    '\\.d\\.ts$',          // type declaration files
  ],
};
```

---

## 12. Property-Based Testing

Property-based testing generates random inputs and verifies that properties (invariants) hold for all of them, rather than testing specific examples.

```bash
npm install --save-dev fast-check
```

### Basic Property Test

```javascript
import fc from 'fast-check';

// Instead of testing specific cases:
test('add(2,3) = 5, add(0,1) = 1, ...', () => {
  expect(add(2, 3)).toBe(5);
  expect(add(0, 1)).toBe(1);
});

// Test the PROPERTY that must hold for ALL inputs:
test('add is commutative for all integers', () => {
  fc.assert(
    fc.property(
      fc.integer(),  // generate random integers
      fc.integer(),
      (a, b) => {
        // Property: add(a,b) === add(b,a) for any a and b
        expect(add(a, b)).toBe(add(b, a));
      }
    )
  );
});

test('add is associative for all integers', () => {
  fc.assert(
    fc.property(
      fc.integer(), fc.integer(), fc.integer(),
      (a, b, c) => {
        expect(add(add(a, b), c)).toBe(add(a, add(b, c)));
      }
    )
  );
});
```

### Finding Edge Cases Automatically

```javascript
test('sort always produces sorted output', () => {
  fc.assert(
    fc.property(
      fc.array(fc.integer()),
      (arr) => {
        const sorted = mySort([...arr]);
        // Property: every element should be <= the next element
        for (let i = 0; i < sorted.length - 1; i++) {
          expect(sorted[i]).toBeLessThanOrEqual(sorted[i + 1]);
        }
      }
    )
  );
  // fast-check will try hundreds of random arrays
  // including: [], [1], [1,1,1], negative numbers, large arrays
});

test('parse(serialize(x)) === x (round-trip property)', () => {
  fc.assert(
    fc.property(
      fc.record({
        name:  fc.string(),
        age:   fc.integer({ min: 0, max: 150 }),
        email: fc.emailAddress(),
      }),
      (user) => {
        const serialized   = serialize(user);
        const deserialized = parse(serialized);
        expect(deserialized).toEqual(user);
      }
    )
  );
});
```

---

## 13. Test Organization and Naming

### File Structure

```
src/
  utils/
    formatPrice.ts
    formatPrice.test.ts    ← co-located with source (preferred)
  components/
    Button/
      Button.tsx
      Button.test.tsx

OR:

src/
  utils/
    formatPrice.ts
__tests__/
  utils/
    formatPrice.test.ts    ← separate test directory
```

### Naming Conventions

```javascript
// File naming
formatPrice.test.ts      // standard
formatPrice.spec.ts      // also common (spec = specification)

// Test names: describe what the function DOES, not what it IS
// ✅ Good test names (document behavior)
test('returns empty array when input is empty');
test('throws RangeError when index is negative');
test('returns null when user does not exist');
test('caches results after first call');

// ❌ Bad test names (describe mechanics, not behavior)
test('test 1');
test('it works');
test('handles edge case');
test('returns correct value');

// Describe blocks: group by subject
describe('formatPrice', () => {
  describe('when currency is USD', () => {
    test('formats with dollar sign');
    test('adds thousands separator for large amounts');
  });
  describe('when currency is unknown', () => {
    test('throws RangeError');
  });
});
```

---

## 14. Good Practices

### ✅ One assertion per test concept (not one per test)

```javascript
// ✅ Multiple related assertions in one test are fine
test('createUser returns complete user object', () => {
  const user = createUser('Alice', 'alice@example.com');

  expect(user.name).toBe('Alice');
  expect(user.email).toBe('alice@example.com');
  expect(user.id).toMatch(/^u_/);
  expect(user.createdAt).toBeInstanceOf(Date);
  // These all verify the same concept: the shape of a created user
});
```

### ✅ Test the contract, not the implementation

```javascript
// ✅ Tests don't know HOW sort works, only WHAT it must produce
test('sort returns elements in ascending order', () => {
  expect(sort([3, 1, 4, 1, 5])).toEqual([1, 1, 3, 4, 5]);
});

// ❌ Tests know internal implementation detail
test('sort uses quicksort algorithm', () => {
  const spy = jest.spyOn(quicksortModule, 'partition');
  sort([3, 1, 2]);
  expect(spy).toHaveBeenCalled(); // fragile: breaks if implementation changes
});
```

### ✅ Make dependencies injectable for easy mocking

```typescript
// ✅ Inject dependencies — no need to mock modules
class UserService {
  constructor(
    private readonly repository: UserRepository,
    private readonly emailService: EmailService,
  ) {}

  async createUser(data: CreateUserDto) {
    const user = await this.repository.save(data);
    await this.emailService.sendWelcomeEmail(user.email);
    return user;
  }
}

// Test: pass mock implementations
test('createUser saves to repository and sends email', async () => {
  const mockRepo  = { save: jest.fn().mockResolvedValue({ id: '1', ...data }) };
  const mockEmail = { sendWelcomeEmail: jest.fn().mockResolvedValue(undefined) };

  const service = new UserService(mockRepo, mockEmail);
  const user    = await service.createUser(data);

  expect(mockRepo.save).toHaveBeenCalledWith(data);
  expect(mockEmail.sendWelcomeEmail).toHaveBeenCalledWith(data.email);
  expect(user.id).toBe('1');
});
```

### ✅ Reset mocks between tests

```javascript
// jest.config.js — auto-clear between tests
module.exports = {
  clearMocks: true,      // clears mock.calls and mock.results
  resetMocks: false,     // also resets return values (use cautiously)
  restoreMocks: false,   // also restores spied implementations
};

// Or per-describe:
beforeEach(() => {
  jest.clearAllMocks();
});
```

---

## 15. Bad Practices

### ❌ Testing implementation details

```javascript
// ❌ Test breaks when variable is renamed
test('sets loading to true', () => {
  const service = new DataService();
  service.fetchData();
  expect(service._isLoading).toBe(true); // private implementation
});

// ✅ Test the observable effect
test('calls onLoadingChange with true when fetching starts', () => {
  const onLoading = jest.fn();
  const service   = new DataService({ onLoadingChange: onLoading });
  service.fetchData();
  expect(onLoading).toHaveBeenCalledWith(true);
});
```

### ❌ Asserting without `expect`

```javascript
// ❌ This test always passes — assertion never runs if condition is false
test('fetchUser returns user', async () => {
  const user = await fetchUser('1');
  if (user) {
    expect(user.name).toBe('Alice'); // only runs if user is truthy
  }
  // If fetchUser returns undefined: test passes silently!
});

// ✅ Always assert unconditionally
test('fetchUser returns user', async () => {
  const user = await fetchUser('1');
  expect(user).toBeDefined();
  expect(user.name).toBe('Alice');
});
```

### ❌ Not cleaning up side effects

```javascript
// ❌ Mock leaks into other tests
test('first test', () => {
  jest.spyOn(Date, 'now').mockReturnValue(1000);
  // ...
  // No restore! Date.now is still mocked for next test
});

test('second test - broken', () => {
  // Date.now still returns 1000 from previous test
  expect(Date.now()).toBeGreaterThan(0); // passes but for wrong reason
});

// ✅ Clean up in afterEach or use jest.restoreAllMocks
afterEach(() => {
  jest.restoreAllMocks();
});
```

### ❌ Overly broad mock return values

```javascript
// ❌ Mock returns different data than the real function would
jest.mock('./api');
(api.fetchUsers as jest.Mock).mockResolvedValue([]);
// Real API never returns [] (always at least has an admin user)
// Test passes but doesn't reflect real behavior

// ✅ Mock returns realistic data
(api.fetchUsers as jest.Mock).mockResolvedValue([
  { id: '1', name: 'Admin', role: 'admin' },
]);
```

---

## 16. Common Mistakes

### Mistake 1 — Forgetting to `await` async tests

```javascript
// ❌ No await — test passes without executing assertions
test('fetchUser returns user', async () => {
  fetchUser('1').then(user => {
    expect(user.name).toBe('Alice'); // inside .then — never awaited
  });
  // Test function returns immediately, assertions run AFTER test ends
});

// ✅ Always await
test('fetchUser returns user', async () => {
  const user = await fetchUser('1');
  expect(user.name).toBe('Alice');
});

// OR: return the promise
test('fetchUser returns user', () => {
  return expect(fetchUser('1')).resolves.toMatchObject({ name: 'Alice' });
});
```

### Mistake 2 — `toBe` vs `toEqual` for objects

```javascript
// ❌ toBe uses Object.is — fails for equivalent objects
expect({ a: 1 }).toBe({ a: 1 }); // FAILS: different references

// ✅ toEqual for deep equality
expect({ a: 1 }).toEqual({ a: 1 }); // PASSES: same structure
```

### Mistake 3 — Not covering the else branch

```javascript
function formatStatus(isActive) {
  if (isActive) return 'Active';
  return 'Inactive';
}

// ❌ Only tests one branch
test('shows Active when isActive is true', () => {
  expect(formatStatus(true)).toBe('Active');
});

// ✅ Test both branches
test('shows Active when isActive is true', () => {
  expect(formatStatus(true)).toBe('Active');
});

test('shows Inactive when isActive is false', () => {
  expect(formatStatus(false)).toBe('Inactive');
});
```

### Mistake 4 — Snapshots used for every test

```javascript
// ❌ Snapshots everywhere — brittle, noisy diffs
test('formatUser works', () => {
  expect(formatUser({ name: 'Alice', age: 30 })).toMatchSnapshot();
});
// Breaks every time output format changes — even intentionally

// ✅ Explicit assertions for important properties
test('formatUser formats name and age', () => {
  const result = formatUser({ name: 'Alice', age: 30 });
  expect(result.displayName).toBe('Alice (30)');
  expect(result.initials).toBe('A');
});
```

---

## 17. Interview-Level Explanation

> **"What is unit testing? What should you unit test and what should you avoid? How do you mock dependencies?"**

**Strong answer:**

> "Unit tests verify the behavior of individual functions or modules in isolation — same input produces same expected output, with dependencies controlled via mocks. They're the fastest and cheapest tests to write and run, giving you immediate feedback about exactly which function broke.
>
> The key principle is to test behavior, not implementation. A well-written unit test verifies what a function does — its contract — without caring how it does it. Tests that check internal state, call private methods, or assert that specific sub-functions were called are fragile: they break on any refactoring even if the observable behavior is unchanged.
>
> For what to test: focus on pure functions and functions with clear inputs and outputs. Test happy paths, edge cases, and error cases. For edge cases, think about: empty input, zero values, very large values, type mismatches, and null/undefined. For error cases, verify both that the function throws and what it throws.
>
> For mocking, there are two clean approaches. The first is dependency injection — design functions and classes to accept their dependencies as parameters, then pass mock implementations in tests. No module mocking needed, fully deterministic. The second is module mocking with `jest.mock()` for when you can't inject, typically for third-party libraries or modules that are difficult to instantiate. Mock at the boundary closest to external I/O — HTTP requests, databases, file system, Date.now(). Avoid mocking your own business logic.
>
> Fake timers are essential for testing debounce, throttle, polling, and anything time-dependent. `jest.useFakeTimers()` replaces setTimeout and setInterval with synchronous fakes you can advance manually, making time-based tests deterministic.
>
> Coverage is a useful guide but not a goal. 80% line coverage with meaningful tests is far better than 100% coverage with tests that don't assert anything important. I track branch coverage specifically — it's harder to achieve and more valuable than line coverage."

---

## 18. Exercises

### Exercise 1 — Write a complete test suite

Write unit tests for this `Cart` class that has the following methods:
- `addItem(item)` — adds item or increments quantity if already exists
- `removeItem(id)` — removes item by id
- `updateQuantity(id, quantity)` — sets quantity (removes if quantity <= 0)
- `getTotal()` — returns sum of (price × quantity) for all items
- `clear()` — removes all items
- `itemCount` — getter that returns total number of items (sum of quantities)

```typescript
interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}
```

<details>
<summary>Solution</summary>

```typescript
import { Cart } from './Cart';

const WIDGET = { id: '1', name: 'Widget', price: 10.00, quantity: 1 };
const GADGET  = { id: '2', name: 'Gadget', price: 25.00, quantity: 2 };

describe('Cart', () => {
  let cart: Cart;
  beforeEach(() => { cart = new Cart(); });

  describe('addItem', () => {
    test('adds new item to empty cart', () => {
      cart.addItem(WIDGET);
      expect(cart.itemCount).toBe(1);
    });

    test('increments quantity when adding existing item', () => {
      cart.addItem(WIDGET);
      cart.addItem(WIDGET);
      expect(cart.itemCount).toBe(2);
    });

    test('keeps both items when adding different items', () => {
      cart.addItem(WIDGET);
      cart.addItem(GADGET);
      expect(cart.itemCount).toBe(3); // 1 widget + 2 gadgets
    });
  });

  describe('removeItem', () => {
    test('removes existing item', () => {
      cart.addItem(WIDGET);
      cart.removeItem('1');
      expect(cart.itemCount).toBe(0);
    });

    test('does not throw when removing non-existent item', () => {
      expect(() => cart.removeItem('unknown')).not.toThrow();
    });
  });

  describe('updateQuantity', () => {
    test('updates item quantity', () => {
      cart.addItem(WIDGET);
      cart.updateQuantity('1', 5);
      expect(cart.itemCount).toBe(5);
    });

    test('removes item when quantity set to 0', () => {
      cart.addItem(WIDGET);
      cart.updateQuantity('1', 0);
      expect(cart.itemCount).toBe(0);
    });

    test('removes item when quantity is negative', () => {
      cart.addItem(WIDGET);
      cart.updateQuantity('1', -1);
      expect(cart.itemCount).toBe(0);
    });
  });

  describe('getTotal', () => {
    test('returns 0 for empty cart', () => {
      expect(cart.getTotal()).toBe(0);
    });

    test('calculates total for single item', () => {
      cart.addItem({ ...WIDGET, quantity: 3 });
      expect(cart.getTotal()).toBe(30); // 10 × 3
    });

    test('calculates total for multiple items', () => {
      cart.addItem(WIDGET);  // 10 × 1 = 10
      cart.addItem(GADGET);  // 25 × 2 = 50
      expect(cart.getTotal()).toBe(60);
    });
  });

  describe('clear', () => {
    test('removes all items', () => {
      cart.addItem(WIDGET);
      cart.addItem(GADGET);
      cart.clear();
      expect(cart.itemCount).toBe(0);
      expect(cart.getTotal()).toBe(0);
    });
  });
});
```

</details>

---

### Exercise 2 — Mock a dependency correctly

This function has a bug in how it uses its dependency. Write a test that catches it:

```typescript
// userService.ts
import { emailService } from './emailService';

export async function resetPassword(userId: string) {
  const token = generateToken();
  await saveResetToken(userId, token);
  // BUG: should pass userId, not token, as first argument
  await emailService.sendResetEmail(token, userId);
  return { success: true };
}
```

<details>
<summary>Solution</summary>

```typescript
import { resetPassword } from './userService';
import { emailService } from './emailService';

jest.mock('./emailService');
jest.mock('./userService', () => ({
  ...jest.requireActual('./userService'),
  generateToken: jest.fn().mockReturnValue('test-token'),
  saveResetToken: jest.fn().mockResolvedValue(undefined),
}));

test('sends reset email to correct userId', async () => {
  const mockSendEmail = emailService.sendResetEmail as jest.Mock;
  mockSendEmail.mockResolvedValue(undefined);

  await resetPassword('user-123');

  // This assertion will FAIL due to the bug:
  // Expected: called with ('user-123', 'test-token')
  // Received: called with ('test-token', 'user-123')
  expect(mockSendEmail).toHaveBeenCalledWith('user-123', 'test-token');
});
```

</details>

---

## 🔗 Related Topics

- [`testing/02-integration-testing.md`](./02-integration-testing.md) — Integration testing with RTL
- [`testing/03-e2e-testing.md`](./03-e2e-testing.md) — E2E testing with Playwright
- [`javascript-core/10-async-patterns.md`](../javascript-core/10-async-patterns.md) — Async patterns that need testing
- [`patterns/05-proxy-pattern.md`](../patterns/05-proxy-pattern.md) — Dependency injection patterns

---

<div align="center">

**`testing/` complete!** 🎉

**`01-unit-testing.md`** ✓ · [`02-integration-testing.md`](./02-integration-testing.md) ✓ · [`03-e2e-testing.md`](./03-e2e-testing.md) ✓

</div>