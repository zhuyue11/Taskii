# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Taskii is a full-stack task management application with:
- **TaskiiApp**: React Native/Expo mobile application with TypeScript
- **TaskiiBackend**: Node.js/Express backend API with TypeScript and PostgreSQL

## Key Commands

### Frontend (TaskiiApp)

```bash
# Development
npm start                 # Start Expo with auto IP detection
npm run start:manual      # Start Expo without IP detection  
npm run android          # Start Android development
npm run ios              # Start iOS development
npm run find-ip          # Find and set backend IP for device testing

# Type checking
npx tsc --noEmit         # Run TypeScript type checking

# Build & Deployment
make build               # Build for production
make build-android       # Build for Android
make build-ios           # Build for iOS
```

### Backend (TaskiiBackend)

```bash
# Development
npm run dev              # Start development server with nodemon
npm run build            # Build TypeScript to JavaScript

# Database
npm run db:migrate       # Run database migrations
npm run db:migrate:deploy # Deploy migrations to production
npm run db:studio        # Open Prisma Studio for database management
npm run db:seed          # Seed database with default data
npm run db:reset         # Reset database

# Docker Development
make compose-dev         # Start services with auto-reload
make compose-dev-seed    # Start services with seeding
docker-compose logs -f   # View container logs

# Testing
npm run test:auth0       # Test Auth0 integration
npm run test:oauth       # Test OAuth flows
npm run test:security    # Run security tests
```

## CRITICAL RULES - READ CAREFULLY

### ⚠️ Database Sync Rules

**EXTREMELY IMPORTANT: `sqliteService.syncInitialData()` and `sqliteService.sync()` should ONLY be called ONCE when the app launches. NEVER call these methods after user actions like creating/updating tasks.**

- ✅ CORRECT: Call sync methods only during app initialization
- ❌ WRONG: Calling sync after creating tasks, updating tasks, or any user action
- ❌ WRONG: Using sync to "refresh" data after API calls

The app follows a **push-only** pattern after initialization:
- Changes are pushed to backend immediately when online
- Local SQLite is updated simultaneously
- NO periodic pulling from backend after initial sync

### ⚠️ Local-First Architecture Rules

**EXTREMELY IMPORTANT: The app is LOCAL-FIRST/OFFLINE-FIRST. Never rely on server response data for local state.**

- ✅ CORRECT: Use local data for all UI updates and local database operations
- ✅ CORRECT: Send data to server, but continue using local data regardless of server response
- ❌ WRONG: Using server response data to update local SQLite or UI state
- ❌ WRONG: Trusting that server response contains all necessary fields

**Key Principle**: The app must work offline. Local SQLite is the source of truth for the UI. Server responses are only used for confirmation that sync succeeded, never for data content.

**Example of CORRECT pattern**:
```typescript
// ✅ CORRECT: Create task with local data, save locally, then sync to server
const taskData = { title, description, originalTaskId, ... };
const localTask = await sqliteService.createTask(taskData); // Uses local data
await apiService.createTask(taskData); // Send to server but ignore response
// UI uses localTask data, not server response
```

**Example of WRONG pattern**:
```typescript
// ❌ WRONG: Relying on server response data
const response = await apiService.createTask(taskData);
await sqliteService.saveTaskFromServer(response); // Don't do this!
// This breaks offline-first architecture
```

## Architecture

### Data Sync Architecture

The app follows a **one-way sync pattern** for task instances:

1. **Initial Sync (on app launch after clear data)**:
   - Fetches tasks and categories from backend via `syncInitialData()`
   - Fetches task instances for recurring tasks (±3 months range)
   - Stores everything in SQLite for offline access

2. **Runtime Updates**:
   - Changes are pushed to backend immediately when online
   - Local SQLite is updated simultaneously
   - No periodic pulling from backend - we only push changes

3. **Task Instance Management**:
   - When marking instance complete: POST to `/api/tasks/{taskId}/instances`
   - Instance is saved to both backend and local SQLite
   - UI updates from local SQLite data

**Important**: There's no periodic sync pulling data from backend. The app only:
- Pulls data during initial sync (when database is empty)
- Pushes changes to backend when user makes modifications (create task, complete task, delete task, etc.)
- This ensures local changes aren't overwritten and reduces unnecessary API calls

### Frontend Architecture

- **Navigation**: React Navigation with bottom tab navigator and stack navigators
- **State Management**: React Context API for authentication (`lib/auth-context.tsx`)
- **Data Storage**: 
  - SQLite via expo-sqlite for local data (implementation in `services/sqliteService.ts`)
  - Expo SecureStore for auth tokens
- **API Communication**: Axios with interceptors for auth headers
- **Styling**: NativeWind (Tailwind CSS for React Native)

### Backend Architecture

