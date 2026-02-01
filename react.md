# React Application Patterns

Platform-agnostic patterns for building React applications with functional components, hooks, and feature-based architecture.

## Project Structure

### Feature-Based Organization

Organize by feature/domain rather than by file type:

```
src/
├── App/                    # Root application and routing
│   ├── index.jsx          # Main App component
│   ├── config.js          # Configuration management
│   └── styles.module.css  # App-level styles
├── Feature1/              # Feature domain (e.g., Users, Products)
│   ├── index.jsx          # Feature entry/list component
│   ├── Detail/            # Sub-feature: detail view
│   │   └── index.jsx
│   ├── Create/            # Sub-feature: create view
│   │   └── index.jsx
│   ├── Form/              # Feature-specific form
│   │   ├── index.jsx
│   │   └── styles.module.css
│   ├── service.feature1.js  # API service layer
│   └── styles.module.css    # Feature styles
├── Feature2/              # Another feature domain
│   └── ...
├── SharedComponents/      # Reusable components
│   ├── Form/
│   │   ├── TextInput/
│   │   │   └── index.jsx
│   │   ├── NumberInput/
│   │   │   └── index.jsx
│   │   └── index.jsx
│   └── Layout/
│       └── index.jsx
└── main.jsx              # Application entry point
```

### File Organization Rules

Each feature folder contains:
- **Component files** (`index.jsx`) - Main component export
- **Service files** (`service.*.js`) - API/business logic
- **Style modules** (`styles.module.css`) - Scoped styles
- **Sub-components** - Nested folders for complex features

## Component Patterns

### 1. Feature Entry Component (List/Index)

**File**: `src/Feature/index.jsx`

**Purpose**: Main entry point for a feature, typically displays a list

**Exports**: Default export of functional component

```javascript
import { useState, useEffect } from 'react';
import { Link } from 'react-router-dom';
import { getItems } from './service.feature';
import styles from './styles.module.css';

export default function FeatureList() {
  const [items, setItems] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function fetchData() {
      try {
        const data = await getItems();
        setItems(data);
      } catch (error) {
        console.error(error);
      } finally {
        setLoading(false);
      }
    }
    fetchData();
  }, []);

  if (loading) return <p>Loading...</p>;

  return (
    <div className={styles.container}>
      <h1>Items</h1>
      <Link to="/items/new">Create New</Link>
      <ul className={styles.list}>
        {items.map(item => (
          <li key={item.id}>
            <Link to={`/items/${item.id}`}>{item.name}</Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 2. Detail Component (View/Edit)

**File**: `src/Feature/Detail/index.jsx`

**Purpose**: Display and edit a single item

**Exports**: Default export of functional component

```javascript
import { useState, useEffect } from 'react';
import { useParams, useNavigate, Link } from 'react-router-dom';
import { retrieveItem, updateItem, deleteItem } from '../service.feature';
import ItemForm from '../Form';

export default function ItemDetail() {
  const { id } = useParams();
  const [item, setItem] = useState(null);
  const navigate = useNavigate();

  useEffect(() => {
    async function fetchData() {
      try {
        const data = await retrieveItem(id);
        setItem(data);
      } catch (error) {
        console.error(error);
      }
    }
    fetchData();
  }, [id]);

  async function handleSubmit(itemData) {
    try {
      return await updateItem(id, itemData);
    } catch (error) {
      console.error(error);
    }
  }

  function handleSuccess(updatedItem) {
    setItem(updatedItem);
  }

  async function handleDelete() {
    const confirmed = confirm('Are you sure?');
    if (confirmed) {
      try {
        await deleteItem(id);
        navigate('/items');
      } catch (error) {
        console.error(error);
      }
    }
  }

  if (!item) return <p>Loading...</p>;

  return (
    <div>
      <h1>{item.name}</h1>
      <ItemForm
        initialItem={item}
        submitHandler={handleSubmit}
        onSuccess={handleSuccess}
        submitBtnLabel="Save"
      />
      <button onClick={handleDelete}>Delete</button>
      <Link to="/items">Back to List</Link>
    </div>
  );
}
```

### 3. Create Component

**File**: `src/Feature/Create/index.jsx`

**Purpose**: Create a new item

**Exports**: Default export of functional component

```javascript
import { useNavigate } from 'react-router-dom';
import { createItem } from '../service.feature';
import ItemForm from '../Form';

