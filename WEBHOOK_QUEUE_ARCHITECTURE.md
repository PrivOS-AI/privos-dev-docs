# Webhook Queue Architecture - Telegram-Style

## Overview

The webhook queue system uses **BullMQ** with **Redis** to handle asynchronous webhook delivery with retry logic, strict FIFO ordering per bot, and no message starvation. This implementation follows **Telegram's architecture**: one queue per bot with independent workers.

## Architecture

### One Queue Per Bot (Telegram-Style)

```
┌─────────────────────────────────────────────────────────────────┐
│                         Redis                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Queue: bot-webhook-user-id-A      Queue: bot-webhook-user-id-B │
│  ┌───────────────────┐             ┌───────────────────┐          │
│  │ Bot A Msg 1      │             │ Bot B Msg 1      │          │
│  │ Bot A Msg 2      │             │ Bot B Msg 2      │          │
│  │ Bot A Msg 3      │             │                  │          │
│  └───────────────────┘             └───────────────────┘          │
│       FIFO Order (per bot)              FIFO Order (per bot)       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
         ▲                                    ▲
         │                                    │
    Worker for Bot A                  Worker for Bot B
    (concurrency: 10)                 (concurrency: 10)
```

### Key Components

#### 1. WebhookQueue Class

**Purpose**: Manages dynamic bot queues and workers

**Key Properties**:
```typescript
class WebhookQueue {
    private botQueues: Map<string, IBotQueueManager> // Bot ID → Queue + Worker
    private connection: Redis                          // Shared Redis connection
    private concurrency: number                        // Jobs per worker (default: 10)
    private maxRetries: number                         // Max retry attempts (default: 24)
    private retryDelay: number                         // Initial retry delay (default: 30000ms)
    private webhookSequenceCounter: Map<string, number> // Track order per webhook
}
```

#### 2. Dynamic Queue & Worker Creation

When a bot first sends a webhook, a queue and worker are created:

```typescript
private getBotQueueManager(botUserId: string): IBotQueueManager {
    if (!this.botQueues.has(botUserId)) {
        const queueName = `bot-webhook-${botUserId}`;

        // Create queue for this bot
        const queue = new Queue(queueName, {
            connection: this.connection,
            defaultJobOptions: {
                attempts: this.maxRetries,
                backoff: { type: 'exponential', delay: this.retryDelay }
            }
        });

        // Create dedicated worker for this bot's queue
        const worker = new Worker(queueName, processWebhook, {
            connection: this.connection,
            concurrency: this.concurrency
        });

        this.botQueues.set(botUserId, { queue, worker, sequenceCounter: 0 });
    }
    return this.botQueues.get(botUserId);
}
```

**Benefits**:
- **Isolation**: Each bot has independent queue and worker
- **No Starvation**: Bot A with 1000 messages doesn't block Bot B with 1 message
- **FIFO Ordering**: Messages for each bot process in strict FIFO order
- **Scalability**: Automatically scales with number of bots

#### 3. Sequence IDs for Ordering

Each webhook gets a unique sequence ID:
```typescript
private getNextSequenceId(webhookId: string): string {
    const current = this.webhookSequenceCounter.get(webhookId) || 0;
    const next = current + 1;
    this.webhookSequenceCounter.set(webhookId, next);
    return `${webhookId}-${next}`;  // e.g., "abc123-1", "abc123-2"
}
```

## Message Ordering

### How FIFO Ordering Works

**Scenario**: Bot A has messages 1, 2, 3, and message 3 fails.

