# React Application Patterns

Platform-agnostic patterns for building React applications with functional components, hooks, and feature-based architecture.

> **Note**: This guide assumes familiarity with JavaScript fundamentals. See [javascript.md](./javascript.md) for async/await, array methods, fetch API, and other JavaScript patterns.

## Best Practices

### Architecture & Organization
1. **Feature-based folder structure** - Organize by feature/domain (Users, Products) not by type (components, containers)
2. **Co-locate related files** - Keep component, styles, and service files together
3. **One component per file** - Export as default, name file `index.jsx`
4. **Service layer for API calls** - Isolate all fetch logic in `service.*.js` files

### Component Design
5. **Functional components only** - No class components, use hooks for state/effects
6. **Single responsibility** - Each component does one thing well
7. **Composition over inheritance** - Build complex UIs from simple components
8. **Props for data flow** - Data flows down, events flow up

### State Management
9. **useState for component state** - Local state in components
10. **localStorage for config** - App-wide configuration, non-sensitive data
11. **sessionStorage for auth** - Temporary session data, auth tokens
12. **No global state library** - Keep it simple until you need Redux/Zustand

### Data & Side Effects
13. **useEffect for side effects** - Data fetching, subscriptions, DOM manipulation
14. **Async/await in effects** - Define async function inside useEffect, then call it
15. **Cleanup in effects** - Return cleanup function for subscriptions/timers

### Styling
16. **CSS Modules for scoped styles** - Import styles as JavaScript object
17. **Inline styles for dynamic values** - Use for one-off or computed styles

### Code Quality
18. **Explicit conditionals** - Use `{data && <Component />}` not ternaries everywhere
19. **Destructure props** - Extract what you need: `function User({ name, age })`
20. **Meaningful names** - `isLoading` not `flag`, `handleSubmit` not `submit`
21. **Try-catch for errors** - Wrap async operations, display errors to users
22. **Native fetch API** - No axios needed for simple requests

### Routing
23. **React Router for navigation** - Single `<Router>` at app root
24. **useParams for URL params** - Access dynamic route segments
25. **useNavigate for redirects** - Programmatic navigation after actions

## Core Concepts

### JSX

JSX is JavaScript syntax extension that looks like HTML. It describes UI structure and compiles to JavaScript.

```javascript
// JSX (what you write)
const element = <h1 className="title">Hello World</h1>;

// Compiles to
const element = React.createElement('h1', { className: 'title' }, 'Hello World');
```

**Rules**:
- Use `className` not `class` (class is JavaScript keyword)
- Use `htmlFor` not `for` (for is JavaScript keyword)
- Close all tags: `<img />` not `<img>`
- Wrap in parentheses for multi-line
- One root element (or use fragment `<>...</>`)

**Embedding JavaScript**:
```javascript
const name = 'Alice';
const age = 30;

// Curly braces for JavaScript expressions
<h1>Hello {name}</h1>
<p>Next year you'll be {age + 1}</p>
<div className={isActive ? 'active' : 'inactive'}>Content</div>
```

### React Hooks

Hooks are functions that let you "hook into" React features from functional components.

**Rules**:
- Only call at top level (not in loops, conditions, or nested functions)
- Only call from React functions (components or custom hooks)
- Names start with `use` (useState, useEffect, useCustomHook)

## Complete Component Example

This example demonstrates a full CRUD detail page with all key patterns explained inline.

**File**: `src/Users/Detail/index.jsx`

