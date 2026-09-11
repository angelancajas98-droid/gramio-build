# TELEGRAM LOGIC SDK
## Architecture Specification

> **Document ID:** SDK-SPEC-001  
> **Status:** DRAFT  
> **Scope:** SDK Core, Logic Modules, Plugin Interface, Workflow Engine, Action Engine, State Engine, Telegram Adapter  
> **Design Principle:** Telegram is the platform. This SDK is only the logic.

---

---

# PART 1 — ARCHITECTURAL OVERVIEW

---

## 1.1 DESIGN PHILOSOPHY

The SDK does not own or manage a Telegram bot.  
The SDK does not provision bots, manage tokens, or maintain runtime state.  
The SDK assumes it is already running for a configured bot.

The SDK is a **reusable logic engine** that:

- Receives normalized events from any messaging adapter
- Resolves context before any logic executes
- Routes events to the correct logic module
- Executes workflows composed of discrete steps
- Returns first-class action objects
- Delegates transport execution to the adapter

This makes the SDK transport-independent except at the adapter boundary.

---

## 1.2 THE CORE ARCHITECTURE

```
Telegram Platform
        │
        ▼
┌───────────────────┐
│ Telegram Adapter  │  ◄── only layer that knows about Telegram
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Event Engine     │  ◄── normalizes raw updates into internal events
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Context Engine   │  ◄── resolves bot, chat, user, session, permissions
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Logic Router     │  ◄── matches event+context to a Logic Module
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Logic Module     │  ◄── plugin; returns a workflow or direct actions
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Workflow Engine  │  ◄── executes multi-step, stateful workflows
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Action Engine    │  ◄── validates and dispatches first-class actions
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  State Engine     │  ◄── persists domain state, not Telegram state
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│  Telegram Adapter │  ◄── translates actions into Bot API calls
└────────┬──────────┘
         │
         ▼
Telegram Platform
```

---

## 1.3 SDK CORE COMPONENTS

```
SDK Core
├── Event Engine
├── Context Engine
├── Logic Router
├── Workflow Engine
├── Action Engine
├── State Engine
└── Telegram Adapter
```

Everything else is a plugin.

```
plugins/
├── content/
├── community/
├── support/
├── scheduler/
├── automation/
├── broadcast/
├── analytics/
└── audit/
```

---

---

# PART 2 — SDK CORE SPECIFICATION

---

## 2.1 TELEGRAM ADAPTER (INGRESS)

The Adapter is the only component that communicates with Telegram.  
On ingress, it receives raw Telegram updates and forwards them inward.  
On egress, it receives Action objects and translates them into Bot API calls.

```
╔══════════════════════════════╗
║ TELEGRAM PLATFORM            ║
╚══════════════╦═══════════════╝
               │
               │  Raw Telegram Update
               ▼
┌──────────────────────────────┐
│ TELEGRAM ADAPTER             │
│                              │
│  Ingress                     │
│  ─────────────────────────   │
│  Receive Update              │
│  Strip transport metadata    │
│  Forward raw payload         │
└──────────────┬───────────────┘
               │
               │  Raw Update Payload
               ▼
┌──────────────────────────────┐
│  EVENT ENGINE                │
└──────────────────────────────┘
```

**Architectural constraint:**

```
TELEGRAM ADAPTER (ingress)
      │
      └── MUST NOT ──► Logic Router
      └── MUST NOT ──► Logic Modules
      └── MUST      ──► Event Engine
```

---

## 2.2 EVENT ENGINE

The Event Engine normalizes any raw update into a typed, platform-independent Internal Event.

```
┌──────────────────────────────┐
│  EVENT ENGINE                │
├──────────────────────────────┤
│  Input:  Raw Update Payload  │
│  Output: InternalEvent       │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────┐
│  InternalEvent                                       │
├──────────────────────────────────────────────────────┤
│  event_id          : UUID                            │
│  event_type        : EventType                       │
│  source            : AdapterSource                   │
│  actor_ref         : TelegramUserRef                 │
│  chat_ref          : TelegramChatRef                 │
│  message_ref       : TelegramMessageRef | null       │
│  payload           : EventPayload                    │
│  timestamp         : ISO8601                         │
│  correlation_id    : UUID                            │
└──────────────────────────────────────────────────────┘
```

