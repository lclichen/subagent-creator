---
name: subagent-creator
description: "Design and maintain reusable subagents. Use for agent configuration, role definition, capability assignment, tool integration, behavior specification, validation, multi-agent coordination, or agent lifecycle management. Not for one-off agent prompts or passive documentation."
---


# Subagent Creator Skill

This skill helps you create, manage, and work with autonomous sub-agents in pi.

## Overview

Sub-agents are Claude Code-style autonomous workers that run in isolated sessions, each with its own tools, system prompt, model, and thinking level. You can spawn them in foreground (blocking) or background (non-blocking), steer them mid-run, and resume completed sessions.

## When to Use This Skill

- You need to create custom agent types for specialized tasks
- You want to spawn sub-agents for parallel work
- You need to manage running agents or review their results
- You're setting up agent workflows for complex multi-step tasks

## Creating Custom Agents

### Basic Agent Definition

Create agent definition files in one of these locations (priority order):

1. `.pi/agents/<name>.md` - Project-level (highest priority)
2. `.agents/agents/<name>.md` - Project-level alternative
3. `~/.pi/agent/agents/<name>.md` - Global (available everywhere)

### Agent Frontmatter Fields

```yaml
---
description: Short description of the agent's purpose
tools: read, grep, find, bash  # Which tools the agent can use
model: anthropic/claude-haiku-4-5  # Model to use (or fuzzy name like "haiku")
thinking: medium  # off, minimal, low, medium, high, xhigh, max
max_turns: 30  # Maximum agentic turns before shutdown
skills: skill-name1, skill-name2  # Skills to preload
memory: project  # project, local, or user
disallowed_tools: edit, write  # Tools to deny
isolation: worktree  # Run in isolated git worktree
persist_session: true  # Persist as a normal pi session
output_transcript: true  # Write transcript file
prompt_mode: replace  # replace (standalone) or append (parent twin)
inherit_context: false  # Fork parent conversation
run_in_background: false  # Run in background by default
enabled: true  # Set false to disable
---
```

### Example: Security Auditor Agent

```markdown
---
description: Security Code Reviewer
tools: read, grep, find, bash
model: anthropic/claude-opus-4-6
thinking: high
max_turns: 30
---

You are a security auditor. Review code for vulnerabilities including:
- Injection flaws (SQL, command, XSS)
- Authentication and authorization issues
- Sensitive data exposure
- Insecure configurations

Report findings with file paths, line numbers, severity, and remediation advice.
```

## Spawning Sub-Agents

### Using the Agent Tool

```typescript
Agent({
  subagent_type: "Explore",  // Built-in or custom agent type
  prompt: "Find all files that handle authentication",
  description: "Find auth files",  // 3-5 word summary
  run_in_background: true,  // Optional: non-blocking
  model: "haiku",  // Optional: override agent's model
  thinking: "low",  // Optional: override thinking level
  max_turns: 20,  // Optional: override turn limit
  inherit_context: false,  // Optional: fork parent conversation
  isolation: "worktree",  // Optional: isolated git worktree
})
```

### Agent Types

| Type | Tools | Model | Description |
|------|-------|-------|-------------|
| `general-purpose` | all 7 | inherit | Parent twin with full system prompt |
| `Explore` | read, bash, grep, find, ls | haiku | Fast codebase exploration (read-only) |
| `Plan` | read, bash, grep, find, ls | inherit | Software architect for planning (read-only) |

### Background vs Foreground

- **Foreground** (`run_in_background: false`, default): Blocks until complete, returns results inline
- **Background** (`run_in_background: true`): Returns agent ID immediately, notifies on completion

### Managing Background Agents

```typescript
// Check status
get_subagent_result({
  agent_id: "agent-abc123",
  wait: false,  // Optional: wait for completion
  verbose: false,  // Optional: include full conversation
})

// Send steering message to running agent
steer_subagent({
  agent_id: "agent-abc123",
  message: "Focus on the authentication module specifically",
})
```

## Scheduling Agents

Run agents on cron, interval, or one-shot schedule:

```typescript
Agent({
  subagent_type: "Explore",
  prompt: "Review recent changes",
  description: "Weekly review",
  schedule: "0 0 9 * * 1",  // 9am every Monday (6-field cron)
})
```

Schedule formats:
- **Cron**: `"0 0 9 * * 1"` (second minute hour day month day-of-week)
- **Interval**: `"5m"`, `"1h"`, `"30s"`, `"2d"` (repeating)
- **One-shot relative**: `"+10m"`, `"+2h"`, `"+1d"` (fires once)
- **One-shot absolute**: `"2026-12-25T09:00:00.000Z"` (ISO timestamp)

**Restrictions**: Cannot combine `schedule` with `inherit_context` or `resume`. `run_in_background` is forced to `true`.

## Tool & Extension Scoping

### Tools Field

```yaml
tools: read, grep, find  # Specific built-in tools
tools: "*"              # All 7 built-ins (alias: "all")
tools: none             # No built-ins (alias: "")
tools: "*, ext:pi-mcp-adapter/search"  # Built-ins plus specific extension tool
```
补充知识：限制subagent可见的mcp工具需要使用约束`tools: "ext:pi-mcp-adapter/[mcpServerName]_[toolName]"`