- **API Framework**: Express.js with TypeScript
- **Database**: PostgreSQL with Prisma ORM
- **Authentication**: 
  - JWT tokens for session management
  - Auth0 integration for identity management
  - OAuth providers (Google, Apple, WeChat)
- **Security Layers**:
  - Rate limiting (express-rate-limit)
  - CORS configuration
  - Helmet for security headers
  - Input sanitization middleware
- **Real-time**: WebSocket service for live updates
- **Services Pattern**: Business logic separated into service files

### Database Schema

Key models in Prisma schema:
- **User**: Core user entity with Auth0 integration
- **Task**: Main task entity with recurrence support (JSON repeat field)
- **TaskInstance**: Individual occurrences of recurring tasks
- **Category**: Task categorization with custom colors
- **Project/Group**: Collaboration entities
- **Notification**: User notification system

## Critical Files & Locations

### Frontend
- `App.tsx`: Main app entry with navigation setup
- `lib/auth-context.tsx`: Authentication state management
- `lib/api.ts`: API client configuration
- `lib/config.ts`: App configuration and feature flags
- `screens/`: Screen components for each route
- `components/`: Reusable UI components
- `services/sqliteService.ts`: SQLite database implementation for local storage

### Backend
- `src/index.ts`: Express server setup and middleware configuration
- `src/middleware/security.ts`: Security middleware stack
- `src/controllers/`: Request handlers for each resource
- `src/services/`: Business logic and external integrations
- `src/config/`: Configuration for Auth0, OAuth, database
- `prisma/schema.prisma`: Database schema definition

## Development Workflow

### Setting Up Local Development

1. **Backend Setup**:
   ```bash
   cd TaskiiBackend
   npm install
   cp .env.example .env  # Configure environment variables
   make compose-dev      # Start PostgreSQL and backend
   npm run db:migrate    # Run database migrations
   npm run db:seed       # Seed initial data
   ```

2. **Frontend Setup**:
   ```bash
   cd TaskiiApp
   npm install
   npm run find-ip       # Auto-detect backend IP for device testing
   npm start             # Start Expo development server
   ```

### API Integration Notes

- Backend runs on port 3001 by default
- Frontend auto-detects backend IP for physical device testing via `scripts/find-ip.js`
- OAuth redirect URIs need proper configuration for both development and production
- WebSocket connections for real-time updates use the same base URL

### Testing on Physical Devices

The app includes IP detection for testing on physical devices:
1. Run `npm run find-ip` to detect and set the backend IP
2. Ensure backend is accessible from your device's network
3. OAuth flows require proper redirect URI configuration

## Important Patterns

### Authentication Flow
1. User credentials → Backend `/api/auth/login`
2. Backend validates → Returns JWT token
3. Frontend stores token in SecureStore
4. Token included in Authorization header for API calls

### Task Recurrence System
- Recurring tasks use JSON `repeat` field with pattern configuration
- `TaskInstance` table tracks individual occurrences
- Backend service generates instances based on recurrence rules

### Offline Sync & Sync Queue
- Local SQLite database for offline storage
- **Sync Queue Usage**:
  - Only used when API calls fail or user is offline
  - NOT used when operations succeed on the server
  - Pass `skipSync: true` to SQLite methods to prevent adding to sync queue
  - Example: `sqliteService.deleteTask(taskId, true)` - second param skips sync queue
  - Example: `sqliteService.updateTask(taskId, data, true)` - third param skips sync queue
- Background sync service reconciles with backend when online
- Prevents duplicate operations (e.g., deleting an already-deleted task)

### Infinite Scroll Calendar System

The app implements a sophisticated **infinite scroll system** for the calendar with a two-tier architecture:

#### 1. Multi-Layer Architecture

- **SQLite Storage Layer**: Persistent storage with dynamic boundary expansion
  - Initial Range: 71 weeks (±35 from current week)
  - Expands by 15 weeks when window edges approach boundaries
  - Tracks `furthest_past_week` and `furthest_future_week` in generation metadata

- **51-Week Sliding Window** (`weeks` state in ExpandableCalendar.tsx):
  - **FIXED SIZE**: Always 51 weeks (25 before + current + 25 after)
  - **Sliding Mechanism**: Shifts by 5 weeks when scrolling near edges
  - **Window Regeneration**: Rebuilds entire window when navigating outside 51-week range
  - Data loaded from SQLite via `calculateWeekData()`

- **UI Rendering Layer**:
  - Expanded view: 5 weeks visible
  - Collapsed view: 1 week visible
  - FlatList for smooth scrolling

#### 2. Key Constants

```typescript
const WEEKS_BUFFER = 25;              // Center position (25 before + current + 25 after)
const WINDOW_SIZE = 51;               // Total weeks in sliding window (FIXED)
const SHIFT_SIZE = 5;                 // Weeks to add/remove when sliding
const SHIFT_THRESHOLD = 20;           // Trigger shift within 20 weeks of edge
const SQLITE_EXTENSION_WEEKS = 15;    // SQLite expansion size
```

