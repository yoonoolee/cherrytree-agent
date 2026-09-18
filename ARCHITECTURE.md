# Cherrytree Agents — Architecture Notes

Working design doc for the AI system that guides cofounders through building their
cofounder agreement by facilitating the necessary conversations along the way.
Inspired by Listen Labs' AI-moderated interview model, adapted for a non-chat,
non-voice, guided-survey product experience.

Not built yet. This captures decisions made during design discussion, plus open
TODOs. The `cherrytree-chat-agent` repo is being retired and should not be referenced. 

```
Cofounders
    ⇅
Advisor Agent  →  Drafting Agent  →  Draft
    ↑                                 |
    └───────── editing loop ──────────┘
                    |
                    ↓  (after finalizing)
            Evaluation Agent
                (internal)
```

## Product shape

- Not a chatbot. UI/UX still TBD. One option is a progressive, one-question-at-a-time 
  UI like a survey. Possibly presented via an AI character. The agent decides what 
  to ask next behind the scenes.
- Text only for now. 
- Joint session. All cofounders go through this together, at the same time. Part of 
  the product's value is surfacing conversations cofounders haven't had yet, which 
  requires them being present together when a topic comes up.
- Cofounders assumed to be physically together, with one account driving the session. 
  This will most likely become a Multi-account (separate logins, per-question targeting,
  attribution, real-time sync) but those details will be decided later. Adding it 
  later shouldn't require redesigning the agent's core reasoning, since the agent 
  reasons over shared session state regardless of how many people are contributing to it.

---

None of the agents below need MCP. Integrations (Firestore, document storage) are
first-party, so plain custom tools are enough.

## 1. Advisor Agent

The user-facing agent. Runs the guided flow: 
1. asks questions, 
2. decides on follow-ups,
3. decides when to move on, 
4. later handles the post-draft editing conversation.

Tools:
- Read/write structured answers
- Trigger the Drafting Agent to generate/regenerate the draft
- Knowledge base search, for grounding advice in curated reference content (tbd if needed)

### Guide structure

Instructions are organized around the existing 10 survey sections (Formation,
Cofounders, Equity, Vesting, Decision-Making, IP, Compensation, Performance,
Non-Competition, General Provisions).

- Core system prompt (.md): loaded at all times. Contains identity, tone, safety, response format. 
- Instruction files (.md): Each section has its own set of instructions. They contain
the questions for that section, what matters and why for that topic, and red-flag guidance.
- A generic "Others" file: Parallel to the 10 section files, for topics that don't
fit an existing section. Also contains the questions for that section, what matters 
and why for that topic, and red-flag guidance. 

### Deciding what to ask next

After each answer, a decision step chooses one of the following:
- Probe deeper. Ask follow up questions on the same topic if the 
  answer surfaces something worth exploring further
- Skip a scripted question. No need to ask if it doesn't apply 
  anymore given the prior answers
- Ask the next question. 

### Context and data management — TBD

Still undecided how much loads into context at a given point, and exactly how
conversation history and answers get stored. Two directions discussed:

- Progressive: only the current section's file loads, plus structured answers
  extracted incrementally as the session progresses, to keep context lean.
- Full-context: full conversation history and all guide files load every turn,
  relying on a large-context model rather than managing this closely.

Either way, answers are stored in Firestore as a JSON document per project. Flexible
schema, so ad hoc/non-section entries don't need special handling at the data layer.
This store is what lets the Drafting Agent and Evaluation Agent consume structured
state directly instead of re-parsing a transcript, and what a session resumes from.

### Post-draft editing loop 

After the Drafting Agent produces a draft, this agent is re-entered with the draft as
context, with the ability to mutate the underlying structured answers. The document
is a rendering of that structured state.

- Draft renders as a real document, organized section by section
- Users can edit fields directly, or ask the agent to change, deepen, or add a topic
- Editing mode prompts for what to change, classifies the request, and reuses the
  relevant section's existing guidance if a topic is named. Exact loading mechanism 
  depends on the context and data management decision above
- Each section shows whether it's resolved or still open
- Every change is logged with attribution: AI-suggested-and-approved,
  AI-executed-on-request, or manually typed. This log feeds the Evaluation Agent
- The agent classifies each incoming request as resolvable immediately or needing
  its own mini-interview. For example, "change cliff to 6 months" vs "what if one 
  of us leaves early?" 
- Requires an explicit finalize step

### TODOs
- Decide context and data management strategy: progressive per-section loading vs.
  full-context, and how the Firestore structured store fits in either way
- Design the UI/UX
- Side channel for on-demand questions, clarification, or advice on what to choose
  for the current question, without derailing the main guided flow (like /btw)
- Define per-question config for joint vs. individual-per-cofounder answers, and
  criteria for when each applies
- Build an eval set of realistic scenarios to tune follow-up quality
- Decide on how finalizing or locking the final document version works. 
  Revisit once multi-account exists
- Versioning/rollback approach for edits
- Decide on multi-account vs single-account 

---

## 2. Drafting Agent

Turns the collected structured answers into the actual cofounder agreement document.

