# OpenClaw Architecture & Operating Principles — Visual Guide

A comprehensive visual knowledge transfer covering every major subsystem, the project's evolution, and design principles.

---

## Diagram 1: System Architecture Overview

```mermaid
graph TD
    subgraph CLI["CLI Layer"]
        Entry["entry.ts"]
        RunMain["cli/run-main.ts<br/>Commander.js"]
    end

    subgraph Gateway["Gateway Server"]
        Server["gateway/server.ts<br/>Hono HTTP + WebSocket"]
        ServerChat["server-chat.ts<br/>Chat Orchestration"]
        ServerChannels["server-channels.ts<br/>Channel Lifecycle"]
        ServerPlugins["server-plugins.ts<br/>Plugin Wiring"]
    end

    subgraph Routing["Routing & Sessions"]
        ResolveRoute["routing/resolve-route.ts<br/>Binding -> Agent -> SessionKey"]
        SessionKey["routing/session-key.ts<br/>Key Construction"]
        Bindings["routing/bindings.ts<br/>Channel/Account/Peer Match"]
        SessionStore["channels/session.ts<br/>Session State Persistence"]
    end

    subgraph Agents["Agent Runtime"]
        PiRunner["agents/pi-embedded-runner/<br/>Pi Agent Core"]
        SystemPrompt["agents/system-prompt.ts"]
        ModelCatalog["agents/model-catalog.ts"]
        Tools["agents/pi-tools.ts<br/>Tool Definitions"]
        Subagents["agents/subagent-registry.ts"]
    end

    subgraph Channels["Channel Adapters"]
        Registry["channels/registry.ts"]
        ChannelPlugins["extensions/<br/>Slack, Discord, Telegram,<br/>WhatsApp, Signal, etc."]
        Transport["channels/transport/<br/>Inbound/Outbound"]
        RunStateMachine["channels/run-state-machine.ts<br/>Draft -> Stream -> Send"]
    end

    subgraph Plugins["Plugin System"]
        Discovery["plugins/discovery.ts"]
        Loader["plugins/loader.ts<br/>jiti dynamic import"]
        PluginRegistry["plugins/registry.ts"]
        Hooks["plugins/hooks.ts<br/>25+ Lifecycle Hooks"]
        PluginSDK["plugin-sdk/<br/>70+ Subpath Exports"]
    end

    subgraph Support["Support Subsystems"]
        Memory["memory/ LanceDB Vector Search"]
        Media["media/ Pipeline"]
        Config["config/ JSON5 + Zod"]
        TUI["tui/ Terminal UI"]
    end

    Entry --> RunMain --> Server
    Server --> ServerChat --> ResolveRoute
    Server --> ServerChannels --> Registry --> ChannelPlugins --> Transport
    Server --> ServerPlugins --> Discovery --> Loader --> PluginRegistry --> Hooks
    ResolveRoute --> Bindings --> SessionKey --> SessionStore
    ServerChat --> PiRunner --> SystemPrompt & ModelCatalog & Tools & Subagents
    PiRunner --> RunStateMachine --> Transport
    PluginSDK -.->|definePluginEntry| PluginRegistry
    PiRunner --> Memory
    Tools --> Media
    Server --> Config
```

---

## Diagram 2: Message Flow Pipeline (Sequence)

