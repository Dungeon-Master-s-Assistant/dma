**doc\_id**: [ADR‑004‑orchestrator](https://docs.google.com/document/u/0/d/1olOId6qBqslfubiGM8dpORdcwIs29rqvj37vIeA62Jk/edit)  
**title**: Orchestrator Internal Design  
**status**: Review  
**version**: 0.3  
**last\_updated**: Jun 28, 2025  
**audience**: \["platform‑engineers","gameplay‑engineers","QA","SRE"\]  
**revision\_history**:  
  \- {version: 0.1, date: 2025‑06‑28, author: JP Dow, note: "initial draft"}  
  \- {version: 0.2, date: 2025‑07‑01, author: gemini\_team,  note: "peer‑review edits"}  
  \- {version: 0.3, date: 2025‑07‑01, author: claude\_team,  note: "gameplay alignment, CKB integration"}  
**related\_docs**:  
  \- [DMA Technical Design v2](https://docs.google.com/document/u/0/d/1En98LlZxITx5PTQ8ilkqhBPJIlIhGdJwDdgIs3ytac4/edit)  
  \- [DMA API Design Spec v2](https://docs.google.com/document/u/0/d/1by14LYfZXvpmzv8wAKMWDkIfcsqi9QD95Rs4xWHi_5Y/edit)  
  \- [CKB Data Schema v2](https://docs.google.com/document/u/0/d/1_Krw8N8Tmmp8mitBLapVzO575TMh4WkNjUjaaam4KHw/edit)  
  \- [DMA Plan v2](https://docs.google.com/document/u/0/d/1OJDaiNjW1WlIdukKxddC-pAtSqMyASZ_hwzYZ16rku0/edit)  
---

1. ### **Executive Summary**

   **Abstract**   
   The Orchestrator is the single async ingress for all Dungeon Master (DM)‑initiated requests.  
   It enriches each request with gameplay context (campaign, session, DM preferences), decomposes it  
   into agent‑level tasks, enforces rate‑limit and resilience rules, aggregates results,  
   validates lore consistency, updates the Central Knowledge Base (CKB), and streams the final  
   response back to the UI. Two latency classes are supported:  
   **Fast Lane (\< 0.5 s)** for rule look‑ups and **Live Play (\< 2 s)** for reactive generation,  
   while longer “Prep” and “Bulk” jobs run fully async.

2. ### **Topology Diagram** (Mermaid)

   graph TD  
       subgraph UI Tier  
           UI\[DM Interface\]  
       end  
       subgraph Orchestrator  
           A\[/REST /v1/prompts/\]  
           B\[/Fast Lane /v1/fast/\*/\]  
           ORC\[Orchestrator (core)\]  
           FL\[Fast‑Lane Router\]  
           CE\[Context Enrichment\<br/\>(Session \+ Campaign)\]  
       end  
       subgraph Bus & Agents  
           RS((Redis Streams))  
           SCR\[Scribe Agent\]  
           SMA\[System Mastery Agent\]  
           WED\[World Designer Agent\]  
       end  
       subgraph Data Stores  
           PG\[(PostgreSQL CKB)\]  
           RC\[(In‑Mem Rule Cache)\]  
       end  
     
       UI \--\>|REST JSON| A \--\> ORC  
       UI \--\>|REST| B \--\> FL \--\> RC  
       ORC \--\> CE \--\> |XADD agent\_tasks| RS  
       RS \--\> SCR & SMA & WED  
       SCR & SMA & WED \--\>|XADD agent\_results| ORC  
       ORC \--\> PG  
   

   #### **Gameplay Content Enrichment Pipeline**

   Before any task decomposition the Orchestrator calls \*\*\`SessionContextExtractor\`\*\*  
   to attach:  
     
   | Field | Source | Notes |  
   |-------|--------|-------|  
   | \`campaign\_id\` | JWT custom claim \`cmp\` | Required |  
   | \`session\_id\`  | Header \`X-DMA-Session\` (optional) | Live‑play only |  
   | \`dm\_preferences\` | cache(\`dm\_prefs:{dm\_id}\`) | TTL \= 10 min |  
   | \`current\_game\_state\` | CKB read via \`CampaignStateManager.get\_current\_session\_context()\` | Round‑trip budget ≤ 50 ms |  
     
   If lookup fails the request is rejected with \`422\` \*\*“missing campaign context”\*\*.    
   

3. ### **Process Model & Concurrency Strategy**

| Sub‑loop | Concurrency primitive | Typical fan‑out | Notes |
| ----- | ----- | ----- | ----- |
| FastAPI request loop | `asyncio` tasks | ≤ 1 K req/s | CPU‑light, I/O bound |
| Intent classifier pool | `ThreadPoolExecutor(max_workers=4)` | 4 | Offloads HuggingFace pipeline to threads |
| Redis Streams consumers | N × `asyncio` tasks (default = 6) | 6 | One consumer group per agent |
| Result aggregation | `asyncio.Queue` | bounded (1 K) | Back‑pressure buffer |
| Metrics flushing | Background task every 5 s | 1 | Push to Prometheus gateway |

   **Design rationale**: keep the whole service **single‑process async** to avoid cross‑process context switching while still allowing blocking ML inference to run in a confined thread pool. 

4. ### **Internal Subsystems**

   | Subsystem | Responsibility | Key Classes / Files |

   |-----------|----------------|---------------------|

   | \*\*API Layer\*\* | Validate & auth REST \+ Fast Lane | \`entrypoints/http.py\` |

   | \*\*Context Enrichment\*\* | Inject campaign/session state & DM prefs | \`services/context.py\` |

   | \*\*Intent Classifier\*\* | Rule‑table lookup → local DeBERTa fallback | \`services/intent.py\` |

   | \*\*Task Decomposer\*\* | Prompt \+ intent ⇒ \`AgentTask\[\]\` | \`services/decompose.py\` |

   | \*\*Dispatcher\*\* | XADD to \`agent\_tasks\`, attach \*work‑token\* | \`services/dispatcher.py\` |

   | \*\*Result Collector\*\* | Consume \`agent\_results\`, merge, timeout logic | \`services/collector.py\` |

   | \*\*LoreConsistencyEngine\*\* | Cross‑check agent outputs vs CKB | \`services/lore\_validator.py\` |

   | \*\*CampaignStateManager\*\* | Persist net changes to CKB | \`services/state\_manager.py\` |

   | \*\*DMPreferenceManager\*\* | Fetch & cache DM overrides | \`services/prefs.py\` |

   | \*\*CreativeContentValidator\*\* | Detect narrative conflicts | \`services/creative\_validate.py\` |

   | \*\*Back‑Pressure Ctrl\*\* | 90 % util → 429 & metrics | \`services/backpressure.py\` |

   | \*\*Observability\*\* | Prometheus, OpenTelemetry, structlog | \`instrumentation/…\` |

   **Overflow policy:** when `result_queue` is full, new tasks are *not* accepted;

   Orchestrator returns **429 Retry‑After** where delay = `min(2×p99_latency, 30 s)`.

   This propagates back‑pressure to the UI tier.

   **Why single‑process async?** A fork/worker adds \~180 µs context‑switch latency;

   at 300 rps that is \~54 ms P95, violating the Fast Lane SLO.

5. ### **Sequence Diagram (happy path, prompt \-\> NPC generation)**

   sequenceDiagram  
       participant UI  
       participant ORC as Orchestrator  
       participant SCR as Scribe Agent  
       participant SMA as System Mastery  
       participant WED as World Designer  
       rect rgb(245,245,245)  
           UI-\>\>+ORC: POST /v1/prompts (Create NPC...)  
           ORC-\>\>ORC: deduplicate(request\_id)  
           ORC-\>\>ORC: classify\_intent()  
           ORC-\>\>ORC: decompose\_tasks()  
       end  
       ORC--\>\>SCR: XADD agent\_tasks (store entity)  
       ORC--\>\>SMA: XADD agent\_tasks (validate THAC0)  
       ORC--\>\>WED: XADD agent\_tasks (generate description)  
       SCR--\>\>ORC: XADD agent\_results (success)  
       SMA--\>\>ORC: XADD agent\_results (success)  
       WED--\>\>ORC: XADD agent\_results (success)  
       ORC-\>\>ORC: aggregate\_results()  
       ORC--\>\>UI: 200 OK (final content, entity\_ids)  
     
   sequenceDiagram  
       participant ORC  
       participant WED  
       participant LORE as LoreValidator  
       participant UI  
     
       ORC-\>\>WED: XADD task.generate\_content  
       WED--\>\>ORC: XADD result.success (draft lore)  
       ORC-\>\>LORE: validate\_lore\_consistency()  
       LORE--\>\>ORC: conflict\_found  
       ORC--\>\>UI: 409 Conflict (requires DM decision)  
       Note over ORC,LORE: Result moved to \<agent\_results:dlq:lore\_conflict\>

   Retries and DLQ flows follow the simplified event schema in the protocol spec.

6. ### **Data Model Snippets**

   \-- Idempotency & tracking  
   CREATE TABLE orchestrator\_requests (  
     request\_id UUID PRIMARY KEY,  
     status       TEXT CHECK (status IN ('processing','completed','error')),  
     prompt       TEXT,  
     user\_id      UUID,  
     created\_at   TIMESTAMPTZ DEFAULT now(),  
     completed\_at TIMESTAMPTZ,  
     result\_json  JSONB  
   );  
     
   \-- Fast‑lane cache (warm on startup)  
   CREATE UNLOGGED TABLE rule\_cache (  
     rule\_key TEXT PRIMARY KEY,  
     payload  JSONB,  
     updated\_at TIMESTAMPTZ  
   );  
     
   The `rule_cache` table is loaded into an in‑memory dict at startup and refreshed every 6 h.   
     
   ALTER TABLE orchestrator\_requests  
     ADD COLUMN latency\_class TEXT CHECK (latency\_class IN ('fast','live','prep','bulk')) DEFAULT 'live';  
     
   \-- partial index to speed dashboard queries  
   CREATE INDEX idx\_orc\_req\_campaign ON orchestrator\_requests(campaign\_id)  
     WHERE status \= 'completed';  
     
   `latency_class` is filled by Context Enrichment based on request type; used by SLO dashboards.

7. ### **Internal Message Bus**

   {  
     "message\_type": "generate\_npc",  
     "campaign\_id": "cmp\_123",  
     "session\_id": "ses\_456",  
     "dm\_preferences": { "tone": "heroic", "house\_rules": \["max\_hp\_first\_level"\] },  
     "session\_context": {  
       "current\_location": "Tavern of the Weary Traveler",  
       "party\_level": 5,  
       "active\_npcs": \["Gareth the Innkeeper"\]  
     },  
     "requirements": {  
       "npc\_role": "mysterious\_stranger",  
       "personality\_traits": \["secretive", "knowledgeable"\],  
       "plot\_relevance": "knows\_about\_artifact"  
     }  
   }

8. ### **Resilience & Back-Pressure Rules**

* **Circuit Breakers** wrap outbound LLM calls (`llm_gateway.generate`) and each Redis consumer loop; open after 5 failures / 60 s cool‑down.  
* **Bulkhead** limits max in‑flight tasks per agent to 50; excess tasks stay in in‑memory queue.  
* **Retry With Jitter**: exponential backoff, 3 attempts on network / timeout errors only.  
* **Load‑Shed**: if `queue_depth > 9 000` or `p99_request_latency > 5 s`, return `429` with `Retry‑After`.  
* Built using the shared package `dma_shared_resilience` so that semantics stay consistent across agents. DMA Agent Contract Test…  
* CircuitBreaker counts `TimeoutError | ConnectionError | LLMError`.  
* Bulkhead buffer capacity \= **2 000**; overflow → orchestrator queue.  
* RetryWithBackoff budget: *3 attempts or 15 s total, whichever first*.  
* `Retry‑After` \= `ceil(p99_latency × 2)` seconds, capped at **30**.

9. ### **Sizing Guide (reference, local MVC)**

| Resource | Default | Rationale |
| ----- | ----- | ----- |
| FastAPI workers | 1 (async) | Avoid GIL contention, driven by asyncio |
| Thread pool (classifier) | 4 | DeBERTa‑small \< 250 MB RAM |
| Redis connection pool | 20 | 6 consumers \+ spikes |
| PostgreSQL pool | 20 | 10 read, 10 write |
| In‑memory result buffer | 1 024 requests | Worst‑case 3‑s spike at 300 rps |
| CPU / RAM | 2 vCPU / 4 GB | Fits laptop docker‑compose |

These numbers should be load‑tested and adjusted before cloud alpha. 

| Tier | Thread‑pool | Max latency | Notes |
| ----- | ----- | ----- | ----- |
| Fast Lane | N/A (sync lookup) | 0.5 s p95 | In‑memory rule cache |
| Live Play | 4 | 2 s p95 | small content |
| Prep | 12 | 120 s | medium generation |
| Bulk | 12 | streamed ≤ 15 min | world scaffold |

10. ### **Container Entrypoint & Health**

    \# orchestrator/Dockerfile  
    FROM python:3.12-slim  
    WORKDIR /app  
    COPY . .  
    RUN pip install \-r requirements.txt  
    CMD \["uvicorn", "entrypoints.http:app", \\  
         "--host", "0.0.0.0", "--port", "8000", \\  
         "--loop", "uvloop", "--http", "httptools", \\  
         "--lifespan", "on"\]  
    HEALTHCHECK CMD curl \-f http://localhost:8000/health || exit 1  
      
* `/health` : returns 200 if circuit‑breakers are **CLOSED** and queue depth \< 80 %.  
* `/metrics` : Prometheus scrape endpoint.

  @app.post("/v1/prompts/{request\_id}/cancel", status\_code=202)

  async def cancel(request\_id: UUID, user=Depends(verify\_token)):

      await dispatcher.cancel(request\_id, user.id)

      return {"status": "canceled", "request\_id": str(request\_id)}

* *Agents monitor `control.cancel` stream and fast‑fail.*

11. ### **Reference Implementation Skeleton**

    \# entrypoints/http.py  
    from fastapi import FastAPI, Depends, BackgroundTasks  
    from services.intent import IntentClassifier  
    from services.decompose import decompose  
    from services.dispatcher import Dispatcher  
    from models import PromptRequest, PromptResponse  
      
    app \= FastAPI(title="DMA Orchestrator")  
      
    classifier \= IntentClassifier()  
    dispatcher \= Dispatcher()  
      
    @app.post("/v1/prompts", response\_model=PromptResponse)  
    async def handle\_prompt(req: PromptRequest,  
                            bg: BackgroundTasks,  
                            user=Depends(auth.verify\_token)):  
        if await request\_store.exists(req.request\_id):  
            return await request\_store.fetch(req.request\_id)   \# idempotency  
      
        intent \= await classifier.classify(req.prompt)  
        tasks  \= decompose(req, intent)  
        bg.add\_task(dispatcher.enqueue, req.request\_id, tasks)  
      
        return PromptResponse(request\_id=req.request\_id,  
                              status="processing",  
                              poll\_url=f"/v1/prompts/{req.request\_id}")  
      
    Full project scaffold (`src/`) is included in the attached ADR branch. 

12. ### **Implementation Checklist (two-week sprint)**

| Day | Deliverable | Owner | Depends On |
| ----- | ----- | ----- | ----- |
| 1 – 2 | Scaffold FastAPI project, env config | Platform |  |
| 3 – 4 | Implement `IntentClassifier` \+ unit tests | ML |  |
| 4 | SessionContextExtractor, DMPreference cache | Platform | Auth |
| 5 – 6 | Build `Dispatcher` & `Collector`; local Redis Streams up | Platform |  |
| 6 | JSON Schema update incl. `campaign_id` | QA |  |
| 7 | PostgreSQL `orchestrator_requests` table, repository | DB |  |
| 8 | CKB write path in `CampaignStateManager` | DB | Day 4 |
| 8 | Back‑pressure & circuit‑breaker middleware | Platform |  |
| 9 | `/prompts/{id}/cancel` \+ `control.cancel` topic | Platform | Day 6 |
| 9 | `/health`, `/metrics`, basic dashboards | DevOps |  |
| 10 | `Lore conflict DLQ consumer → DM alert` | WED \+ UX | Day 8 |
| 10 | Load & smoke tests (\< 300 rps, P95 \< 120 ms Fast‑Lane) | QA |  |
| 11 | ADR review, code freeze | Architecture |  |
| 12 | Sprint demo & retro | All |  |

13. ### **Open Questions (flagged for ADRs)**

    ADR‑008 – Gameplay Context Layer    
    ADR‑009 – Lore Validation vs Intermediate Persistence

    1. Do we persist **intermediate task state** for long‑running flows, or only final aggregate?  
    2. What is the cut‑over policy when **Fast‑Lane** cache is stale but DB is slow (serve last‑known good vs. fail‐open)?  
    3. Should **token metering** be enforced here or solely in the shared LLM gateway?

    ### **12\. Next Steps for the Cross-Cutting Section**

* Convert this design into an **ADR‑004‑orchestrator‑internal.md** file in `/docs/adr/`.  
* Produce **state‑machine diagrams** for each agent (Scribe, SMA, WED) that align with this dispatch model.  
* Draft the **Back‑Pressure Policy document** describing thresholds and operator levers.

	Once those are merged, the Orchestrator epic is formally **“Ready for Implementation.”**