```javascript
import { useState, useEffect } from 'react';
import { useParams, useNavigate, Link } from 'react-router-dom';
import { retrieveUser, updateUser, deleteUser } from '../service.users';
import UserForm from '../Form';
import styles from './styles.module.css';

export default function UserDetail() {
  // URL parameters from route /users/:id
  const { id } = useParams();
  
  // Navigation for redirects after actions
  const navigate = useNavigate();
  
  // Component state
  const [user, setUser] = useState(null); // Loaded user data
  const [loading, setLoading] = useState(true); // Loading indicator
  const [error, setError] = useState(null); // Error message
  
  // Fetch user data when component mounts or id changes
  useEffect(() => {
    async function fetchData() {
      try {
        setLoading(true);
        setError(null);
        const data = await retrieveUser(id);
        setUser(data);
      } catch (err) {
        console.error('Failed to fetch user:', err);
        setError('Failed to load user. Please try again.');
      } finally {
        setLoading(false);
      }
    }
    
    fetchData();
  }, [id]); // Re-run when id changes
  
  // Update user handler - passed to form
  async function handleUpdate(userData) {
    try {
      return await updateUser(id, userData);
    } catch (err) {
      console.error('Failed to update user:', err);
      throw err; // Re-throw so form can handle it
    }
  }
  
  // Success callback - update local state with saved data
  function handleSuccess(updatedUser) {
    setUser(updatedUser);
  }
  
  // Delete user with confirmation
  async function handleDelete() {
    const confirmed = confirm('Are you sure you want to delete this user?');
    if (!confirmed) return;
    
    try {
      await deleteUser(id);
      navigate('/users'); // Redirect to list after delete
    } catch (err) {
      console.error('Failed to delete user:', err);
      setError('Failed to delete user. Please try again.');
    }
  }
  
  // Loading state
  if (loading) {
    return <div className={styles.loading}>Loading...</div>;
  }
  
  // Error state
  if (error) {
    return (
      <div className={styles.error}>
        <p>{error}</p>
        <Link to="/users">Back to Users</Link>
      </div>
    );
  }
  
  // No user found
  if (!user) {
    return (
      <div>
        <p>User not found</p>
        <Link to="/users">Back to Users</Link>
      </div>
    );
  }
  
  // Main render
  return (
    <div className={styles.container}>
      <h1>{user.name}</h1>
      <p className={styles.email}>{user.email}</p>
      
      <UserForm
        initialUser={user}
        submitHandler={handleUpdate}
        onSuccess={handleSuccess}
        submitBtnLabel="Save Changes"
      />
      
      <div className={styles.actions}>
        <button 
          onClick={handleDelete} 
          className={styles.deleteBtn}
        >
          Delete User
        </button>
        <Link to="/users">Back to List</Link>
      </div>
    </div>
  );
}
```

**Explanation**:
- **useParams**: Extracts `id` from URL (`/users/123` → `id = '123'`)
- **useNavigate**: Enables redirect after delete
- **useState**: Three pieces of state (user, loading, error)
- **useEffect**: Fetches data on mount and when `id` changes
- **Async function in effect**: Can't make useEffect async directly, so define async function inside
- **Error handling**: Try-catch blocks with user-friendly error messages
- **Conditional rendering**: Show different UI for loading, error, and success states
- **Props to child**: Pass handlers to UserForm for controlled behavior
- **CSS Modules**: Import styles as object, use `styles.className`

**Component Structure**:
1. Imports (React hooks, Router hooks, services, styles)
2. Component function definition
3. Hooks at top (useParams, useNavigate, useState, useEffect)
4. Handler functions
5. Early returns for loading/error states
6. Main render return

## Project Structure

Organize code by feature/domain rather than by file type. Each feature is self-contained with its components, services, and styles.

```
src/
├── App/                    # Root application
│   ├── index.jsx          # Router and routes
│   ├── config.js          # Configuration loader
│   └── styles.module.css
├── Users/                 # Users feature
│   ├── index.jsx          # User list component
│   ├── Detail/            # User detail page
│   │   ├── index.jsx
│   │   └── styles.module.css
│   ├── Create/            # User creation page
│   │   └── index.jsx
│   ├── Form/              # Reusable user form
│   │   ├── index.jsx
│   │   └── styles.module.css
│   ├── service.users.js   # API calls for users
│   └── styles.module.css
├── Products/              # Products feature
│   ├── index.jsx
│   ├── Detail/
│   ├── service.products.js
│   └── styles.module.css
├── Shared/                # Shared components
│   ├── Form/
│   │   ├── TextInput/
│   │   │   ├── index.jsx
│   │   │   └── styles.module.css
│   │   └── NumberInput/
│   │       └── index.jsx
│   └── Layout/
│       └── index.jsx
└── main.jsx              # App entry point
```