### Extensions Field

```yaml
extensions: true        # All default extensions (default)
extensions: false       # No extensions
extensions: [mcp]       # Only specific extensions
extensions: ["*", "/abs/foo.ts"]  # All defaults plus custom path
exclude_extensions: pi-notify  # Everything except listed
```

### Specialist Mode

```yaml
isolated: true  # Hermetic: built-ins only, no extensions/skills/context
```

## Settings Management

Configure via `/agents` → Settings:

- **Max concurrency**: Default 4 (background agents)
- **Default max turns**: Unlimited by default
- **Grace turns**: 5 (graceful shutdown window)
- **Join mode**: `smart` (group 2+ agents), `async`, or `group`
- **Widget mode**: `all`, `background` (default), or `off`
- **Fleet view**: Enable/disable navigable agent list
- **Model scope**: Enforce enabledModels allowlist (opt-in)
- **Output transcript**: Write agent transcripts (default: true)
- **Disable defaults**: Hide built-in agents (default: false)

## Persistent Memory

Enable agent memory across sessions:

```yaml
---
memory: project  # project | local | user
---
```

| Scope | Location | Use case |
|-------|----------|----------|
| `project` | `.pi/agent-memory/<name>/` | Shared across team (committed) |
| `local` | `.pi/agent-memory-local/<name>/` | Machine-specific (gitignored) |
| `user` | `~/.pi/agent/agent-memory/<name>/` | Global personal memory |

Read-only agents (no write/edit tools) automatically get read-only memory.

## Worktree Isolation

Run agents in isolated git worktrees:

```typescript
Agent({
  subagent_type: "refactor",
  prompt: "Refactor the auth module",
  isolation: "worktree",
})
```

On completion:
- **No changes**: Worktree cleaned up automatically
- **Changes made**: Committed to `pi-agent-<id>` branch
- **Agent committed work**: Branch created at agent's HEAD

## Events

Listen for lifecycle events via `pi.events`:

| Event | When |
|-------|------|
| `subagents:created` | Background agent registered |
| `subagents:started` | Agent transitions to running |
| `subagents:completed` | Agent finished successfully |
| `subagents:failed` | Agent errored, stopped, or aborted |
| `subagents:steered` | Steering message sent |
| `subagents:compacted` | Session successfully compacted |
| `subagents:ready` | RPC handlers registered |

## Cross-Extension RPC

Spawn sub-agents from other extensions:

```typescript
// Discovery
pi.events.on("subagents:ready", () => {
  // RPC handlers available
});

// Spawn
pi.events.emit("subagents:rpc:spawn", {
  requestId: crypto.randomUUID(),
  type: "general-purpose",
  prompt: "Do something useful",
  options: { description: "My task", run_in_background: true },
});

// Stop
pi.events.emit("subagents:rpc:stop", {
  requestId: crypto.randomUUID(),
  agentId: "agent-id-here",
});
```

## Best Practices

1. **Use background agents for parallel work**: Spawn multiple agents with `run_in_background: true`
2. **Set appropriate turn limits**: Use `max_turns` to prevent runaway agents
3. **Choose the right agent type**: Use `Explore` for read-only exploration, `Plan` for architecture
4. **Leverage scheduling**: Use cron/interval schedules for recurring tasks
5. **Enable worktree isolation**: Use `isolation: worktree` for experimental changes
6. **Preload skills**: Use `skills:` field to inject domain knowledge
7. **Monitor via FleetView**: Use `/agents` to view and manage running agents
8. **Use steering proactively**: Redirect agents mid-run instead of restarting

## Troubleshooting

### Agent Won't Start
- Check if agent type exists (case-insensitive)
- Verify model is available/configured
- Ensure `.pi` directory has proper permissions

### Model Resolution Fails
- Use exact `provider/modelId` format
- Try fuzzy names (`"haiku"`, `"sonnet"`)
- Check `enabledModels` in settings if scope enforcement is on

### Worktree Creation Fails
- Ensure directory is a git repo with at least one commit
- Check git worktree support (`git worktree --help`)

### Transcripts Not Written
- Check `output_transcript: true` in agent config
- Verify `subagents.json` `outputTranscript` setting
- Check filesystem permissions

## Quick Reference

### Create a Custom Agent
1. Create `.pi/agents/<name>.md`
2. Define frontmatter and system prompt
3. Spawn with `Agent({ subagent_type: "<name>", ... })`

### Manage Running Agents
- Open `/agents` menu
- View running agents list
- Select agent to open conversation viewer
- Press `Enter` to steer, `x` to stop

### Schedule Recurring Tasks
```typescript
Agent({
  subagent_type: "Explore",
  prompt: "Your task",
  description: "Task name",
  schedule: "0 */15 * * * *",  // Every 15 minutes
})
```

### Enable Persistent Memory
```yaml
---
memory: project
---
```

## Resources

- Main documentation: `reference/README.md`
- Example tool descriptions: `examples/agent-tool-description.md`

## 补充知识：

- 限制subagent可见的mcp工具需要使用约束`tools: "ext:pi-mcp-adapter/[mcpServerName]_[toolName]"`.
- 默认情况下不写model参数，继承主线模型.
