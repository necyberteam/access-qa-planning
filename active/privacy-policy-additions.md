# Proposed Additions to ACCESS Data Sharing and Privacy Policy

*Draft: February 2026*

The ACCESS privacy policy (v1.4, December 2025) does not address the AI-powered Q&A assistant. These are proposed additions to the main policy at [access-ci.org/privacy-policy/](https://access-ci.org/privacy-policy/).

For the full AI Assistant Privacy Notice and tool information page drafts, see:
- [`pages-current-production.md`](../archive/pages-current-production.md) — current production version
- [`pages-access-qa-tool.md`](../archive/pages-access-qa-tool.md) — future agentic version

---

## Proposed new section: 4.7 AI Assistant Interactions

Add under **Section 4: Information ACCESS Collects**:

> **4.7 AI Assistant Interactions**
>
> ACCESS provides an AI-powered assistant on its support website to help users find information about ACCESS services and resources. When users interact with the assistant:
>
> - The text of user questions is processed by a large language model (LLM) provided by a third-party service. The LLM provider does not use ACCESS user data to train its models.
> - If the user is logged in, their ACCESS ID may be used to provide personalized responses and to attribute any content created through the assistant.
> - Conversation content is stored temporarily to provide context for follow-up questions within a session.
> - Query metadata (topic, tools used, timing) is logged with a hashed, non-reversible user identifier for aggregate usage reporting.
> - AI-generated responses may be inaccurate. The assistant is not a substitute for official ACCESS documentation or human support.
>
> For full details, see the [AI Assistant Privacy Notice](https://support.access-ci.org/tools/access-qa-tool/privacy).

## Proposed new section: 6.6 AI Assistant Data

Add under **Section 6: Information ACCESS Shares**:

> **6.6 AI Assistant Data**
>
> User questions submitted to the ACCESS AI Assistant are processed by a third-party large language model provider to generate responses. This data is transmitted via API and is not used by the provider for model training. Usage analytics events (not containing question text or personally identifiable information) are sent to Google Analytics 4 via Google Tag Manager. See the [AI Assistant Privacy Notice](https://support.access-ci.org/tools/access-qa-tool/privacy) for details.

---

## Implementation Checklist

### Policy and content

- [x] Draft in-product disclosure (implemented in `@access-ci/ui` qa-bot.jsx)
- [x] Draft AI Assistant Privacy Notice — current and future versions
- [x] Draft main privacy policy amendments (above)
- [ ] Review with ACCESS leadership and legal counsel
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
