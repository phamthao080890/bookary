# Bookary - Library Management System

A full-stack Vietnamese library management system built with React 18, Node.js/Express, Prisma, and SQLite.

## Project Structure

```
bookary/
├── backend/          # Node.js + Express + TypeScript
│   ├── prisma/
│   │   ├── schema.prisma    # Database schema
│   │   ├── migrations/       # Database migrations
│   │   └── seed.ts          # Seed data script
│   ├── src/
│   │   ├── middleware/       # Authentication middleware
│   │   ├── routes/           # API routes
│   │   └── index.ts          # Express server
│   └── package.json
│
└── frontend/         # React + TypeScript + Tailwind CSS
    ├── src/
    │   ├── api/              # API clients
    │   ├── components/       # Reusable components
    │   ├── context/          # Auth context
    │   ├── pages/            # Route pages
    │   ├── types.ts          # TypeScript types
    │   ├── App.tsx           # Main app
    │   └── main.tsx          # Entry point
    └── package.json
```

## Features Implemented

### Module 1: Onboarding & Authentication ✅
- Login & registration pages
- Role-based access control (Admin/Guest)
- JWT-based authentication
- Protected routes

### Module 2: Book Management ✅
- CRUD operations (Create, Read, Update, Delete)
- Book search & pagination
- Inventory tracking (available quantity)
- Admin-only write operations

### Module 3: Reader Management ✅
- CRUD operations for readers
- Search by code/name/email/phone
- Membership expiry tracking
- Red flag for expired memberships

### Module 4: Borrow Books (3-Step Wizard) ✅
- Step 1: Select reader from list
- Step 2: Select book to borrow
- Step 3: Confirmation & print option
- Automatic inventory decrement

### Module 5: Return Books ✅
- Search active borrow tickets
- Mark books as returned or lost
- Calculate overdue fines (5000 VND/day)
- Automatic inventory increment

### Module 6: Extend Borrowing (Gia Hạn) ✅
- Extend due date by 7 days
- View all active tickets
- Search functionality

### Module 7: Dashboard & Reports ✅
- Summary statistics (total books, readers, borrowing, overdue)
- Recent borrow tickets list
- Status indicators (red for overdue)
- Real-time data

## Database Schema

### Tables
- **User**: Authentication (email, password, role)
- **Book**: Book catalog (code, title, author, quantities)
- **Reader**: Library members (code, name, contact, membership)
- **BorrowTicket**: Borrow records (reader, book, dates, status)

### Demo Data
- 2 admin users
- 25 sample books
- 50 sample readers
- 13 sample borrow tickets (including overdue, returned, and lost records)

## Getting Started

### Prerequisites
- Node.js 20+ (required for backend)
- npm or yarn

### Project Structure
This is a **monorepo** with separate backend and frontend services:
```
bookary/
├── backend/          # Node.js + Express API (port 3001)
├── frontend/         # React + Vite SPA (port 5173)
└── render.yaml       # Render deployment config
```

Each service must be built and run independently.

### Quick Start (Development)

**Terminal 1 - Backend:**
```bash
cd backend
npm install
npm run dev       # Runs with hot-reload (ts-node)
```
Backend runs on `http://localhost:3001`

**Terminal 2 - Frontend:**
```bash
cd frontend
npm install
npm run dev       # Runs Vite dev server
```
Frontend runs on `http://localhost:5173`

### Production Build & Run

**Backend (Production):**
```bash
cd backend
npm install
npx prisma generate
npm run build     # Compiles TypeScript to dist/
npm start         # Runs compiled JavaScript
```

**Frontend (Production):**
```bash
cd frontend
npm install
npm run dev

npm run build     # Bundles with Vite to dist/
npm run preview   # Preview production build
```

### Render Deployment
The backend automatically seeds demo data on every deployment. See [RENDER_DEPLOYMENT.md](./RENDER_DEPLOYMENT.md) for complete instructions.

### Backend Setup (Detailed)

**First-time setup:**
```bash
cd backend

# 1. Install dependencies
npm install

# 2. Generate Prisma Client
npx prisma generate

# 3. Apply database migrations (creates tables)
npx prisma migrate deploy

# 4. Seed database with demo data
npm run db:seed

# 5. Start development server
npm run dev
```

**Development:**
```bash
# Start with hot-reload (ts-node)
npm run dev

# View database in browser
npx prisma studio
```

**Production:**
```bash
# Compile TypeScript
npm run build

# Run compiled server
npm start
```

**Database Management:**
```bash
# Reset database completely (drop and recreate)
npm run db:reset

# Create new migration
npx prisma migrate dev --name migration_name

# Apply existing migrations (CI/CD)
npx prisma migrate deploy

# Seed with demo data
npm run db:seed

# View/edit database in GUI
npx prisma studio
```

Backend runs on `http://localhost:3001`

### Frontend Setup

**Development:**
```bash
cd frontend

# Install dependencies
npm install

# Start dev server with hot-reload
npm run dev
```
Frontend runs on `http://localhost:5173`

**Production:**
```bash
# Build for production
npm run build

# Preview production build locally
npm run preview
```

**Other commands:**
```bash
# Lint code
npm run lint

# View build output
npm run build  # Creates dist/ folder
```

## Demo Credentials

### Admin 1 (Full access)
- Email: `thuthu@quanlysach.com`
- Password: `123456`
- Role: Admin
- Can access all modules

### Admin 2 (Full access)
- Email: `admin@quanlysach.com`
- Password: `admin123`
- Role: Admin
- Can access all modules

**Note:** There are no pre-created guest users. Register a new account with a guest role to test guest functionality. Guest users can only view the book catalog.

## API Endpoints