export default function ItemCreate() {
  const navigate = useNavigate();

  async function handleSubmit(itemData) {
    try {
      return await createItem(itemData);
    } catch (error) {
      console.error(error);
    }
  }

  function handleSuccess(newItem) {
    navigate(`/items/${newItem.id}`);
  }

  return (
    <div>
      <h1>Create New Item</h1>
      <ItemForm
        submitHandler={handleSubmit}
        onSuccess={handleSuccess}
        submitBtnLabel="Create"
      />
    </div>
  );
}
```

### 4. Form Component (Reusable)

**File**: `src/Feature/Form/index.jsx`

**Purpose**: Reusable form for create/edit operations

**Exports**: Default export of functional component

```javascript
import { useState } from 'react';
import styles from './styles.module.css';

export default function ItemForm({ 
  initialItem = {}, 
  submitHandler, 
  onSuccess,
  submitBtnLabel = 'Submit' 
}) {
  const [formData, setFormData] = useState({
    name: initialItem.name || '',
    description: initialItem.description || '',
    quantity: initialItem.quantity || 0,
  });
  const [errors, setErrors] = useState({});
  const [submitting, setSubmitting] = useState(false);

  function handleChange(e) {
    const { name, value, type } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: type === 'number' ? parseFloat(value) : value
    }));
  }

  async function handleSubmit(e) {
    e.preventDefault();
    setErrors({});
    setSubmitting(true);

    try {
      const result = await submitHandler(formData);
      if (result.error) {
        setErrors({ submit: result.error });
      } else {
        onSuccess(result);
      }
    } catch (error) {
      setErrors({ submit: error.message });
    } finally {
      setSubmitting(false);
    }
  }

  return (
    <form onSubmit={handleSubmit} className={styles.form}>
      <div className={styles.field}>
        <label htmlFor="name">Name</label>
        <input
          type="text"
          id="name"
          name="name"
          value={formData.name}
          onChange={handleChange}
          required
        />
      </div>

      <div className={styles.field}>
        <label htmlFor="description">Description</label>
        <textarea
          id="description"
          name="description"
          value={formData.description}
          onChange={handleChange}
          rows={4}
        />
      </div>

      <div className={styles.field}>
        <label htmlFor="quantity">Quantity</label>
        <input
          type="number"
          id="quantity"
          name="quantity"
          value={formData.quantity}
          onChange={handleChange}
          min={0}
        />
      </div>

      {errors.submit && <p className={styles.error}>{errors.submit}</p>}

      <button type="submit" disabled={submitting}>
        {submitting ? 'Submitting...' : submitBtnLabel}
      </button>
    </form>
  );
}
```

### 5. Service Layer

**File**: `src/Feature/service.feature.js`

**Purpose**: Isolate API calls and business logic

**Exports**: Named exports of async functions

```javascript
function apiUrl() {
  return localStorage.getItem('apiBaseUrl');
}

function baseUrl() {
  return `${apiUrl()}/items`;
}

function getAuthHeader() {
  const token = sessionStorage.getItem('auth-token');
  return token ? { 'Authorization': `Bearer ${token}` } : {};
}

