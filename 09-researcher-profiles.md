# Researcher Profiles

> **Related**: [Agent Architecture](./01-agent-architecture.md) | [MCP Authentication](./06-mcp-authentication.md)

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

### Drupal JSON:API Extension

Expose profile data via JSON:API for the MCP server:

**Endpoint**: `GET /jsonapi/user/user/{uuid}?include=field_ai_*`

Or create a custom lightweight endpoint:

**Endpoint**: `GET /api/ai-profile`

```json
{
  "enabled": true,
  "research_domain": ["bioinformatics", "machine_learning"],
  "expertise_level": "intermediate",
  "preferences": {
    "detail_level": "moderate",
    "include_examples": true,
    "show_su_estimates": true
  },
  "context_summary": "Researcher working on genomics pipelines...",
  "recent_topics": ["pytorch", "gpu_memory", "distributed_training"],
  "preferred_resources": ["delta.ncsa.access-ci.org", "expanse.sdsc.access-ci.org"],
  "community_persona": {
    "skills": ["python", "machine_learning", "genomics"],
    "interests": ["deep_learning", "bioinformatics"],
    "affinity_groups": ["ai_institute"],
    "hpc_experience": "Familiar with Slurm..."
  }
}
```

**Update Endpoint**: `PATCH /api/ai-profile`

```json
{
  "recent_topics": ["new_topic"],
  "context_summary": "Updated summary..."
}
```

### MCP Tool (Alternative)

If using MCP for profile access:

| Tool | Auth | Description |
|------|------|-------------|
| `get_researcher_profile` | Required | Fetch user's AI profile |
| `update_researcher_profile` | Required | Update derived fields |

This keeps profile access within the existing MCP auth flow.

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

## Implementation Plan

### Phase 1: Drupal Foundation

- [ ] Create new user fields (`field_ai_*`)
- [ ] Build AI settings form at `/community-persona/ai-settings`
- [ ] Implement profile enable/disable toggle
- [ ] Add export functionality
- [ ] Integrate with existing Community Persona page

**Deliverable**: Users can set explicit profile preferences

### Phase 2: API Exposure

- [ ] Create `/api/ai-profile` endpoints (GET/PATCH)
- [ ] Add authentication (require logged-in user)
- [ ] Implement rate limiting
- [ ] Add to MCP server as tool (or direct HTTP)

**Deliverable**: Agent can read/write profile data

### Phase 3: Agent Integration

- [ ] Fetch profile in agent initialization
- [ ] Inject profile context into system prompt
- [ ] Implement personalized response synthesis
- [ ] Add profile-aware query expansion

**Deliverable**: Agent provides personalized responses

### Phase 4: Derived Data

- [ ] Implement topic extraction from conversations
- [ ] Track resource preferences from MCP calls
- [ ] Generate context summaries (LLM)
- [ ] Build retention/expiry system

**Deliverable**: Profiles improve automatically

### Phase 5: Allocation Streamlining

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