**Event types:**

```
EventType
├── COMMAND
├── CALLBACK_QUERY
├── MESSAGE
├── POLL_ANSWER
├── JOIN_REQUEST
├── MEMBER_EVENT
├── SCHEDULED
└── SYSTEM
```

**Constraint:**

```
Raw Telegram Update
        │
        ├── X ──► Logic Router
        ├── X ──► Logic Modules
        └── ✓ ──► Event Engine only
```

---

## 2.3 CONTEXT ENGINE

The Context Engine runs after normalization and before routing.  
No logic module ever executes without a resolved context.

```
┌──────────────────────────────┐
│  InternalEvent               │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  CONTEXT ENGINE              │
│                              │
│  Resolve:                    │
│  ─────────────────────────   │
│  bot                         │
│  chat                        │
│  user                        │
│  permissions                 │
│  session / conversation      │
│  locale                      │
│  active workflow             │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────┐
│  ResolvedContext                                     │
├──────────────────────────────────────────────────────┤
│  bot          : BotConfig                            │
│  chat         : ChatContext                          │
│  user         : UserContext                          │
│  permissions  : PermissionSet                        │
│  session      : Session | null                       │
│  locale       : Locale                               │
│  workflow     : ActiveWorkflow | null                │
└──────────────────────────────────────────────────────┘
```

**Resolution order:**

```
InternalEvent
      │
      ├── 1. Resolve Bot config
      ├── 2. Resolve Chat context
      ├── 3. Resolve User context
      ├── 4. Evaluate Permissions
      ├── 5. Load Session / Conversation state
      ├── 6. Detect Locale
      └── 7. Check for active Workflow
      │
      ▼
ResolvedContext  ──►  Logic Router
```

---

## 2.4 LOGIC ROUTER

The Logic Router receives an InternalEvent and a ResolvedContext, then selects the correct Logic Module.

```
┌──────────────────────────────┐
│  InternalEvent               │
│  ResolvedContext             │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  LOGIC ROUTER                │
│                              │
│  1. Check active workflow    │
│     → resume if exists       │
│                              │
│  2. Match against modules    │
│     → call module.match()    │
│                              │
│  3. Select first match       │
│                              │
│  4. Dispatch to module       │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Logic Module                │
└──────────────────────────────┘
```

**Active workflow priority:**

```
ResolvedContext.workflow != null
        │
        ▼
RESUME WORKFLOW  ──►  Workflow Engine
        │
        (skip module matching entirely)
```

---

## 2.5 LOGIC MODULE INTERFACE

Every feature is a plugin that implements a single interface.  
No module has privileged access to the transport layer.

```typescript
interface LogicModule {
  /**
   * Returns true if this module should handle the event.
   * Evaluated against the normalized event and resolved context.
   */
  match(event: InternalEvent, context: ResolvedContext): boolean

  /**
   * Executes the module logic.
   * Returns a Workflow, a list of Actions, or null.
   */
  execute(
    event: InternalEvent,
    context: ResolvedContext
  ): Workflow | Action[] | null

  /**
   * Declares which Action types this module may produce.
   * Used by the Action Engine for validation.
   */
  actions(): ActionType[]
}
```

**Constraint:**

```
Logic Module
      │
      ├── MUST NOT ──► Telegram Adapter
      ├── MUST NOT ──► Bot API
      ├── MAY      ──► State Engine  (read/write domain state)
      └── MUST     ──► return Actions or Workflow
```

---

## 2.6 WORKFLOW ENGINE

A Workflow is a multi-step, stateful execution sequence.  
The Workflow Engine drives each step and persists intermediate state.

