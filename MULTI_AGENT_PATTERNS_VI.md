# Phase 2: Kiến Trúc Diagrams & Implementation Patterns

**Ngày**: 2026-04-03  
**Ngôn Ngữ**: Tiếng Việt  
**Phạm Vi**: Multi-Agent Architecture Analysis  

---

## 2.1: ARCHITECTURE DIAGRAMS

### 1. Multi-Agent Workflow Diagram

```
┌──────────────────────────────────────────────────────────┐
│                      USER INPUT                          │
│                   (e.g., "refactor code")               │
└──────────────────┬───────────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │   COORDINATOR MODE   │
        │ (Orchestrator/điều   │
        │  phối viên)          │
        │                      │
        │ • Parse request      │
        │ • Route to agents    │
        │ • Manage lifecycle   │
        └──────────┬───────────┘
                   │
        ┌──────────┴──────────┬───────────────┬──────────────┐
        │                     │               │              │
        ▼                     ▼               ▼              ▼
    ┌────────────┐   ┌──────────────┐  ┌───────────┐  ┌──────────┐
    │GeneralPurp │   │ExploreAgent  │  │PlanAgent  │  │VerifyAgn │
    │oseAgent    │   │(codebase)    │  │(decompose)│  │(validate)│
    └────┬───────┘   └──────┬───────┘  └─────┬─────┘  └────┬─────┘
         │                  │                 │             │
         ▼                  ▼                 ▼             ▼
    ┌──────────────────────────────────────────────────────┐
    │            TOOLS EXECUTION LAYER                     │
    │  ┌──────────┐ ┌──────────┐ ┌──────────┐            │
    │  │  Shell   │ │   File   │ │  Search  │            │
    │  │  Tools   │ │  Tools   │ │  Tools   │            │
    │  └──────────┘ └──────────┘ └──────────┘            │
    │  ┌──────────┐ ┌──────────┐ ┌──────────┐            │
    │  │   Web    │ │   Config │ │   Skills │            │
    │  │  Tools   │ │  Tools   │ │  Tools   │            │
    │  └──────────┘ └──────────┘ └──────────┘            │
    │                                                      │
    │  [Hook Layer: PreToolUse + PostToolUse]            │
    └──────────────┬───────────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │  RESULT AGGREGATION  │
        │                      │
        │ • Combine outputs    │
        │ • Validate results   │
        │ • Format response    │
        └──────────┬───────────┘
                   │
                   ▼
        ┌──────────────────────┐
        │   RETURN TO USER     │
        │  (Final response)    │
        └──────────────────────┘
```

---

### 2. Coordinator Orchestration Flow

```
REQUEST LIFECYCLE:

1. REQUEST ENTRY
   Input → Coordinator.parse()
   ├─ Analyze intent
   ├─ Identify required agents
   └─ Prepare agent parameters

2. AGENT SPAWNING
   Coordinator → spawnMultiAgent()
   ├─ Fork process for each agent
   ├─ Initialize agent memory
   ├─ Set tool access permissions
   └─ Establish communication channel

3. AGENT EXECUTION (Parallel/Sequential)
   Agent loop:
   ├─ Agent.initialize() → Load SKILL.md context
   ├─ Agent.executeTools() → Run tools with hooks
   ├─ Agent.saveState() → Update memory
   └─ Agent.reportResult() → Send to coordinator

4. RESULT AGGREGATION
   Coordinator.aggregate()
   ├─ Collect results from all agents
   ├─ Merge insights
   ├─ Resolve conflicts
   └─ Format final output

5. SESSION PERSISTENCE
   SessionMemory.save()
   ├─ Save to ~/.claw/sessions/
   ├─ Update agent memory
   └─ Store execution trace
```

---

### 3. Agent Spawning Sequence

