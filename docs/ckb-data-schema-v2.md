# **Central Knowledge Base (CKB) \- Streamlined Data Schema v2.0**

## **DMA System Data Architecture \- MVC Focus**

---

## **Table of Contents**

1. [Architecture Overview](#architecture-overview)  
2. [Core Entity Schemas](#core-entity-schemas)  
3. [Relationship Model](#relationship-model)  
4. [Metadata & Versioning](#metadata--versioning)  
5. [Validation & Constraints](#validation--constraints)  
6. [API Patterns](#api-patterns)  
7. [Implementation Guide](#implementation-guide)

---

## **Architecture Overview**

### **Storage Strategy**

- **Primary Database**: PostgreSQL 15+ with JSONB columns  
- **Extensions**: pgvector for future semantic search, pg\_trgm for text search  
- **Phase 2 Consideration**: Add Neo4j only when graph traversals exceed PostgreSQL recursive CTE performance limits

### **Design Principles**

1. **Core vs Extended**: Every entity has a minimal "Core Slice" (required) and "Extended Data" (optional)  
2. **Progressive Enhancement**: Start simple, add complexity through versioned migrations  
3. **Single Source of Truth**: One database, one schema, clear ownership

### **Schema Version**

- **Current**: 2.0.0 (MVC Release)  
- **Migration Strategy**: Flyway or Alembic for version control  
- **Compatibility**: Forward-compatible design for future extensions

---

## **Core Entity Schemas**

### **Base Entity Pattern**

All entities inherit this base structure:

interface BaseEntity {

  // Core Slice \- Always Required

  id: string;           // UUID v4

  entity\_type: string;  // campaign|location|character|item|faction|plotline|encounter|session

  name: string;         // Max 100 chars

  description: string;  // Max 500 chars

  

  // Metadata \- Always Required

  campaign\_id: string;

  created\_at: timestamp;

  created\_by: string;

  updated\_at: timestamp;

  updated\_by: string;

  version: number;      // Optimistic locking

  status: 'active' | 'archived' | 'deleted';

  

  // Extended Data \- Optional JSONB

  extended\_data?: Record\<string, any\>;

  

  // Search & Filter

  tags: string\[\];       // Simple string array

  search\_vector?: tsvector; // PostgreSQL full-text search

}

### **1\. Campaign Entity**

interface Campaign extends BaseEntity {

  entity\_type: 'campaign';

  

  // Core Slice

  setting\_name: string;

  current\_date: string;  // In-game date

  

  // Extended Data Examples (in JSONB)

  extended\_data?: {

    genre?: string;

    tone?: string;

    house\_rules?: Array\<{name: string, description: string}\>;

    dm\_notes?: string;

  }

}

**SQL Table**:

CREATE TABLE campaigns (

  id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),

  name VARCHAR(100) NOT NULL,

  description TEXT,

  setting\_name VARCHAR(100) NOT NULL,

  current\_date VARCHAR(50),

  

  \-- Metadata

  created\_at TIMESTAMP NOT NULL DEFAULT CURRENT\_TIMESTAMP,

  created\_by UUID NOT NULL,

  updated\_at TIMESTAMP NOT NULL DEFAULT CURRENT\_TIMESTAMP,

  updated\_by UUID NOT NULL,

  version INTEGER NOT NULL DEFAULT 1,

  status VARCHAR(20) NOT NULL DEFAULT 'active',

  

  \-- Extended & Search

  extended\_data JSONB DEFAULT '{}',

  tags TEXT\[\] DEFAULT '{}',

  search\_vector TSVECTOR GENERATED ALWAYS AS (

    to\_tsvector('english', coalesce(name,'') || ' ' || coalesce(description,''))

  ) STORED,

  

  \-- Constraints

  CONSTRAINT campaigns\_status\_check CHECK (status IN ('active', 'archived', 'deleted'))

);

CREATE INDEX idx\_campaigns\_search ON campaigns USING GIN(search\_vector);

CREATE INDEX idx\_campaigns\_tags ON campaigns USING GIN(tags);

CREATE INDEX idx\_campaigns\_status ON campaigns(status) WHERE status \= 'active';

### **2\. Location Entity**

interface Location extends BaseEntity {

  entity\_type: 'location';

  

  // Core Slice

  location\_type: 'continent' | 'kingdom' | 'city' | 'town' | 'dungeon' | 'building' | 'room' | 'wilderness';

  parent\_location\_id?: string;  // Hierarchical relationship

  

  // Extended Data Examples

  extended\_data?: {

    population?: number;

    climate?: string;

    notable\_features?: string\[\];

    dm\_notes?: string;

    map\_coordinates?: {x: number, y: number};

  }

}

### **3\. Character Entity**

interface Character extends BaseEntity {

  entity\_type: 'character';

  

  // Core Slice

  char\_type: 'pc' | 'major\_npc' | 'minor\_npc' | 'monster';

  level: number;

  current\_location\_id?: string;

  alive: boolean;

  

  // Extended Data Examples

  extended\_data?: {

    race?: string;

    class?: string;

    alignment?: string;

    stats?: {

      str: number;

      dex: number;

      con: number;

      int: number;

      wis: number;

      cha: number;

    };

    personality?: string\[\];

    backstory?: string;

    portrait\_url?: string;

  }

}

### **4\. Item Entity**

interface Item extends BaseEntity {

  entity\_type: 'item';

  

  // Core Slice  

  item\_type: 'weapon' | 'armor' | 'consumable' | 'treasure' | 'quest\_item' | 'other';

  current\_owner\_id?: string;

  current\_location\_id?: string;

  quantity: number;

  

  // Extended Data Examples

  extended\_data?: {

    magical?: boolean;

    value\_gp?: number;

    weight\_lbs?: number;

    properties?: string\[\];

    dm\_notes?: string;

  }

}

### **5\. Simplified Entities**

For MVC, these entities use only the base pattern with minimal extensions:

- **Faction**: Just name, description, and influence level in extended\_data  
- **Plotline**: Title, status (active/complete), and current objective in extended\_data  
- **Encounter**: Name, difficulty rating, and participant IDs in extended\_data  
- **Session**: Session number, date, and summary in extended\_data

---

## **Relationship Model**

### **Foreign Key Approach (Phase 1\)**

Use traditional foreign keys for primary relationships:

\-- Direct relationships via columns

ALTER TABLE characters 

  ADD CONSTRAINT fk\_character\_location 

  FOREIGN KEY (current\_location\_id) 

  REFERENCES locations(id);

ALTER TABLE items

  ADD CONSTRAINT fk\_item\_owner

  FOREIGN KEY (current\_owner\_id)

  REFERENCES characters(id);

### **Relationship Table (Future Option)**

For complex many-to-many relationships:

CREATE TABLE entity\_relationships (

  id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),

  source\_id UUID NOT NULL,

  source\_type VARCHAR(50) NOT NULL,

  target\_id UUID NOT NULL,  

  target\_type VARCHAR(50) NOT NULL,

  relationship\_type VARCHAR(50) NOT NULL,

  properties JSONB DEFAULT '{}',

  created\_at TIMESTAMP NOT NULL DEFAULT CURRENT\_TIMESTAMP,

  

  CONSTRAINT unique\_relationship 

    UNIQUE(source\_id, target\_id, relationship\_type)

);

CREATE INDEX idx\_relationships\_source ON entity\_relationships(source\_id, source\_type);

CREATE INDEX idx\_relationships\_target ON entity\_relationships(target\_id, target\_type);

---

## **Metadata & Versioning**

### **Lightweight Versioning**

Use optimistic locking with version numbers:

\# Pydantic model example

class UpdateEntity(BaseModel):

    id: UUID

    version: int  \# Must match current version

    updates: Dict\[str, Any\]

    

\# Update logic

def update\_entity(update: UpdateEntity):

    result \= db.execute("""

        UPDATE entities 

        SET data \= %s, version \= version \+ 1

        WHERE id \= %s AND version \= %s

        """, (update.updates, update.id, update.version))

    

    if result.rowcount \== 0:

        raise ConcurrentModificationError()

### **Audit Trail (Simplified)**

Single audit table for all entity changes:

CREATE TABLE audit\_log (

  id BIGSERIAL PRIMARY KEY,

  entity\_id UUID NOT NULL,

  entity\_type VARCHAR(50) NOT NULL,

  operation VARCHAR(20) NOT NULL,

  user\_id UUID NOT NULL,

  timestamp TIMESTAMP NOT NULL DEFAULT CURRENT\_TIMESTAMP,

  changes JSONB,  \-- JSON diff of changes

  

  \-- Partitioned by month for performance

) PARTITION BY RANGE (timestamp);

---

## **Validation & Constraints**

### **Pydantic Models (Shared Package)**

\# dma\_shared\_models/entities.py

from pydantic import BaseModel, Field, validator

from typing import Optional, Literal

from uuid import UUID

from datetime import datetime

class LocationBase(BaseModel):

    name: str \= Field(..., min\_length=1, max\_length=100)

    description: str \= Field(..., max\_length=500)

    location\_type: Literal\[

        'continent', 'kingdom', 'city', 'town', 

        'dungeon', 'building', 'room', 'wilderness'

    \]

    parent\_location\_id: Optional\[UUID\] \= None

    

    @validator('name')

    def name\_not\_empty(cls, v):

        if not v.strip():

            raise ValueError('Name cannot be empty')

        return v.strip()

class CharacterBase(BaseModel):

    name: str \= Field(..., min\_length=1, max\_length=100)

    description: str \= Field(..., max\_length=500)

    char\_type: Literal\['pc', 'major\_npc', 'minor\_npc', 'monster'\]

    level: int \= Field(default=1, ge=0, le=20)

    alive: bool \= True

    

    class Config:

        \# Allow extended\_data field

        extra \= 'allow'

### **Database Constraints**

\-- Business rule constraints

ALTER TABLE characters 

  ADD CONSTRAINT valid\_level CHECK (level \>= 0 AND level \<= 20);

ALTER TABLE items

  ADD CONSTRAINT positive\_quantity CHECK (quantity \>= 0);

\-- Ensure referential integrity

ALTER TABLE campaigns

  ADD CONSTRAINT valid\_status CHECK (status IN ('active', 'archived', 'deleted'));

---

## **API Patterns**

### **RESTful Endpoints**

\# Core CRUD Operations

GET    /api/v1/entities/{type}          \# List with filters

GET    /api/v1/entities/{type}/{id}     \# Get single entity

POST   /api/v1/entities/{type}          \# Create new

PUT    /api/v1/entities/{type}/{id}     \# Update (with version)

DELETE /api/v1/entities/{type}/{id}     \# Soft delete

\# Search Operations  

POST   /api/v1/search                   \# Full-text search

GET    /api/v1/entities/{type}/{id}/related  \# Get relationships

\# Fast Lane Operations (bypass event bus)

GET    /api/v1/fast/lookup/{rule}       \# Rule lookups

POST   /api/v1/fast/calculate           \# Combat calculations

### **Query Examples**

\# FastAPI endpoint example

@app.get("/api/v1/entities/location")

async def list\_locations(

    campaign\_id: UUID,

    location\_type: Optional\[str\] \= None,

    tags: Optional\[List\[str\]\] \= Query(None),

    search: Optional\[str\] \= None,

    limit: int \= Query(100, le=1000),

    offset: int \= 0

):

    query \= """

        SELECT \* FROM locations 

        WHERE campaign\_id \= %s 

        AND status \= 'active'

    """

    params \= \[campaign\_id\]

    

    if location\_type:

        query \+= " AND location\_type \= %s"

        params.append(location\_type)

        

    if tags:

        query \+= " AND tags && %s"  \# Array overlap

        params.append(tags)

        

    if search:

        query \+= " AND search\_vector @@ plainto\_tsquery('english', %s)"

        params.append(search)

        

    query \+= " ORDER BY updated\_at DESC LIMIT %s OFFSET %s"

    params.extend(\[limit, offset\])

    

    return await db.fetch\_all(query, params)

---

## **Implementation Guide**

### **1\. Database Setup**

\-- Initial setup

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE EXTENSION IF NOT EXISTS "pg\_trgm";  \-- For fuzzy text search

\-- Create base tables

CREATE TABLE entities (

  id UUID PRIMARY KEY DEFAULT gen\_random\_uuid(),

  entity\_type VARCHAR(50) NOT NULL,

  campaign\_id UUID NOT NULL,

  name VARCHAR(100) NOT NULL,

  description TEXT,

  

  \-- Core fields based on type

  core\_data JSONB NOT NULL DEFAULT '{}',

  

  \-- Extended data

  extended\_data JSONB DEFAULT '{}',

  

  \-- Metadata

  created\_at TIMESTAMP NOT NULL DEFAULT CURRENT\_TIMESTAMP,

  created\_by UUID NOT NULL,

  updated\_at TIMESTAMP NOT NULL DEFAULT CURRENT\_TIMESTAMP,

  updated\_by UUID NOT NULL,

  version INTEGER NOT NULL DEFAULT 1,

  status VARCHAR(20) NOT NULL DEFAULT 'active',

  

  \-- Search

  tags TEXT\[\] DEFAULT '{}',

  search\_vector TSVECTOR,

  

  \-- Indexes

  CONSTRAINT entities\_status\_check CHECK (status IN ('active', 'archived', 'deleted'))

);

\-- Performance indexes

CREATE INDEX idx\_entities\_type\_campaign ON entities(entity\_type, campaign\_id);

CREATE INDEX idx\_entities\_search ON entities USING GIN(search\_vector);

CREATE INDEX idx\_entities\_tags ON entities USING GIN(tags);

CREATE INDEX idx\_entities\_extended ON entities USING GIN(extended\_data);

### **2\. Migration Strategy**

\# alembic/versions/001\_initial\_schema.py

def upgrade():

    \# Create tables

    op.create\_table('entities', ...)

    

    \# Add indexes

    op.create\_index('idx\_entities\_search', ...)

    

    \# Insert seed data

    op.bulk\_insert(entities\_table, \[

        {'entity\_type': 'rule', 'name': 'THAC0', ...},

        {'entity\_type': 'rule', 'name': 'Saving Throws', ...}

    \])

def downgrade():

    op.drop\_table('entities')

### **3\. Repository Pattern**

\# repositories/entity\_repository.py

class EntityRepository:

    def \_\_init\_\_(self, db: Database):

        self.db \= db

        

    async def create(self, entity: BaseEntity) \-\> UUID:

        """Create entity with automatic versioning"""

        query \= """

            INSERT INTO entities 

            (entity\_type, campaign\_id, name, description, 

             core\_data, extended\_data, created\_by, updated\_by, tags)

            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)

            RETURNING id

        """

        return await self.db.fetchval(query, 

            entity.entity\_type, entity.campaign\_id, entity.name,

            entity.description, entity.core\_data, entity.extended\_data,

            entity.created\_by, entity.updated\_by, entity.tags

        )

    

    async def get\_by\_id(self, entity\_id: UUID) \-\> Optional\[Dict\]:

        """Get with caching support"""

        \# Check cache first

        cached \= await redis.get(f"entity:{entity\_id}")

        if cached:

            return json.loads(cached)

            

        \# Database query

        query \= "SELECT \* FROM entities WHERE id \= $1 AND status \= 'active'"

        result \= await self.db.fetchrow(query, entity\_id)

        

        if result:

            \# Cache for 5 minutes

            await redis.setex(f"entity:{entity\_id}", 300, json.dumps(dict(result)))

            

        return dict(result) if result else None

### **4\. Performance Optimizations**

\# Connection pooling

async def create\_db\_pool():

    return await asyncpg.create\_pool(

        DATABASE\_URL,

        min\_size=10,

        max\_size=20,

        max\_inactive\_connection\_lifetime=300,

        command\_timeout=60

    )

\# Batch operations

async def bulk\_create\_entities(entities: List\[BaseEntity\]):

    """Efficient bulk insert"""

    async with db.transaction():

        await db.executemany("""

            INSERT INTO entities (entity\_type, name, ...) 

            VALUES ($1, $2, ...)

        """, \[(e.entity\_type, e.name, ...) for e in entities\])

### **5\. Testing Strategy**

\# tests/test\_entity\_repository.py

import pytest

from uuid import uuid4

@pytest.mark.asyncio

async def test\_create\_location(db):

    repo \= EntityRepository(db)

    

    location \= Location(

        name="The Broken Tower",

        description="A crumbling watchtower on the northern border",

        location\_type="building",

        campaign\_id=uuid4(),

        created\_by=uuid4(),

        updated\_by=uuid4()

    )

    

    entity\_id \= await repo.create(location)

    assert entity\_id is not None

    

    \# Verify retrieval

    saved \= await repo.get\_by\_id(entity\_id)

    assert saved\['name'\] \== "The Broken Tower"

    assert saved\['version'\] \== 1

---

## **Migration Path from v1.0**

1. **Week 1**: Set up PostgreSQL, create base schema  
2. **Week 2**: Implement entity repository and API endpoints  
3. **Week 3**: Add validation and testing  
4. **Week 4**: Performance optimization and indexing  
5. **Future**: Evaluate graph database needs based on actual query patterns

## **Key Improvements**

- **80% reduction** in required fields  
- **Single database** reduces operational complexity  
- **Progressive enhancement** allows gradual feature addition  
- **Shared validation** via Pydantic models  
- **Clear performance path** with PostgreSQL-first approach

This streamlined schema maintains all essential functionality while dramatically reducing complexity and improving maintainability for the MVC release.  