```
┌──────────────────────────────┐
│  Logic Module                │
│  returns: Workflow           │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  WORKFLOW ENGINE             │
│                              │
│  Load workflow definition    │
│  Restore workflow state      │
│  Execute current step        │
│  Evaluate step result        │
│  Advance or terminate        │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────┐
│  WorkflowStep result                                 │
├──────────────────────────────────────────────────────┤
│  actions   : Action[]                                │
│  next_step : StepID | null                           │
│  state     : WorkflowState                           │
└──────────────────────────────────────────────────────┘
```

**Step execution model:**

```
Workflow
  │
  ├── Step 1
  │     ├── execute()
  │     ├── return Action[]
  │     └── next: Step 2
  │
  ├── Step 2
  │     ├── WAIT (awaiting user input)
  │     ├── event arrives → resume
  │     └── next: Step 3
  │
  └── Step 3
        ├── execute()
        ├── return Action[]
        └── next: null (terminal)
```

**Persistence:**

```
WORKFLOW STATE
      │
      ▼
STATE ENGINE
      │
      ▼
PERSISTED

PROCESS RESTART
      │
      ▼
STATE ENGINE
      │
      ▼
NON-TERMINAL WORKFLOWS RESTORED
```

---

## 2.7 ACTION ENGINE

Actions are first-class objects. Modules never call Telegram directly.  
The Action Engine validates, sequences, and dispatches all actions.

```
┌──────────────────────────────┐
│  Action[]                    │
│  (from module or workflow)   │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  ACTION ENGINE               │
│                              │
│  1. Validate action types    │
│     against module.actions() │
│                              │
│  2. Check approval gate      │
│     (if action is protected) │
│                              │
│  3. Sequence actions         │
│                              │
│  4. Dispatch to adapter      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  TELEGRAM ADAPTER (egress)   │
└──────────────────────────────┘
```

**Action catalogue:**

```
ActionType
├── SendMessage
├── EditMessage
├── DeleteMessage
├── AnswerCallbackQuery
├── AnswerInlineQuery
├── BanMember
├── UnbanMember
├── MuteMember
├── ApproveJoinRequest
├── DeclineJoinRequest
├── PinMessage
├── CreateTopic
├── PublishPost
├── ForwardMessage
├── SendPoll
└── SendFile
```

**Action object structure:**

```typescript
interface Action {
  type       : ActionType
  target_ref : TelegramChatRef | TelegramUserRef | TelegramMessageRef
  payload    : ActionPayload
  requires_approval : boolean
  correlation_id    : UUID
}
```

**Approval gate:**

```
Action.requires_approval == true
        │
        ▼
APPROVAL WORKFLOW
        │
        ├── APPROVED ──► dispatch to adapter
        │
        └── REJECTED ──► cancel, emit AuditRecord
```

---

## 2.8 STATE ENGINE

The State Engine stores business domain state.  
It does not store Telegram transport state.

```
┌──────────────────────────────────────────────────────┐
│  STATE ENGINE                                        │
├──────────────────────────────────────────────────────┤
│  Domain entities (stored)                            │
│  ─────────────────────────────────────────────────   │
│  Ticket                                              │
│  Content                                             │
│  Broadcast                                           │
│  Reminder                                            │
│  Poll                                                │
│  Workflow                                            │
│  Session                                             │
│  AuditRecord                                         │
│                                                      │
│  Telegram references (minimal, stored as refs only)  │
│  ─────────────────────────────────────────────────   │
│  TelegramUserRef     { remote_id, platform_alias }   │
│  TelegramChatRef     { remote_id, chat_type }        │
│  TelegramMessageRef  { remote_id, chat_ref }         │
└──────────────────────────────────────────────────────┘
```

**Constraint:**

```
STATE ENGINE
      │
      ├── MUST NOT mirror Telegram member databases
      ├── MUST NOT store full Telegram user profiles
      └── MUST store only what the SDK logic requires
```

---

## 2.9 TELEGRAM ADAPTER (EGRESS)

