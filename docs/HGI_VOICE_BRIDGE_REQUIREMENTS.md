# HGI Voice Bridge Requirements Specification

**Version:** 1.0  
**Date:** May 19, 2026  
**Status:** Draft — Based on Dograh Audit  
**Target Systems:** MOLIE, EVA, MarshantaMX, RedVecinalMX

---

## 1. Vision

HGI Voice Bridge is a **local-first, privacy-preserving** voice AI infrastructure that enables natural, real-time conversations between humans and AI agents. It serves as the voice layer for HGI applications while maintaining data sovereignty and supporting offline operation.

### 1.1 Core Principles

| Principle | Description |
|-----------|-------------|
| **Local-First** | All audio processing stays on-device or HGI-local-node by default |
| **Privacy by Design** | No raw audio retention; opt-in only for improvements |
| **Cloud-Optional** | Works fully offline; cloud providers are BYOK opt-in |
| **HGI-Native** | Deep integration with HGI-EDGE-RUNTIME and hgi-local-node |
| **Open Standards** | WebRTC, WebSocket, gRPC — no proprietary protocols |
| **Lightweight** | Single binary deployment; minimal dependencies |

---

## 2. Functional Requirements

### 2.1 Audio Pipeline

#### 2.1.1 STT (Speech-to-Text)

| Requirement | Priority | Notes |
|-------------|----------|-------|
| Streaming transcription | P0 | Real-time, word-by-word |
| Local Whisper/Faster-Whisper | P0 | Default: `Systran/faster-distil-whisper-small.en` |
| VAD integration | P0 | Silero VAD or WebRTC VAD |
| Language detection | P1 | Auto-detect or configured |
| Custom vocabulary | P2 | Keyterm boosting for domain terms |
| Cloud provider fallback | P2 | Deepgram, OpenAI (BYOK) |
| Offline models | P0 | Full functionality without internet |

#### 2.1.2 LLM (Language Model)

| Requirement | Priority | Notes |
|-------------|----------|-------|
| Streaming responses | P0 | Token-by-token for low latency |
| Local LLM support | P0 | Ollama, llama.cpp, vLLM |
| Function/tool calling | P0 | For workflow transitions |
| Context management | P0 | Sliding window, summarization |
| Cloud provider fallback | P2 | OpenAI, Anthropic (BYOK) |
| HGI-EDGE integration | P0 | Direct gRPC to HGI-EDGE-RUNTIME |
| System prompt templating | P1 | Mustache/Handlebars style |

#### 2.1.3 TTS (Text-to-Speech)

| Requirement | Priority | Notes |
|-------------|----------|-------|
| Streaming synthesis | P0 | First audio chunk < 200ms |
| Local Kokoro | P0 | Default: `hexgrad/Kokoro-82M` |
| Piper TTS support | P1 | Lightweight alternative |
| Voice selection | P1 | Multiple voices per language |
| Speed control | P2 | 0.5x - 2.0x |
| SSML support | P2 | Basic prosody |
| Cloud provider fallback | P3 | ElevenLabs, Cartesia (BYOK) |
| Interruptible playback | P0 | Barge-in support |

### 2.2 Streaming Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Microphone │────▶│     VAD     │────▶│     STT     │────▶│     LLM     │
│  (WebRTC)   │     │(Silero/WebRTC)│    │  (Local)    │     │  (Local)    │
└─────────────┘     └─────────────┘     └─────────────┘     └──────┬──────┘
                                                                    │
                                                                    ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Speaker   │◀────│ Audio Mixer │◀────│     TTS     │◀────│  Response   │
│  (WebRTC)   │     │ (interrupt) │     │  (Local)    │     │   Stream    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

**Key Requirements:**
- **P0:** Frame-based streaming with typed events
- **P0:** Interruption handling (barge-in detection)
- **P0:** Backpressure handling for slow consumers
- **P1:** Smart turn detection (end-of-turn)
- **P2:** Filler word injection for natural pauses

