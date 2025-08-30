# Summary: Optional Date Parameters for Task Instances

## Changes Made

### Backend (TaskiiBackend)

1. **Updated `getTaskInstances` controller** (`src/controllers/taskController.ts`):
   - Made `from` and `to` parameters optional
   - If no dates provided: returns ALL instances for the user
   - If one date provided: returns error (both must be provided together)
   - If both dates provided: filters instances by date range

2. **Updated Swagger documentation** (`src/routes/tasks.ts`):
   - Marked `from` and `to` as `required: false`
   - Updated descriptions to clarify the optional nature

3. **Fixed route ordering issue**:
   - Moved `/instances` and `/labels` routes BEFORE `/:id` route
   - This prevents Express from matching `/instances` as `/:id` where `id="instances"`

### Frontend (TaskiiApp)

1. **Updated TypeScript interfaces** (`lib/api.ts`):
   - Made `from` and `to` optional in `TaskInstancesRequest`
   - Added default empty object parameter to `getTaskInstances()`

2. **Updated API method** (`lib/api.ts`):
   - Only adds date parameters to query string if both are provided
   - Handles empty query string correctly

3. **Updated sync logic** (`services/sqliteService.ts`):
   - Now fetches ALL instances (no date filter) during initial sync
   - This ensures all historical completed instances are synced

## How It Works Now

### During Initial Sync (after clearing data):
```javascript
// Before: Limited to ±3 months
const instancesResponse = await apiService.getTaskInstances({
  from: '2025-05-30',
  to: '2025-11-30'
});

// After: Gets ALL instances
const instancesResponse = await apiService.getTaskInstances({});
```

### API Behavior:
- `GET /api/tasks/instances` → Returns ALL instances for user
- `GET /api/tasks/instances?from=2025-01-01&to=2025-12-31` → Returns instances in date range
- `GET /api/tasks/instances?from=2025-01-01` → Returns error (both dates required)

## Benefits

1. **Complete sync**: All historical task instances are synced, not just recent ones
2. **Flexibility**: API can be used with or without date filtering
3. **Simplicity**: No need to calculate date ranges in the frontend for initial sync
4. **Performance**: Can still use date filters when needed for specific queries

## Testing

1. Restart the backend server
2. Clear app data and relaunch frontend
3. Check logs for: `📅 [SYNC] Fetching ALL task instances from backend...`
4. Verify all completed instances are synced, regardless of date