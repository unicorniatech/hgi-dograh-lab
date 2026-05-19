# HGI Dograh Voice Infrastructure Audit

**Date:** May 19, 2026  
**Auditor:** HGI Development Team  
**Repository:** https://github.com/unicorniatech/hgi-dograh-lab  
**Scope:** Evaluation for HGI-VOICE-BRIDGE, MOLIE voice, EVA voice, Marshanta voice ordering, RedVecinal emergency voice flows

---

## Executive Summary

| Criteria | Status | Notes |
|----------|--------|-------|
| **License** | ✅ BSD-2-Clause | Permissive, compatible with HGI |
| **Self-Hosting** | ✅ Full support | Docker-first deployment |
| **Offline Capability** | ⚠️ Partial | Requires cloud for Dograh-managed STT/TTS/LLM |
| **Local-First** | ⚠️ Possible | With local LLM (Speaches), but complex |
| **HGI-EDGE Integration** | ❌ No native | Requires custom bridge |
| **Privacy** | ⚠️ Configurable | Telemetry opt-out available |

**Recommendation:** **USE AS INSPIRATION ONLY** — Adopt architectural patterns but do not adopt Dograh as a runtime dependency.

---

## 1. Repository Structure Analysis

### 1.1 Tech Stack Overview

| Component | Technology | Version | HGI Relevance |
|-----------|------------|---------|---------------|
| **Backend Framework** | FastAPI (Python) | 0.135.3 | ✅ Proven, async |
| **Frontend Framework** | Next.js + React | 15.3.3, 19.1.0 | Modern, good patterns |
| **Database** | PostgreSQL + pgvector | PG17 | ✅ Industry standard |
| **Cache/Queue** | Redis + ARQ | 5.3.1, 0.26.3 | ✅ Standard stack |
| **Audio Pipeline** | Pipecat Framework | Custom fork | Core dependency |
| **WebRTC** | small-webrtc-transport | Custom | Self-hosted TURN |
| **Telephony** | Twilio/Vonage/Telnyx/etc. | - | Multiple providers |
| **Storage** | MinIO (S3-compatible) | Latest | ✅ Self-hosted option |
| **Authentication** | Local JWT + Stack Auth | - | OSS: local only |

### 1.2 Key Directories

```
dograh/
├── api/                    # FastAPI backend
│   ├── services/           # Core business logic
│   │   ├── pipecat/        # Voice pipeline framework
│   │   ├── workflow/       # Workflow builder engine
│   │   ├── telephony/      # Telephony providers (7+ integrations)
│   │   ├── configuration/  # STT/TTS/LLM registry
│   │   └── gen_ai/         # Embeddings/AI services
│   ├── db/                 # SQLAlchemy models
│   ├── routes/             # API endpoints
│   └── alembic/            # Database migrations
├── ui/                     # Next.js frontend
│   └── src/                # React 19, TypeScript, Tailwind v4
├── pipecat/                # Git submodule (empty in this clone)
├── docs/                   # Mintlify documentation
├── deploy/                 # Deployment scripts
└── scripts/                # Dev helper scripts
```

### 1.3 Dependencies Analysis

**Backend Requirements** (`api/requirements.txt`):
- `fastapi==0.135.3` - Web framework
- `asyncpg==0.30.0` - PostgreSQL async driver
- `redis==5.3.1` - Cache/queue client
- `arq==0.26.3` - Distributed task queue
- `twilio==9.8.0` - Telephony (optional)
- `minio==7.2.16` - S3-compatible storage
- `posthog==7.11.1` - **Telemetry**
- `langfuse==3.9.3` - Tracing
- `sentry-sdk==2.38.0` - Error tracking
- `fastmcp==3.2.4` - MCP server support

