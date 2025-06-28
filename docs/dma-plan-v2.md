# **Dungeon Master's Assistant (DMA) Plan v2**

**Version:** 2.0 (Strategic Engineering Revision)  
**Date:** June 27, 2025  
**Author:** J.P. Dow  
**Based on:** Strategic Engineering Critique & Original DMA Plan

---

## **Executive Summary & Strategic Focus**

The Dungeon Master's Assistant (DMA) is a multi-agent AI system designed to enhance Advanced Dungeons & Dragons 2nd Edition campaign management through intelligent automation and real-time assistance. This revised plan prioritizes **disciplined scope management**, **performance-first architecture**, and **rapid delivery of demonstrable value** based on strategic engineering feedback.

**Core Value Proposition:** Enable DMs to run richer, more immersive AD\&D 2e campaigns by automating routine tasks, maintaining lore consistency, and providing intelligent real-time assistance during live play.

**Success Metrics:**

- **Minimum Viable Campaign (MVC):** Complete a 3-hour one-shot with zero system crashes  
- **Performance Target:** \< 2s response time for 95% of live gameplay prompts  
- **User Satisfaction:** 80% DM satisfaction on weekly surveys by Q1 2026

---

## **1\. Scope & MVP Discipline**

### **Phase 1: Minimum Viable Campaign (MVC) \- Q3 2025**

**Disciplined Scope \- Only Essential Components:**

#### **Core Architecture (3 Components Only)**

1. **Orchestrator** \- Central coordination engine  
2. **Central Knowledge Base (CKB)** \- Single PostgreSQL instance  
3. **Fast Lane RPC Service** \- Direct latency-critical operations

#### **Essential Agents (3 Only)**

1. **Scribe & Archivist** \- Data management and retrieval  
2. **System Mastery** \- AD\&D 2e rules and mechanics  
3. **World & Encounter Designer** \- Basic content generation

#### **External Dependencies (Minimal)**

- **Single LLM Provider:** OpenAI GPT-4 (primary), Claude Sonnet 4 (backup)  
- **No integrations** in MVC (Foundry, World Anvil, etc. deferred to Phase 2\)  
- **Local deployment only** (Docker Compose)

### **Deferred to Later Phases**

- Storytelling & Narrative Controller  
- Visual Assets & Immersion Architect  
- Player Management & Psychology Agent  
- Performance & Atmosphere Agent  
- Tool & System Use Agent  
- Advanced Content Creation Agent  
- All external tool integrations  
- Cloud deployment  
- Multi-database engines

---

## **2\. Performance-First Architecture**

### **Latency Optimization Strategy**

#### **Fast Lane Architecture**

DM Input → Orchestrator → Fast Lane RPC → Response (\< 1s)

                     ↘ Event Bus → Agent Processing (async)

**Fast Lane Operations:**

- THAC0 calculations  
- Spell/item lookups  
- Basic rule checks  
- Combat mechanics

**Event Bus Operations:**

- Complex content generation  
- Lore consistency checks  
- Multi-agent coordination

#### **Response Time Error Budgets**

- **Critical Operations:** \< 500ms (combat mechanics, rule lookups)  
- **Standard Operations:** \< 2s (descriptions, basic generation)  
- **Complex Generation:** \< 10s (detailed encounters, lore synthesis)  
- **Background Tasks:** No limit (bulk content generation)

#### **Performance Monitoring**

- **Real-time Metrics:** E2E timing from UI keystroke to response  
- **Load Testing:** Simulated high-usage scenarios during development  
- **Performance Regression Tests:** Automated CI/CD performance gates

---

## **3\. Simplified Data Model**

### **Progressive Disclosure Data Strategy**

#### **Core Entities (Required Fields Only)**

{

  "id": "unique\_identifier",

  "name": "display\_name", 

  "type": "entity\_type",

  "visibility": "public|private",

  "content": "basic\_description"

}

#### **Extended Properties (Optional)**

- Relationships, detailed stats, rich descriptions added incrementally  
- UI enforces progressive disclosure \- start simple, add complexity as needed  
- **Soft validation** on core fields only

#### **Single Database Technology**

- **PostgreSQL** with JSONB for flexible schema evolution  
- **Vector extensions** for semantic search  
- **Graph queries** via recursive CTEs  
- Defer specialized graph databases (Neo4j) to Phase 2+

---

## **4\. Legal & Cost Management**

### **AD\&D 2e Content Strategy**

- **No copyrighted content** stored in system  
- **DM-provided PDFs** processed locally with OCR  
- **Derived tables only** (THAC0, saves, XP charts calculated independently)  
- **Open-source training** on publicly available SRD content only

### **LLM Cost Controls**

- **Central metering service** tracks token usage per session  
- **Usage dashboards** show real-time burn rate to DMs  
- **Pluggable provider abstraction** for easy model switching  
- **Cost alerts** at configurable spending thresholds

---

## **5\. Security & Authentication Foundation**

### **Security Architecture (Built from Day 1\)**

- **JWT-based service-to-service auth** for all internal APIs  
- **API key rotation** for external LLM providers  
- **Encrypted data at rest** (AES-256)  
- **TLS 1.3** for all network communication  
- **RBAC** for CKB access control

### **Secret Management**

- **HashiCorp Vault** or cloud-native secret management  
- **No hardcoded secrets** in containers  
- **Automated secret rotation** where supported

---

## **6\. Quality Assurance & Testing**

