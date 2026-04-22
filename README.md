# @mhingston5/agent-workflows

Define agent workflows as data, not code. Write a YAML DAG, validate it, and execute it against any runner — no harness lock-in.

## Why

Hardcoded pipeline sequences break when you add branches, gates, or conditional skips. A declarative DAG lets you change pipeline topology by editing YAML, not refactoring dispatch code.

## Install

```bash
npm install @mhingston5/agent-workflows
```

Requires a peer dependency on Zod v3 or v4.

## 30-second example

```yaml
# workflow.yaml
name: build-and-deploy
on:
  invoke:
    args:
      - name: branch
        type: string
        required: true

nodes:
  - id: test
    deterministic: true
    exec:
      command: "npm test"
    max_retries: 1

  - id: build
    deterministic: true
    depends_on: [test]
    exec:
      command: "npm run build"

  - id: deploy
    deterministic: false
    depends_on: [build]
    when: $args.branch == 'main'
    exec:
      command: "npm run deploy"
```

```typescript
import { WorkflowEngine, ShellRunner, parseYaml } from "@mhingston5/agent-workflows";
import { readFileSync } from "fs";

const yaml = readFileSync("workflow.yaml", "utf-8");
const { workflow } = parseYaml(yaml);
if (!workflow) process.exit(1);

const engine = new WorkflowEngine(new ShellRunner());
const result = await engine.execute(workflow, { branch: "main" });

console.log(result.success ? "Deployed" : `Failed: ${result.error}`);
```

## Workflow YAML reference

### Top-level

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Workflow identifier |
| `require_deterministic` | boolean | no | Skip non-deterministic nodes when true |
| `on.invoke.args` | Arg[] | no | Input arguments for the workflow |
| `env.variables` | EnvVar[] | no | Workflow-level environment variables |
| `env.secrets` | string[] | no | Secret names available to all nodes |

### Node types

Each node has exactly **one** of these types:

| Type | Field | Description |
|------|-------|-------------|
| Shell | `exec.command` | Runs a command via the runner |
| Skill | `invoke_skill.name` | Dispatches to an agent skill via the runner |
| Gate | `gate.type: approval` | Pauses for human approval |
| Sub-workflow | `uses.workflow` | Loads and runs another workflow file |

### Node fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `id` | string | required | Unique node identifier |
| `deterministic` | boolean | false | Produces reproducible output |
| `depends_on` | string[] | [] | Upstream node IDs |
| `trigger_rule` | enum | all_succeeded | When this node can run |
| `when` | string | — | Conditional expression; node is skipped if falsy |
| `on_failure` | string | — | Node ID to route to on failure |
| `max_retries` | integer | 0 | Retry count (0–3) |
| `timeout_seconds` | integer | — | Per-node timeout |
| `output.bind` | string | — | State key to bind node result to |
| `artifacts` | Artifact[] | — | Named artifacts produced by this node |
| `env` | EnvVar[] | — | Node-level environment variables |
| `secrets` | string[] | — | Node-level secret names |

### Trigger rules

- **`all_succeeded`** (default) — all dependencies must succeed
- **`any_completed`** — at least one dependency finished (succeeded, failed, or skipped)
- **`always`** — runs regardless of dependency outcomes

### When conditions

Use `when` to conditionally skip nodes. Supports:

| Operator | Example |
|----------|---------|
| `==` | `$args.track == 'express'` |
| `!=` | `$state.redirect != true` |
| `>` / `>=` / `<` / `<=` | `$state.score >= 7` |
| `&&` | `$a && $b` |
| `\|\|` | `$a \|\| $b` |

Variable references in conditions and `invoke_skill.with`:

| Syntax | Reads from |
|--------|-----------|
| `$args.FIELD` | Workflow input arguments |
| `$state.FIELD` | Mutable workflow state |
| `$node_id.output` | Output of an upstream node |
| `$node_id.artifact.NAME` | Artifact from an upstream node |

## Runner contract

Implement `AgentWorkflowRunner` to connect the engine to your platform:

```typescript
import type { AgentWorkflowRunner, WorkflowNode, ExecutionContext, NodeResult } from "@mhingston5/agent-workflows";

class MyRunner implements AgentWorkflowRunner {
  async execute(node: WorkflowNode, ctx: ExecutionContext): Promise<NodeResult> {
    if (node.invoke_skill) {
      // dispatch to your skill system
      return { success: true, output: { ... } };
    }
    if (node.exec) {
      // run a shell command
      return { success: true, output: "done" };
    }
    if (node.gate) {
      // wait for human approval
      const approved = await askHuman(node.id);
      return { success: approved, output: approved ? "approved" : "rejected" };
    }
    return { success: false, error: `Unknown node type` };
  }
}
```

### ExecutionContext

The runner receives a context with these methods:

| Method / field | Description |
|----------------|-------------|
| `ctx.args` | Workflow input arguments |
| `ctx.env` | Merged environment variables |
| `ctx.secrets` | Available secret names |
| `ctx.readState(key)` | Read a value from workflow state |
| `ctx.writeState(key, value)` | Write a value into workflow state |
| `ctx.log(event)` | Emit a structured event |

`writeState` is useful when a runner needs to flatten complex outputs into state keys that downstream `when` conditions can reference.

### NodeResult

```typescript
interface NodeResult {
  success: boolean;
  output?: unknown;      // Node output (stored in state if output.bind is set)
  artifacts?: Artifact[]; // Named artifacts
  error?: string;       // Error message on failure
}
```

## Engine options

The `WorkflowEngine` constructor accepts an optional third argument:

```typescript
const engine = new WorkflowEngine(runner, subWorkflowLoader, {
  onEvent: (event) => {
    // Receive every engine event (node_start, node_end, node_skip, etc.)
    eventLog.append(event);
  },
});
```

### Event types

| Type | When |
|------|------|
| `node_start` | Node begins execution (includes attempt number) |
| `node_end` | Node completes successfully |
| `node_skip` | Node is skipped (includes reason) |
| `node_retry` | Node fails and will be retried |
| `node_failure` | Node fails permanently |
| `workflow_complete` | All nodes finished successfully |
| `workflow_failure` | A node failed and the workflow stops |

## Built-in runners

| Runner | Purpose |
|--------|---------|
| `InMemoryRunner` | Test runner with per-node handler registration and mock fallbacks |
| `ShellRunner` | Runs `exec.command` via `child_process.execSync` |

## Validation

### CLI

```bash
npx workflow-validate workflow.yaml
```

### Programmatic

```typescript
import { parseYaml } from "@mhingston5/agent-workflows";

const result = parseYaml(yamlString);
if (!result.success) {
  for (const error of result.errors) {
    console.error(`${error.type}: ${error.message}`);
  }
}
```

### Validation checks

- **Structural** (Zod): field types, required properties, enum values
- **Semantic** (DAG): cycle detection, unknown dependencies, unresolved output/artifact references, one node type per node

## Common patterns

### Conditional branch (investigation redirect)

```yaml
nodes:
  - id: intake
    invoke_skill: { name: intake }
    output: { bind: intake_result }

  - id: investigate
    depends_on: [intake]
    when: $state.redirect_to_investigate == true
    invoke_skill: { name: investigate }

  - id: checkout
    depends_on: [intake]
    when: $state.redirect_to_investigate != true
    invoke_skill: { name: checkout }
```

The runner uses `ctx.writeState("redirect_to_investigate", true)` from the intake handler to set the flat state key the `when` conditions read.

### Approval gate

```yaml
nodes:
  - id: gate-review
    when: $state.requires_review == true
    gate: { type: approval, on_timeout: abort }

  - id: deploy
    depends_on: [build, gate-review]
    trigger_rule: any_completed
    exec: { command: "npm run deploy" }
```

The gate is skipped when `requires_review` is falsy. `trigger_rule: any_completed` lets `deploy` proceed regardless.

### Express track (skip heavy nodes)

```yaml
nodes:
  - id: design
    when: $args.pipeline_track != 'express'
    invoke_skill: { name: brainstorming }

  - id: implement
    depends_on: [design, checkout]
    trigger_rule: any_completed
    invoke_skill: { name: coordinator }
```

## API

| Export | Description |
|--------|-------------|
| `WorkflowEngine` | DAG execution engine |
| `parseYaml(yaml)` | Parse and validate a YAML workflow |
| `parseJson(json)` | Parse and validate a JSON workflow |
| `parseWorkflow(obj)` | Validate an already-parsed object |
| `InMemoryRunner` | Test runner with mock fallbacks |
| `ShellRunner` | Shell command runner |

## License

MIT