**Pipecat Services** (from `api/services/pipecat/service_factory.py`):
- 9 STT providers (Deepgram, OpenAI, Cartesia, Dograh, Sarvam, Speaches, AssemblyAI, Gladia, Speechmatics)
- 10 TTS providers (Deepgram, OpenAI, ElevenLabs, Cartesia, Dograh, Camb, Sarvam, Rime, Speaches)
- 9 LLM providers (OpenAI, Groq, OpenRouter, Google, Azure, Dograh, AWS Bedrock, Speaches)
- 3 Realtime providers (OpenAI Realtime, Google Realtime, Google Vertex Realtime)

---

## 2. License Analysis

### 2.1 License Terms

**BSD 2-Clause License** — Zansat Technologies Private Limited, 2025

```
BSD 2-Clause License

Copyright (c) 2025, Zansat Technologies Private Limited

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice...
2. Redistributions in binary form must reproduce the above copyright notice...
```

### 2.2 License Implications for HGI

| Aspect | Status | Details |
|--------|--------|---------|
| **Commercial Use** | ✅ Allowed | BSD 2-Clause permits commercial use |
| **Modification** | ✅ Allowed | Can fork and modify |
| **Distribution** | ✅ Allowed | No copyleft requirements |
| **Attribution** | ✅ Minimal | Must preserve copyright notice |
| **Patent Grant** | ❌ None | BSD-2 has no explicit patent grant |
| **Model Licensing** | ⚠️ Separate | STT/TTS/LLM models have their own licenses |

### 2.3 Model/Provider License Issues

| Provider | Model | License Risk |
|----------|-------|--------------|
| OpenAI | GPT-4.1, Whisper, TTS | Proprietary API terms |
| Google | Gemini, Vertex AI | Proprietary API terms |
| Deepgram | Nova STT, Aura TTS | Commercial license |
| ElevenLabs | TTS | Proprietary, data concerns |
| **Speaches** | Whisper/Faster-Whisper, Kokoro | ✅ Open source (Apache 2.0) |
| Groq | LLaMA, Mixtral | Depends on base model |
| OpenRouter | Various | Varies by endpoint |

**Key Finding:** Dograh's "auto-generated keys" for quick start use their hosted service (MPS - Model Proxy Service), which is **proprietary** and requires connectivity to `services.dograh.com`.

---

## 3. Deployment Architecture

### 3.1 Services Required (Production - docker-compose.yaml)

| Service | Image | Port | Purpose | Self-Hostable |
|---------|-------|------|---------|---------------|
| **postgres** | pgvector/pgvector:pg17 | 5432 | Database + vector store | ✅ Yes |
| **redis** | redis:7 | 6379 | Cache + job queue | ✅ Yes |
| **minio** | minio/minio | 9000/9001 | Object storage | ✅ Yes |
| **api** | dograhai/dograh-api | 8000 | FastAPI backend | ✅ Yes |
| **ui** | dograhai/dograh-ui | 3010 | Next.js frontend | ✅ Yes |
| **coturn** | coturn/coturn:4.8.0 | 3478 | TURN server for WebRTC | ✅ Yes |
| **nginx** | nginx:alpine | 80/443 | Reverse proxy | ✅ Yes |
| **cloudflared** | cloudflare/cloudflared | 2000 | Tunnel (optional) | ⚠️ Cloudflare |
| **dograh-init** | bash:5.2 | - | Config generator | ✅ Yes |

### 3.2 Development Services (docker-compose-local.yaml)

Minimal stack for local development:
- postgres (5432)
- redis (6379)
- minio (9000/9001)

### 3.3 External API Dependencies

| Service | Required For | Can Self-Host Alternative |
|---------|--------------|---------------------------|
| `services.dograh.com` | MPS (Model Proxy Service) — auto-generated API keys, Dograh-hosted LLM/STT/TTS | ⚠️ Use local Speaches + Ollama/vLLM |
| `us.i.posthog.com` | Telemetry (opt-out via `ENABLE_TELEMETRY=false`) | ✅ Disable |
| `cloud.langfuse.com` | Tracing (optional) | ✅ Self-host Langfuse or disable |
| Telephony providers | PSTN calls | ⚠️ Use local VoIP/Asterisk |

### 3.4 Environment Variables (Critical)

