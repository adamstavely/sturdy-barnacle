# Agentic Stack Architecture

AIAS and MACH1 alignment. Draft, 2026-10-16.

Diagram: `agentic-stack-architecture.svg` (vector) and `agentic-stack-architecture.png` (2x raster).

## Summary

One agent loop. Lumbra Nebula is the only component in the organization that calls a model to decide its next action. Tyk is the gateway in three places: ingress in front of Nebula and the platform APIs, the MCP gateway between Nebula and the tools, and the API route in front of the model endpoints. THEMIS attaches at every layer but initiates no runtime path. Astrolabe joins the telemetry from all of it on one trace.

## Layers

| Layer | Components | Colour in diagram |
|---|---|---|
| UXP SuperApps | IntelWorkbench, Mission Planner, GeoDesk, LLM Control Center; KAIROS, ATHENA, SAGA attach here | Neutral; THEMIS coral |
| Ingress | Tyk API Gateway (STS token exchange, rate limits, routing); KAIROS phase gate as an ingress plugin | Tyk teal |
| AIAS agent fabric | Lumbra Nebula (workflows, capabilities, skills, tools; model router, model registry); MCP registry; Astrolabe; MOIRAI, METIS, MIMIR, ALETHEIA, NOMOS, MGS, SKS, LOGOS attach here | Lumbra purple |
| Governed exits | Tyk MCP Gateway with Cerbos PDP sidecar; Tyk API route to models; MOIRAI, NOMOS, LOGOS enforced per call | Tyk teal |
| Capability platforms | TESSERA, canonical world model, LPS, ARBOR, model providers; all exposed to agents as MCP servers | Neutral |
| THEMIS governance plane | AI Governance, Trust Calibration, Content Integrity, Provenance and Custody; telemetry spine (ARGUS, LOGOS, RAI) | THEMIS coral |
| Foundation | ASGARD, BIFROST, Nitro CI/CD, Platforma IDP, Elastic, Neo4j, Snowflake, NATS JetStream | Neutral |

## Runtime path

1. A SuperApp sends analyst intent to the Tyk ingress listener. Tyk exchanges the STS token, applies rate limits, and runs the KAIROS phase gate (commit-before-assist and similar) before routing to Nebula as an upstream.
2. Nebula receives MOIRAI session capability bounds and attaches them to the session.
3. Nebula asks the MCP registry what tools exist. The registry calls Cerbos PlanResources and returns only the tools this analyst could call under these bounds. Nebula plans against that filtered set.
4. The model router inside Nebula selects a model per subtask: hard constraints first (classification ceiling, modality, context window), then MIMIR Elo baselines for the task type, latency budget, and health from Astrolabe. The decision is recorded on the trace.
5. Model calls leave through the Tyk API route to the model endpoints. Tyk holds credentials, enforces the allowlist, and meters. It does not choose the model.
6. Tool calls leave through the Tyk MCP Gateway. Each call carries analyst identity, MOIRAI bounds, and the capability and workflow IDs it belongs to. Tyk authenticates, runs the mandated STS and access decision service coarse gate, then asks the Cerbos sidecar for the fine-grained decision. Effective permission is the analyst's entitlements intersected with the session bounds.
7. The MCP server runs behind ARBOR, which records chain of custody on the result.
8. Nebula evaluates, corroborates, and synthesizes. The insight returns to the analyst through ingress.
9. Nebula spans, Tyk MCP metrics, router decisions, and Cerbos decision logs land in Astrolabe on one trace ID. Astrolabe rolls primitives up to capability and workflow and feeds the THEMIS telemetry spine.
10. The analyst accepts or overrides. RAI updates from that decision, ATHENA selects an intervention, and MIMIR re-baselines. The loop closes at the top.

### Three things the sequence makes visible

- Discovery is a Nebula self-call because the registry filters with Cerbos before Nebula plans. Identity and session bounds must therefore arrive at the registry and at the gateway, and both come from Nebula carrying them.
- Nebula appears on both sides of every Tyk box. Tyk never calls a model and never calls Nebula except to forward a sampling or elicitation request. An arrow from any Tyk role to a model is the second harness appearing.
- Astrolabe receives from every layer and initiates nothing. It depends on the trace ID, capability ID, and workflow ID being propagated end to end, which is the one integration requirement to confirm with Lumbra.

## Deterministic plane and stochastic plane

The cleanest statement of the boundary: the gateway is deterministic and the harness is stochastic. A component belongs to the gateway if, given the same input and the same policy, it produces the same output every time in bounded time. A component belongs to the harness if its output depends on a model. Nothing sits in between, and no component may be both.