```
┌──────────────────────────────┐
│  Action[]                    │
│  (dispatched by Action Engine│
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  TELEGRAM ADAPTER            │
│                              │
│  Egress                      │
│  ─────────────────────────   │
│  Translate Action → API call │
│  Apply pacing                │
│  Handle rate limits          │
│  Retry on failure            │
│  Return result               │
└──────────────┬───────────────┘
               │
               ▼
╔══════════════════════════════╗
║ TELEGRAM PLATFORM            ║
╚══════════════════════════════╝
```

**Transport implementation is hidden from the SDK:**

```
ACTION ENGINE
      │
      ▼
TELEGRAM ADAPTER INTERFACE
      │
      ▼
[ Bot API implementation hidden ]
[ Webhooks vs polling hidden     ]
[ Retry strategy hidden          ]
```

---

---

# PART 3 — PLUGIN SPECIFICATION

---

## 3.1 PLUGIN INTERFACE CONTRACT

Every plugin is a Logic Module. The interface is the contract.

```typescript
interface LogicModule {
  id       : string          // unique plugin identifier
  version  : string          // semver

  match(
    event   : InternalEvent,
    context : ResolvedContext
  ): boolean

  execute(
    event   : InternalEvent,
    context : ResolvedContext
  ): Workflow | Action[] | null

  actions(): ActionType[]
}
```

Plugin registration:

```
SDK Core
  │
  ▼
Plugin Registry
  │
  ├── register(module: LogicModule)
  ├── resolve(event, context) → LogicModule | null
  └── list() → LogicModule[]
```

---

## 3.2 PLUGIN DIRECTORY

```
plugins/
│
├── content/         Content creation, versioning, approval, publication
├── community/       Join requests, member events, moderation rules
├── support/         Support tickets, assignment, resolution lifecycle
├── scheduler/       Scheduled task creation and execution triggering
├── automation/      Rule evaluation engine, trigger-action pairs
├── broadcast/       Audience targeting, delivery job management
├── analytics/       Telemetry collection (non-blocking)
└── audit/           Immutable audit record generation
```

---

## 3.3 PLUGIN: CONTENT

```
match:
  event_type IN [COMMAND, CALLBACK_QUERY]
  AND command IN content commands
  AND context.permissions ALLOWS content

execute:
  ├── CREATE   → Workflow(content_create_workflow)
  ├── EDIT     → Workflow(content_edit_workflow)
  ├── PREVIEW  → Action[SendMessage(preview)]
  ├── APPROVE  → Workflow(approval_workflow)
  └── PUBLISH  → Workflow(publish_workflow)

actions:
  [SendMessage, EditMessage, DeleteMessage, PublishPost, PinMessage]
```

**Content workflow:**

```
content_create_workflow
  │
  ├── Step 1: Collect content input
  │     actions: [SendMessage(prompt)]
  │     next: Step 2
  │
  ├── Step 2: Edit / Preview
  │     WAIT for user input
  │     actions: [SendMessage(preview)]
  │     next: Step 3
  │
  ├── Step 3: Approval gate (if required)
  │     Action[PublishPost].requires_approval = true
  │     next: Step 4
  │
  └── Step 4: Publish
        actions: [PublishPost, SendMessage(confirmation)]
        next: null (terminal)
```

**Domain state:**

```typescript
interface Content {
  id          : UUID
  versions    : ContentVersion[]
  state       : 'draft' | 'pending' | 'approved' | 'published' | 'archived'
  chat_ref    : TelegramChatRef
  created_at  : ISO8601
  updated_at  : ISO8601
}

interface ContentVersion {
  version_id  : UUID
  payload     : string
  state       : 'draft' | 'preview' | 'approved'
  created_at  : ISO8601
}
```

---

## 3.4 PLUGIN: COMMUNITY

```
match:
  event_type IN [JOIN_REQUEST, MEMBER_EVENT]

execute:
  ├── JOIN_REQUEST   → Workflow(join_request_workflow)
  └── MEMBER_EVENT   → Rule evaluation → Action[] | null

actions:
  [ApproveJoinRequest, DeclineJoinRequest, BanMember,
   UnbanMember, MuteMember, SendMessage]
```

