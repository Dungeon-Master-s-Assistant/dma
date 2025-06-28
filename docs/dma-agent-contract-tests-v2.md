# **DMA Agent Contract Tests & Shared Components**

## **Directory Structure**

dma/

├── agent\_contracts/

│   ├── schemas/

│   │   ├── bus-message.v1.json

│   │   ├── scribe-request.v1.json

│   │   ├── scribe-response.v1.json

│   │   ├── sma-request.v1.json

│   │   ├── sma-response.v1.json

│   │   ├── wed-request.v1.json

│   │   └── wed-response.v1.json

│   ├── golden/

│   │   ├── scribe\_ingest\_npc.json

│   │   ├── sma\_thac0\_lookup.json

│   │   └── wed\_dungeon\_generation.json

│   └── examples/

│       └── message\_flows.md

├── shared/

│   ├── dma\_shared\_resilience/

│   │   ├── \_\_init\_\_.py

│   │   ├── circuit\_breaker.py

│   │   ├── bulkhead.py

│   │   └── retry.py

│   ├── dma\_llm\_gateway/

│   │   ├── \_\_init\_\_.py

│   │   ├── client.py

│   │   ├── prompt\_cache.py

│   │   └── token\_meter.py

│   └── dma\_common/

│       ├── \_\_init\_\_.py

│       ├── validators.py

│       ├── constants.py

│       └── models.py

└── tests/

    ├── contracts/

    │   ├── test\_bus\_messages.py

    │   ├── test\_agent\_messages.py

    │   └── test\_schema\_evolution.py

    ├── golden/

    │   ├── test\_scribe\_golden.py

    │   ├── test\_sma\_golden.py

    │   └── test\_wed\_golden.py

    └── integration/

        └── test\_agent\_flows.py

## **Shared Components Implementation**

### **1\. DMA Shared Resilience Package**

\# shared/dma\_shared\_resilience/\_\_init\_\_.py

"""

DMA Shared Resilience Library

Provides circuit breakers, bulkheads, and retry logic for all agents

"""

from .circuit\_breaker import CircuitBreaker

from .bulkhead import Bulkhead

from .retry import RetryWithBackoff

\_\_version\_\_ \= "1.0.0"

\_\_all\_\_ \= \["CircuitBreaker", "Bulkhead", "RetryWithBackoff"\]

\# shared/dma\_shared\_resilience/circuit\_breaker.py

import time

import asyncio

from enum import Enum

from typing import Callable, Optional

from functools import wraps

class CircuitState(Enum):

    CLOSED \= "closed"

    OPEN \= "open"

    HALF\_OPEN \= "half\_open"

class CircuitBreaker:

    """

    Circuit breaker pattern implementation for fault tolerance

    """

    def \_\_init\_\_(self, 

                 failure\_threshold: int \= 5,

                 recovery\_timeout: int \= 60,

                 expected\_exception: type \= Exception):

        self.failure\_threshold \= failure\_threshold

        self.recovery\_timeout \= recovery\_timeout

        self.expected\_exception \= expected\_exception

        self.failure\_count \= 0

        self.last\_failure\_time: Optional\[float\] \= None

        self.state \= CircuitState.CLOSED

    

    def \_\_call\_\_(self, func: Callable) \-\> Callable:

        @wraps(func)

        async def wrapper(\*args, \*\*kwargs):

            if self.state \== CircuitState.OPEN:

                if self.\_should\_attempt\_reset():

                    self.state \= CircuitState.HALF\_OPEN

                else:

                    raise Exception(f"Circuit breaker is OPEN for {func.\_\_name\_\_}")

            

            try:

                result \= await func(\*args, \*\*kwargs)

                self.\_on\_success()

                return result

            except self.expected\_exception as e:

                self.\_on\_failure()

                raise

        

        return wrapper

    

    def \_should\_attempt\_reset(self) \-\> bool:

        return (

            self.last\_failure\_time and 

            time.time() \- self.last\_failure\_time \>= self.recovery\_timeout

        )

    

    def \_on\_success(self):

        self.failure\_count \= 0

        self.state \= CircuitState.CLOSED

    

    def \_on\_failure(self):

        self.failure\_count \+= 1

        self.last\_failure\_time \= time.time()

        if self.failure\_count \>= self.failure\_threshold:

            self.state \= CircuitState.OPEN

\# shared/dma\_shared\_resilience/retry.py

import asyncio

import random

from typing import Callable, Type, Tuple

from functools import wraps