### 2.3 Conversation Management

#### 2.3.1 Session State

| State | Description |
|-------|-------------|
| `idle` | Waiting for user to speak |
| `listening` | VAD detected speech, STT active |
| `thinking` | STT complete, LLM processing |
| `speaking` | TTS streaming audio |
| `interrupted` | User barge-in during speaking |
| `ended` | Conversation terminated |

#### 2.3.2 Context Management

| Requirement | Priority | Description |
|-------------|----------|-------------|
| Conversation history | P0 | Rolling window of messages |
| Variable extraction | P1 | Slot filling from conversation |
| Summarization | P2 | Compress long conversations |
| Knowledge base RAG | P2 | Document retrieval |
| Multi-turn goals | P1 | Track user intent across turns |

### 2.4 Workflow Engine

#### 2.4.1 Node Types

| Node | Purpose | Priority |
|------|---------|----------|
| `start` | Entry point, greeting | P0 |
| `agent` | LLM-powered conversation | P0 |
| `condition` | Rule-based branching | P1 |
| `action` | Execute tool/HGI-EDGE call | P0 |
| `transfer` | Handoff to human/system | P2 |
| `end` | Termination, disposition | P0 |

#### 2.4.2 Workflow Format

```yaml
# hgi-voice-workflow.yaml
version: "1.0"
workflow:
  name: "marshanta_ordering"
  nodes:
    start:
      type: start
      greeting: "Hola, soy su asistente de pedidos. ¿En qué puedo ayudarle?"
      next: main_agent
    
    main_agent:
      type: agent
      system_prompt: |
        Eres un asistente para tomar pedidos de abarrotes.
        Productos disponibles: {{products}}
        Variables a extraer: producto, cantidad, dirección.
      tools:
        - name: check_inventory
          endpoint: "hgi-edge://inventory/check"
        - name: place_order
          endpoint: "hgi-edge://orders/create"
      transitions:
        - condition: "intent == 'complete_order'"
          target: confirm_order
        - condition: "intent == 'escalate'"
          target: transfer_human
    
    confirm_order:
      type: agent
      system_prompt: "Confirme el pedido: {{order_summary}}"
      transitions:
        - condition: "confirmed"
          target: place_order_action
    
    place_order_action:
      type: action
      action: place_order
      next: end_success
    
    end_success:
      type: end
      disposition: "order_placed"
```

### 2.5 Transport Layer

#### 2.5.1 WebRTC (Browser/App)

| Requirement | Priority |
|-------------|----------|
| PeerConnection management | P0 |
| Opus codec support | P0 |
| Adaptive bitrate | P2 |
| Echo cancellation | P0 |
| Noise suppression | P1 |
| TURN server support | P1 |

#### 2.5.2 Telephony (Optional)

| Provider | Priority | Notes |
|----------|----------|-------|
| Twilio | P2 | PSTN connectivity |
| SIP Trunk | P2 | Direct PBX integration |
| WebRTC→PSTN gateway | P3 | Asterisk/FreePBX |

---

## 3. Integration Requirements

### 3.1 HGI-EDGE-RUNTIME Integration

#### 3.1.1 gRPC Service Definition

```protobuf
// hgi_voice_bridge.proto
syntax = "proto3";
package hgi.voice;

service VoiceBridge {
  // Bidirectional streaming for full conversation
  rpc Conversate(stream ConversationEvent) returns (stream ConversationEvent);
  
  // One-shot commands
  rpc StartConversation(StartRequest) returns (ConversationState);
  rpc EndConversation(EndRequest) returns (EndResponse);
  rpc UpdateContext(ContextUpdate) returns (Ack);
  
  // Tool execution from workflow
  rpc ExecuteTool(ToolRequest) returns (ToolResponse);
}

message ConversationEvent {
  oneof payload {
    AudioFrame audio_in = 1;      // From user
    AudioFrame audio_out = 2;     // To user
    Transcript transcript = 3;    // STT result
    LLMToken llm_token = 4;       // Streaming response
    StateChange state = 5;        // State transition
    ToolCall tool_call = 6;       // Tool invocation
  }
  uint64 timestamp_ms = 100;
  string conversation_id = 101;
}
```

