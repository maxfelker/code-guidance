# Docker Patterns for Frontend Applications

This document outlines Docker containerization patterns for frontend web applications, including development and production configurations.

## Multi-Stage Production Build

### Pattern: Build + Serve

A two-stage Dockerfile optimizes image size by separating build dependencies from runtime:

```dockerfile
# Build stage
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json .
COPY package-lock.json .
COPY index.html .
COPY vite.config.js .
COPY public/ public/
COPY src/ src/
RUN npm ci
RUN npm run build

# Production stage
FROM alpine
RUN apk add --update nodejs
COPY --from=build /app/package.json /app/package.json
COPY --from=build /app/dist /app/dist
COPY --from=build /app/node_modules /app/node_modules
COPY server/ /app/server
EXPOSE 80
ENTRYPOINT node /app/server/index.js
```

### Key Principles

1. **Build Stage** (`node:22-alpine AS build`)
   - Uses full Node.js image with build tools
   - Copies only necessary source files
   - Runs `npm ci` for reproducible installs
   - Executes build command (`npm run build`)

2. **Production Stage** (final stage)
   - Uses minimal `alpine` base image
   - Installs only Node.js runtime (no build tools)
   - Copies only built artifacts from build stage
   - Copies custom server for serving static files
   - Minimal attack surface and smaller image size

## Development Dockerfile

### Pattern: Hot Reload Development Server

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package.json .
COPY package-lock.json .
COPY index.html .
COPY vite.config.js .
COPY public/ public/
COPY src/ src/
RUN npm ci
ENTRYPOINT npm run dev -- --host
```

### Key Features

- Single stage for simplicity
- All source files included
- Dev server runs with `--host` flag to allow external access
- Pairs with volume mounts for hot reload (see docker-compose below)

## Docker Compose Configuration

### Pattern: Environment Variable Management + Volume Mounting

```yaml
x-environment-vars: &environment-vars
  VITE_API_BASE_URL: ${VITE_API_BASE_URL}
  VITE_BUILD_BASE_URL: ${VITE_BUILD_BASE_URL}
  VITE_BUILD_VERSION: ${VITE_BUILD_VERSION}
  VITE_CHARACTER_FBX_URL: ${VITE_CHARACTER_FBX_URL}
  VITE_DOWNLOAD_URL: ${VITE_DOWNLOAD_URL}
  VITE_DISCORD_URL: ${VITE_DISCORD_URL}
  VITE_HOST: ${VITE_HOST}

services:

  dev: 
    build: 
      context: .
      dockerfile: ./Dockerfile.dev
    environment:
      <<: *environment-vars
      PORT: 5173
    ports:
      - 5173:5173
    volumes:
      - ./public:/app/public
      - ./src:/app/src
      - ./index.html:/app/index.html

  release:
    build: 
      context: .
      dockerfile: ./Dockerfile
    environment:
      <<: *environment-vars
      PORT: 80
    ports:
      - 80:80
```

### Key Patterns

1. **YAML Anchors** (`x-environment-vars: &environment-vars`)
   - Define shared environment variables once
   - Reference with `<<: *environment-vars`
   - Reduces duplication across services

2. **Development Service**
   - Volume mounts for hot reload (`./src:/app/src`)
   - Maps source directories to container
   - Changes reflect immediately without rebuild

3. **Production Service**
   - No volume mounts (uses built files in image)
   - Different port mapping (80:80)
   - Same environment variables as dev

## Custom Node.js Server for Production

### Pattern: SPA Server with Config Endpoint

Production requires a server to:
1. Serve static files from build
2. Handle SPA routing (all routes → index.html)
3. Provide runtime configuration endpoint

```
server/
├── index.js          # Server entry point
└── routes/
    ├── config.js     # Configuration endpoint
    └── spa.js        # SPA routing handler
```

### Server Entry Point

```javascript
// server/index.js
import Hapi from '@hapi/hapi';
import Inert from '@hapi/inert';
import path from 'path';
import dotenv from 'dotenv';
import fs from 'fs';
import spaRoute from './routes/spa.js';
import configRoute from './routes/config.js';

function getServerVersion() {
  const packageJsonPath = path.resolve('/app/package.json');
  const packageJson = JSON.parse(fs.readFileSync(packageJsonPath, 'utf8'));
  return packageJson.version;
}

async function init() {
  dotenv.config();
  const { PORT } = process.env;

  const server = Hapi.server({
    port: PORT || 80,
    host: '0.0.0.0',
  });

  await server.register(Inert);

  const routes = [
    configRoute,
    spaRoute,
  ];

  server.route(routes);

  const version = getServerVersion();
  const startMessage = `App v${version} running on ${server.info.uri}`;

  await server.start();
  console.log(startMessage);
}

process.on('unhandledRejection', (err) => {
  console.log(err);
  process.exit(1);
});

init();
```

### Configuration Route

```javascript
// server/routes/config.js
const { VITE_HOST } = process.env;

