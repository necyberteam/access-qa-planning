# Proposed Additions to ACCESS Data Sharing and Privacy Policy

*Draft: February 2026, updated April 2026*

The ACCESS privacy policy (v1.4, December 2025) does not address the AI-powered Q&A assistant. These are proposed additions to the main policy at [access-ci.org/privacy-policy/](https://access-ci.org/privacy-policy/).

**Status:** Pending review by ACCESS cybersecurity group. The agent is deploying to production — these updates should be reviewed before or concurrent with launch.

For the full AI Assistant Privacy Notice and tool information page drafts, see:
- [`pages-current-production.md`](../archive/pages-current-production.md) — current production version
- [`pages-access-qa-tool.md`](../archive/pages-access-qa-tool.md) — future agentic version

---

## Proposed new section: 4.7 AI Assistant Interactions

Add under **Section 4: Information ACCESS Collects**:

> **4.7 AI Assistant Interactions**
>
> ACCESS operates an AI-powered assistant, embedded on ACCESS websites, to help users find information about ACCESS services and resources. The assistant:
>
> - Processes user questions using third-party large language model services. ACCESS requires contractual commitments from these providers that ACCESS user data will not be used to train their models.
> - May query ACCESS services on behalf of authenticated users to answer questions or perform actions the user has explicitly requested, such as checking allocations or opening support tickets.
> - Logs query metadata (including a pseudonymous user identifier where applicable) for usage reporting, quality evaluation, and security monitoring, consistent with the rest of this Section.
> - Stores conversation content temporarily to provide context for follow-up questions within a session.
> - May, with explicit user opt-in, retain personalization context such as preferences and recent topics to improve future responses. Users may view, export, or delete this context at any time through their ACCESS account settings.
>
> The assistant is designed to support, not replace, the ACCESS support team. Users can reach human support at any time by opening a ticket on the ACCESS support website. AI-generated responses may be inaccurate and are not a substitute for official ACCESS documentation or human support. For details on data flows, providers, retention periods, and deletion procedures, see the [AI Assistant Privacy Notice](https://support.access-ci.org/tools/access-qa-tool/privacy).

## User guidance for the AI Assistant Privacy Notice

The following content is drafted for inclusion in the subordinate AI Assistant Privacy Notice at `support.access-ci.org/tools/access-qa-tool/privacy`. It tells users what to share with the assistant and acknowledges that users at some institutions are subject to their own AI use policies.

### Guidance on what to share with the assistant

> The ACCESS AI Assistant is designed to answer questions about publicly available ACCESS information — resources, software, events, allocations, documentation, and similar topics. You do not need to share personal or sensitive information to get useful answers.
>
> **Do not paste the following into the chat:**
>
> - Allocation request proposal text or supporting documentation (see the [ACCESS Acceptable Use Policy](https://access-ci.org/acceptable-use/))
> - Unpublished research data or findings
> - Passwords, SSH keys, API tokens, or other credentials
> - Personally identifiable information about other people
> - Data that is regulated under HIPAA, FERPA, export controls, or similar frameworks
> - Content that is confidential or restricted under your institution's data classification policies
>
> If you are uncertain whether something is appropriate to share, err on the side of not sharing it. You can usually rephrase the question to use public information only (for example, "how do I submit a Slurm job on Delta?" instead of pasting your job script).
>
> Authenticated users may share information ACCESS already holds about them — such as their allocations, affinity group memberships, and account history — to receive personalized answers. ACCESS does not use this information to train language models.

### Relationship to other institutional AI policies

> The ACCESS AI Assistant is operated by ACCESS as an institutional service. Individual users and their home institutions may be subject to their own AI use policies — for example, the [University of Colorado Boulder Guidelines on Ethical and Responsible Use of Generative AI](https://www.colorado.edu/bfa/media/1699). These policies typically restrict what kinds of data users may share with AI tools and may require users to disclose AI use in academic or research work.
>
> ACCESS provides the disclosures and guidance in this notice so users can make informed decisions under their own institutional rules. ACCESS does not enforce other institutions' policies on behalf of those institutions. If you are uncertain whether using the ACCESS AI Assistant is appropriate for a particular task, consult your institution's IT, data governance, or research integrity office.
>
> If you use information from the ACCESS AI Assistant in written work where AI-use disclosure is required, a suggested citation format is:
>
> > *ACCESS AI Assistant, queried YYYY-MM-DD. Response verified against [source URL].*
>
> Always verify AI-generated content against official ACCESS documentation or human support before relying on it in a decision that matters.

---

## Acceptable Use Policy reconciliation

The current [ACCESS Acceptable Use Policy](https://access-ci.org/acceptable-use/) prohibits uploading allocation request proposals and supporting documentation to "non-approved generative AI tools." The ACCESS AI Assistant is itself a generative AI tool, operated by ACCESS, so this language needs to be reconciled before launch. Options:

1. **Classify the ACCESS AI Assistant as an approved tool** via internal designation, leaving the AUP language unchanged. Requires a documented approval decision.
2. **Amend the AUP** to clarify the distinction between ACCESS-operated AI services and external tools. Example clarifying language:
   > *"The ACCESS AI Assistant is an approved ACCESS service. Users are still prohibited from uploading allocation request proposals and supporting documentation to non-approved generative AI tools operated by third parties."*

Recommended: Option 2. This makes the distinction explicit in the policy itself, rather than relying on an unwritten approval list. The existing in-product disclosure in `@access-ci/ui` qa-bot.jsx already warns users about sensitive content; no additional in-product changes are required.

## Proposed new section: 6.6 AI Assistant Data

Add under **Section 6: Information ACCESS Shares**:

> **6.6 AI Assistant Data**
>
> Questions submitted to the ACCESS AI Assistant are transmitted to third-party large language model providers to generate responses, under contracts prohibiting use of ACCESS user data for model training. Information shared with these providers is subject to their own privacy commitments in addition to this policy. Queries made by the assistant on the user's behalf to ACCESS services, and usage analytics events that do not contain question text or personally identifiable information, are governed by the rest of this policy. See the [AI Assistant Privacy Notice](https://support.access-ci.org/tools/access-qa-tool/privacy) for the current list of providers and details on data flows.

---

## LLM provider contract requirements

Every LLM provider in the data path must have a written commitment covering:

1. **No training on ACCESS data.** The provider will not use prompts, completions, or any derived artifacts to train, fine-tune, or improve models.
2. **Data retention limits.** How long the provider retains request/response data, and a commitment to delete it. Zero data retention (ZDR) is preferred where available.
3. **Sub-processors.** Disclosure of any downstream processors, with the same commitments flowed through.
4. **Security and breach notification.** Standard infosec controls and a commitment to notify ACCESS within a defined window if user data is exposed.

### Current provider chain

`user → access-agent → UKY RAG service → OpenAI API`

- **UKY RAG service** — hosted at the University of Kentucky, which is an ACCESS Resource Provider. Data handling should be covered by the existing ACCESS ↔ UKY Resource Provider agreement; this should be confirmed in writing with ACCESS cybersecurity and documented in an internal memorandum describing the full data flow.
- **OpenAI API** — covered by [OpenAI's API data usage policy](https://openai.com/enterprise-privacy/): API data is not used for model training, and zero-data-retention is available for approved use cases. Requires a signed DPA on file with ACCESS.

### What is sent to the LLM provider

- **Today:** The text the user types into the chat, plus ACCESS documentation context retrieved from the UKY RAG vector store. ACCESS does not send user identifiers, email addresses, session cookies, or other PII beyond what the user themselves chooses to include in their question.
- **Planned (Phase 2+ personalization, opt-in):** In addition to the above, public profile information the user has already chosen to expose via their [Community Persona](https://support.access-ci.org/community-persona) — such as institution, skills, interests, affinity group memberships, HPC experience level, and active allocations. This data is already publicly visible on the user's Community Persona page; personalization does not broaden its exposure. Users opt in per the [Researcher Profile](09-researcher-profiles.md) design and can disable or delete it at any time.

### Questions for ACCESS cybersecurity

- Does the existing ACCESS ↔ UKY Resource Provider agreement cover LLM processing of ACCESS user queries, or is a supplemental memorandum needed?
- Is a signed DPA with OpenAI required on file, or is OpenAI's public API data usage policy sufficient disclosure? Do we need to request zero data retention?
- Should the sub-processor chain (ACCESS → UKY → OpenAI) be documented in the AI Assistant Privacy Notice with each named explicitly, or described generically?

---

## Implementation Checklist

### Policy and content

- [x] Draft in-product disclosure (implemented in `@access-ci/ui` qa-bot.jsx)
- [x] Draft AI Assistant Privacy Notice — current and future versions
- [x] Draft main privacy policy amendments (above)
- [ ] Review with ACCESS cybersecurity group and legal counsel
- [ ] Reconcile Acceptable Use Policy language re: generative AI tools (see section above)
- [ ] Confirm UKY Resource Provider agreement covers LLM processing, or obtain supplemental memorandum
- [ ] Confirm OpenAI data handling posture (DPA on file and/or zero-data-retention if required)
- [ ] Create `/tools/access-qa-tool/privacy` page on support.access-ci.org
- [ ] Update `/tools/access-qa-tool` page content on support.access-ci.org
- [ ] Add sections 4.7 and 6.6 to main privacy policy at access-ci.org
- [ ] Update support.access-ci.org/privacy-policy to list SESSaccess_auth cookie (when agent deploys)

### Technical

- [ ] Publish updated `@access-ci/ui` package (in-product disclosure)
- [ ] Update `headerfooter.js` to reference new package version
- [ ] Implement data retention TTLs (when access-agent deploys)
- [ ] Add data deletion procedure for user requests

### Notification

- [ ] Communicate privacy policy update to ACCESS user community per policy Section 9 (Governance)

---

## References

- [ACCESS Data Sharing and Privacy Policy v1.4](https://access-ci.org/privacy-policy/)
- [NIST AI Risk Management Framework (AI 100-1)](https://nvlpubs.nist.gov/nistpubs/ai/nist.ai.100-1.pdf)
- [NIST Generative AI Profile (AI 600-1)](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf)
- [FTC: Operation AI Comply](https://www.ftc.gov/business-guidance/blog/2024/09/operation-ai-comply-continuing-crackdown-overpromises-ai-related-lies)
- [EU AI Act Article 50 (Transparency)](https://artificialintelligenceact.eu/article/50/)
- [U-M ITS AI Services Privacy Notice](https://its.umich.edu/computing/ai/privacy-notice)
- [OpenAI Enterprise Privacy](https://openai.com/enterprise-privacy/)
- [Anthropic Privacy Center](https://privacy.claude.com/)
- [Google Analytics Data Practices](https://support.google.com/analytics/answer/6004245)
- California SB 243, Maine LD 1916, Utah SB 131, New York A8209 (state chatbot disclosure laws)
