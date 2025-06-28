# **DMA Technical Design v2.0 \- Engineering-Focused Rewrite Guide**

**Version:** 2.0 (Engineering-Focused Revision)  
**Date:** June 27, 2025  
**Status:** Complete Rewrite Guidelines Based on Engineering Critique

---

## **Executive Summary of Changes**

This document provides specific guidance for rewriting the DMA Technical Design to address critical engineering concerns identified in the review. The primary focus is on **reducing complexity**, **improving performance**, and **ensuring deliverability** of the Minimum Viable Campaign (MVC).

---

## **1\. Architecture & Technology Stack Simplification**

### **Current Issues**

- Mixing multiple messaging patterns (Redis Pub/Sub, RabbitMQ references)  
- Unclear database strategy (PostgreSQL \+ graph references)  
- Overly complex workflow orchestration

### **Rewrite Guidelines**

#### **Message Bus Architecture**

Replace all Redis Pub/Sub references with Redis Streams:

\# REMOVE: Redis Pub/Sub pattern

redis\_client.publish('agent.task', json.dumps(task))

\# REPLACE WITH: Redis Streams

redis\_client.xadd(

    'task-stream',

    {

        'task\_id': task\_id,

        'agent': 'scribe',

        'payload': json.dumps(task),

        'correlation\_id': correlation\_id

    },

    maxlen=10000  \# Automatic trimming

)

**Rationale:** Redis Streams provide persistence, ordering, consumer groups, and dead-letter queues out of the box.

#### **Database Strategy Clarification**

Remove all Neo4j/Cypher references. Use PostgreSQL exclusively:

\-- Graph-like queries using recursive CTEs

WITH RECURSIVE related\_entities AS (

    SELECT id, name, type, 1 as depth

    FROM entities

    WHERE id \= $1

    

    UNION ALL

    

    SELECT e.id, e.name, e.type, re.depth \+ 1

    FROM entities e

    INNER JOIN relationships r ON e.id \= r.target\_id

    INNER JOIN related\_entities re ON r.source\_id \= re.id

    WHERE re.depth \< 3

)

SELECT \* FROM related\_entities;

**Add to Design:**

- Clear statement: "PostgreSQL is the ONLY database for v0.1"  
- Document pgvector usage for semantic search  
- Show JSONB schema evolution patterns

---

## **2\. Orchestrator Optimization**

### **Current Issues**

- LLM calls for intent classification (expensive)  
- Conflated scheduling and queue management  
- Missing fast-lane implementation

### **Rewrite Guidelines**

#### **Intent Classification Without LLM**

class IntentClassifier:

    def \_\_init\_\_(self):

        \# Use lightweight local model

        self.classifier \= pipeline(

            "text-classification",

            model="microsoft/deberta-v3-small"

        )

        self.intent\_cache \= TTLCache(maxsize=1000, ttl=3600)

    

    def classify(self, prompt: str) \-\> Intent:

        \# Check cache first

        cache\_key \= hash(prompt.lower().strip())

        if cache\_key in self.intent\_cache:

            return self.intent\_cache\[cache\_key\]

        

        \# Local classification (no API call)

        result \= self.classifier(prompt)

        intent \= Intent(

            type=result\[0\]\['label'\],

            confidence=result\[0\]\['score'\]

        )

        

        self.intent\_cache\[cache\_key\] \= intent

        return intent

#### **Fast-Lane RPC Implementation**

Add dedicated section for fast-lane operations:

@app.get("/v1/fast/{operation}")

async def fast\_lane\_operation(

    operation: str,

    params: dict \= Depends()

):

    """

    Bypasses event bus for sub-500ms operations:

    \- THAC0 calculations

    \- Saving throw lookups

    \- Simple spell effects

    """

    if operation in FAST\_OPERATIONS:

        return FAST\_OPERATIONS\[operation\](\*\*params)

    else:

        raise HTTPException(404, "Operation not in fast lane")

\# In-memory rule cache

FAST\_OPERATIONS \= {

    'thac0': calculate\_thac0,

    'save': lookup\_saving\_throw,

    'ac': calculate\_armor\_class

}

---

## **3\. Agent Simplification**

### **Current Issues**

- Too many agents for MVC  
- Incomplete agent specifications  
- Complex inter-agent dependencies

### **Rewrite Guidelines**

#### **Remove These Sections Entirely:**

- Player Management & Psychology Agent  
- Performance & Atmosphere Agent  
- Tool & System Use Agent  
- Visual Assets & Immersion Architect  
- Improvisation & Creativity Engine  
- Advanced Content Creation Agent

