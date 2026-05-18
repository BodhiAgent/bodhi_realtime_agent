# CLAUDE.md
Session ID: 041ded2b-eea5-453d-9f85-f5908368b601

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Rules
- **Codex Oversight:** Codex is monitoring your work while you are in Plan Mode.

- **Mandatory Consultation:** If the ask-codex skill is available, you must consult Codex regarding the code before exiting Plan Mode and handing it over for user review.

- **Approval First:** Describe your proposed approach and wait for approval before writing any code.

- **Clarify Ambiguity:** If the requirements are vague, ask clarifying questions before proceeding with code implementation.

- **Post-Implementation Analysis:** After completing any code, list potential edge cases and suggest test cases to cover them.

- **Task Decomposition:** If a task requires modifying more than 3 files, stop and break it down into smaller, manageable sub-tasks.

- **Test-Driven Bug Fixing:** When a bug occurs, first write a test that reproduces the bug, then apply fixes until the test passes.

- **Error Reflection:** Every time I correct you, reflect on what went wrong and create a plan to ensure the mistake is never repeated.

- **Strict Mode Separation:** Do not write any code while in Plan Mode.

## Project Overview

**Bodhi Realtime Agent Framework** — TypeScript/Node.js framework for real-time voice agent applications with sub-500ms E2E latency. Supports multiple LLM transports via the `LLMTransport` interface: Google Gemini Live API (`@google/genai`) and OpenAI Realtime API (`openai`) for bidirectional audio streaming via WebSocket. Pluggable STT via the `STTProvider` interface (batch or streaming providers).

See `dev_docs/framework/architecture.md` for the full design.

## Architecture Summary

- **Two communication planes:** Realtime plane (WebSocket/WebRTC) carries audio directly between Client ↔ Backend ↔ Gemini — no EventBus on this path. Control plane (EventBus/PubSub) carries agent transitions, tool results, GUI events, turn signals.
- **Two-layer agent model:** **Main agents** are realtime voice agents that own the LLM Live session; they transfer control between each other (e.g., general → booking → general). **Subagents** run in background via Vercel AI SDK (`generateText` + tools) for async tasks; they are not voice agents.
- **ToolExecutor per agent:** Each agent (main or sub) owns a `ToolExecutor` that manages tool call validation (Zod), execution, cancellation (AbortSignal), and result delivery. Tool results always flow back through the LLM — never spoken directly.
- **Inline/Background tools:** Inline tools execute within the voice turn. Background tools spawn a subagent (control plane) that loops until done. Subagent results cross back to realtime plane via `sendToolResponse(WHEN_IDLE)` using the original `toolCallId` — the LLM then generates a natural voice response to inform the user.
- **Non-blocking agent transfer:** Reconfigure LLM session immediately on main agent transfer (no drain-then-start).
- **Multi-transport STT:** Pluggable `STTProvider` interface for user input transcription. `GeminiBatchSTTProvider` (batch, uses `generateContent()`), with streaming providers (ElevenLabs Scribe v2) planned. Audio format negotiated via `configure(STTAudioConfig)` — provider receives the transport's actual sample rate (16kHz Gemini, 24kHz OpenAI).

- **Session management:** `SessionManager` tracks Gemini Live session state machine (CREATED→CONNECTING→ACTIVE→RECONNECTING/TRANSFERRING→CLOSED). Handles `goAway` gracefully (save handle → buffer → reconnect → replay). `ResumptionState` tracks latest handle and buffered messages.
- **Dual-authority context:** Gemini server-side history is authoritative for the live session. Framework maintains a local `ConversationContext` shadow (built from transcription events) for subagent context assembly, memory extraction, and crash recovery.
- **Two-level compression:** Gemini server-side compression (high trigger, safety net) + framework-level summarization (earlier trigger, background subagent generates summary, injected as system message to preserve audio modality).
- **Conversation history persistence:** `ConversationHistoryStore` interface with `ConversationHistoryWriter` subscribing to EventBus. Batches writes at turn boundaries (`turn.end`), flushes on agent transfer, writes `SessionReport` on close. PostgreSQL recommended. Framework emits — consumer persists.
- **Persistent memory:** Pluggable `MemoryStore` interface. Current: `JsonMemoryStore` — local JSON files per user (`memory/{userId}.json`) with structured directives and categorized facts. Single-instance deployment only. Swap adapter for multi-instance (DB/object storage).
- **Memory distillation:** Background subagent extracts facts every 5 turns + at checkpoints (agent transfer, tool completion, session end). Uses **merge-on-write** — each extraction produces the complete updated fact list (existing + new merged, deduplicated, contradictions resolved) via `replaceAll()`. No separate consolidation step needed.

