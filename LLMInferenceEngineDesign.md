# AI Assistant System Design
### vLLM-Powered, Cloud-Native, Multi-Tenant Architecture

**Document Type:** Architecture Design Document (ADD)  
**Author:** Software Architect  
**Version:** 1.0  
**Status:** Draft

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architectural Goals & Constraints](#2-architectural-goals--constraints)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Component Deep Dive](#4-component-deep-dive)
5. [Low-Level Design](#5-low-level-design)
6. [Data Architecture](#6-data-architecture)
7. [Security Architecture](#7-security-architecture)
8. [Scalability & Performance Strategy](#8-scalability--performance-strategy)
9. [Deployment Architecture](#9-deployment-architecture)
10. [Observability & Operations](#10-observability--operations)
11. [Architecture Decision Records (ADRs)](#11-architecture-decision-records-adrs)
12. [Risk Register](#12-risk-register)

---

## 1. Executive Summary

This document describes the end-to-end architecture of a cloud-hosted AI Assistant powered by **vLLM** as the LLM inference engine. The system is designed to serve **multiple concurrent users** with low latency, high availability, and cost-efficient GPU utilization.

The architecture follows a **layered, API-first** design with clear separation of concerns across: client-facing API gateway, request orchestration, inference serving, and persistence.

---

## 2. Architectural Goals & Constraints

### 2.1 Functional Requirements

- Accept conversational natural language queries from users via REST and WebSocket APIs
- Maintain per-user conversation history (multi-turn dialogue)
- Support streaming token responses for real-time UX
- Support multiple LLM models (model routing)
- Authenticate and authorize users

### 2.2 Non-Functional Requirements (NFRs)

| NFR | Target |
|-----|--------|
| Concurrent users | 1,000+ simultaneous sessions |
| Request throughput | 500+ requests/second at peak |
| Response latency (P95) | < 3s to first token (TTFT) |
| Availability | 99.9% SLA |
| GPU utilization | > 75% average |
| Token throughput | Maximize via vLLM continuous batching |
| Horizontal scalability | Auto-scale GPU nodes within 3 minutes |

### 2.3 Constraints

- Hosted entirely in public cloud (AWS primary; GCP as DR)
- vLLM as the mandatory inference framework
- No persistent PII storage beyond session lifetime (configurable)
- Cost-optimized using spot/preemptible GPU instances where safe

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLIENT TIER                                     │
│   Web App   │   Mobile App   │   API Consumers   │   CLI / SDK          │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │  HTTPS / WSS
┌──────────────────────────▼──────────────────────────────────────────────┐
│                        EDGE & GATEWAY TIER                               │
│                                                                          │
│   ┌────────────┐    ┌────────────────┐    ┌──────────────────────────┐  │
│   │  CDN /     │    │  API Gateway   │    │  WAF / DDoS Protection   │  │
│   │  CloudFront│    │  (Kong / APIGW)│    │  (AWS Shield / Cloudflare│  │
│   └────────────┘    └───────┬────────┘    └──────────────────────────┘  │
└──────────────────────────── │ ──────────────────────────────────────────┘
                              │  Internal mTLS
┌─────────────────────────────▼───────────────────────────────────────────┐
│                      APPLICATION TIER                                    │
│                                                                          │
│  ┌──────────────────────┐    ┌──────────────────────────────────────┐   │
│  │   Auth Service        │    │      Chat Orchestration Service      │   │
│  │  (JWT / OAuth2 / OIDC)│    │  (FastAPI / async Python)            │   │
│  └──────────────────────┘    │  - Request validation                 │   │
│                               │  - Conversation context assembly     │   │
│  ┌──────────────────────┐    │  - Prompt construction               │   │
│  │  Session Manager     │    │  - Response streaming                │   │
│  │  (Redis)             │    │  - Rate limiting enforcement         │   │
│  └──────────────────────┘    └────────────────┬─────────────────────┘   │
└───────────────────────────────────────────────│─────────────────────────┘
                                                │  HTTP / gRPC
┌───────────────────────────────────────────────▼─────────────────────────┐
│                      INFERENCE TIER                                      │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │               LLM Request Router / Load Balancer                 │   │
│  │          (Round-robin + least-pending-requests policy)           │   │
│  └────────┬─────────────────────┬──────────────────────────────────┘   │
│           │                     │                                        │
│  ┌────────▼───────┐    ┌────────▼───────┐    ┌──────────────────────┐  │
│  │  vLLM Worker 1 │    │  vLLM Worker 2 │    │  vLLM Worker N       │  │
│  │  (GPU Node)    │    │  (GPU Node)    │    │  (GPU Node)          │  │
│  │  A100/H100     │    │  A100/H100     │    │  (auto-scaled)       │  │
│  └────────────────┘    └────────────────┘    └──────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                                │
┌───────────────────────────────────────────────▼─────────────────────────┐
│                        DATA TIER                                         │
│                                                                          │
│  ┌────────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │
│  │  PostgreSQL    │  │  Redis       │  │  Object      │  │  Vector  │  │
│  │  (User data,   │  │  (Sessions,  │  │  Storage     │  │  DB      │  │
│  │  audit logs)   │  │  KV cache)   │  │  (Model      │  │(pgvector/│  │
│  │                │  │              │  │  artifacts)  │  │ Pinecone)│  │
│  └────────────────┘  └──────────────┘  └──────────────┘  └──────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                                │
┌───────────────────────────────────────────────▼─────────────────────────┐
│                   OBSERVABILITY & OPERATIONS TIER                        │
│   Prometheus │ Grafana │ OpenTelemetry │ ELK Stack │ PagerDuty           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Component Deep Dive

### 4.1 API Gateway

**Technology:** AWS API Gateway + Kong (self-managed for advanced routing)

**Responsibilities:**
- TLS termination
- API key validation (pre-auth check)
- Request/response logging
- Rate limiting (per user, per tier)
- Route traffic to Chat Orchestration Service
- WebSocket proxy for streaming responses

**Key Configurations:**
- WebSocket idle timeout: 30 minutes
- HTTP request timeout: 120 seconds (allows long LLM responses)
- Throttle limits: configurable per user plan (e.g., Free: 10 RPM, Pro: 100 RPM)

---

### 4.2 Auth Service

**Technology:** Keycloak or AWS Cognito

**Responsibilities:**
- OAuth 2.0 / OIDC identity provider integration
- JWT issuance and validation
- Role-based access control (RBAC): `user`, `admin`, `api_consumer`
- API key management for programmatic access
- Token refresh and revocation

**Token Structure (JWT Claims):**
```json
{
  "sub": "user-uuid",
  "email": "user@example.com",
  "roles": ["user"],
  "tier": "pro",
  "rate_limit_rpm": 100,
  "iat": 1700000000,
  "exp": 1700003600
}
```

---

### 4.3 Chat Orchestration Service

**Technology:** Python (FastAPI) with asyncio

This is the core application service. It coordinates all layers of the request lifecycle.

**Key Modules:**

| Module | Responsibility |
|--------|----------------|
| `RequestHandler` | Validates incoming requests, extracts JWT context |
| `ConversationManager` | Loads/stores conversation history from Redis |
| `PromptBuilder` | Constructs the final LLM prompt with system prompt + history + user message |
| `LLMClient` | Sends requests to vLLM OpenAI-compatible API, handles streaming |
| `ResponseStreamer` | Streams tokens back to client via SSE or WebSocket |
| `RateLimiter` | Enforces per-user token and request quotas using Redis |
| `ContentFilter` | Pre/post filters for safety (input toxicity, output guardrails) |

---

### 4.4 LLM Request Router

**Technology:** Custom Python service or NGINX with Lua scripting

**Responsibilities:**
- Maintains a registry of healthy vLLM worker endpoints
- Routes requests using **least-pending-requests** algorithm (minimizes queue depth per worker)
- Health checks vLLM workers every 5 seconds (`/health` endpoint)
- Drains workers gracefully before scale-down
- Supports model-based routing (e.g., route "fast" queries to a smaller model)

**Routing Policy:**
```
if pending_requests[worker_i] == min(all_workers):
    route → worker_i
else if all workers overloaded:
    queue request (max queue depth = 200, else 503)
```

---

### 4.5 vLLM Worker

**Technology:** vLLM (v0.4+) on GPU-backed instances (AWS `p4d.24xlarge` / `p3.8xlarge`)

**Key vLLM Features Leveraged:**

| Feature | Benefit |
|---------|---------|
| **Continuous Batching** | Dynamic batching of incoming requests — no static batch size, maximizes GPU utilization |
| **PagedAttention** | Efficient KV-cache memory management — dramatically reduces GPU memory waste |
| **OpenAI-compatible API** | `/v1/chat/completions` endpoint — easy integration with standard tooling |
| **Streaming** | Server-Sent Events (SSE) support for real-time token delivery |
| **Tensor Parallelism** | Splits large models (70B+) across multiple GPUs on the same node |
| **Async Engine** | Non-blocking request processing within the vLLM process |

**Startup Configuration Example:**
```bash
python -m vllm.entrypoints.openai.api_server \
  --model /models/llama-3-70b-instruct \
  --tensor-parallel-size 4 \
  --max-num-seqs 256 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90 \
  --enable-chunked-prefill \
  --host 0.0.0.0 \
  --port 8000
```

---

## 5. Low-Level Design

### 5.1 Request Lifecycle (Step-by-Step)

```
User Request (POST /v1/chat)
        │
        ▼
[1] API Gateway
    - Validate API key / Bearer token header present
    - Apply rate limit check (Tier 1: IP-based)
    - Forward to Chat Orchestration Service

        │
        ▼
[2] Auth Middleware (FastAPI)
    - Decode & verify JWT signature
    - Extract: user_id, tier, rate_limit_rpm
    - Inject user context into request state

        │
        ▼
[3] RateLimiter.check(user_id)
    - Redis INCR key: ratelimit:{user_id}:{minute_bucket}
    - If count > rpm_limit → return HTTP 429

        │
        ▼
[4] ConversationManager.load(session_id)
    - Redis GET: session:{session_id}
    - Returns: List[Message] (last N messages, truncated to context window)
    - If session not found → create new session

        │
        ▼
[5] PromptBuilder.build(system_prompt, history, user_message)
    - Prepend system prompt
    - Append conversation history (role: user/assistant pairs)
    - Append new user message
    - Apply token budget: trim history if total > max_context_tokens
    - Output: OpenAI-format messages array

        │
        ▼
[6] ContentFilter.check_input(user_message)
    - Run toxicity / jailbreak classifier (fast, lightweight model)
    - If flagged → return HTTP 400 with reason

        │
        ▼
[7] LLMClient.complete(messages, stream=True)
    - POST to LLM Router → routed to best vLLM worker
    - Request: { model, messages, temperature, max_tokens, stream: true }
    - Receive: SSE stream of token chunks

        │
        ▼
[8] ResponseStreamer.stream_to_client(token_stream)
    - Forward SSE events to client in real-time
    - Accumulate full response in memory

        │
        ▼
[9] ConversationManager.save(session_id, user_msg, assistant_msg)
    - Append user message + assistant full response to history
    - Redis SETEX: session:{session_id} TTL=3600 (1 hour)

        │
        ▼
[10] AuditLogger.log(user_id, session_id, input_tokens, output_tokens, latency_ms)
    - Async write to PostgreSQL audit_logs table
    - Emit metrics to Prometheus (token counts, latency, model used)
```

---

### 5.2 Class / Module Design

#### 5.2.1 Chat Orchestration Service — Core Classes

```python
# Domain Models
@dataclass
class Message:
    role: Literal["system", "user", "assistant"]
    content: str
    timestamp: datetime

@dataclass
class ChatRequest:
    session_id: str
    user_message: str
    stream: bool = True
    model: str = "default"
    max_tokens: int = 1024
    temperature: float = 0.7

@dataclass
class UserContext:
    user_id: str
    tier: str
    rate_limit_rpm: int
    roles: List[str]

# Service Interfaces
class IConversationStore(ABC):
    async def load(self, session_id: str) -> List[Message]: ...
    async def save(self, session_id: str, messages: List[Message]) -> None: ...

class ILLMClient(ABC):
    async def complete(self, messages: List[Message], **kwargs) -> AsyncIterator[str]: ...

class IContentFilter(ABC):
    async def check_input(self, text: str) -> FilterResult: ...
    async def check_output(self, text: str) -> FilterResult: ...

# Core Orchestrator
class ChatOrchestrator:
    def __init__(
        self,
        conversation_store: IConversationStore,
        llm_client: ILLMClient,
        prompt_builder: PromptBuilder,
        rate_limiter: RateLimiter,
        content_filter: IContentFilter,
        audit_logger: AuditLogger,
    ): ...

    async def handle(
        self,
        request: ChatRequest,
        user_ctx: UserContext
    ) -> AsyncIterator[str]:
        await self.rate_limiter.check(user_ctx)
        history = await self.conversation_store.load(request.session_id)
        await self.content_filter.check_input(request.user_message)
        messages = self.prompt_builder.build(history, request.user_message)
        
        full_response = ""
        async for token in self.llm_client.complete(messages, ...):
            full_response += token
            yield token
        
        await self.conversation_store.save(request.session_id, ...)
        await self.audit_logger.log(...)
```

---

#### 5.2.2 vLLM Client Implementation

```python
class VLLMClient(ILLMClient):
    """
    Communicates with vLLM's OpenAI-compatible /v1/chat/completions endpoint.
    Supports streaming via SSE.
    """
    def __init__(self, router_url: str, timeout_s: int = 90):
        self.router_url = router_url
        self.client = httpx.AsyncClient(timeout=timeout_s)

    async def complete(
        self,
        messages: List[dict],
        model: str = "default",
        max_tokens: int = 1024,
        temperature: float = 0.7,
        stream: bool = True,
    ) -> AsyncIterator[str]:
        payload = {
            "model": model,
            "messages": messages,
            "max_tokens": max_tokens,
            "temperature": temperature,
            "stream": stream,
        }
        async with self.client.stream(
            "POST",
            f"{self.router_url}/v1/chat/completions",
            json=payload,
        ) as response:
            response.raise_for_status()
            async for line in response.aiter_lines():
                if line.startswith("data: "):
                    data = line[6:]
                    if data == "[DONE]":
                        return
                    chunk = json.loads(data)
                    delta = chunk["choices"][0]["delta"].get("content", "")
                    if delta:
                        yield delta
```

---

#### 5.2.3 Session / Conversation Store

```python
class RedisConversationStore(IConversationStore):
    """
    Stores conversation history per session in Redis as a JSON list.
    TTL of 3600s (1 hour) by default.
    History is window-truncated to avoid exceeding LLM context limits.
    """
    MAX_HISTORY_TOKENS = 4096  # Reserve remainder for new prompt + response

    def __init__(self, redis: Redis, ttl_seconds: int = 3600):
        self.redis = redis
        self.ttl = ttl_seconds

    async def load(self, session_id: str) -> List[Message]:
        raw = await self.redis.get(f"session:{session_id}")
        if not raw:
            return []
        messages = [Message(**m) for m in json.loads(raw)]
        return self._truncate_to_token_budget(messages)

    async def save(self, session_id: str, messages: List[Message]) -> None:
        serialized = json.dumps([asdict(m) for m in messages])
        await self.redis.setex(f"session:{session_id}", self.ttl, serialized)

    def _truncate_to_token_budget(self, messages: List[Message]) -> List[Message]:
        # Sliding window: drop oldest messages if over token budget
        # Uses tiktoken for approximate token counting
        ...
```

---

#### 5.2.4 LLM Router (Load Balancer)

```python
class LLMRouter:
    """
    Least-pending-requests router across vLLM worker pool.
    Health-checked and automatically removes unhealthy workers.
    """
    def __init__(self, worker_urls: List[str]):
        self.workers: Dict[str, WorkerState] = {
            url: WorkerState(url=url) for url in worker_urls
        }

    async def get_best_worker(self) -> str:
        healthy = [w for w in self.workers.values() if w.healthy]
        if not healthy:
            raise NoHealthyWorkersError()
        return min(healthy, key=lambda w: w.pending_requests).url

    async def health_check_loop(self):
        while True:
            for worker in self.workers.values():
                try:
                    resp = await httpx.get(f"{worker.url}/health", timeout=2)
                    worker.healthy = resp.status_code == 200
                except Exception:
                    worker.healthy = False
            await asyncio.sleep(5)

@dataclass
class WorkerState:
    url: str
    healthy: bool = True
    pending_requests: int = 0
```

---

### 5.3 API Contract

#### POST `/v1/chat/completions` — Standard Chat

**Request:**
```json
{
  "session_id": "sess_abc123",
  "model": "llama-3-70b",
  "messages": [
    { "role": "user", "content": "Explain quantum entanglement simply." }
  ],
  "stream": true,
  "max_tokens": 512,
  "temperature": 0.7
}
```

**Response (Streaming SSE):**
```
data: {"id":"chatcmpl-xyz","choices":[{"delta":{"content":"Quantum "},"index":0}]}
data: {"id":"chatcmpl-xyz","choices":[{"delta":{"content":"entanglement"},"index":0}]}
...
data: [DONE]
```

**Response (Non-streaming):**
```json
{
  "id": "chatcmpl-xyz",
  "model": "llama-3-70b",
  "choices": [{
    "message": { "role": "assistant", "content": "Quantum entanglement is..." },
    "finish_reason": "stop"
  }],
  "usage": { "prompt_tokens": 42, "completion_tokens": 180, "total_tokens": 222 }
}
```

---

#### GET `/v1/sessions/{session_id}/history`

Returns the conversation history for a session (scoped to authenticated user).

---

#### DELETE `/v1/sessions/{session_id}`

Clears conversation history and revokes session.

---

### 5.4 Database Schema

#### PostgreSQL — `users` Table

```sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       TEXT UNIQUE NOT NULL,
    tier        TEXT NOT NULL DEFAULT 'free',  -- free | pro | enterprise
    created_at  TIMESTAMPTZ DEFAULT now(),
    is_active   BOOLEAN DEFAULT TRUE
);
```

#### PostgreSQL — `audit_logs` Table

```sql
CREATE TABLE audit_logs (
    id              BIGSERIAL PRIMARY KEY,
    user_id         UUID NOT NULL REFERENCES users(id),
    session_id      TEXT NOT NULL,
    model           TEXT NOT NULL,
    prompt_tokens   INTEGER,
    completion_tokens INTEGER,
    latency_ms      INTEGER,
    status          TEXT,         -- success | error | filtered
    created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_audit_user_created ON audit_logs(user_id, created_at DESC);
```

#### Redis Key Schema

| Key Pattern | Type | TTL | Purpose |
|---|---|---|---|
| `session:{session_id}` | String (JSON) | 3600s | Conversation history |
| `ratelimit:{user_id}:{YYYYMMDDHHMM}` | Counter (INCR) | 60s | Per-minute rate limit |
| `worker_registry` | Hash | — | vLLM worker endpoints and health |
| `model_routing` | Hash | — | Model name → worker pool mapping |

---

### 5.5 Concurrency Model

```
                    ┌───────────────────────────────────────┐
                    │     Chat Orchestration Service         │
                    │     (FastAPI + Uvicorn + asyncio)      │
                    │                                        │
                    │  Event Loop                            │
                    │  ┌─────────────────────────────────┐  │
                    │  │  Coroutine 1: User A Request    │  │
                    │  │  Coroutine 2: User B Request    │  │
                    │  │  Coroutine 3: User C Request    │  │
                    │  │  ...                             │  │
                    │  │  Coroutine N: User N Request    │  │
                    │  └─────────────────────────────────┘  │
                    │  All coroutines share single thread    │
                    │  I/O ops yield control (non-blocking)  │
                    └───────────────────────────────────────┘
                                       │
                    Multiple Uvicorn worker processes (x CPU cores)
                    deployed behind a load balancer

Concurrency per pod:
  - Uvicorn workers: 4 (= CPU cores)
  - Async coroutines per worker: ~500 (limited by Redis/vLLM connections)
  - Total concurrent requests per pod: ~2,000

At scale (10 pods): ~20,000 concurrent active requests
```

**vLLM Concurrency:**

vLLM uses its own async engine with **Continuous Batching**, meaning:
- No fixed batch size; new requests are dynamically added as GPU processes existing sequences
- Multiple users' token generation is interleaved at the sequence scheduler level
- KV-cache is managed per-sequence by PagedAttention — no wasted GPU VRAM

---

## 6. Data Architecture

### 6.1 Data Flow Diagram

```
User Input
    │
    ├──► Redis (session history read)
    │
    ├──► PostgreSQL (rate limit + user tier lookup, cached in Redis)
    │
    ├──► vLLM (inference)
    │         └──► Model weights loaded from S3/GCS on startup
    │
    ├──► Redis (session history write)
    │
    └──► PostgreSQL (audit log async write)
```

### 6.2 Model Artifact Storage

- **Location:** AWS S3 (`s3://ai-models/llama-3-70b/`)
- **Access:** EC2 instance role with read-only S3 policy
- **Mount:** Models pre-downloaded to local NVMe SSD on GPU node startup (via init container or startup script)
- **Versioning:** Model versions tracked in S3 object versioning + metadata registry in PostgreSQL

---

## 7. Security Architecture

### 7.1 Threat Model Summary

| Threat | Mitigation |
|--------|-----------|
| Unauthorized API access | JWT authentication + API key scoping |
| DDoS / abuse | Rate limiting at gateway + WAF rules |
| Prompt injection / jailbreak | Input content filter before LLM call |
| Data exfiltration | No raw user data stored in logs; PII masking |
| Model extraction | Rate limits + response token caps |
| Man-in-the-middle | TLS 1.3 end-to-end; mTLS between internal services |
| Insider threat | RBAC; audit logs; VPC isolation |
| Supply chain (model tampering) | SHA256 hash verification of model artifacts on load |

### 7.2 Network Security

```
Internet
    │
    ▼
AWS Shield + WAF (Layer 7 rules)
    │
    ▼
Public ALB (TLS termination)
    │
    ▼
VPC — Private Subnet
    ├── API Gateway / Kong (Security Group: 443 inbound only)
    ├── App Pods (Security Group: accept only from Gateway SG)
    ├── GPU Nodes (Security Group: accept only from App SG on port 8000)
    └── Data Tier (Security Group: accept only from App SG on DB ports)
```

### 7.3 Secrets Management

- All secrets stored in **AWS Secrets Manager**
- Applications retrieve secrets at startup via IAM Role (no hardcoded credentials)
- Secrets rotated automatically (DB passwords every 30 days)

---

## 8. Scalability & Performance Strategy

### 8.1 Horizontal Scaling

| Tier | Scaling Mechanism | Trigger Metric |
|------|-------------------|----------------|
| Chat Orchestration Service | Kubernetes HPA | CPU > 70% or request queue > 50 |
| LLM Router | Kubernetes HPA | Request throughput |
| vLLM Workers | Kubernetes HPA (Karpenter for GPU nodes) | Average pending requests > 20 per worker |
| Redis | Redis Cluster (auto-sharding) | Memory > 75% |
| PostgreSQL | Read replicas | Read query latency > 50ms |

### 8.2 vLLM Performance Tuning

- **`--max-num-seqs`**: Set to maximize GPU batch utilization (tune per GPU memory)
- **`--enable-chunked-prefill`**: Reduces TTFT by interleaving prefill and decode
- **`--gpu-memory-utilization 0.90`**: Leave 10% headroom for CUDA kernels
- **`--quantization awq`**: Use AWQ 4-bit quantization for 2x memory efficiency on smaller GPUs
- **KV-cache offloading**: vLLM supports CPU offload for KV blocks when GPU VRAM is constrained

### 8.3 Caching Strategy

```
Request comes in
    │
    ├── [Cache Check] Redis: Has this exact prompt been answered recently?
    │       └── Semantic deduplication for common queries (optional)
    │
    ├── [Session Cache] Redis: Load conversation history (fast, < 2ms)
    │
    └── [vLLM Prefix Cache] vLLM caches KV states for common system prompts
            └── Reuses computed attention for identical prefixes across users
```

---

## 9. Deployment Architecture

### 9.1 Infrastructure (AWS)

```
Region: us-east-1 (primary)
    │
    ├── VPC (10.0.0.0/16)
    │     ├── Public Subnets: ALB, NAT Gateway
    │     └── Private Subnets: All application and data components
    │
    ├── EKS Cluster
    │     ├── Node Group: CPU (m6i.4xlarge) — App services, Redis, routing
    │     └── Node Group: GPU (p4d.24xlarge, Spot + On-Demand mix)
    │           └── Managed by Karpenter for fast scale-out
    │
    ├── RDS PostgreSQL (Multi-AZ, db.r6g.2xlarge)
    ├── ElastiCache Redis Cluster (cache.r7g.xlarge, 3 shards)
    └── S3 (model artifacts, logs archive)

DR Region: us-west-2
    └── Warm standby: RDS read replica promoted on failover
```

### 9.2 Kubernetes Resources

```yaml
# Chat Orchestration Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chat-orchestrator
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: orchestrator
        image: ai-assistant/orchestrator:v1.2.0
        resources:
          requests: { cpu: "1", memory: "2Gi" }
          limits:   { cpu: "2", memory: "4Gi" }
        env:
        - name: VLLM_ROUTER_URL
          value: "http://llm-router-svc:8080"
        - name: REDIS_URL
          valueFrom:
            secretKeyRef: { name: app-secrets, key: redis-url }
---
# vLLM Worker
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-worker
spec:
  replicas: 2
  template:
    spec:
      nodeSelector:
        nvidia.com/gpu: "true"
      containers:
      - name: vllm
        image: vllm/vllm-openai:v0.4.3
        resources:
          limits:
            nvidia.com/gpu: "4"
        args:
        - "--model=/models/llama-3-70b"
        - "--tensor-parallel-size=4"
        - "--max-num-seqs=256"
```

### 9.3 CI/CD Pipeline

```
Developer Push → GitHub
    │
    ▼
GitHub Actions
    ├── Unit Tests (pytest)
    ├── Integration Tests (testcontainers)
    ├── Security Scan (Trivy, Bandit)
    └── Build & Push Docker image → ECR

    │
    ▼
ArgoCD (GitOps)
    ├── Staging deploy → smoke tests → manual approval gate
    └── Production deploy (rolling update, 0 downtime)
```

---

## 10. Observability & Operations

### 10.1 Metrics (Prometheus + Grafana)

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| `llm_requests_total` | App | — |
| `llm_ttft_seconds` (P95) | App | > 5s |
| `llm_tokens_per_second` | vLLM | < 50 tok/s |
| `vllm_pending_requests` | vLLM `/metrics` | > 100 |
| `vllm_gpu_utilization` | DCGM Exporter | < 50% (scale down) |
| `session_cache_hit_rate` | Redis | < 80% |
| `error_rate_5xx` | App + Gateway | > 1% |

### 10.2 Distributed Tracing

- **OpenTelemetry** instrumented across all services
- Trace propagated: API Gateway → Orchestrator → vLLM Client → Router → vLLM
- Traces exported to **AWS X-Ray** or **Jaeger**
- Every request tagged with: `user_id`, `session_id`, `model`, `trace_id`

### 10.3 Logging

- Structured JSON logs (all services)
- Log fields: `timestamp`, `level`, `service`, `trace_id`, `user_id`, `message`
- Shipped to **OpenSearch (ELK)** via Fluent Bit DaemonSet
- PII fields masked before ingestion (email, IP)

### 10.4 Alerting

- PagerDuty integration for P1/P2 alerts
- Runbooks stored in Confluence, linked from alert annotations

---

## 11. Architecture Decision Records (ADRs)

### ADR-001: vLLM as Inference Engine

**Status:** Accepted  
**Decision:** Use vLLM over alternatives (TGI, Triton, DeepSpeed-MII)  
**Rationale:** vLLM's PagedAttention and continuous batching provide superior throughput for concurrent multi-user serving. OpenAI-compatible API minimizes integration effort. Active community and production-ready.  
**Trade-off:** Tied to vLLM's deployment model; GPU cluster management complexity.

---

### ADR-002: Redis for Session Storage

**Status:** Accepted  
**Decision:** Store conversation history in Redis with TTL rather than PostgreSQL  
**Rationale:** Sub-millisecond read latency critical for every request. Conversation data is ephemeral (1 hour TTL). Redis Cluster provides horizontal scaling.  
**Trade-off:** Sessions lost if Redis cluster fails without persistence configured. Mitigation: Enable Redis AOF persistence.

---

### ADR-003: FastAPI + asyncio for Orchestration Service

**Status:** Accepted  
**Decision:** Use FastAPI (async Python) over Go or Node.js  
**Rationale:** Python ecosystem aligns with ML/AI tooling (tokenizers, content filters). asyncio handles thousands of concurrent I/O-bound coroutines efficiently. FastAPI provides native streaming response support.  
**Trade-off:** Higher memory footprint vs Go. Mitigation: Uvicorn multi-process deployment.

---

### ADR-004: Spot Instances for GPU Workers

**Status:** Accepted  
**Decision:** Use 70% Spot + 30% On-Demand GPU instances  
**Rationale:** Spot instances are 60–80% cheaper. vLLM workers are stateless; requests in-flight on a preempted node are retried by the router.  
**Trade-off:** Interruption risk. Mitigation: Router detects unhealthy workers; client retry logic; mixed On-Demand floor.

---

## 12. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| GPU node preemption (Spot) | Medium | Medium | Router retries; On-Demand floor |
| vLLM OOM on large context | Medium | High | `max-model-len` cap; token budget enforcement in PromptBuilder |
| Redis cluster failure | Low | High | AOF + RDB persistence; Multi-AZ; session reconstruction from PostgreSQL fallback |
| Model weight corruption | Low | Critical | SHA256 hash verification; S3 versioning |
| Prompt injection attack | High | Medium | Input content filter; system prompt hardening; output monitoring |
| GPU supply shortage | Medium | High | Multi-cloud GPU strategy (AWS + GCP fallback); queue-based graceful degradation |
| Cold start latency (scale-out) | High | Medium | Warm pools; pre-pull model images; Karpenter fast provisioning |

---

*Document maintained by the Software Architecture team. Review cycle: quarterly or on major system changes.*
