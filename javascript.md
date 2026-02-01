# JavaScript Fundamentals

Core JavaScript patterns and APIs used in modern web development. These patterns are framework-agnostic and applicable across all JavaScript environments.

## Async/Await Pattern

Modern JavaScript uses `async/await` for handling asynchronous operations (API calls, file reads, timers).

```javascript
// Function marked as async can use await inside
async function fetchUser(id) {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();
  return data;
}

// Calling async functions
async function main() {
  const user = await fetchUser(123);
  console.log(user);
}

// Or with .then()
fetchUser(123).then(user => console.log(user));
```

**Key Points**:
- `async` functions always return a Promise
- `await` pauses execution until Promise resolves
- Only use `await` inside `async` functions
- Cleaner than `.then()` chains for sequential operations

## Error Handling with Try-Catch

Wrap async operations in try-catch blocks to handle failures gracefully.

```javascript
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    
    // Check if request was successful
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    
    const data = await response.json();
    return data;
  } catch (error) {
    console.error('Failed to fetch user:', error);
    throw error; // Re-throw if caller needs to handle it
  }
}

// Usage with error handling at call site
async function loadUser(id) {
  try {
    const user = await fetchUser(id);
    console.log('User loaded:', user);
  } catch (error) {
    console.log('Could not load user');
  }
}
```

**Key Points**:
- `try` block contains code that might fail
- `catch` block executes if error occurs
- `throw` creates/re-throws errors
- `finally` block (optional) runs regardless of success/failure

## Fetch API

Native browser API for making HTTP requests.

### GET Request

```javascript
async function getItems() {
  const response = await fetch('/api/items');
  const data = await response.json();
  return data;
}

// With query parameters
async function searchItems(query) {
  const params = new URLSearchParams({ q: query, limit: 10 });
  const response = await fetch(`/api/items?${params}`);
  return await response.json();
}
```

### POST Request

```javascript
async function createItem(item) {
  const response = await fetch('/api/items', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(item),
  });
  return await response.json();
}
```

### PUT/PATCH Request (Update)

```javascript
async function updateItem(id, updates) {
  const response = await fetch(`/api/items/${id}`, {
    method: 'PATCH', // or 'PUT' for full replacement
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(updates),
  });
  return await response.json();
}
```

### DELETE Request

```javascript
async function deleteItem(id) {
  const response = await fetch(`/api/items/${id}`, {
    method: 'DELETE',
  });
  return await response.json();
}
```

### With Authentication Headers

```javascript
async function authenticatedRequest(url, options = {}) {
  const token = sessionStorage.getItem('auth-token');
  
  const response = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`,
    },
  });
  
  return await response.json();
}
```

## LocalStorage API

Persistent key-value storage in the browser (data survives page reloads and browser restarts).

```javascript
// Store data
localStorage.setItem('theme', 'dark');
localStorage.setItem('userId', '12345');

// Store objects (must stringify)
const user = { name: 'Alice', email: 'alice@example.com' };
localStorage.setItem('user', JSON.stringify(user));

// Retrieve data
const theme = localStorage.getItem('theme'); // 'dark'
const userId = localStorage.getItem('userId'); // '12345'

// Retrieve and parse objects
const userString = localStorage.getItem('user');
const userObj = JSON.parse(userString);

// Remove item
localStorage.removeItem('theme');

// Clear all
localStorage.clear();

// Check if item exists
if (localStorage.getItem('theme')) {
  console.log('Theme is set');
}
```

**Key Points**:
- Stores strings only (use JSON.stringify/parse for objects)
- Persists across browser sessions
- Limit ~5-10MB per domain
- Synchronous API (blocks execution)

## SessionStorage API

Temporary key-value storage (data cleared when tab/browser closes).

```javascript
// Store authentication token
sessionStorage.setItem('auth-token', 'abc123xyz');

// Retrieve token
const token = sessionStorage.getItem('auth-token');

// Remove token
sessionStorage.removeItem('auth-token');

// Clear all session data
sessionStorage.clear();
```

**Key Points**:
- Same API as localStorage
- Data lost when tab/window closes
- Not shared between tabs (each tab has own storage)
- Ideal for temporary session data, auth tokens

## Array Methods

### map() - Transform Array Elements

```javascript
const numbers = [1, 2, 3, 4];
const doubled = numbers.map(n => n * 2); // [2, 4, 6, 8]

const users = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' }
];
const names = users.map(user => user.name); // ['Alice', 'Bob']
```

**Use**: Transform each element, return new array of same length.

### filter() - Select Elements

```javascript
const numbers = [1, 2, 3, 4, 5];
const evens = numbers.filter(n => n % 2 === 0); // [2, 4]