```bash
# Core Infrastructure
DATABASE_URL=postgresql+asyncpg://postgres:postgres@postgres:5432/postgres
REDIS_URL=redis://:redissecret@redis:6379

# Storage
ENABLE_AWS_S3=false
MINIO_ENDPOINT=minio:9000
MINIO_PUBLIC_ENDPOINT=http://localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_BUCKET=voice-audio

# Deployment Mode
DEPLOYMENT_MODE=oss              # "oss" = self-hosted, no Dograh cloud
ENABLE_TELEMETRY=false           # Disable PostHog tracking

# MPS (Model Proxy Service) - for Dograh-hosted models
MPS_API_URL=https://services.dograh.com  # Can be swapped for local
DOGRAH_MPS_SECRET_KEY=              # Leave empty for pure OSS

# TURN Server (WebRTC)
TURN_HOST=localhost
TURN_SECRET=change-in-production

# Auth (OSS mode)
OSS_JWT_SECRET=ChangeMeInProduction
```

### 3.5 Self-Hosting Feasibility

| Capability | Status | Complexity |
|------------|--------|------------|
| **Fully offline** | ⚠️ Possible | High — must configure all local providers |
| **No cloud APIs** | ⚠️ Possible | High — replace all cloud STT/TTS/LLM |
| **No Dograh account** | ✅ Yes | Set `DEPLOYMENT_MODE=oss` |
| **No telemetry** | ✅ Yes | Set `ENABLE_TELEMETRY=false` |
| **Local-only audio** | ✅ Yes | All audio stored in MinIO (configurable) |

---

## 4. HGI Compatibility Audit

### 4.1 Local-First Assessment

| Criteria | Score | Notes |
|----------|-------|-------|
| **Offline capability** | 3/10 | Requires significant reconfiguration |
| **Local data storage** | 9/10 | MinIO/PostgreSQL — fully local |
| **BYOK/BYOM support** | 10/10 | Excellent provider registry |
| **No forced cloud** | 6/10 | Default config routes to Dograh MPS |
| **Self-contained** | 4/10 | Heavy Python dependencies |

### 4.2 Local STT/TTS/LLM Configuration

**To achieve full local operation:**

```python
# Local STT: Speaches (faster-whisper)
STT_PROVIDER=speaches
STT_BASE_URL=http://localhost:8000/v1
STT_MODEL=Systran/faster-whisper-small.en

# Local TTS: Speaches (Kokoro)
TTS_PROVIDER=speaches
TTS_BASE_URL=http://localhost:8000/v1
TTS_MODEL=hexgrad/Kokoro-82M
TTS_VOICE=af_heart

# Local LLM: Ollama or vLLM
LLM_PROVIDER=speaches
LLM_BASE_URL=http://localhost:11434/v1
LLM_MODEL=llama3
```

**Required Additional Services:**
1. **Speaches** (STT+TTS) — `github.com/speaches-ai/speaches`
2. **Ollama** or **vLLM** (LLM) — local inference
3. **CoTURN** — already included for WebRTC

### 4.3 HGI-EDGE-RUNTIME Integration

| Integration Point | Status | Notes |
|-------------------|--------|-------|
| **Native HGI-EDGE support** | ❌ None | No built-in adapter |
| **Webhook bridge** | ⚠️ Possible | Custom tool node → HGI-EDGE |
| **gRPC bridge** | ⚠️ Possible | Requires custom proxy |
| **Event streaming** | ⚠️ Possible | WebSocket → SSE/MCP |

**Implementation Approach:**
```python
# Custom tool in Dograh workflow that calls HGI-EDGE
class HGIEDGEToolManager:
    async def call_hgi_edge(self, params: dict) -> dict:
        # Transform Dograh tool call to HGI-EDGE format
        hgi_request = self.transform_to_hgi(params)
        # Async call to HGI-EDGE runtime
        response = await self.hgi_client.execute(hgi_request)
        return self.transform_from_hgi(response)
```