export async function createItem(item) {
  const response = await fetch(baseUrl(), {
    method: 'POST',
    headers: {
      ...getAuthHeader(),
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(item),
  });
  return await response.json();
}

export async function retrieveItem(id) {
  const response = await fetch(`${baseUrl()}/${id}`, {
    headers: getAuthHeader(),
  });
  return await response.json();
}

export async function updateItem(id, item) {
  const response = await fetch(`${baseUrl()}/${id}`, {
    method: 'PATCH',
    headers: {
      ...getAuthHeader(),
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(item),
  });
  return await response.json();
}

export async function deleteItem(id) {
  const response = await fetch(`${baseUrl()}/${id}`, {
    method: 'DELETE',
    headers: {
      ...getAuthHeader(),
      'Content-Type': 'application/json',
    }
  });
  return await response.json();
}

export async function getItems() {
  const response = await fetch(baseUrl(), {
    headers: getAuthHeader(),
  });
  return await response.json();
}

export async function getMyItems() {
  const token = sessionStorage.getItem('auth-token');
  const url = `${apiUrl()}/my/items`;
  const response = await fetch(url, {
    headers: { 'Authorization': `Bearer ${token}` },
  });
  return await response.json();
}
```

### 6. Shared Components

**File**: `src/SharedComponents/Form/TextInput/index.jsx`

**Purpose**: Reusable form input components

**Exports**: Default export of functional component

```javascript
import styles from './styles.module.css';

export default function TextInput({ 
  label, 
  name, 
  value, 
  onChange, 
  required = false,
  placeholder = '',
  error = null 
}) {
  return (
    <div className={styles.field}>
      <label htmlFor={name}>
        {label}
        {required && <span className={styles.required}>*</span>}
      </label>
      <input
        type="text"
        id={name}
        name={name}
        value={value}
        onChange={onChange}
        required={required}
        placeholder={placeholder}
        className={error ? styles.inputError : ''}
      />
      {error && <span className={styles.error}>{error}</span>}
    </div>
  );
}
```

## Routing

### App Router Setup

**File**: `src/App/index.jsx`

**Purpose**: Root component with routing configuration

**Exports**: Default export of App component

```javascript
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import HomePage from '../HomePage';
import Dashboard from '../Dashboard';
import ItemList from '../Items';
import ItemDetail from '../Items/Detail';
import ItemCreate from '../Items/Create';
import NotFound from '../NotFound';
import styles from './styles.module.css';

export default function App() {
  return (
    <div className={styles.app}>
      <Router>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/dashboard" element={<Dashboard />} />
          
          <Route path="/items" element={<ItemList />} />
          <Route path="/items/new" element={<ItemCreate />} />
          <Route path="/items/:id" element={<ItemDetail />} />
          
          <Route path="*" element={<NotFound />} />
        </Routes>
      </Router>
    </div>
  );
}
```

**Routing Patterns**:
- Single `<Router>` at app root
- Nested routes use path segments (`/items/:id`)
- Dynamic parameters accessed via `useParams()`
- Catch-all route for 404s (`path="*"`)

## Application Entry Point

**File**: `src/main.jsx`

**Purpose**: Bootstrap and render React application

**Exports**: None (executes immediately)

```javascript
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import { setLocalStorageConfigs } from './App/config';

async function render() {
  // Load configuration before rendering
  await setLocalStorageConfigs();

  ReactDOM.createRoot(document.getElementById('root')).render(
    <React.StrictMode>
      <App />
    </React.StrictMode>,
  );
}

render();
```

## Configuration Management

### Dual-Mode Configuration Pattern

**File**: `src/App/config.js`

**Purpose**: Load environment variables for dev and production

**Exports**: Named export of async function

```javascript
/**
 * Configuration management for different environments:
 * - Development: Uses Vite env vars from .env file
 * - Production: Fetches from server endpoint
 */

function isDefined(param) {
  return typeof param !== 'undefined';
}

function getViteVars() {
  return {
    VITE_API_BASE_URL: import.meta.env.VITE_API_BASE_URL,
    VITE_BUILD_VERSION: import.meta.env.VITE_BUILD_VERSION,
    VITE_FEATURE_FLAG_X: import.meta.env.VITE_FEATURE_FLAG_X,
  };
}

async function getNodeVars() {
  const response = await fetch(`/mw/config`);
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
    VITE_FEATURE_FLAG_X,
  } = await getVars();
  
  localStorage.clear();

  if (isDefined(VITE_API_BASE_URL)) {
    localStorage.setItem('apiBaseUrl', VITE_API_BASE_URL);
  }

  if (isDefined(VITE_BUILD_VERSION)) {
    localStorage.setItem('buildVersion', VITE_BUILD_VERSION);
  }

  if (isDefined(VITE_FEATURE_FLAG_X)) {
    localStorage.setItem('featureFlagX', VITE_FEATURE_FLAG_X);
  }
}
```

**Usage in components**:
```javascript
const apiUrl = localStorage.getItem('apiBaseUrl');
const buildVersion = localStorage.getItem('buildVersion');
```

## State Management Patterns

### Component-Level State

```javascript
const [data, setData] = useState(null);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);
```

### Derived State

```javascript
const [items, setItems] = useState([]);
const activeItems = items.filter(item => item.active);
const itemCount = items.length;
```

### Form State

```javascript
const [formData, setFormData] = useState({
  name: '',
  email: '',
  age: 0,
});