const users = [
  { id: 1, active: true },
  { id: 2, active: false },
  { id: 3, active: true }
];
const activeUsers = users.filter(user => user.active);
```

**Use**: Keep elements that match condition, return new array.

### find() - Get First Match

```javascript
const users = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' }
];
const user = users.find(u => u.id === 2); // { id: 2, name: 'Bob' }
```

**Use**: Return first element that matches, or undefined.

### reduce() - Accumulate Values

```javascript
const numbers = [1, 2, 3, 4];
const sum = numbers.reduce((total, n) => total + n, 0); // 10

const items = [
  { name: 'apple', price: 1 },
  { name: 'banana', price: 2 }
];
const total = items.reduce((sum, item) => sum + item.price, 0); // 3
```

**Use**: Reduce array to single value (sum, object, etc.).

### sort() - Order Elements

```javascript
const numbers = [3, 1, 4, 2];
numbers.sort((a, b) => a - b); // [1, 2, 3, 4] ascending
numbers.sort((a, b) => b - a); // [4, 3, 2, 1] descending

const users = [
  { name: 'Charlie' },
  { name: 'Alice' },
  { name: 'Bob' }
];
users.sort((a, b) => a.name.localeCompare(b.name)); // Alphabetical
```

**Use**: Sort array in place (mutates original).

### includes() - Check Existence

```javascript
const numbers = [1, 2, 3];
numbers.includes(2); // true
numbers.includes(5); // false
```

**Use**: Check if array contains value.

## Object Manipulation

### Spread Operator (...)

```javascript
// Copy object
const original = { name: 'Alice', age: 30 };
const copy = { ...original }; // { name: 'Alice', age: 30 }

// Merge objects
const defaults = { theme: 'light', language: 'en' };
const userPrefs = { theme: 'dark' };
const settings = { ...defaults, ...userPrefs }; // { theme: 'dark', language: 'en' }

// Update property
const user = { name: 'Alice', age: 30 };
const updated = { ...user, age: 31 }; // { name: 'Alice', age: 31 }

// Add property
const withId = { ...user, id: 123 };
```

**Use**: Create new objects without mutating originals.

### Destructuring

```javascript
// Extract properties
const user = { name: 'Alice', age: 30, email: 'alice@example.com' };
const { name, age } = user; // name = 'Alice', age = 30

// With rename
const { name: userName } = user; // userName = 'Alice'

// With defaults
const { role = 'user' } = user; // role = 'user' (not in object)

// Nested destructuring
const data = { user: { name: 'Alice' } };
const { user: { name } } = data; // name = 'Alice'

// In function parameters
function greet({ name, age }) {
  console.log(`Hello ${name}, you are ${age}`);
}
greet(user);
```

**Use**: Extract values from objects cleanly.

### Object.keys/values/entries

```javascript
const user = { name: 'Alice', age: 30 };

// Get keys
Object.keys(user); // ['name', 'age']

// Get values
Object.values(user); // ['Alice', 30]

// Get key-value pairs
Object.entries(user); // [['name', 'Alice'], ['age', 30]]

// Iterate over object
Object.entries(user).forEach(([key, value]) => {
  console.log(`${key}: ${value}`);
});
```

**Use**: Iterate over objects, convert to arrays.

## String Manipulation

### Template Literals

```javascript
const name = 'Alice';
const age = 30;

// String interpolation
const message = `Hello ${name}, you are ${age} years old`;

// Multi-line strings
const html = `
  <div>
    <h1>${name}</h1>
    <p>Age: ${age}</p>
  </div>
`;

// Expression evaluation
const greeting = `You are ${age >= 18 ? 'an adult' : 'a minor'}`;
```

**Use**: Build dynamic strings with embedded expressions.

### Common String Methods

```javascript
const text = 'Hello World';

text.toLowerCase(); // 'hello world'
text.toUpperCase(); // 'HELLO WORLD'
text.trim(); // Remove whitespace from ends
text.includes('World'); // true
text.startsWith('Hello'); // true
text.endsWith('World'); // true
text.split(' '); // ['Hello', 'World']
text.replace('World', 'Universe'); // 'Hello Universe'
```

## Conditional (Ternary) Operator

Shorthand for if-else statements.

```javascript
// Basic ternary
const age = 20;
const status = age >= 18 ? 'adult' : 'minor';

// In expressions
const message = `You are ${age >= 18 ? 'an adult' : 'a minor'}`;

// Nested (avoid if possible)
const level = score > 90 ? 'A' : score > 80 ? 'B' : 'C';

// With function calls
const user = isLoggedIn ? fetchUserData() : null;
```

**Use**: Concise conditional assignments.

## Logical Operators

### && (AND) for Conditional Execution

```javascript
// Execute right side only if left is truthy
const isLoggedIn = true;
isLoggedIn && console.log('Welcome!'); // Logs 'Welcome!'

const user = null;
user && console.log(user.name); // Does nothing (safe)