#### 3.1.2 HGI-EDGE Event Mapping

| HGI-EDGE Event | Voice Bridge Event | Direction |
|----------------|-------------------|-----------|
| `ToolInvocation` | `ToolCall` | VBridge → HGI-EDGE |
| `ToolResult` | `ToolResponse` | HGI-EDGE → VBridge |
| `ContextUpdate` | `ContextUpdate` | Bidirectional |
| `UserEvent` | `StateChange` | VBridge → HGI-EDGE |

### 3.2 hgi-local-node Integration

#### 3.2.1 Event Bus Connection

| Channel | Purpose | Priority |
|---------|---------|----------|
| `voice.audio.in` | Raw audio from input device | P0 |
| `voice.audio.out` | Synthesized audio to output | P0 |
| `voice.transcript` | STT output | P0 |
| `voice.llm.stream` | LLM token stream | P0 |
| `voice.state` | State machine updates | P1 |
| `voice.tools` | Tool call requests | P0 |

#### 3.2.2 Local Service Discovery

```json
{
  "service": "hgi-voice-bridge",
  "version": "1.0",
  "endpoints": {
    "webrtc": "ws://localhost:8080/webrtc",
    "grpc": "grpc://localhost:50051",
    "health": "http://localhost:8080/health"
  },
  "capabilities": ["stt", "tts", "llm", "webrtc"],
  "models": {
    "stt": ["faster-whisper-small.en"],
    "tts": ["kokoro-82M"],
    "llm": ["llama3.2-3b"]
  }
}
```

### 3.3 EVA Integration

#### 3.3.1 EVA-Specific Requirements

| Requirement | Description |
|-------------|-------------|
| Voice print auth | Biometric speaker identification |
| Emotion detection | Prosody analysis for sentiment |
| Proactive prompts | EVA-initiated notifications |
| Multi-language | Spanish, English, Mixtec |

### 3.4 MOLIE Integration

#### 3.4.1 MOLIE-Specific Requirements

| Requirement | Description |
|-------------|-------------|
| Conversation memory | Long-term user preference storage |
| Interruption sensitivity | Quick barge-in for corrections |
| Multiple voices | Voice per persona/context |
| Whisper mode | Low-volume speech detection |

### 3.5 MarshantaMX Integration

#### 3.5.1 Ordering-Specific Requirements

| Requirement | Description |
|-------------|-------------|
| Product catalog speech | "Quiero 5 kilos de arroz" |
| Price confirmation | TTS price quotes |
| Address confirmation | Speech-to-address parsing |
| Order summary | TTS order recap |

### 3.6 RedVecinalMX Integration

#### 3.6.1 Emergency-Specific Requirements

| Requirement | Description |
|-------------|-------------|
| Priority escalation | Emergency keywords trigger alert |
| Location reporting | Speech-to-location extraction |
| Multi-party | Conference bridge support |
| Fallback SMS | Text backup if voice fails |

---

## 4. Non-Functional Requirements

### 4.1 Performance

| Metric | Target | Measurement |
|--------|--------|-------------|
| First audio chunk latency | < 300ms | TTFB after user stops |
| STT latency | < 200ms | Audio to transcript |
| LLM time-to-first-token | < 500ms | Local LLM |
| End-to-end response | < 1.5s | User speech → bot speech |
| Concurrent calls | 10+ | Per hgi-local-node |
| Memory footprint | < 2GB | Base + models loaded |
| CPU usage | < 50% | 4-core, no GPU |

### 4.2 Reliability

| Requirement | Target |
|-------------|--------|
| Uptime | 99.9% |
| Graceful degradation | Cloud failure → local models |
| Auto-restart | Watchdog for core services |
| State recovery | Resume conversation after crash |
| Circuit breaker | Fail fast on provider errors |

