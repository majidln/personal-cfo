# Personal CFO: Build Plan

The step-by-step plan for building Personal CFO. For the overview, see [README.md](README.md).

## How to use this plan

- Work is split into **milestones**, and each milestone into **small steps**.
- A step should fit in **one sitting (about 1–3 hours)** and **one commit or PR**.
- Every step has:
  - **Do:** what to build.
  - **Done when:** how you know it works.
  - **Learn:** the concept the step teaches.
- Every milestone ends with a **demo** you can show someone.
- Tick the box when a step is done. If a step grows past one sitting, split it here before starting.
- When a step involves a real design choice, write a short decision record in `docs/decisions/` (see [Decision log](#decision-log)).

### Definition of done (every step)

- Code is typed (no `any` without a comment saying why) and lint passes.
- Tests or evals exist for any new logic.
- No secrets in code or git.
- Any new AWS resources are defined in CDK, not created by hand.
- README or this plan is updated if something changed.

---

## Milestone 1: Foundations

**Goal:** a local LangGraph agent on Bedrock that calls one tool against the fake mailbox.
**Demo:** `pnpm cfo "what bills do I have this week?"` in a terminal gives a sensible answer.

- [ ] **1.1 Monorepo skeleton**
  - Do: pnpm workspaces with `apps/` (ui later), `packages/` (agents, tools, shared), `infra/` (CDK). TypeScript, ESLint, Prettier, Vitest.
  - Done when: `pnpm build`, `pnpm lint` and `pnpm test` all pass on an empty project.
  - Learn: monorepo layout for agent projects.
- [ ] **1.2 AWS account safety**
  - Do: a dedicated sandbox AWS account (or at least a dedicated profile), a budget alarm, and the region choice written down.
  - Done when: the budget alarm email arrives as a test notification.
  - Learn: cost control before anything else.
- [ ] **1.3 CDK bootstrap**
  - Do: an empty `infra/` CDK app with one stack, bootstrapped and deployed.
  - Done when: `cdk deploy` succeeds and `cdk destroy` cleans up.
  - Learn: CDK basics, stacks, bootstrapping.
- [ ] **1.4 First Bedrock call**
  - Do: call a model with `ChatBedrockConverse` from `@langchain/aws` in a small script.
  - Done when: the script prints a model response; model ID and region come from config.
  - Learn: Bedrock model access, Converse API, inference profiles.
- [ ] **1.5 Shared schemas package**
  - Do: `packages/shared` with zod schemas for `Message`, `MailQuery` and `Signal`.
  - Done when: the schemas are exported and unit-tested with valid and invalid examples.
  - Learn: schemas as the contract between agents, tools and UI.
- [ ] **1.6 Fake mailbox fixtures (first 10)**
  - Do: 10 seeded emails as JSON: bills, a broker statement, a newsletter mentioning a ticker, one scam.
  - Done when: every fixture validates against the `Message` schema.
  - Learn: designing test data with known right answers.
- [ ] **1.7 First LangGraph agent**
  - Do: a minimal graph (model node plus tool node) with one tool, `mail_search`, reading the fixtures directly.
  - Done when: the agent answers "what bills do I have?" using the tool.
  - Learn: LangGraph state, nodes, edges, the tool-calling loop.
- [ ] **1.8 CLI entry point**
  - Do: a `pnpm cfo "<question>"` command that streams the answer.
  - Done when: the demo command works.
  - Learn: streaming graph output.
- [ ] **1.9 Checkpointer**
  - Do: add an in-memory checkpointer and a `--thread` flag so follow-up questions keep context.
  - Done when: a follow-up like "and which is the biggest?" works.
  - Learn: short-term memory via checkpointers.

---

## Milestone 2: Inbox agent

**Goal:** a provider-agnostic mailbox and an agent that triages it, measured by evals.
**Demo:** run triage on the full fake mailbox and print an eval score report.

- [ ] **2.1 `MailProvider` interface**
  - Do: the interface (`search`, `getMessage`, `getThread`, `capabilities`) and the normalized `MailQuery` type.
  - Done when: the interface compiles and is documented.
  - Learn: ports and adapters.
- [ ] **2.2 `FakeMailProvider`**
  - Do: implement the interface over the JSON fixtures, including query filtering.
  - Done when: unit tests cover from, subject, date range and label filters.
  - Learn: the adapter side of the pattern.
- [ ] **2.3 Contract test suite**
  - Do: a reusable test suite any `MailProvider` must pass; run it against `FakeMailProvider`.
  - Done when: the suite is a function `runMailProviderContract(provider)` and passes.
  - Learn: contract tests that prove adapters are swappable.
- [ ] **2.4 Expand fixtures to ~50**
  - Do: add more bills, dividend notices, earnings newsletters, a phishing email and two prompt-injection emails. Store the expected labels and extractions next to each email.
  - Done when: every fixture has an `expected` block.
  - Learn: building a golden dataset.
- [ ] **2.5 Email classification**
  - Do: a structured-output node that labels each email (bill, statement, newsletter, scam, phishing, other).
  - Done when: it returns a valid zod-typed result for every fixture.
  - Learn: structured output, classification prompts.
- [ ] **2.6 Field extraction**
  - Do: per-category extraction (amount, due date, merchant, tickers mentioned).
  - Done when: extracted fields validate against the schemas.
  - Learn: extraction with typed schemas.
- [ ] **2.7 Signals from email**
  - Do: turn ticker mentions and important bills into `Signal` records saved to a local store (DynamoDB later).
  - Done when: running triage produces a list of signals.
  - Learn: agents that produce data, not just text.
- [ ] **2.8 Eval harness**
  - Do: a script that runs triage over all fixtures and scores accuracy per label and per field.
  - Done when: `pnpm eval:inbox` prints a score table.
  - Learn: exact-match evals, precision and recall.
- [ ] **2.9 Injection red-team set**
  - Do: eval cases where emails contain instructions to the AI. Check that the agent never follows them and flags them instead.
  - Done when: the red-team eval runs and you know the current pass rate.
  - Learn: prompt injection through untrusted content.
- [ ] **2.10 Model comparison**
  - Do: run the inbox eval on a cheap model and a strong model, and record cost, latency and accuracy.
  - Done when: a decision record states which model triage uses and why.
  - Learn: choosing models with data.
- [ ] **2.11 (Stretch) `GmailProvider` spike**
  - Do: implement the Gmail adapter against a test Gmail account and run the contract suite.
  - Done when: the contract suite passes for Gmail.
  - Learn: OAuth and the real Gmail API.

---

## Milestone 3: News agent

**Goal:** an agent that tracks news and filings for a watchlist and emits ranked signals.
**Demo:** "what happened to my watchlist this week?" returns ranked, cited stories.

- [ ] **3.1 Watchlist and portfolio model**
  - Do: schemas and a local store for watchlist tickers and paper positions.
  - Done when: the CLI can add and list watchlist tickers.
  - Learn: domain modelling.
- [ ] **3.2 Finnhub tool**
  - Do: tools for company news and quotes, with an API key from config.
  - Done when: the tool returns typed results for a ticker.
  - Learn: wrapping an external API as a tool.
- [ ] **3.3 Alpha Vantage news-sentiment tool**
  - Do: a tool for news with sentiment scores.
  - Done when: the tool returns typed results.
  - Learn: a second provider, comparing data quality.
- [ ] **3.4 SEC EDGAR tool**
  - Do: tools to list recent filings for a ticker and fetch filing text.
  - Done when: the agent can fetch the latest 10-Q for a ticker.
  - Learn: public data APIs, EDGAR's User-Agent rules.
- [ ] **3.5 Rate limiting, retries and caching**
  - Do: a shared HTTP helper with retry and backoff, plus a short-lived cache.
  - Done when: repeated calls hit the cache; rate-limit errors are retried.
  - Learn: reliable tools.
- [ ] **3.6 Dedupe and relevance ranking**
  - Do: merge duplicate stories across sources; an LLM scores relevance to *your* holdings and explains why.
  - Done when: output is a ranked list with a "why it matters" line per story.
  - Learn: ranking with LLMs, keeping context small.
- [ ] **3.7 News signals**
  - Do: emit `Signal` records for earnings surprises, downgrades and new filings.
  - Done when: signals from news and email share one store and one format.
  - Learn: one signal model for several sources.
- [ ] **3.8 Citation check**
  - Do: an eval that every story in the output links to a real source returned by a tool.
  - Done when: `pnpm eval:news` reports citation accuracy.
  - Learn: grounding and citation evals.

---

## Milestone 4: Investment committee

**Goal:** a multi-agent LangGraph debate that produces a cited memo for one ticker.
**Demo:** `pnpm cfo committee NVDA` shows the bull case, bear case, risk review and final memo.

- [ ] **4.1 Investment Policy Statement**
  - Do: write `docs/ips.md` with your rules: max position size, banned asset types, risk level, time horizon.
  - Done when: the document exists and is short enough for the risk manager to apply.
  - Learn: turning preferences into rules an agent can enforce.
- [ ] **4.2 Committee graph skeleton**
  - Do: a subgraph with Analyst → (Bull ∥ Bear) → Risk → Chair nodes, using stub outputs.
  - Done when: the graph runs end to end with stubs, and parallel branches merge correctly.
  - Learn: parallel branches, reducers, subgraphs.
- [ ] **4.3 Analyst node**
  - Do: gathers quote, fundamentals, recent news and filings using Milestone 3 tools; writes a fact sheet into state.
  - Done when: the fact sheet is typed and every fact has a source.
  - Learn: a research agent with a fixed output shape.
- [ ] **4.4 Bull and Bear nodes**
  - Do: role prompts that argue from the fact sheet only; optionally one rebuttal round.
  - Done when: both produce arguments that cite fact-sheet items.
  - Learn: debate and critique patterns.
- [ ] **4.5 Knowledge Base setup**
  - Do: decide the vector store first (decision 002). Then, in CDK, create it, a Bedrock Knowledge Base, and an S3 data source containing the IPS and a few 10-K/10-Q filings.
  - Done when: a test query returns relevant chunks with sources.
  - Learn: Bedrock KB, embeddings, chunking, your chosen vector store.
- [ ] **4.6 Chunking experiment**
  - Do: compare two chunking strategies on 10 test questions about filings.
  - Done when: a decision record picks one strategy, with results.
  - Learn: how chunking affects retrieval quality.
- [ ] **4.7 Risk manager node**
  - Do: retrieves IPS rules from the KB and checks the proposal; can veto.
  - Done when: a proposal that breaks a rule (e.g. position too large) is vetoed with the rule cited.
  - Learn: policy enforcement with retrieval.
- [ ] **4.8 Chair node and memo**
  - Do: produces a typed memo: verdict (pass / watch / paper-buy / paper-sell), confidence, size, reasoning and citations.
  - Done when: the memo validates and renders as Markdown in the CLI.
  - Learn: synthesis from several agents.
- [ ] **4.9 Code Interpreter for numbers**
  - Do: the Analyst uses AgentCore Code Interpreter for calculations (ratios, returns, volatility) instead of the model doing math.
  - Done when: every number in the memo comes from a tool or Code Interpreter output.
  - Learn: agents that write and run code in a sandbox.
- [ ] **4.10 Signal → committee trigger**
  - Do: a supervisor node that sends strong ticker signals to the committee.
  - Done when: a newsletter email about a ticker can trigger a committee review.
  - Learn: supervisor routing between agents.
- [ ] **4.11 Memo quality eval**
  - Do: an LLM-as-judge eval on memos (uses only facts, both sides considered, IPS respected) plus a check that every number has a source.
  - Done when: `pnpm eval:committee` reports scores.
  - Learn: LLM-as-judge, its limits and calibration.

---

## Milestone 5: Paper trading and guardrails

**Goal:** approved memos become paper trades, and guardrails enforce the safety rules.
**Demo:** approve a paper buy from the CLI and see it in Alpaca; injection and advice-seeking prompts get handled correctly.

- [ ] **5.1 Alpaca paper broker tool**
  - Do: tools for account, positions and placing orders on the paper endpoint only. Refuse to start if a live URL is configured.
  - Done when: a test order appears in the Alpaca paper dashboard.
  - Learn: tools with side effects, environment safety checks.
- [ ] **5.2 Human-in-the-loop approval**
  - Do: `interrupt()` before any order; the CLI asks approve/reject, then the graph resumes.
  - Done when: rejecting leaves no order; approving places it.
  - Learn: LangGraph interrupts and resuming runs.
- [ ] **5.3 Guardrail: denied topics and wording**
  - Do: a Bedrock Guardrail that blocks guarantees ("guaranteed returns"), tax and legal advice, and real-money execution requests.
  - Done when: red-team prompts are blocked; normal research still works.
  - Learn: denied topics, content filters.
- [ ] **5.4 Guardrail: PII masking**
  - Do: mask account numbers, card numbers and emails in inputs and outputs.
  - Done when: a fixture email with a card number shows masked output.
  - Learn: sensitive-information filters.
- [ ] **5.5 Guardrail: contextual grounding**
  - Do: enable grounding checks on committee and news answers.
  - Done when: an answer with made-up numbers is flagged or blocked.
  - Learn: grounding checks and their thresholds.
- [ ] **5.6 Disclaimer and tone**
  - Do: memos and investment answers always include a short "research, not advice" note.
  - Done when: an eval checks it is always present.
  - Learn: output policy enforcement.
- [ ] **5.7 Full red-team eval set**
  - Do: combine inbox injection, scam tickers, advice-seeking and guardrail-bypass attempts into one suite.
  - Done when: `pnpm eval:redteam` reports a pass rate; failures are tracked as issues.
  - Learn: adversarial testing of agents.

---

## Milestone 6: AgentCore deploy and Gateway

**Goal:** agents run on AgentCore Runtime; all tools go through AgentCore Gateway over MCP.
**Demo:** the CLI talks to the cloud agent; a trace shows agent → Gateway → Lambda → API.

- [ ] **6.1 Containerize the agent**
  - Do: a Dockerfile exposing `/invocations` and `/ping` on port 8080.
  - Done when: the container answers locally with `curl`.
  - Learn: the AgentCore Runtime contract.
- [ ] **6.2 Deploy to AgentCore Runtime**
  - Do: CDK for ECR plus the Runtime, with an IAM role for Bedrock access.
  - Done when: the CLI invokes the cloud agent.
  - Learn: Runtime deployment, sessions.
- [ ] **6.3 Gateway with a first Lambda target**
  - Do: create the Gateway; move `mail_search` into a Lambda target.
  - Done when: the agent loads `mail_search` from the Gateway's MCP endpoint via `@langchain/mcp-adapters`.
  - Learn: Gateway basics, Lambda targets, MCP consumption.
- [ ] **6.4 Gateway inbound auth**
  - Do: only callers with a valid token (Auth0 or Cognito for now) can reach the Gateway.
  - Done when: a call with no token is rejected.
  - Learn: inbound auth.
- [ ] **6.5 OpenAPI targets**
  - Do: add Finnhub, Alpha Vantage and Frankfurter as OpenAPI targets, with API keys in AgentCore Identity.
  - Done when: news tools work through the Gateway with no Lambda code of your own.
  - Learn: OpenAPI targets, outbound API-key auth.
- [ ] **6.6 More Lambda targets**
  - Do: move the Alpaca broker and signal store to Lambda targets; signals move to DynamoDB.
  - Done when: all M2–M5 features work through the Gateway.
  - Learn: tool Lambdas, IAM per tool.
- [ ] **6.7 Your existing Lambdas as tools**
  - Do: register your chosen existing functions as Gateway targets *(open question: which ones?)*.
  - Done when: the agent can call them.
  - Learn: bringing existing services into an agent.
- [ ] **6.8 MCP server target**
  - Do: package the EDGAR tools as an MCP server on AgentCore Runtime, and add it as a Gateway target.
  - Done when: EDGAR tools load through the Gateway.
  - Learn: MCP server targets, combining servers behind one endpoint.
- [ ] **6.9 Semantic tool search**
  - Do: turn on Gateway tool search; compare loading every tool against search.
  - Done when: a decision record notes token use and accuracy for both.
  - Learn: scaling to many tools.
- [ ] **6.10 Observability**
  - Do: AgentCore Observability with OpenTelemetry traces to CloudWatch, plus LangSmith tracing.
  - Done when: one request is visible end to end in both tools.
  - Learn: tracing agents, comparing the two tools.
- [ ] **6.11 Cost dashboard**
  - Do: track tokens and cost per request and per agent.
  - Done when: a CloudWatch dashboard shows cost per agent.
  - Learn: unit economics of agents.

---

## Milestone 7: Web UI and auth

**Goal:** a Next.js glass-box UI where users chat, watch agents work and approve actions.
**Demo:** log in, ask about a ticker, watch the committee debate live, approve a paper trade.

- [ ] **7.1 Next.js app skeleton**
  - Do: `apps/web` with a layout, deployed to S3 and CloudFront via CDK.
  - Done when: the app loads from the CloudFront URL.
  - Learn: static hosting on AWS.
- [ ] **7.2 Auth0 login**
  - Do: login and logout; roles (Free, Premium, Admin) in token claims.
  - Done when: a logged-in user sees their role in the UI.
  - Learn: OIDC, role claims.
- [ ] **7.3 API Gateway and authorizer**
  - Do: API Gateway with a Lambda authorizer verifying Auth0 tokens, in front of the Runtime.
  - Done when: calls with no token fail; calls with a token reach the agent.
  - Learn: securing the agent's front door.
- [ ] **7.4 Streaming chat**
  - Do: a chat panel that streams agent output.
  - Done when: tokens appear as they arrive.
  - Learn: streaming from Runtime to the browser.
- [ ] **7.5 Glass-box panel**
  - Do: a side panel showing graph steps, tool calls with inputs and outputs, guardrail hits and per-turn cost.
  - Done when: a committee run shows each node as it happens.
  - Learn: streaming graph events, making agents explainable.
- [ ] **7.6 Approval cards**
  - Do: interrupts render as Approve/Reject cards that resume the graph.
  - Done when: a paper trade can be approved from the browser.
  - Learn: human-in-the-loop in a real UI.
- [ ] **7.7 Signals feed and memo view**
  - Do: a page listing signals and committee memos, with the bull and bear arguments side by side.
  - Done when: memos render with clickable citations.
  - Learn: presenting agent output.
- [ ] **7.8 Role-based tool access**
  - Do: Free users get inbox and news only; Premium adds the committee and paper trading; Admin adds the Ops agent. Enforce at the Gateway *and* in each tool.
  - Done when: a Free user cannot trigger the committee, even by asking cleverly.
  - Learn: defence in depth, authorization from claims, not chat text.

---

## Milestone 8: Memory, morning briefing and track record

**Goal:** the CFO remembers you, reports to you unprompted, and grades its own calls.
**Demo:** a morning briefing that references your preferences and last week's memos.

- [ ] **8.1 AgentCore Memory setup**
  - Do: a Memory resource with per-user namespaces; connect it to the agents.
  - Done when: a fact said in one session is recalled in another.
  - Learn: long-term vs short-term memory.
- [ ] **8.2 Memory strategies**
  - Do: enable preference and semantic-fact extraction; save risk profile and "no crypto"-style rules.
  - Done when: the committee respects a preference stated weeks earlier.
  - Learn: memory extraction strategies.
- [ ] **8.3 Remember suggestions and outcomes**
  - Do: save each suggestion and whether the user accepted it.
  - Done when: the CFO can say "last time you passed on this".
  - Learn: episodic memory.
- [ ] **8.4 Morning briefing graph**
  - Do: a graph combining new inbox signals, news and committee memos into one short digest.
  - Done when: the CLI prints a briefing.
  - Learn: combining several agents into one output.
- [ ] **8.5 Schedule it**
  - Do: EventBridge Scheduler triggers the briefing every weekday morning; the result is saved and shown in the UI (email optional).
  - Done when: a briefing appears without anyone asking.
  - Learn: event-driven, proactive agents.
- [ ] **8.6 Track record**
  - Do: a scheduled job compares each memo's verdict with later price moves; a chart in the UI.
  - Done when: the UI shows the committee's hit rate.
  - Learn: evaluating agents on real outcomes.

---

## Milestone 9: MCP server, Ops agent and CI evals

**Goal:** share your tools beyond this app, let an agent manage Lambdas safely, and guard quality in CI.
**Demo:** use Personal CFO tools from Claude Desktop; approve a Lambda config change from the UI.

- [ ] **9.1 Public MCP server**
  - Do: expose selected CFO tools (signals, memos, watchlist) as an authenticated MCP server.
  - Done when: Claude Desktop or Cursor can list and call the tools.
  - Learn: building MCP servers for other clients.
- [ ] **9.2 Smithy target for the Lambda API**
  - Do: add the AWS Lambda API (read operations first) as a Smithy target, with IAM limited to functions tagged `agent-managed=true`.
  - Done when: the agent can list and describe tagged functions only.
  - Learn: Smithy targets, letting agents call AWS APIs.
- [ ] **9.3 Ops agent (read and invoke)**
  - Do: an Ops subgraph that inspects config and recent logs and invokes test events.
  - Done when: "why is function X slow?" gets an answer grounded in its config and logs.
  - Learn: agents for operations work.
- [ ] **9.4 Ops agent (propose and approve changes)**
  - Do: proposes config changes (memory, timeout, env vars) as a diff; applies only after approval.
  - Done when: an approved change is applied; a rejected one is not; an untagged function can't be touched even when asked.
  - Learn: safe write actions, IAM as the final safety net.
- [ ] **9.5 Evals in CI**
  - Do: a GitHub Actions workflow runs unit tests plus a small eval subset on each PR, and the full suites nightly.
  - Done when: a quality regression fails the PR.
  - Learn: continuous evaluation.
- [ ] **9.6 Demo polish**
  - Do: seed a demo user, record a 3–5 minute walkthrough, and finish the README with screenshots and architecture.
  - Done when: someone new can understand the project in 5 minutes.
  - Learn: presenting your work.

---

## Decision log

Write a short record in `docs/decisions/NNN-title.md` covering the context, options, decision and consequences. Planned decisions:

| # | Decision | Status |
|---|---|---|
| 001 | TypeScript + LangChain.js/LangGraph.js over Python/Strands | Decided |
| 002 | Vector store for the Knowledge Base. Options: Aurora PostgreSQL Serverless (pgvector), OpenSearch Serverless, Neptune Analytics (GraphRAG), Pinecone, others. S3 Vectors was already covered in another project | Open (decide before step 4.5) |
| 003 | Infrastructure as code: CDK (TypeScript) over Terraform | Proposed |
| 004 | Auth: Auth0 over Cognito | Proposed |
| 005 | Model choice per agent (from step 2.10) | Open |
| 006 | Chunking strategy (from step 4.6) | Open |
| 007 | Gateway tool search vs loading all tools (from step 6.9) | Open |
| 008 | Paper-only trading enforcement | Proposed |

## Open questions

- Which of your existing Lambda functions should become Gateway tools (step 6.7), and which should the Ops agent be allowed to manage (step 9.2)?
- Which vector store for the Knowledge Base (decision 002)? Compare learning value and cost; OpenSearch Serverless, for example, has a standing minimum cost.
- Should the morning briefing also be emailed, or stay in the UI only?