```mermaid
sequenceDiagram
    participant Ext as External Platform<br/>(Slack/Discord/Telegram)
    participant Chan as Channel Adapter
    participant GW as Gateway Server
    participant Hook as Plugin Hooks
    participant Route as Route Resolver
    participant Sess as Session Store
    participant Agent as Pi Agent Runner
    participant LLM as LLM Provider
    participant SM as Run State Machine

    Ext->>Chan: Webhook / WebSocket event
    Chan->>GW: Normalized InboundMessage

    GW->>Hook: inbound_claim hook
    GW->>Hook: message_received hook

    GW->>Route: resolveAgentRoute({channel, accountId, peer})
    Route->>Route: Match bindings priority:<br/>peer > parent > guild+roles > account > channel > default
    Route-->>GW: {agentId, sessionKey}

    GW->>Sess: recordInboundSession(sessionKey)
    GW->>Hook: before_agent_start hook

    GW->>Agent: runEmbeddedPiAgent(session, message)
    Agent->>Hook: before_prompt_build hook
    Agent->>Hook: llm_input hook
    Agent->>LLM: API call (streaming)
    LLM-->>Agent: Token stream + tool calls
    Agent->>Hook: llm_output hook

    loop Tool Execution
        Agent->>Hook: before_tool_call hook
        Agent->>Agent: Execute tool
        Agent->>Hook: after_tool_call hook
    end

    Agent->>Hook: agent_end hook
    Agent-->>GW: Agent result

    GW->>SM: Draft -> chunk -> finalize
    SM->>Hook: message_sending hook
    SM->>Chan: Platform-specific formatting
    Chan->>Ext: Send reply
    SM->>Hook: message_sent hook
```

---

## Diagram 3: Plugin System Lifecycle

```mermaid
graph TD
    subgraph Discovery["Phase 1: Discovery"]
        Scan["resolvePluginSourceRoots()<br/>~/.openclaw/extensions/<br/>workspace node_modules/<br/>bundled stock plugins"]
        Find["Scan for package.json<br/>with openclaw manifest"]
        Candidates["PluginCandidate[]"]
        Scan --> Find --> Candidates
    end

    subgraph Loading["Phase 2: Loading"]
        Config["normalizePluginsConfig()<br/>enable/disable state"]
        Jiti["jiti createJiti() dynamic import<br/>TS/ESM/CJS support"]
        Module["OpenClawPluginModule"]
        Candidates --> Config --> Jiti --> Module
    end

    subgraph Registration["Phase 3: Registration"]
        API["OpenClawPluginApi"]
        Module --> API
        RegChannel["registerChannel()"]
        RegProvider["registerProvider()"]
        RegHook["registerHook()"]
        RegTool["registerTool()"]
        RegHTTP["registerHttpRoute()"]
        RegCommand["registerCommand()"]
        API --> RegChannel & RegProvider & RegHook & RegTool & RegHTTP & RegCommand
    end

    subgraph Runtime["Phase 4: Runtime"]
        HookRunner["createHookRunner(registry)<br/>run* methods, priority-ordered"]
        Integration["Gateway + Agents<br/>consume registered plugins"]
        RegChannel & RegProvider & RegHook & RegTool --> HookRunner --> Integration
    end
```

---

## Diagram 4: Project Evolution Timeline

```mermaid
gantt
    title OpenClaw Evolution (Nov 2025 - Mar 2026, 23,645 commits)
    dateFormat YYYY-MM-DD

    section Phase 1: Webhook Relay
    Initial commit - warelay CLI, 511 lines    :p1, 2025-11-24, 3d

    section Phase 2: Multi-Channel + AI
    WhatsApp Web + Auto-reply via Claude        :p2, 2025-11-27, 4d
    v0.1.0                                      :milestone, 2025-12-01, 0d

    section Phase 3: Media Pipeline
    Media, audio transcription, multi-provider  :p3, 2025-12-02, 2d
    v1.0.4                                      :milestone, 2025-12-04, 0d

    section Phase 4: Control Plane
    Health heartbeat, event streaming, cron     :p4, 2025-12-04, 15d
    v1.2.2                                      :milestone, 2025-12-19, 0d

    section Phase 5: Platform Rewrite (1,463 commits)
    Gateway + Plugin SDK + Provider abstraction :p5a, 2025-12-19, 14d
    Multi-device, Canvas, native macOS/iOS      :p5b, 2026-01-02, 14d
    v2.0.0-beta5                                :milestone, 2026-01-16, 0d

    section Phase 6: Ecosystem Maturation
    Calendar versioning, 75+ extensions         :p6, 2026-01-16, 62d
    v2026.3.x (current)                         :milestone, 2026-03-19, 0d
```

---

## Diagram 5: Extension Ecosystem Map