### 4.3 Security

| Requirement | Implementation |
|-------------|----------------|
| Audio encryption | TLS 1.3 for transport |
| No raw audio storage | Ephemeral processing only |
| Transcript encryption | At-rest encryption |
| API key isolation | Per-organization vault |
| WebSocket auth | JWT with short expiry |
| Rate limiting | Per-user, per-endpoint |

### 4.4 Privacy

| Requirement | Default |
|-------------|---------|
| Raw audio retention | **NEVER** |
| Transcript retention | 24 hours (configurable) |
| Model improvement data | Opt-in only |
| Telemetry | Disabled by default |
| Local processing | **ALWAYS** for sensitive data |

### 4.5 Operational

| Requirement | Target |
|-------------|--------|
| Deployment | Single binary or Docker |
| Configuration | YAML/JSON files |
| Health checks | HTTP + gRPC |
| Metrics | OpenTelemetry compatible |
| Logs | Structured JSON |
| Updates | Rolling, zero-downtime |

---

## 5. BYOK/BYOM Support

### 5.1 Bring Your Own Key

| Provider | Configuration | Use Case |
|----------|---------------|----------|
| OpenAI | `OPENAI_API_KEY` | Better quality when needed |
| Deepgram | `DEEPGRAM_API_KEY` | Superior STT |
| ElevenLabs | `ELEVENLABS_API_KEY` | Premium voices |
| Groq | `GROQ_API_KEY` | Fast cloud LLM |

### 5.2 Bring Your Own Model

| Model Type | Local Endpoint | Configuration |
|------------|----------------|---------------|
| Whisper | `http://localhost:8000/v1/audio/transcriptions` | `STT_ENDPOINT` |
| Kokoro | `http://localhost:8000/v1/audio/speech` | `TTS_ENDPOINT` |
| Ollama | `http://localhost:11434/v1/chat/completions` | `LLM_ENDPOINT` |
| vLLM | `http://localhost:8000/v1/chat/completions` | `LLM_ENDPOINT` |

### 5.3 Fallback Chain

```yaml
stt:
  primary: local-whisper
  fallback: deepgram  # If local fails or language not supported
  
tts:
  primary: local-kokoro
  fallback: piper  # If voice not available
  
llm:
  primary: local-ollama
  fallback: groq  # If context too large for local
```

---

## 6. Deployment Architecture

### 6.1 Single-Node (hgi-local-node)

```
┌─────────────────────────────────────────────────────────────────┐
│                    hgi-local-node (single machine)              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Voice Agent │  │  Voice Agent │  │  Voice Agent │  ...      │
│  │   (WebRTC)   │  │   (WebRTC)   │  │  (Telephony) │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                  │                  │                  │
│  ┌──────┴──────────────────┴──────────────────┴──────┐          │
│  │           HGI Voice Bridge Core                   │          │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │          │
│  │  │   STT   │ │   LLM   │ │   TTS   │ │ Workflow│  │          │
│  │  │ Service │ │ Service │ │ Service │ │ Engine  │  │          │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘  │          │
│  └──────────────────┬─────────────────────────────────┘          │
│                     │                                            │
│         ┌───────────┴───────────┐                               │
│         ▼                       ▼                               │
│  ┌─────────────┐        ┌─────────────┐                        │
│  │  Ollama/    │        │   SQLite    │                        │
│  │  llama.cpp  │        │   (state)   │                        │
│  └─────────────┘        └─────────────┘                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Distributed (HGI Cloud)

```
┌─────────────────────────────────────────────────────────────────┐
│                         HGI Cloud                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Edge Node   │  │  Edge Node   │  │  Edge Node   │          │
│  │  (Region 1)  │  │  (Region 2)  │  │  (Region N)  │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                  │
│         └──────────────────┼──────────────────┘                  │
│                            │                                    │
│  ┌─────────────────────────┴─────────────────────────────┐        │
│  │              Voice Bridge Controller                 │        │
│  │         (workflow distribution, metrics)           │        │
│  └─────────────────────────────────────────────────────┘        │
│                            │                                    │
│  ┌─────────────────────────┴─────────────────────────────┐        │
│  │              Shared Model Cache (Redis)               │        │
│  │    (preloaded embeddings, voice profiles)              │        │
│  └─────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Configuration Schema