### Gateway: deterministic only

Tyk, its plugins, and Cerbos. Same input, same policy, same answer, microseconds to low milliseconds. Every capability below can be unit tested with fixtures and promoted through Nitro like any other code. None of them calls a model.

Ingress (API gateway role)

- TLS termination and protocol handling for SuperApp traffic
- STS token validation and token exchange; propagation of analyst identity downstream
- Routing to Nebula as an upstream and to platform APIs for non-agent calls
- Rate limiting, quotas, and circuit breaking per caller and per route
- KAIROS phase gate as a plugin: is assistance permitted in this phase, yes or no
- Request and response schema validation against published API contracts
- Header normalization, correlation ID and W3C traceparent injection when absent

MCP gateway role

- MCP protocol termination for remote (HTTP) servers
- Authentication of the calling agent on behalf of the analyst
- Filtered discovery: tools/list returns only the tools, resources, and prompts this principal may see, driven by the registry and Cerbos PlanResources
- Per-call authorization: mandated STS and access decision service coarse gate, then the Cerbos sidecar for the fine-grained ABAC and PBAC decision, with MOIRAI session bounds and capability and workflow IDs passed as attributes
- Tool input validation: JSON schema conformance, type and range checks, required fields, size limits
- Tool output validation: schema conformance, classification markings present and well formed, PII and secret pattern blocking, size limits, malformed payload rejection
- Response transformation: field renaming, envelope wrapping, stripping internal keys, unit conversion, redaction by rule
- API-to-MCP tool minting from OpenAPI specs, published into the MCP registry as system of record
- Sampling and elicitation routing: MCP server requests for a completion or for user input are forwarded to Nebula, never serviced at the gateway
- OpenTelemetry metrics and spans per MCP primitive, exported to Astrolabe on the propagated trace ID
- Cerbos decision logs emitted to the telemetry spine as an authorization receipt per call

Model route role

- Publishing enclave model endpoints as plain API routes
- Credential holding for model providers; callers never hold provider keys
- Model allowlist per principal and per classification boundary
- Token and request metering per caller for capacity planning
- Rate limits and timeouts on model traffic
- Request and response logging to Astrolabe with the trace ID

What the gateway does not do

- Call a model for any reason
- Decide which model a subtask should use
- Judge whether a tool result is plausible, relevant, or trustworthy
- Summarize, rewrite, or decide what to omit from a response
- Hold session state, plans, or belief state
- Compose tools into capabilities or sequence capabilities into workflows

A gateway plugin may call a Nebula skill or a THEMIS service synchronously as a check, the same way it calls Cerbos. The judgment runs in the governed place; the gateway enforces the answer.

### Harness: stochastic, and everything that depends on it

Lumbra Nebula and Astrolabe. Output depends on a model, so every one of these needs the controls that only exist inside the harness: session bounds, evaluation gates, provenance, and a trace.

Orchestration

- Task decomposition: breaking analyst intent into subtasks
- Hierarchical multi-agent orchestration and delegation
- Workflow execution: sequencing capabilities, branching, retries, human-in-the-loop pauses
- Capability composition: combining tools and skills into a unit the analyst experiences
- Skill execution: model-backed behaviors such as extraction, summarization, causal analysis, network analysis, synthesis
- Belief state management and decision trees across a session
- Tool selection from the filtered catalog the registry returns

Model interaction

- The model router: per-subtask model selection under hard constraints (classification ceiling, modality, context window) and scored preferences (MIMIR Elo for the task type, latency budget, Astrolabe health)
- Prompt construction and context assembly
- Servicing sampling and elicitation requests forwarded from the gateway
- Recording the chosen model, the eligible candidates, and the reason on the trace

Evaluation and trust

- Evaluation gates on every output, with rubrics read from ALETHEIA ground truth
- Semantic validation: plausibility, relevance, corroboration across sources, contradiction with known state
- METIS self-evaluation
- Consuming MOIRAI session bounds and propagating them on every tool call
- Enforcing KAIROS phase constraints on what may be returned mid-phase

Provenance and state

- Full provenance from intent to insight: reasoning steps, sources, evaluation gates passed
- Stamping each tool call with analyst identity, session bounds, capability ID, and workflow ID
- Session state, plans, intermediate results
- Emitting the records ARBOR needs for chain of custody and SAGA needs for session narrative

Observability (Astrolabe)

- Collecting spans from Nebula, the router, and both Tyk roles on one trace ID
- Rolling up per-primitive gateway metrics to capability and workflow
- Model health and latency signals fed back to the router
- Feeding ARGUS, LOGOS, and RAI on the THEMIS telemetry spine

