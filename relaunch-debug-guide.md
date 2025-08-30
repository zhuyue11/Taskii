# Debug Guide: Task Instances After Relaunch

## Test Scenario
1. Mark a recurring task instance as complete
2. Clear app data / reinstall app
3. Relaunch the app
4. Check if the instance shows as completed

## Expected Log Flow After Relaunch

### 1. Initial Sync Phase
Look for these logs during app initialization:

```
📅 [SYNC] Fetching task instances from backend...
📅 [SYNC] Requesting instances from [date] to [date]
📅 [SYNC] Response received: {
  hasData: true,
  hasInstancesByTask: true,
  taskCount: [number]
}
```

**Check Point 1**: Does the response have instances data?

```
📅 [SYNC] Task [taskId] has [X] instances
📅 [SYNC] Found COMPLETED instance: [taskId] on [date]
```

**Check Point 2**: Are completed instances found in the API response?

```
✅ [SYNC] Synced [X] task instances ([Y] completed) from backend
📅 [SYNC] Verification: [X] instances in SQLite
📅 [SYNC] Verification: [Y] completed instances in SQLite
```

**Check Point 3**: Are completed instances saved to SQLite?

### 2. Loading Phase
When the 35-week cache loads:

```
📅 [LOAD] Loading instances for 35-week cache: [X] total
📅 [LOAD] Found [Y] completed instances
📅 [LOAD] Sample completed instance: { ... }
```

**Check Point 4**: Are completed instances loaded from SQLite?

### 3. Generation Phase
When generating calendar items:

```
📅 [GENERATE] Task "[name]" ([taskId]): [X] completed instances
```

**Check Point 5**: Are instances matched to the correct task?

```
📅 [CALENDAR] Creating completed calendar item for [date], task: [taskId]
```

**Check Point 6**: Are completed calendar items created?

### 4. Display Phase

```
📅 [DISPLAY] Converting completed instance for display: [date], task: [taskId]
```

**Check Point 7**: Are completed instances converted for UI display?

## Troubleshooting Checklist

### ❌ If no instances in sync response:
- Check backend API endpoint `/api/tasks/[taskId]/instances`
- Verify instances exist in backend database
- Check date range in request

### ❌ If instances not saved to SQLite:
- Check `saveTaskInstance` method
- Verify SQLite table structure
- Check for errors in console

### ❌ If instances not loaded from SQLite:
- Check `getTaskInstances` date range
- Verify SQLite query
- Check instance date format

### ❌ If instances not matched to tasks:
- Check task ID mapping (serverId vs local ID)
- Verify task exists in cache35Weeks
- Check instance.taskId value

### ❌ If calendar items not marked completed:
- Check instance.status value (should be 'COMPLETED' uppercase)
- Verify existingInstancesMap contains the instance
- Check date string format matching

### ❌ If UI doesn't show completed:
- Check item.isCompleted value
- Verify status conversion in convertCalendarTasksToDisplayTasks
- Check final task.status value

## Key Data Flow
```
Backend API Response
    ↓
SQLite Storage (saveTaskInstance)
    ↓
SQLite Load (getTaskInstances)
    ↓
Instance Matching (filter by taskId)
    ↓
Calendar Item Generation (isCompleted = status === 'COMPLETED')
    ↓
Display Conversion (status = isCompleted ? 'COMPLETED' : 'PENDING')
    ↓
UI Rendering (checkbox filled if status === 'COMPLETED')
```

## What to Report
If the instance doesn't show as completed, report:
1. At which check point the logs stop showing expected data
2. Any error messages in console
3. The actual values logged vs expected values