```
PARENT AGENT                          COORDINATOR                      SUB-AGENT
     │                                     │                                │
     │──(Request spawn)─────────────────→ │                                │
     │                                     │                                │
     │                                     │─(fork process)──────────────→  │
     │                                     │                        [Initialize]
     │                                     │                                │
     │                                     │◄─(ready signal)────────────────│
     │                                     │                                │
     │                                     │─(pass memory snapshot)────────→│
     │                                     │                        [Load State]
     │                                     │                                │
     │◄─(confirm spawned)─────────────────│                                │
     │                                     │                                │
     │                                     │                    ┌─────────→ Execute Tools
     │                                     │                    │           [Tool Loop]
     │                                     │                    │                │
     │                                     │◄─(result report)──┴────────────│
     │                                     │                                │
     │◄─(aggregate result)────────────────│                                │
     │                                     │                                │
     [Continue work]                       [Update state]             [Exit/Resume]
```

---

### 4. Hook Execution Pipeline

```
TOOL CALL ENTRY:
     │
     ▼
┌─────────────────────────────────────┐
│    PERMISSION CHECK                 │
│  (Check ResolvedPermissionMode)     │
│                                     │
│  ReadOnly → Deny file writes        │
│  WorkspaceWrite → Allow workspace   │
│  DangerFullAccess → Allow all       │
└──────────────┬──────────────────────┘
               │
         ┌─────▼─────┐
         │   ALLOW?  │
         └─┬───────┬─┘
        NO │       │ YES
           │       ▼
           │   ┌──────────────────────────┐
           │   │   PRE-TOOL-USE HOOK      │
           │   │  (Validation layer)      │
           │   │                          │
           │   │ • Input validation       │
           │   │ • Input mutation/clean   │
           │   │ • Deny tool if needed    │
           │   │ • Append pre-messages    │
           │   └──────────┬───────────────┘
           │              │
           │         ┌────▼────┐
           │         │Denied?  │
           │         └─┬──────┬─┘
           │        YES│      │NO
           │           │      ▼
           │           │   ┌────────────────┐
           │           │   │ TOOL EXECUTE   │
           │           │   │  (Run actual   │
           │           │   │   command)     │
           │           │   └────────┬───────┘
           │           │            │
           │           │            ▼
           │           │   ┌──────────────────────────┐
           │           │   │  POST-TOOL-USE HOOK      │
           │           │   │  (Result processing)     │
           │           │   │                          │
           │           │   │ • Result validation      │
           │           │   │ • Result mutation        │
           │           │   │ • Append feedback        │
           │           │   │ • Can deny result        │
           │           │   └──────────┬───────────────┘
           │           │              │
           │           ▼              ▼
           └───────→┌────────────────────────────┐
                    │  FORMAT TOOL RESULT        │
                    │                            │
                    │ • Mark as error if denied  │
                    │ • Attach hook messages     │
                    │ • Return to agent         │
                    └────────────┬───────────────┘
                                 │
                                 ▼
                          [Continue Loop]
```

---

### 5. Service Dependency Graph

```
                    ┌─────────────────┐
                    │  USER/SESSION   │
                    └────────┬────────┘
                             │
                             ▼
                    ┌──────────────────────┐
                    │   COORDINATOR MODE   │
                    │  (Orchestrator)      │
                    └────────┬─────────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
                ▼            ▼            ▼
        ┌────────────┐  ┌─────────────┐  ┌──────────┐
        │   AGENTS   │  │   TASKS     │  │  TEAMS   │
        │  (6 types) │  │  (Queue)    │  │(Swarms)  │
        └────┬───────┘  └─────┬───────┘  └────┬─────┘
             │                │              │
             └────────┬───────┴──────────────┘
                      │
                      ▼
        ┌─────────────────────────────┐
        │    TOOLS EXECUTION          │
        │  (150+ tool implementations)│
        │                             │
        │  ┌────────────────────┐    │
        │  │ Hook Middleware    │    │
        │  │ PreToolUse/Post    │    │
        │  └────────────────────┘    │
        └─────────────┬───────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
    ┌────────┐  ┌──────────┐  ┌───────────┐
    │SESSION │  │  AGENT   │  │ ANALYTICS │
    │MEMORY  │  │ SUMMARY  │  │SERVICES   │
    │SERVICE │  │ SERVICE  │  │(Datadog)  │
    └────────┘  └──────────┘  └───────────┘
```