**Replace with:** "Future Agents (Post-MVC)" section listing names only.

#### **Focus on 3 Core Agents:**

**1\. Scribe Agent (Simplified)**

class ScribeAgent:

    """

    Minimal viable Scribe \- data in/out only

    No external sync, no complex parsing

    """

    async def handle\_task(self, task: ScribeTask):

        match task.type:

            case TaskType.STORE:

                return await self.store\_entity(task.payload)

            case TaskType.RETRIEVE:

                return await self.get\_entity(task.entity\_id)

            case TaskType.SEARCH:

                return await self.search\_entities(task.query)

**2\. System Mastery Agent (Rules Only)**

class SystemMasteryAgent:

    """

    AD\&D 2e rules \- no balance analysis, no recommendations

    """

    def \_\_init\_\_(self):

        \# Load rules from static JSON files

        self.rules \= load\_rules\_from\_json()

        self.tables \= load\_tables\_from\_json()

    

    async def lookup(self, rule\_type: str, params: dict):

        \# Direct lookup, no LLM interpretation

        return self.rules.get(rule\_type, params)

**3\. World Designer (Basic Generation)**

- Simple template-based generation  
- No complex narrative analysis  
- Focus on usable game content

---

## **4\. Communication Protocol Refinement**

### **Current Issues**

- Complex message schemas  
- No idempotency handling  
- Missing work-token specification

### **Rewrite Guidelines**

#### **Simplified Event Schema**

{

  "event\_id": "uuid",

  "correlation\_id": "uuid",

  "idempotency\_key": "hash",

  "timestamp": "ISO8601",

  "source": "orchestrator",

  "target": "scribe", 

  "type": "entity.store",

  "payload": {},

  "auth": {

    "work\_token": "jwt",

    "expires\_at": "ISO8601"

  }

}

#### **Work Token Implementation**

class WorkTokenService:

    """

    Task-scoped tokens for agent authentication

    """

    def create\_work\_token(

        self,

        task\_id: str,

        agent: str,

        ttl\_seconds: int \= 300

    ) \-\> str:

        payload \= {

            'task\_id': task\_id,

            'agent': agent,

            'exp': time.time() \+ ttl\_seconds,

            'iat': time.time()

        }

        return jwt.encode(payload, self.secret, algorithm='HS256')

---

## **5\. Shared Resilience Package**

### **Current Issue**

- Circuit breakers only in documentation

### **Rewrite Guidelines**

Create `dma-shared-resilience` package specification:

\# dma\_shared\_resilience/\_\_init\_\_.py

from .circuit\_breaker import CircuitBreaker

from .bulkhead import Bulkhead

from .retry import RetryWithBackoff

\_\_all\_\_ \= \['CircuitBreaker', 'Bulkhead', 'RetryWithBackoff'\]

\# Usage in any agent:

from dma\_shared\_resilience import CircuitBreaker

class MyAgent:

    def \_\_init\_\_(self):

        self.llm\_breaker \= CircuitBreaker(

            failure\_threshold=5,

            recovery\_timeout=60,

            expected\_exception=LLMException

        )

    

    @self.llm\_breaker

    async def call\_llm(self, prompt):

        return await self.llm\_client.generate(prompt)

---

## **6\. Testing Strategy**

### **Current Issues**

- No contract testing  
- Missing replay harness  
- No performance regression tests

### **Rewrite Guidelines**

#### **Contract Testing Framework**

\# contracts/events/task.request.yaml

name: task.request

version: 1.0.0

fields:

  \- name: event\_id

    type: uuid

    required: true

  \- name: correlation\_id

    type: uuid

    required: true

  \- name: payload

    type: object

    required: true

    schema: ./schemas/task\_payload.json

#### **Replay Test Harness**

class ReplayHarness:

    """

    Records and replays event streams for testing

    """

    def record\_session(self, session\_id: str):

        \# Capture all events to file

        pass

    

    def replay\_session(self, recording\_path: str):

        \# Replay events, verify same outcomes

        pass

\# In CI/CD:

@pytest.mark.replay

def test\_combat\_sequence\_replay():

    harness \= ReplayHarness()

    result \= harness.replay\_session('fixtures/combat\_sequence.json')

    assert result.final\_state \== expected\_state

---

## **7\. Performance Specifications**

### **Current Issues**

- Unrealistic latency targets  
- No streaming for long operations

### **Rewrite Guidelines**