```
Time T1: Messages arrive in order
┌────────────────────────────────────────┐
│ Queue: bot-webhook-user-id-A          │
├────────────────────────────────────────┤
│ Msg 1 → processing                    │
│ Msg 2 → processing                    │
│ Msg 3 → processing (fails)            │
│ Msg 4 → waiting                       │
│ Msg 5 → waiting                       │
└────────────────────────────────────────┘

Time T2: Msg 3 fails, retry scheduled
┌────────────────────────────────────────┐
│ Queue: bot-webhook-user-id-A          │
├────────────────────────────────────────┤
│ Msg 4 → processing ✓                  │ ← New messages process first
│ Msg 5 → processing ✓                  │
│ Msg 3 → retry in 30s                   │ ← Failed job goes to back
└────────────────────────────────────────┘

Time T3: Msg 3 retry completes
┌────────────────────────────────────────┐
│ Queue: bot-webhook-user-id-A          │
├────────────────────────────────────────┤
│ Msg 1 ✓ completed                     │
│ Msg 2 ✓ completed                     │
│ Msg 4 ✓ completed                     │
│ Msg 5 ✓ completed                     │
│ Msg 3 ✓ completed (retry)             │
└────────────────────────────────────────┘
```

### Why This Works

1. **Per-Bot Isolation**: Each bot has its own queue, so Bot A's messages don't affect Bot B
2. **FIFO Within Queue**: Jobs process in order within each bot's queue
3. **Failed Jobs Go to Back**: When a job fails and retries, it goes to the back of that bot's queue
4. **No Global Concurrency Limit**: Each bot's worker has independent concurrency

## Concurrency and Scalability

### How Concurrency Works

```
Bot A (1000 messages):  Queue A → Worker A (concurrency=10) → 10 parallel jobs
Bot B (1 message):     Queue B → Worker B (concurrency=10) → 1 job
Bot C (500 messages):  Queue C → Worker C (concurrency=10) → 10 parallel jobs
```

**Key Points**:
- Each bot has **independent** workers
- Bot B with 1 message is **NOT** blocked by Bot A with 1000 messages
- No starvation - all bots get fair processing

### Scalability

```
Total bots     = botQueues.size
Total queues   = botQueues.size (1 per bot)
Total workers  = botQueues.size (1 per bot)
Max parallel   = botQueues.size × concurrency

Example with 100 bots:
- 100 queues in Redis
- 100 workers (1 per bot)
- 100 × 10 = 1000 parallel webhook deliveries possible
```

## Retry Logic

### Exponential Backoff

| Attempt | Delay      | Formula                        |
|---------|------------|--------------------------------|
| 1       | 30 seconds | 30,000 × 2⁰                    |
| 2       | 60 seconds | 30,000 × 2¹                    |
| 3       | 120 seconds| 30,000 × 2²                    |
| 4       | 240 seconds| 30,000 × 2³                    |
| ...     | ...        | ...                            |
| Max     | 10 minutes | min(30,000 × 2^(n-1), 600,000) |

### Logging

The system provides detailed retry logging:

```
[BOT-WEBHOOK] Webhook job abc123-3 (webhook abc123) failed (attempt 2/24): Connection refused
[BOT-WEBHOOK] Webhook job abc123-3 (webhook abc123) will retry in 60s (exponential backoff)
```

When all retries are exhausted:

```
[BOT-WEBHOOK] Webhook job abc123-3 (webhook abc123) exhausted all retries after 24 attempts
```

### Error Handling

- **Inactive/Missing Webhooks**: Return `{ success: false }` without retrying
- **Network Errors**: Throw error to trigger BullMQ retry
- **Delivery Failures**: Throw error to trigger BullMQ retry

## Job Data Structure

```typescript
interface WebhookJobData {
    webhookId: string;      // ID of the webhook
    userId: string;         // Bot user ID
    event: BotWebhookEvent; // Event type (e.g., 'message.new', 'message.mention')
    payload: any;           // Event payload
    sequenceId: string;     // Unique sequence ID for ordering
}
```

## Webhook Payload Structure

### Standard Events (message, room, user, list, file, folder events)

All standard events share a common structure with room and actor context:

```typescript
{
    event: 'message.new' | 'list.item.created' | 'file.created' | 'folder.created' | ...,
    timestamp: '2025-12-31T10:00:00.000Z',
    bot: {
        id: 'bot-user-id',
        username: 'my-bot',
        name: 'My Bot',
    },
    room: {
        id: 'room-id',
        name: 'general',
        type: 'c',
    },
    actor: {
        id: 'user-id',        // User who triggered event
        username: 'username',
        name: 'User Name'
    },
    // ... event-specific data (message, item, file, folder, etc.)
}
```

