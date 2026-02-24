# ACCESS Q&A Tool — Page Drafts

*Draft: February 2026*

These are proposed content for two pages on support.access-ci.org:

1. **`/tools/access-qa-tool`** — About the tool (replaces existing page)
2. **`/tools/access-qa-tool/privacy`** — AI privacy notice (new page)

---

## Page 1: ACCESS Q&A Tool Information

**Path:** `/tools/access-qa-tool` (replaces existing content)

---

### ACCESS Q&A Tool

The ACCESS Q&A tool is an AI-powered assistant available on the ACCESS Support portal. It helps users find information about ACCESS services, resources, policies, and procedures.

#### What the tool does

The assistant can:

- **Answer questions** about ACCESS services, allocations, resource providers, and policies using a curated knowledge base of ACCESS documentation
- **Look up real-time information** including system status, upcoming events, announcements, resource availability, NSF award details, and usage metrics
- **Help you take action** such as creating announcements or submitting support tickets (when logged in)
- **Provide usage and performance data** from XDMoD for ACCESS-allocated resources

The assistant is available through the chat widget in the lower-right corner of the support portal, or embedded directly on specific pages.

#### How it works

When you ask a question, the assistant:

1. **Classifies your question** to determine whether it can be answered from documentation, requires live data, or both
2. **Retrieves relevant information** from the ACCESS knowledge base (a curated set of Q&A pairs from ACCESS documentation) and, when needed, queries live ACCESS data services
3. **Generates a response** using a large language model (LLM) that synthesizes the retrieved information into a natural-language answer
4. **Evaluates the response** for quality and completeness before presenting it to you

#### Technology

The tool is built on several components:

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **AI agent** | LangGraph (Python) | Orchestrates the question-answering workflow: classification, retrieval, tool use, and response synthesis |
| **Large language model** | OpenAI GPT-4o (configurable) | Generates natural-language responses from retrieved data and documentation |
| **Knowledge base** | Curated Q&A pairs with semantic search | Provides verified answers to common ACCESS questions |
| **Data services** | Model Context Protocol (MCP) servers | Provide real-time access to ACCESS systems: status, allocations, events, announcements, XDMoD, NSF awards, and more |
| **Frontend** | React (react-chatbotify) | Chat interface with support for guided flows, file uploads, and interactive menus |

The LLM provider is configurable and may change over time. Self-hosted model options (ACCESS AI, vLLM) are available and keep all data within ACCESS-managed infrastructure.

#### Limitations

