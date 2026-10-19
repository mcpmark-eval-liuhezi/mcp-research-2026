# Model Context Protocol (MCP) — Research Notes

> Compiled for the internal team talk. Tracked in issue #1; follow-up work in sub-issues #2, #3, #4.
> Research performed October 2026.

## 1. What is MCP, and who created it?

The **Model Context Protocol (MCP)** is an **open standard / open-source protocol for connecting AI applications (LLMs) to external systems** — data sources, tools, and workflows.

- **Created at Anthropic by David Soria Parra and Justin Spahr-Summers** (named explicitly in Anthropic's announcement post).
- **Announced and open-sourced on November 25, 2024** — this is the date Anthropic's own announcement post carries, and it is corroborated by contemporaneous press coverage (VentureBeat, TechCrunch-era reporting all dated Nov 25, 2024).
- Launched with three components: the specification and SDKs, local MCP server support in the Claude Desktop apps, and an open-source repository of MCP servers (Google Drive, Slack, GitHub, Git, Postgres, Puppeteer were the pre-built examples).

The most common mental model (used in the official docs): **MCP is a USB-C port for AI applications** — a standardized way to connect AI apps to external systems, just as USB-C standardizes device connections.

## 2. The problem it solves

Anthropic's framing: even the most sophisticated models are **constrained by their isolation from data** — trapped behind information silos and legacy systems. Before MCP:

- Every new data source required its own **custom, bespoke integration**.
- With *N* models and *M* tools, you needed up to **N × M custom connectors**, which doesn't scale ("fragmented integrations").
- Truly connected systems were therefore difficult to build and maintain.

MCP's answer: **replace fragmented integrations with a single universal protocol.** Developers either expose their data/tools through an **MCP server** (write once), or build AI applications (**MCP clients/hosts**) that can connect to any MCP server. The result is a simpler, more reliable way to give AI systems access to the data and capabilities they need, with context preserved as agents move between tools.

Benefits by stakeholder (official docs):
- **Developers**: less development time/complexity when building or integrating AI applications.
- **AI applications/agents**: access to an ecosystem of data sources, tools, and apps.
- **End users**: more capable AI applications that can act on their behalf.

## 3. How the spec and SDKs are maintained

### Where it lives
- The specification, documentation, and SDKs are hosted under the **`modelcontextprotocol` GitHub organization**: https://github.com/modelcontextprotocol
  - `modelcontextprotocol/modelcontextprotocol` — spec and documentation
  - Per-language SDK repositories, plus `servers` (maintained server list), `inspector` (visual testing tool), `conformance` (conformance tests)

### Governance: Linux Foundation / Agentic AI Foundation
- On **December 9, 2025**, Anthropic **donated MCP to the newly created Agentic AI Foundation (AAIF)**, a **directed fund under the Linux Foundation**, co-founded by **Anthropic, Block, and OpenAI**, with support from Google, Microsoft, AWS, Cloudflare, and Bloomberg.
- Founding projects of the AAIF: **MCP**, Block's **goose**, and OpenAI's **AGENTS.md**.
- Division of responsibility: the **AAIF Governing Board** decides strategic investments, budget allocation, member recruitment, and new-project approval; **individual projects retain full autonomy over technical direction** (MCP's maintainer structure and SEP — Spec Evolution Process — are unchanged).
- The Linux Foundation provides neutral stewardship in the same vein as Kubernetes, PyTorch, and Node.js; Anthropic continues to invest and maintain core infrastructure.
- At the time of the donation, MCP reported **~97 million monthly SDK downloads** and **~10,000 active servers**.

### Official SDKs
Ten official SDKs, several maintained in collaboration with major vendors:

| Language | Repo | Co-maintainer |
|----------|------|---------------|
| TypeScript | `typescript-sdk` | — |
| Python | `python-sdk` | — |
| Java | `java-sdk` | — |
| Kotlin | `kotlin-sdk` | **JetBrains** |
| C# | `csharp-sdk` | **Microsoft** |
| Go | `go-sdk` | **Google** |
| PHP | `php-sdk` | **PHP Foundation** |
| Ruby | `ruby-sdk` | — |
| Rust | `rust-sdk` | — |
| Swift | `swift-sdk` | — |

Protocol versions are **dated strings** (e.g. `2025-06-18`, `2026-07-28`), and the data layer is a JSON-RPC 2.0-based protocol with capability and version discovery.

## 4. Major vendor adoption

First-class MCP client support across major AI platforms (per the official MCP blog, Dec 2025): **ChatGPT, Claude, Cursor, Gemini, Microsoft Copilot, Visual Studio Code**.

Key milestones:

| Date | Vendor / event |
|------|----------------|
| **Nov 25, 2024** | **Anthropic** announces and open-sources MCP |
| **Mar 26, 2025** | **OpenAI** adopts MCP across its products, including the ChatGPT desktop app; Agents SDK gains MCP support |
| **Apr 1, 2025** | **AWS** — Amazon Bedrock agents can use MCP servers |
| **Apr 2, 2025** | **Microsoft** partners with Anthropic on the official C# SDK |
| **May 20, 2025** | **Google I/O** — MCP becomes built into the Gemini API and SDK (hinted earlier by Demis Hassabis) |
| **Jul → Nov 2025** | Microsoft ships the **Azure DevOps MCP Server** (public preview July 2025 → GA Nov 2025) |
| **Dec 9, 2025** | MCP donated to the **Agentic AI Foundation** (Linux Foundation) |
| **Apr 2026** | Azure MCP tools ship built into **Visual Studio 2022** (no extension needed) |

Additional ecosystem: Block (early adopter), Apollo, Zed, Replit, Codeium, Sourcegraph, JetBrains, Cloudflare (Workers + hosted remote MCP servers), Cursor. Microsoft Windows also gained local MCP servers for desktop AI apps.

## 5. Core building blocks

### Participants (client-server architecture)

| Component | Role |
|-----------|------|
| **MCP Host** | The AI application (e.g. Claude Desktop, Claude Code, VS Code) that coordinates and manages one or more MCP clients |
| **MCP Client** | Maintains a dedicated connection to one MCP server; obtains context for the host |
| **MCP Server** | A program that provides context, tools, and prompts to clients |

- A host creates **one client per server**; each client maintains a dedicated connection.
- **Local** servers (stdio transport) typically serve a single client; **remote** servers (Streamable HTTP) typically serve many clients.

### Two layers

1. **Data layer** — a **JSON-RPC 2.0-based** exchange protocol:
   - **Discovery**: clients query a server's supported protocol versions, capabilities, and identity
   - **Server features**: tools, resources, prompts
   - **Client features**: elicitation (servers asking the user for input); *sampling is deprecated as of protocol version `2026-07-28`*
   - **Utility features**: notifications, progress for long-running operations
2. **Transport layer** — communication mechanisms:
   - **stdio**: standard input/output streams for local, same-machine process communication; optimal performance, no network overhead
   - **Streamable HTTP**: HTTP POST (client→server) with optional Server-Sent Events streaming; supports remote servers and standard HTTP auth (bearer tokens, API keys, custom headers); **OAuth recommended**
   - The same JSON-RPC message format works across all transports.

### Server primitives

| Primitive | What it is | Example |
|-----------|-----------|---------|
| **Tools** | Executable functions the AI application can invoke (`tools/list`, `tools/call`) | file operations, API calls, database queries |
| **Resources** | Data sources providing contextual information | file contents, database records, API responses |
| **Prompts** | Reusable interaction templates | system prompts, few-shot examples |

Each primitive type has methods for discovery (`*/list`), retrieval (`*/get`), and where applicable execution (`tools/call`).

## 6. How we should start using it

1. **Start small with a local server**: build a minimal MCP server exposing 1–2 tools over **stdio** (e.g. internal doc search, or a read-only internal API), and validate it with the **MCP Inspector** before wiring it into a client. *(Tracked in sub-issue #2.)*
2. **Pick an SDK from the official catalog** — Python and TypeScript are the most common starting points and have the most examples; C#/Go/Kotlin are co-maintained with Microsoft/Google/JetBrains respectively if those match our stack. *(Sub-issue #3.)*
3. **Inventory adoption in our own toolchain** — if our editors, chat clients, and cloud platform already support MCP as clients, the marginal cost of trying it is low. *(Sub-issue #4.)*
4. **For remote/hosted servers**, use Streamable HTTP with OAuth rather than the deprecated HTTP+SSE transport.
5. **Governance risk is low**: MCP is now under the Linux Foundation's Agentic AI Foundation with multi-vendor backing (Anthropic, OpenAI, Google, Microsoft, AWS, Block…), so building on it isn't a bet on a single vendor.

## 7. Sources

| # | Source | URL | Used for |
|---|---------|-----|---------|
| 1 | Anthropic — "Introducing the Model Context Protocol" (Nov 25, 2024) | https://www.anthropic.com/news/model-context-protocol | Primary announcement; creators; problem statement; launch components; early adopters |
| 2 | MCP official site — "What is the Model Context Protocol?" | https://modelcontextprotocol.io | USB-C analogy; stakeholder benefits; ecosystem support |
| 3 | MCP official docs — Architecture overview | https://modelcontextprotocol.io/docs/concepts/architecture | Hosts/clients/servers; data & transport layers; primitives; transports |
| 4 | `modelcontextprotocol` GitHub org | https://github.com/modelcontextprotocol | SDK list; co-maintainers; project structure |
| 5 | MCP blog — "MCP joins the Agentic AI Foundation" (Dec 9, 2025) | https://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation | AAIF/Linux Foundation governance; adoption stats; first-class clients |
| 6 | Anthropic — "Donating the Model Context Protocol and establishing the Agentic AI Foundation" (Dec 9, 2025) | https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation | AAIF co-founders and founding projects |
| 7 | TechCrunch — "OpenAI adopts rival Anthropic's standard for connecting AI models to data" (Mar 26, 2025) | https://techcrunch.com/2025/03/26/openai-adopts-rival-anthropics-standard-for-connecting-ai-models-to-data | OpenAI adoption date and scope |
| 8 | The New Stack — "Google Embraces MCP" (May 20, 2025) | https://thenewstack.io/google-embraces-mcp | Gemini API/SDK MCP support at Google I/O |
| 9 | AWS ML Blog — "Harness the power of MCP servers with Amazon Bedrock agents" (Apr 1, 2025) | https://aws.amazon.com/blogs/machine-learning/harness-the-power-of-mcp-servers-with-amazon-bedrock-agents | AWS adoption |
| 10 | Microsoft Developer Blog — "Microsoft partners with Anthropic to create official C# SDK" (Apr 2, 2025) | https://developer.microsoft.com/blog/microsoft-partners-with-anthropic-to-create-official-c-sdk-for-model-context-protocol | Microsoft C# SDK collaboration |
| 11 | InfoQ — "Microsoft moves Azure DevOps MCP server from preview" (Nov 7, 2025) | https://www.infoq.com/news/2025/11/microsoft-ado-mcp-server | Azure DevOps MCP Server GA |
| 12 | VentureBeat — "Anthropic releases Model Context Protocol to standardize AI data integration" (Nov 25, 2024) | https://venturebeat.com/infrastructure/anthropic-releases-model-context-protocol-to-standardize-ai-data-integration | Corroborates the Nov 25, 2024 announcement date |
| 13 | Microsoft Visual Studio DevBlog — "Azure MCP tools now ship built into Visual Studio 2022" (Apr 15, 2026) | https://devblogs.microsoft.com/visualstudio/azure-mcp-tools-now-ship-built-into-visual-studio-2022-no-extension-required | VS 2022 built-in MCP tools |

## 8. Open questions / to verify later

- Exact current spec revision we should target for new builds (check https://modelcontextprotocol.io at implementation time — see sub-issue #3)
- Whether any internal system we want to expose first has an existing community MCP server we could reuse instead of writing our own (sub-issue #4)
- Security review of remote/Streamable HTTP + OAuth before any production use