**List Item Events Structure**:
```typescript
{
    event: 'list.item.created' | 'list.item.deleted' | 'list.item.stage_changed' | 'list.item.attributes_changed',
    bot: { ... },
    room: { ... },
    item: {
        id: 'item-id',
        name: 'Item name',
        description?: 'Item description'
    },
    list: {
        id: 'list-id',
        name: 'List name'
    },
    stage?: { id: 'stage-id', name: 'Stage name' },
    newStage?: { ... },          // list.item.stage_changed
    previousStage?: { ... },     // list.item.stage_changed (optional)
    changedFields?: { ... },     // list.item.attributes_changed
    changedFieldIds?: [...],     // list.item.attributes_changed (custom fields only)
    actor: { ... }
}
```

**File Events Structure**:
```typescript
{
    event: 'file.created' | 'file.updated' | 'file.deleted',
    bot: { ... },
    room: { ... },
    file: {
        id: 'file-id',
        name: 'filename.pdf',
        size: 2048576,
        type: 'application/pdf',
        channelId: 'channel-id',
        folderId: 'folder-id',
        folderPath: '/Path/To/Folder'  // Display path (not internal MinIO path)
    },
    changes?: {                        // file.updated only
        previousName: 'old-name.pdf',
        newName: 'new-name.pdf',
        previousFolderId: 'old-folder',
        newFolderId: 'new-folder'
    },
    actor: { ... }
}
```

**Folder Events Structure**:
```typescript
{
    event: 'folder.created' | 'folder.deleted' | 'folder.renamed',
    bot: { ... },
    room: { ... },
    folder: {
        id: 'folder-id',
        name: 'Folder name',
        channelId: 'channel-id',
        parentFolderId?: 'parent-id',
        folderPath: '/Path/To/Folder'
    },
    previousName?: 'Old name',         // folder.renamed only
    actor: { ... }
}
```

### Mention Event (message.mention)

The `message.mention` event uses a `form` object instead of `room`, providing additional context for secure room operations:

```typescript
{
    event: 'message.mention',
    timestamp: '2025-12-31T10:00:00.000Z',
    bot: {
        id: 'bot-user-id',
        username: 'my-bot',
        name: 'My Bot',
    },
    form: {
        roomId: 'room-id',
        roomName: 'general',
        roomSessionKey: 'jwt-token-string',       // JWT for secure room session operations
        privosEndpointUrl: 'https://your-server',  // Server URL for API callbacks
        metadata?: {                               // Optional: Present for inline button clicks
            action: 'approve',
            requestId: 'req_001',
            // ... other custom metadata
        }
    },
    message: {
        _id: 'msg-id',
        rid: 'room-id',
        msg: 'Hello @username',
        mentions: [
            { _id: 'user-id', username: 'username' }
        ],
    },
}
```

**Note**: The `metadata` field in `form` is populated when the webhook is triggered by an inline button click (action type `bot-event`). Bots can distinguish button clicks from regular mentions by checking for the presence of `metadata` in the `form` object.

## Environment Variables

| Variable                    | Default | Description                          |
|-----------------------------|---------|--------------------------------------|
| `REDIS_URL`                 | -       | Redis connection URL                 |
| `WEBHOOK_RETRY_ATTEMPTS`    | 24      | Maximum retry attempts               |
| `WEBHOOK_RETRY_INITIAL_DELAY`| 30000   | Initial retry delay in milliseconds  |
| `WEBHOOK_RETRY_MAX_DELAY`   | 600000  | Maximum retry delay in milliseconds (capped at 10 minutes) |

## API Usage

### Adding a Single Webhook Job

```typescript
import { getWebhookQueue } from './bot-webhook-queue/server';

const queue = getWebhookQueue();
await queue.addWebhook(webhookId, userId, 'MessagePosted', payload);
```