### 4.4 hgi-local-node Integration

| Integration Point | Status | Notes |
|-------------------|--------|-------|
| **Direct integration** | ❌ None | No native support |
| **WebSocket events** | ⚠️ Possible | Event handler bridge |
| **Pub/Sub** | ⚠️ Possible | Redis adapter |

### 4.5 Privacy & Data Retention

| Aspect | Status | Notes |
|--------|--------|-------|
| **Audio retention** | ⚠️ Configurable | Stored in MinIO by default |
| **Call recordings** | ⚠️ Configurable | Can disable workflow recordings |
| **Transcript storage** | ✅ Local | PostgreSQL |
| **Telemetry** | ✅ Opt-out | `ENABLE_TELEMETRY=false` |
| **Model data sharing** | ⚠️ Provider-dependent | Use local models to avoid |

### 4.6 BYOK/BYOM (Bring Your Own Key/Model)

| Provider Type | BYOK Support | BYOM Support | Notes |
|---------------|--------------|--------------|-------|
| **Cloud APIs** | ✅ Yes | ❌ No | OpenAI, Anthropic, etc. |
| **Self-hosted** | ✅ N/A | ✅ Yes | Ollama, vLLM, Speaches |
| **Hybrid** | ✅ Yes | ⚠️ Partial | Groq, OpenRouter |

---

## 5. Architecture Patterns for HGI

### 5.1 STT → LLM → TTS Streaming Pipeline

**Pipecat Pattern Analysis:**

```python
# From api/services/pipecat/pipeline_builder.py
class PipelineBuilder:
    """Builds voice pipeline with frame-based streaming"""
    
    def build_pipeline():
        # 1. Transport (WebRTC/Telephony) → Audio frames
        # 2. VAD (Voice Activity Detection) → Speech segments
        # 3. STT Service → Transcription frames
        # 4. User Aggregator → Complete utterances
        # 5. LLM Service → Response frames (streaming)
        # 6. TTS Service → Audio frames (streaming)
        # 7. Transport → Playback
```

**Key Components:**
1. **Frame-based architecture** — Each component emits typed frames
2. **Async streaming** — TTS starts before LLM completes
3. **Interruption handling** — VAD detects barge-in, cancels TTS
4. **Smart turn detection** — Determines when user finished speaking

### 5.2 Interruption Handling

```python
# From run_pipeline.py
UserTurnStrategies(
    start=[VADUserTurnStartStrategy()],  # Detect user started
    stop=[TurnAnalyzerUserTurnStopStrategy()],  # Detect user finished
)

# When interruption detected:
# 1. CancelFrame sent to TTS
# 2. BotStoppedSpeakingFrame emitted
# 3. New user audio queued to STT
```

### 5.3 Workflow Builder Architecture

**Node Types** (from `api/services/workflow/node_specs/`):
- `start_call.py` — Entry point, greeting
- `agent.py` — LLM agent with tools
- `trigger.py` — Conditional branching
- `webhook.py` — External HTTP calls
- `end_call.py` — Termination, disposition
- `global_node.py` — Shared behaviors
- `qa.py` — Quality analysis

**Workflow Graph:**
```python
class WorkflowGraph:
    nodes: dict[str, Node]
    edges: list[Edge]
    
    # Nodes connected by LLM function calls
    # Each agent node registers "transition_to_X" tools
    # LLM decides next node via function call
```

### 5.4 Call Routing & Telephony

**Provider Registry Pattern:**
```python
# From api/services/telephony/registry.py
TELEPHONY_REGISTRY = {
    "twilio": TwilioSpec(...),
    "vonage": VonageSpec(...),
    "telnyx": TelnyxSpec(...),
    "plivo": PlivoSpec(...),
    "vobiz": VobizSpec(...),
    "cloudonix": CloudonixSpec(...),
    "ari": ArISpec(...),  # Asterisk
}
```

**Transport Abstraction:**
- All providers implement same WebSocket interface
- Audio format normalized per-provider
- Unified callback handling

### 5.5 Latency Handling

