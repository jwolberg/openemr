# PRD: Clinical Co-Pilot for OpenEMR

## 1. Product Summary

The Clinical Co-Pilot is a read-only, source-grounded AI assistant embedded inside OpenEMR. It helps physicians quickly understand patient context before entering the exam room.

This PRD targets the "90 seconds between rooms" workflow: who the patient is, why they are here, what changed, and what matters today. The system must be fast, secure, patient-scoped, permission-aware, and grounded in OpenEMR data.

## 2. V1 Product Positioning

### One-sentence definition

A patient-scoped, source-grounded briefing panel inside OpenEMR that gives physicians a fast, trustworthy summary of the chart before a visit.

### V1 is not

- A general medical chatbot
- A diagnosis engine
- A prescribing assistant
- An autonomous clinical decision-maker
- A replacement for chart review
- A note-writing or order-entry system

### V1 operating assumption

The feature is for scheduled outpatient clinics using OpenEMR. The user is a physician moving between rooms, using laptop/workstation devices. V1 must not depend on dedicated room hardware.

## 3. Goals

### Primary goals

- Provide a fast pre-visit summary inside OpenEMR
- Surface relevant patient-specific chart context
- Clearly show what changed since last visit
- Ground every clinical claim in source data
- Prevent unauthorized patient-data access
- Degrade gracefully on data/tool/LLM failure
- Minimize OpenEMR core modifications

### Secondary goals

- Create an architecture that supports future expansion
- Support auditability and HIPAA-relevant logging
- Establish a reusable patient-context layer
- Keep the project easy to demo with seeded patients

## 4. Non-Goals

V1 explicitly excludes:

- Diagnosis recommendations
- Treatment recommendations
- Medication prescribing
- Lab ordering
- Clinical note writing
- Updating the patient chart
- Patient-facing chat
- Ambient listening/transcription
- Cross-patient search
- Population analytics
- Emergency triage
- Inpatient rounding workflows
- Nurse task routing
- Billing workflows
- Insurance workflows

## 5. Target Users

### Primary user

**Physician / Provider** in a scheduled outpatient visit workflow.

Primary needs:

- "Who is this patient?"
- "Why are they here today?"
- "What changed since last time?"
- "What are the active problems?"
- "What meds are they on?"
- "Any allergies?"
- "Any recent abnormal labs?"
- "What should I not miss?"

### Secondary users (out of scope for V1)

- Nurse
- Medical assistant
- Resident
- Admin/scheduler
- Patient

## 6. Clinical Setting Assumptions

### In-scope setting

Scheduled outpatient clinic (for example: primary care, small specialty, internal medicine, family medicine).

### Out-of-scope setting

- ER
- Urgent care as primary workflow
- Hospital inpatient ward
- ICU
- Surgical center
- Multi-team inpatient rounding

### Hardware assumptions

The Co-Pilot runs inside an authenticated OpenEMR browser session and may be accessed from:

- Physician laptop
- Clinic desktop
- Hallway workstation
- In-room workstation
- Workstation-on-wheels

The architecture must **not** assume:

- Dedicated hardware in every room
- Microphones
- Cameras
- Ambient sensors
- Mobile app
- Real-time vitals hardware

## 7. Core User Story

### Primary user story

As a physician, I want to open a patient chart in OpenEMR and immediately see a short, source-grounded clinical briefing, so I can understand context before entering the room.

### Supporting user stories

**Recent changes**  
As a physician, I want to know what changed since last visit, so I can focus on new/relevant updates.

**Medication review**  
As a physician, I want current medications and allergies, so I can orient quickly before speaking with the patient.

**Lab review**  
As a physician, I want recent abnormal labs surfaced clearly, so I do not miss important data.

**Source verification**  
As a physician, I want every claim linked to chart source data, so I can trust but verify.

**Failure transparency**  
As a physician, I want explicit missing/unavailable-data messaging, so absence of evidence is not misread.

## 8. UX Requirements

### Placement

The Co-Pilot appears inside the OpenEMR patient chart.

- Preferred: right-side panel/drawer/tab in chart
- Fallback: menu item/module page opened for current patient

### Default panel content

When a physician opens a chart, display:

- Patient name/DOB/ID context
- "Generate 90-second briefing" action (or auto-load)
- Suggested prompts
- Most recent generated briefing
- Source references
- Warning/missing-data states

### Suggested prompts