### Adding Bulk Webhook Jobs

```typescript
const jobs = [
    { webhookId: 'webhook1', userId: 'bot1', event: 'MessagePosted', payload: {...} },
    { webhookId: 'webhook2', userId: 'bot2', event: 'MessagePosted', payload: {...} },
];

await queue.addBulkWebhooks(jobs);
```

### Getting Queue Statistics

```typescript
const stats = await queue.getStats();
console.log(stats);
// {
//     totalBots: 5,
//     totalWaiting: 12,
//     totalActive: 3,
//     totalCompleted: 145,
//     totalFailed: 2,
//     bots: [
//         { botUserId: 'user-A', waiting: 5, active: 1 },
//         { botUserId: 'user-B', waiting: 7, active: 2 }
//     ]
// }
```

### Delivering to Room Bots

```typescript
import { WebhookDeliveryService } from './bot-webhook-queue/server';

// Standard event
await WebhookDeliveryService.deliverToRoomBots({
    roomId: 'room-id',
    event: 'message.new',
    data: {
        message: { ... },
        sender: { ... },
    },
});

// Mention event (with form context)
await WebhookDeliveryService.deliverToRoomBots({
    roomId: 'room-id',
    event: 'message.mention',
    data: {
        form: { roomId, roomName, roomSessionKey, privosEndpointUrl },
        message: { _id, rid, msg, mentions: [...] },
    },
});
```

## Comparison with Telegram

| Feature          | Telegram         | Our System        |
|------------------|------------------|-------------------|
| Per-Bot Queue    | ✓                | ✓                 |
| FIFO Ordering    | ✓                | ✓                 |
| Retry Logic      | ✓                | ✓ (exponential)   |
| Concurrency      | Multiple bots    | ✓ (configurable)  |
| Dynamic Queues   | ✓                | ✓                 |
| Webhook Signature| ✓ (SHA-256)      | ✓ (HMAC-SHA256)   |

## Benefits Over Alternatives

### vs Single Queue with Priority
- ❌ Priority system complex and unnecessary
- ❌ Priority doesn't guarantee ordering
- ✅ **Per-bot queues**: Simple, natural ordering

### vs Sharded Queues
- ❌ Shared queues = mixed bot messages
- ❌ Bot A with 1000 messages blocks Bot B with 1
- ✅ **Per-bot queues**: Independent processing, no starvation

### vs Flow Producer Groups
- ❌ BullMQ groups don't work as expected
- ❌ Complex API, limited support
- ✅ **Per-bot queues**: Simple, reliable

## Monitoring and Debugging

### Log Patterns

**Success**:
```
[BOT-WEBHOOK] Processing webhook abc123 for event MessagePosted
[BOT-WEBHOOK] Delivering webhook to https://example.com/webhook
[BOT-WEBHOOK] Webhook delivery result: SUCCESS
```

**Failure with Retry**:
```
[BOT-WEBHOOK] Webhook job abc123-3 (webhook abc123) failed (attempt 2/24): Connection refused
[BOT-WEBHOOK] Webhook job abc123-3 (webhook abc123) will retry in 60s (exponential backoff)
```

**Exhausted Retries**:
```
[BOT-WEBHOOK] Webhook job abc123-3 (webhook abc123) exhausted all retries after 24 attempts
```

### Queue Statistics

Monitor queue health using `getStats()`:

```typescript
const stats = await queue.getStats();

// Alert if too many failed jobs
if (stats.totalFailed > 100) {
    logger.error(`High webhook failure rate: ${stats.totalFailed} failed jobs`);
}

// Alert if specific bot has backlog
for (const bot of stats.bots) {
    if (bot.waiting > 1000) {
        logger.warn(`Bot ${bot.botUserId} has ${bot.waiting} waiting jobs`);
    }
}
```

## Graceful Shutdown

```typescript
const queue = getWebhookQueue();

// Close all bot queues and workers
await queue.close();
// - All workers stop processing new jobs
// - All queues are closed
// - Redis connection is closed gracefully
```
