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

**Not Applicable:**

1. The call transcript should be toggleable, allowing it to be translated into the selected language.  
2. **UI Typing Number:** The owner should be able to call anyone directly through the keypad. \#\* Siri seems to be doing a good job with this even with multiple similarly saved names, worth evaluating how they do it  
3. **Chat UI, Response Streaming:** , the agent should respond using token-based Server-Sent Events (SSE). Markdown for streaming chat response.

---

**Other Non Product Requirements:**

1. We need a company card which enables auto-payments. Let’s pick Aman’s brain on this  
2. Perform pricing analysis across all supported channels in order to help determine subscription pricing. *Re-Evaluate if required\! This is needed, but has nothing to do with development*  
3. What about making Try\_Bo-SDK 😉; We gotta make it.. And since we called ourselves a god app, we gott make AI bhoot \+ AI astrologer too XD

\_\_\_\_

**Roadmap planning**

**Agentic orchestration infra (40 to 50 % standalone bandwidth)**

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

**GTM Focus (50 to 60% effort)**

13. B2C GTM build (Communication and web research as a 1P skill)  
    1. Voice intelligence (RKP)  
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

**Owners**

1. Voice Intelligence  
2. Task decomposition \+ execution  
3. Agentic memory  
4. Traceability and analytics (dev \+ user engagement)

**Scenarios**

1. Focus purely on GTM; agentic dev tool will be adhoc on-demand  
2. Set foundation for agentic development, then focus purely on GTM with adhoc agent dev, and then focus on agentic orchestration infra  
3. Set foundation for agentic development, focus 60% on GTM and 40% on infra with ad-hoc agent dev 

**What apps do I use as a consumer?**

1. Communication \[1p skill\]  
   1. WhatsApp  
   2. Email  
   3. Call  
   4. Meeting apps  
   5. Slack  
2. Shopping  
   1. Groceries and home essentials (FMCG) \- Quick Commerce  
   2. Regular shopping \- Amazon, Flipkart, Meesho, Bestbuy, etc  
      1. Fashion  
      2. Decor  
      3. Stationery  
      4. Electronics  
3. Commute planning  
   1. Maps  
   2. Public transport  
   3. Cabs  
4. Experience planning  
   1. Mobility: Flights, trains, etc  
   2. Dineout, nightlife, movies, etc  
   3. Travel and tourism  
5. Infotainment \[best case scenario is content recommendation and info digest\]  
   1. Instagram  
   2. LinkedIn  
   3. YouTube  
   4. X  
   5. Discord  
   6. Any other social media  
   7. Gaming apps  
   8. News apps  
6. Search, research, analysis and planning \[to be entirely outsourced to LLMs\]  
   1. Google  
   2. GPTs of the world

**Tasks**

1. Research and analysis \- outcome is to decide what to do (watch, buy, eat, etc) and where \[research and analysis are to be done purely by LLMs, we work on providing the richest data from all possible sources, perhaps with permissions\]  
2. Coordination and consent \- may involve communicating with someone back-and-forth with advanced file intelligence \[we build 1P communication, web search \+ web scraping\]  
3. Confirmation action \- reservation, purchase, etc \[involves device-use or interface operation agents\]  
4. Checkout \- Transaction \[requires user consent\]

**Nature of tasks**

1. High frequency (generally lower ticket) \- daily to weekly  
   1. Groceries and home supplies  
   2. Commute planning  
   3. Weekend experiences planning  
2. Medium ticket medium frequency \- bimonthly to quarterly  
   1. Fashion  
   2. Flights, trains  
3. High ticket (generally lower frequency) \- quarterly or fewer  
   1. Electronics  
   2. Decor  
   3. Travel and tourism

One very plausible scenario is that our sprint plan enables B2B a lot more than B2C and hence we should work on it if we want to go B2B. And if so, B2C might require a reduced parallel sprint plan. Or if the current sprint plan is required for even B2C, what will change dramatically for the user if we stick to just the first 3 months (Apr-Jun) and ignore the last 3 months (Jul-Sep) of effort?

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

1. Enable more data sources to be connected  
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
2. Improve the number of data/file formats that can be attached/read  
   1. **Text**  
   2. Images  
   3. **Audio \-\>Domain Type  Refinements later**  
   4. Video  
   5. Tables  
   6. **Documents**  
   7. Charts  
   8. **Numbers (csv, xls, etc)**  
   9. GeoCoordinates  
3. **Enable file attachment for outbound communication**  
4. **Outbound consent flow**  
5. **Voice intelligence**  
   1. **Improve coherence/accuracy (might involve separate consent flow)**  
   2. **Reduce latency**

	