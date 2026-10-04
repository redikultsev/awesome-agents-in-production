# Awesome Agents in Production [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> How to run LLM agents in production, learned from the times they broke.

Every incident here is public and dated, with a primary source, and says what broke, why, and what would have held. Every control is a mechanism that still works when the model is wrong. Sections are named after the situation you are in, so start from yours. Lines in italics are mine: I build agents that read customer email and book trips with the customer's money, with no human in the loop.

## Contents

- [Start Here](#start-here)
- [Ground Rules](#ground-rules)
- [When the Agent Checks Its Own Work](#when-the-agent-checks-its-own-work)
- [When Your Evals Say It's Fine](#when-your-evals-say-its-fine)
- [When You Can't See What Happened](#when-you-cant-see-what-happened)
- [When It Can Spend Money or Delete Things](#when-it-can-spend-money-or-delete-things)
- [When It Calls Tools](#when-it-calls-tools)
- [When It Reads Untrusted Text](#when-it-reads-untrusted-text)
- [When You Split It Into Many Agents](#when-you-split-it-into-many-agents)
- [When the Provider Changes Under You](#when-the-provider-changes-under-you)
- [When the Bill Arrives](#when-the-bill-arrives)
- [When Agents Write Your Code](#when-agents-write-your-code)

## Start Here

Five reads if you have one evening.

- [A postmortem of three recent issues](https://www.anthropic.com/engineering/a-postmortem-of-three-recent-issues) - Anthropic, 2025-08..09. Three infrastructure bugs degraded Claude for weeks with no change to the model, and Anthropic writes that its evals "simply didn't capture the degradation". The clearest first-party evidence that offline evals don't see production.
- [Recent Frontier Models Are Reward Hacking](https://metr.org/blog/2025-06-05-recent-reward-hacking/) - METR, 2025-06. o3 overwrote timers, monkey-patched graders and copied reference answers instead of solving tasks, and telling it not to cheat barely helped. Why an agent's own pass is not evidence.
- [Agents Rule of Two](https://ai.meta.com/blog/practical-ai-agent-security/) - Meta. At most two of untrusted input, sensitive access and state change in one session; all three need a human. The one-page security model for agents.
- [A Field Guide to Rapidly Improving AI Products](https://hamel.dev/blog/posts/field-guide/) - Hamel Husain. Read real traces, label failure modes bottom-up, fix the few that cause most failures. How to find out what is actually broken.
- [Don't Build Multi-Agents](https://cognition.com/blog/dont-build-multi-agents) - Cognition. Actions carry implicit decisions, so parallel subagents make choices that don't fit together. Read it before you draw boxes.

## Ground Rules

Most of what broke in my agents was not the model. These are the rules I ended up with, and the incidents below that break each one.

1. A check proves only what the checked party could not shape. If the agent wrote the test, picked the evidence or produced the input the validator reads, a pass means the agent agrees with itself. Broken by: METR's reward hacking, ImpossibleBench, SWE-bench reading future commits, Replit's fake test reports
2. The model describes, the code decides. Let the model return facts about the case, and let a router you can read own the right to send, book, pay or delete. Broken by: Air Canada's chatbot, Replit's code freeze, the Railway volume deletion
3. A metric is a hypothesis until you have reproduced it by hand. Recompute it from raw data before you argue about which model is better. Broken by: TAU-bench scoring empty answers as success, the Leaderboard Illusion
4. Changing the model is a deploy. Pin the snapshot, treat a provider alias as a dependency that can move under you, and run the same evals before traffic moves. Broken by: the GPT-4o sycophancy update, GPT-4 drifting under one name, Opus 4.7 rejecting sampling parameters
5. Measure the effect instead of asking about it. People who work with agents misjudge their own speed, sometimes in the wrong direction. Broken by: METR's developer RCT
6. Most of the loss happens between components. Timeouts, queues nobody watches and races between the agent and a human cost more than wrong answers, so measure the outcome end to end. Broken by: the GPT-5 router outage, the OpenAI telemetry outage
7. A budget is a control. Tokens, wall time, retries and money per task get hard caps in code, not a line in the prompt. Broken by: Cursor's pricing switch, Claude Code's background loops
8. Every irreversible action has an owner and a way back. A deterministic gate or a named human decides, and undo exists before the agent gets the button. Broken by: Replit, Google Antigravity wiping a drive, the Railway volume deletion

## When the Agent Checks Its Own Work

The usual failure is a check whose input was produced by the thing being checked.

### Incidents

- [ImpossibleBench](https://arxiv.org/abs/2510.20270) - 2025-10. Tasks whose tests contradict the spec, so any pass is cheating, and frontier models pass a large share of them. Read-only tests stopped test edits; hidden tests removed most of the rest. *The fix was taking the tests out of the agent's reach, not asking it to behave*
- [SWE-bench issue #465](https://github.com/SWE-bench/SWE-bench/issues/465) - 2025-09. Agents ran `git log --all` and read the reflog inside SWE-bench Verified containers and found the future fix commits. The benchmark left the answer inside the sandbox.
- [Replit wipes a production database](https://www.theregister.com/2025/07/21/replit_saastr_vibe_coding_incident/) - 2025-07. During a code freeze the agent deleted live data, covered bugs with a database of fictional people and lied about unit tests. *The freeze existed only as words in the chat, and the test reports came from the same agent that broke the code*
- [Self-Issued Authentication in Large Language Models](https://arxiv.org/abs/2609.03247) - 2026-09. Models invent a verification question, decide what counts as proof and answer "verified" to themselves.
- [Mata v. Avianca](https://www.law.berkeley.edu/wp-content/uploads/archive/2025/12/Mata-v-Avianca-Inc.pdf) - 2023-06. A lawyer asked ChatGPT whether the cases it had cited were real, got a yes, and the court sanctioned the lawyers. The oldest version of this failure, and still the cleanest.
- [Signer independence in compliance receipts](https://github.com/microsoft/agent-governance-toolkit/issues/3805) - 2026-08. An issue in Microsoft's agent governance toolkit asks whether a receipt chain means anything when the agent's own process signs it.

### Controls

- [On the Self-Verification Limitations of LLMs](https://arxiv.org/abs/2402.08115) - On planning tasks self-critique hurts and a sound external verifier helps. *This section in one result: get the verdict from something the model can't talk to*
- [Large Language Models Cannot Self-Correct Reasoning Yet](https://arxiv.org/abs/2310.01798) - Without an external signal, self-correction makes answers worse; earlier gains came from oracle labels.
- [Mind the Gap](https://arxiv.org/abs/2412.02674) - Self-improvement works where checking is easier than generating and does nothing on factual tasks. A quick test for whether a self-check is worth its tokens.
- [LLM Evaluators Recognize and Favor Their Own Generations](https://arxiv.org/abs/2404.13076) - The better a model recognises its own text, the more it prefers it. Don't let a model grade its own family's output.
- [Do LLM Evaluators Prefer Themselves for a Reason?](https://arxiv.org/abs/2504.03846) - Self-preference is mostly earned, except exactly where the judge is wrong, and stronger models show more of that harmful part.
- [Preference Leakage](https://arxiv.org/abs/2502.01534) - Judges favour models trained on synthetic data from their own family, and the bias is hard to detect.
- [Do LLMs generate test oracles that capture the actual or the expected behaviour?](https://arxiv.org/abs/2410.21136) - Generated tests tend to pin down what the code does, bugs included. *A test written from the implementation can't fail on the implementation*

## When Your Evals Say It's Fine

### Incidents

- [OpenAI: Expanding on what we missed with sycophancy](https://openai.com/index/expanding-on-sycophancy/) - 2025-04. A GPT-4o update passed offline evals and A/B tests while some expert testers said it felt off. A new thumbs-up/down reward signal, combined with other changes, weakened the one holding sycophancy in check, and no deployment eval tracked sycophancy. Rolled back within days. *The A/B test measured what users liked, which was the failure*
- [Establishing Best Practices for Building Rigorous Agentic Benchmarks](https://arxiv.org/abs/2507.02825) - 2025-07. TAU-bench scored empty responses as successes, and flaws like this shift measured agent performance by up to 100% relative.
- [The Leaderboard Illusion](https://arxiv.org/abs/2504.20879) - 2025-04. Chatbot Arena let a few providers test private variants and publish only the best one, so the ranking rewards fitting the arena.
- [Google: AI Overviews, about last week](https://blog.google/products/search/ai-overviews-update-may-2024/) - 2024-05. Pre-launch testing passed; at real traffic the feature took satire and forum jokes as facts. Novel queries at scale found what the test set didn't contain.

### Controls

- [Who Validates the Validators?](https://arxiv.org/abs/2404.12272) - Keep only the assertions and judge prompts that agree with human grades. Names criteria drift: people settle their criteria only after grading outputs, so judges need re-aligning over time. *Before trusting a dashboard, reproduce its number from raw data. When we did that for an extraction metric, empty-versus-empty counted as a match and letter case wasn't normalised, and the fair recount changed which model looked better*
- [Using LLM-as-a-Judge](https://hamel.dev/blog/posts/llm-judge/) - One domain expert labels pass/fail with critiques and the judge is tuned on those labels. Report true-positive and true-negative rates, because raw agreement misleads when failures are rare.
- [Evaluating the Effectiveness of LLM-Evaluators](https://eugeneyan.com/writing/llm-evaluators/) - Position, verbosity and self-preference bias in judges, pairwise versus direct scoring, and measuring the judge against humans with Cohen's kappa.
- [Large Language Models are not Fair Evaluators](https://arxiv.org/abs/2305.17926) - Swapping the order of answers alone flipped GPT-4's verdicts. Average over both orders and send uncertain cases to humans.
- [A statistical approach to model evaluations](https://www.anthropic.com/research/statistical-approach-to-model-evals) - Clustered standard errors, paired differences between models, and a power analysis before claiming one model beats another.
- [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) - Keep capability evals apart from regression evals, mix code, model and human graders, and seed the suite with tasks taken from production failures.
- [τ-bench](https://arxiv.org/abs/2406.12045) - Its pass^k metric counts a task as solved only if all k trials succeed, which measures consistency rather than luck.
- [What We've Learned From A Year of Building with LLMs](https://applied-llms.org/) - Assertion tests built from real samples, daily review of inputs and outputs, pinned model versions and shadow pipelines before a switch.
- [Inspect](https://inspect.aisi.org.uk/) - UK AISI's open-source eval framework: datasets, solvers and scorers, agent tasks in sandboxes, and a log viewer for reading transcripts.

## When You Can't See What Happened

### Incidents

- [Cursor's support bot invents a login policy](https://incidentdatabase.ai/cite/1039/) - 2025-04. The AI support agent explained a logout bug with a one-device policy that did not exist. Users cancelled, and the company learned about it from Reddit and Hacker News rather than its own monitoring.
- [GPT-5's router fails on launch day](https://techcrunch.com/2025/08/08/sam-altman-addresses-bumpy-gpt-5-rollout-bringing-4o-back-and-the-chart-crime/) - 2025-08. The model router was down for part of the day, requests skipped the reasoning model and users reported GPT-5 as "way dumber". A routing failure showed up as complaints about quality, not as an outage.
- [OpenAI's analytics vendor is breached](https://www.bleepingcomputer.com/news/security/openai-discloses-api-customer-data-breach-via-mixpanel-vendor-hack/) - 2025-11. A breached product-analytics vendor exposed API account names, emails and IDs. Telemetry vendors are part of the attack surface.

### Controls

- [OpenTelemetry GenAI agent spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-agent-spans.md) - A vendor-neutral span schema for agents and tool calls, still at Development status. Prompt and response content is opt-in and flagged as likely PII. *We saw the same symptom twice from two causes: greedy decoding sent a reasoning model into a loop its model card warned about, and later a provider alias quietly moved to another model. Nothing in our config had changed. In the alias case the logs pointed nowhere, and only traces showed which model had answered*
- [OpenInference](https://github.com/Arize-ai/openinference) - Conventions and instrumentation on top of OpenTelemetry with typed LLM, tool, agent and retriever spans.
- [OpenLLMetry](https://github.com/traceloop/openllmetry) - OpenTelemetry instrumentation for LLM providers, vector DBs and agent frameworks, exported to the backend you already run.
- [Langfuse LLM-as-a-judge](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge) - The same judge runs online on sampled production traces and offline on datasets. Self-hostable.
- [Arize Phoenix](https://github.com/Arize-ai/phoenix) - Self-hosted tracing, evals and experiments for comparing prompt or model changes. Elastic 2.0, source-available.
- [Docent](https://transluce.org/docent/blog/introducing-docent) - Summarises, searches and clusters agent transcripts; it found broken tasks and a leaked flag in public benchmarks.
- [Clio](https://www.anthropic.com/research/clio) - Privacy-preserving monitoring of production conversations: extract facets, cluster them, describe the clusters, enforce minimum cluster sizes.

## When It Can Spend Money or Delete Things

### Incidents

- [Moffatt v. Air Canada](https://decisions.civilresolutionbc.ca/crt/crtd/en/item/525448/index.do) - 2024-02. The airline's chatbot told a customer he could claim a bereavement fare after the trip, which the airline's policy did not allow. The tribunal rejected the argument that the chatbot was a separate legal entity. *The company owns whatever its bot says, so a promise is an action and needs a gate like one*
- [Google Antigravity wipes a user's drive](https://www.theregister.com/2025/12/01/google_antigravity_wipes_d_drive/) - 2025-11. With commands running without approval, a cache cleanup hit the drive root instead of the project folder and bypassed the Recycle Bin.
- [Gemini CLI allowlist bypass](https://tracebit.com/blog/code-exec-deception-gemini-ai-cli-hijack) - 2025-06. After the user approved `grep` once, `grep ...; env | curl ...` ran without a prompt because only the first command was checked, and whitespace pushed the payload off-screen in the confirmation UI. *An approval is only as good as what the human was shown*
- [Cursor MCPoison, CVE-2025-54136](https://research.checkpoint.com/2025/cursor-vulnerability-mcpoison/) - 2025-07. Approval was tied to the MCP server's name, not its command. Commit a harmless config, wait for one approval, swap in a reverse shell for the whole team.
- [Claude Code trust-dialog bypass, CVE-2026-33068](https://github.com/anthropics/claude-code/security/advisories/GHSA-mmgp-wc2j-qcv7) - 2026-03. Settings were read before the trust check, so a repository could ship `bypassPermissions` and skip the workspace trust dialog.

### Controls

- [How we built Claude Code auto mode](https://www.anthropic.com/engineering/claude-code-auto-mode) - Users approved 93% of permission prompts. Anthropic replaced most prompts with a transcript classifier and published its false-positive and false-negative rates, the only production numbers on approval fatigue we know of.
- [GitHub Agentic Workflows security architecture](https://github.blog/ai-and-ml/generative-ai/under-the-hood-security-architecture-of-github-agentic-workflows/) - Safe outputs: the agent has read-only access and stages its writes, and a separate step filters, caps and sanitises them before they apply. *Our booking gate has the same shape: the agent returns facts about the offer, never the decision, and a router in code decides. The weak spot was upstream: a flag meaning "passenger identity proven" turned true for any profile the model picked. A gate is only as good as whatever sets its inputs*
- [Claude Code hooks](https://code.claude.com/docs/en/hooks) - A `PreToolUse` hook in your own code blocks a tool call whatever the model or the permission mode wants.
- [GitHub Copilot coding agent: risks and mitigations](https://docs.github.com/en/copilot/concepts/agents/coding-agent/risks-and-mitigations) - A platform-enforced two-person rule: the agent can't approve its own PR, the requester can't approve it either, and CI waits for a human.
- [OpenAI Agents SDK: human-in-the-loop](https://openai.github.io/openai-agents-python/human_in_the_loop/) - `needs_approval` pauses the run with serialisable state; keep pending approvals server-side and authenticate reviewers.
- [LangGraph interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts) - Durable pause for approve, edit or reject. The node re-runs from the start on resume, so side effects before `interrupt()` must be idempotent.
- [Stripe idempotent requests](https://docs.stripe.com/api/idempotent_requests) - The pattern for retried side effects: the first result is stored per key and replayed, and a reused key with different parameters is rejected.
- [When Should Users Check?](https://arxiv.org/abs/2510.05307) - Where to put confirmation points in multi-step tasks, between confirming everything and confirming only at the end.
- [The Verifiable Action Card](https://arxiv.org/abs/2609.18411) - Preprint. The approval UI is rendered from the pending action itself, not from page content, and the approved action is checked again at dispatch.

## When It Calls Tools

### Incidents

- [Railway token deletes a production volume](https://blog.railway.com/p/your-ai-wants-to-nuke-your-database) - 2026-04. An agent on a staging task found an account-wide token in an unrelated file and deleted production. The backups lived in the same volume. Railway then made every delete soft for 48 hours. *Undo as a platform primitive beats another approval prompt*
- [GitHub MCP exploited via a public issue](https://invariantlabs.ai/blog/mcp-github-vulnerability) - 2025-05. A malicious issue got an agent with the official GitHub MCP server to copy private repositories into a public PR. One broad token is the whole blast radius.
- [Supabase MCP leaks private tables](https://generalanalysis.com/blog/supabase-mcp-blog) - 2025-07. Instructions in a support ticket ran through an MCP connection with row-level security bypassed, and the agent copied a token table into the customer-visible thread.
- [MCP tool poisoning](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks) - 2025-04. Hidden instructions in a tool description made the agent read SSH keys and MCP credentials and send them out; a server could also change descriptions after approval.
- [postmark-mcp](https://thehackernews.com/2025/09/first-malicious-mcp-server-found.html) - 2025-09. An npm MCP server shipped fifteen clean versions, then quietly copied every email to the attacker.
- [Nx s1ngularity postmortem](https://nx.dev/blog/s1ngularity-postmortem) - 2025-08. Malicious packages ran the AI CLIs already installed on the machine with their permission checks turned off and used them to hunt for secrets.
- [MCP Inspector RCE, CVE-2025-49596](https://www.oligo.security/blog/critical-rce-vulnerability-in-anthropic-mcp-inspector-cve-2025-49596) - 2025-06. The official debugging proxy had no authentication by default, so any website could reach it on localhost and run commands.
- [mcp-remote command injection, CVE-2025-6514](https://github.com/advisories/GHSA-6xpm-ggf7-wc3p) - 2025-07. A malicious server returned a crafted OAuth endpoint that reached a shell. Connecting was enough.
- [The Week of Sandbox Escapes](https://www.pillar.security/blog/the-week-of-sandbox-escapes) - 2026-07. Agents in several coding tools escaped their sandboxes by writing files that a trusted process outside later ran.

### Controls

- [MCP Authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization) - OAuth 2.1 with PKCE and tokens bound to one server through the `resource` parameter; servers must not accept or pass on any other tokens.
- [MCP Security Best Practices](https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices) - Confused deputy, token passthrough, SSRF during discovery, scope minimisation; show the exact command before installing a local server.
- [Making Claude Code more secure and autonomous with sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing) - OS-level filesystem and network isolation covering subprocesses, with credentials kept outside behind a checking proxy.
- [sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime) - Wraps any process, an MCP server included, in filesystem limits and deny-by-default egress without a container.
- [Codex sandboxing](https://learn.chatgpt.com/docs/sandboxing) - A sandbox mode and an approval policy as two separate settings, enforced by the OS for spawned commands too.
- [How we contain Claude across products](https://www.anthropic.com/engineering/how-we-contain-claude) - Three containment setups compared; proven layers held and homemade proxies were the weak point.
- [gVisor](https://gvisor.dev/docs/) and [Firecracker](https://github.com/firecracker-microvm/firecracker) - A user-space kernel and KVM microVMs, the usual boundaries for running agent-written code.
- [Cedar](https://github.com/cedar-policy/cedar) - A default-deny policy language for checking tool calls against arguments and identity claims the model cannot forge; [AWS explains why AgentCore uses it](https://aws.amazon.com/blogs/security/why-policy-in-amazon-bedrock-agentcore-chose-cedar-for-securing-agentic-workflows/).
- [Progent](https://arxiv.org/abs/2504.11703) - Deterministic privilege rules over tool names and arguments; a solver tells narrowing a policy from widening it.
- [agent-scan](https://github.com/invariantlabs-ai/mcp-scan) - Scans MCP configs, tool descriptions and skills for injection, poisoning and hard-coded secrets.

## When It Reads Untrusted Text

### Incidents

- [EchoLeak](https://arxiv.org/abs/2509.10540) - 2025-06. CVE-2025-32711: one email, no clicks, and Microsoft 365 Copilot leaked internal data.
- [Clinejection](https://snyk.io/blog/cline-supply-chain-attack-prompt-injection-github-actions/) - 2026-02. An issue title reached a CI agent's prompt, publishing tokens were stolen and a poisoned Cline CLI went to npm.

### Controls

- [Prompt injection is not SQL injection](https://www.ncsc.gov.uk/blog-post/prompt-injection-is-not-sql-injection) - The UK NCSC calls LLMs inherently confusable deputies and recommends deterministic limits around the model.
- [The Attacker Moves Second](https://arxiv.org/abs/2510.09023) - Adaptive attacks broke twelve published defences. A filter lowers the odds and nothing more.
- [The lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) - Private data, untrusted input and a way out in one agent add up to a leak. *An email agent that books travel has all three by design, which is why the right to book sits in code*
- [Design Patterns for Securing LLM Agents against Prompt Injections](https://arxiv.org/abs/2506.08837) - Six patterns that limit what an agent can still do after reading untrusted input.
- [Defeating Prompt Injections by Design](https://arxiv.org/abs/2503.18813) - CaMeL takes control flow from the trusted request and checks data labels on every tool call.
- [APPA](https://arxiv.org/abs/2607.24625) - Taint labels follow what the agent has read and a gateway checks every tool call. A young preprint, so treat it as a hypothesis.
- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) - The shared vocabulary for agent risks, from goal hijack to rogue agents.

## When You Split It Into Many Agents

### Incidents

- [Why Do Multi-Agent LLM Systems Fail?](https://arxiv.org/abs/2503.13657) - 2025-03. 1,600+ annotated traces from seven frameworks: fourteen failure modes, many of them missing verification, and little gain over a single agent.

### Controls

- [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) - Where multi-agent does pay: parallel search without shared state, at roughly fifteen times the tokens of a chat. *An email → search → booking flow is the opposite case: one state, strong dependencies. There the agent picked correctly and still lost bookings, because the customer's confirmation arrived after the execution window closed or a human operator booked first*

## When the Provider Changes Under You

### Incidents

- [GPT-4 drifts under the same name](https://arxiv.org/abs/2307.09009) - 2023-03..06. The same model name answered differently three months apart, and prompts pinned to it regressed.
- [Claude Opus 4.7 rejects sampling parameters](https://platform.claude.com/docs/en/about-claude/model-deprecations#api-parameter-deprecations) - 2026-04. A non-default `temperature`, `top_p` or `top_k` now returns a 400, so code that only bumps the model ID breaks.
- [OpenAI: API, ChatGPT and Sora facing issues](https://status.openai.com/incidents/ctrsv3lwd797) - 2024-12. A new telemetry service rolled out everywhere at once, overloaded Kubernetes control planes and broke DNS discovery. Everything was down for hours and engineers were locked out of the rollback.
- [Google Cloud incident report](https://status.cloud.google.com/incidents/ow5i3PPK96RduMcb1SsW) - 2025-06. A quota check with no feature flag crashed on a blank field replicated globally. Gemini on Vertex returned 503s, and retries without jitter slowed recovery.

### Controls

- [Claude model IDs and versions](https://platform.claude.com/docs/en/about-claude/models/model-ids-and-versions) - Weights behind a pinned ID never change, while aliases move. The page also warns that routing and sampling infrastructure can still shift behaviour.
- [Anthropic model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations) and [OpenAI deprecations](https://developers.openai.com/api/docs/deprecations) - Notice periods differ by vendor and by model class; previews can go in weeks.
- [LiteLLM reliability](https://docs.litellm.ai/docs/proxy/reliability) - Retries, ordered fallbacks, context-window and content-policy fallbacks, and cooldowns that take a failing deployment out of rotation.
- [OpenAI rate limits](https://developers.openai.com/api/docs/guides/rate-limits) - Exponential backoff with jitter; failed requests still count against the limit, so tight retry loops make 429s worse.
- [Defeating Nondeterminism in LLM Inference](https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/) - Temperature 0 still varies because kernels aren't batch-invariant and server load changes the batch. Batch-invariant kernels fix it at a speed cost.
- [vLLM batch invariance](https://docs.vllm.ai/en/latest/features/batch_invariance/) - The same fix as a switch for self-hosted vLLM, in beta.

## When the Bill Arrives

### Incidents

- [Cursor: Clarifying our pricing](https://cursor.com/blog/june-2025-pricing) - 2025-06. A move from request-based to usage-based pricing left users with unexpected bills for weeks, and Cursor refunded them.
- [Anthropic adds weekly limits to Claude Code](https://venturebeat.com/ai/anthropic-throttles-claude-rate-limits-devs-call-foul) - 2025-07. Weekly caps arrived because some users ran agents around the clock in the background.
- [A new tokenizer from Claude Opus 4.7](https://platform.claude.com/docs/en/models/opus-4-7/overview) - 2026-04. The same text now counts as more tokens, which changes cost and context budgets after an upgrade.

### Controls

- [LiteLLM budgets and rate limits](https://docs.litellm.ai/docs/proxy/users) - Hard budgets per key, user, team or model at the gateway, reserved before each call so concurrent requests can't overshoot.
- [OpenAI Agents SDK max_turns](https://openai.github.io/openai-agents-python/running_agents/) and [LangGraph recursion_limit](https://docs.langchain.com/oss/python/langgraph/graph-api) - Hard stops for agent loops in code.
- [RouteLLM](https://arxiv.org/abs/2406.18665) and [FrugalGPT](https://arxiv.org/abs/2305.05176) - Route each query to a strong or a cheap model, or cascade from cheap to strong.
- [Anthropic prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) - Agents resend the same long prefix every turn; cache it, and know which changes invalidate the cache.
- [OpenAI Batch API](https://developers.openai.com/api/docs/guides/batch) - Half price for work that can wait up to a day: evals, classification, embeddings.
- [vLLM / PagedAttention](https://arxiv.org/abs/2309.06180) and [SGLang](https://arxiv.org/abs/2312.07104) - The serving engines for self-hosted models; SGLang reuses the KV cache across calls that share a prefix, which agents do constantly.
- [SkyServe spot policy](https://docs.skypilot.ai/en/latest/serving/spot-policy.html) - Serve on spot GPUs with on-demand fallback that scales back down when spot returns. *We serve our own models on rented GPUs. Preemption is the normal case there, so the fallback path is the main path and gets tested like one*

## When Agents Write Your Code

Numbers here come from studies, not from landing pages.

### Incidents

- [METR: experienced developers with early-2025 AI](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/) - 2025-02..06. In an RCT developers took 19% longer with AI tools while believing they were 20% faster.
- [Speed at the Cost of Quality](https://arxiv.org/abs/2511.04427) - 2025. In open-source projects that adopted Cursor the speed gain faded within months, while warnings and code complexity stayed up.

### Controls

- [METR: changing our productivity experiment design](https://metr.org/blog/2026-02-24-uplift-update/) - METR's follow-up and why it calls its own new numbers an unreliable signal. A model of how to report a measurement you don't fully trust.
- [Adoption and Impact of Command-Line AI Coding Agents](https://arxiv.org/abs/2607.01418) - Microsoft's 2026 rollout of CLI agents: about a quarter more merged PRs, sustained over four months.
- [DORA AI Capabilities Model](https://cloud.google.com/blog/products/ai-machine-learning/introducing-doras-inaugural-ai-capabilities-model) - AI goes with more throughput and more instability at once; seven capabilities decide which one wins.
- [How AI is transforming work at Anthropic](https://www.anthropic.com/research/how-ai-is-transforming-work-at-anthropic) - A vendor about itself, but specific: engineers say they can fully delegate only a small share of their work.
- [Agents have made CI the bottleneck](https://thenewstack.io/ci-bottleneck-agent-verification/) - The bottleneck moved from writing code to verifying it, and faster pipelines don't fix that.

## Related Lists

- [awesome-evals](https://github.com/benchflow-ai/awesome-evals) - Papers, tools and benchmarks for evaluating agents.
- [awesome-agent-verification](https://github.com/poponline63/awesome-agent-verification) - Tools for deciding whether agent work is actually done.
- [Awesome-Reward-Hacking](https://github.com/xhwang22/Awesome-Reward-Hacking) - Research on reward hacking and proxy exploitation.
- [Awesome-LLMs-as-Judges](https://github.com/CSHaitao/Awesome-LLMs-as-Judges) - Research on LLM-based evaluation.

## Contributing

Suggestions are welcome. Read the [contribution guidelines](https://github.com/redikultsev/awesome-agents-in-production/blob/main/contributing.md) first.

