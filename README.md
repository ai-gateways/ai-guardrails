# AI Guardrails

AI guardrails are automated checks that inspect what goes into and comes out of an AI model, then block, redact, or flag content that breaks a policy. They catch personal data, leaked secrets, prompt injection, unsafe content, and malformed output before it reaches a model provider, a user, or a downstream system. Teams add them once AI features handle customer data, run in regulated environments, or let models call tools that change real systems.

This guide explains the types of AI guardrails, the actions they take, how they behave on streaming responses, where to enforce them, and which open-source projects implement them.

## Contents

- [What are AI guardrails?](#what-are-ai-guardrails)
- [Input and output guardrails](#input-and-output-guardrails)
- [Types of AI guardrails](#types-of-ai-guardrails)
- [Guardrail actions](#guardrail-actions)
- [Guardrails for streaming responses](#guardrails-for-streaming-responses)
- [Where to enforce guardrails](#where-to-enforce-guardrails)
- [Guardrails for tools and agents](#guardrails-for-tools-and-agents)
- [How to evaluate AI guardrails](#how-to-evaluate-ai-guardrails)
- [Open-source AI guardrails projects](#open-source-ai-guardrails-projects)
- [Related guides](#related-guides)

## What are AI guardrails?

AI guardrails are policy checks on the request path of an AI application. Each check inspects a prompt, a response, a tool argument, or a tool result, decides whether it violates a rule, and returns an action. Guardrails do not change how a model reasons; they control what the model receives and what the application releases.

A guardrail can be a regular expression, a trained classifier, an external moderation service, or a second model that judges content against a written policy. Pattern checks are fast but match only known patterns; model-based checks understand context but add latency, cost, and their own error rate.

The OWASP Top 10 for LLM Applications lists [prompt injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/#:~:text=A%20Prompt%20Injection%20Vulnerability%20occurs%20when%20user%20prompts%20alter), [sensitive information disclosure](https://genai.owasp.org/llmrisk/llm022025-sensitive-information-disclosure/#:~:text=Sensitive%20information%20can%20affect%20both%20the%20LLM%20and%20its%20application%20context), [system prompt leakage](https://genai.owasp.org/llmrisk/llm072025-system-prompt-leakage/#:~:text=The%20system%20prompt%20leakage%20vulnerability%20in%20LLMs%20refers%20to%20the%20risk), and [improper output handling](https://genai.owasp.org/llmrisk/llm052025-improper-output-handling/#:~:text=Improper%20Output%20Handling%20refers%20specifically%20to%20insufficient%20validation) among its risks, and each one maps to a guardrail on the input or the output side.

## Input and output guardrails

Input guardrails run before a request reaches the model, and output guardrails run after the model responds. Input checks stop sensitive data and attacks from reaching a provider. Output checks stop unsafe, leaked, or malformed content from reaching a user or a downstream system. Most policies need both, because a risk can enter on either side.

```text
prompt
  -> input guardrails ...... personal data, secrets, prompt injection, topic rules
  -> model provider
  -> output guardrails ..... personal data, unsafe content, format, grounding
response
```

Input guardrails protect the provider boundary: a prompt containing a customer's card number or an engineer's API key should be redacted or blocked before it leaves the company's network. They also catch direct prompt injection, where a user writes instructions meant to override the system prompt.

Output guardrails protect the user and the application. A model can repeat personal data from its context, reveal its system prompt, produce harmful content, or break the JSON schema the application expects. Model output is untrusted input to whatever consumes it next, so output checks matter even when the input was clean.

[Indirect prompt injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/#:~:text=Indirect%20prompt%20injections%20occur%20when%20an%20LLM%20accepts%20input%20from%20external%20sources) arrives through retrieved documents or tool results rather than the user's message, so it needs a check wherever external content enters the context.

## Types of AI guardrails

AI guardrails fall into six common types: personal data and secrets detection, prompt injection and jailbreak detection, content safety, topic restrictions, schema and format validation, and grounding checks. Each type targets a different failure and uses different techniques, so most deployments combine several types under one policy.

| Guardrail type | What it catches | Common techniques | Typical side |
|---|---|---|---|
| Personal data and secrets | Names, card numbers, health data, API keys, tokens, private keys | Regex and checksum rules, named entity recognition, secret scanners | Input and output |
| Prompt injection and jailbreaks | Instructions that try to override the system prompt or policy | Classifier models, heuristics, LLM judges | Input, plus tool results |
| Content safety | Hate, violence, self-harm, sexual content, harmful instructions | Moderation classifiers, moderation APIs | Input and output |
| Topic restrictions | Requests outside the application's purpose, competitor or legal topics | Allow and deny lists, classifiers, LLM judges | Input |
| Schema and format validation | Responses that break the expected JSON, types, or length | Schema validators, typed parsers, retries | Output |
| Grounding checks | Claims not supported by the retrieved sources | Source comparison, LLM judges | Output |

Grounding checks address [misinformation](https://genai.owasp.org/llmrisk/llm092025-misinformation/#:~:text=One%20of%20the%20major%20causes%20of%20misinformation%20is%20hallucination), where a model produces fluent content its sources do not support. They usually need a second model call, so teams apply them to high-stakes routes only.

## Guardrail actions

A guardrail returns one of four actions when a check matches: block the content, redact the matched span, flag it for review, or log it without changing anything. The right action depends on the risk. Leaked credentials justify a block, personal data often needs redaction, and new or uncertain rules start in log-only mode.

- **Block.** Stop the request or response and return an error or a fixed message. This is the safe default for secrets, attacks, and clearly harmful content.
- **Redact.** Replace the matched span with a placeholder and let the rest through, so one phone number does not break an otherwise useful request.
- **Flag.** Allow the content but mark the request for human review.
- **Log.** Record the finding with no enforcement, to measure false positives before a rule blocks real users.

Redaction has two targets: live traffic and stored records. A team may allow raw text to reach an approved provider but require that request logs and exported traces keep only a redacted copy, which needs runtime redaction and log redaction as separate settings.

Each check also needs a failure policy. When a guardrail service times out, the check either fails open, letting content through, or fails closed, blocking it. Failing closed suits secrets; failing open protects availability for low-risk checks.

## Guardrails for streaming responses

Streaming responses complicate output guardrails because the user sees tokens before the full response exists. A guardrail can observe the stream without delaying it, check short buffered segments and release them, or hold the whole stream until the complete response passes. Each choice trades time to first token against how much unchecked content can reach the user.

```text
detect only ..... stream flows to the client; findings are logged afterward
redact .......... buffer a short segment, redact it, release it
block ........... hold the stream until the full response is checked
```

- **Observe.** Tokens reach the client immediately and nothing is enforced, which suits monitoring.
- **Segment and release.** Suits pattern-based redaction, where a match does not depend on the rest of the response.
- **Hold and release.** The only way to block on whole-response checks, at the cost of the latency benefit of streaming.

Latency is the main cost of guardrails on any path. Regex and secret scanners run in process, classifiers and moderation APIs add a network call, and an LLM judge adds a full model call. Running cheap local checks first and sampling high-volume routes keeps the added latency predictable.

## Where to enforce guardrails

Guardrails can be enforced inside each application through a library, or centrally in a gateway that every model and tool call passes through. A library has full application context, such as the user's role or the retrieved documents. A gateway applies one policy to every application, team, and agent, including ones the security team did not build.

```text
library:   app A -> guardrails A -> model provider
           app B -> guardrails B -> model provider

gateway:   app A  --+
           app B  --+--> gateway (one guardrail policy) --> model providers, MCP servers
           agent  --+
```

| Concern | Library in each application | Central gateway |
|---|---|---|
| Policy consistency | One configuration per application | One policy for all traffic |
| Coverage | Only applications that adopt the library | Every request routed through the gateway |
| Application context | Full access to user, session, and retrieved data | Request content plus caller identity, such as key, team, or user |
| Changes | Redeploy each application | Update the policy once |
| Audit | Scattered across application logs | One record of every decision |

The two approaches combine well: a gateway enforces the organization-wide baseline for secrets, personal data, and content safety, and a library adds checks that need application context, such as whether a retrieved document belongs to the requesting user. The [request pipeline of an LLM gateway](https://github.com/ai-gateways/llm-gateway#how-an-llm-gateway-works) shows where input and output guardrails sit relative to authentication, budgets, and routing.

## Guardrails for tools and agents

Guardrails for agents extend beyond prompts and responses to tool arguments and tool results. An agent that can send email, run queries, or change records needs checks on what it passes to a tool and on what the tool returns, because both can carry sensitive data or injected instructions.

- **Tool arguments.** Inspect and redact arguments before a tool runs, so an agent cannot send personal data to an external service. Unchecked tool calls are the core of the OWASP risk of [excessive agency](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/#:~:text=Excessive%20Agency%20is%20the%20vulnerability%20that%20enables%20damaging%20actions).
- **Tool results.** Check results before they return to the model. The MCP specification asks clients to [validate tool results before passing them to the model](https://modelcontextprotocol.io/specification/2026-07-28/server/tools#security-considerations), and results are a common path for [tool poisoning](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks#:~:text=We%20have%20discovered%20a%20critical%20vulnerability%20in%20the%20Model%20Context%20Protocol) and indirect injection.
- **Human approval.** Require confirmation for destructive or irreversible actions. The A2A specification lists [human approval before a destructive action](https://a2a-protocol.org/latest/specification/#76-in-task-authorization) as a case of in-task authorization between agents.

When model and tool calls pass through one gateway, a single policy covers the prompt, the tool call, the tool result, and the final answer. The [MCP gateway security](https://github.com/ai-gateways/mcp-gateway#mcp-gateway-security) and [agent access control](https://github.com/ai-gateways/agent-gateway#security-and-access-control-for-agents) guides cover the permission checks that sit next to these content checks.

## How to evaluate AI guardrails

Evaluate AI guardrails on coverage, accuracy, latency, and operations. The key questions are which traffic the guardrails see, which risk types they detect, how precisely policies can be scoped, how they behave on streams and failures, and whether content stays inside your infrastructure while it is checked.

- **Coverage of traffic.** Guardrails enforced centrally on requests and responses apply to every application without code changes. Confirm they also cover MCP tool arguments and results.
- **Detection breadth.** Built-in checks for personal data, secrets, and prompt injection, plus connections to external guardrail providers you already license, combined in one policy.
- **Policy scoping.** Rules that target specific API keys, teams, users, or models, so a support bot and a coding agent run different policies.
- **Streaming behavior.** How detect, redact, and block rules handle streamed output, and what happens to time to first token.
- **Latency controls.** Ordered checks, per-rule timeouts, sampling, and a published figure for the base overhead of the enforcement layer.
- **Safe rollout.** A monitor mode that evaluates without enforcing, to measure false positives first.
- **Failure policy.** A stated fail-open or fail-closed behavior per rule.
- **Redaction scope.** Separate settings for live traffic, stored logs, and exported traces.
- **Data boundary.** A self-hosted enforcement point inside your VPC or on-premises, including air-gapped, so content stays inside your infrastructure while it is inspected.

Before enforcing, replay a sample of real prompts, including known attacks and edge cases, and measure both missed detections and false positives.

## Open-source AI guardrails projects

Open-source guardrails projects come in three types: gateways that enforce policies on all traffic, libraries that run inside an application, and classifier models that other tools call. The table lists facts each project publishes in its own repository or documentation.

| Project | Type | Language | License | Checks | Scope | Deployment |
|---|---|---|---|---|---|---|
| [Bifrost](https://github.com/maximhq/bifrost/blob/dev/docs/enterprise/guardrails.mdx) | Gateway | Go | Apache 2.0 core; guardrails in Enterprise tier | Built-in secrets detection, regex rules with a PII template, and LLM-judge policies; connects to Presidio, Azure AI Language PII, AWS Bedrock Guardrails, Azure Content Safety, Google Model Armor, CrowdStrike AIDR, and other providers | LLM prompts and responses, MCP tool arguments and results; rules per virtual key, team, or user | Self-hosted single binary; in-VPC, on-premises, or air-gapped |
| [NeMo Guardrails](https://github.com/NVIDIA-NeMo/Guardrails#types-of-guardrails) | Library and server | Python | Apache 2.0 | Input, dialog, retrieval, execution, and output rails; jailbreak detection, fact-checking, hallucination checks, content and topic safety | Conversational applications, including tool calls through execution rails | pip package; server with a `/v1/chat/completions` endpoint |
| [Guardrails AI](https://github.com/guardrails-ai/guardrails#what-is-guardrails) | Library and server | Python | Apache 2.0 | Input and output guards built from validators; structured output with Pydantic | LLM calls in the application | pip package; Flask server, Docker for production |
| [Presidio](https://github.com/data-privacy-stack/presidio#main-features) | Library and service | Python | MIT | Personal data detection and anonymization in text, images, and structured data | Any data pipeline, not specific to LLMs | Python, PySpark, Docker, or Kubernetes |
| [Llama Guard and Prompt Guard](https://github.com/meta-llama/PurpleLlama#system-level-safeguards) | Classifier models | Python | Llama Community License for models; MIT for Code Shield and evals | Input and output content moderation (Llama Guard), malicious prompt detection (Prompt Guard), insecure code filtering (Code Shield) | Called by a library or gateway as one check | Model weights on Hugging Face |

The types are not exclusive: a gateway can call a PII service or a classifier model as one of its checks, while a library adds checks that depend on application context.

Details reflect each repository in October 2026; confirm current capabilities in each project's documentation before choosing.

## Related guides

- [How an LLM gateway works](https://github.com/ai-gateways/llm-gateway#how-an-llm-gateway-works): where guardrails sit in the model request pipeline
- [MCP gateway security](https://github.com/ai-gateways/mcp-gateway#mcp-gateway-security): token handling and tool poisoning on the agent-to-tool hop
- [Security and access control for agents](https://github.com/ai-gateways/agent-gateway#security-and-access-control-for-agents): permission checks for agent-to-agent traffic
- [The three kinds of AI traffic](https://github.com/ai-gateways/ai-gateway#the-three-kinds-of-ai-traffic): how LLM, MCP, and agent gateways fit together
- [Core AI governance controls](https://github.com/ai-gateways/ai-governance#core-ai-governance-controls): keys, budgets, access control, and audit logs
- [Privacy and data retention](https://github.com/ai-gateways/llm-observability#privacy-and-data-retention): redacting prompts and responses in logs and traces
- [AI Gateways](https://github.com/ai-gateways): all guides in this organization