- **Parallel subagents (G2/G3):** Main agent spawns multiple background tools concurrently — each runs as an independent subagent. Results arrive independently via `toolResponse(WHEN_IDLE)`. No DAG, no workflow engine — just parallel fan-out using existing background tool mechanism.
- **Actor runtime:** Optional actor-model orchestration (`RuntimeOrchestrator`) that replaces the legacy procedural `VoiceSession` wiring. Actors: `SessionActor` (lifecycle), `TransportActor` (LLM connection), `MainAgentActor` (agent transfer), `ToolRouterActor` (tool dispatch + background tasks), `SubagentSupervisorActor` (subagent lifecycle), `ClientGatewayActor` (client transport). Supervised with dead-letter queue and reconnect recovery. Enabled via `orchestrationMode: 'actor'` in `VoiceSessionConfig`.
- **External audio agents:** `MainAgent.audioMode: 'external'` lets agents bypass the LLM transport and manage their own audio pipeline (e.g., Twilio phone bridge). Framework saves/restores LLM session handles for seamless return.
- **Telephony:** `TwilioBridge` for outbound calls (AI → human transfer), `TwilioWebhookServer` for webhooks/Media Streams, `audio-codec.ts` for mulaw G.711 ↔ PCM L16 conversion. Standalone inbound bridge (`app/twilio-inbound-bridge.ts`) connects phone callers to any VoiceSession.
- **Pluggable TTS:** `TTSProvider` interface for external text-to-speech (actor-mode only). Framework routes LLM text output through `SentenceBuffer` to the provider, handles turn gating, barge-in, resampling, and stale-chunk filtering. Providers: `ElevenLabsTTSProvider`, `CartesiaTTSProvider`.
- **Observability:** Lightweight `FrameworkHooks` — typed callbacks at key points (turn latency breakdown, tool calls, agent transfers, TTS synthesis, errors). No OpenTelemetry dependency. Consumers wire their own handlers. Zero overhead when unattached.

Key abstractions: `VoiceSession`, `SessionManager`, `EventBus`, `MainAgent`, `Subagent`, `AgentRouter`, `ToolExecutor`, `ToolCallRouter`, `ToolDefinition`, `BehaviorManager`, `LLMTransport` (interface), `GeminiLiveTransport`, `OpenAIRealtimeTransport`, `ClientTransport`, `STTProvider` (interface), `GeminiBatchSTTProvider`, `ElevenLabsStreamingSTTProvider`, `TTSProvider` (interface), `CartesiaTTSProvider`, `ElevenLabsTTSProvider`, `TransportFactory`, `ConversationContext`, `ConversationHistoryStore`, `MemoryStore`, `MemoryDistiller`, `FrameworkHooks`, `RuntimeOrchestrator`, `ActorRuntime`, `SessionActor`, `TransportActor`, `MainAgentActor`, `ToolRouterActor`, `SubagentSupervisorActor`, `ClientGatewayActor`, `TwilioBridge`, `TwilioWebhookServer`.

## Tech Stack

- **Runtime:** Node.js 22+, TypeScript 5.5+
- **Voice transport:** `@google/genai` (Gemini Live API), `openai` (OpenAI Realtime API)
- **Subagent engine:** `ai` + `@ai-sdk/google` (Vercel AI SDK)
- **Client transport:** `ws` (WebSocket server)
- **Validation:** Zod
- **Build:** tsup
- **Test:** Vitest
- **Package manager:** pnpm
- **Linting:** Biome

## Commands

```bash
pnpm install          # Install dependencies
pnpm build            # Build TypeScript
pnpm test             # Run all tests
pnpm test -- --run src/core/event-bus.test.ts  # Run single test file
pnpm lint             # Lint with Biome
pnpm lint --fix       # Auto-fix lint issues
```

## Environment Variables