**Join request workflow:**

```
join_request_workflow
  │
  ├── Step 1: Evaluate auto-approval rules
  │     → if match: Action[ApproveJoinRequest]
  │     → if no match: next Step 2
  │
  ├── Step 2: Notify owner
  │     actions: [SendMessage(notification to owner)]
  │     WAIT for owner decision
  │
  └── Step 3: Execute decision
        ├── APPROVE → Action[ApproveJoinRequest]
        └── REJECT  → Action[DeclineJoinRequest]
```

**Domain state:**

```typescript
interface CommunityRule {
  id         : UUID
  trigger    : RuleTrigger
  conditions : Condition[]
  action     : ActionType
}
```

---

## 3.5 PLUGIN: SUPPORT

```
match:
  event_type == MESSAGE
  AND chat.type == PRIVATE
  AND NOT active_workflow

execute:
  → Workflow(support_ticket_workflow)

actions:
  [SendMessage, EditMessage]
```

**Support ticket workflow:**

```
support_ticket_workflow
  │
  ├── Step 1: Create ticket
  │     state: Ticket { state: 'open' }
  │     actions: [SendMessage(acknowledgement to user)]
  │     next: Step 2
  │
  ├── Step 2: Notify owner / assign
  │     state: Ticket { state: 'assigned' }
  │     actions: [SendMessage(notification to owner)]
  │     WAIT for owner reply
  │
  ├── Step 3: Owner replies
  │     actions: [SendMessage(reply to user)]
  │     next: Step 4 or loop Step 2
  │
  └── Step 4: Resolve
        state: Ticket { state: 'resolved' }
        actions: [SendMessage(resolution to user)]
        next: null (terminal)
```

**Domain state:**

```typescript
interface Ticket {
  id          : UUID
  user_ref    : TelegramUserRef
  chat_ref    : TelegramChatRef
  state       : 'open' | 'assigned' | 'in_progress' | 'resolved' | 'closed'
  messages    : TicketMessage[]
  created_at  : ISO8601
  resolved_at : ISO8601 | null
}
```

---

## 3.6 PLUGIN: SCHEDULER

```
match:
  event_type == SCHEDULED

execute:
  → Resolve target Logic Module from task definition
  → Delegate execution to target module

actions:
  []  (Scheduler does not produce actions directly)
```

**Scheduler model:**

```typescript
interface ScheduledTask {
  id             : UUID
  target_module  : string      // Logic Module ID
  trigger_time   : ISO8601
  payload        : TaskPayload
  state          : 'pending' | 'triggered' | 'completed' | 'failed'
}
```

**Execution path:**

```
ScheduledTask.trigger_time reached
        │
        ▼
Synthetic SCHEDULED event created
        │
        ▼
Event Engine → Context Engine → Logic Router
        │
        ▼
Scheduler Plugin matches
        │
        ▼
Resolves target Logic Module
        │
        ▼
Target module executes
        │
        ▼
Actions → Action Engine → Adapter
```

**Constraint:**

```
SCHEDULER PLUGIN
      │
      └── MUST NOT ──► Telegram Adapter directly
      └── MUST      ──► Synthetic event → Logic Router
```

---

## 3.7 PLUGIN: AUTOMATION

```
match:
  any event_type
  (evaluated after all other modules if no match found)

execute:
  ├── Evaluate rules against event + context
  ├── If rule matches → resolve target action or module
  └── If no rule matches → null

actions:
  [SendMessage, DeleteMessage, BanMember, MuteMember]
```

**Rule evaluation:**

```
InternalEvent + ResolvedContext
        │
        ▼
RULE ENGINE
        │
        ├── Rule 1: evaluate conditions → false → skip
        ├── Rule 2: evaluate conditions → false → skip
        └── Rule 3: evaluate conditions → true
                │
                ▼
             ACTION or MODULE DISPATCH
```

**Domain state:**

