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

### 35-Week Cache System

The app implements a **35-week rolling cache** for optimal performance when viewing tasks:

1. **Cache Structure** (`TaskiiApp/screens/HomeScreen.tsx`):
   - Maintains a 35-week window of tasks in memory (17 weeks before + current week + 17 weeks after)
   - Cache is centered on the current week and stored in `cache35Weeks` state
   - Includes both regular tasks and task instances for recurring tasks

2. **Loading Strategy**:
   - `load35WeekCache(centerWeekStart)`: Loads tasks and instances for 35-week window from SQLite
   - Automatically reloads when user scrolls >10 weeks from cache center
   - Combines tasks with dates and unplanned tasks in single cache

3. **Data Flow**:
   ```
   SQLite Database → 35-Week Cache → View-Specific Filtering → UI Display
   ```

4. **Cache Updates**:
   - After any task modification (create/update/delete/complete)
   - After marking recurring task instances as complete
   - When navigating to dates outside current cache window
   - During initial app load and data sync

5. **Performance Benefits**:
   - Reduces SQLite queries by maintaining tasks in memory
   - Enables smooth calendar scrolling without database hits
   - Supports offline-first architecture with local data priority

6. **Implementation Details**:
   - Calendar component (`ExpandableCalendar.tsx`) generates 35 weeks of UI
   - HomeScreen filters cache based on current view (day/week/5-week)
   - Task instances loaded separately but displayed alongside regular tasks

## Security Considerations

- Never commit `.env` files
- JWT secrets must be rotated in production
- OAuth client secrets stored securely
- Database URLs contain credentials - handle carefully
- CORS configuration restricts origins in production
- Rate limiting prevents API abuse