**Why feature-based?**
- All related code in one place
- Easy to find and modify features
- Clear boundaries between features
- Can move/delete entire features easily

**File Naming Rules**:
- **Components**: `index.jsx` (one per folder)
- **Services**: `service.featurename.js` (e.g., `service.users.js`)
- **Styles**: `styles.module.css` (CSS Modules)
- **Folders**: PascalCase for components (`UserDetail/`), camelCase for utilities

## Service Layer

The service layer isolates all API interactions. Components call service functions, never fetch directly.

**File**: `src/Users/service.users.js`

```javascript
// Base URL from localStorage (set at app startup)
function apiUrl() {
  return localStorage.getItem('apiBaseUrl');
}

function baseUrl() {
  return `${apiUrl()}/users`;
}

// Helper to get auth header
function getAuthHeader() {
  const token = sessionStorage.getItem('auth-token');
  return token ? { 'Authorization': `Bearer ${token}` } : {};
}

// CREATE - POST request
export async function createUser(user) {
  const response = await fetch(baseUrl(), {
    method: 'POST',
    headers: {
      ...getAuthHeader(),
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(user),
  });
  return await response.json();
}

// READ - GET single user
export async function retrieveUser(id) {
  const response = await fetch(`${baseUrl()}/${id}`, {
    headers: getAuthHeader(),
  });
  return await response.json();
}

// UPDATE - PATCH request
export async function updateUser(id, user) {
  const response = await fetch(`${baseUrl()}/${id}`, {
    method: 'PATCH',
    headers: {
      ...getAuthHeader(),
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(user),
  });
  return await response.json();
}

// DELETE - DELETE request
export async function deleteUser(id) {
  const response = await fetch(`${baseUrl()}/${id}`, {
    method: 'DELETE',
    headers: getAuthHeader(),
  });
  return await response.json();
}

// LIST - GET all users
export async function getUsers() {
  const response = await fetch(baseUrl(), {
    headers: getAuthHeader(),
  });
  return await response.json();
}

// GET current user's items
export async function getMyUsers() {
  const url = `${apiUrl()}/my/users`;
  const response = await fetch(url, {
    headers: getAuthHeader(),
  });
  return await response.json();
}
```

**Explanation**:
- **Named exports**: Each function is exported individually
- **Helper functions**: `apiUrl()`, `baseUrl()`, `getAuthHeader()` not exported (internal)
- **Consistent patterns**: All functions follow same structure
- **RESTful naming**: `create`, `retrieve`, `update`, `delete` match HTTP verbs
- **Authorization**: Auth token added to headers when available
- **localStorage**: API URL configured at runtime, not hardcoded

**Benefits**:
- Components stay clean (no fetch code)
- Easy to test (mock service functions)
- API changes isolated to one file
- Consistent error handling
- Reusable across components

## Routing

**File**: `src/App/index.jsx`

```javascript
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import HomePage from '../HomePage';
import Dashboard from '../Dashboard';
import UserList from '../Users';
import UserDetail from '../Users/Detail';
import UserCreate from '../Users/Create';
import NotFound from '../NotFound';
import styles from './styles.module.css';

export default function App() {
  return (
    <div className={styles.app}>
      <Router>
        <Routes>
          {/* Static routes */}
          <Route path="/" element={<HomePage />} />
          <Route path="/dashboard" element={<Dashboard />} />
          
          {/* Feature routes */}
          <Route path="/users" element={<UserList />} />
          <Route path="/users/new" element={<UserCreate />} />
          <Route path="/users/:id" element={<UserDetail />} />
          
          {/* Catch-all for 404 */}
          <Route path="*" element={<NotFound />} />
        </Routes>
      </Router>
    </div>
  );
}
```

**Explanation**:
- **BrowserRouter**: Enables client-side routing
- **Routes**: Container for all Route components
- **Route**: Maps path to component
- **path="/"**: Root route
- **path="/users/:id"**: Dynamic parameter (accessed with useParams)
- **path="*"**: Catch-all for unmatched routes (404)
- **element prop**: Component to render for this route