class RetryWithBackoff:

    """

    Retry with exponential backoff and jitter

    """

    def \_\_init\_\_(self,

                 max\_attempts: int \= 3,

                 base\_delay: float \= 1.0,

                 max\_delay: float \= 60.0,

                 exceptions: Tuple\[Type\[Exception\], ...\] \= (Exception,)):

        self.max\_attempts \= max\_attempts

        self.base\_delay \= base\_delay

        self.max\_delay \= max\_delay

        self.exceptions \= exceptions

    

    def \_\_call\_\_(self, func: Callable) \-\> Callable:

        @wraps(func)

        async def wrapper(\*args, \*\*kwargs):

            attempt \= 0

            while attempt \< self.max\_attempts:

                try:

                    return await func(\*args, \*\*kwargs)

                except self.exceptions as e:

                    attempt \+= 1

                    if attempt \>= self.max\_attempts:

                        raise

                    

                    \# Exponential backoff with jitter

                    delay \= min(

                        self.base\_delay \* (2 \*\* (attempt \- 1)),

                        self.max\_delay

                    )

                    jitter \= random.uniform(0, delay \* 0.1)

                    await asyncio.sleep(delay \+ jitter)

        

        return wrapper

### **2\. LLM Gateway Service**

\# shared/dma\_llm\_gateway/client.py

"""

Centralized LLM client with caching and metering

"""

import hashlib

import time

from typing import Dict, Optional, Any

from dataclasses import dataclass

@dataclass

class LLMResponse:

    content: str

    tokens\_used: int

    model: str

    cached: bool \= False

class LLMGateway:

    """

    Centralized LLM access with token metering and caching

    """

    def \_\_init\_\_(self, 

                 api\_keys: Dict\[str, str\],

                 cache\_ttl: int \= 3600,

                 token\_limit\_per\_minute: int \= 10000):

        self.api\_keys \= api\_keys

        self.cache \= {}  \# In production, use Redis

        self.cache\_ttl \= cache\_ttl

        self.token\_meter \= TokenMeter(token\_limit\_per\_minute)

        

    async def generate(self,

                      prompt: str,

                      model: str \= "gpt-4",

                      max\_tokens: int \= 1000,

                      temperature: float \= 0.7) \-\> LLMResponse:

        """

        Generate text with caching and metering

        """

        \# Check cache first

        cache\_key \= self.\_get\_cache\_key(prompt, model, temperature)

        cached\_response \= self.\_get\_from\_cache(cache\_key)

        if cached\_response:

            return cached\_response

        

        \# Check token limits

        if not self.token\_meter.can\_proceed(max\_tokens):

            raise Exception("Token rate limit exceeded")

        

        \# Make API call

        response \= await self.\_call\_llm(prompt, model, max\_tokens, temperature)

        

        \# Update meter

        self.token\_meter.record\_usage(response.tokens\_used)

        

        \# Cache response

        self.\_cache\_response(cache\_key, response)

        

        return response

    

    def \_get\_cache\_key(self, prompt: str, model: str, temperature: float) \-\> str:

        """Generate deterministic cache key"""

        content \= f"{prompt}|{model}|{temperature}"

        return hashlib.sha256(content.encode()).hexdigest()

    

    def \_get\_from\_cache(self, key: str) \-\> Optional\[LLMResponse\]:

        """Retrieve from cache if not expired"""

        if key in self.cache:

            entry \= self.cache\[key\]

            if time.time() \- entry\['timestamp'\] \< self.cache\_ttl:

                response \= entry\['response'\]

                response.cached \= True

                return response

        return None

class TokenMeter:

    """Track and limit token usage"""

    def \_\_init\_\_(self, tokens\_per\_minute: int):

        self.tokens\_per\_minute \= tokens\_per\_minute

        self.usage\_window \= \[\]  \# (timestamp, tokens) tuples

    

    def can\_proceed(self, requested\_tokens: int) \-\> bool:

        """Check if request can proceed within rate limits"""

        self.\_clean\_old\_entries()

        current\_usage \= sum(tokens for \_, tokens in self.usage\_window)

        return current\_usage \+ requested\_tokens \<= self.tokens\_per\_minute

    

    def record\_usage(self, tokens: int):

        """Record token usage"""

        self.usage\_window.append((time.time(), tokens))

    

    def \_clean\_old\_entries(self):

        """Remove entries older than 1 minute"""

        cutoff \= time.time() \- 60

        self.usage\_window \= \[

            (ts, tokens) for ts, tokens in self.usage\_window 

            if ts \> cutoff

        \]

