**Roadmap planning**

**Agentic orchestration infra (40 to 50 % standalone bandwidth)**

1. Core platform (KPR)  
2. Task decomposition and planning (RKP)  
   1. LLM Planner and Clarification Engine (also  
   2. DAG Compilations  
   3. Interrupt System (Approval, clarifications, oAuth, agent question interrupt with durable checkpoints)  
   4. Plan Validation & Safety   
3. Skill management (AKN)  
   1. Agent crawl  
   2. Indexing  
4. AgentRank and AgentAllot (AKN)  
5. Execution engine (RKP)  
   1. LangGraph Runtime: State Graph with Checkpoints  
   2. mapping of tasks to batches- agent would do that  
   3. Batch Orchestration \- atomic progress, reconciliation, completion detection and LLM synthesis, retry, report back  
   4. Adapter Framework  
   5. Execution Observatbility   
6. New skill creation (AKN)  
7. 1P skills (KPR)  
   1. Communication  
   2. Web research  
8. User memory (KPR)  
9. Agent intelligence and decisioning authority (RKP)  
   1. Feedback and reinforcement (post execution)  
      1. prompt Evolution System  
      2. Implicit and Explicit feedback handling system and Storage  
   2. Decisioning authority Framework  
      1. Complete consent mangement  
      2. MultiPlan Generation  
   3. Strategy adaptation and error classification system  
   4. Plan quality Scorning  
10. UI/UX (KPR)  
11. Persona management (RKP)  
    1. Agent Persona Definations  
    2. Voice Persona   
    3. Commmunication Style  
    4. Persona Context in Planning  
    5. Multi-persona Support and Persona Memory Separation

*Think of personas (most common 10-20) without task planning*

12. Privacy/Security (RKP)  
    1. Content: Safety Guardrailing system  
    2. Approval Gates  
    3. Authetication and Authorization RBAC  
    4. Compliance and Audit  
    5. Encryption

**GTM Focus (50 to 60% effort)**

13. B2C GTM build (Communication and web research as a 1P skill)  
    1. Voice intelligence (RKP)  
       1. Voice Synthesis and Cloning  
       2. Speech recognition and Understanding  
          1. ASR, and Mobile Voice input  
          2. Transcript Understanding  
       3. Conversational AI Call intelligence: Inbound/Outbound  
          1. Intent DEtection and Routing   
          2. Call Session Management  
          3. Agent Persona Defination  
       4. Real-time Voice Streaming Infra: TTS  
       5. Voice UX  
    2. PII protection (KPR)  
    3. Audit trails and compliance (KPR)  
    4. Debug logs (AKN)  
    5. Analytics setup (AKN)  
    6. Payment gateway integration (KPR)  
14. B2B GTM build \- to be focused on separately as a pilot

**Dev tools**

15. Agentic coding as a team  
    1. Foundation setup \+ training Claude (AKN)  
    2. Product planning (AKN)  
    3. System design (AKN)  
    4. Development (RKP)  \[i think we should merge d \+ a\]  
       1. Backend Developement  
       2. Frontend Developemnt   
       3. Coordination agent for planningn  
    5. Unit testing (KPR)  
    6. Review (RKP)  
       1. Code review Subagent  
       2. Plan Review SubAgent  
       3. PR Review SubAgent  
       4. Plan Reviewer Agent 