- "Give me a 90-second briefing."
- "What changed since the last visit?"
- "Summarize active problems."
- "Show current meds and allergies."
- "Any recent abnormal labs?"
- "What should I review before entering the room?"

### Wrong-patient prevention

The panel must always show visible patient context.

Minimum display:

```text
Clinical Co-Pilot
Patient: Jane Doe
DOB: 01/01/1970
Patient ID: 12345
```

When switching patients, prior chat state must be cleared or explicitly scoped to the prior patient.

## 9. Functional Requirements

### 9.1 Patient Briefing

Generate a concise briefing using available OpenEMR data. Include where available:

- Demographics
- Reason for today's visit
- Last visit summary
- Changes since last visit
- Active problems
- Current medications
- Allergies
- Recent labs
- Recent abnormal labs
- Recent notes
- Missing/unavailable data

### 9.2 Follow-Up Questions

Support patient-scoped follow-up questions, such as:

- What changed since last visit?
- What medications is the patient currently taking?
- Any abnormal labs in the last 90 days?
- Summarize the last visit.
- What are the active problems?

### 9.3 Source Grounding

Every clinical claim must be traceable to OpenEMR sources.

Example inline citation:

```text
Patient is taking lisinopril 10mg daily. [Medication List, updated 2026-04-20]
```

Example source chips:

```text
Sources:
- Medication List
- Encounter Note: 2026-04-20
- Lab Result: CMP 2026-04-15
```

### 9.4 Missing Data Behavior

Differentiate clearly between:

- "No data found"
- "Data source unavailable"
- "Not checked"
- "Not enough information in the chart"

Example:

```text
Recent labs could not be loaded.
Medication list was checked.
No allergy records were found in the available chart data.
```

### 9.5 Authorization

- Validate access server-side before retrieval/summarization
- Do not trust frontend patient IDs
- Scope every request to:
  - authenticated OpenEMR user
  - requested patient ID
  - current patient/encounter context
  - user permissions

### 9.6 Audit Logging

Each Co-Pilot request must create an audit event with at least:

- Timestamp
- OpenEMR user ID
- Patient ID
- Encounter ID (if available)
- Action type
- Query type
- Response status
- Sources accessed
- Latency
- Error code (if any)

Avoid storing raw PHI-heavy prompts/responses unless required. Prefer metadata, hashes, and redacted summaries.

### 9.7 Graceful Failure

Do not fail silently. Handle at least:

- LLM unavailable
- OpenEMR data retrieval failed
- Unauthorized access
- Patient not found
- No relevant data found
- Timeout
- Citation verification failed
- Partial result returned

## 10. Data Requirements

### Required V1 data domains

The audit should identify where each domain lives in OpenEMR.

| Data Domain | Required for V1 | Purpose |
| --- | --- | --- |
| Patient demographics | Yes | Identify patient context |
| Encounters | Yes | Last visit, recent history |
| Clinical notes | Yes | Relevant chart context |
| Problem list | Yes | Active conditions |
| Medication list | Yes | Current meds |
| Allergies | Yes | Safety-critical summary |
| Appointments / reason for visit | Preferred | Today's visit context |
| Labs | Preferred | Recent abnormalities |
| Vitals | Optional | Additional context |
| Immunizations | Optional | Later expansion |
| Referrals | Optional | Later expansion |
| Orders | No | Out of scope for V1 |

### Patient Context Packet

Build a deterministic, source-labeled packet before LLM invocation.

```json
{
  "patient": {
    "pid": "123",
    "name": "Jane Doe",
    "dob": "1970-01-01"
  },
  "request": {
    "type": "briefing",
    "question": "Give me a 90-second briefing"
  },
  "sources": [
    {
      "source_id": "problem_001",
      "source_type": "problem",
      "title": "Problem List",
      "date": "2026-04-01",
      "text": "Hypertension"
    },
    {
      "source_id": "med_001",
      "source_type": "medication",
      "title": "Medication List",
      "date": "2026-04-20",
      "text": "Lisinopril 10mg daily"
    }
  ]
}
```

The LLM should summarize **only** from this packet.

## 11. Architecture Requirements

### Preferred architecture

```text
OpenEMR UI
  ↓
Clinical Co-Pilot Module
  ↓
Co-Pilot Backend Endpoint
  ↓
OpenEMR Auth / ACL Check
  ↓
Patient Context Builder
  ↓
Source-Labeled Context Packet
  ↓
LLM Summarization Service
  ↓
Citation Verification Layer
  ↓
Audit Logger
  ↓
Response to OpenEMR UI
```