// Return value
const displayName = user && user.name; // Returns user.name if user exists
```

**Use**: Short-circuit evaluation, conditional execution.

### || (OR) for Default Values

```javascript
// Return first truthy value
const name = user.name || 'Guest'; // Use 'Guest' if user.name is falsy

const port = process.env.PORT || 3000; // Default to 3000

// Chain multiple fallbacks
const value = option1 || option2 || option3 || 'default';
```

**Use**: Provide fallback/default values.

### ?? (Nullish Coalescing)

```javascript
// Only fallback if left is null/undefined (not other falsy values)
const count = 0;
const displayCount = count ?? 10; // 0 (not 10, because 0 is not null/undefined)

const name = null;
const displayName = name ?? 'Guest'; // 'Guest'

// Difference from ||
const value1 = 0 || 10; // 10 (because 0 is falsy)
const value2 = 0 ?? 10; // 0 (because 0 is not null/undefined)
```

**Use**: Default values when you want to allow falsy values like 0, false, ''.

## Array Destructuring

```javascript
// Extract array elements
const colors = ['red', 'green', 'blue'];
const [first, second] = colors; // first = 'red', second = 'green'

// Skip elements
const [, , third] = colors; // third = 'blue'

// Rest operator
const [primary, ...others] = colors; // primary = 'red', others = ['green', 'blue']

// With defaults
const [a, b, c = 'yellow'] = ['red', 'green']; // c = 'yellow'

// Swap variables
let x = 1, y = 2;
[x, y] = [y, x]; // x = 2, y = 1
```

**Use**: Extract values from arrays.

## Arrow Functions

Concise function syntax.

```javascript
// Traditional function
function add(a, b) {
  return a + b;
}

// Arrow function
const add = (a, b) => a + b;

// With single parameter (parentheses optional)
const double = n => n * 2;

// With no parameters
const getRandom = () => Math.random();

// With multiple lines (need braces and explicit return)
const greet = (name) => {
  const message = `Hello ${name}`;
  return message;
};

// In callbacks
const numbers = [1, 2, 3];
numbers.map(n => n * 2);
numbers.filter(n => n > 1);
```

**Key Points**:
- Implicit return for single expressions
- No `this` binding (inherits from parent scope)
- Cannot be used as constructors

## Module System (ES6)

### Named Exports

```javascript
// math.js
export function add(a, b) {
  return a + b;
}

export function subtract(a, b) {
  return a - b;
}

export const PI = 3.14159;

// Or export at end
function multiply(a, b) {
  return a * b;
}
export { multiply };
```

### Default Export

```javascript
// user.js
export default function User(name) {
  return { name };
}

// Or
function User(name) {
  return { name };
}
export default User;
```

### Importing

```javascript
// Named imports
import { add, subtract } from './math.js';

// Import all as namespace
import * as math from './math.js';
math.add(1, 2);

// Default import
import User from './user.js';

// Mix default and named
import React, { useState, useEffect } from 'react';

// Rename imports
import { add as sum } from './math.js';
```

## Common Patterns

### Optional Chaining (?.)

```javascript
// Safe property access
const user = null;
const name = user?.name; // undefined (no error)

// Nested access
const city = user?.address?.city;

// With function calls
const result = user?.getName?.();

// With arrays
const firstItem = items?.[0];
```

**Use**: Avoid errors when accessing properties on null/undefined.

### Guard Clauses

```javascript
function processUser(user) {
  // Early returns for invalid cases
  if (!user) {
    console.error('No user provided');
    return;
  }
  
  if (!user.id) {
    console.error('User has no ID');
    return;
  }
  
  // Main logic
  console.log(`Processing user ${user.id}`);
}
```

**Use**: Reduce nesting, handle edge cases first.

### IIFE (Immediately Invoked Function Expression)

```javascript
// Execute function immediately
(async function() {
  const data = await fetchData();
  console.log(data);
})();

// Or with arrow function
(async () => {
  await initialize();
})();
```

**Use**: Run async code at top level (in non-module scripts).

## Best Practices

1. **Use const by default**, let when reassignment needed, avoid var
2. **Prefer async/await** over promise chains
3. **Use arrow functions** for callbacks and short functions
4. **Destructure** objects and arrays for cleaner code
5. **Use template literals** for string concatenation
6. **Check for null/undefined** before accessing properties
7. **Use optional chaining** (?.) for safe property access
8. **Use nullish coalescing** (??) for defaults when 0/false are valid
9. **Use array methods** (map, filter) over for loops
10. **Keep functions small** and single-purpose
11. **Handle errors** with try-catch
12. **Use meaningful variable names**
13. **Avoid mutating data** (use spread operator for copies)
14. **Use strict equality** (===) over loose equality (==)
15. **Extract magic numbers** into named constants
