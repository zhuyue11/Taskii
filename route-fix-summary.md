# Fix Summary: Task Instances 404 Error

## Problem
The GET `/api/tasks/instances` endpoint was returning a 404 error when the frontend tried to sync task instances during app initialization.

## Root Cause
**Route Order Conflict in Express**

In Express.js, routes are matched in the order they are defined. The routes were defined as:
1. Line 343: `router.route('/:id')` - Matches `/api/tasks/{anything}`
2. Line 764: `router.get('/instances')` - Matches `/api/tasks/instances`

When a request came for `/api/tasks/instances`, Express matched it against `/:id` first (where `id = "instances"`), causing it to look for a task with ID "instances" instead of calling the instances endpoint.

## Solution
Moved the specific routes (`/instances` and `/labels`) BEFORE the generic `/:id` route:

```javascript
// Correct order:
router.route('/')           // Line 205: Base routes
router.get('/instances')    // Line 210: Specific route (moved here)
router.get('/labels')       // Line 211: Specific route (moved here)
router.route('/:id')        // Line 347: Generic parameterized route
```

## Changes Made
1. **TaskiiBackend/src/routes/tasks.ts**:
   - Moved `router.get('/instances', getTaskInstances)` from line 764 to line 210
   - Moved `router.get('/labels', getLabels)` from line 801 to line 211
   - Added comment explaining the route order importance
   - Removed duplicate route definitions

2. **TaskiiBackend/src/controllers/taskController.ts**:
   - Added `taskId` field to instance response (line 1327)

## Testing
After these changes:
1. Restart the backend server
2. Clear app data and relaunch the frontend
3. The sync should now work:
   - Categories: ✓
   - Tasks: ✓  
   - Task Instances: ✓ (should no longer get 404)

## Key Lesson
In Express.js, always define specific routes before generic parameterized routes to avoid route matching conflicts.