```mermaid
mindmap
  root((OpenClaw Extensions<br/>75+ packages))
    **Channel Extensions**
      Mainstream
        slack
        discord
        telegram
        msteams
        googlechat
      Messaging
        whatsapp
        signal
        imessage
        matrix
        line
        feishu
      Niche
        irc
        nostr
        twitch
        mattermost
    **LLM Providers**
      Tier 1
        anthropic
        openai
        google
      Cloud
        bedrock
        openrouter
        together
        perplexity
      Self-Hosted
        ollama
        vllm
        sglang
      Regional
        qianfan
        minimax
        moonshot
    **Features**
      Memory
        memory-core
        memory-lancedb
      Voice
        voice-call
        elevenlabs
      Dev Tools
        copilot-proxy
        diagnostics-otel
```

---

## Diagram 6: Hook Lifecycle Flow

```mermaid
flowchart TD
    GS[/"gateway_start"/] --> SS[/"session_start"/]
    SS --> MR[/"message_received"/]
    MR --> IC[/"inbound_claim"/]
    IC --> BAS[/"before_agent_start"/]
    BAS --> BMR[/"before_model_resolve"/]
    BMR --> BPB[/"before_prompt_build"/]
    BPB --> LI[/"llm_input"/]
    LI --> LO[/"llm_output"/]

    LO -->|tool_use| BTC[/"before_tool_call"/]
    BTC --> ATC[/"after_tool_call"/]
    ATC --> TRP[/"tool_result_persist"/]
    TRP -->|more tools| BTC
    TRP -->|done| LI

    LO -->|context overflow| BC[/"before_compaction"/]
    BC --> AC[/"after_compaction"/]
    AC --> BPB

    LO -->|spawn subagent| SPS[/"subagent_spawning"/]
    SPS --> SDT[/"subagent_delivery_target"/]
    SDT --> SPD[/"subagent_spawned"/]
    SPD --> SEN[/"subagent_ended"/]

    LO -->|final response| BMW[/"before_message_write"/]
    BMW --> MSN[/"message_sending"/]
    MSN --> MST[/"message_sent"/]
    MST --> AE[/"agent_end"/]
    AE --> SE[/"session_end"/]
    SE --> GE[/"gateway_stop"/]

    style GS fill:#2d6a4f,color:#fff
    style GE fill:#9b2226,color:#fff
    style LI fill:#e76f51,color:#fff
    style LO fill:#e76f51,color:#fff
    style BTC fill:#f4a261,color:#000
    style ATC fill:#f4a261,color:#000
```

---

## Diagram 7: Agent & Session Architecture

```mermaid
flowchart TB
    MSG["Inbound Message"] --> ROUTE

    subgraph ROUTE["Route Resolution"]
        RI["Input: channel, accountId, peer, guildId, teamId, roles"]
        MATCH["Binding Priority Cascade:<br/>1. peer<br/>2. parent peer<br/>3. guild+roles<br/>4. guild<br/>5. team<br/>6. account<br/>7. channel<br/>8. default"]
        RAR["Output: agentId + sessionKey"]
        RI --> MATCH --> RAR
    end

    RAR --> SKEY

    subgraph SKEY["Session Key: agent:{id}:{scope}"]
        DM["DM: agent:main:dm:{senderId}"]
        GRP["Group: agent:main:group:{groupId}"]
        THR["Thread: agent:main:thread:{threadId}"]
        GLB["Global: agent:main:main"]
    end

    SKEY --> AGENT

    subgraph AGENT["Agent Execution"]
        MAIN["Main Agent"]
        SUB["Subagents via ACP spawn"]
        TOOLS["Tools: bash, plugins, MCP, skills, memory"]
        AUTH["Auth Profiles: OAuth + API key fallback"]
        MAIN --> TOOLS & AUTH
        SUB --> TOOLS & AUTH
    end

    style ROUTE fill:#e3f2fd,stroke:#2196F3
    style SKEY fill:#fff3e0,stroke:#FF9800
    style AGENT fill:#e8f5e9,stroke:#4CAF50
```