function handleChange(e) {
  const { name, value, type } = e.target;
  setFormData(prev => ({
    ...prev,
    [name]: type === 'number' ? parseFloat(value) : value
  }));
}
```

### Storage-Based State

**sessionStorage** for temporary/auth data:
```javascript
// Set
sessionStorage.setItem('auth-token', token);

// Get
const token = sessionStorage.getItem('auth-token');

// Remove
sessionStorage.removeItem('auth-token');

// Clear all
sessionStorage.clear();
```

**localStorage** for persistent data:
```javascript
localStorage.setItem('theme', 'dark');
const theme = localStorage.getItem('theme');
```

## Common Hooks Usage

### useState

```javascript
// Simple state
const [count, setCount] = useState(0);

// Object state
const [user, setUser] = useState({ name: '', email: '' });

// Array state
const [items, setItems] = useState([]);

// Function initial state (lazy)
const [data, setData] = useState(() => {
  const saved = localStorage.getItem('data');
  return saved ? JSON.parse(saved) : null;
});
```

### useEffect

```javascript
// Run once on mount
useEffect(() => {
  fetchData();
}, []);

// Run when dependency changes
useEffect(() => {
  fetchUser(userId);
}, [userId]);

// Cleanup function
useEffect(() => {
  const timer = setInterval(() => {
    console.log('tick');
  }, 1000);

  return () => clearInterval(timer);
}, []);

// Multiple effects for separation of concerns
useEffect(() => {
  // Effect 1: fetch user
  fetchUser(id);
}, [id]);

useEffect(() => {
  // Effect 2: update document title
  document.title = `User ${id}`;
}, [id]);
```

### useParams (React Router)

```javascript
import { useParams } from 'react-router-dom';

function UserDetail() {
  const { id } = useParams(); // From route: /users/:id
  const { userId, postId } = useParams(); // From route: /users/:userId/posts/:postId
  
  useEffect(() => {
    fetchUser(id);
  }, [id]);
}
```

### useNavigate (React Router)

```javascript
import { useNavigate } from 'react-router-dom';

function CreateUser() {
  const navigate = useNavigate();

  async function handleSubmit(userData) {
    const newUser = await createUser(userData);
    navigate(`/users/${newUser.id}`); // Navigate to detail page
  }

  function handleCancel() {
    navigate(-1); // Go back one page
  }

  function goHome() {
    navigate('/'); // Navigate to home
  }
}
```

### useLocation (React Router)

```javascript
import { useLocation } from 'react-router-dom';

function Navigation() {
  const location = useLocation();
  
  return (
    <nav>
      <a className={location.pathname === '/' ? 'active' : ''}>Home</a>
      <a className={location.pathname === '/about' ? 'active' : ''}>About</a>
    </nav>
  );
}
```

## Styling Approach

### CSS Modules

**File**: `src/Feature/styles.module.css`

```css
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.list {
  list-style: none;
  padding: 0;
}

.listItem {
  padding: 1rem;
  border-bottom: 1px solid #eee;
}

.error {
  color: red;
  font-size: 0.875rem;
  margin-top: 0.25rem;
}
```

**Usage in component**:
```javascript
import styles from './styles.module.css';

export default function FeatureList() {
  return (
    <div className={styles.container}>
      <ul className={styles.list}>
        <li className={styles.listItem}>Item</li>
      </ul>
    </div>
  );
}
```

**Benefits**:
- Scoped styles (no conflicts)
- Co-located with components
- Standard CSS syntax
- Imported as JavaScript objects

### Inline Styles

Use for dynamic or one-off styles:

```javascript
<div style={{ margin: '2rem auto', maxWidth: '600px' }}>
  <h1 style={{ color: error ? 'red' : 'black' }}>Title</h1>
</div>
```

### Combining Styles

```javascript
<div 
  className={styles.button} 
  style={{ backgroundColor: isActive ? 'blue' : 'gray' }}
>
  Click me
</div>
```

## Data Fetching Patterns

### Basic Fetch Pattern

```javascript
const [data, setData] = useState(null);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);