#### **Realistic Performance Targets**

performance\_slos:

  fast\_lane:

    \- operation: thac0\_lookup

      p99\_latency\_ms: 100

      availability: 99.9%

    

  standard:

    \- operation: generate\_npc

      p99\_latency\_ms: 2000

      availability: 99%

    

  complex:

    \- operation: generate\_dungeon

      p99\_latency\_ms: 15000

      streaming: true

      availability: 95%

#### **Streaming Implementation**

@app.post("/v1/generate/dungeon", response\_class=StreamingResponse)

async def generate\_dungeon\_streaming(request: DungeonRequest):

    async def generate():

        \# Stream layout immediately

        layout \= await generate\_layout(request)

        yield f"data: {json.dumps({'type': 'layout', 'data': layout})}\\n\\n"

        

        \# Stream rooms as generated

        async for room in generate\_rooms(layout):

            yield f"data: {json.dumps({'type': 'room', 'data': room})}\\n\\n"

    

    return StreamingResponse(generate(), media\_type="text/event-stream")

---

## **8\. Deployment & Infrastructure**

### **Current Issues**

- Complex hybrid deployment for MVC  
- Multiple infrastructure dependencies

### **Rewrite Guidelines**

#### **MVC: Local Only**

\# docker-compose.yml

version: '3.8'

services:

  orchestrator:

    build: ./orchestrator

    ports:

      \- "8000:8000"

    depends\_on:

      \- redis

      \- postgres

  

  postgres:

    image: postgres:15-alpine

    environment:

      POSTGRES\_DB: dma

      POSTGRES\_USER: dma

      POSTGRES\_PASSWORD: ${DB\_PASSWORD}

    volumes:

      \- ./data/postgres:/var/lib/postgresql/data

  

  redis:

    image: redis:7-alpine

    command: redis-server \--appendonly yes

    volumes:

      \- ./data/redis:/data

  

  \# Agents as separate containers

  scribe:

    build: ./agents/scribe

    environment:

      REDIS\_URL: redis://redis:6379

      DATABASE\_URL: postgresql://dma:${DB\_PASSWORD}@postgres/dma

**Remove:** All cloud deployment sections for v0.1

---

## **9\. Documentation Structure**

### **Rewrite the Technical Design into these focused documents:**

1. **Core Architecture** (15 pages max)  
     
   - System overview  
   - Component responsibilities  
   - Data flow

   

2. **API Specification** (10 pages)  
     
   - Event schemas  
   - REST endpoints  
   - Authentication

   

3. **Agent Specifications** (5 pages each)  
     
   - Scribe Agent  
   - System Mastery Agent  
   - World Designer Agent

   

4. **Deployment Guide** (5 pages)  
     
   - Local setup only  
   - Docker Compose  
   - Quick start

   

5. **Developer Guide** (10 pages)  
     
   - Contributing guidelines  
   - Testing approach  
   - Code standards

---

## **10\. Implementation Priorities**

### **Reorder sections to reflect actual build order:**

1. **Week 1-2: Foundation**  
     
   - PostgreSQL schema  
   - Redis Streams setup  
   - Basic Orchestrator shell

   

2. **Week 3-4: Fast Lane**  
     
   - Rules micro-library  
   - Direct RPC endpoints  
   - Performance tests

   

3. **Week 5-6: Scribe Agent**  
     
   - Basic CRUD operations  
   - Search functionality  
   - Integration tests

   

4. **Week 7-8: System Mastery**  
     
   - Rule lookup  
   - Combat calculations  
   - Validation

   

5. **Week 9-10: World Designer**  
     
   - Template-based generation  
   - Basic room/encounter creation

   

6. **Week 11-12: Integration**  
     
   - End-to-end workflows  
   - UI connection  
   - Demo preparation

---

## **Summary of Key Changes**

1. **Replace Redis Pub/Sub → Redis Streams**  
2. **Single Database → PostgreSQL only**  
3. **Remove 6 agents → Focus on 3**  
4. **Add Fast-Lane RPC → Sub-500ms operations**  
5. **Extract shared resilience → pip package**  
6. **Add contract testing → CI/CD requirement**  
7. **Define work tokens → Agent authentication**  
8. **Set realistic SLOs → Based on architecture**  
9. **Local deployment only → For MVC**  
10. **Split into focused docs → Max 15 pages each**

This rewrite will transform the Technical Design from an aspirational 240-page document into a pragmatic, implementable specification that can actually ship in Q3 2025\.  