**Techniques Observed:**
1. **Early TTS** — Start speaking before LLM completes
2. **Streaming JSON parser** — Handle partial LLM responses
3. **Silence detection** — VAD-based turn detection
4. **Connection pooling** — HTTP/2 for cloud APIs
5. **Local VAD** — SileroVADAnalyzer for fast detection

### 5.6 Agent State Management

```python
# From api/services/workflow/pipecat_engine.py
class PipecatEngine:
    _call_context_vars: dict        # Session variables
    _current_node: Node             # Current workflow position
    _gathered_context: dict         # Collected data
    _variable_extraction_manager    # Async extraction
    _custom_tool_manager             # Tool execution
```

---

## 6. Risk Assessment

### 6.1 Vendor Lock-in Risk

| Aspect | Risk Level | Mitigation |
|--------|------------|------------|
| **Pipecat framework** | Medium | Open source but Dograh has custom fork |
| **Dograh MPS** | High | Can swap for local alternatives |
| **Telephony providers** | Low | Multiple options, standardized |
| **Database schema** | Low | SQLAlchemy, can migrate |
| **Cloud providers** | Low | BYOK design |

### 6.2 Telemetry Risk

| Source | Data Type | Opt-out | Risk |
|--------|-----------|---------|------|
| **PostHog** | Usage analytics, feature adoption | ✅ `ENABLE_TELEMETRY=false` | Low when disabled |
| **Langfuse** | LLM traces, prompts | ✅ Optional config | Low when disabled |
| **Sentry** | Error reports | ✅ Optional DSN | Low when disabled |
| **MPS** | API calls (if used) | ⚠️ Use local providers | Medium if enabled |

**Telemetry Code Locations:**
- `api/services/posthog_client.py` — Analytics
- `api/services/pipecat/tracing_config.py` — Langfuse
- `docker-compose.yaml:180` — PostHog key hardcoded (but opt-out works)

### 6.3 Docker Complexity

| Concern | Assessment |
|---------|------------|
| **Service count** | 8+ services in production |
| **Resource requirements** | High — Python + Node.js + PostgreSQL + Redis |
| **Startup time** | 2-3 minutes initial pull |
| **Configuration surface** | Large — 20+ env vars |
| **Debugging** | Complex — distributed logs |

### 6.4 Python-Heavy Complexity

| Aspect | Impact |
|--------|--------|
| **Dependency count** | 22+ direct dependencies |
| **Async complexity** | Heavy use of asyncio, requires expertise |
| **Pipecat abstraction** | Custom framework, learning curve |
| **Type hints** | Good coverage, helps maintenance |
| **Testing** | pytest with fixtures, well-structured |

### 6.5 Latency Risks

| Source | Impact | Mitigation |
|--------|--------|------------|
| **Cloud STT round-trip** | 100-300ms | Local Speaches |
| **Cloud TTS round-trip** | 100-500ms | Local Kokoro |
| **Cloud LLM inference** | 500-2000ms | Local LLM or edge caching |
| **Frame pipeline overhead** | 20-50ms | Optimized in Pipecat |
| **WebRTC NAT traversal** | Variable | CoTURN relay |

### 6.6 Security Risks

| Aspect | Risk | Mitigation |
|--------|------|------------|
| **API keys in database** | Medium | Encrypted at rest |
| **JWT secret** | High (if default) | Must change `OSS_JWT_SECRET` |
| **WebSocket auth** | Medium | Token-based, needs review |
| **Webhook verification** | Medium | HMAC signatures |
| **TURN credentials** | Medium | Time-limited, REST API |

### 6.7 Privacy Risks

| Aspect | Risk | Mitigation |
|--------|------|------------|
| **Audio storage** | High (if retained) | Configure retention, encrypt |
| **Transcript logs** | Medium | Access controls |
| **Call metadata** | Medium | Minimize PII collection |
| **Third-party STT/TTS** | High | Audio leaves premises |
| **Model training** | Unknown | Provider terms vary |