### **Automated Testing Pipeline**

#### **Schema Tests**

def test\_agent\_ckb\_roundtrip():

    \# Generate content → Store in CKB → Retrieve → Validate JSON Schema

    assert round\_trip\_data\_integrity()

#### **Rules Regression Tests**

def test\_combat\_mechanics\_consistency():

    \# Same combat scenario must yield same probabilities

    assert thac0\_calculation(scenario) \== expected\_result

#### **Performance Tests**

- **Load testing** simulating concurrent DM sessions  
- **Stress testing** with complex multi-agent workflows  
- **CI/CD performance gates** preventing regression

#### **LLM Output Validation**

- **Golden file tests** for consistent generation quality  
- **Schema validation** for all structured outputs  
- **Toxicity/safety screening** for generated content

---

## **7\. Observability & SLOs**

### **Service Level Objectives (SLOs)**

1. **Availability:** 99.5% uptime during scheduled play sessions  
2. **Latency:** 90th percentile response time \< 2s  
3. **Accuracy:** 95% rules lookups return correct results  
4. **Consistency:** 99% lore generation maintains CKB consistency

### **Monitoring Stack**

- **Prometheus** \- Metrics collection  
- **Grafana** \- Dashboards and alerting  
- **Jaeger** \- Distributed tracing  
- **ELK Stack** \- Centralized logging

### **Key Metrics**

- Request latency (p50, p90, p95, p99)  
- Error rates by service and endpoint  
- LLM token usage and cost  
- User engagement and satisfaction

---

## **8\. Human-in-the-Loop UX Design**

### **DM Control Mechanisms**

#### **Accept/Reject Interface**

┌─────────────────────────────────────┐

│ Agent Suggestion: \[Content Preview\] │

│ ┌─────────┐ ┌─────────┐ ┌─────────┐│

│ │ Accept  │ │ Edit    │ │ Reject  ││

│ └─────────┘ └─────────┘ └─────────┘│

│ Source: World Designer, 42 tokens  │

└─────────────────────────────────────┘

#### **Provenance Tracking**

- **Source attribution** for every generated element  
- **Edit history** with rollback capabilities  
- **Confidence scoring** from generating agents  
- **Bias controls** for agent behavior tuning

#### **Inline Editing**

- **Real-time collaboration** between DM and AI  
- **Suggestion modes** (conservative, balanced, creative)  
- **Template libraries** for common scenarios

---

## **9\. Delivery Milestones & Roadmap**

### **Q3 2025: MVC Delivery**

**Goal:** Run a complete 3-hour one-shot with zero crashes

**Deliverables:**

- Orchestrator with fast lane architecture  
- PostgreSQL CKB with core entity types  
- 3 essential agents (Scribe, System Mastery, World Designer)  
- Local Docker Compose deployment  
- Basic web UI with accept/reject controls

**Success Criteria:**

- All combat mechanics calculate correctly  
- Lore consistency maintained across session  
- \< 2s response time for 95% of operations

### **Q4 2025: Cloud Alpha**

**Goal:** 95% uptime with \< 2s latency for live prompts

**Deliverables:**

- Cloud deployment on single provider (AWS/GCP)  
- Enhanced UI with provenance tracking  
- Basic Foundry VTT integration  
- Automated testing pipeline

**Success Criteria:**

- 10 concurrent DM alpha testers  
- Performance SLOs consistently met  
- Positive user feedback (\>70% satisfaction)

### **Q1 2026: Feature Expansion**

**Goal:** 80% DM satisfaction on weekly surveys

**Deliverables:**

- Additional agents (Storytelling, Visual Assets)  
- External tool integrations (World Anvil, Notion)  
- Advanced customization options  
- Performance optimizations

**Success Criteria:**

- 100+ active DMs using the system  
- 80% user satisfaction rating  
- Sub-second response times for critical operations

---

## **10\. Implementation Strategy**

### **Development Approach**

1. **Sprint Duration:** 2-week sprints  
2. **Team Structure:** Small, cross-functional teams (2-3 engineers per component)  
3. **Definition of Done:** All features must pass automated tests \+ manual QA  
4. **Technical Debt:** 20% of each sprint reserved for refactoring

### **Risk Mitigation**

- **Weekly architecture reviews** to prevent scope creep  
- **Bi-weekly user testing** with real DMs  
- **Monthly cost reviews** and provider evaluations  
- **Quarterly security audits**

### **Community Strategy**

- **Open-source core components** to encourage community contributions  
- **Public roadmap** with quarterly updates  
- **Beta testing program** with experienced DMs  
- **Documentation-first** approach for all APIs and features

---

## **Conclusion**

This engineering-focused revision of the DMA plan prioritizes **shipping working software over comprehensive features**. By ruthlessly focusing on the Minimum Viable Campaign, implementing performance-first architecture, and building quality assurance from day one, we can deliver a product that provides immediate value to DMs while establishing a foundation for future expansion.

The key insight from the strategic critique is that **each deferred agent represents a month gained** on shipping a delightful user experience. This plan embraces that philosophy while ensuring we build the right architectural foundations for long-term success.

**Next Steps:**

1. Finalize technical architecture specifications  
2. Set up development environment and CI/CD pipeline  
3. Begin MVC implementation with Orchestrator core  
4. Establish user testing program with target DMs  
5. Create detailed API specifications and data schemas

Success will be measured not by feature completeness, but by the ability to enhance real DM experiences in live gameplay sessions.  