---

## Diagram 8: Configuration & Validation Pipeline

```mermaid
flowchart LR
    subgraph Sources["Sources (priority order)"]
        CLI["CLI Flags"]
        ENV["Env Vars"]
        FILE["JSON5 Config"]
    end

    Sources --> LOAD

    subgraph LOAD["Load"]
        PARSE["JSON5.parse()"]
        ENVSUB["${VAR} substitution"]
        INCLUDES["Resolve includes"]
        PARSE --> ENVSUB --> INCLUDES
    end

    LOAD --> VALIDATE

    subgraph VALIDATE["Validate"]
        LEGACY["Legacy migration"]
        ZOD["Zod schema check"]
        PLUGIN_V["Plugin-aware validation"]
        LEGACY --> ZOD --> PLUGIN_V
    end

    VALIDATE --> DEFAULTS

    subgraph DEFAULTS["Normalize"]
        D["Apply defaults:<br/>agents, models, sessions,<br/>logging, compaction"]
    end

    DEFAULTS --> OUTPUT["OpenClawConfig<br/>(typed, validated)"]
    OUTPUT --> PERSIST["Atomic write<br/>+ backup rotation"]
```

---

## Diagram 9: Design Principles & Build Order (Teaching Map)

```mermaid
graph TB
    CORE(("OpenClaw<br/>Design Principles"))

    CORE --> ADAPTER["Adapter Pattern<br/>Channels & Providers are<br/>interchangeable adapters"]
    CORE --> PLUGIN["Plugin-First<br/>Even bundled channels<br/>are plugins. Core is thin."]
    CORE --> HOOKS["Hook-Driven<br/>25+ lifecycle hooks replace<br/>tight coupling & inheritance"]
    CORE --> SESSION["Session Isolation<br/>Concurrent users never<br/>interfere. Key = scope."]
    CORE --> CONFIG["Config-as-Code<br/>JSON5 + Zod schemas =<br/>human-friendly + type-safe"]
    CORE --> BOUNDARY["Strict Boundaries<br/>70+ SDK subpaths prevent<br/>cross-cutting imports"]
    CORE --> EVENT["Event-Driven<br/>Message pipeline with<br/>hooks at every stage"]

    subgraph BUILD["Recommended Build Order for Similar System"]
        direction LR
        B1["1. Simple<br/>webhook relay"] --> B2["2. Channel<br/>adapter interface"]
        B2 --> B3["3. Session key<br/>+ routing"]
        B3 --> B4["4. Config system<br/>+ Zod validation"]
        B4 --> B5["5. Plugin lifecycle<br/>discover->load->register->run"]
        B5 --> B6["6. Hook system<br/>+ tool registry"]
        B6 --> B7["7. Multi-agent<br/>+ subagent spawn"]
    end

    CORE --> BUILD

    style CORE fill:#e8eaf6,stroke:#3F51B5,stroke-width:3px
    style BUILD fill:#efebe9,stroke:#795548,stroke-width:2px
```

---

## Key Files Reference

| Component | File Path |
|-----------|-----------|
| Entry point | `src/entry.ts` |
| CLI routing | `src/cli/run-main.ts` |
| Gateway | `src/gateway/server.ts`, `server-chat.ts` |
| Route resolver | `src/routing/resolve-route.ts` |
| Session keys | `src/routing/session-key.ts` |
| Plugin discovery | `src/plugins/discovery.ts` |
| Plugin loader | `src/plugins/loader.ts` |
| Plugin registry | `src/plugins/registry.ts` |
| Hook system | `src/plugins/hooks.ts` |
| Agent runtime | `src/agents/pi-embedded-runner.ts` |
| Agent scope | `src/agents/agent-scope.ts` |
| Config loader | `src/config/io.ts` |
| Config types | `src/config/types.*.ts` (37 files) |
| Plugin SDK | `src/plugin-sdk/` (70+ subpaths) |
| Channel plugins | `extensions/*/` (75+ extensions) |