**Route Order**: More specific routes before general ones (e.g., `/users/new` before `/users/:id`)

## React Hooks Reference

### useState

Adds state to functional components. Returns current value and setter function.

```javascript
import { useState } from 'react';

function Counter() {
  // [currentValue, setterFunction] = useState(initialValue)
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

**Explanation**: `useState(0)` initializes count to 0. `setCount` updates the value and triggers re-render.

**With Objects**:
```javascript
const [user, setUser] = useState({ name: '', email: '' });

// Update entire object
setUser({ name: 'Alice', email: 'alice@example.com' });

// Update one property (merge with existing)
setUser(prev => ({ ...prev, name: 'Alice' }));
```

**Explanation**: When updating objects, spread previous state (`...prev`) to keep other properties intact.

**With Arrays**:
```javascript
const [items, setItems] = useState([]);

// Add item
setItems(prev => [...prev, newItem]);

// Remove item
setItems(prev => prev.filter(item => item.id !== idToRemove));

// Update item
setItems(prev => prev.map(item => 
  item.id === targetId ? { ...item, name: newName } : item
));
```

**Explanation**: Use array methods (filter, map) with spread to create new arrays rather than mutating.

### useEffect

Performs side effects (data fetching, subscriptions, DOM manipulation) in functional components.

```javascript
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  // Run after every render
  useEffect(() => {
    console.log('Component rendered');
  });
  
  // Run once on mount
  useEffect(() => {
    console.log('Component mounted');
  }, []);
  
  // Run when dependency changes
  useEffect(() => {
    fetchUser(userId).then(data => setUser(data));
  }, [userId]); // Re-run when userId changes
  
  // With cleanup
  useEffect(() => {
    const timer = setInterval(() => console.log('tick'), 1000);
    
    // Cleanup function (runs before next effect and on unmount)
    return () => clearInterval(timer);
  }, []);
  
  return <div>{user?.name}</div>;
}
```

**Explanation**:
- **No dependency array**: Runs after every render
- **Empty array `[]`**: Runs once on mount
- **With dependencies `[userId]`**: Runs when userId changes
- **Return cleanup function**: Unsubscribe, clear timers, cancel requests

**Data fetching pattern**:
```javascript
useEffect(() => {
  // Define async function inside effect
  async function fetchData() {
    try {
      const data = await getUsers();
      setUsers(data);
    } catch (error) {
      console.error(error);
    }
  }
  
  fetchData();
}, []);
```

**Explanation**: Can't make useEffect callback async directly. Define async function inside and call it immediately.

### useParams

Accesses dynamic URL parameters from React Router.

```javascript
import { useParams } from 'react-router-dom';

function UserDetail() {
  // Route: /users/:id
  const { id } = useParams(); // id = '123' from /users/123
  
  // Route: /posts/:postId/comments/:commentId
  const { postId, commentId } = useParams();
  
  return <div>User ID: {id}</div>;
}
```

**Explanation**: Parameter names in route definition (`:id`) become object keys in useParams().

### useNavigate

Programmatically navigate to different routes.

```javascript
import { useNavigate } from 'react-router-dom';

function CreateUser() {
  const navigate = useNavigate();
  
  async function handleSubmit(userData) {
    const newUser = await createUser(userData);
    
    // Navigate to detail page
    navigate(`/users/${newUser.id}`);
  }
  
  function handleCancel() {
    // Go back one page in history
    navigate(-1);
  }
  
  function goHome() {
    // Navigate to specific path
    navigate('/');
  }
  
  return (
    <form onSubmit={handleSubmit}>
      {/* form fields */}
      <button type="submit">Create</button>
      <button type="button" onClick={handleCancel}>Cancel</button>
    </form>
  );
}
```

**Explanation**: 
- `navigate('/path')` - Go to specific route
- `navigate(-1)` - Go back one page (like browser back button)
- `navigate(1)` - Go forward one page

### useLocation

Access current route information.

```javascript
import { useLocation } from 'react-router-dom';