---

## 2.2: IMPLEMENTATION PATTERNS

### Pattern 1: Coordinator Pattern (Orchestrator)

**Mục đích**: Quản lý toàn bộ agent lifecycle, resource, và task scheduling

**Cấu trúc**:
```rust
// Conceptual structure
struct Coordinator {
    agents: Vec<AgentHandle>,
    tasks: TaskQueue,
    teams: Vec<TeamHandle>,
    state: CoordinatorState,
    hooks: HookRegistry,
}

impl Coordinator {
    fn handle_request(&mut self, request: Request) -> Response {
        // 1. Parse and route
        let agent_type = self.route(request);
        
        // 2. Spawn agents
        let agents = self.spawn_agents(agent_type);
        
        // 3. Execute and monitor
        let results = agents.iter_mut()
            .map(|a| a.execute_with_hooks())
            .collect();
        
        // 4. Aggregate results
        self.aggregate_results(results)
    }
}
```

**Ứng Dụng**: Routing requests, spawning, lifecycle management

---

### Pattern 2: Agent Execution Pattern

**Mục đích**: Chạy agent tính năng đầy đủ với tool loop

**Cấu Trúc**:
```rust
// Conceptual
struct Agent {
    name: String,
    memory: AgentMemory,
    tools: ToolRegistry,
    hooks: HookRegistry,
}

impl Agent {
    fn execute(&mut self, task: Task) -> AgentResult {
        // 1. Initialize memory
        self.memory.initialize(task.context);
        
        // 2. Tool execution loop
        loop {
            // Get next action
            let action = self.decide_next_action();
            
            if action == "use_tool" {
                // 3. Run hooks + tool
                let result = self.execute_tool_with_hooks();
                self.memory.record(result);
            }
            
            if self.should_finish() { break; }
        }
        
        // 4. Save state
        self.memory.persist();
        
        AgentResult { ... }
    }
    
    fn execute_tool_with_hooks(&mut self) -> ToolResult {
        // Pre-hook
        if self.hooks.pre_tool_use.deny() { return Err(...) }
        
        // Execute
        let output = self.tools.execute();
        
        // Post-hook
        if self.hooks.post_tool_use.deny() { mark_error(); }
        
        output
    }
}
```

**Ứng Dụng**: Core agent reasoning, tool calling, memory management

---

### Pattern 3: Hook Middleware Pattern

**Mục đích**: Intercept tool execution để thêm validation, permission, mutation

**Cấu Trúc**:
```rust
// Conceptual
struct HookPipeline {
    pre_hooks: Vec<HookHandler>,
    post_hooks: Vec<HookHandler>,
}

impl HookPipeline {
    fn execute_with_hooks(
        &self,
        tool_name: &str,
        input: &str,
    ) -> Result<String, HookError> {
        // 1. Pre-tool-use hooks (sequential)
        for hook in &self.pre_hooks {
            let outcome = hook.run(tool_name, input)?;
            match outcome {
                HookOutcome::Deny => return Err("Denied by hook"),
                HookOutcome::MutateInput(new) => input = new,
                HookOutcome::Allow => continue,
            }
        }
        
        // 2. Execute tool
        let output = Tool::execute(tool_name, input)?;
        
        // 3. Post-tool-use hooks (sequential)
        for hook in &self.post_hooks {
            let outcome = hook.run(tool_name, &output)?;
            match outcome {
                HookOutcome::DenyResult => return Err("Result denied"),
                HookOutcome::MutateOutput(new) => output = new,
                HookOutcome::Allow => continue,
            }
        }
        
        Ok(output)
    }
}
```

**Ứng Dụng**: Permission enforcement, input/output validation, audit logging

---

### Pattern 4: Task Queue Pattern

**Mục đích**: Background task execution, lifecycle management

