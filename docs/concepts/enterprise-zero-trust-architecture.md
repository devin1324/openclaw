---
summary: "End-to-end OpenClaw architecture map with enterprise zero-trust overlays"
read_when:
  - Designing enterprise deployments
  - Planning zero-trust boundaries
  - Mapping channels, routing, and execution surfaces
title: "Enterprise zero-trust architecture"
---

# Enterprise zero-trust architecture

## 1) Full system map (logical)

```mermaid
flowchart TB
  subgraph U[Users and operators]
    WA[WhatsApp users]
    TG[Telegram users]
    SL[Slack users]
    DC[Discord users]
    SG[Signal users]
    IM[iMessage users]
    WEB[WebChat users]
    OP[Operators: CLI, macOS app, web UI]
  end

  subgraph C[Channel adapters + plugin channels]
    CHCORE[Core channels: WhatsApp Telegram Slack Discord Signal iMessage IRC Google Chat]
    CHEXT[Extension channels: Teams Matrix Zalo ZaloUser BlueBubbles etc]
  end

  subgraph G[Gateway control plane (single WS/HTTP runtime)]
    WS[WebSocket API: req res event]
    HTTP[HTTP surfaces: Control UI chat endpoints hooks canvas host]
    AUTH[Auth pairing device trust token checks]
    ROUTE[Routing + session key resolution + bindings]
    METHODS[Gateway methods + events catalog]
    HEALTH[Presence health heartbeat cron telemetry]
  end

  subgraph A[Agent runtime]
    MODEL[LLM model provider calls]
    TOOLS[Tools exec browser filesystem network node invoke]
    POLICY[Tool policy + approvals + allow/deny profiles]
    MEMORY[Session history and memory]
  end

  subgraph X[Execution isolation]
    HOST[Host execution]
    SANDBOX[Per-session sandbox Docker mode]
    NODE[Paired nodes macOS iOS Android headless]
  end

  subgraph S[State and secrets]
    CFG[openclaw.json runtime config]
    CREDS[credentials and channel auth material]
    SESS[session storage + previews]
    LOGS[logs and audit streams]
  end

  WA --> CHCORE
  TG --> CHCORE
  SL --> CHCORE
  DC --> CHCORE
  SG --> CHCORE
  IM --> CHCORE
  WEB --> HTTP
  OP --> WS
  OP --> HTTP

  CHCORE --> ROUTE
  CHEXT --> ROUTE
  ROUTE --> A
  WS --> METHODS
  HTTP --> METHODS
  AUTH --> WS
  AUTH --> HTTP
  METHODS --> A
  HEALTH --> OP

  A --> MODEL
  A --> TOOLS
  TOOLS --> POLICY
  POLICY --> HOST
  POLICY --> SANDBOX
  TOOLS --> NODE
  A --> MEMORY

  G <--> S
  A <--> S
```

## 2) Runtime trust boundaries (recommended enterprise layering)

```mermaid
flowchart LR
  subgraph T0[Boundary 0: External networks]
    EXTUSERS[External users and channel networks]
  end

  subgraph T1[Boundary 1: Gateway ingress]
    TLS[TLS termination]
    AUTHZ[Gateway auth + device pairing + rate limits]
    INGRESS[WS and HTTP ingress]
  end

  subgraph T2[Boundary 2: Control plane core]
    ROUTER[Route resolver + session mapper]
    EVENTS[Event bus and method handlers]
    CONFIG[Config + policy evaluation]
  end

  subgraph T3[Boundary 3: Agent execution]
    AGENT[Model orchestration]
    APPROVALS[Exec approval manager]
    TOOLING[Tool adapter layer]
  end

  subgraph T4[Boundary 4: Data and privileged resources]
    SECRETS[Secrets and credentials store]
    FS[File system workspace]
    DEVICES[Paired node devices]
    AUDIT[Audit + security telemetry]
  end

  EXTUSERS --> INGRESS --> AUTHZ --> ROUTER --> AGENT --> TOOLING --> FS
  TOOLING --> DEVICES
  CONFIG --> ROUTER
  CONFIG --> TOOLING
  APPROVALS --> TOOLING
  AGENT --> SECRETS
  EVENTS --> AUDIT
  AUTHZ --> AUDIT
```

## 3) Practical zero-trust target state for enterprise hardening

```mermaid
flowchart TB
  subgraph E1[Identity]
    I1[Mutual identity per client device]
    I2[Non-local explicit pairing only]
    I3[Short-lived rotated tokens]
  end

  subgraph E2[Policy]
    P1[Deny-by-default channel allowlists]
    P2[Per-agent scoped tools profile]
    P3[Per-session sandbox non-main or stricter]
    P4[Two-party approval for high-risk exec]
  end

  subgraph E3[Segmentation]
    S1[One trust boundary per gateway instance]
    S2[Dedicated OS user and isolated host per boundary]
    S3[Separate credentials per environment and channel]
  end

  subgraph E4[Verification]
    V1[Continuous security audit command]
    V2[Structured logs with sensitive redaction]
    V3[Health checks + anomaly alerts]
  end

  E1 --> E2 --> E3 --> E4
```

## 4) Channel to agent routing model (how inbound turns become execution)

```mermaid
sequenceDiagram
  participant User
  participant Channel
  participant Gateway
  participant Router
  participant Agent
  participant Policy
  participant Tooling

  User->>Channel: inbound message
  Channel->>Gateway: normalized inbound event
  Gateway->>Router: resolve route by channel/account/peer/binding
  Router-->>Gateway: agentId + sessionKey + matchedBy
  Gateway->>Agent: run with session + memory context
  Agent->>Policy: request tool capability
  Policy-->>Agent: allow/deny/approval required
  Agent->>Tooling: execute approved action
  Tooling-->>Agent: result
  Agent-->>Gateway: response chunks + final
  Gateway-->>Channel: outbound message
```

## 5) Enterprise implementation notes

- Keep one gateway per trust boundary and avoid adversarial multi-tenant sharing of one runtime.
- Place gateway ingress behind private network controls and require explicit authentication/pairing.
- Use minimal tool profiles plus sandbox defaults for non-main sessions, then elevate only by explicit approval flow.
- Maintain separate channel credentials, model keys, and operator identities per environment (dev/stage/prod, team-by-team).
- Treat node invocation (camera/screen/location/canvas) as privileged remote execution and gate it like production infrastructure access.