useEffect(() => {
  async function fetchData() {
    try {
      const response = await fetch('/api/items');
      const json = await response.json();
      setData(json);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }
  fetchData();
}, []);
```

### With Service Layer

```javascript
import { getItems } from './service.items';

const [items, setItems] = useState([]);

useEffect(() => {
  async function fetchData() {
    try {
      const data = await getItems();
      setItems(data);
    } catch (error) {
      console.error(error);
    }
  }
  fetchData();
}, []);
```

### Refetch on Dependency Change

```javascript
useEffect(() => {
  async function fetchData() {
    const data = await getUser(userId);
    setUser(data);
  }
  fetchData();
}, [userId]); // Refetch when userId changes
```

## Error Handling

### Try-Catch Pattern

```javascript
async function handleSubmit(data) {
  try {
    const result = await createItem(data);
    if (result.error) {
      setError(result.error);
    } else {
      onSuccess(result);
    }
  } catch (error) {
    console.error(error);
    setError(error.message);
  }
}
```

### Conditional Error Display

```javascript
{error && <p className={styles.error}>{error}</p>}
{response.error && <p>Error: {response.error}</p>}
```

### Field-Level Errors

```javascript
const [errors, setErrors] = useState({});

function validate(data) {
  const newErrors = {};
  if (!data.name) newErrors.name = 'Name is required';
  if (!data.email) newErrors.email = 'Email is required';
  return newErrors;
}

async function handleSubmit(e) {
  e.preventDefault();
  const newErrors = validate(formData);
  
  if (Object.keys(newErrors).length > 0) {
    setErrors(newErrors);
    return;
  }
  
  // Submit...
}

// In JSX
{errors.name && <span>{errors.name}</span>}
{errors.email && <span>{errors.email}</span>}
```

## Conditional Rendering

### Loading States

```javascript
{loading && <p>Loading...</p>}
{!loading && data && <DataDisplay data={data} />}
```

### Null/Undefined Checks

```javascript
{!item && <p>Loading...</p>}
{item && <ItemDetail item={item} />}
```

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

### Boolean Checks

```javascript
{isAdmin && <AdminPanel />}
{!isAuthenticated && <LoginPrompt />}
{items.length === 0 && <EmptyState />}
{items.length > 0 && <ItemList items={items} />}
```

## Authentication Patterns

### Token Storage

```javascript
// Login
function handleLogin(token) {
  sessionStorage.setItem('auth-token', token);
  navigate('/dashboard');
}

// Logout
function handleLogout() {
  sessionStorage.removeItem('auth-token');
  // or sessionStorage.clear();
  navigate('/');
}

// Check auth
function isAuthenticated() {
  return !!sessionStorage.getItem('auth-token');
}
```

### Protected Route Pattern

```javascript
import { Navigate } from 'react-router-dom';

function ProtectedRoute({ children }) {
  const token = sessionStorage.getItem('auth-token');
  
  if (!token) {
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

### Auth Context Pattern (Optional)

```javascript
import { createContext, useContext, useState } from 'react';

const AuthContext = createContext();

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  function login(token, userData) {
    sessionStorage.setItem('auth-token', token);
    setUser(userData);
  }

  function logout() {
    sessionStorage.clear();
    setUser(null);
  }

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  return useContext(AuthContext);
}

// Usage
const { user, login, logout } = useAuth();
```

## Build Configuration

### Vite Config

**File**: `vite.config.js`

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    host: true, // Expose to network
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
  },
})
```

### Package.json Scripts

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint src --ext js,jsx"
  }
}
```

## Best Practices Summary

1. **Feature-based organization** over type-based
2. **Functional components with Hooks** exclusively
3. **Service layer** isolates API calls
4. **CSS Modules** for scoped styling
5. **React Router** for navigation
6. **Native fetch** for HTTP requests
7. **localStorage** for persistent config
8. **sessionStorage** for auth tokens
9. **Explicit conditionals** for rendering logic
10. **Component composition** over inheritance
11. **Props for data flow** down the tree
12. **Async/await** for promises
13. **Try-catch** for error handling
14. **useEffect** for side effects
15. **Single responsibility** per component

## When to Use This Pattern

This architecture is ideal for:
- Small to medium SPAs (5-50 components)
- Apps with clear feature boundaries
- Teams preferring minimal dependencies
- Projects with straightforward state needs
- Rapid prototyping and MVPs

Consider alternatives (Redux, TanStack Query, etc.) for:
- Large apps with complex state
- Heavy caching requirements
- Real-time data synchronization
- Complex data normalization needs