**Cấu Trúc**:
```rust
// Conceptual
struct TaskQueue {
    tasks: HashMap<TaskId, TaskState>,
    pending: VecDeque<TaskId>,
}

impl TaskQueue {
    fn create_task(&mut self, desc: String) -> TaskId {
        let task = Task {
            id: new_id(),
            description: desc,
            state: TaskState::Pending,
            result: None,
        };
        self.tasks.insert(task.id, task);
        task.id
    }
    
    fn execute_all(&mut self) {
        while let Some(task_id) = self.pending.pop_front() {
            if let Some(task) = self.tasks.get_mut(&task_id) {
                task.state = TaskState::Running;
                let result = self.run_task(task);
                task.result = Some(result);
                task.state = TaskState::Complete;
            }
        }
    }
    
    fn get_task(&self, id: TaskId) -> Option<&Task> {
        self.tasks.get(&id)
    }
}
```

**Ứng Dụng**: Async task execution, background jobs, result retrieval

---

### Pattern 5: Team Coordination Pattern

**Mục đích**: Manage multiple agents working together

**Cấu Trúc**:
```rust
// Conceptual
struct Team {
    id: TeamId,
    agents: Vec<AgentHandle>,
    shared_memory: SharedMemory,
    state: TeamState,
}

impl Team {
    fn create(agent_specs: Vec<AgentSpec>) -> Self {
        let agents = agent_specs
            .into_iter()
            .map(|spec| Agent::spawn(spec))
            .collect();
        
        Team {
            id: new_id(),
            agents,
            shared_memory: SharedMemory::new(),
            state: TeamState::Initialized,
        }
    }
    
    fn execute_parallel(&mut self) -> Vec<AgentResult> {
        self.agents
            .iter_mut()
            .map(|agent| {
                agent.set_shared_memory(self.shared_memory.clone());
                agent.execute()
            })
            .collect()
    }
    
    fn aggregate_results(&self, results: Vec<AgentResult>) -> TeamResult {
        // Merge insights, resolve conflicts
        TeamResult { ... }
    }
}
```

**Ứng Dụng**: Multi-agent coordination, shared context, team workflows

---

## 2.3: TYPESCRIPT vs RUST COMPARISON TABLE

| Tính Năng | TypeScript | Rust | Status | Độ Khó Porting |
|-----------|-----------|------|--------|-----------------|
| **Coordinator Mode** | ✓ Full | ✗ Missing | High Priority | Very High |
| **Built-in Agents (6)** | ✓ All | ✗ None | High Priority | Very High |
| **Agent Spawning** | ✓ spawnMultiAgent | ✗ Missing | High Priority | High |
| **Task Queue (6 tools)** | ✓ Full | ✗ Missing | High Priority | High |
| **Team Tools (2 tools)** | ✓ Full | ✗ Missing | High Priority | High |
| **Hook System (104 modules)** | ✓ Full execution | ⚠ Config only | Medium Priority | Very High |
| **Tool Execution** | ✓ Full (150+ tools) | ✓ MVP (26 tools) | Core OK | Low |
| **Sessions** | ✓ Full | ✓ Implemented | Core OK | Complete |
| **Permissions** | ✓ 3-tier | ✓ 3-tier | Core OK | Complete |
| **Services (130 modules)** | ✓ Full | ✓ Core services | Partial | Medium |
| **Analytics** | ✓ Datadog/GrowthBook | ✗ Missing | Low Priority | High |
| **Plugin System** | ✓ Full | ✗ Skeleton | Medium Priority | Very High |
| **Skills (20 bundled)** | ✓ Full registry | ✓ Local file | Partial | Medium |
| **CLI Commands** | ✓ 50+ commands | ✓ 15 commands | Partial | High |
| **Cron/Scheduling** | ✓ 3 tools | ✗ Missing | Low Priority | Medium |

**Độ phủ Rust**: ~18-21% so với TypeScript (350-400 files vs 1,902 files)

---

## 2.4: CODE EXAMPLES (Conceptual Pseudocode)

### Example 1: Coordinator Spawning Agents

