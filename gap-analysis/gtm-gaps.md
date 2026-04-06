**Gap Analysis in respect of the GTM benchmark**

1. All call flows should be managed through agentic orchestration. *Plan if inbound/outbound calls require the same agentic brain. Plan filler injection. Reverse engineer BlueMachine capability.*    
2. *Explore ready made solutions like Perplexity/Gpt/Gemini Speaker.*  Integrate Sarvam models and any other latest STT/TTS models available so that the conversation feels more coherent and quick without much redundancy. The agent should respond during the call in the same language that the caller is speaking. Do we plan B2C for US first followed by B2C India after few weeks or we need simultaneous timelines. *Re-Evaluate if required for GTM\! Even if not all the languages, English \+ Hindi or Hinglish is a good starting point. Does anything change significantly on India vs US barring Hindi/Indian model integration?*  
3. Outbound calls should also include the consent request capability.  
4. The consent request screen should be included as part of the chat UI experience. *( Outbound Case. The same should also be in the old pending list for outbound. Things remain same for inbound )*  
5. Latency in the agent response during a call flow should not exceed 2.5 seconds.   
6. Latency needs to be reduced in terms of interactions too (click, scroll, swipe) across the platform so that it doesn’t feel as glitchy; can introduce smooth animations for any transitions if that helps.  
7. Noise cancellation should be implemented for WebRTC-based call flows.  
8. The consent flow should support a hold feature (already exists). *The hold duration should have a template voice in intervals saying Please hold it’s taking time*, but the caller should also be able to resume the call at any time after the hold period starts.  
9. If the owner attempts to call someone through the Trybo app and that person already exists on Trybo, the call should automatically take place through WebRTC. If the user already exists on Trybo, is there a way to fetch required info without even calling?  
10. WhatsApp should also support the consent request capability (Re-use Main Agentic Brain for any whatsapp interaction). *This would mean a filler injection if the user is taking really long to consent. In steady state, there shouldn’t be much need for consent, agent should know enough and make the choices that user would prefer (part of agent intelligence and decisioning authority) : Can be tackled by Heartbeat (two-way execution)*  
11. Users should be able to delete their current voice sample and create a new one.  
12. Voice input should also be accessible on Android devices.

\-------------- 

