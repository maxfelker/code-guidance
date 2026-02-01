# React Application Patterns

This document outlines the React patterns and architectural approaches used in our applications, specifically derived from the terra-major-webapp project.

## Tech Stack

### Core Dependencies
- **React 18.2.0** - UI library
- **React Router DOM 6.11.2** - Client-side routing
- **Vite 4.3.2** - Build tool and dev server
- **@vitejs/plugin-react** - React support for Vite

### Specialized Libraries
- **@react-three/fiber** & **@react-three/drei** - 3D graphics with Three.js
- **react-unity-webgl** - Unity WebGL integration
- **@azure/app-configuration** - Cloud configuration management

## Project Structure

```
src/
├── App/              # Root application component and routing
├── Account/          # Authentication components
├── Character/        # Character management feature
├── Dashboard/        # User dashboard
├── HomePage/         # Landing page
├── AdminDashboard/   # Admin features
├── Form/             # Reusable form components
├── UnityWebClient/   # Unity integration
└── main.jsx          # Application entry point
```

### Feature-Based Organization

Components are organized by feature/domain (Character, Dashboard, Account) rather than by type. Each feature folder contains:
- Component files (`index.jsx`)
- Service files (`service.*.js`) for API calls
- Style modules (`styles.module.css`)
- Related sub-components

## Routing Pattern

### React Router Setup
```javascript
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';

export default function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/characters/:id" element={<CharacterDetail />} />
        <Route path="/admin/sandboxes/:sandboxId/instances/:instanceId" element={<InstanceDetail />} />
      </Routes>
    </Router>
  );
}
```

Key points:
- Single `<Router>` at app root
- Nested routes use path segments (e.g., `/admin/sandboxes/:sandboxId`)
- Dynamic parameters accessed via `useParams()` hook

## Component Patterns

### Functional Components with Hooks

All components use functional components with React Hooks (no class components).

```javascript
export default function CharacterDetail() {
  const { id } = useParams();
  const [character, setCharacter] = useState(null);
  const navigate = useNavigate();

  useEffect(() => {
    async function fetchData() {
      try {
        const data = await retrieveCharacter(id);
        setCharacter(data);
      } catch (error) {
        console.error(error);
      }
    }
    fetchData();
  }, [id]);

  return (
    <>
      {!character && <p>Loading</p>}
      {character && <CharacterForm initialCharacter={character} />}
    </>
  );
}
```

### Common Hook Usage

- **useState** - Component state management
- **useEffect** - Data fetching, side effects
- **useParams** - URL parameter access
- **useNavigate** - Programmatic navigation

## Data Fetching Pattern

### Service Layer
API calls are isolated in service files (e.g., `service.characters.js`):

```javascript
function apiUrl() {
  return localStorage.getItem('apiBaseUrl');
}

function baseUrl() {
  return `${apiUrl()}/characters`;
}

export async function createCharacter(character) {
  const token = sessionStorage.getItem('account-token');
  const response = await fetch(baseUrl(), {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(character),
  });
  return await response.json();
}

export async function retrieveCharacter(id) {
  const response = await fetch(`${baseUrl()}/${id}`);
  return await response.json();
}
```

### Key Patterns
- **Service functions** export async functions for each API operation
- **localStorage/sessionStorage** used for configuration and auth tokens
- **Native fetch API** for HTTP requests
- **RESTful conventions** (GET, POST, PATCH, DELETE)

## Styling Approach

### CSS Modules
```javascript
import styles from "./styles.module.css";

export default function Dashboard() {
  return <nav className={styles.nav}>...</nav>;
}
```

Benefits:
- Scoped styles prevent conflicts
- Co-located with components
- Standard CSS syntax

### Inline Styles for One-Offs
```javascript
<div style={{margin:'4rem auto', maxWidth:'600px'}}>
```

Used sparingly for component-specific layout adjustments.

## Configuration Management

### Dual-Mode Configuration
The app supports both development and production environments:

```javascript
// main.jsx
async function render() {
  await setLocalStorageConfigs();
  ReactDOM.createRoot(document.getElementById('root')).render(
    <React.StrictMode>
      <App />
    </React.StrictMode>,
  )
}
```

### Configuration Pattern
- **Development**: Vite loads `.env` file variables via `import.meta.env`
- **Production**: Node.js middleware serves configuration via `/mw/config` endpoint
- **Storage**: All config stored in `localStorage` for runtime access
- **Initialization**: Config loaded before React renders

## Form Patterns

### Reusable Form Components
Form inputs are abstracted into reusable components:
- `TextInput/`
- `NumberInput/`
- `RangeInput.jsx`
- `Vector3GroupInput/` (specialized 3D input)

### Form Submission Pattern
```javascript
async function handleSubmit(characterData) {
  try {
    return await updateCharacter(id, characterData);
  } catch (error) {
    console.error(error);
  }
}

function handleSuccess(updatedCharacter) {
  setCharacter(updatedCharacter);
}

<CharacterForm
  initialCharacter={character}
  submitHandler={handleSubmit}
  onSuccess={handleSuccess}
  submitBtnLabel="Save"
/>
```

## Authentication Pattern

- **Token Storage**: JWT tokens stored in `sessionStorage`
- **Authorization Header**: Token included in API requests
- **Protected Routes**: Components check auth and redirect if needed
- **Logout**: Clear session and navigate to home

```javascript
function signOut() {
  logout();
  navigate('/');
}
```

## State Management

**No external state management library** (Redux, Zustand, etc.) is used. State is managed with:
- Component-level `useState` for local state
- Props for passing data down
- Service layer for shared API logic
- `localStorage`/`sessionStorage` for persistence

This approach works well for medium-sized applications without excessive prop drilling.

## Error Handling

Simple try-catch with console logging:
```javascript
try {
  const data = await retrieveCharacter(id);
  setCharacter(data);
} catch (error) {
  console.error(error);
}
```

For user-facing errors, inline conditionals display messages:
```javascript
{response.error && <p>Error: {response.error}</p>}
```

## Conditional Rendering

Explicit boolean checks for loading and content states:
```javascript
{!character && <p>Loading</p>}
{character && <CharacterForm />}
```

## Build and Development

### Scripts
```json
{
  "dev": "vite",              // Development server
  "build": "vite build",      // Production build
  "lint": "eslint src",       // Code linting
  "preview": "vite preview"   // Preview production build
}
```

### Vite Configuration
Minimal configuration with React plugin:
```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
})
```

## Deployment

### Docker Support
The project includes `Dockerfile` and `Dockerfile.dev` for containerization, plus a middleware server in the `server/` directory for production configuration serving.

## Best Practices Summary

1. **Feature-based folder structure** over type-based
2. **Functional components with Hooks** exclusively
3. **Service layer** for API calls
4. **CSS Modules** for styling
5. **React Router** for navigation
6. **Native fetch** for HTTP
7. **localStorage** for configuration
8. **sessionStorage** for authentication
9. **Vite** for fast development and optimized builds
10. **No global state library** for simpler architecture

## When to Use This Pattern

This architecture is ideal for:
- Medium-sized SPAs (5-50 components)
- Apps with clear feature boundaries
- Projects requiring Unity/WebGL integration
- Teams preferring minimal dependencies
- Applications with straightforward state management needs

Consider alternative patterns (Redux, React Query, etc.) for:
- Large applications with complex state interdependencies
- Apps requiring sophisticated caching strategies
- Projects with extensive real-time data requirements