```python
# Conceptual Python/Rust equivalent

class Coordinator:
    def handle_request(self, user_input: str):
        # 1. Route request to appropriate agents
        agent_types = self.route_request(user_input)
        # e.g., ["explore", "plan", "verify"]
        
        # 2. Spawn agents in parallel
        agents = []
        for agent_type in agent_types:
            agent = self.spawn_agent(agent_type)
            # - Initialize memory
            # - Set permissions
            # - Load CLAW.md context
            agents.append(agent)
        
        # 3. Execute with monitoring
        results = []
        for agent in agents:
            result = agent.execute_with_timeout()
            results.append(result)
        
        # 4. Aggregate and return
        final = self.aggregate_results(results)
        
        # 5. Save session
        self.session_memory.save(final)
        
        return final
```

---

### Example 2: Agent Execution with Hooks

```python
class Agent:
    def execute_tool(self, tool_name: str, input_str: str):
        # 1. Check permissions
        if not self.permissions.allow(tool_name):
            return Error("Permission denied")
        
        # 2. Run PreToolUse hooks
        for hook in self.hooks.pre_tool_use:
            outcome = hook.execute(tool_name, input_str)
            if outcome.denied:
                return Error("Hook denied execution")
            if outcome.mutated_input:
                input_str = outcome.new_input
        
        # 3. Execute tool
        try:
            output = Tool.execute(tool_name, input_str)
        except ToolError as e:
            output = str(e)
            is_error = True
        
        # 4. Run PostToolUse hooks
        for hook in self.hooks.post_tool_use:
            outcome = hook.execute(tool_name, output, is_error)
            if outcome.denied:
                is_error = True
            if outcome.mutated_output:
                output = outcome.new_output
        
        # 5. Record in memory and return
        self.memory.record(tool_name, output)
        return output
```

---

### Example 3: Task Queue Usage

```python
class TaskQueue:
    def example_workflow(self):
        # Create background tasks
        task1 = self.create_task("Analyze codebase structure")
        task2 = self.create_task("Generate test cases")
        task3 = self.create_task("Create documentation")
        
        # Execute tasks
        self.execute_all()
        
        # Retrieve results
        result1 = self.get_task(task1).result
        result2 = self.get_task(task2).result
        result3 = self.get_task(task3).result
        
        # Aggregate
        final = self.aggregate([result1, result2, result3])
        return final
```

---

### Example 4: Team Creation & Coordination

```python
class TeamCoordinator:
    def create_review_team(self):
        # Create team of agents
        team = Team.create([
            AgentSpec(type="explore", task="understand structure"),
            AgentSpec(type="plan", task="identify issues"),
            AgentSpec(type="verify", task="validate fixes"),
        ])
        
        # Share context
        team.shared_memory.set("codebase_context", context)
        
        # Execute in parallel
        results = team.execute_parallel()
        
        # Aggregate insights
        final = team.aggregate_results(results)
        
        return final
```

---

### Example 5: Hook Middleware Implementation

```python
class HookMiddleware:
    def process_tool_call(self, tool_name, input_data):
        # 1. Pre-processing hooks
        for pre_hook in self.pre_hooks:
            if pre_hook.should_deny(tool_name, input_data):
                return Error("Pre-hook denial")
            
            input_data = pre_hook.transform(input_data)
        
        # 2. Tool execution
        try:
            output = execute_tool(tool_name, input_data)
            had_error = False
        except Exception as e:
            output = str(e)
            had_error = True
        
        # 3. Post-processing hooks
        for post_hook in self.post_hooks:
            if post_hook.should_deny_result(output, had_error):
                had_error = True
            
            output = post_hook.transform(output)
            
            # Can add messages
            if post_hook.has_feedback():
                output += post_hook.get_feedback()
        
        return output
```

---

## Summary: Key Takeaways

1. **Coordinator Pattern** — Central orchestrator manages everything
2. **Agent Spawning** — Dynamic multi-agent creation with fork model
3. **Hook Middleware** — Flexible extensibility without modifying core
4. **Task Queue** — Background execution and result aggregation
5. **Team Coordination** — Parallel agent execution with shared memory
6. **Service Integration** — Multi-service architecture for state/analytics

**Rust Port Status**: Core loop works (18-21% complete), multi-agent orchestration pending