function Navigation() {
  const location = useLocation();
  
  return (
    <nav>
      <a 
        href="/" 
        className={location.pathname === '/' ? 'active' : ''}
      >
        Home
      </a>
      <a 
        href="/about" 
        className={location.pathname === '/about' ? 'active' : ''}
      >
        About
      </a>
    </nav>
  );
}
```

**Explanation**: `location.pathname` contains current route path. Use it to highlight active navigation links.

## Styling with CSS Modules

CSS Modules scope styles to components, preventing naming conflicts.

**File**: `src/Users/Detail/styles.module.css`

```css
.container {
  max-width: 800px;
  margin: 2rem auto;
  padding: 2rem;
}

.loading {
  text-align: center;
  color: #999;
  padding: 2rem;
}

.error {
  color: #d32f2f;
  background-color: #ffebee;
  padding: 1rem;
  border-radius: 4px;
  margin: 1rem 0;
}

.email {
  color: #666;
  font-size: 0.875rem;
  margin-bottom: 2rem;
}

.actions {
  margin-top: 2rem;
  display: flex;
  gap: 1rem;
}

.deleteBtn {
  background-color: #d32f2f;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

.deleteBtn:hover {
  background-color: #b71c1c;
}
```

**Using in Component**:

```javascript
import styles from './styles.module.css';

function UserDetail() {
  return (
    <div className={styles.container}>
      <p className={styles.email}>user@example.com</p>
      <button className={styles.deleteBtn}>Delete</button>
    </div>
  );
}
```

**Explanation**: Import styles as JavaScript object. Access class names as object properties. Vite/Webpack transforms class names to be unique (`container_a3x9b`).

**Combining Multiple Classes**:
```javascript
<div className={`${styles.button} ${styles.primary}`}>Click</div>
```

**Conditional Classes**:
```javascript
<div className={isActive ? styles.active : styles.inactive}>Content</div>
```

**Inline Styles for Dynamic Values**:
```javascript
<div 
  className={styles.box}
  style={{ backgroundColor: color, width: `${width}px` }}
>
  Content
</div>
```

**Explanation**: Use inline styles when values are computed or dynamic. CSS Modules for static styles.

## Configuration Management

Load environment variables differently in development vs production.

**File**: `src/main.jsx`

```javascript
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import { setLocalStorageConfigs } from './App/config';

async function render() {
  // Load config before rendering app
  await setLocalStorageConfigs();

  ReactDOM.createRoot(document.getElementById('root')).render(
    <React.StrictMode>
      <App />
    </React.StrictMode>,
  );
}

render();
```

**Explanation**: Call config loader before rendering React. Ensures all components have access to configuration from localStorage.

**File**: `src/App/config.js`

```javascript
function isDefined(param) {
  return typeof param !== 'undefined';
}

// Development: Use Vite environment variables
function getViteVars() {
  return {
    VITE_API_BASE_URL: import.meta.env.VITE_API_BASE_URL,
    VITE_BUILD_VERSION: import.meta.env.VITE_BUILD_VERSION,
  };
}

// Production: Fetch from server endpoint
async function getNodeVars() {
  const response = await fetch('/mw/config');
  return await response.json();
}

async function getVars() {
  // Check if running in Vite dev mode
  if (import.meta.env && import.meta.env.DEV) {
    return getViteVars();
  }
  // Production: fetch from server
  return await getNodeVars();
}

export async function setLocalStorageConfigs() {
  const {
    VITE_API_BASE_URL,
    VITE_BUILD_VERSION,
  } = await getVars();
  
  localStorage.clear();

  if (isDefined(VITE_API_BASE_URL)) {
    localStorage.setItem('apiBaseUrl', VITE_API_BASE_URL);
  }

  if (isDefined(VITE_BUILD_VERSION)) {
    localStorage.setItem('buildVersion', VITE_BUILD_VERSION);
  }
}
```

**Explanation**:
- **Development**: Vite reads `.env` file and exposes vars via `import.meta.env`
- **Production**: Server provides endpoint (`/mw/config`) with environment variables
- **localStorage**: All config stored in localStorage for easy access throughout app
- **Why this pattern**: Same Docker image works in multiple environments (dev, staging, prod) with different configs

**Usage in Components**:
```javascript
const apiUrl = localStorage.getItem('apiBaseUrl');
const buildVersion = localStorage.getItem('buildVersion');
```

## Conditional Rendering

Show/hide UI based on state.

### Loading States

```javascript
// Show loading, then content
{loading && <p>Loading...</p>}
{!loading && data && <DataDisplay data={data} />}
```

**Explanation**: `&&` operator short-circuits. If left side is falsy, right side doesn't render.

### Null Checks

```javascript
// Only render if data exists
{user && <UserProfile user={user} />}
{!user && <p>No user found</p>}
```

### Ternary Operator

```javascript
// Choose between two options
{loading ? <Spinner /> : <Content />}

{isAdmin ? <AdminPanel /> : <UserPanel />}
```

**Explanation**: Ternary for binary choices (show this OR that).

### Multiple Conditions

```javascript
{loading ? (
  <LoadingSpinner />
) : error ? (
  <ErrorMessage error={error} />
) : data ? (
  <DataDisplay data={data} />
) : (
  <NoData />
)}
```

**Explanation**: Chain ternaries for multiple states. Check loading first, then error, then data, finally fallback.

### Boolean Checks

```javascript
{isAuthenticated && <Dashboard />}
{!isAuthenticated && <LoginPrompt />}
{items.length === 0 && <EmptyState />}
{items.length > 0 && <ItemList items={items} />}
```

**Explanation**: Use boolean expressions with `&&` for simple show/hide logic.

## Authentication Pattern

**Login**:
```javascript
async function handleLogin(email, password) {
  try {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });
    
    const { token, user } = await response.json();
    
    // Store token in sessionStorage
    sessionStorage.setItem('auth-token', token);
    
    // Navigate to dashboard
    navigate('/dashboard');
  } catch (error) {
    setError('Login failed');
  }
}
```

**Explanation**: Store auth token in sessionStorage (cleared when browser closes). Navigate to protected route after successful login.

**Logout**:
```javascript
function handleLogout() {
  // Clear session data
  sessionStorage.clear();
  
  // Navigate to home
  navigate('/');
}
```

**Explanation**: Remove all session data and redirect to public page.

**Protected Routes**:
```javascript
import { Navigate } from 'react-router-dom';