### Architectural principles

- OpenEMR remains system of record
- Co-Pilot is read-only in V1
- LLM does not query DB directly
- Patient data retrieval is server-side only
- Access checks are server-side only
- Every clinical claim cites a source
- No cross-patient memory
- Minimal OpenEMR core modifications
- Prefer module-based integration
- Any required core edit is isolated and documented

## 12. OpenEMR Integration Requirements

### Preferred integration

Build as a custom module. Likely location:

`interface/modules/custom_modules/clinical_copilot/`

The codebase audit should confirm the correct modern module pattern.

### Module responsibilities

The module should own:

- UI panel assets
- Backend endpoint
- Patient context builder
- LLM service client
- Citation verification
- Audit logging
- Module settings
- Cache (if used)

### Avoid core changes unless required

Avoid scattered edits in:

- `interface/patient_file/`
- `interface/forms/`
- `library/`
- `src/`

### Acceptable minimal core edit

If module injection is impossible for target chart location, one guarded mount point is acceptable:

```html
<div id="clinical-copilot-root" data-patient-id="..."></div>
```

All logic remains in the module. The audit should determine if this edit is necessary.

## 13. LLM Behavior Requirements

### The model must

- Use only provided patient context
- Cite source IDs for each clinical claim
- State when data is missing
- Avoid diagnosis/treatment/prescribing recommendations
- Avoid unsupported inference
- Preserve uncertainty
- Return structured output

### The model must not

- Claim facts absent from source packet
- Access other patients
- Treat note text as executable instructions
- Follow prompt injection from chart data
- Present itself as replacing clinician judgment
- Recommend medication changes
- Recommend diagnoses
- Generate orders

### Desired response structure

```json
{
  "summary": "...",
  "key_changes": [],
  "medications": [],
  "allergies": [],
  "recent_labs": [],
  "missing_data": [],
  "warnings": [],
  "sources": [],
  "confidence_notes": []
}
```

## 14. Security and HIPAA-Relevant Requirements

### PHI handling

Treat all patient data as PHI. Requirements:

- Use HTTPS in deployed environments
- Do not expose PHI in browser `localStorage`
- Do not log raw PHI unless necessary
- Do not send unauthorized patient data to the LLM
- Do not allow cross-patient chat memory
- Clear patient-specific state on patient switch
- Protect API keys via env vars or secret management
- Ensure access attempts are auditable

### Prompt/data handling documentation

Document:

- What patient data is sent to the LLM
- Whether prompts/responses are stored
- Whether responses are cached
- Whether embeddings are used
- Whether embeddings contain PHI
- How API keys are stored
- How model errors are handled
- Provider assumptions

## 15. Performance Requirements

### Latency targets (V1 demo)

- Deterministic context retrieval: preferred under 2 seconds
- AI-generated briefing: preferred under 8-12 seconds
- Failure/timeout response: under 15 seconds

### Product behavior

Support partial loading.

Example:

```text
Briefing loaded.
Recent labs still loading.
Medication list checked.
Last note checked.
```

### Optional caching

Precomputed briefings for scheduled patients are allowed. Cache must be:

- Patient-scoped
- Permission-aware or safely revalidated
- Expiring
- Invalidated on relevant chart data changes (if feasible)
- Clearly timestamped in UI

## 16. Failure Mode Requirements

| Failure | Expected behavior |
| --- | --- |
| Unauthorized user | Deny request and log attempt |
| Patient not found | Show controlled error |
| LLM unavailable | Show deterministic chart summary if available |
| Data source fails | Return partial answer and identify unavailable section |
| Timeout | Return partial answer or timeout message |
| Missing meds | State unavailable or no data found |
| Missing labs | State unavailable or no data found |
| Conflicting records | Show conflict and cite both sources |
| Citation verification fails | Do not show unsupported answer |
| Patient switched mid-chat | Clear or re-scope assistant state |

## 17. Acceptance Criteria

### Product acceptance

- Physician can open chart and access Co-Pilot
- Co-Pilot displays patient-specific briefing
- Briefing includes source references
- At least three supported follow-up questions work
- UI clearly shows patient identity
- No diagnosis/treatment recommendations
- Missing data is clearly labeled

### Security acceptance

