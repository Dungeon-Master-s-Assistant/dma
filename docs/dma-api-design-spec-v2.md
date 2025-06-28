# **DMA API Design Specification v2.0 \- MVC Edition**

## **Streamlined API Architecture for Q3 2025 Launch**

**Version:** 2.0 (MVC-Focused)  
**Date:** June 28, 2025  
**Author:** J.P. Dow

---

## **Table of Contents**

1. [Overview](#overview)  
2. [Public REST API](#public-rest-api)  
3. [Fast Lane API](#fast-lane-api)  
4. [Internal Message Bus](#internal-message-bus)  
5. [Authentication & Security](#authentication--security)  
6. [Error Handling](#error-handling)  
7. [Observability](#observability)  
8. [API Contracts](#api-contracts)  
9. [Implementation Checklist](#implementation-checklist)

---

## **Overview**

This specification defines the API architecture for the DMA Minimum Viable Campaign (MVC) release, focusing on:

- **3 agents only**: Scribe, System Mastery, World Designer  
- **Single PostgreSQL database**  
- **Redis Streams for messaging**  
- **Fast Lane RPC for sub-500ms operations**

### **Architecture Diagram**

graph TB

    UI\[DM Interface\] \--\>|REST| ORC\[Orchestrator\]

    UI \--\>|Fast Lane| FL\[Fast Lane Service\]

    

    ORC \--\>|Redis Streams| RS\[Message Bus\]

    RS \--\> SCA\[Scribe Agent\]

    RS \--\> SMA\[System Mastery Agent\]

    RS \--\> WED\[World Designer Agent\]

    

    SCA \--\> PG\[(PostgreSQL)\]

    SMA \--\> PG

    WED \--\> PG

    FL \--\> PG

    

    ORC \--\> LLM\[OpenAI GPT-4\]

---

## **Public REST API**

### **Base Configuration**

api:

  version: "v1"

  base\_url: "http://localhost:8000/v1"

  timeout: 30s

  max\_request\_size: 10MB

### **Core Endpoints**

#### **1\. Prompt Processing**

POST /v1/prompts

Authorization: Bearer {token}

Content-Type: application/json

{

  "prompt": "Describe the ancient dwarven hall",

  "context": {

    "campaign\_id": "uuid",

    "location\_id": "uuid",

    "session\_id": "uuid"

  },

  "options": {

    "timeout\_ms": 30000,

    "include\_stats": false

  }

}

Response 202 Accepted:

{

  "request\_id": "req\_123",

  "status": "processing",

  "poll\_url": "/v1/prompts/req\_123"

}

#### **2\. Result Polling**

GET /v1/prompts/{request\_id}

Authorization: Bearer {token}

Response 200 OK:

{

  "request\_id": "req\_123",

  "status": "completed",

  "result": {

    "content": "The ancient dwarven hall...",

    "entities\_created": \["loc\_789", "npc\_456"\],

    "tokens\_used": 150

  },

  "completed\_at": "2025-06-28T10:30:00Z"

}

#### **3\. Entity CRUD**

\# List entities

GET /v1/entities/{type}?campaign\_id={uuid}\&limit=100\&offset=0

\# Get single entity

GET /v1/entities/{type}/{id}

\# Create entity

POST /v1/entities/{type}

{

  "name": "Broken Tower",

  "description": "A crumbling watchtower",

  "location\_type": "building",

  "campaign\_id": "uuid"

}

\# Update entity (with optimistic locking)

PUT /v1/entities/{type}/{id}

If-Match: {version}

{

  "description": "Updated description"

}

\# Soft delete

DELETE /v1/entities/{type}/{id}

#### **4\. Search**

POST /v1/search

{

  "query": "ancient elven",

  "entity\_types": \["location", "item"\],

  "campaign\_id": "uuid",

  "limit": 20

}

Response:

{

  "results": \[

    {

      "entity\_id": "loc\_123",

      "entity\_type": "location",

      "name": "Ancient Elven Temple",

      "score": 0.95

    }

  \],

  "total": 3

}

### **Standard Headers**

\# All responses include:

X-Request-ID: req\_123

X-RateLimit-Remaining: 95

X-RateLimit-Reset: 1719573600

X-Response-Time: 234ms

---

## **Fast Lane API**

Direct RPC endpoints bypassing the message bus for latency-critical operations.

### **Design Principles**

- **No LLM calls**  
- **No database writes**  
- **In-memory caching**  
- **Sub-500ms P95 latency**

### **Endpoints**

#### **1\. THAC0 Calculation**

GET /v1/fast/thac0?level=5\&class=fighter\&strength=18

Response 200 OK (12ms):

{

  "base\_thac0": 16,

  "strength\_bonus": 1,

  "final\_thac0": 15,

  "cache\_hit": true

}

#### **2\. Saving Throws**

GET /v1/fast/saves?class=wizard\&level=7

Response 200 OK (8ms):

{

  "paralyzation": 13,

  "rod\_staff\_wand": 9,

  "petrification": 11,

  "breath\_weapon": 13,

  "spell": 10

}

#### **3\. Armor Class**

POST /v1/fast/ac

{

  "base\_ac": 10,

  "armor": "chain\_mail",

  "shield": true,

  "dexterity": 16

}

Response 200 OK (15ms):

{

  "armor\_ac": 5,

  "shield\_bonus": \-1,

  "dex\_bonus": \-2,

  "final\_ac": 2

}

### **Cache Strategy**

\# In-memory rule cache with startup preload

RULE\_CACHE \= {

    'thac0': load\_thac0\_tables(),

    'saves': load\_saving\_throw\_tables(),

    'armor': load\_armor\_values()

}

@app.on\_event("startup")

async def preload\_cache():

    """Load all 2e rules into memory at startup"""

    await load\_rules\_from\_database()

---

## **Internal Message Bus**

### **Technology Choice: Redis Streams**

Chosen over Pub/Sub for:

- **Persistence**: Messages survive crashes  
- **Consumer groups**: Load balancing  
- **Dead letter queues**: Failed message handling  
- **Message ordering**: FIFO guarantees

### **Stream Configuration**

streams:

  agent\_tasks:

    max\_length: 10000

    retention: "24h"

    consumer\_groups:

      \- scribe\_workers

      \- system\_mastery\_workers

      \- world\_designer\_workers

  

  agent\_results:

    max\_length: 5000

    retention: "24h"

    consumer\_groups:

      \- orchestrator\_workers

### **Message Schema**

interface AgentMessage {

  // Required fields

  message\_id: string;        // UUID

  timestamp: string;         // ISO 8601

  source: string;            // Agent name or "orchestrator"

  target: string;            // Agent name or "orchestrator"

  message\_type: MessageType;

  correlation\_id: string;    // Links request/response

  

  // Payload

  payload: Record\<string, any\>;

  

  // Delivery guarantees

  max\_retries: number;       // Default: 3

  timeout\_ms: number;        // Default: 30000

  priority: 1 | 2 | 3;       // 1 \= highest

}

enum MessageType {

  // Task assignments

  TASK\_CREATE\_ENTITY \= "task.create\_entity",

  TASK\_UPDATE\_ENTITY \= "task.update\_entity",

  TASK\_SEARCH\_ENTITIES \= "task.search\_entities",

  TASK\_VALIDATE\_RULE \= "task.validate\_rule",

  TASK\_GENERATE\_CONTENT \= "task.generate\_content",

  

  // Results

  RESULT\_SUCCESS \= "result.success",

  RESULT\_ERROR \= "result.error",

  RESULT\_PARTIAL \= "result.partial",

  

  // System

  AGENT\_READY \= "agent.ready",

  AGENT\_BUSY \= "agent.busy",

  HEALTH\_CHECK \= "system.health\_check"

}

### **Example Messages**

// Task Assignment

{

  "message\_id": "msg\_123",

  "timestamp": "2025-06-28T10:00:00Z",

  "source": "orchestrator",

  "target": "scribe",

  "message\_type": "task.create\_entity",

  "correlation\_id": "req\_456",

  "payload": {

    "entity\_type": "location",

    "data": {

      "name": "The Sunken Temple",

      "description": "An ancient temple half-submerged...",

      "location\_type": "dungeon"

    }

  },

  "max\_retries": 3,

  "timeout\_ms": 5000,

  "priority": 2

}

// Task Result

{

  "message\_id": "msg\_124",

  "timestamp": "2025-06-28T10:00:02Z",

  "source": "scribe",

  "target": "orchestrator",

  "message\_type": "result.success",

  "correlation\_id": "req\_456",

  "payload": {

    "entity\_id": "loc\_789",

    "version": 1,

    "created\_at": "2025-06-28T10:00:02Z"

  }

}

### **Dead Letter Queue**

\# Automatic DLQ handling

async def process\_with\_dlq(message: AgentMessage):

    try:

        result \= await process\_message(message)

        await redis.xack(STREAM\_NAME, GROUP\_NAME, message.id)

        return result

    except Exception as e:

        message.retry\_count \+= 1

        

        if message.retry\_count \>= message.max\_retries:

            \# Move to DLQ

            await redis.xadd(

                f"{STREAM\_NAME}:dlq",

                {

                    \*\*message.dict(),

                    "error": str(e),

                    "failed\_at": datetime.utcnow().isoformat()

                }

            )

        else:

            \# Retry with backoff

            delay \= 2 \*\* message.retry\_count

            await redis.xadd(

                f"{STREAM\_NAME}:retry",

                message.dict(),

                id=f"{int(time.time() \+ delay) \* 1000}-0"

            )

---

## **Authentication & Security**

### **Token Strategy**

auth:

  \# UI \-\> Orchestrator: JWT tokens

  ui\_auth:

    type: "jwt"

    issuer: "dma-auth"

    expiry: "1h"

    refresh\_enabled: true

  

  \# Internal services: Shared secret

  service\_auth:

    type: "bearer"

    token\_prefix: "svc\_"

    rotation\_interval: "24h"

### **Implementation**

from fastapi import Security, HTTPException

from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security \= HTTPBearer()

async def verify\_token(credentials: HTTPAuthorizationCredentials \= Security(security)):

    token \= credentials.credentials

    

    if token.startswith("svc\_"):

        \# Service token validation

        if not await validate\_service\_token(token):

            raise HTTPException(401, "Invalid service token")

    else:

        \# JWT validation

        try:

            payload \= jwt.decode(token, SECRET\_KEY, algorithms=\["HS256"\])

            return payload

        except jwt.ExpiredSignatureError:

            raise HTTPException(401, "Token expired")

        except jwt.InvalidTokenError:

            raise HTTPException(401, "Invalid token")

\# Apply to all endpoints

@app.post("/v1/prompts", dependencies=\[Depends(verify\_token)\])

async def process\_prompt(request: PromptRequest):

    ...

### **Local Development**

\# .env.local

AUTH\_ENABLED=true

DEV\_TOKEN=dev\_local\_token\_123

SECRET\_KEY=local\_development\_key

\# Docker compose

environment:

  \- AUTH\_ENABLED=${AUTH\_ENABLED:-true}

  \- DEV\_TOKEN=${DEV\_TOKEN}

---

## **Error Handling**

### **Error Response Format**

Following RFC 7807 Problem Details:

{

  "type": "https://api.dma.com/errors/validation",

  "title": "Validation Error",

  "status": 400,

  "detail": "The location\_type 'castle' is not valid",

  "instance": "/v1/entities/location",

  "errors": \[

    {

      "field": "location\_type",

      "message": "Must be one of: continent, kingdom, city, town, dungeon, building, room, wilderness"

    }

  \]

}

### **HTTP Status Codes**

\# Proper status codes, not 200 for everything

class ErrorResponses:

    BAD\_REQUEST \= (400, "Invalid request format")

    UNAUTHORIZED \= (401, "Authentication required")

    FORBIDDEN \= (403, "Insufficient permissions")

    NOT\_FOUND \= (404, "Resource not found")

    CONFLICT \= (409, "Resource version conflict")

    UNPROCESSABLE \= (422, "Validation failed")

    TOO\_MANY\_REQUESTS \= (429, "Rate limit exceeded")

    INTERNAL\_ERROR \= (500, "Internal server error")

    BAD\_GATEWAY \= (502, "Upstream service error")

    SERVICE\_UNAVAILABLE \= (503, "Service temporarily unavailable")

    TIMEOUT \= (504, "Request timeout")

### **Error Categories**

from enum import Enum

class ErrorCategory(Enum):

    \# Transient \- can retry

    NETWORK\_ERROR \= "network\_error"

    TIMEOUT \= "timeout"

    RATE\_LIMIT \= "rate\_limit"

    SERVICE\_UNAVAILABLE \= "service\_unavailable"

    

    \# Persistent \- don't retry

    VALIDATION\_ERROR \= "validation\_error"

    AUTHENTICATION\_ERROR \= "auth\_error"

    PERMISSION\_ERROR \= "permission\_error"

    NOT\_FOUND \= "not\_found"

    

    \# AI-specific

    LLM\_ERROR \= "llm\_error"

    CONTENT\_FILTER \= "content\_filter"

    TOKEN\_LIMIT \= "token\_limit"

---

## **Observability**

### **Required Labels**

Every metric and trace must include:

standard\_labels:

  \- service: "orchestrator|scribe|system\_mastery|world\_designer"

  \- environment: "local|staging|production"

  \- version: "2.0.0"

  \- method: "endpoint or message\_type"

  \- status: "success|error"

  \- error\_type: "category if error"

### **Key Metrics**

\# Prometheus metrics

from prometheus\_client import Counter, Histogram, Gauge

\# Request metrics

request\_duration \= Histogram(

    'dma\_request\_duration\_seconds',

    'Request duration',

    \['service', 'method', 'status'\],

    buckets=\[0.1, 0.5, 1.0, 2.0, 5.0, 10.0\]

)

request\_counter \= Counter(

    'dma\_requests\_total',

    'Total requests',

    \['service', 'method', 'status'\]

)

\# Token usage

tokens\_used \= Counter(

    'dma\_llm\_tokens\_total',

    'LLM tokens consumed',

    \['service', 'model', 'operation'\]

)

\# Queue depth

queue\_depth \= Gauge(

    'dma\_queue\_depth',

    'Current queue depth',

    \['stream', 'consumer\_group'\]

)

### **Trace Examples**

from opentelemetry import trace

tracer \= trace.get\_tracer(\_\_name\_\_)

@app.post("/v1/prompts")

async def process\_prompt(request: PromptRequest):

    with tracer.start\_as\_current\_span(

        "prompt\_processing",

        attributes={

            "campaign\_id": request.context.campaign\_id,

            "prompt\_length": len(request.prompt)

        }

    ) as span:

        \# Intent classification

        with tracer.start\_span("classify\_intent"):

            intent \= await classifier.classify(request.prompt)

            span.set\_attribute("intent", intent.type)

        

        \# Task decomposition

        with tracer.start\_span("decompose\_tasks"):

            tasks \= await decompose\_into\_tasks(request, intent)

            span.set\_attribute("task\_count", len(tasks))

        

        \# Dispatch to agents

        with tracer.start\_span("dispatch\_tasks"):

            results \= await dispatch\_to\_agents(tasks)

        

        return results

### **Logging Standards**

import structlog

logger \= structlog.get\_logger()

\# Structured logging with context

logger.info(

    "task\_completed",

    task\_id="task\_123",

    agent="scribe",

    duration\_ms=234,

    entity\_type="location",

    entity\_id="loc\_789"

)

\# Error logging with full context

logger.error(

    "task\_failed",

    task\_id="task\_124",

    agent="world\_designer",

    error\_type="llm\_timeout",

    error\_message=str(e),

    correlation\_id="req\_456",

    retry\_count=2

)

---

## **API Contracts**

### **JSON Schema Repository**

contracts/

├── v1/

│   ├── requests/

│   │   ├── prompt\_request.json

│   │   ├── entity\_create.json

│   │   └── search\_request.json

│   ├── responses/

│   │   ├── prompt\_response.json

│   │   ├── entity.json

│   │   └── error.json

│   └── messages/

│       ├── agent\_task.json

│       └── agent\_result.json

└── README.md

### **Example Schema**

{

  "$schema": "http://json-schema.org/draft-07/schema\#",

  "$id": "https://api.dma.com/schemas/v1/prompt\_request.json",

  "title": "PromptRequest",

  "type": "object",

  "required": \["prompt", "context"\],

  "properties": {

    "prompt": {

      "type": "string",

      "minLength": 1,

      "maxLength": 5000

    },

    "context": {

      "type": "object",

      "required": \["campaign\_id"\],

      "properties": {

        "campaign\_id": {

          "type": "string",

          "format": "uuid"

        },

        "location\_id": {

          "type": "string",

          "format": "uuid"

        },

        "session\_id": {

          "type": "string",

          "format": "uuid"

        }

      }

    },

    "options": {

      "type": "object",

      "properties": {

        "timeout\_ms": {

          "type": "integer",

          "minimum": 1000,

          "maximum": 60000,

          "default": 30000

        },

        "include\_stats": {

          "type": "boolean",

          "default": false

        }

      }

    }

  }

}

### **Contract Testing**

\# tests/contract/test\_api\_contracts.py

import json

from jsonschema import validate

import pytest

class TestAPIContracts:

    @pytest.fixture

    def schemas(self):

        """Load all schemas"""

        schemas \= {}

        for schema\_file in Path("contracts/v1").rglob("\*.json"):

            with open(schema\_file) as f:

                schema \= json.load(f)

                schemas\[schema\["$id"\]\] \= schema

        return schemas

    

    def test\_prompt\_request\_valid(self, schemas):

        """Test valid prompt request"""

        request \= {

            "prompt": "Describe the dungeon",

            "context": {

                "campaign\_id": "123e4567-e89b-12d3-a456-426614174000"

            }

        }

        

        schema \= schemas\["https://api.dma.com/schemas/v1/prompt\_request.json"\]

        validate(instance=request, schema=schema)  \# Should not raise

    

    def test\_prompt\_request\_invalid(self, schemas):

        """Test invalid prompt request"""

        request \= {

            "prompt": "",  \# Empty prompt

            "context": {}  \# Missing campaign\_id

        }

        

        schema \= schemas\["https://api.dma.com/schemas/v1/prompt\_request.json"\]

        with pytest.raises(ValidationError):

            validate(instance=request, schema=schema)

---

## **Implementation Checklist**

### **Week 1: Foundation**

- [ ] Set up FastAPI with proper structure  
- [ ] Implement authentication middleware  
- [ ] Configure Redis Streams  
- [ ] Create base error handling  
- [ ] Set up Prometheus metrics  
- [ ] Define JSON schemas

### **Week 2: Core APIs**

- [ ] Implement `/v1/prompts` endpoints  
- [ ] Implement `/v1/entities` CRUD  
- [ ] Implement `/v1/search`  
- [ ] Create message bus publisher  
- [ ] Add request validation  
- [ ] Write initial tests

### **Week 3: Fast Lane**

- [ ] Build rule cache loader  
- [ ] Implement THAC0 endpoint  
- [ ] Implement saving throws endpoint  
- [ ] Implement AC calculation  
- [ ] Add cache warming  
- [ ] Performance test \< 500ms

### **Week 4: Integration**

- [ ] Connect Orchestrator to agents  
- [ ] Implement message consumers  
- [ ] Add circuit breakers  
- [ ] Set up OpenTelemetry  
- [ ] Contract testing suite  
- [ ] Load testing

### **Week 5: Hardening**

- [ ] Security audit  
- [ ] Rate limiting  
- [ ] DLQ monitoring  
- [ ] Documentation  
- [ ] Docker packaging  
- [ ] Final testing

---

## **Key Improvements from v1**

1. **Focused on MVC**: Only 3 agents, single database  
2. **Redis Streams**: Replaced unreliable Pub/Sub  
3. **Fast Lane API**: Sub-500ms operations documented  
4. **Proper HTTP codes**: Not everything returns 200  
5. **Machine-readable contracts**: JSON Schema validation  
6. **Required auth**: Even for local development  
7. **Clear observability**: Specific labels and traces  
8. **Version prefix**: `/v1/` on all endpoints  
9. **Pagination**: Default limits on list operations  
10. **5-week timeline**: Realistic implementation plan

This specification provides a complete, implementable API design that can ship in Q3 2025 while maintaining quality and performance standards.  