13. UI, connecting data sources and respective permissions \- to be tested for seamlessness   
14. UI, Task History Screen: Show Batch summary and task Summary smartly organized (***Complete UI Re-design)***; show overall task summary with status of all things done from prev session and all things planned ahead: *with Paginataion*  
15. What happens with Outbound and DND? Any created/scheduled task continues normally. Any consent needed will be shown in the “pending” screen when the user returns; realistically, very unlikely that a trybo user would mark themselves away or put themselves in DND mode  
16. Profile Section, Task Scheduling: Is this still required or to be deleted? Our thought is that we should remove it because the trybo should be intelligent enough to act based on the user prompt only in Chat ; Let us retain the old UI and experience as well in the app for beta.. We want to learn how users respond to each. Accordingly, retain the old task scheduling feature. Also we can replug the schedule button even in the new chat UI in the prompt bar; refer to yellow figma (To be confirmed later. Hold For now)  
17. Fix Google Integration for Data gathering *Hint:[https://composio.dev/](https://composio.dev/) Composio*  
18. UI, chat prompt \- allow attaching files to be shared with someone on WA/email: This is straight up file transfer like quick share ; If only files, yes.. But the task could involve drafting a certain message and attaching these files \- especially on email/WA  
19.  Build capability to read voice attachments for understanding context for a call (basically attaching audio files): How is this aligned with GTM? ; There could be context for call in the attached audio files, we can discuss if it’s too complex to address now  
20. UI, Profile Screen: Re-design Voice Connect Card.  
21. UI, Remove old create Task Screen completely.  
22. UI, Call Logs screen: Remove duplicate tabs All ⇔ Calls. Remove To-Respond from call logs, not it will be a generic consent list for both inbound and outbound. *: Full UX revamp*  
23. UI, Chat: Implement rename/delete chats. Give copy option on chat responses/messages.  
24. Within the consent request flow, the owner should be able to grant the required permissions. *Re-Evaluate if this is required\! Required if permissions not already given \- we can show a popup seeking relevant permissions as needed*  
25. Feature, Agent Chat: Implement Search within and across chats.  
26. Feature, Task Scheduling: Daily task scheduling should work from chat. We need to explore a ui which can effectively convey all the scheduled tasks to the user (Explore likes of Openclaw or Claude Computer) *: To test end to end*  
27. Backend: Fix Call Summaries. It says “Your AI Assistant Called….”  
28. Backend: Episodic memory and Semantic Memory required.  
29. Backend: Intelligence for Communication GTM \- Channel auto-selection — "contact John" should resolve to best channel (WhatsApp if online, call if urgent).  
30. Backend: Execution Resilience: Task Failure handling \- Single Node failure. (This is separate from batch/request restart) *: Inside Chat UI (should feel seamless) \-\> Proactive message on chat in failure cases and possibility to restart it task/batch from chat ui itself , agent should understand if task needs to be restrated or batch*  
31. Backend: **Planning Quality \- Planner reliability/success rate** — how often does the plan come out correct for communication flows? This IS the product.   
    **Testing, benchmarking, and improving plan quality.**   
    \- Fast path for trivial intents — "call John" goes through full LLM planning today. For GTM latency, simple single-skill intents should bypass the planner.   
    \- Clarification UX in chat — when the agent asks "which John?", how does that flow look? Is it interruptible? Can the user correct mid-clarification?   
    \- Plan failure messaging — when planning fails (invalid skill, missing info, LLM error), what does the user actually see?

\------------- Low Priority Below

32. Backend: Rate Limiting *(for unautherized \- infra is handling it bt for authrised misuse like triggering 1000 calls using some csv file and all)*  
33. Cost Visibility?  
34. Feature, Agent Chat: Enable bigger file upload size limit from 10MB. Move document storage to PGVector (Make RAG).  
35. Twilio-based calls should preferably be routed through LiveKit SIP trunking, because this approach provides a more manageable and robust calling screen UI within the app. *Question: How?*  
36. The chat UI should be interactive enough that, after reviewing the live transcript, the owner can redirect or modify the course of the conversation for both calls and WhatsApp. *Bringing Consent into chat is going to add many cases for us. Need to evaluate the gain vs loss or gain vs UI complexity.* Live transcript streaming should be supported for both calls and WhatsApp*. We can show prev transcript during consent but no use in the rest of the cases.*

---

\_\_\_\_

**Roadmap planning**

**Agentic orchestration infra (40% standalone bandwidth)**

1. Core platform (KPR)  
2. Task decomposition and planning (RKP)  
3. Skill management (AKN)  
   1. Agent crawl  
   2. Indexing  
   3. Skill registration  
   4. Agent invocation and output retrieval  
   5. Health metrics  
4. AgentRank and AgentAllot (AKN)  
5. Execution engine (RKP)  
6. New skill creation (AKN)  
7. 1P skills (KPR)  
   1. Communication  
   2. Web research  
8. User memory (KPR)  
9. Agent intelligence and decisioning authority (RKP)  
   1. Feedback and reinforcement  
10. UI/UX (KPR)  
11. Persona management (RKP)  
12. Privacy/Security (RKP)

**GTM Focus (60% effort)**

Detailed Below

**Dev tools**

15. Agentic coding as a team  
    1. Foundation setup \+ training Claude (AKN)  
    2. Product planning (AKN)  
    3. System design (AKN)  
    4. Development (RKP)  
    5. Unit testing (KPR)  
    6. Review (RKP)  
    7. Deployment  
    8. Testing (KPR)

**Constraints**

1. Dev team \= 3 for Apr  
2. Dev team \= 4 for May; best case scenario 5  
3. Dev team \= 4 for June; best case scenario 6  
4. Assume infinite resources from July, so we work backward from chosen milestones and mobilise resources accordingly



**Scenarios**

1. Focus purely on GTM; agentic dev tool will be adhoc on-demand  
2. Set foundation for agentic development, then focus purely on GTM with adhoc agent dev, and then focus on agentic orchestration infra  
3. Set foundation for agentic development, focus 60% on GTM and 40% on infra with ad-hoc agent dev 


**What are the primary use cases for a prosumer?**

Prosumers basically loathe doing repetitive non-strategic and non-creative tasks which happen to consume most of their time and seek to outsource it.

1. Delegation  
2. Information retrieval  
   1. Through searching connected data sources (third party apps)  
   2. Web search  
   3. Web scraping  
   4. Communicating with someone to source unavailable relevant data  
3. Analysis of the available information  
4. Follow on action  
   1. Scheduling  
   2. Communicating  
      1. Call  
      2. WhatsApp  
      3. Other channels (email, slack)  
      4. Meeting apps  
   3. Documenting

**Tasks**

Understand a requirement, figure out who to delegate to, delegate it and follow up to completion, analyse received submissions, do further followups on any missing pieces, consolidate everything into an executive summary for user \[think BU head use case, can be extended to most real professional use cases\]

**Adopters**

1. Executives (Directors, VP, CXOs)  
2. Middle management (Managers, Leads, Staff Engg/PM, etc)  
3. ICs (different skills and personas kick in)  
   1. Analyst  
   2. Accountant  
   3. Sales  
   4. Support  
   5. Product manager  
   6. Growth strategy  
   7. Developer  
   8. QA  
   9. Asset manager  
   10. Doctor  
   11. Lawyer  
   12. Nutritionist

**Nature of communication (different personas requiring communication as a 1P skill)**

1. Procurement  
2. Sales  
3. Receivables recovery  
4. Support  
5. Secretary (for coordination and delegation)

**Things to do for GTM** 

For EA/secretary as the first persona focusing on information retrieval for executives

1. Enable more data sources to be connected  KPR
   1. **Note taking apps**  
   2. **Meeting apps \- Zoom, Teams, Meet, etc (Transcript/Summary)**  
   3. **Device file storage** \-\> Basic \-\> Text/PDF only  
   4. Communication apps  
      1. **Email (Gmail, Outlook, etc) \-\> Composio**  
      2. Slack  
      3. **WhatsApp**  
      4. **Calls/Contacts**  
      5. **SMS**  
      6. Telegram  
      7. Discord  
   5. Notifications  
   6. **Calendars**  
   7. **Location**  
   8. **Camera**  
   9. **Web (web research and scraping)**  
   10. **User input**  
   11. Third party app data on device (low priority for GTM)  
2. **Outbound/inbound consent flow (user input as a data source) \-\> KPR**  
3. Improve the number of data/file formats that can be read  RKP
   1. **Text**  
   2. Images  
   3. **Audio \-\>Domain Type  Refinements later**  
   4. Video  
   5. Tables  
   6. **Documents**  
   7. Charts  
   8. **Numbers (csv, xls, etc)**  
   9. GeoCoordinates  
4. **Enable file attachment for outbound communication**  RKP
5. **Voice intelligence  \-\> RKP**  
   1. **Improve coherence/accuracy (might involve separate consent flow)**  
   2. **Reduce latency**  
6. **Task visibility**  AKN
   1. **Search**  
   2. **Status of subtasks and batches**  
7. **Execution intelligence**   RKP
   1. **Task restarting**  
   2. **Followups**  
   3. **Channel autoselection**  
   4.   
8. **Traceability\`**  AKN
   1. Debug logs  
   2. Analytics setup  
9. **Compliance**  AKN
   1. PII protection  
   2. Audit trails and compliance  
   3. Payment gateway integration 

\-\> 9 milestones towards beta release with owners assigned for each milestone that could involve multiple contributors

Technical architecture reviews (40%)

Demo progress on GTM (60%)

Final Layered Ownership Split for 40% long term (Clean No intersection Model)

L1 — Platform Foundation

Barebone Platform

Execution Tools

L2 — Agent Runtime Brain

Planning / Decomposition

Agent Rank

L3 — Capability System

Skill Map / Tree

Skill Creation (on-the-fly)

L4 — Intelligence & Memory

User Memory

Agent Intelligence

L5 — Product Layer

UI/UX \+ Frontend

