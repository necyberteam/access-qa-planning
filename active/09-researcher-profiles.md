# Researcher Profiles

> **Related**: [Agent Architecture](./01-agent-architecture.md) | [MCP Authentication](./06-mcp-authentication.md) | [Capability Registry](./11-capability-registry.md) | [Capability Registry Spec](https://github.com/necyberteam/access-agent/blob/main/docs/superpowers/specs/2026-03-18-capability-registry-design.md)

## Overview

Researcher profiles enable personalized AI assistance by capturing research context, preferences, and interaction history. Following [MyData principles](https://mydata.org/participate/declaration/), profiles serve the researcher—not the other way around.

**Core promise**: Better understanding of research needs enables personalized advice and simplifies resource requests, while users retain full control to view, correct, delete, or opt out at any time.

### Relationship to Existing Community Persona

ACCESS already has a **Community Persona** system in Drupal (`/community-persona`) that captures:
- Bio, skills, interests (public-facing)
- Affinity group memberships
- MATCH engagements and mentorships
- Knowledge base contributions
- Event registrations

The Researcher Profile is **complementary**—it captures *private* AI interaction context that enhances the agent experience without being publicly visible.

| Community Persona | Researcher Profile |
|-------------------|-------------------|
| Public-facing | Private (user only) |
| Community networking | AI personalization |
| Skills & interests (tags) | Research context & preferences |
| Manual updates | Learned from interactions |
| Stored in Drupal user fields | Stored in new profile entity/fields |

### Why Profiles Matter for the Agent

| Without Profile | With Profile |
|-----------------|--------------|
| "What resource should I use?" → Generic answer | → Recommendation based on field, past usage, allocation history |
| "Request an allocation" → Start from scratch | → Pre-filled with typical request patterns |
| "Is my job running?" → Which project? Which resource? | → Context-aware, knows active allocations |
| Repeat questions session to session | → Remembers context from previous sessions |

---

## Design Principles

Aligned with [MyData Declaration](https://mydata.org/participate/declaration/) principles:

| Principle | Implementation |
|-----------|----------------|
| **Human-centric control** | User can view, edit, delete profile anytime |
| **Individual as integration point** | Profile aggregates context from multiple sources |
| **Transparency** | Clear explanation of what's stored and why |
| **Portability** | Export profile in standard format (JSON) |
| **Data minimization** | Store only what improves the experience |
| **Purpose limitation** | Profile exists solely to improve AI assistance |

---

## Profile Architecture

### Storage in Drupal

Extend the existing user entity with new fields for AI profile data:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         USER ENTITY EXTENSIONS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  EXISTING (Community Persona)          NEW (Researcher Profile)             │
│  ┌─────────────────────────┐          ┌─────────────────────────┐          │
│  │ field_user_bio          │          │ field_ai_research_domain│          │
│  │ field_user_badges       │          │ field_ai_expertise_level│          │
│  │ field_institution       │          │ field_ai_preferences    │          │
│  │ field_hpc_experience    │          │ field_ai_context_summary│          │
│  │ field_academic_status   │          │ field_ai_profile_enabled│          │
│  │ + flagged skills        │          │ field_ai_last_topics    │          │
│  │ + flagged interests     │          │ field_ai_resource_prefs │          │
│  └─────────────────────────┘          └─────────────────────────┘          │
│                                                                             │
│  Via Flags (existing)                  Via Flags or separate table          │
│  ┌─────────────────────────┐          ┌─────────────────────────┐          │
│  │ skill (taxonomy)        │          │ ai_preferred_resource   │          │
│  │ interest (taxonomy)     │          │ ai_topic_history        │          │
│  │ affinity_group          │          │                         │          │
│  └─────────────────────────┘          └─────────────────────────┘          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### New User Fields

| Field | Type | Description | Source |
|-------|------|-------------|--------|
| `field_ai_profile_enabled` | boolean | Master toggle for AI personalization | User setting |
| `field_ai_research_domain` | list (string) | Primary research domain(s) | User-provided |
| `field_ai_expertise_level` | list (string) | HPC/computing expertise | User-provided or inferred |
| `field_ai_preferences` | JSON blob | Response preferences (detail level, examples) | User settings |
| `field_ai_context_summary` | text_long | LLM-generated summary of recent interactions | Derived |
| `field_ai_last_topics` | text_long | Recent question topics (JSON array) | Derived |
| `field_ai_resource_prefs` | entity_reference (multi) | Preferred compute resources | Derived from usage |

### Leveraging Existing Data

The profile should automatically incorporate existing Community Persona data:

| Existing Field | AI Profile Use |
|----------------|----------------|
| `field_hpc_experience` | Initialize expertise level |
| `field_academic_status` | Context for recommendations |
| `field_institution` | Institutional context |
| Flagged skills | Domain inference |
| Flagged interests | Topic relevance |
| Affinity group memberships | Community context |

---

## Profile Contents

### User-Provided (Explicit)

| Field | Options | Purpose |
|-------|---------|---------|
| **Research Domain** | Dropdown: Bioinformatics, Materials Science, Climate, AI/ML, Physics, Chemistry, Engineering, Social Sciences, Other | Filter recommendations |
| **Computing Expertise** | Beginner / Intermediate / Advanced | Adjust response detail |
| **Preferred Detail Level** | Concise / Moderate / Detailed | Response length |
| **Include Examples** | Yes / No | Code/command examples |
| **Show SU Estimates** | Yes / No | Include cost estimates |

### System-Derived (Implicit)

| Data | Source | Retention |
|------|--------|-----------|
| Recent topics | Conversation history | 90 days |
| Preferred resources | Usage via MCP allocations | Ongoing |
| Question patterns | Session analysis | 90 days |
| Unanswered questions | Session tracking | Until resolved |
| Feedback patterns | [Feedback API](./03-review-system.md#feedback-collection) | Ongoing |

**Feedback-Derived Insights**: When users provide explicit feedback (thumbs up/down), patterns can inform the profile. For example, repeated negative feedback on GPU recommendations might indicate the user needs different information. See [Feedback Collection](./03-review-system.md#feedback-collection) for details on how feedback flows through the system.

### What We Don't Store

Following [data minimization principles](https://www.alation.com/blog/gdpr-data-compliance-best-practices-2025/):

- Full conversation transcripts
- Raw MCP API responses
- Authentication tokens
- Specific job/project details (fetched live via MCP)
- Anything without clear personalization benefit

---

## Integration with ACCESS Agent

### Profile Injection

When a user with an enabled profile queries the agent:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PROFILE-ENHANCED AGENT FLOW                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   User Query                                                                │
│       │                                                                     │
│       ▼                                                                     │
│   ┌─────────────────┐                                                       │
│   │ Fetch Profile   │◀──── GET /api/ai-profile (Drupal)                    │
│   │ (if enabled)    │      or MCP tool: get_researcher_profile             │
│   └────────┬────────┘                                                       │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────┐                                                       │
│   │ Query Classifier│ ◀── Profile context influences classification        │
│   │ + Profile       │                                                       │
│   └────────┬────────┘                                                       │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────┐                                                       │
│   │ RAG + MCP       │ ◀── Profile boosts relevant domain results           │
│   │ (as usual)      │                                                       │
│   └────────┬────────┘                                                       │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────┐                                                       │
│   │ Synthesize      │ ◀── Response tailored to expertise level             │
│   │ + Personalize   │                                                       │
│   └────────┬────────┘                                                       │
│            │                                                                │
│            ▼                                                                │
│   ┌─────────────────┐                                                       │
│   │ Update Profile  │──── PATCH /api/ai-profile (async)                    │
│   │ (topics, etc)   │      Update recent topics, unanswered Qs             │
│   └─────────────────┘                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### System Prompt Enhancement

When profile is available, agent system prompt includes:

```
User Context (from profile):
- Research Domain: Computational Biology
- Institution: University of Example (from Community Persona)
- HPC Experience: "Familiar with Slurm, have run jobs on XSEDE" (existing field)
- Expertise Level: Intermediate
- Preferred Resources: Delta, Expanse (from usage history)
- Recent Topics: GPU memory optimization, PyTorch distributed training
- Preferences: Include command examples, show SU estimates

Relevant Community Persona data:
- Skills: Python, Machine Learning, Genomics
- Interests: Deep Learning, Bioinformatics
- Affinity Groups: AI Institute, Research Computing Facilitators
```

### Personalization Examples

**Query Expansion**:
```
User: "What resource should I use?"

Without profile: → Needs clarification
With profile (domain: bioinformatics, past: GPU workloads):
  → Expanded to: "What GPU resource should I use for bioinformatics work?"
```

**Response Tailoring**:
```
User: "How do I submit a job?"

Expertise: Beginner → Full explanation with basic concepts
Expertise: Advanced → Just the commands, skip basics
```

**Resource Recommendations**:
```
User: "I need more compute time"

Without profile: Generic allocation info
With profile (prefers Delta, typical 50K SU requests):
  → "Based on your usage, a Maximize allocation on Delta (~100K SUs)
     would cover about 6 months. Want me to help draft the request?"
```

---

## User Control Interface

### Profile Settings Page

New page at `/community-persona/ai-settings` (extends existing Community Persona):

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          AI ASSISTANT SETTINGS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ PERSONALIZATION                                          [Toggle]   │   │
│  │ ○ Enabled  ● Disabled                                              │   │
│  │                                                                     │   │
│  │ When enabled, the AI assistant learns from your interactions to    │   │
│  │ provide better recommendations. You control all stored data.       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ RESEARCH CONTEXT                                          [Edit]    │   │
│  │ Domain: Computational Biology                                       │   │
│  │ Expertise: Intermediate                                             │   │
│  │ (Your skills and interests from Community Persona are also used)   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ RESPONSE PREFERENCES                                      [Edit]    │   │
│  │ Detail level: Moderate                                              │   │
│  │ Include examples: Yes                                               │   │
│  │ Show SU estimates: Yes                                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ LEARNED FROM YOUR USAGE                                  [Clear]    │   │
│  │ • Preferred resources: Delta (GPU), Expanse                         │   │
│  │ • Recent topics: PyTorch, CUDA, distributed training                │   │
│  │ • Last active: 2 days ago                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ DATA MANAGEMENT                                                     │   │
│  │                                                                     │   │
│  │ [Export My Data]  [Clear All Learned Data]  [Disable & Delete]     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Conversational Management

Users can also manage profiles through the agent:

```
User: "What do you know about me?"
Agent: "Based on your profile, I know you work in computational biology,
       typically run GPU jobs on Delta, and recently asked about PyTorch.

       I also see from your Community Persona that you have skills in
       Python and Machine Learning, and you're part of the AI Institute
       affinity group.

       You can update your AI settings at [link] or say 'forget my history'
       to clear learned data."

User: "Forget my history"
Agent: "Done. I've cleared your recent topics and resource preferences.
       Your explicit settings (domain, expertise level) are unchanged.
       Want me to clear those too?"
```

### One-Click Rights

Following MyData's vision of [actionable rights](https://mydata.org/participate/declaration/):

| Right | Action | Implementation |
|-------|--------|----------------|
| **View** | See all stored data | Profile settings page |
| **Correct** | Edit any field | Edit forms for each section |
| **Export** | Download as JSON | Export button → JSON download |
| **Forget** | Clear learned data | Clears derived fields only |
| **Delete** | Remove everything | Clears all + disables profile |
| **Opt out** | Never personalize | Master toggle off |

---

## API Design

### Phase 1: Agent Fetches Directly from Drupal

The agent's `/api/v1/capabilities/personalized` endpoint calls Drupal's JSON:API to fetch existing user data. No custom Drupal endpoint needed for Phase 1 — standard JSON:API on the user entity provides everything.

**Drupal data the agent reads:**
- `field_ai_profile_enabled` — opt-in toggle (new field, the only Drupal change)
- `field_cider_resources` — active allocations (existing)
- `field_institution` — institution (existing)
- `field_hpc_experience` — HPC experience level (existing)
- Community Persona flags — skills, interests (existing)
- Affinity group memberships + coordinator status — via `mcp_my_affinity_groups` view (existing)

**Agent `/personalized` response (Phase 1):**

```json
{
  "user": {
    "name": "Drew Pasquale",
    "access_id": "apasquale@access-ci.org"
  },
  "highlighted_capabilities": [
    {
      "id": "manage_announcements",
      "label": "Manage Pegasus announcements",
      "reason": "You coordinate the Pegasus affinity group"
    }
  ],
  "context": {
    "institution": "University of Example",
    "hpc_experience": "Familiar with Slurm...",
    "skills": ["python", "machine_learning", "genomics"],
    "interests": ["deep_learning", "bioinformatics"],
    "affinity_groups": [
      {"name": "AI Institute", "is_coordinator": false},
      {"name": "Pegasus", "is_coordinator": true}
    ],
    "active_allocations": [
      {"resource": "Delta", "project": "TG-CIS123456"}
    ]
  }
}
```

### Future: MCP Server Wrapper

If other AI tools (Claude Desktop, ChatGPT, etc.) need access to personalization data, wrap the same Drupal calls in an MCP server:

| Tool | Auth | Description |
|------|------|-------------|
| `get_researcher_profile` | Required | Fetch user's personalization context |

This is not needed for Phase 1 (chatbot-only) but can be added later within the existing MCP auth flow.

---

## Privacy & Security

### Access Control

| Data | Who Can Read | Who Can Write |
|------|--------------|---------------|
| Own profile | User, System | User, System (derived) |
| Other's profile | Nobody | Nobody |
| Aggregated analytics | Admins | System |

### Drupal Permissions

```php
// New permissions
'view own ai profile' => 'View own AI assistant profile',
'edit own ai profile' => 'Edit own AI assistant profile',
'administer ai profiles' => 'Administer AI profiles (admin only)',
```

### Data Protection

| Measure | Implementation |
|---------|----------------|
| Encryption at rest | Drupal database encryption |
| Access logging | Log all profile reads/writes |
| Retention limits | Auto-expire derived data after 90 days |
| Deletion | Complete removal on user request |

### Consent Model

| Data Type | Consent | Withdrawal |
|-----------|---------|------------|
| Explicit settings | Implied (user enters) | Delete anytime |
| Derived from usage | Requires opt-in | Clear anytime |
| Community Persona integration | Existing consent | Via Community Persona |

---

## Relationship to Capability Registry

The capability registry spec defines a `/api/v1/capabilities/personalized` endpoint on the access-agent that fetches user context for the agent and UI. This is the integration point for personalization — it starts with existing Drupal data and grows as profile features are built.

| Phase | `/personalized` returns | Data source |
|-------|------------------------|-------------|
| v1 (now) | Allocations, affinity group membership + coordinator status, skills, interests, institution, HPC experience | Existing Drupal fields |
| v2 (future) | + Research domain, expertise level, response preferences | New Drupal fields |
| v3 (future) | + Conversation summaries, topic history, resource preferences | Derived from interactions |

The agent fetches personalization data directly from Drupal (JSON:API) — no MCP server in between. The agent already has the user's JWT cookie for authentication. MCP wrapping can be added later if other AI tools need access.

No separate `/api/ai-profile` endpoint needed — the agent's `/personalized` endpoint is the single integration point that aggregates data from Drupal.

## Implementation Plan

### Phase 1: Opt-In Toggle + Existing Data

**Drupal:**
- [ ] Add `field_ai_profile_enabled` boolean to user entity (single toggle, default off)
- [ ] Expose toggle on community persona page

**Agent (`/api/v1/capabilities/personalized`):**
- [ ] Build the endpoint (not yet implemented)
- [ ] Check `field_ai_profile_enabled` via Drupal JSON:API before returning data
- [ ] If disabled (default): return `{"highlighted_capabilities": [], "context": {}}`
- [ ] If enabled: fetch and return from existing Drupal fields:
  - Active allocations (from `field_cider_resources`)
  - Affinity group memberships + coordinator status (from `mcp_my_affinity_groups` view)
  - Skills and interests (from Community Persona flags)
  - Institution (from `field_institution`)
  - HPC experience (from `field_hpc_experience`)
  - User name (for personalized greeting)
- [ ] Cache results per-user with 5-minute TTL
- [ ] Graceful degradation: if any Drupal call fails, return partial data

**Deliverable**: Users opt in to personalization using data that already exists. No new data collection.

### Phase 2: Agent Integration

- [ ] Inject personalized context into agent system prompt
- [ ] Personalized welcome greeting (use name if opted in)
- [ ] Placeholder text rotation with contextual suggestions ("Try: 'check my usage on Delta'")
- [ ] Highlighted capabilities based on user context (e.g., "Manage Pegasus announcements" for coordinators)

**Deliverable**: Agent provides personalized responses for opted-in users

### Phase 3: User-Provided Preferences (Future)

- [ ] Add new Drupal fields: research domain, expertise level, response preferences
- [ ] Build settings form at `/community-persona/ai-settings`
- [ ] Agent tailors response style (detail level, examples) based on preferences
- [ ] Export functionality

**Deliverable**: Users explicitly customize their AI experience

### Phase 4: Derived Data (Future)

- [ ] Topic extraction from conversations
- [ ] Resource preferences from usage patterns
- [ ] LLM-generated context summaries
- [ ] Retention/expiry system (90 days for derived data)

**Deliverable**: Profiles improve automatically from interactions

### Phase 5: Allocation Streamlining (Future)

- [ ] Pre-fill allocation forms from profile
- [ ] Estimate SUs from usage history
- [ ] Draft abstracts from context summary

**Deliverable**: Simplified allocation requests

---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Profile adoption | ≥30% of QA Bot users | Users with profile enabled |
| Personalization satisfaction | ≥80% positive | Feedback on personalized responses |
| Settings engagement | ≥20% modify settings | Users who edit profile |
| Opt-out rate | <15% | Users who disable after trying |
| Data export usage | Track | Number of exports requested |
| Deletion requests | Track | Should be rare if value is clear |

---

## Open Questions

| Area | Question | Considerations |
|------|----------|----------------|
| Storage | Drupal fields vs separate entity? | Fields simpler; entity if profile grows complex |
| Sync | How often to update derived data? | Per-session? Daily batch? |
| Cross-session | How to handle multiple concurrent sessions? | Last-write-wins? Merge? |
| Sharing | Should PIs be able to share profiles with group? | Privacy implications |
| Migration | Import from other systems (XSEDE profile)? | One-time migration path |

---

## References

- [MyData Declaration](https://mydata.org/participate/declaration/) - Human-centric personal data principles
- [GDPR Privacy by Design](https://secureprivacy.ai/blog/privacy-by-design-gdpr-2025) - Privacy-first design principles
- [GDPR Data Governance Best Practices](https://www.alation.com/blog/gdpr-data-compliance-best-practices-2025/) - Data minimization, purpose limitation
- Existing Community Persona: `cssn` module in `cyberteam_drupal`