### 6.8 Cloud/Provider Dependency

| Service | Dependency Level | Local Alternative |
|---------|------------------|-------------------|
| **Core platform** | None | Fully self-hosted |
| **STT** | Medium | Speaches, Whisper |
| **TTS** | Medium | Speaches, Kokoro, Piper |
| **LLM** | Medium | Ollama, vLLM, llama.cpp |
| **Telephony** | High | Asterisk/FreePBX |
| **Push notifications** | Low | WebSockets |

### 6.9 Operational Cost

| Resource | Estimated Cost (Self-Hosted) |
|----------|------------------------------|
| **Compute** | 2-4 vCPU, 4-8GB RAM minimum |
| **GPU (local LLM)** | Optional but recommended |
| **Storage** | 10GB+ for audio recordings |
| **Network** | Bandwidth for WebRTC/TURN |
| **Maintenance** | High — complex stack |

---

## 7. Recommendation

### 7.1 Decision: USE AS INSPIRATION ONLY

**Rationale:**

1. **Architecture is solid** — Pipecat streaming pipeline, frame-based design, and workflow builder are excellent patterns
2. **Deployment complexity too high** — 8+ Docker services, heavy Python stack
3. **Not truly local-first by default** — Requires significant reconfiguration
4. **HGI integration requires bridge** — No native HGI-EDGE or hgi-local-node support
5. **Python-heavy stack** — HGI core is Rust/TypeScript, mismatch in expertise

### 7.2 Patterns to Adopt

| Pattern | Implementation Approach | Priority |
|---------|------------------------|----------|
| **Streaming STT→LLM→TTS** | Implement in HGI-EDGE-RUNTIME | High |
| **Frame-based architecture** | Use with HGI's event system | High |
| **VAD-based interruptions** | Integrate with WebRTC VAD | Medium |
| **Workflow builder (visual)** | Adapt for HGI config system | Medium |
| **Node-based call flows** | YAML/JSON declarative format | Medium |
| **Multi-telephony support** | Abstract transport layer | Low |

### 7.3 Patterns to Avoid

| Pattern | Reason | Alternative |
|---------|--------|-------------|
| **Heavy Python backend** | HGI stack is Rust/TS | Keep HGI-EDGE-RUNTIME in Rust |
| **PostgreSQL + Redis required** | Operational burden | SQLite for single-node, NATS for distributed |
| **MinIO for audio** | Overkill for voice | Direct streaming, optional S3 |
| **Docker-first deployment** | Complex for edge | Single binary or WASM |
| **MPS for model proxy** | External dependency | Direct model connections |

### 7.4 Recommended Next Steps

1. **Do NOT adopt Dograh as dependency**
2. **Document patterns** in HGI architecture docs
3. **Design HGI-VOICE-BRIDGE** with lessons learned
4. **Prototype streaming pipeline** in HGI-EDGE-RUNTIME
5. **Evaluate Speaches** for STT/TTS component
6. **Test local LLM integration** with Ollama/vLLM

---

## 8. Appendix

### 8.1 External Documentation

- Dograh Docs: https://docs.dograh.com/deployment/docker
- Pipecat Framework: https://github.com/pipecat-ai/pipecat
- Speaches (local STT/TTS): https://github.com/speaches-ai/speaches

### 8.2 Audit Methodology

1. Repository structure analysis
2. License file review
3. Docker Compose inspection
4. Environment variable analysis
5. Code review of core services
6. Telemetry and privacy audit
7. Integration capability assessment

### 8.3 Key Files Referenced

- `LICENSE` — BSD-2-Clause
- `docker-compose.yaml` — Production deployment
- `api/constants.py` — Core configuration
- `api/services/configuration/registry.py` — Provider registry
- `api/services/pipecat/service_factory.py` — Service instantiation
- `api/services/mps_service_key_client.py` — Dograh MPS client
- `api/services/posthog_client.py` — Telemetry
- `api/services/workflow/pipecat_engine.py` — Workflow engine

---

*End of Audit Report*