### Authentication
- `POST /api/auth/login` - User login
- `POST /api/auth/register` - User registration

### Books (Paginated)
- `GET /api/books?search=&page=1` - List books
- `GET /api/books/:id` - Get book details
- `POST /api/books` - Create book (Admin only)
- `PUT /api/books/:id` - Update book (Admin only)
- `DELETE /api/books/:id` - Delete book (Admin only)

### Readers (Paginated)
- `GET /api/readers?search=&page=1` - List readers
- `GET /api/readers/:id` - Get reader details
- `GET /api/readers/:id/history` - Borrow history
- `POST /api/readers` - Create reader (Admin only)
- `PUT /api/readers/:id` - Update reader (Admin only)
- `DELETE /api/readers/:id` - Delete reader (Admin only)

### Borrowing
- `POST /api/borrow` - Create borrow ticket (Admin only)
- `GET /api/return?search=` - List active tickets
- `POST /api/return/:ticketId` - Return book
- `GET /api/renew?search=` - List tickets for renewal
- `POST /api/renew/:ticketId` - Extend due date

### Dashboard
- `GET /api/dashboard` - Get statistics and recent tickets

## Troubleshooting

### Monorepo Build Issues

**Error: "Could not read package.json"**
- **Cause**: Trying to build from root directory instead of subdirectory
- **Solution**: Build backend and frontend separately:
  ```bash
  cd backend && npm run build && npm run start
  cd frontend && npm run build
  ```

**Error: "Cannot find module 'dist/index.js'" or files in wrong directory**
- **Cause**: `tsconfig.json` has incorrect `rootDir` setting
- **Solution**: Verify `backend/tsconfig.json` contains `"rootDir": "./src"` (not `./`)
- TypeScript compilation puts output in `dist/` not `dist/src/`
- Then rebuild: `npm run build`

**Error: "tsc: command not found"**
- **Cause**: Dependencies not installed
- **Solution**: Run `npm install` in the backend or frontend directory

### Backend Build Issues

**Error: "Cannot find type definition file for 'node'"** (on Render)
- **Cause**: `NODE_ENV=production` causes npm to skip devDependencies
- **Solution**: Type packages must be in `dependencies`, not `devDependencies`:
  - Verified in `backend/package.json`: @types/node, @types/express, etc. are in dependencies
  - Also includes: typescript, ts-node (needed for building and seeding)

**Error: "Cannot find module 'ts-node'"** (on Render)
- **Cause**: ts-node is in devDependencies but needed during build
- **Solution**: Move ts-node to `dependencies` in `backend/package.json`

**Error: "PrismaClient not exported"**
- **Cause**: Prisma Client not generated
- **Solution**: 
  ```bash
  npx prisma generate
  npm run build
  ```

### Backend Runtime Issues

**Error: Backend won't start after build**
- **Check**: Verify `dist/index.js` exists: `ls -la dist/`
- **Check**: PORT environment variable (default 3001)
- **Solution**: `npm run start` should work after successful build

**Error: Database connection failed**
- **Check**: DATABASE_URL is set correctly
- **Solution**: 
  ```bash
  # Create .env file if missing
  cp .env.example .env
  ```

### Frontend Build Issues

**Error: "Port 5173 in use"**
```bash
npm run dev -- --port 5174
```

**Error: Build succeeds but blank page**
- **Check**: Browser console for errors
- **Check**: API_BASE_URL is correct
- **Solution**: Ensure backend is running on localhost:3001

### Database Issues

**Error: "Table does not exist"**
- **Cause**: Migrations haven't been applied
- **Solution**:
  ```bash
  npx prisma migrate deploy  # Apply existing migrations
  npm run db:seed           # Populate with demo data
  ```

**Error: Seed data not created**
- **Cause**: Database schema doesn't exist yet
- **Solution**: Always run migrations before seeding:
  ```bash
  npx prisma migrate deploy
  npm run db:seed
  ```

**Reset database completely**
```bash
npm run db:reset
# This will:
# 1. Drop the database
# 2. Re-run all migrations
# 3. Run the seed script
```

**Manual database cleanup**
```bash
# Remove database file and start fresh
rm backend/prisma/dev.db
npx prisma migrate deploy
npm run db:seed
```

**View database with GUI**
```bash
npx prisma studio
```

## Tech Stack

### Backend
- Node.js + Express 5.2
- TypeScript 6.0
- Prisma 7.8 (ORM)
- SQLite (Database)
- JWT (Authentication)
- bcryptjs (Password hashing)
- CORS

### Frontend
- React 18
- TypeScript
- Vite
- Tailwind CSS
- React Router v6
- Axios (HTTP client)
- React Query (State management)

## Development Notes

- All routes require authentication (Bearer token in Authorization header)
- Admin routes are protected with `adminMiddleware`
- Guest users can only view books
- Timestamps use ISO 8601 format
- All monetary values in VND (Vietnamese Dong)
- Fine calculation: 5000 VND per day overdue

## CI/CD Configuration

### GitHub Actions
- **Frontend**: Lints and builds React app on every push
- **Backend**: Generates Prisma Client and builds TypeScript on every push
- **Workflow file**: `.github/workflows/ci.yml`

The CI/CD pipeline runs on:
- Push to `main`, `develop`, or `master` branches
- Pull requests to those branches

**Note:** Database seeding is not part of the build process. It's executed locally or during deployment.

## Future Enhancements

- Email notifications for overdue books
- Reservation system
- Book ratings/reviews
- User profile management
- Advanced reporting/analytics
- Barcode scanning support
- Mobile app (React Native)

## License

MIT

## Support

For issues or questions, check the troubleshooting section above or review the code structure.

Test triggering auto-deploy