Tools:
- Read structured answers
- Code execution: for formatting and generating the document
- File/text editor: to create and edit the document
- Document-generation skill (docx-style)
- Save the generated document

- Structured answers remain the source of truth. The document is generated from
  that state.

### TODOs
- Decide the drafting approach: rules-guided drafting, or fully generative. 

---

## 3. Evaluation Agent

Internal only, never shown to users. Evaluates completed agreements to judge whether
the Advisor Agent asked the right questions and surfaced the right topics, to improve
it over time.

Tools:
- Read structured answers, the edit and audit log, and the final agreement
- Code execution, for diffing and structured analysis

Signal: diff what the Advisor Agent originally asked or produced against what the
user ended up adding, editing, removing, or raising themselves. That gap is what can 
be improved with our agent so there is less editing needed from users next time.

### TODOs
- Define the rubric for what counts as the agent doing a good job
- Decide how findings feed back into prompt or guide changes

---

## Deferred

- Cross-agreement analytics, once there's real volume across many companies
- Multi-account, real-time joint sessions, see Advisor Agent TODOs
- Voice
- Associative cross-topic follow-ups (raising a related topic when another comes up,
  not just probing deeper on the same one). Relying on the base model's default
  judgment for now.

---

## Reference: how Listen Labs works

Background research this design draws from. Kept here rather than in the sections
above so those stay focused on our own design.

### Participant experience

- Participants join via a study link, no account required. Desktop opens in-browser;
  iOS opens through the Listen app after a download prompt, needed for screen
  recording and app-switching capture.
- Digital consent is collected up front, and again per-session for screen sharing or
  video.
- The discussion guide is organized into sequential question blocks (screener,
  demographics, concept tests, open-ended exploration), with an estimated interview
  length calculated ahead of time.
- Follow-up depth is configurable per question: none, light, or 2-3 follow-ups,
  applied conditionally based on whether the initial answer was substantive.
- Conditional branching is evaluated live. A question whose condition fails is
  silently skipped, no broken or empty screen.
- Supports video, audio, or text, with voice read aloud in 40+ languages.
- Everything is recorded and transcribed automatically, with synchronized screen
  recording when enabled.

### Conversational style

- Mirrors the participant's language rather than reformulating it
- Allows silence instead of rushing to fill pauses
- Avoids evaluative affirmations like "great answer," which can bias later responses
- Applies the same probing depth and rigor across every session, with no fatigue or
  drift
- Participants report feeling less judged by the AI than by a human moderator on
  sensitive topics

### Agent architecture

Production system is multi-agent, evolved from an earlier single-bot RAG approach
that wasn't sufficient on its own.

- Interviewer agent: runs the live conversation, reasoning within the bounds of the
  guide and its per-question settings
- Reviewer/QA agent: runs asynchronously, checks the quality of the generated report
  rather than live conversational behavior
- Report-builder agent: generates the deliverable (e.g. slide decks), rebuilt on
  Claude's Agent SDK for document/file generation
- Parallel analysis workers: for synthesizing across many completed interviews, a
  hardcoded orchestrator fans work out to many agent instances in parallel, each
  handling a slice, then aggregates results

Guides are AI-drafted through Listen's study builder, then customized by the
researcher running the study: adjusting follow-up depth, adding free-text probing
instructions, and setting conditional logic.

### Eval methodology

An internal eval for avoiding repetitive questions and following instructions
started around 20% pass rate on GPT-4, iterated up to about 85%, then replaced with
a harder eval once it stopped being discriminating. Quality came from building a
measurable eval loop and iterating against it, not one-shot prompting.

### Other capabilities

- Emotional intelligence layer analyzing tone, word choice, and facial expression
  against Ekman's six-emotion framework, as a separate post-hoc analysis pass
- Quality guard cross-references identity verification against video and checks
  answer consistency, to catch fraudulent participants
- No training on customer data, with automatic PII scrubbing
- A separate capability beyond interviewing: after enough interview data, they can
  build a predictive model of a person's preferences, with backtested accuracy up to
  about 95% in some cases

### Sources

- https://listenlabs.ai/
- https://listenlabs.ai/articles/best-ai-moderated-research-tools/
- https://docs.listenlabs.com/setup-to-launch/study-design-101.md
- https://docs.listenlabs.com/setup-to-launch/conditional-logic.md
- https://docs.listenlabs.com/setup-to-launch/multi-lingual-voice-interviewer.md
- https://docs.listenlabs.com/setup-to-launch/desktop-and-mobile-screensharing.md
- https://docs.listenlabs.com/interview-data/interview-grid-and-transcripts.md
- https://docs.listenlabs.com/llms.txt
- https://sequoiacap.com/podcast/knowing-what-your-customers-want-all-the-time-listen-labs-alfred-wahlforss
- https://www.linkedin.com/posts/harrison-chase-961287118_in-the-second-episode-of-max-agency-i-activity-7453097486665261056-9cVe
- https://research.contrary.com/company/listen-labs
- https://venturebeat.com/technology/listen-labs-raises-usd69m-after-viral-billboard-hiring-stunt-to-scale-ai
- https://getthecheckpod.substack.com/p/breaking-news-listen-labs-raises
