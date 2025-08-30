# Debug Guide: Task Instance Completion

## How to Test and Debug

1. **Create a recurring task** (e.g., daily task)
2. **Mark an instance as complete** by clicking the checkbox
3. **Check the console logs** in the following order:

## Expected Log Flow

### 1. When clicking the checkbox:
```
🔍 DEBUG: handleInstanceAction called with:
  type: 'complete'
  taskId: [task-server-id]
  date: '2025-08-30'
```

### 2. API Response:
```
🔍 DEBUG: Instance result from API:
  {
    id: '...',
    taskId: '...',
    status: 'COMPLETED',
    completedAt: '2025-08-30T...',
    ...
  }
```

### 3. SQLite Save:
```
🔍 DEBUG: Saving to SQLite:
  taskId: [task-server-id]
  instanceDate: '2025-08-30'
  status: 'COMPLETED'
  completedAt: '2025-08-30T...'

🔍 DEBUG SQLite: Saving task instance:
  id: [taskId]_[date]
  status: 'COMPLETED'
  completedAt: [timestamp]
```

### 4. State Update:
```
🔍 DEBUG: Adding new instance (or Updating existing instance)
```

### 5. Loading from SQLite:
```
🔍 DEBUG SQLite: Getting task instances for range
🔍 DEBUG SQLite: Found X task instances
🔍 DEBUG SQLite: Sample raw instance: {...}
🔍 DEBUG SQLite: Sample mapped instance: {
  taskId: [should match task serverId],
  status: 'COMPLETED',
  ...
}
```

### 6. Instance Matching:
```
🔍 DEBUG: Generating instances for task:
  taskId: [task-server-id]
  taskTitle: 'Your Task Title'
  matchingInstancesCount: 1 (or more)

🔍 DEBUG: Matching instances: [
  {
    taskId: [should match],
    instanceDate: '2025-08-30',
    status: 'COMPLETED',
    ...
  }
]
```

### 7. Generate Instances:
```
🔍 DEBUG generateInstances: Found existing instance:
  dateString: '2025-08-30'
  taskId: [task-id]
  instanceStatus: 'COMPLETED'
  isCompleted: true
```

### 8. Convert to Display:
```
🔍 DEBUG convertToDisplay: Converting instance:
  isCompleted: true
  instanceStatus: 'COMPLETED'
  resultStatus: 'COMPLETED'
```

## What to Check

### ✅ Success Indicators:
- Instance saved with status 'COMPLETED' in SQLite
- Instance loaded back with status 'COMPLETED'
- `isCompleted: true` in generateInstances
- `resultStatus: 'COMPLETED'` in convertToDisplay
- UI shows filled checkbox

### ❌ Common Issues:

1. **Task ID Mismatch**:
   - Check if `taskId` in instance matches `task.serverId` or `task.id`
   - Instance might be saved with wrong ID

2. **Status Not Uppercase**:
   - Status should be 'COMPLETED' not 'completed'
   - Check SQLite save and load

3. **Instance Not Found**:
   - `matchingInstancesCount: 0` means instances aren't matching tasks
   - Check the taskId mapping

4. **Date Format Mismatch**:
   - Dates should be in 'YYYY-MM-DD' format
   - Check instanceDate format

5. **Instance Not Persisted**:
   - After reload, check if instances are loaded from SQLite
   - Check if SQLite save was successful

## Quick Troubleshooting

1. **If checkbox doesn't fill after clicking:**
   - Check if API call succeeded
   - Check if instance was saved to SQLite
   - Check if UI refresh was triggered

2. **If checkbox doesn't stay filled after navigation:**
   - Check if instance is loaded from SQLite
   - Check if instance status is 'COMPLETED'
   - Check if taskId matches

3. **If no logs appear:**
   - Make sure you're testing with a recurring task
   - Check if the task has a repeat pattern set