Catalogs the harness owns

- Capability and skill catalog (the MCP registry holds tools; capabilities and skills are Nebula's)
- Model registry: approved models with classification ceiling, context window, modality, latency class, endpoint, and MIMIR baseline per task type

### The test for a new component

If it calls a model to decide what it does next, it is stochastic and it belongs in the harness, whatever it is named. If it can be written as a schema, a pattern, a policy, or a transform, it is deterministic and it belongs at the gateway. If a team cannot say which, the component is not specified well enough to build.

## Nebula object model and the governance boundary

Nebula has four primitives: workflows, capabilities, skills, and tools. A capability is a combination of tools and skills. A workflow sequences capabilities.

Only tools cross Tyk. A tool is an MCP primitive, so the gateway sees it, authorizes it per call, filters it from discovery, and meters it. Skills, capabilities, and workflows are Nebula constructs and never leave Nebula. When the gateway needs to reason about composition (allow this tool only from that capability), Nebula stamps each outgoing tool call with capability and workflow IDs as attributes and Cerbos policy reads them. The composition logic stays in Nebula; the policy gets richer at the gateway.

## Who owns what

**Harness team (Lumbra).** Nebula, the model router and model registry, the capability and skill catalog, Astrolabe. The only team that owns a component which calls a model to decide its next action.

**Gateway team (Tyk).** Ingress API gateway, MCP gateway, model API route. Protocol termination, authN, STS token exchange, filtered discovery, Cerbos integration, OTel metrics, API-to-MCP tool minting, the KAIROS ingress plugin, and the sampling and elicitation adapter that routes MCP server requests into Nebula. Deterministic validation and response transformation as Tyk plugins. No model loop.

**THEMIS.** Attaches at every layer, initiates nothing. Policy ships through NOMOS in Git, is tested in Nitro with Cerbos test suites, and is promoted per environment. Signals return through the spine.

## Decision rules

1. One loop. A second component that calls a model to choose its next step is a second harness, whatever it is named.
2. SuperApps never reach Nebula, a tool, or a model directly. Everything enters through the Tyk ingress listener. Nebula is an upstream like any other.
3. Only tools cross Tyk. Skills, capabilities, and workflows never leave Nebula.
4. The model router is a trust decision, so it lives inside Nebula beside MIMIR and ALETHEIA, not in a proxy.
5. Model endpoints are published through Tyk as plain API routes. That gives credential holding, allowlisting, and metering without AI Studio. The router chooses; Tyk enforces the route.
6. Agents hold no standing entitlements. Effective permission is the analyst's entitlements intersected with MOIRAI session bounds.
7. Validation and formatting at the gateway are deterministic: schema conformance, type and range checks, classification markings, PII patterns, size limits, field reshaping. Anything that needs judgment (plausibility, relevance, summarization, source trust) is a Nebula skill or a THEMIS content integrity service. A gateway plugin may call one synchronously the way it calls Cerbos; it may not host the judgment itself.

## Why not a second harness at the gateway

### What the gateway team may actually be asking for

When a gateway team says it needs a general harness, it usually means one of four things. Only the first is a vocabulary problem; the rest are real needs that are solved by integration, not by a second runtime.

1. A layering distinction. Separating the agent runtime from the mission-specific agent definitions on top of it. That is one execution engine with many configurations, which is already the design.
2. Sampling and elicitation. MCP servers behind Tyk can ask the client for a completion or for user input. The gateway has to route those somewhere that can call a model. That is an adapter into Nebula, not a loop.
3. Non-agentic MCP consumers. IDE plugins, desktop assistants, analyst notebooks. The gateway must not depend on the harness, which argues for a harness-agnostic gateway, not a second harness.
4. Roadmap independence. Not wanting to be blocked by the harness team's backlog. Solved by a platform contract, not duplicated software.

The gateway team proposed a general agentic harness built around Tyk, with the specific harness (Nebula) connected behind it, and later justified it with gateway-side agents for data validation and response formatting.

- Tyk's own product split draws the line: the MCP gateway handles authentication, access control, rate limiting, and observability; what happens inside the LLM loop belongs to a separate product. A harness around Tyk rebuilds that separate layer, which Nebula already occupies.
- Two loops means every THEMIS control is implemented twice or, in practice, implemented once and bypassed once. MOIRAI bounds, ARBOR custody, and the Astrolabe trace all assume one place where model calls, tool calls, and session state can be observed and constrained.
- A gateway-side harness becomes the path of least resistance for every new MCP consumer because the gateway team controls onboarding. Within a year there would be Nebula for designed agents and the Tyk harness for everything that walked in through the gateway.
- Model-backed agents at the gateway break the governance chain (model output enters the tool result stream without custody), add a second prompt injection surface in front of every tool, and make the policy enforcement point nondeterministic with model-dependent latency and failure modes.
- The legitimate needs underneath the request (per-agent identity at the gateway, sampling and elicitation routing, local server hosting, registry sync, roadmap independence) are each solved by an integration contract, not a runtime.

## Why not Tyk AI Studio

AI Studio is an LLM proxy with a control plane: multi-vendor model configuration, group-based access, per-interaction analytics, a portal and chat UI, a plugin system. It is to models what Tyk Gateway is to tools.

It is out of scope here because the requirement is a model router, not a model proxy. A proxy enforces; a router decides. The routing decision depends on the subtask, MIMIR baselines, and ALETHEIA ground truth, none of which a proxy can see. The proxy functions AI Studio would provide (credentials, allowlist, metering) come from publishing the model endpoints as plain API routes on the Tyk gateway already in place. Revisit if a second model consumer appears that cannot go through Nebula and needs more than a plain API route.

### Router versus proxy

A proxy enforces: it holds credentials, applies the allowlist, meters, and logs. It does not care which model was asked for. A router decides: given this subtask, which model should handle it. In this architecture the routing judgment is a trust question, because MIMIR exists to baseline models by task type and RAI exists to measure whether reliance on a model output was warranted. That makes the router a consumer of THEMIS and places it inside Nebula. The proxy function is served by the Tyk model route. AI Studio's routing is proxy-level (fallbacks, load balancing, request rules) and cannot see a Nebula subtask or a MIMIR baseline.

## THEMIS attach points

| Attach point | Services | Role there |
|---|---|---|
| SuperApps | ATHENA, SAGA, KAIROS | Interventions, session narrative, phase gate decided before Nebula is called |
| Ingress | KAIROS | Phase policy enforced as a Tyk plugin |
| Nebula | MOIRAI, KAIROS, METIS | Session bounds received and attached, phase constraint on mid-phase returns, self-evaluation |
| Model router | MIMIR, ALETHEIA | Elo baselines per task type, ground truth the baselines were scored against |
| MCP registry | NOMOS, MGS, SKS, LOGOS | Publish-time trust, policy-as-code, discovery filter via Cerbos PlanResources |
| Tyk MCP Gateway and Cerbos | MOIRAI, NOMOS, LOGOS | Bounds intersected with entitlements per call, policy bundles, access decision receipt |
| MCP servers | ARBOR | Chain of custody on every result |
| Astrolabe and spine | ARGUS, LOGOS, RAI | Aggregate monitoring, reasoning chain, reliance calibration returned to ATHENA |

MOIRAI appears at Nebula and at Tyk because bounds are received in one place and enforced in the other. LOGOS appears three times because access has three moments: who may publish or see a tool, who may call it now, and what reasoning chain exercised that access. NOMOS is a supply chain, not a runtime service; nothing calls it at request time.

## Open before AERB

- Owner for MOIRAI and LOGOS. The September carve moved them toward ARBOR, but they still act as the THEMIS telemetry spine. The model router makes this more pressing: ARBOR is custody of results, LOGOS is custody of decisions, and a routing choice is a decision.
- ContextForge: retire, or narrow to the sampling and elicitation adapter between Tyk and Nebula.
- Confirm with Lumbra that Nebula propagates W3C traceparent plus capability and workflow IDs through tool calls, and whether model selection is pluggable (a Nebula extension) or static (a routing service in front of Nebula's model connections).
- Registry of record for tools minted by Tyk from OpenAPI specs: Tyk publishes, the MCP registry holds.
- Ingress and MCP listeners: one Tyk cluster with separate policy bundles, or separate deployments. Decide on blast radius and the classification boundary, not convenience.
- Tyk AI Studio is out of scope for this design.

## Questions for the two team leads

For the gateway lead: what does Nebula fail to do today that makes Tyk unable to enforce per-agent policy? And which validation or formatting rule do you need that cannot be written as a schema, a pattern, or a deterministic transform?

For the harness lead: what would the gateway team need from Nebula to stop needing their own runtime? Identity propagation, a sampling and elicitation entry point, and a stable framework-neutral API are the expected answers.

If the gateway team's answer is a capability gap, fund it inside the single harness. If the answer is that they do not want to wait on the harness team, the request was organizational and the remedy is a platform contract: published API, SLA, inner-source contribution rights, and an escalation path.
