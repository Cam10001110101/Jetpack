# Technical Design: Task Progress Monitoring & Tracking System

**Date:** 2026-01-14
**Task:** Design solution for monitoring and tracking task progress
**Priority:** High
**Status:** Design Phase

## Executive Summary

This document provides a comprehensive technical design for enhancing Jetpack's monitoring capabilities to provide real-time visibility into task execution, agent activity, and system health. The design builds upon existing infrastructure (RuntimeManager, Agent Registry, MCPMail, CASS) while adding new capabilities for historical metrics, granular progress tracking, and advanced visualizations.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Web UI Dashboard                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Overview │  │ Timeline │  │ Analytics│  │ Dependencies │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘   │
└─────────────────────────────┬───────────────────────────────────┘
                              │ REST API + SSE
                              │
┌─────────────────────────────┴───────────────────────────────────┐
│                      API Layer (Next.js)                         │
│  /api/metrics/* | /api/monitoring/* | /api/messages/stream      │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────┴───────────────────────────────────┐
│                    MetricsCollector Service                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Event        │  │ Time-Series  │  │ Progress             │  │
│  │ Listener     │→│ Database     │→│ Aggregator           │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────┴───────────────────────────────────┐
│                    Existing Infrastructure                       │
│  RuntimeManager | BeadsAdapter | AgentController | MCPMail      │
└──────────────────────────────────────────────────────────────────┘
```

## Phase 1: Foundation (Week 1)

### 1.1 Historical Metrics Database

**New Package:** `@jetpack/metrics-adapter`

**Purpose:** Time-series storage for all runtime metrics with efficient querying

**Database Schema:**

```sql
-- Main metrics table with composite primary key
CREATE TABLE metrics (
  timestamp INTEGER NOT NULL,           -- Unix timestamp in milliseconds
  metric_type TEXT NOT NULL,            -- 'task' | 'agent' | 'system' | 'custom'
  metric_name TEXT NOT NULL,            -- Specific metric identifier
  value REAL NOT NULL,                  -- Numeric value
  metadata TEXT,                        -- JSON string for additional context
  PRIMARY KEY (timestamp, metric_type, metric_name)
) WITHOUT ROWID;

-- Optimized indexes for common queries
CREATE INDEX idx_metrics_type_name_time
  ON metrics(metric_type, metric_name, timestamp DESC);

CREATE INDEX idx_metrics_time
  ON metrics(timestamp DESC);

-- Task events table for detailed tracking
CREATE TABLE task_events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  task_id TEXT NOT NULL,
  event_type TEXT NOT NULL,             -- 'created' | 'claimed' | 'started' | 'completed' | 'failed'
  agent_id TEXT,
  timestamp INTEGER NOT NULL,
  duration_ms INTEGER,                  -- For completed/failed events
  metadata TEXT,                        -- JSON string
  FOREIGN KEY (task_id) REFERENCES tasks(id)
);

CREATE INDEX idx_task_events_task_id ON task_events(task_id);
CREATE INDEX idx_task_events_time ON task_events(timestamp DESC);

-- Agent activity log
CREATE TABLE agent_events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  agent_id TEXT NOT NULL,
  event_type TEXT NOT NULL,             -- 'started' | 'stopped' | 'status_change' | 'heartbeat'
  old_status TEXT,
  new_status TEXT,
  timestamp INTEGER NOT NULL,
  metadata TEXT
);

CREATE INDEX idx_agent_events_agent_id ON agent_events(agent_id);
CREATE INDEX idx_agent_events_time ON agent_events(timestamp DESC);

-- Aggregated statistics (pre-computed for performance)
CREATE TABLE stats_cache (
  stat_key TEXT PRIMARY KEY,            -- e.g., 'tasks_completed_24h'
  stat_value TEXT NOT NULL,             -- JSON string with computed stats
  computed_at INTEGER NOT NULL,         -- Timestamp of computation
  expires_at INTEGER NOT NULL           -- TTL for cache invalidation
);
```

**Implementation Files:**

1. **`packages/metrics-adapter/src/MetricsAdapter.ts`**

```typescript
export interface MetricsConfig {
  workDir: string;
  retentionDays?: number;  // Default: 30 days
  enableAutoCleanup?: boolean;
  cacheEnabled?: boolean;
}

export interface MetricEntry {
  timestamp: number;
  metricType: 'task' | 'agent' | 'system' | 'custom';
  metricName: string;
  value: number;
  metadata?: Record<string, any>;
}

export interface QueryOptions {
  startTime?: number;
  endTime?: number;
  metricTypes?: string[];
  metricNames?: string[];
  limit?: number;
  aggregation?: 'none' | 'avg' | 'sum' | 'count' | 'min' | 'max';
  groupBy?: 'hour' | 'day' | 'week';
}

export class MetricsAdapter {
  private db: Database;
  private metricsFile: string;
  private config: Required<MetricsConfig>;
  private writeBuffer: MetricEntry[] = [];
  private flushInterval: NodeJS.Timeout | null = null;

  constructor(config: MetricsConfig) {
    this.config = {
      retentionDays: 30,
      enableAutoCleanup: true,
      cacheEnabled: true,
      ...config,
    };

    const metricsDir = path.join(this.config.workDir, '.jetpack');
    this.metricsFile = path.join(metricsDir, 'metrics.db');
  }

  async initialize(): Promise<void> {
    // Create directory if needed
    await fs.mkdir(path.dirname(this.metricsFile), { recursive: true });

    // Open database with optimizations
    this.db = new Database(this.metricsFile);
    this.db.pragma('journal_mode = WAL');
    this.db.pragma('synchronous = NORMAL');
    this.db.pragma('cache_size = -64000'); // 64MB cache

    // Create schema
    this.createSchema();

    // Start background tasks
    if (this.config.enableAutoCleanup) {
      this.startCleanupTask();
    }

    // Start buffered writes
    this.startFlushTask();
  }

  private createSchema(): void {
    // Execute SQL schema from above
    this.db.exec(METRICS_SCHEMA);
  }

  // Buffered write for performance
  async recordMetric(entry: MetricEntry): Promise<void> {
    this.writeBuffer.push(entry);

    // Flush immediately if buffer is large
    if (this.writeBuffer.length >= 100) {
      await this.flush();
    }
  }

  private async flush(): Promise<void> {
    if (this.writeBuffer.length === 0) return;

    const entries = this.writeBuffer.splice(0, this.writeBuffer.length);

    const insert = this.db.prepare(`
      INSERT OR REPLACE INTO metrics (timestamp, metric_type, metric_name, value, metadata)
      VALUES (?, ?, ?, ?, ?)
    `);

    const transaction = this.db.transaction((entries: MetricEntry[]) => {
      for (const entry of entries) {
        insert.run(
          entry.timestamp,
          entry.metricType,
          entry.metricName,
          entry.value,
          entry.metadata ? JSON.stringify(entry.metadata) : null
        );
      }
    });

    transaction(entries);
  }

  async query(options: QueryOptions): Promise<MetricEntry[]> {
    const {
      startTime = 0,
      endTime = Date.now(),
      metricTypes,
      metricNames,
      limit = 1000,
      aggregation = 'none',
    } = options;

    let sql = 'SELECT * FROM metrics WHERE timestamp BETWEEN ? AND ?';
    const params: any[] = [startTime, endTime];

    if (metricTypes && metricTypes.length > 0) {
      sql += ` AND metric_type IN (${metricTypes.map(() => '?').join(',')})`;
      params.push(...metricTypes);
    }

    if (metricNames && metricNames.length > 0) {
      sql += ` AND metric_name IN (${metricNames.map(() => '?').join(',')})`;
      params.push(...metricNames);
    }

    if (aggregation !== 'none') {
      sql = `SELECT metric_type, metric_name, ${aggregation}(value) as value,
             MIN(timestamp) as timestamp FROM metrics
             WHERE timestamp BETWEEN ? AND ?`;
      // Add rest of WHERE clause...
      sql += ' GROUP BY metric_type, metric_name';
    } else {
      sql += ' ORDER BY timestamp DESC';
    }

    sql += ` LIMIT ?`;
    params.push(limit);

    const stmt = this.db.prepare(sql);
    const rows = stmt.all(...params);

    return rows.map(row => ({
      timestamp: row.timestamp,
      metricType: row.metric_type,
      metricName: row.metric_name,
      value: row.value,
      metadata: row.metadata ? JSON.parse(row.metadata) : undefined,
    }));
  }

  async recordTaskEvent(event: {
    taskId: string;
    eventType: 'created' | 'claimed' | 'started' | 'completed' | 'failed';
    agentId?: string;
    durationMs?: number;
    metadata?: Record<string, any>;
  }): Promise<void> {
    const stmt = this.db.prepare(`
      INSERT INTO task_events (task_id, event_type, agent_id, timestamp, duration_ms, metadata)
      VALUES (?, ?, ?, ?, ?, ?)
    `);

    stmt.run(
      event.taskId,
      event.eventType,
      event.agentId || null,
      Date.now(),
      event.durationMs || null,
      event.metadata ? JSON.stringify(event.metadata) : null
    );

    // Also record as metric
    await this.recordMetric({
      timestamp: Date.now(),
      metricType: 'task',
      metricName: `task.${event.eventType}`,
      value: 1,
      metadata: { taskId: event.taskId, agentId: event.agentId },
    });
  }

  async getTaskHistory(taskId: string): Promise<any[]> {
    const stmt = this.db.prepare(`
      SELECT * FROM task_events
      WHERE task_id = ?
      ORDER BY timestamp ASC
    `);
    return stmt.all(taskId);
  }

  async getAgentStats(agentId: string, hours: number = 24): Promise<any> {
    const startTime = Date.now() - (hours * 60 * 60 * 1000);

    const stmt = this.db.prepare(`
      SELECT
        COUNT(*) as total_tasks,
        SUM(CASE WHEN event_type = 'completed' THEN 1 ELSE 0 END) as completed,
        SUM(CASE WHEN event_type = 'failed' THEN 1 ELSE 0 END) as failed,
        AVG(CASE WHEN duration_ms IS NOT NULL THEN duration_ms ELSE NULL END) as avg_duration_ms
      FROM task_events
      WHERE agent_id = ? AND timestamp >= ?
    `);

    return stmt.get(agentId, startTime);
  }

  private startCleanupTask(): void {
    // Run cleanup daily
    setInterval(() => {
      const cutoffTime = Date.now() - (this.config.retentionDays * 24 * 60 * 60 * 1000);

      this.db.prepare('DELETE FROM metrics WHERE timestamp < ?').run(cutoffTime);
      this.db.prepare('DELETE FROM task_events WHERE timestamp < ?').run(cutoffTime);
      this.db.prepare('DELETE FROM agent_events WHERE timestamp < ?').run(cutoffTime);
      this.db.prepare('VACUUM').run();

      console.log(`Cleaned up metrics older than ${this.config.retentionDays} days`);
    }, 24 * 60 * 60 * 1000);
  }

  private startFlushTask(): void {
    // Flush every 5 seconds
    this.flushInterval = setInterval(() => {
      this.flush().catch(err => console.error('Flush error:', err));
    }, 5000);
  }

  async close(): Promise<void> {
    if (this.flushInterval) {
      clearInterval(this.flushInterval);
    }

    await this.flush();
    this.db.close();
  }
}
```

2. **`packages/metrics-adapter/src/MetricsCollector.ts`**

```typescript
export class MetricsCollector {
  private metrics: MetricsAdapter;
  private orchestrator: JetpackOrchestrator;
  private subscriptions: Map<string, () => void> = new Map();

  constructor(metrics: MetricsAdapter, orchestrator: JetpackOrchestrator) {
    this.metrics = metrics;
    this.orchestrator = orchestrator;
  }

  async start(): Promise<void> {
    // Subscribe to RuntimeManager events
    const runtime = this.orchestrator.getRuntime();

    runtime.on('cycle_complete', (data) => {
      this.metrics.recordMetric({
        timestamp: Date.now(),
        metricType: 'system',
        metricName: 'cycle.complete',
        value: data.cycleNumber,
        metadata: { duration: data.duration },
      });
    });

    runtime.on('task_complete', (data) => {
      this.metrics.recordTaskEvent({
        taskId: data.taskId,
        eventType: 'completed',
        agentId: data.agentId,
        durationMs: data.duration,
      });
    });

    runtime.on('task_failed', (data) => {
      this.metrics.recordTaskEvent({
        taskId: data.taskId,
        eventType: 'failed',
        agentId: data.agentId,
        metadata: { error: data.error },
      });
    });

    // Subscribe to MCPMail for task events
    await this.subscribeToMailEvents();

    // Start periodic system metrics collection
    this.startSystemMetrics();
  }

  private async subscribeToMailEvents(): Promise<void> {
    // Listen to task.created, task.claimed, agent.* events
    // Record them in metrics database
  }

  private startSystemMetrics(): void {
    setInterval(async () => {
      // Collect system-wide metrics
      const tasks = await this.orchestrator.getTasks();
      const agents = await this.orchestrator.getAgents();

      this.metrics.recordMetric({
        timestamp: Date.now(),
        metricType: 'system',
        metricName: 'tasks.total',
        value: tasks.length,
      });

      this.metrics.recordMetric({
        timestamp: Date.now(),
        metricType: 'system',
        metricName: 'tasks.pending',
        value: tasks.filter(t => t.status === 'pending').length,
      });

      this.metrics.recordMetric({
        timestamp: Date.now(),
        metricType: 'system',
        metricName: 'agents.idle',
        value: agents.filter(a => a.status === 'idle').length,
      });

      // ... more metrics
    }, 10000); // Every 10 seconds
  }

  async stop(): Promise<void> {
    // Cleanup subscriptions
    for (const unsub of this.subscriptions.values()) {
      unsub();
    }
  }
}
```

**Files to Create:**
- `packages/metrics-adapter/src/MetricsAdapter.ts`
- `packages/metrics-adapter/src/MetricsCollector.ts`
- `packages/metrics-adapter/src/index.ts`
- `packages/metrics-adapter/package.json`
- `packages/metrics-adapter/tsconfig.json`

**Files to Modify:**
- `packages/orchestrator/src/JetpackOrchestrator.ts` - Integrate MetricsCollector

### 1.2 Real-Time Task Progress Tracking

**Purpose:** Capture granular progress within task execution

**Data Model Enhancement:**

```typescript
// In packages/shared/src/types/task.ts
export interface TaskProgress {
  percentage: number;          // 0-100
  currentStep: string;          // Human-readable description
  stepNumber: number;           // e.g., 3
  totalSteps: number;           // e.g., 7
  startedAt: Date;
  estimatedCompletionAt: Date | null;
  lastUpdated: Date;
}

export interface Task {
  // ... existing fields
  progress?: TaskProgress;
}
```

**Implementation:**

1. **`packages/orchestrator/src/ProgressTracker.ts`**

```typescript
export class ProgressTracker {
  private progressMap: Map<string, TaskProgress> = new Map();
  private workDir: string;
  private progressDir: string;

  constructor(workDir: string) {
    this.workDir = workDir;
    this.progressDir = path.join(workDir, '.jetpack', 'task-progress');
  }

  async initialize(): Promise<void> {
    await fs.mkdir(this.progressDir, { recursive: true });
  }

  async updateProgress(
    taskId: string,
    update: Partial<TaskProgress>
  ): Promise<void> {
    const existing = this.progressMap.get(taskId) || {
      percentage: 0,
      currentStep: 'Initializing',
      stepNumber: 0,
      totalSteps: 1,
      startedAt: new Date(),
      estimatedCompletionAt: null,
      lastUpdated: new Date(),
    };

    const updated: TaskProgress = {
      ...existing,
      ...update,
      lastUpdated: new Date(),
    };

    // Calculate ETA if we have enough data
    if (updated.stepNumber > 1 && updated.totalSteps > 0) {
      const elapsed = Date.now() - updated.startedAt.getTime();
      const avgTimePerStep = elapsed / updated.stepNumber;
      const remainingSteps = updated.totalSteps - updated.stepNumber;
      const estimatedRemainingMs = avgTimePerStep * remainingSteps;
      updated.estimatedCompletionAt = new Date(Date.now() + estimatedRemainingMs);
    }

    this.progressMap.set(taskId, updated);

    // Persist to disk
    const progressFile = path.join(this.progressDir, `${taskId}.json`);
    await fs.writeFile(progressFile, JSON.stringify(updated, null, 2));

    // Emit event for SSE streaming
    EventEmitter.emit('task.progress', { taskId, progress: updated });
  }

  getProgress(taskId: string): TaskProgress | undefined {
    return this.progressMap.get(taskId);
  }

  async loadProgress(taskId: string): Promise<TaskProgress | null> {
    if (this.progressMap.has(taskId)) {
      return this.progressMap.get(taskId)!;
    }

    const progressFile = path.join(this.progressDir, `${taskId}.json`);
    try {
      const content = await fs.readFile(progressFile, 'utf-8');
      const progress = JSON.parse(content) as TaskProgress;
      this.progressMap.set(taskId, progress);
      return progress;
    } catch {
      return null;
    }
  }

  async clearProgress(taskId: string): Promise<void> {
    this.progressMap.delete(taskId);
    const progressFile = path.join(this.progressDir, `${taskId}.json`);
    await fs.unlink(progressFile).catch(() => {});
  }
}
```

2. **Modify `packages/orchestrator/src/ClaudeCodeExecutor.ts`**

Add progress parsing from Claude Code output:

```typescript
async execute(task: Task, memories: MemoryEntry[]): Promise<ExecutionResult> {
  // ... existing setup

  const progressTracker = this.getProgressTracker();

  await progressTracker.updateProgress(task.id, {
    currentStep: 'Starting Claude Code execution',
    stepNumber: 1,
    totalSteps: 5,
    percentage: 0,
  });

  const claudeProcess = spawn('claude', args, { cwd, stdio: ['pipe', 'pipe', 'pipe'] });

  let stdoutBuffer = '';
  let stderrBuffer = '';
  let currentStep = 1;

  claudeProcess.stdout?.on('data', (data) => {
    const chunk = data.toString();
    stdoutBuffer += chunk;

    // Parse progress indicators from Claude output
    // Look for patterns like "Reading file:", "Analyzing code:", etc.
    const progressPatterns = [
      { pattern: /Reading file:/i, step: 'Reading files', stepNum: 2 },
      { pattern: /Analyzing/i, step: 'Analyzing code', stepNum: 3 },
      { pattern: /Writing/i, step: 'Writing changes', stepNum: 4 },
      { pattern: /Complete/i, step: 'Finalizing', stepNum: 5 },
    ];

    for (const { pattern, step, stepNum } of progressPatterns) {
      if (pattern.test(chunk) && stepNum > currentStep) {
        currentStep = stepNum;
        progressTracker.updateProgress(task.id, {
          currentStep: step,
          stepNumber: stepNum,
          totalSteps: 5,
          percentage: (stepNum / 5) * 100,
        });
        break;
      }
    }
  });

  // ... rest of execution

  await progressTracker.updateProgress(task.id, {
    currentStep: 'Completed',
    stepNumber: 5,
    totalSteps: 5,
    percentage: 100,
  });

  // Clear progress after completion
  setTimeout(() => progressTracker.clearProgress(task.id), 60000); // 1 minute
}
```

**Files to Create:**
- `packages/orchestrator/src/ProgressTracker.ts`

**Files to Modify:**
- `packages/orchestrator/src/ClaudeCodeExecutor.ts`
- `packages/shared/src/types/task.ts`
- `packages/orchestrator/src/JetpackOrchestrator.ts` - Integrate ProgressTracker

## Phase 2: Visualization (Week 2)

### 2.1 Advanced Analytics Dashboard

**Location:** `apps/web/src/app/dashboard/page.tsx`

**Components to Create:**

1. **`apps/web/src/components/dashboard/MetricsOverview.tsx`**
   - Task completion rate (last 24h)
   - Average task duration
   - Agent utilization percentage
   - Current system load

2. **`apps/web/src/components/dashboard/TaskThroughputChart.tsx`**
   - Line chart showing tasks completed per hour
   - Uses Recharts library
   - Time range selector (1h, 6h, 24h, 7d, custom)

3. **`apps/web/src/components/dashboard/AgentUtilizationChart.tsx`**
   - Stacked area chart showing agent status distribution over time
   - Colors: idle (gray), busy (blue), error (red), offline (dark)

4. **`apps/web/src/components/dashboard/TaskDurationHistogram.tsx`**
   - Bar chart showing distribution of task execution times
   - Helps identify outliers and optimization opportunities

5. **`apps/web/src/components/dashboard/LiveActivityTimeline.tsx`**
   - Real-time event stream
   - Shows recent task claims, completions, failures
   - Auto-scrolls with pause button

**API Routes:**

```typescript
// apps/web/src/app/api/metrics/query/route.ts
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const startTime = parseInt(searchParams.get('start') || '0');
  const endTime = parseInt(searchParams.get('end') || Date.now().toString());
  const metricTypes = searchParams.get('types')?.split(',');
  const aggregation = searchParams.get('agg') || 'none';

  const metrics = await getMetricsAdapter();
  const results = await metrics.query({
    startTime,
    endTime,
    metricTypes,
    aggregation,
    limit: 10000,
  });

  return NextResponse.json({ results, count: results.length });
}

// apps/web/src/app/api/metrics/aggregate/route.ts
export async function GET(request: Request) {
  const metrics = await getMetricsAdapter();

  const last24h = {
    tasksCompleted: await metrics.query({
      startTime: Date.now() - 24 * 60 * 60 * 1000,
      metricNames: ['task.completed'],
      aggregation: 'count',
    }),
    tasksFailed: await metrics.query({
      startTime: Date.now() - 24 * 60 * 60 * 1000,
      metricNames: ['task.failed'],
      aggregation: 'count',
    }),
    avgDuration: await getAvgTaskDuration(24),
  };

  return NextResponse.json(last24h);
}
```

**Files to Create:**
- `apps/web/src/app/dashboard/page.tsx`
- `apps/web/src/components/dashboard/MetricsOverview.tsx`
- `apps/web/src/components/dashboard/TaskThroughputChart.tsx`
- `apps/web/src/components/dashboard/AgentUtilizationChart.tsx`
- `apps/web/src/components/dashboard/TaskDurationHistogram.tsx`
- `apps/web/src/components/dashboard/LiveActivityTimeline.tsx`
- `apps/web/src/app/api/metrics/query/route.ts`
- `apps/web/src/app/api/metrics/aggregate/route.ts`

**Files to Modify:**
- `apps/web/src/app/layout.tsx` - Add navigation link to dashboard
- `apps/web/package.json` - Add `recharts` dependency

### 2.2 Task Dependency Graph Visualization

**Location:** New tab in `apps/web/src/app/plans/[id]/page.tsx`

**Component:**

```typescript
// apps/web/src/components/DependencyGraph.tsx
import ReactFlow, {
  Node,
  Edge,
  Background,
  Controls,
  MiniMap
} from 'reactflow';
import 'reactflow/dist/style.css';
import dagre from 'dagre';

export function DependencyGraph({ tasks }: { tasks: Task[] }) {
  // Build graph nodes
  const nodes: Node[] = tasks.map(task => ({
    id: task.id,
    type: 'custom',
    data: {
      label: task.title,
      status: task.status,
      priority: task.priority,
    },
    position: { x: 0, y: 0 }, // Will be calculated
    style: {
      background: getStatusColor(task.status),
      border: task.priority === 'critical' ? '2px solid red' : '1px solid #ddd',
    },
  }));

  // Build edges from dependencies
  const edges: Edge[] = [];
  for (const task of tasks) {
    for (const depId of task.dependencies) {
      edges.push({
        id: `${depId}-${task.id}`,
        source: depId,
        target: task.id,
        animated: task.status === 'in_progress',
        style: { stroke: '#888' },
      });
    }
  }

  // Auto-layout using Dagre
  const layoutedGraph = getLayoutedElements(nodes, edges);

  return (
    <div style={{ height: '600px' }}>
      <ReactFlow
        nodes={layoutedGraph.nodes}
        edges={layoutedGraph.edges}
        fitView
      >
        <Background />
        <Controls />
        <MiniMap />
      </ReactFlow>
    </div>
  );
}

function getLayoutedElements(nodes: Node[], edges: Edge[]) {
  const dagreGraph = new dagre.graphlib.Graph();
  dagreGraph.setDefaultEdgeLabel(() => ({}));
  dagreGraph.setGraph({ rankdir: 'TB' }); // Top to bottom

  nodes.forEach(node => {
    dagreGraph.setNode(node.id, { width: 200, height: 100 });
  });

  edges.forEach(edge => {
    dagreGraph.setEdge(edge.source, edge.target);
  });

  dagre.layout(dagreGraph);

  return {
    nodes: nodes.map(node => {
      const { x, y } = dagreGraph.node(node.id);
      return { ...node, position: { x, y } };
    }),
    edges,
  };
}

function getStatusColor(status: string): string {
  switch (status) {
    case 'completed': return '#10b981';
    case 'in_progress': return '#3b82f6';
    case 'failed': return '#ef4444';
    case 'blocked': return '#f59e0b';
    default: return '#6b7280';
  }
}
```

**Files to Create:**
- `apps/web/src/components/DependencyGraph.tsx`

**Files to Modify:**
- `apps/web/src/app/plans/[id]/page.tsx` - Add graph tab
- `apps/web/package.json` - Add `reactflow`, `dagre`, `@types/dagre`

## Phase 3: Operations (Week 3)

### 3.1 Alerting System

**New Package:** `@jetpack/alerts`

**Configuration Schema:**

```typescript
// packages/alerts/src/types.ts
export interface AlertRule {
  id: string;
  name: string;
  condition: string;           // e.g., "taskFailureRate > 0.2"
  duration: string;            // e.g., "5m"
  severity: 'info' | 'warning' | 'error' | 'critical';
  actions: AlertAction[];
  enabled: boolean;
}

export interface AlertAction {
  type: 'log' | 'file' | 'webhook';
  config: Record<string, any>;
}

export interface Alert {
  id: string;
  ruleId: string;
  severity: string;
  message: string;
  firedAt: Date;
  resolvedAt?: Date;
  metadata: Record<string, any>;
}
```

**Implementation:**

```typescript
// packages/alerts/src/AlertEngine.ts
export class AlertEngine {
  private rules: Map<string, AlertRule> = new Map();
  private activeAlerts: Map<string, Alert> = new Map();
  private metrics: MetricsAdapter;
  private checkInterval: NodeJS.Timeout | null = null;

  async initialize(): Promise<void> {
    await this.loadRules();
    this.startMonitoring();
  }

  private async loadRules(): Promise<void> {
    const configFile = path.join(this.workDir, '.jetpack', 'alerts.json');
    // Load and validate rules
  }

  private startMonitoring(): void {
    this.checkInterval = setInterval(() => {
      this.evaluateRules();
    }, 30000); // Check every 30 seconds
  }

  private async evaluateRules(): Promise<void> {
    for (const rule of this.rules.values()) {
      if (!rule.enabled) continue;

      const triggered = await this.evaluateCondition(rule);

      if (triggered && !this.activeAlerts.has(rule.id)) {
        await this.fireAlert(rule);
      } else if (!triggered && this.activeAlerts.has(rule.id)) {
        await this.resolveAlert(rule.id);
      }
    }
  }

  private async evaluateCondition(rule: AlertRule): Promise<boolean> {
    // Parse condition and query metrics
    // Example: "taskFailureRate > 0.2" over last 5 minutes

    const durationMs = parseDuration(rule.duration);
    const startTime = Date.now() - durationMs;

    // This is simplified - real implementation would parse the condition
    if (rule.condition.includes('taskFailureRate')) {
      const completed = await this.metrics.query({
        startTime,
        metricNames: ['task.completed'],
        aggregation: 'count',
      });

      const failed = await this.metrics.query({
        startTime,
        metricNames: ['task.failed'],
        aggregation: 'count',
      });

      const total = completed[0]?.value + failed[0]?.value || 0;
      const failureRate = total > 0 ? failed[0]?.value / total : 0;

      // Extract threshold from condition
      const threshold = parseFloat(rule.condition.split('>')[1].trim());
      return failureRate > threshold;
    }

    return false;
  }

  private async fireAlert(rule: AlertRule): Promise<void> {
    const alert: Alert = {
      id: generateId(),
      ruleId: rule.id,
      severity: rule.severity,
      message: `Alert: ${rule.name}`,
      firedAt: new Date(),
      metadata: {},
    };

    this.activeAlerts.set(rule.id, alert);

    // Execute actions
    for (const action of rule.actions) {
      await this.executeAction(action, alert);
    }
  }

  private async executeAction(action: AlertAction, alert: Alert): Promise<void> {
    switch (action.type) {
      case 'log':
        console.error(`[ALERT] ${alert.severity}: ${alert.message}`);
        break;

      case 'file':
        const logFile = path.join(this.workDir, '.jetpack', 'alerts.log');
        await fs.appendFile(logFile, JSON.stringify(alert) + '\n');
        break;

      case 'webhook':
        await fetch(action.config.url, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(alert),
        });
        break;
    }
  }
}
```

**Files to Create:**
- `packages/alerts/src/AlertEngine.ts`
- `packages/alerts/src/types.ts`
- `packages/alerts/src/index.ts`
- `.jetpack/alerts.json` (default configuration)

**Files to Modify:**
- `packages/orchestrator/src/JetpackOrchestrator.ts` - Integrate AlertEngine

### 3.2 Audit Trail System

**Implementation:**

```typescript
// packages/orchestrator/src/AuditLogger.ts
export class AuditLogger {
  private auditFile: string;
  private writeStream: WriteStream;

  async initialize(): Promise<void> {
    this.auditFile = path.join(this.workDir, '.jetpack', 'audit.log');
    this.writeStream = createWriteStream(this.auditFile, { flags: 'a' });

    // Set up log rotation (90 days)
    this.setupRotation();
  }

  async log(event: AuditEvent): Promise<void> {
    const entry = {
      timestamp: new Date().toISOString(),
      ...event,
    };

    this.writeStream.write(JSON.stringify(entry) + '\n');
  }

  async search(filters: AuditFilters): Promise<AuditEvent[]> {
    // Read and parse log file with filters
    // This is simplified - production would use streaming for large files
  }

  private setupRotation(): void {
    // Rotate logs daily, keep 90 days
  }
}
```

**Files to Create:**
- `packages/orchestrator/src/AuditLogger.ts`

## API Endpoints Summary

### New Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/metrics/query` | Query time-series metrics |
| GET | `/api/metrics/aggregate` | Get aggregated statistics |
| GET | `/api/metrics/tasks/:id/history` | Get task event history |
| GET | `/api/metrics/agents/:id/stats` | Get agent performance stats |
| GET | `/api/monitoring/progress/:taskId` | Get task progress |
| GET | `/api/monitoring/alerts` | List active alerts |
| POST | `/api/monitoring/alerts/:id/ack` | Acknowledge alert |
| GET | `/api/audit/search` | Search audit logs |
| GET | `/api/audit/export` | Export logs to CSV/JSON |

### Enhanced Endpoints

| Method | Endpoint | Enhancement |
|--------|----------|-------------|
| GET | `/api/messages/stream` | Add progress events |
| GET | `/api/status` | Include metrics summary |
| GET | `/api/tasks/:id` | Include progress field |

## Testing Strategy

### Unit Tests
- MetricsAdapter CRUD operations
- ProgressTracker calculations
- AlertEngine condition evaluation
- Query performance (10k+ records)

### Integration Tests
- End-to-end metrics collection pipeline
- SSE streaming with multiple clients
- Alert triggering and resolution
- Dashboard data loading

### Performance Tests
- Metrics database with 100k+ records
- Concurrent SSE connections (50+ clients)
- Dashboard rendering with large datasets
- Background collection overhead (<1% CPU)

## Dependencies

### New npm packages

```json
{
  "recharts": "^2.10.0",
  "reactflow": "^11.10.0",
  "dagre": "^0.8.5",
  "@types/dagre": "^0.7.52"
}
```

### Existing packages (already in project)
- `better-sqlite3` - Metrics database
- `date-fns` - Time formatting
- `zod` - Schema validation

## Implementation Timeline

### Week 1: Foundation
- **Day 1-2:** MetricsAdapter and schema
- **Day 3:** MetricsCollector integration
- **Day 4-5:** ProgressTracker implementation

### Week 2: Visualization
- **Day 1-2:** API endpoints for metrics
- **Day 3-4:** Dashboard components (charts)
- **Day 5:** Dependency graph visualization

### Week 3: Operations
- **Day 1-3:** AlertEngine and configuration
- **Day 4:** AuditLogger implementation
- **Day 5:** Testing and documentation

## Files Summary

### New Packages
- `packages/metrics-adapter/` (MetricsAdapter, MetricsCollector)
- `packages/alerts/` (AlertEngine)

### New Components
- `apps/web/src/app/dashboard/` (Dashboard page)
- `apps/web/src/components/dashboard/` (Chart components)
- `apps/web/src/components/DependencyGraph.tsx`

### New API Routes
- `apps/web/src/app/api/metrics/query/route.ts`
- `apps/web/src/app/api/metrics/aggregate/route.ts`
- `apps/web/src/app/api/monitoring/progress/[taskId]/route.ts`
- `apps/web/src/app/api/monitoring/alerts/route.ts`

### Modified Files
- `packages/orchestrator/src/JetpackOrchestrator.ts` - Integrate new systems
- `packages/orchestrator/src/ClaudeCodeExecutor.ts` - Add progress tracking
- `packages/shared/src/types/task.ts` - Add progress field
- `apps/web/src/app/layout.tsx` - Add dashboard nav link
- `apps/web/package.json` - Add chart dependencies

### Configuration Files
- `.jetpack/metrics.db` - Time-series database
- `.jetpack/alerts.json` - Alert rules
- `.jetpack/audit.log` - Audit trail
- `.jetpack/task-progress/` - Progress snapshots

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Metrics DB performance | Medium | High | Indexing, retention policy, archival |
| SSE connection drops | Medium | Medium | Auto-reconnect, exponential backoff |
| Dashboard complexity | Medium | Medium | Progressive disclosure, user testing |
| Monitoring overhead | Low | High | Async collection, batching, buffering |
| Disk space growth | Medium | High | Log rotation, cleanup tasks |

## Success Metrics

### Quantitative
- 100% of task state changes captured
- <1% monitoring overhead on execution time
- Dashboard load <2s (p95)
- Metrics query <500ms for 7-day range
- Zero data loss in collection

### Qualitative
- Bottlenecks identifiable in <30 seconds
- Task failures debuggable via audit trail
- Real-time visibility enables intervention
- System health clear at a glance

## Next Steps

1. Review and approve this design
2. Create Beads tasks for each phase
3. Set up feature branches
4. Begin Phase 1 implementation
5. Deploy with staged rollout

---

**Design Status:** ✅ Complete and ready for implementation
**Estimated Effort:** 3 weeks (120-150 hours)
**Risk Level:** Low-Medium (building on solid foundation)