```typescript
interface AutomationRule {
  id          : UUID
  name        : string
  trigger     : EventType
  conditions  : Condition[]
  action      : ActionType | ModuleID
  is_active   : boolean
}
```

---

## 3.8 PLUGIN: BROADCAST

```
match:
  event_type == COMMAND
  AND command IN broadcast commands
  AND context.permissions ALLOWS broadcast

execute:
  → Workflow(broadcast_workflow)

actions:
  [SendMessage, ForwardMessage, SendPoll, SendFile]
```

**Broadcast workflow:**

```
broadcast_workflow
  │
  ├── Step 1: Define payload
  │     WAIT for content input
  │     next: Step 2
  │
  ├── Step 2: Select audience
  │     WAIT for audience selection
  │     next: Step 3
  │
  ├── Step 3: Approval gate
  │     Action[SendMessage * N].requires_approval = true
  │     next: Step 4
  │
  └── Step 4: Execute delivery
        actions: [SendMessage × audience]
        next: null (terminal)
```

**Separation of concerns:**

```
Broadcast Plugin
  │
  ├── Targeting logic         (plugin responsibility)
  ├── Payload construction    (plugin responsibility)
  └── Produces Action[]       (then delegates)
        │
        ▼
Action Engine
        │
        ├── Pacing            (Action Engine / Adapter responsibility)
        ├── Rate limiting     (Adapter responsibility)
        └── Retry             (Adapter responsibility)
```

**Domain state:**

```typescript
interface Broadcast {
  id           : UUID
  payload      : BroadcastPayload
  audience     : TelegramChatRef[]
  state        : 'draft' | 'pending' | 'approved' | 'delivering' | 'delivered'
  created_at   : ISO8601
  delivered_at : ISO8601 | null
}
```

---

## 3.9 PLUGIN: ANALYTICS

```
match:
  never (analytics is not a primary logic handler)

execute:
  invoked as a side-effect hook, not by the Logic Router directly

actions:
  []  (analytics produces no Telegram actions)
```

**Non-blocking telemetry model:**

```
Logic Module
  │
  ▼
Core Workflow Completes
  │
  ├────────────────────────────► BUSINESS RESULT
  │
  └── emit TelemetryEvent (non-blocking, fire-and-forget)
          │
          ▼
    Analytics Plugin
          │
          ▼
    Metric aggregation

ANALYTICS FAILURE
      │
      └── MUST NOT affect core workflow result
```

**Tracked events:**

```
TelemetryEvent
├── content.published
├── ticket.created
├── ticket.resolved
├── broadcast.delivered
├── join_request.approved
├── join_request.declined
├── workflow.completed
├── workflow.failed
└── action.dispatched
```

---

## 3.10 PLUGIN: AUDIT

```
match:
  never (audit is not a primary logic handler)

execute:
  invoked on every state mutation, not by Logic Router

actions:
  []  (audit produces no Telegram actions)
```

**Immutable audit model:**

```
State Mutation (any domain entity)
        │
        ▼
AUDIT PLUGIN (hook)
        │
        ▼
┌──────────────────────────────────────────────────────┐
│  AuditRecord                                         │
├──────────────────────────────────────────────────────┤
│  id           : UUID                                 │
│  actor        : TelegramUserRef | 'system'           │
│  action       : string                               │
│  target       : DomainEntity reference               │
│  timestamp    : ISO8601                              │
│  result       : 'success' | 'failure'                │
│  payload      : AuditPayload                         │
└──────────────────────────────────────────────────────┘
        │
        ▼
IMMUTABLE  (no update, no delete)
```

---

---

# PART 4 — DATA MODEL

---

## 4.1 TRANSPORT REFERENCES (Telegram-specific, minimal)

```typescript
interface TelegramUserRef {
  remote_id      : number         // Telegram user ID
  platform_alias : string | null  // username if available
}

interface TelegramChatRef {
  remote_id  : number
  chat_type  : 'private' | 'group' | 'supergroup' | 'channel'
}

interface TelegramMessageRef {
  remote_id  : number
  chat_ref   : TelegramChatRef
}
```