#### 3. Two-Tier Check System (`checkAndExtendGeneration()`)

**Tier 1: Window Sliding (weeks state)**
- Triggers when scroll position within 20 weeks of either edge (SHIFT_THRESHOLD)
- Shifts window by 5 weeks in scroll direction
- Removes 5 weeks from opposite end
- Maintains 51-week total size

**Tier 2: SQLite Boundary Expansion**
- Triggers when window edge within 10 weeks (2 × SHIFT_SIZE) of SQLite boundary
- Extends SQLite by 15 weeks in needed direction
- Generates task instances for ALL recurring tasks in extended range
- Updates generation metadata (furthest_past_week/furthest_future_week)

#### 4. Navigation Behavior

**Within 51-week window:**
- Scroll to target week (no regeneration needed)
- Update `currentWeekIndex` immediately for both expanded/collapsed views
- Manually trigger `checkAndExtendGeneration()` for boundary checks

**Outside 51-week window:**
- Regenerate entire 51-week window centered on target date via `ensureWeeksAroundDateLoaded()`
- Load data from SQLite for new window range
- Extend SQLite boundaries if target date outside current SQLite range

#### 5. Critical Implementation Rules

- ✅ **51-week window is FIXED SIZE** - it slides, never grows
- ✅ **checkAndExtendGeneration() is ESSENTIAL** - handles both tiers (window sliding + SQLite expansion)
- ✅ **NEVER remove checkAndExtendGeneration() calls** - breaks infinite scroll
- ✅ **Update currentWeekIndex immediately** when navigating in both expanded and collapsed views (don't rely solely on scroll callbacks)
- ✅ **Use callback form of setWeeks** to avoid race conditions when updating
- ✅ **filteredTasks depends on currentWeekIndex** - ensure it's updated before deselecting dates

#### 6. State Update Pattern for New Tasks

```typescript
// ✅ CORRECT: Use callback form to work with latest state
setWeeks(prevWeeks => {
  const instances = generateInstances(prevWeeks, ...);
  return prevWeeks.map(week => /* update with instances */);
});

// ❌ WRONG: Accessing weeks directly can cause race conditions
const instances = generateInstances(weeks, ...);
setWeeks(/* update */);
```

#### 7. Task Creation and Navigation Flow

When creating a task for a date not in the current view:
1. Save task to SQLite
2. Check if date is in weeks state (51-week window)
3. If yes: update weeks state in-memory with `updateTaskInCalendar()`
4. Navigate to date if not visible: `navigateToDate()` updates `currentWeekIndex` immediately
5. Select the date to show the task
6. **On deselect**: `filteredTasks` recalculates based on `currentWeekIndex` and shows all tasks from visible weeks

#### 8. Performance Optimizations

- **Fixed Window Size**: Prevents unbounded memory growth (constant 51 weeks)
- **Lazy Loading**: Prioritizes visible weeks, fills surrounding weeks in background
- **Pre-calculated Data**: Week data processed once via `calculateWeekData()` and cached
- **Background SQLite Expansion**: Async generation without blocking UI
- **Incremental Updates**: Only updates affected weeks when data changes


## Code Quality & Anti-Patterns

### ⚠️ setTimeout Usage Guidelines

**CRITICAL: Always question if `setTimeout` is actually needed before using it.**

#### When `setTimeout` IS appropriate:
- ✅ **Delaying execution** for UI animations or user experience
- ✅ **Debouncing/throttling** user input
- ✅ **Breaking up CPU-intensive work** to avoid blocking the main thread
- ✅ **Coordinating with React lifecycle** (rare cases where you need to wait for renders)

#### When `setTimeout` is NOT needed:
- ❌ **In async functions** - just use `await` directly
- ❌ **"Making something asynchronous"** - if it's already async, don't wrap it
- ❌ **Database/network operations** - these are already non-blocking
- ❌ **State updates** - React already batches these efficiently

**Key Question**: "What am I actually trying to delay/defer, and why?" If the answer is "nothing, I just want it to run in the background" - then `setTimeout` is probably wrong.

**Example of WRONG pattern**:
```typescript
// ❌ WRONG: Wrapping async work in setTimeout
setTimeout(async () => {
  await sqliteService.generateInstances();
  await updateWeeksState();
}, 0);
```

**Example of CORRECT pattern**:
```typescript
// ✅ CORRECT: Direct async execution
await sqliteService.generateInstances();
await updateWeeksState();
```

## Security Considerations

- Never commit `.env` files
- JWT secrets must be rotated in production
- OAuth client secrets stored securely
- Database URLs contain credentials - handle carefully
- CORS configuration restricts origins in production
- Rate limiting prevents API abuse