```
GOOGLE_API_KEY=           # Required: Gemini API key for Live API
OPENAI_API_KEY=           # Required for OpenAI Realtime transport
ELEVENLABS_API_KEY=       # Optional: ElevenLabs streaming STT (falls back to Chrome browser STT if absent)
GOOGLE_CLOUD_PROJECT=     # Optional: GCP project (Vertex AI)
GOOGLE_CLOUD_LOCATION=    # Optional: GCP location (Vertex AI)
LOG_LEVEL=debug           # debug | info | warn | error
```

## Project Structure

```
src/
  core/           # EventBus, VoiceSession, SessionManager, ConversationContext, ToolCallRouter
  transport/      # LLMTransport, GeminiLiveTransport, OpenAIRealtimeTransport, ClientTransport, STTProvider, TTSProvider
  agent/          # MainAgent, Subagent, AgentRouter, AgentContext
  tools/          # ToolDefinition, ToolExecutor, sandbox
  behaviors/      # BehaviorManager, built-in presets (speechSpeed, verbosity, etc.)
  audio/          # SentenceBuffer, resamplePcm, format conversion utilities
  runtime/        # Actor runtime: ActorRuntime, RuntimeOrchestrator, Supervisor, DeadLetterQueue
  runtime/actors/ # SessionActor, TransportActor, MainAgentActor, ToolRouterActor, SubagentSupervisorActor, ClientGatewayActor
  runtime/adapters/ # Adapters bridging actor messages to legacy VoiceSession interfaces
  telephony/      # TwilioBridge, TwilioWebhookServer, audio-codec (mulaw G.711 ↔ PCM L16)
  memory/         # MemoryStore interface, JSON file adapter, merge-on-write distiller
  types/          # Shared types, event type definitions
```

## Coding Conventions

- **License header:** Every source file under `src/`, `test/`, and `docs/` MUST start with `// SPDX-License-Identifier: MIT` as the first line
- Use `interface` for public contracts, `type` for unions/intersections
- Zod schemas for all external input validation (tool parameters, config)
- Errors: throw typed error classes extending `FrameworkError` base
- Async: native `async/await`, no raw callbacks except for Gemini SDK bridge
- Naming: `camelCase` for variables/functions, `PascalCase` for types/classes, `UPPER_SNAKE` for constants
- Tests: colocate as `*.test.ts` next to source files
- Audio format: transport-dependent. Gemini: PCM 16-bit LE 16kHz mono input, 24kHz output. OpenAI: PCM 16-bit LE 24kHz mono both directions. Negotiated via `LLMTransport.audioFormat`.

## Instrumentation

Every new transport, tool, or agent component MUST fire the relevant `FrameworkHooks` callback. Hooks are defined in `dev_docs/framework/architecture.md` under Observability.

**Required hook points:**

| Component action | Hook to fire | Who measures |
|---|---|---|
| Tool execution starts | `onToolCall` | `ToolExecutor` — before `tool.execute()` |
| Tool execution completes/fails/cancels | `onToolResult` with `durationMs` | `ToolExecutor` — wall-clock time of `execute()` |
| Agent transfer completes | `onAgentTransfer` with `reconnectMs` | `SessionManager` — WS close to `setupComplete` |
| Turn completes | `onTurnLatency` with segment breakdown | Transports — each timestamps its segment |
| Subagent step completes | `onSubagentStep` | Subagent runner — from AI SDK `onStepFinish` |
| Memory extraction completes | `onMemoryExtraction` | `MemoryDistiller` |
| Any caught error | `onError` with component + severity | The catching component |

**Pattern:** Always use a zero-overhead guard:
```typescript
if (hooks.onToolCall) hooks.onToolCall({ sessionId, toolCallId, toolName, execution, agentName });
```

**Latency segment ownership:**
- `clientToBackendMs`: `ClientTransport` timestamps audio chunk arrival
- `backendToGeminiMs`: `GeminiLiveTransport` timestamps forwarding to Gemini
- `geminiProcessingMs`: `GeminiLiveTransport` measures last sent → first response (TTFB)
- `backendToClientMs`: `ClientTransport` timestamps response relay
- `totalE2EMs`: `VoiceSession` computes client arrival → client delivery

When adding a new component, check which hooks apply and wire them. If no existing hook covers a new event, propose an addition to `FrameworkHooks` in the architecture doc.

## Conversation History Persistence

Conversation data flows through a strict pipeline. Follow these rules:

- **All conversation mutations go through `ConversationContext`** — never write to `ConversationHistoryStore` directly. `ConversationContext` is the single in-memory authority for the current session's state.
- **Persistence is EventBus-driven.** `ConversationHistoryWriter` subscribes to `turn.end`, `agent.transfer`, and `session.close` events. It reads new items from `ConversationContext.getItemsSinceCheckpoint()` and batch-inserts to the database. No other component should call `ConversationHistoryStore.addItems()`.
- **New modules that produce conversation items** (e.g., a new tool type, a new agent lifecycle event) must emit via EventBus, not call storage directly. The writer picks them up at the next turn boundary.
- **Never mutate `ConversationContext` from outside the session layer.** Only `VoiceSession` and its direct children (transcription handlers, tool result handlers) should call `addUserMessage()`, `addToolCall()`, etc.

## Key Reference Docs

- `dev_docs/README.md` — Where internal docs live (framework vs `app/` server, touchpoints, integrations)
- `docs/service/hosted-voice-api.md` — External integrators: HTTPS + WSS for hosted Bodhi (not framework wiki)
- `dev_docs/framework/architecture.md` — Full architecture design, event flows, latency budget
- `dev_docs/framework/architecture-critique.md` — Known gaps and open design questions
- `dev_docs/framework/execution-plan.md` — 111-step (and growing) implementation plan
- `Bodhi_Agent_Architect.md` — Original design requirements document
- `dev_docs/framework/investigation-js-genai.md` — Gemini Live API capabilities and wire protocol
- `dev_docs/framework/investigation-gemini-session.md` — Gemini session resumption, context compression, goAway handling
- `dev_docs/framework/investigation-llm-agent-decoupling.md` — LLMTransport interface design, 10 coupling layers, recovery contract
- `dev_docs/framework/investigation-openai-realtime-compatibility.md` — Gemini vs OpenAI protocol comparison, 13 gaps
- `dev_docs/framework/investigation-openai-realtime-node-sdk.md` — OpenAI SDK usage, concept mapping, GA API changes
- `dev_docs/framework/investigation-realtime-stt-providers.md` — Deepgram/ElevenLabs/AssemblyAI/OpenAI STT comparison
- `dev_docs/framework/investigation-streaming-stt-display.md` — Root cause of non-streaming transcript display
- `dev_docs/framework/investigation-requirements.md` — Requirements from Bodhi_Agent_Architect.md
- `dev_docs/framework/investigation-livekit-agents.md` — LiveKit agents-js analysis and 13 identified limitations
- `dev_docs/framework/investigation-livekit-context.md` — LiveKit two-level ChatContext, dual-write, diff-based sync
- `dev_docs/framework/investigation-memory-solutions.md` — Mem0, Zep/Graphiti, sliding window, Redis persistence
- `dev_docs/framework/investigation-subagent-frameworks.md` — AI SDK vs GenKit vs LangGraph vs others; chose Vercel AI SDK
- `dev_docs/framework/investigation-conversation-history.md` — Persistence patterns from Retell/Vapi/LiveKit; turn-boundary batching
- `dev_docs/framework/investigation-memory-distillation.md` — Extraction triggers, prompts, consolidation from Mem0/OpenAI/Zep
- `dev_docs/framework/design-behavior-manager.md` — BehaviorManager design: declarative behavior tuning with auto-generated tools
- `dev_docs/app/integrations/design-stt-provider-interface.md` — STTProvider interface, audio format negotiation, turn-aware ordering
- `dev_docs/app/integrations/design-tts-provider-interface.md` — TTSProvider interface, sentence buffering, turn gating, barge-in
- `dev_docs/framework/design-full-actor-workflow-engine.md` — Actor/workflow runtime design: actors, supervisors, message contracts, migration phases
- `dev_docs/framework/design-interactive-subagent-session.md` — V2 interactive subagent sessions: SubagentSession, ask_user tool, routing
- `dev_docs/framework/design-openclaw-subagent.md` — OpenClaw integration: WebSocket relay, protocol mapping
- `dev_docs/app/integrations/design-twilio-human-transfer.md` — Twilio human transfer: TwilioBridge, external audio agents, session resumption
- `dev_docs/app/touchpoints/design-twilio-inbound-bridge.md` — Twilio inbound call bridge: standalone protocol translator, codec generalization
- `dev_docs/framework/design-openclaw-file-transfer.md` — OpenClaw file/image transfer pipeline: artifacts, multimodal flows