These are references, not profiles. The SDK does not mirror Telegram's user database.

---

## 4.2 DOMAIN ENTITIES (Business state, SDK-owned)

```typescript
interface Ticket {
  id, user_ref, chat_ref, state, messages, created_at, resolved_at
}

interface Content {
  id, versions, state, chat_ref, created_at, updated_at
}

interface ContentVersion {
  version_id, payload, state, created_at
}

interface Broadcast {
  id, payload, audience, state, created_at, delivered_at
}

interface Reminder {
  id, user_ref, chat_ref, message, trigger_time, state
}

interface Poll {
  id, question, options, chat_ref, state, results
}

interface Workflow {
  id, module_id, current_step, state, context_snapshot, created_at, updated_at
}

interface Session {
  id, user_ref, chat_ref, data, expires_at
}

interface AuditRecord {
  id, actor, action, target, timestamp, result, payload
}

interface ScheduledTask {
  id, target_module, trigger_time, payload, state
}

interface AutomationRule {
  id, name, trigger, conditions, action, is_active
}

interface CommunityRule {
  id, trigger, conditions, action
}
```

---

## 4.3 CONTEXT OBJECTS (Runtime-only, not persisted)

```typescript
interface ResolvedContext {
  bot         : BotConfig
  chat        : ChatContext
  user        : UserContext
  permissions : PermissionSet
  session     : Session | null
  locale      : Locale
  workflow    : ActiveWorkflow | null
}

interface BotConfig {
  bot_id   : string
  features : string[]    // enabled plugin IDs
  locale   : Locale
}

interface UserContext {
  ref          : TelegramUserRef
  access_level : 'owner' | 'admin' | 'member' | 'guest'
}

interface PermissionSet {
  allowed_actions  : ActionType[]
  allowed_modules  : string[]
}

interface ActiveWorkflow {
  workflow_id  : UUID
  module_id    : string
  current_step : string
  state        : WorkflowState
}
```

---

---

# PART 5 — COMPLETE END-TO-END FLOWS

---

## 5.1 STANDARD EVENT FLOW

```
╔══════════════════════════════╗
║ TELEGRAM PLATFORM            ║
╚══════════════╦═══════════════╝
               │  Raw Update
               ▼
┌──────────────────────────────┐
│  TELEGRAM ADAPTER (ingress)  │
└──────────────┬───────────────┘
               │  Raw payload
               ▼
┌──────────────────────────────┐
│  EVENT ENGINE                │
│  → InternalEvent             │
└──────────────┬───────────────┘
               │  InternalEvent
               ▼
┌──────────────────────────────┐
│  CONTEXT ENGINE              │
│  → ResolvedContext           │
└──────────────┬───────────────┘
               │  InternalEvent + ResolvedContext
               ▼
┌──────────────────────────────┐
│  LOGIC ROUTER                │
│  → module.match() → dispatch │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  LOGIC MODULE                │
│  → Action[] or Workflow      │
└──────────────┬───────────────┘
               │
       ┌───────┴───────┐
       │               │
       ▼               ▼
 Action[]          Workflow
       │               │
       │               ▼
       │    ┌──────────────────────┐
       │    │  WORKFLOW ENGINE     │
       │    │  → Step execution    │
       │    │  → Action[]         │
       │    └──────────┬───────────┘
       │               │
       └───────┬───────┘
               │  Action[]
               ▼
┌──────────────────────────────┐
│  ACTION ENGINE               │
│  → validate                  │
│  → approval gate (if needed) │
│  → dispatch                  │
└──────────────┬───────────────┘
               │  validated Action[]
               ▼
┌──────────────────────────────┐
│  STATE ENGINE                │
│  → persist domain state      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  TELEGRAM ADAPTER (egress)   │
│  → translate Action → API    │
│  → pacing, rate limit, retry │
└──────────────┬───────────────┘
               │
               ▼
╔══════════════════════════════╗
║ TELEGRAM PLATFORM            ║
╚══════════════════════════════╝
               │
       ┌───────┴───────┐
       │               │
       ▼               ▼
 TelemetryEvent    AuditRecord
       │               │
       ▼               ▼
 Analytics         Immutable log
 (non-blocking)
```

