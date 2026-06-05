# Render.com Deployment Guide

This guide explains how to deploy the Bookary application (backend + frontend) to Render.com.

## Project Structure

```
bookary/
├── backend/                 # Node.js + Express API
├── frontend/                # React + Vite SPA
├── render.yaml             # Render configuration (monorepo)
└── .github/workflows/       # CI/CD pipeline
```

## Quick Start

### Prerequisites

1. GitHub repository with this code pushed
2. Render.com account (free tier available)
3. Basic understanding of environment variables

### Deployment Steps

1. **Connect Repository to Render**
   - Go to [render.com/dashboard](https://render.com/dashboard)
   - Click "New +" → "Web Service"
   - Select your GitHub repository
   - Render will auto-detect `render.yaml`

2. **Configure Backend Service**
   - Service Name: `bookary-backend` (auto-detected)
   - Environment: Node
   - Build Command: `npm install && npx prisma generate && npm run build`
   - Start Command: `cd /opt/render/project/backend && npm start`
   - Plan: Free tier (for testing)
   
   **Important**: The start command must navigate to the backend directory explicitly because:
   - This is a monorepo with separate `backend/` and `frontend/` directories
   - The `start` script runs `node dist/index.js`, which expects to find `dist/` in the current working directory
   - Without the `cd` command, the wrong path would be referenced

3. **Configure Frontend Service**
   - Service Name: `bookary-frontend` (auto-detected)
   - Environment: Static Site
   - Build Command: `npm install && npm run build`
   - Publish Directory: `dist`
   - Plan: Free tier

### Environment Variables Setup

#### Backend Service
In Render Dashboard → Backend Service → Environment:

```
NODE_ENV = production
DATABASE_URL = file:./prisma/prod.db
PORT = 10000
```

#### Frontend Service  
In Render Dashboard → Frontend Service → Environment:

```
VITE_API_BASE_URL = https://bookary-backend.onrender.com/api
```

⚠️ **Important**: Replace `bookary-backend` with your actual backend service name.

## Build Configuration

### Backend Build Process
The backend build command runs three critical steps in order:

1. **`npm install`** - Installs dependencies (both `dependencies` and `devDependencies`)
2. **`npx prisma generate`** - Generates the Prisma Client from schema
3. **`npm run build`** - Compiles TypeScript to JavaScript using `tsc`

**Build command in `render.yaml`:**
```yaml
buildCommand: npm install && npx prisma generate && npx prisma migrate deploy && npm run build && npm run db:seed
```

This executes in order:
1. `npm install` - Install dependencies (build tools now in `dependencies`)
2. `npx prisma generate` - Generate Prisma Client
3. `npx prisma migrate deploy` - Apply database migrations (create tables)
4. `npm run build` - Compile TypeScript to JavaScript
5. `npm run db:seed` - Populate database with demo data

**Dependencies setup** (Critical for `NODE_ENV=production`):
- Build tools must be in `dependencies`, not `devDependencies`
- Reason: Render sets `NODE_ENV=production` which skips devDependencies during `npm install`
- Therefore: TypeScript, @types/*, and ts-node must be in `dependencies`
- Keep only development-only tools in `devDependencies` (e.g., nodemon)

### TypeScript Compilation Configuration
The backend uses `tsc` to compile TypeScript. The `tsconfig.json` must have correct output directory settings:

- **`"outDir": "./dist"`** - Compiled files go to `dist/` directory
- **`"rootDir": "./src"`** - TypeScript root is the `src/` folder (NOT `./`)
- **Result**: `src/index.ts` compiles to `dist/index.js` (not `dist/src/index.js`)

If `rootDir` is set to `./`, the output structure preserves the full path (`dist/src/index.js`), causing the start command to fail with "Cannot find module 'dist/index.js'".

## Database Configuration

### Current Setup (SQLite)
- **File**: `prisma/prod.db`
- **Issue**: Render's file system is ephemeral - files don't persist between deployments
- **Duration**: Data will be lost after dyno restarts or updates

### Production Recommendations

#### Option 1: PostgreSQL (Recommended for Production)
1. Add PostgreSQL database in Render Dashboard
2. Update `DATABASE_URL` environment variable:
   ```
   DATABASE_URL=postgresql://user:password@host:5432/bookary
   ```
3. Render will handle backups automatically

#### Option 2: Keep SQLite (Testing Only)
- Current setup: `DATABASE_URL=file:./prisma/prod.db`
- Data persistence: **Not guaranteed** between deployments
- Use only for development/testing purposes

#### Option 3: Cloud SQLite (Turso)
1. Sign up at [turso.tech](https://turso.tech)
2. Create a database
3. Update `DATABASE_URL`:
   ```
   DATABASE_URL=libsql://name.turso.io?authToken=your_token
   ```

## Database Initialization

### Automatic Seeding on Deploy

The backend is configured to automatically seed the database on every Render deployment.

**Build command in `render.yaml`:**
```yaml
buildCommand: npm install && npx prisma generate && npx prisma migrate deploy && npm run build && npm run db:seed
```

**What gets seeded** - `backend/prisma/seed.ts`:
- 2 admin users (credentials below)
- 25 sample books with inventory
- 50 sample readers with membership dates
- 13 sample borrow tickets with various statuses (active, overdue, returned, lost)

**Admin credentials after seeding:**
- Email: `thuthu@quanlysach.com` / Password: `123456`
- Email: `admin@quanlysach.com` / Password: `admin123`

### Deployment Execution Order

During Render deployment, the build command runs sequentially:

1. **`npm install`** 
   - Installs dependencies including: typescript, @types/*, ts-node
   - Note: devDependencies skipped because `NODE_ENV=production`

2. **`npx prisma generate`** 
   - Generates Prisma Client from schema
   - Creates `node_modules/@prisma/client/`

3. **`npx prisma migrate deploy`** 
   - Applies migrations from `prisma/migrations/`
   - Creates database schema (tables, relationships)
   - Creates SQLite file at `prisma/prod.db` (if using SQLite)

4. **`npm run build`** 
   - Compiles TypeScript → JavaScript via `tsc`
   - Reads `tsconfig.json` (must have `rootDir: "./src"`)
   - Outputs to `dist/` directory

5. **`npm run db:seed`** 
   - Runs `ts-node prisma/seed.ts`
   - Populates tables with demo data
   - Uses `DATABASE_URL` from Render environment

6. **Server starts**
   - Render executes `npm start` → `node dist/index.js`
   - Backend listens on port 10000 (set by Render)
   - Database is pre-populated with demo data
   - Ready to receive requests

### Database URL for Seeding

The seed script automatically uses `DATABASE_URL` from environment:
- **Render environment**: `file:./prisma/prod.db` (SQLite)
- **Local development**: `file:./dev.db` (from `.env`)

The script checks `process.env.DATABASE_URL` first, then falls back to `dev.db` for local runs.

## Verify Deployment

1. **Backend Health Check**
   ```
   curl https://bookary-backend.onrender.com/api/health
   ```
   Expected response: `{"status":"ok"}`

2. **Frontend**
   - Visit `https://bookary-frontend.onrender.com`
   - Should load the login page
   - Check browser console for errors

3. **API Connectivity**
   - Login with test credentials from seed data:
     - Email: `admin@quanlysach.com`
     - Password: `admin123`

## Troubleshooting

### Frontend shows blank page
- Check VITE_API_BASE_URL in Render dashboard
- Open browser DevTools Console for errors
- Verify backend service is running (check health endpoint)

### "Cannot connect to API"
- Verify backend service URL is correct
- Check VITE_API_BASE_URL includes `/api` suffix
- Ensure backend service is deployed and healthy

### Database errors on first deploy
- Render Shell → run migrations manually:
  ```bash
  npx prisma migrate deploy
  npx prisma db seed
  ```
- Check `DATABASE_URL` environment variable is set

### Service keeps restarting
- Check application logs in Render Dashboard
- Ensure `start` script in package.json is correct
- Verify PORT environment variable handling

### Port binding error
- Backend expects `process.env.PORT` (default: 3001)
- Render sets this automatically (default: 10000)
- Should not need manual configuration

### "Cannot find type definition file for 'node'"
- **Error**: `error TS2688: Cannot find type definition file for 'node'`
- **Cause**: `@types/node` is in `devDependencies`, but `NODE_ENV=production` skips devDependencies
- **Solution**: Move `@types/node` and all `@types/*` packages to `dependencies` in `backend/package.json`
  - They're needed by TypeScript compiler during build, not just at runtime
  - Verified in current setup: all type packages are in `dependencies`

### "Cannot find module 'ts-node'"
- **Error**: During `npm run db:seed` on Render build
- **Cause**: `ts-node` is in `devDependencies` but needed during build
- **Solution**: Move `ts-node` to `dependencies` in `backend/package.json`
  - Seed script runs as part of build process
  - Needs ts-node to execute TypeScript

### Seeding fails - "Table does not exist"
- **Error**: During `npm run db:seed`: "The table `main.User` does not exist"
- **Cause**: Database migrations haven't been applied yet
- **Solution**: Ensure `npx prisma migrate deploy` runs BEFORE `npm run db:seed` in build command
  - Current build command order is correct: `...npx prisma migrate deploy && npm run build && npm run db:seed`
  - Migrations MUST create tables before seed can insert data

### Seed data not created on deploy
- **Cause**: 
  - Database initialization failed
  - Seed script encountered an error
  - Build command doesn't include `npm run db:seed`
- **Check logs**: 
  - View "Build Logs" in Render Dashboard
  - Look for "🌱 Seeding database..." message
  - Look for "✅ Database seeded successfully!" at the end
  - Check for error messages
- **Manual seed** (if needed):
  1. Connect via Render Shell: Dashboard → Backend Service → Shell
  2. Run: `npm run db:seed`
  3. Check output for errors
- **Verify database populated**:
  - Check logs for creation messages (users, books, readers, tickets)
  - Try login with demo credentials

### Build fails with "Cannot find module '/opt/render/project/src/backend/dist/index.js'"
- **Cause 1**: `npm run build` not in build command
  - Solution: Use `npm install && npx prisma generate && npm run build`
- **Cause 2**: Production packages in `devDependencies`
  - Solution: Move express, prisma, typescript, etc. to `dependencies` section
  - With `NODE_ENV=production`, Render skips devDependencies during `npm install`
- **Cause 3**: Prisma Client not generated
  - Solution: Add `npx prisma generate` before `npm run build` in build command
- **Cause 4**: Incorrect `tsconfig.json` `rootDir` setting
  - **Error**: Output files generated in `dist/src/` instead of `dist/`
  - **Solution**: Verify `backend/tsconfig.json` has `"rootDir": "./src"` (not `./`)
  - This ensures compiled files output directly to `dist/` with correct structure

### TypeScript compilation errors for Prisma imports
- **Cause**: Prisma Client not generated before `tsc` runs
- **Solution**: Ensure `npx prisma generate` runs before `npm run build`
- Check `render.yaml` build command order: `npm install && npx prisma generate && npm run build`

### "Cannot find type definition file for 'node'" during build
- **Error**: `error TS2688: Cannot find type definition file for 'node'`
- **Cause**: `NODE_ENV=production` causes Render to skip devDependencies during `npm install`, but `@types/node` and other type packages are needed during the TypeScript build
- **Solution**: Move type packages to `dependencies` instead of `devDependencies`:
  ```json
  "dependencies": {
    "@types/node": "^25.9.1",
    "@types/express": "^5.0.6",
    "@types/cors": "^2.8.19",
    "@types/bcryptjs": "^2.4.6",
    "@types/jsonwebtoken": "^9.0.10",
    "typescript": "^6.0.3",
    ...
  },
  "devDependencies": {
    "nodemon": "^3.1.14",
    "ts-node": "^10.9.2"
  }
  ```
- Why: Build tools are always needed during the build process, regardless of `NODE_ENV`

### Wrong directory path in error message
- **Error**: `Cannot find module '/opt/render/project/src/backend/dist/index.js'` (extra `src/` in path)
- **Cause**: Start command running from wrong working directory in monorepo
- **Solution**: Use explicit directory in start command: `cd /opt/render/project/backend && npm start`
- The `dir: ./backend` setting in render.yaml applies to the build, but the start command needs explicit navigation

## File Structure Details

### render.yaml
- Defines both backend (Web Service) and frontend (Static Site)
- Located at project root
- Render auto-detects on repository connection
- Specifies build commands, start commands, and directories

### Backend Configuration

**Key files:**
- `backend/package.json` - Dependencies and scripts
- `backend/tsconfig.json` - TypeScript compilation settings
- `backend/.env` and `.env.example` - Environment variables
- `backend/prisma/schema.prisma` - Database schema (User, Book, Reader, BorrowTicket)
- `backend/prisma/migrations/` - Database migration files
- `backend/prisma/seed.ts` - Demo data seed script
- `backend/src/` - TypeScript source code

**Dependencies breakdown** (in `package.json`):

```json
{
  "dependencies": {
    "express": "^5.2.1",              // Web framework
    "prisma": "^7.8.0",                // ORM
    "@prisma/client": "^7.8.0",        // Database client
    "@prisma/adapter-better-sqlite3": "^7.8.0",
    "typescript": "^6.0.3",            // TypeScript compiler (MUST be in dependencies)
    "ts-node": "^10.9.2",             // TypeScript runner (for seed script)
    "@types/node": "^25.9.1",          // Type definitions (MUST be in dependencies)
    "@types/express": "^5.0.6",        // Type definitions
    "@types/cors": "^2.8.19",          // Type definitions
    "@types/bcryptjs": "^2.4.6",       // Type definitions
    "@types/jsonwebtoken": "^9.0.10",  // Type definitions
    "bcryptjs": "^3.0.3",              // Password hashing
    "cors": "^2.8.6",                  // CORS middleware
    "jsonwebtoken": "^9.0.3",          // JWT authentication
    "dotenv": "^17.4.2"                // Environment variables
  },
  "devDependencies": {
    "nodemon": "^3.1.14"               // Auto-reload for local development
  },
  "scripts": {
    "dev": "ts-node src/index.ts",     // Development with auto-reload
    "build": "tsc",                    // Compile TypeScript
    "postbuild": "npm run db:seed || true",  // Optional local seeding
    "start": "node dist/index.js",     // Production start
    "db:reset": "npx prisma migrate reset",  // Drop and recreate database
    "db:migrate": "npx prisma migrate dev",  // Create new migration
    "db:seed": "ts-node prisma/seed.ts"     // Seed database
  }
}
```

**Why TypeScript tools in `dependencies`:**
- Render sets `NODE_ENV=production`, which skips devDependencies during `npm install`
- TypeScript compiler is needed during the build step
- ts-node is needed to run seed script
- @types/* packages are needed by TypeScript compiler
- These MUST be in `dependencies` or the build will fail

**tsconfig.json critical settings:**
```json
{
  "compilerOptions": {
    "rootDir": "./src",      // MUST be ./src, not ./
    "outDir": "./dist",      // Output directory
    "target": "ES2020",      // Target JavaScript version
    "module": "commonjs",    // Module system
    "strict": false
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### Frontend Configuration
- `frontend/vite.config.ts`: Vite build configuration
- `frontend/.env.example`: Environment variables template
- `frontend/src/api/client.ts`: Uses `VITE_API_BASE_URL` for API calls
- `frontend/package.json`: Scripts and dependencies
  - `build`: Compiles TypeScript and bundles with Vite
  - Output: `dist/` folder with static files

## Environment Variables Summary

| Variable | Service | Value | Example |
|----------|---------|-------|---------|
| `NODE_ENV` | Backend | `production` | `production` |
| `DATABASE_URL` | Backend | Database connection | `file:./prisma/prod.db` |
| `PORT` | Backend | Port number | `10000` (auto-set by Render) |
| `VITE_API_BASE_URL` | Frontend | API endpoint | `https://bookary-backend.onrender.com/api` |

## Local Development

To test the setup locally before deploying:

```bash
# Terminal 1: Backend
cd backend
npm install
npm run build
NODE_ENV=production npm start

# Terminal 2: Frontend
cd frontend
npm install
VITE_API_BASE_URL=http://localhost:3001/api npm run dev
```

### Testing Build Output Locally

Before deploying to Render, verify the complete build process works locally:

**Test complete build sequence:**
```bash
cd backend

# Step 1: Ensure .env exists with DATABASE_URL
cat .env  # Should have DATABASE_URL=file:./dev.db

# Step 2: Install dependencies (matches Render)
npm install

# Step 3: Generate Prisma Client
npx prisma generate

# Step 4: Apply migrations
npx prisma migrate deploy

# Step 5: Compile TypeScript
npm run build

# Step 6: Seed database
npm run db:seed
```

**Verify build artifacts:**
```bash
# Check that dist/ contains compiled files
ls -la dist/
# Should show: index.js, middleware/, routes/, etc.
# NOT: dist/src/index.js ❌

# Verify TypeScript output structure
ls -la dist/src/  # Should NOT exist
ls -la dist/index.js  # Should exist ✅

# Verify database was seeded
sqlite3 prisma/dev.db ".tables"  # Should list: User, Book, Reader, BorrowTicket
sqlite3 prisma/dev.db "SELECT COUNT(*) FROM User;"  # Should be 2
```

**Verify configuration:**
```bash
# Check tsconfig.json
grep '"rootDir"' tsconfig.json
# Output: "rootDir": "./src" ✅ (not "./")

# Check package.json
grep -A 20 '"dependencies"' package.json | grep -E 'typescript|@types/node|ts-node'
# All should be present in dependencies, not devDependencies
```

**Test production start:**
```bash
# Simulate Render production startup
NODE_ENV=production npm start
# Should output: 🚀 Backend server running on http://localhost:3001
```

This matches Render's build process exactly.

## CI/CD Integration

The `.github/workflows/ci.yml` file runs automated tests:
- Backend build verification
- Frontend linting and build
- Both must pass before deployment

Push to `main` branch to trigger automated deployment.

## Additional Resources

- [Render Web Services Documentation](https://render.com/docs/web-services)
- [Render Static Sites Documentation](https://render.com/docs/static-sites)
- [Prisma with Render](https://www.prisma.io/docs/guides/deployment)