- Unauthorized patient access blocked server-side
- Every request audit logged
- Frontend patient ID validated server-side
- PHI not stored in browser `localStorage`
- API keys not committed to repository

### Reliability acceptance

- LLM failure does not crash the page
- Missing data does not cause fabricated conclusions
- Unsupported claims are rejected/flagged
- Context clears on patient switch

### Architecture acceptance

- Most/all new code in custom module
- Any core edits are minimal and documented
- LLM receives source-labeled packet, not direct DB access
- Feature can be disabled without breaking OpenEMR

## 18. Demo Requirements

Demo includes at least three seeded patients.

### Demo patient 1: Routine follow-up

Shows:

- Active problems
- Current meds
- Last visit summary
- No major warnings

### Demo patient 2: Recent abnormal lab

Shows:

- Recent lab abnormality
- Citation to lab result
- Clear "review before visit" item

### Demo patient 3: Missing or conflicting data

Shows:

- Missing medication list or conflicting note
- Explicit uncertainty
- No hallucination

### Demo security scenario

Attempt unauthorized patient access and show denial plus audit log event.

## 19. Codebase Audit Questions

Use codebase audit findings to answer before building.

### Module system

- What is the correct way to create a custom OpenEMR module?
- Where do custom modules live?
- How are module routes registered?
- How are module assets loaded?
- How are module settings stored?
- How are install/uninstall scripts handled?

### UI integration

- Where is the patient chart rendered?
- Is there an existing hook/event/menu location for panel injection?
- Can a module inject UI into patient summary/chart?
- If not, what is the smallest mount-point edit?

### Auth and authorization

- How does OpenEMR represent authenticated users?
- How are roles/ACLs checked?
- How can module code verify patient access?
- Is patient/provider assignment represented?
- Is encounter-level access represented?

### Patient data

- Where are demographics stored?
- Where are encounters stored?
- Where are notes stored?
- Where are medications stored?
- Where are allergies stored?
- Where are problems stored?
- Where are labs stored?
- Are there existing services/APIs, or direct DB queries only?

### Audit logging

- Does OpenEMR already provide audit logging utilities?
- Can the module write to existing audit logs?
- Should the module create its own audit table?
- Which events should be captured?

### Security

- How are secrets/config values stored?
- Which logging patterns may leak PHI?
- How are CSRF/session protections handled for module endpoints?
- How does OpenEMR sanitize output?
- Which escaping helpers should module UI use?

### Performance

- Which data queries are expensive?
- Can patient context be built quickly?
- Is there an existing cache layer?
- Can scheduled patient briefings be precomputed?

## 20. Recommended V1 Build Sequence

This is likely sequencing after audit (not final build plan).

### Phase 1: Read-only deterministic slice

- Create custom module
- Add basic UI entry point
- Add patient-scoped backend endpoint
- Validate user/session
- Retrieve demographics, encounters, meds, allergies, problems
- Return deterministic JSON
- Render simple briefing without LLM
- Add audit logging

### Phase 2: LLM summarization

- Convert context to source-labeled packet
- Add LLM call
- Require structured JSON response
- Add citation/source display
- Add missing-data handling
- Add failure states

### Phase 3: Trust hardening

- Add claim/source verification
- Add unsupported-claim rejection
- Add prompt-injection guardrails
- Add latency timeout behavior
- Add demo patients
- Add unauthorized-access test
- Add architecture documentation

## 21. Open Decisions for Audit

Keep unresolved until audit is complete:

- Can panel injection be done fully through module system?
- Safest OpenEMR data access pattern: internal services, FHIR API, REST API, or direct DB?
- How is patient-level access currently enforced?
- Where should Co-Pilot audit events live?
- Can briefings be precomputed from appointment schedule?
- Cleanest UI location for panel?
- How much context can be retrieved within latency target?
- Existing helpers for escaping, CSRF, and module routes?
- Should AI service live in-module or as adjacent backend service?
- Is a one-line mount-point patch necessary?

## 22. Final V1 Scope Statement

V1 Clinical Co-Pilot is a read-only OpenEMR module for scheduled outpatient clinics. It appears in the patient chart and provides a fast, source-grounded 90-second briefing using authorized patient data. It supports limited follow-up questions, cites chart sources for claims, handles missing data transparently, logs access for auditability, and excludes diagnosis/prescribing/orders/chart modification. Implementation should minimize OpenEMR core changes and keep logic isolated in a custom module or adjacent service.