---

## 5.2 WORKFLOW RESUME FLOW

```
Telegram Update arrives
        │
        ▼
Event Engine → InternalEvent
        │
        ▼
Context Engine
        │
        ▼
context.workflow != null ?
        │
        ├── YES
        │     │
        │     ▼
        │   Logic Router
        │     │
        │     └── Skip module matching
        │           │
        │           ▼
        │         Workflow Engine
        │           │
        │           └── Resume at current_step
        │
        └── NO
              │
              ▼
            Logic Router (normal matching)
```

---

## 5.3 APPROVAL GATE FLOW

```
Action Engine receives Action
        │
        ▼
Action.requires_approval == true ?
        │
        ├── NO  ──► dispatch to adapter immediately
        │
        └── YES
              │
              ▼
        APPROVAL WORKFLOW
              │
              ├── Notify owner
              │     Action[SendMessage(approval request)]
              │
              ├── WAIT for owner decision
              │
              ├── APPROVED
              │     │
              │     └── dispatch original Action to adapter
              │
              └── REJECTED
                    │
                    └── cancel Action
                          │
                          └── emit AuditRecord
```

---

## 5.4 ANALYTICS AND AUDIT SIDE-EFFECTS

```
Core Workflow
      │
      ├── completes successfully
      │         │
      │         ├──── emit TelemetryEvent (non-blocking)
      │         │           │
      │         │           └── Analytics Plugin
      │         │                   │
      │         │                   └── MUST NOT block or reverse core flow
      │         │
      │         └──── State Engine persists mutation
      │                     │
      │                     └── Audit Plugin (hook)
      │                               │
      │                               └── AuditRecord (IMMUTABLE)
      │
      └── core result delivered to Telegram
```

---

---

# PART 6 — ARCHITECTURAL RULES

---

## 6.1 HARD BOUNDARIES

```
Rule 1: Telegram Adapter is the only component that communicates with Telegram.

Rule 2: Raw Telegram updates MUST NOT enter the Logic Router or any Logic Module.

Rule 3: Logic Modules MUST NOT call the Telegram Adapter directly.

Rule 4: Logic Modules MUST NOT call each other directly.

Rule 5: The Context Engine MUST run before any Logic Module executes.

Rule 6: The Workflow Engine MUST resume an active workflow
        before the Logic Router evaluates module matches.

Rule 7: Modules declare their permitted action types via actions().
        The Action Engine enforces this at dispatch time.

Rule 8: Analytics telemetry MUST be non-blocking.
        Analytics failure MUST NOT reverse a completed workflow.

Rule 9: Audit records are IMMUTABLE once written.

Rule 10: The State Engine MUST NOT store full Telegram user profiles.
         Only TelegramUserRef, TelegramChatRef, TelegramMessageRef are permitted.
```

---

## 6.2 EXTENSIBILITY RULES

```
Rule E1: Any new feature MUST be implemented as a Logic Module plugin.
         No new features are added to SDK Core.

Rule E2: A Logic Module plugin MUST implement the full LogicModule interface.

Rule E3: The Telegram Adapter MAY be replaced with any adapter
         that conforms to the Ingress / Egress interface contracts.
         SDK Core and all Logic Modules are unaffected by adapter replacement.

Rule E4: The SDK is messaging-platform-independent at every layer
         except the Telegram Adapter.
         Supporting a second messaging platform requires only a new Adapter.
```

---

## 6.3 THE UNIVERSAL EXECUTION PRINCIPLE

```
Event
  ↓
Context
  ↓
Decision (Logic Module or Workflow Step)
  ↓
Action[]
  ↓
Adapter
  ↓
Result
  ↓
State + Telemetry + Audit
```

This is the invariant execution sequence for every SDK operation.

---

*End of Document — SDK-SPEC-001*