### **3\. Common Data Models**

\# shared/dma\_common/models.py

"""

Shared Pydantic models for cross-agent communication

"""

from pydantic import BaseModel, Field, validator

from typing import Optional, List, Dict, Any

from uuid import UUID

from datetime import datetime

class BaseEntity(BaseModel):

    """Base class for all CKB entities"""

    id: UUID

    type: str

    name: str \= Field(..., max\_length=255)

    created\_at: datetime \= Field(default\_factory=datetime.utcnow)

    updated\_at: datetime \= Field(default\_factory=datetime.utcnow)

    version: int \= 1

    

    class Config:

        json\_encoders \= {

            datetime: lambda v: v.isoformat(),

            UUID: lambda v: str(v)

        }

class NPC(BaseEntity):

    """Non-Player Character entity"""

    type: str \= "npc"

    attributes: Dict\[str, Any\] \= Field(default\_factory=dict)

    relationships: List\['Relationship'\] \= Field(default\_factory=list)

    

    \# 2e specific fields

    character\_class: Optional\[str\]

    level: Optional\[int\] \= Field(None, ge=0, le=20)

    alignment: Optional\[str\]

    hit\_points: Optional\[int\]

    armor\_class: Optional\[int\]

    thac0: Optional\[int\]

class Location(BaseEntity):

    """Location entity"""

    type: str \= "location"

    location\_type: str  \# dungeon, town, wilderness

    size: str \= Field(..., regex="^(small|medium|large)$")

    description: str

    contents: List\[UUID\] \= Field(default\_factory=list)

    

class Relationship(BaseModel):

    """Relationship between entities"""

    source\_id: UUID

    target\_id: UUID

    relationship\_type: str

    strength: Optional\[float\] \= Field(None, ge=-1.0, le=1.0)

    

class TaskResult(BaseModel):

    """Standard task result format"""

    status: str \= Field(..., regex="^(success|error|partial)$")

    entities\_created: int \= 0

    entities\_updated: int \= 0

    errors: List\[Dict\[str, str\]\] \= Field(default\_factory=list)

    performance\_metrics: Dict\[str, float\] \= Field(default\_factory=dict)

### **4\. Contract Test Implementation**

\# tests/contracts/test\_agent\_messages.py

import pytest

import json

from pathlib import Path

from jsonschema import validate, ValidationError, RefResolver

class TestAgentContracts:

    """Test all agent message contracts"""

    

    @classmethod

    def setup\_class(cls):

        """Load all schemas once"""

        schema\_dir \= Path("agent\_contracts/schemas")

        cls.schemas \= {}

        

        for schema\_file in schema\_dir.glob("\*.json"):

            with open(schema\_file) as f:

                cls.schemas\[schema\_file.stem\] \= json.load(f)

        

        \# Create resolver for $ref resolution

        cls.resolver \= RefResolver(

            base\_uri=f"file://{schema\_dir.absolute()}/",

            referrer=cls.schemas\["bus-message.v1"\]

        )

    

    @pytest.mark.parametrize("message\_file", \[

        "scribe\_store\_entity.json",

        "scribe\_search\_query.json",

        "sma\_thac0\_calc.json",

        "wed\_dungeon\_gen.json"

    \])

    def test\_valid\_messages(self, message\_file):

        """Test valid message examples"""

        with open(f"tests/fixtures/valid/{message\_file}") as f:

            message \= json.load(f)

        

        \# Determine which schema to use

        if "scribe" in message\_file:

            schema \= self.schemas\["scribe-request.v1"\]

        elif "sma" in message\_file:

            schema \= self.schemas\["sma-request.v1"\]

        elif "wed" in message\_file:

            schema \= self.schemas\["wed-request.v1"\]

        

        \# Should not raise

        validate(message, schema, resolver=self.resolver)

    

    def test\_schema\_evolution(self):

        """Ensure schemas are backward compatible"""

        \# Load v1 message

        with open("tests/fixtures/v1\_message.json") as f:

            v1\_message \= json.load(f)

        

        \# Should still validate against current schema

        validate(v1\_message, self.schemas\["bus-message.v1"\])

### **5\. Golden File Tests**

\# tests/golden/test\_sma\_golden.py

import json

from pathlib import Path

from agents.system\_mastery import SystemMasteryAgent

