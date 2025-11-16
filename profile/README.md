# AWFL — Open Source Agents and Workflow Suite

Website: https://awfl.us

AWFL is a modular suite for building, running, and exploring agent-driven workflows. It combines:
- A typed Scala DSL for workflow description
- Agent composition via Scala traits (mixins)
- Utilities for LLM chat, context reads, and distributed locks
- A Python CLI for running agents locally with project-aware tool access
- A React web UI for browsing sessions, inspecting context (“Yoj”), and supervising workflows

Use as much or as little as you need: import the DSL in Scala, ship agents, run with the CLI, and visualize with the web UI.

## Projects (by repository)

- CLI (Python): https://github.com/awfl-us/cli
  - Install via pipx or pip; run agents from your terminal
- DSL (Scala 3): https://central.sonatype.com/artifact/us.awfl/dsl_3
  - Typed, composable workflow description (engine-agnostic)
- workflow-utils (Scala): https://github.com/awfl-us/workflow-utils
  - Small utilities for LLM chat, context reads, locks, and environment helpers
- Workflows (Scala): https://github.com/awfl-us/workflows
  - Reusable agent traits (mixins), example agents, and workflow building blocks
- Web UI (React): https://github.com/awfl-us/web
  - Sessions and Workflow UI for exploring runs, context, tasks, and agent links

## Quick start

### Run agents with the CLI (Python)
- Install
  ```
  pipx install awfl
  # or:
  pip install --user awfl
  ```
  Verify: awfl --version

- Launch in a project
  ```
  cd your/project
  awfl
  ```
  First run prompts Google device login; tokens are cached locally.

- Pick a workflow (agent)
  ```
  awfl
  workflows   # or: ls
  ```

- Chat and apply changes
  ```
  "Add a Demo section to README and a recording script"
  ```
  The agent plans and issues tool calls (read files, write files, run commands). One terminal holds a project lock to apply side effects; all terminals can stream events.

### Use the DSL in Scala
- Add dependency (sbt)
  ```
  libraryDependencies += "us.awfl" %% "dsl" % "<latest>"
  ```
- Describe typed workflows and render or execute with your engine of choice.

## Compose agents with Scala mixins (traits)

Build behavior by stacking small traits: Agents wire prompts, preloads, tasks, CLI tools, and routing.

Minimal agent
```scala
import us.awfl.workflows.traits.Agent
import us.awfl.workflows.traits.Preloads._

object HelloAgent extends Agent {
  override def preloads = List(
    PreloadFile("AGENT.md"),
    PreloadCommand("date -u")
  )

  override def prompt =
    """You are a helpful assistant for the Hello project.
       Respond succinctly and use the preloaded docs as context.""".stripMargin
}
```

Customize tools and prompts
```scala
import us.awfl.workflows.traits.Agent
import us.awfl.dsl.*
import us.awfl.dsl.auto.given

object ReadOnlyAgent extends Agent {
  override def prompt = "Read-only assistant"

  // Keep only READ_FILE; drop UPDATE_FILE and RUN_COMMAND
  override def buildTools = joinSteps(
    "readOnlyTools",
    super.buildTools,
    buildList("limitCliTools", List("READ_FILE"))
  )

  // Add extra guidance to the system prompt stack
  override def buildPrompts = joinSteps(
    "extraGuidance",
    super.buildPrompts,
    buildList("myGuidance", List(ChatMessage("system", str("Prefer small, incremental changes."))))
  )
}
```

Notes
- The composition stack layers traits such as Workflow, EventHandler, Preloads, Tasks, and Cli.
- EventHandler resolves tool names and dispatches via a small dispatcher.

## Typed DSL (Scala 3): building blocks

Describe workflows as typed, composable data — readable, testable, and engine-agnostic.

```scala
import us.awfl.dsl._
import us.awfl.dsl.CelOps._
import us.awfl.dsl.auto.given

case class User(name: Value[String])
val user   = init[User]("input")
val name   = user.flatMap(_.name)

// Build a tiny program
val greetings = ForRange[String]("greetings", from = 0, to = 3) { i =>
  val msg = str(("Hello, ": Cel) + name + " #" + i)
  List(Log("log_msg", msg)) -> msg
}.resultValue

val program = Block("example", List(Log("log_user", name)) -> greetings)
```

Why this approach
- Strong types: Value[T], ListValue[T], and CEL expressions
- Declarative steps: Log, Call, For, Try, Switch, Block
- Pure description: render to JSON/YAML, send to an engine, or interpret locally

## workflow-utils: clients and helpers

LLM chat
```scala
import us.awfl.services.Llm
import us.awfl.ista.ChatMessage

val msgs = buildList("msgs", List(ChatMessage("user", str("Hello!"))))
val chat = Llm.chat("demo_chat", msgs.resultValue)  // Step to llm/chat
```

Context reads (“Yoj”)
```scala
import us.awfl.services.Context
import us.awfl.utils.Env

val kala = Context.segKala(Env.sessionId, obj(1720000000.0), obj(3600.0))
val read  = Context.yojRead[ChatMessage]("read_notes", str("messages"), kala)
```

Distributed locks
```scala
import us.awfl.utils.Locks
import us.awfl.utils.{SegKala, KalaVibhaga}
import us.awfl.utils.Env

given KalaVibhaga = SegKala(Env.sessionId, obj(1720000000.0), obj(300.0))
val acquired = Locks.acquireBool("reports_lock", Locks.key("reports"), str("worker-1"), 60)
```

## Explore and supervise with the Web UI

Sessions and Workflow UI (React + Vite)
- Sessions-first UX
  - Browse and filter sessions by title, summary, tags, highlights
  - Auto-selects the first session; stable state during refresh
  - Detail view renders Yoj context/messages
- Live workflow control
  - Start/stop workflows and monitor latest status
  - Status preserved through transient API gaps to prevent flicker
- Tasks and agents
  - Per-session task counts and inline lists
  - Link sessions to agents via AgentModal
- Mobile and accessibility
  - Responsive panels, headers remain visible; mobile back button in detail pane

Development model
- Feature modules with minimal public surfaces
- Centralized API client and defensive mapping
- Abort-friendly hooks with enabled flags and proper cleanup

Repo: https://github.com/awfl-us/web

## Design principles

- Typed, composable, engine-agnostic workflows
- Small, layered traits for agent behavior
- Minimal, stable public surfaces per module
- Resilient data flows (abort on dependency changes, tolerate partial data)
- Small, safe, incremental changes

## Community and contributions

- Issues and discussions: open issues in the relevant repository (CLI, DSL, workflow-utils, workflows, or UI)
- Contributions welcome:
  - Keep modules cohesive and small
  - Prefer public.ts (or equivalent) surfaces over deep imports
  - Normalize backend shapes at boundaries
  - Add tests and docs for new traits, steps, or tools

## Links

- Website: https://awfl.us
- CLI: https://github.com/awfl-us/cli
- DSL (Maven Central): https://central.sonatype.com/artifact/us.awfl/dsl_3
- workflow-utils: https://github.com/awfl-us/workflow-utils
- Workflows: https://github.com/awfl-us/workflows
- Web UI: https://github.com/awfl-us/web

## License

MIT