export default {
  method: 'GET',
  path: '/mw/config',
  handler: (request, h) => {
    const config = {
      VITE_API_BASE_URL: process.env.VITE_API_BASE_URL,
      VITE_BUILD_BASE_URL: process.env.VITE_BUILD_BASE_URL,
      VITE_BUILD_VERSION: process.env.VITE_BUILD_VERSION,
      VITE_CHARACTER_FBX_URL: process.env.VITE_CHARACTER_FBX_URL,
      VITE_DOWNLOAD_URL: process.env.VITE_DOWNLOAD_URL,
      VITE_DISCORD_URL: process.env.VITE_DISCORD_URL
    };

    return h.response(config).code(200);
  },
  options: {
    cors: {
      origin: [ VITE_HOST ]
    }
  }
};
```

**Purpose**: Expose environment variables to the client at runtime (not build time).

### SPA Route Handler

```javascript
// server/routes/spa.js
import path from 'path';
import Boom from '@hapi/boom';

export default {
  method: 'GET',
  path: '/{param*}',
  handler: {
    directory: {
      path: path.resolve('/app/dist'),
      index: ['index.html'],
      redirectToSlash: true
    },
  },
  options: {
    ext: {
      onPreResponse: {
        method(request, h) {
          const isMiddlewareRequest = request.path.toLowerCase().trim().startsWith('/mw');
          const { response } = request;

          // If the response is a 404
          if (response.isBoom && response.output.statusCode === 404) {
            // If the request is for a middleware route, return a 404
            if (isMiddlewareRequest) {
              return Boom.notFound("Not Found");
            }

            // Serve the index.html file to support deep linking
            const filePath = path.resolve('/app/dist/index.html');
            return h.file(filePath);
          }

          // Continue with the response if inert found it
          return h.continue;
        },
      }
    }
  }
};
```

**Purpose**: 
- Serve static files from `/app/dist`
- Return `index.html` for 404s (enables client-side routing)
- Preserve 404s for API/middleware routes (`/mw/*`)

## File Structure

```
project-root/
├── Dockerfile              # Production multi-stage build
├── Dockerfile.dev          # Development with hot reload
├── docker-compose.yml      # Service orchestration
├── server/                 # Custom Node.js server
│   ├── index.js           # Server entry
│   └── routes/
│       ├── config.js      # Runtime config endpoint
│       └── spa.js         # SPA fallback handler
├── src/                   # Application source
├── public/                # Static assets
├── package.json
└── vite.config.js         # Build configuration
```

## Usage Commands

### Development

```bash
# Build and run development container
docker-compose up dev

# Or with environment file
docker-compose --env-file .env.dev up dev

# Stop
docker-compose down
```

### Production

```bash
# Build production image
docker build -f Dockerfile -t myapp:latest .

# Run production container
docker run -p 80:80 \
  -e VITE_API_BASE_URL=https://api.example.com \
  -e VITE_BUILD_VERSION=1.0.0 \
  myapp:latest

# Or use docker-compose
docker-compose up release
```

## Environment Variable Strategy

### Development
- Variables passed from host `.env` file
- Injected at container runtime
- Accessible via `import.meta.env` in Vite

### Production
- Variables passed as Docker environment variables
- Server exposes them via `/mw/config` endpoint
- Client fetches config before rendering
- Enables environment-agnostic builds (same image, different configs)

### Client-Side Configuration Loading

```javascript
// In client app before rendering
async function getConfig() {
  const response = await fetch('/mw/config');
  const config = await response.json();
  
  // Store in localStorage or context
  Object.keys(config).forEach(key => {
    localStorage.setItem(key, config[key]);
  });
}

await getConfig();
// Now render app
```

## Benefits of This Pattern

1. **Optimized Production Images**
   - Multi-stage builds = smaller images
   - No build dependencies in final image
   - Faster deployments and reduced attack surface

2. **Development Parity**
   - Same environment variables across dev/prod
   - Containerized dev environment matches production
   - Volume mounts enable hot reload

3. **Runtime Configuration**
   - Same Docker image for multiple environments
   - No rebuild needed to change API URLs
   - Configuration lives outside the build

4. **SPA Support**
   - Client-side routing works with deep links
   - Static assets served efficiently
   - Clean separation of middleware/static routes

## Alternative Patterns

### Nginx for Production
Instead of Node.js server, use Nginx:

```dockerfile
# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**When to use**: 
- No need for runtime configuration endpoint
- Pure static site hosting
- Maximum performance for static files

### Express Server
Alternative to Hapi:

```javascript
import express from 'express';
import path from 'path';

const app = express();
const PORT = process.env.PORT || 80;

// Config endpoint
app.get('/mw/config', (req, res) => {
  res.json({
    VITE_API_BASE_URL: process.env.VITE_API_BASE_URL,
    // ... other vars
  });
});

// Static files
app.use(express.static(path.resolve('/app/dist')));

// SPA fallback
app.get('*', (req, res) => {
  res.sendFile(path.resolve('/app/dist/index.html'));
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Best Practices

1. **Use alpine base images** for smaller size
2. **Copy package files first** to leverage layer caching
3. **Use npm ci** instead of npm install for reproducible builds
4. **WORKDIR** for explicit working directory
5. **EXPOSE** to document port usage
6. **Volume mount source in dev** for hot reload
7. **Separate Dockerfiles** for dev/prod clarity
8. **Environment variable anchors** in compose files
9. **Health checks** for production containers
10. **.dockerignore** to exclude node_modules, .git, etc.

## Sample .dockerignore

```
node_modules
.git
.gitignore
*.md
.env
.env.*
dist
coverage
.vscode
.idea
*.log
```