class TestSMAGoldenFiles:

    """Test System Mastery Agent against golden files"""

    

    @classmethod

    def setup\_class(cls):

        cls.sma \= SystemMasteryAgent()

        cls.golden\_dir \= Path("agent\_contracts/golden")

    

    def test\_thac0\_calculations(self):

        """Test THAC0 calculations match golden files"""

        with open(self.golden\_dir / "sma\_thac0\_golden.json") as f:

            test\_cases \= json.load(f)

        

        for test in test\_cases\["thac0\_tests"\]:

            result \= self.sma.calculate\_thac0(

                character\_class=test\["input"\]\["class"\],

                level=test\["input"\]\["level"\]

            )

            

            assert result \== test\["expected"\], (

                f"THAC0 mismatch for {test\['input'\]\['class'\]} "

                f"level {test\['input'\]\['level'\]}: "

                f"expected {test\['expected'\]}, got {result}"

            )

    

    def test\_saving\_throws(self):

        """Test saving throw calculations"""

        with open(self.golden\_dir / "sma\_saves\_golden.json") as f:

            test\_cases \= json.load(f)

        

        for test in test\_cases\["save\_tests"\]:

            result \= self.sma.calculate\_save(

                save\_type=test\["input"\]\["type"\],

                character\_class=test\["input"\]\["class"\],

                level=test\["input"\]\["level"\],

                modifiers=test\["input"\].get("modifiers", {})

            )

            

            assert result \== test\["expected"\]

### **6\. Integration Test Example**

\# tests/integration/test\_agent\_flows.py

import asyncio

import pytest

from unittest.mock import AsyncMock

from agents.orchestrator import Orchestrator

class TestAgentIntegration:

    """Test complete agent workflows"""

    

    @pytest.mark.asyncio

    async def test\_npc\_creation\_flow(self, redis\_mock, postgres\_mock):

        """Test creating an NPC through full agent flow"""

        orchestrator \= Orchestrator(

            redis\_url="redis://localhost",

            db\_url="postgresql://localhost/test"

        )

        

        \# Mock external dependencies

        orchestrator.llm\_gateway \= AsyncMock()

        orchestrator.llm\_gateway.generate.return\_value \= {

            "content": "A mysterious wizard",

            "tokens\_used": 50

        }

        

        \# Submit NPC creation request

        request \= {

            "prompt": "Create a level 7 wizard NPC named Elara",

            "context": {

                "campaign\_id": "test-campaign",

                "session\_id": "test-session"

            }

        }

        

        result \= await orchestrator.process\_request(request)

        

        \# Verify flow

        assert result\["status"\] \== "success"

        assert result\["entities\_created"\] \== 1

        assert "npc" in result\["created\_entities"\]\[0\]\["type"\]

        

        \# Verify agent communications

        assert redis\_mock.xadd.call\_count \>= 3  \# At least 3 messages

        

        \# Verify data persistence

        assert postgres\_mock.execute.called

## **Deployment Checklist**

### **Pre-deployment Validation**

\#\!/bin/bash

\# scripts/validate\_deployment.sh

echo "Running pre-deployment validation..."

\# 1\. Validate all schemas

echo "Validating JSON schemas..."

ajv compile \-s agent\_contracts/schemas/\*.json

\# 2\. Run contract tests

echo "Running contract tests..."

pytest tests/contracts \-v

\# 3\. Check golden files

echo "Validating golden files..."

pytest tests/golden \-v

\# 4\. Verify shared packages

echo "Testing shared packages..."

cd shared/dma\_shared\_resilience && python \-m pytest

cd ../dma\_llm\_gateway && python \-m pytest

cd ../dma\_common && python \-m pytest

\# 5\. Integration tests

echo "Running integration tests..."

docker-compose \-f docker-compose.test.yml up \--abort-on-container-exit

echo "Validation complete\!"

## **Monitoring Configuration**

\# monitoring/alerts.yml

groups:

  \- name: agent\_health

    rules:

      \- alert: AgentDown

        expr: up{job="dma\_agent"} \== 0

        for: 5m

        annotations:

          summary: "Agent {{ $labels.agent\_name }} is down"

      

      \- alert: HighErrorRate

        expr: rate(agent\_errors\_total\[5m\]) \> 0.1

        for: 10m

        annotations:

          summary: "High error rate for {{ $labels.agent\_name }}"

      

      \- alert: SlowResponse

        expr: histogram\_quantile(0.95, agent\_response\_time\_seconds) \> 2

        for: 15m

        annotations:

          summary: "95th percentile response time \> 2s"

This completes the comprehensive agent specification rewrite with machine-readable contracts, shared components, and a complete testing framework. The focus on three core agents for MVC ensures deliverability while maintaining extensibility for future phases.  
