# Spiral Council Improvement Plan

This plan outlines concrete steps to refactor and extend Spiral Council using LangGraph patterns (durable execution, human-in-the-loop controls, layered memory, and observability).

## Guiding objectives
- **Reliability**: Use checkpointing and resumable runs to keep long-lived agents robust.
- **Traceability**: Capture decisions, state, and tool calls for review/debugging.
- **Extensibility**: Compose agents and tools as modular, observable graphs that are easy to evolve.
- **Human oversight**: Allow operators to pause, inspect, and intervene when the graph hits review nodes.

## Immediate next steps (1–2 days)
1. **Bootstrap a minimal agent graph**
   - Start from the `create_react_agent` quickstart and run it locally.
   - Replace the toy model/tooling with Spiral Council’s preferred LLM and domain tools.
   - Wrap the graph with `AsyncRunnableConfig` to pass metadata (tenant/user/request IDs) through the run.
   - Add a smoke test (e.g., `pytest` function) that executes a single turn and asserts a non-empty reply to keep the graph runnable in CI.
2. **Add durable checkpointing**
   - Use SQLite in development (`checkpoint-sqlite`) to persist state between runs and enable resumption.
   - Wire the checkpointer into the graph via `with_checkpointer`, ensuring tool/LLM calls record progress.
   - Add an interruption test: start a run, simulate a crash, resume, and verify the reply continues from the saved state.
3. **Enable review breaks**
   - Insert `Send` nodes or conditional branches that surface intermediate results for human approval before continuing.
   - Define a small policy for when to trigger review (e.g., high-cost tool calls or low-confidence LLM answers).
   - Capture review outcomes in the state (approved, edits requested, rejected) and ensure downstream nodes respect the decision.

## Near-term enhancements (1–2 weeks)
- **Observability via LangSmith**
  - Configure tracing to send runs, tool calls, and custom events to LangSmith for timeline playback and analytics.
  - Add run metadata (tenant, scenario, cost) to traces for easier filtering.
  - Set up one alert on high error rate or latency regression using LangSmith or your monitoring stack.
- **Layered memory**
  - Add short-term conversational state within the graph state.
  - Connect long-term memory to a vector store retriever for past decisions/artifacts.
  - Include a pruning strategy (e.g., retain last N turns, summarize older context) to keep checkpoints compact.
- **Tool hygiene**
  - Standardize tool inputs/outputs (Pydantic models) and include result summaries in graph state to keep checkpoints compact.
  - Enforce timeouts/retries on external calls and log failures to the trace.
- **Safety hooks**
  - Insert guardrails (content filters, allowed tool lists) and log violations to LangSmith for auditing.
  - Add a policy test that feeds disallowed input and verifies the run is blocked with an explainable message.

## Integration milestones
1. **MVP demo**: Single-agent graph with SQLite checkpointing, one review break, LangSmith tracing, and passing smoke/interruption tests.
2. **Multi-actor workflow**: Additional nodes for specialized roles (research, execution, validation) connected through a shared state schema; include a review break between roles.
3. **Production hardening**: Postgres checkpointing, retries/backoff around external tools, SLOs with alerts on failed/slow runs, and dashboards for latency/cost.

## Validation checklist
- Graph resumes correctly after a forced interruption (kill process) and continues from the latest checkpoint.
- Review breaks correctly pause execution and accept/resume with injected feedback; state reflects the decision.
- LangSmith traces show state diffs, tool parameters, latencies, and run metadata for filtering.
- Memory layer returns relevant prior decisions without bloating the state payload due to summarization/pruning.
- Safety policy blocks disallowed inputs and records the decision.

## References
- Quickstart patterns: see `README.md` and examples under `examples/` for `create_react_agent` and checkpointer usage.
- Checkpointing backends: `libs/checkpoint-sqlite` (dev) and `libs/checkpoint-postgres` (prod) for plug-and-play persistence.