### 7.1 Global Configuration

```yaml
# config.yaml
voice_bridge:
  version: "1.0"
  
  server:
    bind_address: "0.0.0.0:8080"
    tls:
      cert_file: "/etc/hgi/server.crt"
      key_file: "/etc/hgi/server.key"
  
  models:
    stt:
      provider: local
      model: "Systran/faster-distil-whisper-small.en"
      device: auto  # cpu, cuda, mps, auto
      
    tts:
      provider: local
      model: "hexgrad/Kokoro-82M"
      voice: "af_heart"
      speed: 1.0
      
    llm:
      provider: local
      endpoint: "http://localhost:11434/v1"
      model: "llama3.2:3b"
      temperature: 0.7
      max_tokens: 1024
  
  webrtc:
    ice_servers:
      - urls: ["stun:stun.l.google.com:19302"]
      - urls: ["turn:turn.hgi.local:3478"]
        username: "hgi"
        credential: "${TURN_PASSWORD}"
    
  privacy:
    retain_audio: false
    retain_transcripts: 24h
    telemetry: false
    
  hgi_integration:
    edge_runtime:
      endpoint: "grpc://localhost:50051"
      timeout_ms: 5000
    local_node:
      event_bus: "nats://localhost:4222"
```

### 7.2 Workflow Configuration

See section 2.4.2 for YAML workflow format.

---

## 8. Success Criteria

### 8.1 MVP (Month 1-2)

- [ ] Local STT (Whisper) working end-to-end
- [ ] Local TTS (Kokoro) with streaming
- [ ] Local LLM (Ollama) integration
- [ ] WebRTC browser demo
- [ ] Basic workflow engine (start → agent → end)

### 8.2 Beta (Month 3-4)

- [ ] Interruption handling
- [ ] HGI-EDGE-RUNTIME gRPC integration
- [ ] hgi-local-node event bus
- [ ] MarshantaMX ordering workflow
- [ ] BYOK cloud provider fallback

### 8.3 Production (Month 5-6)

- [ ] RedVecinalMX emergency integration
- [ ] EVA voice print support
- [ ] MOLIE conversation memory
- [ ] Telephony (Twilio/SIP)
- [ ] Performance targets met
- [ ] Security audit passed

---

## 9. Appendix

### 9.1 Reference Implementations

| Component | Reference | License |
|-----------|-----------|---------|
| Streaming STT | Speaches | Apache 2.0 |
| Streaming TTS | Kokoro-FastAPI | MIT |
| WebRTC transport | Pipecat small-webrtc | BSD-2 |
| VAD | Silero-VAD | Apache 2.0 |
| LLM streaming | Ollama | MIT |

### 9.2 Related Documents

- [HGI_DOGRAH_AUDIT.md](./HGI_DOGRAH_AUDIT.md) — Architecture lessons from Dograh
- [HGI-EDGE-RUNTIME Spec](../hgi-edge-runtime/SPEC.md) — Edge runtime integration
- [hgi-local-node Events](../hgi-local-node/EVENTS.md) — Event bus protocol

### 9.3 Glossary

| Term | Definition |
|------|------------|
| **VAD** | Voice Activity Detection |
| **STT** | Speech-to-Text |
| **TTS** | Text-to-Speech |
| **LLM** | Large Language Model |
| **BYOK** | Bring Your Own Key |
| **BYOM** | Bring Your Own Model |
| **TTFB** | Time to First Byte (audio) |
| **Barge-in** | User interrupting bot speech |
| **Turn** | Single user utterance + bot response |

---

*End of Requirements Specification*