- **Responses are AI-generated and may not always be accurate.** The assistant draws on ACCESS documentation and live data, but the LLM may misinterpret information or produce incomplete answers. Always verify important information through official ACCESS channels.
- **The assistant is not a human support agent.** For complex issues, account problems, or urgent matters, [open a support ticket](https://support.access-ci.org/open-a-ticket).
- **Knowledge has boundaries.** The assistant knows about ACCESS services and policies. It cannot answer questions about the internal workings of specific resource provider systems, individual research projects, or topics outside the ACCESS ecosystem.
- **Do not share sensitive information.** Do not enter passwords, SSH keys, API tokens, or other secrets in the chat.

#### Feedback

Your feedback helps us improve the tool. If you encounter an inaccurate response, you can rate it directly in the chat using the thumbs up/down buttons. For general feedback about the tool, please [open a support ticket](https://support.access-ci.org/open-a-ticket) with "Q&A Tool Feedback" in the subject line.

#### Privacy

The assistant collects your questions and, if you are logged in, your ACCESS ID to provide personalized responses. Questions are processed by a third-party LLM provider (currently OpenAI) that does not use your data for model training. For full details, see the [AI Assistant Privacy Notice](/tools/access-qa-tool/privacy).

---

## Page 2: AI Assistant Privacy Notice

**Path:** `/tools/access-qa-tool/privacy` (new page)

---

### AI Assistant Privacy Notice

*Last updated: [date]*

This notice describes how the ACCESS Q&A tool ("the assistant") collects, uses, and protects information from your interactions. This notice supplements the [ACCESS Data Sharing and Privacy Policy](https://access-ci.org/privacy-policy/). For general information about the tool, see [ACCESS Q&A Tool Information](/tools/access-qa-tool).

#### What the assistant is

The ACCESS Q&A tool is an AI-powered assistant that answers questions about ACCESS services, resources, and policies. It uses a large language model (LLM) to generate responses based on ACCESS documentation and real-time data from ACCESS services. It is not a human support agent.

Responses are generated by AI and may contain errors. The assistant is not a substitute for official ACCESS documentation, support tickets, or direct communication with ACCESS staff.

#### What information is collected

When you use the assistant, the following information is collected:

| Data | Description | Purpose |
|------|-------------|---------|
| **Your questions** | The text of each message you send | Processed by the LLM to generate a response |
| **Conversation history** | Prior messages in the current session | Provides context for follow-up questions |
| **ACCESS ID** | Your identity if you are logged in (e.g., `username@access-ci.org`) | Enables personalized responses (e.g., your allocations, your announcements) and attributes content you create through the assistant |
| **Session identifier** | A random ID stored in your browser | Groups messages into a conversation; does not identify you personally |
| **Timestamps** | When each message was sent | Operational monitoring and usage reporting |
| **Query classification** | What topic your question was about (e.g., "allocations", "system status") | Usage analytics and service improvement |

#### What information is NOT collected

- The assistant does not access your research data, files, or compute jobs unless you explicitly ask about them.
- The assistant does not record your IP address in its application logs.
- The assistant does not track your activity across websites or build a profile of your browsing behavior. A hashed, non-reversible identifier derived from your ACCESS ID is stored in usage logs for aggregate reporting (e.g., counting unique users). This identifier cannot be used to recover your ACCESS ID or link your queries to your identity.

#### How your information is used

Your information is used to:

- **Generate responses** to your questions using AI
- **Retrieve relevant data** from ACCESS services (system status, allocations, events, etc.) on your behalf
- **Improve response quality** through aggregate analysis of question types, tool usage, and response accuracy
- **Produce usage reports** for NSF and ACCESS leadership — reports contain only aggregate statistics, not individual conversations
- **Monitor for errors and abuse** to maintain service quality and security

Your information is **not** used to:

- Train or fine-tune AI models
- Target advertising or marketing
- Build user profiles beyond the current session
- Share with third parties for their own purposes

#### Third-party data processing

Your questions and conversation context are sent to external services for processing:

| Service | What is shared | Retention by service | Used for training? |
|---------|---------------|----------------------|--------------------|
| **OpenAI API** | Question text, conversation context, and data retrieved from ACCESS services | Up to 30 days for abuse monitoring, then deleted ([OpenAI API data policy](https://openai.com/enterprise-privacy/)) | **No.** OpenAI does not train on API data by default. |
| **Honeycomb** (operational monitoring) | First 200 characters of question text, session ID, tool names, response timing | Per retention settings (typically 60 days) | No |

The LLM provider may change over time. Possible providers include OpenAI, Anthropic, and self-hosted models (ACCESS AI, vLLM). This notice will be updated to reflect the current provider.

When ACCESS uses self-hosted models, your data does not leave ACCESS-managed infrastructure. Sensitive values in tool parameters (passwords, tokens, credentials) are automatically redacted before being sent to monitoring services.

#### How your information is stored

| Data store | Contents | User identification | Retention period |
|------------|----------|---------------------|------------------|
| **Conversation checkpoints** | Full conversation state including messages and tool results | ACCESS ID (plaintext) | 90 days |
| **Usage logs** | Question text, topic, tools used, response length, timing | Hashed user ID (SHA-256, not reversible) | 18 months |
| **Application logs** | First 50 characters of question, session ID | ACCESS ID | 30 days |
| **Browser storage** | Session ID only (random UUID) | Not personally identifiable | Until you clear browser data |

All server-side data is stored in a PostgreSQL database on ACCESS-managed infrastructure.

#### Authentication

If you are logged in to an ACCESS site, the assistant identifies you through a secure cookie set by the ACCESS website during login. This cookie:

- Contains only your ACCESS ID and an expiration time
- Is signed cryptographically and verified by the assistant — it cannot be forged
- Is not stored by the assistant beyond the duration of your request

If you are not logged in, the assistant operates anonymously. Some features (such as viewing your allocations or managing announcements) require authentication.

#### Content you create

When logged in, you can use the assistant to create or edit content (such as announcements or support tickets) on your behalf. This content is attributed to your ACCESS identity and governed by standard ACCESS content policies.

#### Your rights

You may:

- **Request access to or deletion of your conversation history** by submitting a ticket at [support.access-ci.org/open-a-ticket](https://support.access-ci.org/open-a-ticket) with the subject "AI Assistant Data Request"
- **Opt out** of the assistant entirely — all ACCESS support services remain available through the [help desk](https://support.access-ci.org) without AI processing
- **Request removal of personal information** per the [ACCESS Data Sharing and Privacy Policy](https://access-ci.org/privacy-policy/) Section 4.7

Usage log entries contain a hashed user identifier that cannot be reversed to your ACCESS ID. These anonymized entries cannot be attributed to individuals and are not subject to individual deletion requests.

#### Changes to this notice

ACCESS will update this notice when the assistant's data practices change materially. Updates will be noted with a revised "last updated" date. Continued use of the assistant after updates constitutes acceptance of the revised notice.

#### Contact

Questions about this notice or the assistant's data practices may be directed to [help@access-ci.org](mailto:help@access-ci.org) or submitted through the [ACCESS help desk](https://support.access-ci.org/open-a-ticket).

---

## Summary of Changes from Current Page

The current `/tools/access-qa-tool` page is replaced entirely. Key differences:

| Aspect | Current page | Updated page |
|--------|-------------|--------------|
| **Framing** | "developmental phase" | Production tool with clear capabilities and limitations |
| **LLM reference** | "ChatGPT", "GPT-3.5 Turbo" | "OpenAI GPT-4o (configurable)" — acknowledges provider may change |
| **Tech stack** | ChromaDB, llamaindex | LangGraph agent, MCP servers, curated Q&A knowledge base |
| **Privacy** | "We do not collect any information regarding the user's identity" | Accurate disclosure: ACCESS ID collected when logged in, questions stored, user hash for analytics |
| **Third-party sharing** | Not mentioned | OpenAI API and Honeycomb explicitly disclosed with retention periods |
| **Data retention** | Not mentioned | Specific retention periods for each data store |
| **User rights** | Not mentioned | Access, deletion, and opt-out procedures documented |
| **Limitations** | Brief accuracy caveat | Clear section on what the tool can and cannot do |
| **Privacy notice** | Three sentences inline | Dedicated page with full disclosure |

## Chatbot Links

The in-product notices (welcome message and footer) link to `/tools/access-qa-tool/privacy` rather than a separate `/ai-privacy` path, keeping everything under the existing tool page hierarchy.
