
## 📅 Week 3 — Web Basics & React Thinking

### Exercises

**1. Simple Web Page with Three Cards**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Vision Pipeline Dashboard</title>
    <style>
        body { font-family: Arial, sans-serif; background: #f0f0f0; padding: 20px; }
        .container { display: flex; gap: 20px; justify-content: center; }
        .card { 
            background: white; 
            border-radius: 8px; 
            padding: 24px; 
            width: 250px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.1);
        }
        .card h2 { margin-top: 0; color: #333; }
        .status { font-weight: bold; color: #666; }
    </style>
</head>
<body>
    <div class="container">
        <div class="card">
            <h2>📥 Input</h2>
            <p class="status">Waiting for image...</p>
        </div>
        <div class="card">
            <h2>🔍 Detection</h2>
            <p class="status">Model ready</p>
        </div>
        <div class="card">
            <h2>📤 Output</h2>
            <p class="status">No results yet</p>
        </div>
    </div>
</body>
</html>
```

**2. Button to Change Stage**

Add to the HTML above inside `<body>`:
```html
<button id="startBtn" onclick="changeStage()">Start Pipeline</button>
<p>Current stage: <span id="stage">waiting</span></p>

<script>
    function changeStage() {
        const stageEl = document.getElementById('stage');
        stageEl.textContent = 'running';
        stageEl.style.color = 'green';
        document.getElementById('startBtn').disabled = true;
    }
</script>
```

**3. TypeScript Type Definitions**

```typescript
// types/pipeline.ts

type Stage = 'waiting' | 'running' | 'completed' | 'error';

interface Detection {
    label: string;
    confidence: number;
    box: {
        x: number;
        y: number;
        width: number;
        height: number;
    };
}

interface PipelineEvent {
    id: string;
    timestamp: Date;
    stage: Stage;
    detections: Detection[];
    message: string;
}

interface DashboardState {
    currentStage: Stage;
    events: PipelineEvent[];
    isLoading: boolean;
    error: string | null;
}
```

**4. React Component for Pipeline Events**

```tsx
import React from 'react';
import { PipelineEvent } from './types/pipeline';

interface EventLogProps {
    events: PipelineEvent[];
}

export const EventLog: React.FC<EventLogProps> = ({ events }) => {
    return (
        <div className="event-log">
            <h3>Pipeline Events</h3>
            {events.length === 0 ? (
                <p>No events recorded yet.</p>
            ) : (
                <ul>
                    {events.map(event => (
                        <li key={event.id} className={`event-${event.stage}`}>
                            <span className="timestamp">
                                {event.timestamp.toLocaleTimeString()}
                            </span>
                            <span className="stage-badge">{event.stage}</span>
                            <span className="message">{event.message}</span>
                            {event.detections.length > 0 && (
                                <span className="detection-count">
                                    ({event.detections.length} objects)
                                </span>
                            )}
                        </li>
                    ))}
                </ul>
            )}
        </div>
    );
};
```

**5. Why a Dashboard Needs State**
A dashboard without state is just a static report. State enables:
- **Reactivity**: When new detection events arrive, the UI updates automatically
- **User control**: Toggle filters, expand/collapse panels, change time ranges
- **Consistency**: The stage indicator, event log, and statistics all reflect the same underlying data
- **Feedback**: Loading spinners during API calls, error messages when requests fail
- **History**: Undo/redo, time-travel debugging, session restoration

Without state management, every user interaction would require a full page reload.

---

### Homework: UI State Diagram

```
┌─────────────┐     Click Start     ┌─────────────┐
│             │ ────────────────────→ │             │
│   IDLE      │                     │  LOADING    │
│  (initial)  │ ←────────────────── │  (fetching) │
│             │    API Error        │             │
└──────┬──────┘                     └──────┬──────┘
       │                                    │
       │ Success                            │ Retry
       ▼                                    ▼
┌─────────────┐                     ┌─────────────┐
│   SUCCESS   │ ←────────────────── │   ERROR     │
│  (display   │    Clear/Reset      │  (message   │
│   results)  │ ──────────────────→ │   + retry)  │
└─────────────┘                     └─────────────┘

State Variables:
├── stage: 'idle' | 'loading' | 'success' | 'error'
├── events: PipelineEvent[]
├── selectedEventId: string | null
├── filterLabel: string
└── autoRefresh: boolean

Transitions:
- idle → loading: User clicks "Start Analysis"
- loading → success: API returns 200 with data
- loading → error: API returns 4xx/5xx or network fails
- success → idle: User clicks "Clear Results"
- error → loading: User clicks "Retry"
- Any → Any: User toggles autoRefresh
```