function ProtectedRoute({ children }) {
  const token = sessionStorage.getItem('auth-token');
  
  if (!token) {
    // Redirect to login if not authenticated
    return <Navigate to="/login" replace />;
  }
  
  return children;
}

// Usage in App
<Route 
  path="/dashboard" 
  element={
    <ProtectedRoute>
      <Dashboard />
    </ProtectedRoute>
  } 
/>
```

**Explanation**: Wrapper component checks for token. If missing, redirects to login. If present, renders protected component. `replace` prevents back button to protected route.

## Build Configuration

**File**: `vite.config.js`

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    host: true, // Expose to network (for Docker)
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
  },
})
```

**Explanation**:
- **plugins**: Enables React with JSX and Fast Refresh
- **host: true**: Makes dev server accessible from network (needed for Docker containers)
- **outDir**: Where production build files go
- **sourcemap**: Generate source maps for debugging

**File**: `package.json`

```json
{
  "name": "my-react-app",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint src --ext js,jsx"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.11.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.0.0",
    "vite": "^4.3.0",
    "eslint": "^8.38.0"
  }
}
```

**Explanation**:
- **type: "module"**: Enables ES6 modules in Node.js
- **dependencies**: Runtime packages included in bundle
- **devDependencies**: Build tools not included in bundle

## When to Use This Pattern

**Ideal for**:
- Small to medium SPAs (5-50 components)
- Apps with clear feature boundaries
- Teams preferring minimal dependencies
- Projects with straightforward state needs
- Rapid prototyping and MVPs

**Consider alternatives for**:
- Large apps with complex state (use Redux, Zustand)
- Heavy caching requirements (use TanStack Query)
- Real-time data synchronization (use WebSockets, GraphQL subscriptions)
- Complex data normalization needs (use Redux Toolkit, Normalize)
