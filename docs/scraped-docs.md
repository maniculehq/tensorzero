# TensorZero — Scraped Documentation

**Source:** https://tensorzero.com
**Pages scraped:** 15
**Total characters:** 83872

---

## 

**URL:** https://www.tensorzero.com/docs/quickstart.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Quickstart
> Get up and running with TensorZero in 5 minutes.
This Quickstart guide shows how we'd upgrade an OpenAI wrapper to a minimal TensorZero deployment with built-in observability and fine-tuning capabilities — in just 5 minutes.
From there, you can take advantage of dozens of features to build best-in-class LLM applications.
This Quickstart covers a tour of TensorZero features.
If you're only interested in inference with the gateway, see the shorter [How to call any LLM](/gateway/call-any-llm) guide.
You can also find the runnable code for this example on [GitHub](https://github.com/tensorzero/tensorzero/tree/main/examples/docs/guides/quickstart).
## Status Quo: OpenAI Wrapper
Imagine we're building an LLM application that writes haikus.
Today, our integration with OpenAI might look like this:
```python title="before.py" theme={null}
from openai import OpenAI
client = OpenAI()
response = client.chat.completions.create(
model="gpt-4o-mini",
messages=[
{
"role": "user",
"content": "Write a haiku about TensorZero.",
}
],
)
print(response)
```
```python theme={null}
ChatCompletion(
id='chatcmpl-A5wr5WennQNF6nzF8gDo3SPIVABse',
choices=[
Choice(
finish_reason='stop',
index=0,
logprobs=None,
message=ChatCompletionMessage(
content='Silent minds awaken,  \nPatterns dance in code and wire,  \nDreams of thought unfold.',
role='assistant',
function_call=None,
tool_calls=None,
refusal=None
)
)
],
created=1725981243,
model='gpt-4o-mini',
object='chat.completion',
system_fingerprint='fp_483d39d857',
usage=CompletionUsage(
completion_tokens=19,
prompt_tokens=22,
total_tokens=41
)
)
```
```ts title="before.ts" theme={null}
import OpenAI from "openai";
const client = new OpenAI();
const response = await client.chat.completions.create({
model: "gpt-4o-mini",
messages: [
{
role: "user",
content: "Write a haiku about TensorZero.",
},
],
});
console.log(JSON.stringify(response, null, 2));
```
```bash title="before.sh" theme={null}
curl https://api.openai.com/v1/chat/completions \
-H "Authorization: Bearer $OPENAI_API_KEY" \
-H "Content-Type: application/json" \
-d '{
"model": "gpt-4o-mini",
"messages": [
{
"role": "user",
"content": "Write a haiku about TensorZero."
}
]
}'
```
## Migrating to TensorZero
TensorZero offers dozens of features covering inference, observability, optimization, evaluations, and experimentation.
But the absolutely minimal setup requires just a simple configuration file: `tensorzero.toml`.
```toml title="tensorzero.toml" theme={null}
# A function defines the task we're tackling (e.g. generating a haiku)...
[functions.generate_haiku]
type = "chat"
# ... and a variant is one of many implementations we can use to tackle it (a choice of prompt, model, etc.).
# Since we only have one variant for this function, the gateway will always use it.
[functions.generate_haiku.variants.gpt_4o_mini]
type = "chat_completion"
model = "openai::gpt-4o-mini"
```
This minimal configuration file tells the TensorZero Gateway everything it needs to replicate our original OpenAI call.
Using the shorthand `openai::gpt-4o-mini` notation is convenient for getting started.
To learn about all configuration options including schemas, templates, and advanced variant types, see [Configure functions and variants](/gateway/configure-functions-and-variants).
For production deployments with multiple providers, routing, and fallbacks, see [Configure models and providers](/gateway/configure-models-and-providers).
## Deploying TensorZero
We're almost ready to start making API calls.
Let's launch TensorZero.
1. Set the environment variable `OPENAI_API_KEY`.
2. Place our `tensorzero.toml` in the `./config` directory.
3. Download the following sample `docker-compose.yml` file.
This Docker Compose configuration sets up a development Postgres database, the TensorZero Gateway, and the TensorZero UI.
```bash theme={null}
curl -LO "https://raw.githubusercontent.com/tensorzero/tensorzero/refs/heads/main/examples/docs/guides/quickstart/docker-compose.yml"
```
```yaml title="docker-compose.yml" theme={null}
# This is a simplified example for learning purposes. Do not use this in production.
# For production-ready deployments, see: https://www.tensorzero.com/docs/gateway/deployment
services:
gateway:
image: tensorzero/gateway
volumes:
# Mount our tensorzero.toml file into the container
- ./config:/app/config:ro
command: --config-file /app/config/tensorzero.toml
environment:
OPENAI_API_KEY: ${OPENAI_API_KEY:?Environment variable OPENAI_API_KEY must be set.}
TENSORZERO_POSTGRES_URL: postgres://postgres:postgres@postgres:5432/tensorzero
ports:
- "3000:3000"
extra_hosts:
- "host.docker.internal:host-gateway"
healthcheck:
test:
[
"CMD",
"wget",
"--no-verbose",
"--tries=1",
"--spider",
"http://localhost:3000/health",
]
start_period: 10s
start_interval: 1s
timeout: 1s
depends_on:
postgres:
condition: service_healthy
gateway-run-postgres-migrations:
condition: service_completed_successfully
ui:
image: tensorzero/ui
environment:
TENSORZERO_GATEWAY_URL: http://gateway:3000
ports:
- "4000:4000"
depends_on:
gateway:
condition: service_healthy
postgres:
image: tensorzero/postgres:17
command: ["postgres", "-c", "cron.database_name=tensorzero"]
environment:
POSTGRES_DB: tensorzero
POSTGRES_USER: postgres
POSTGRES_PASSWORD: postgres
ports:
- "5432:5432"
volumes:
- postgres-data:/var/lib/postgresql/data
healthcheck:
test: ["CMD-SHELL", "pg_isready -U postgres -d tensorzero"]
start_period: 30s
start_interval: 1s
timeout: 1s
# Apply Postgres migrations before the gateway starts
gateway-run-postgres-migrations:
image: tensorzero/gateway
environment:
TENSORZERO_POSTGRES_URL: postgres://postgres:postgres@postgres:5432/tensorzero
depends_on:
postgres:
condition: service_healthy
command: ["--run-postgres-migrations"]
volumes:
postgres-data:
```
Our setup should look like:
```
- config/
- tensorzero.toml
- after.* see below
- before.*
- docker-compose.yml
```
Let's launch everything!
```bash theme={null}
docker compose up
```
## Our First TensorZero API Call
The gateway will replicate our original OpenAI call and store the data in our database — with less than 1ms latency overhead thanks to Rust 🦀.
The TensorZero Gateway is compatible with the **OpenAI SDK**, so you can make inference calls using the OpenAI client for Python, Node, and other languages.
Point your OpenAI client to the TensorZero Gateway:
```python title="after.py" {3, 6} theme={null}
from openai import OpenAI
client = OpenAI(base_url="http://localhost:3000/openai/v1", api_key="not-used")
response = client.chat.completions.create(
model="tensorzero::function_name::generate_haiku",
messages=[
{
"role": "user",
"content": "Write a haiku about TensorZero.",
}
],
)
print(response)
```
```python theme={null}
ChatCompletion(
id='0194061e-2211-7a90-9087-1c255d060b59',
choices=[
Choice(
finish_reason='stop',
index=0,
logprobs=None,
message=ChatCompletionMessage(
content='Circuit dreams awake,  \nSilent minds in metal form—  \nWisdom coded deep.',
refusal=None,
role='assistant',
audio=None,
function_call=None,
tool_calls=[]
)
)
],
created=1735269425,
model='gpt_4o_mini',
object='chat.completion',
service_tier=None,
system_fingerprint='',
usage=CompletionUsage(
completion_tokens=18,
prompt_tokens=15,
total_tokens=33,
completion_tokens_details=None,
prompt_tokens_details=None
),
episode_id='0194061e-1fab-7411-9931-576b067cf0c5'
)
```
Point your OpenAI client to the TensorZero Gateway:
```ts title="after.ts" {4-5,9} theme={null}
import OpenAI from "openai";
const client = new OpenAI({
baseURL: "http://localhost:3000/openai/v1",
apiKey: "not-used",
});
const response = await client.chat.completions.create({
model: "tensorzero::function_name::generate_haiku",
messages: [
{
role: "user",
content: "Write a haiku about TensorZero.",
},
],
});
console.log(JSON.stringify(response, null, 2));
```
```json theme={null}
{
"id": "01958633-3f56-7d33-8776-d209f2e4963a",
"episode_id": "01958633-3f56-7d33-8776-d2156dd1c44b",
"choices": [
{
"index": 0,
"finish_reason": "stop",
"message": {
"content": "Wires pulse with knowledge,  \nDreams crafted in circuits hum,  \nMind of code awakes.  ",
"tool_calls": [],
"role": "assistant"
}
}
],
"created": 1741713261,
"model": "gpt_4o_mini",
"system_fingerprint": "",
"object": "chat.completion",
"usage": {
"prompt_tokens": 15,
"completion_tokens": 23,
"total_tokens": 38
}
}
```
Make an HTTP request to the TensorZero Gateway:
```bash title="after.sh" {1,4} theme={null}
curl http://localhost:3000/openai/v1/chat/completions \
-H "Content-Type: application/json" \
-d '{
"model": "tensorzero::function_name::generate_haiku",
"messages": [
{
"role": "user",
"content": "Write a haiku about TensorZero."
}
]
}'
```
```json theme={null}
{
"id": "019cd8d7-1a7d-7d50-ac23-54da47daba7b",
"episode_id": "019cd8d7-1a7d-7d50-ac23-54e783e7833b",
"choices": [
{
"index": 0,
"finish_reason": "stop",
"message": {
"role": "assistant",
"content": "In silent circuits,  \nTensorZero weaves threads,  \nAI's quiet bloom."
}
}
],
"created": 1773164502,
"model": "gpt_4o_mini",
"system_fingerprint": "",
"object": "chat.completion",
"usage": {
"prompt_tokens": 15,
"completion_tokens": 17,
"total_tokens": 32
}
}
```
## TensorZero UI
The TensorZero UI streamlines LLM engineering workflows like observability and optimization (e.g. fine-tuning).
The Docker Compose file we used above also launched the TensorZero UI.
You can visit the UI at `http://localhost:4000`.
### Observability
The TensorZero UI provides a dashboard for observability data.
We can inspect data about individual inferences, entire functions, and more.
This guide is pretty minimal, so the observability data is pretty simple.
Once we start using more advanced functions like feedback and variants, the observability UI will enable us to track metrics, experiments (A/B tests), and more.
### Fine-Tuning
The TensorZero UI also provides a workflow for fine-tuning models like GPT-4o and Llama 3.
With a few clicks, you can launch a fine-tuning job.
Once the job is complete, the TensorZero UI will provide a configuration snippet you can add to your `tensorzero.toml`.
We can also send [metrics & feedback](/gateway/guides/metrics-feedback/) to the TensorZero Gateway.
This data is used to curate better datasets for fine-tuning and other optimization workflows.
Since we haven't done that yet, the TensorZero UI will skip the curation step before fine-tuning.
## Conclusion & Next Steps
The Quickstart guide gives a tiny taste of what TensorZero is capable of.
We strongly encourage you to check out the guides on [metrics & feedback](/gateway/guides/metrics-feedback/) and [prompt templates & schemas](/gateway/create-a-prompt-template).
Though optional, they unlock many of the downstream features TensorZero offers in experimentation and optimization.
From here, you can explore features like built-in support for [inference-time optimizations](/gateway/guides/inference-time-optimizations/), [retries & fallbacks](/gateway/guides/retries-fallbacks/), [experimentation (A/B testing) with prompts and models](/experimentation/run-adaptive-ab-tests), and a lot more.

---

## 

**URL:** https://www.tensorzero.com/docs/comparison/dspy.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Comparison: TensorZero vs. DSPy
> TensorZero is an open-source alternative to DSPy featuring an LLM gateway, observability, optimization, evaluations, and experimentation.
TensorZero and DSPy serve **different but complementary** purposes in the LLM ecosystem.
TensorZero is a full-stack LLM engineering platform focused on production applications and optimization, while DSPy is a framework for programming with language models through modular prompting.
**You can get the best of both worlds by using DSPy and TensorZero together!**
## Similarities
* **LLM Optimization.**
Both TensorZero and DSPy focus on LLM optimization, but in different ways.
DSPy focuses on automated prompt engineering, while TensorZero provides a complete set of tools for optimizing LLM systems (including prompts, models, and inference strategies).
* **LLM Programming Abstractions.**
Both TensorZero and DSPy provide abstractions for working with LLMs in a structured way, moving beyond raw prompting to more maintainable approaches.
[→ Prompt Templates & Schemas with TensorZero](/gateway/create-a-prompt-template)
* **Automated Prompt Engineering.**
TensorZero implements GEPA, the leading automated prompt engineering algorithms recommended by DSPy.
GEPA iteratively refines your prompt templates based on an inference evaluation.
[→ Guide: Optimize your prompts with GEPA](/optimization/gepa)
## Key Differences
### TensorZero
* **Production Infrastructure.**
TensorZero provides complete production infrastructure including **observability, optimization, evaluations, and experimentation** capabilities.
DSPy focuses on the development phase and prompt programming patterns.
* **Model Optimization.**
TensorZero provides tools for optimizing models, including fine-tuning and RLHF.
DSPy primarily focuses on automated prompt engineering.
[→ LLM Optimization with TensorZero](/optimization/)
* **Inference-Time Optimization.**
TensorZero provides inference-time optimizations like dynamic in-context learning.
DSPy focuses on offline optimization strategies (e.g. static in-context learning).
[→ Inference-Time Optimizations with TensorZero](/gateway/guides/inference-time-optimizations/)
### DSPy
* **Advanced Automated Prompt Engineering.**
DSPy provides sophisticated automated prompt engineering tools for LLMs like teleprompters, recursive reasoning, and self-improvement loops.
TensorZero has some built-in prompt optimization features (more on the way) and integrates with DSPy for additional capabilities.
* **Lightweight Design.**
DSPy is a lightweight framework focused solely on LLM programming patterns, particularly during the R\&D stage.
TensorZero is a more comprehensive platform with additional infrastructure components covering end-to-end LLM engineering workflows.
Is TensorZero missing any features that are really important to you? Let us know on [GitHub Discussions](https://github.com/tensorzero/tensorzero/discussions), [Slack](https://www.tensorzero.com/slack), or [Discord](https://www.tensorzero.com/discord).
## Combining TensorZero and DSPy
You can get the best of both worlds by using DSPy and TensorZero together!
TensorZero provides a number of pre-built optimization recipes covering common LLM engineering workflows like supervised fine-tuning and RLHF.
But you can also easily export observability data for your own recipes and workflows.

---

## 

**URL:** https://www.tensorzero.com/docs/comparison/kong.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Comparison: TensorZero vs. Kong AI Gateway
> TensorZero is an open-source alternative to Kong AI Gateway featuring an LLM gateway, observability, optimization, evaluations, and experimentation.
TensorZero and Kong both offer an LLM gateway, but they focus on different things.
TensorZero is a full-stack LLMOps platform that provides an LLM gateway, observability, evaluations, optimization, and experimentation.
Kong is a general-purpose API gateway platform whose AI Gateway adds LLM-specific capabilities like provider routing, governance plugins, and token-aware rate limiting on top of its existing enterprise features.
That said, **you can get the best of both worlds by using them together: Kong at the edge for API management and TensorZero for LLMOps workflows**.
## Similarities
* **Unified Inference API.**
Both TensorZero and Kong AI Gateway offer a unified API that allows you to access LLMs from multiple model providers with a single integration, with broad support for features like structured generation, tool use, file inputs, and more.
[→ TensorZero Gateway Quickstart](/quickstart/)
* **Automatic Fallbacks for Higher Reliability.**
Both TensorZero and Kong AI Gateway offer automatic fallbacks and load balancing between model providers to increase reliability.
[→ Retries & Fallbacks with TensorZero](/gateway/guides/retries-fallbacks/)
* **LLM Observability.**
Both TensorZero and Kong AI Gateway support OpenTelemetry for exporting traces and metrics.
TensorZero also provides its own LLM-native observability (structured inference + feedback data stored in your database), while Kong provides API-oriented observability (e.g. request latency breakdowns, GenAI OpenTelemetry span attributes).
[→ TensorZero Observability Overview](/observability/)
* **LLM Controls.**
Both TensorZero and Kong AI Gateway offer operational controls for LLM traffic, including rate limiting (by tokens, cost, or request count), usage and spend tracking, credential management, and custom API keys for access control.
[→ Enforce custom rate limits](/operations/enforce-custom-rate-limits/)
[→ Track usage and cost](/operations/track-usage-and-cost/)
[→ Manage credentials (API keys)](/operations/manage-credentials/)
[→ Set up auth for TensorZero](/operations/set-up-auth-for-tensorzero/)
* **Open Source.**
Both TensorZero and Kong Gateway are open source (Apache 2.0).
However, TensorZero is fully open-source, whereas the Kong AI Gateway gates many features behind an enterprise license.
## Key Differences
### TensorZero
* **High Performance.**
The TensorZero Gateway was built from the ground up in Rust with performance in mind (\<1ms P99 latency at 10,000 QPS).
Kong AI Gateway does not publish comparable LLM-specific overhead benchmarks, and its performance is highly dependent on the plugin chain configured.
[→ TensorZero Performance Benchmarks](/gateway/benchmarks/)
* **Built-in LLM-Native Observability.**
TensorZero collects structured inference and feedback data in your own database (Postgres or ClickHouse), enabling a closed-loop system where production data feeds directly into evaluations, optimization, and experimentation.
Kong's observability is API-platform-native (standard request metrics and OpenTelemetry export) rather than LLM-workflow-native.
* **Built-in Evaluations.**
TensorZero offers built-in evaluation functionality, including heuristics and LLM judges.
Kong AI Gateway does not offer native evaluation capabilities; evaluation is typically handled by external tools and pipelines.
[→ TensorZero Evaluations Overview](/evaluations/)
* **Automated Experimentation (A/B Testing).**
TensorZero offers built-in experimentation features, including adaptive A/B testing, to help you identify the best models and prompts for your use cases.
Kong AI Gateway can do weighted routing and load balancing, but metric-driven experiment orchestration is not a built-in feature.
[→ Run adaptive A/B tests with TensorZero](/experimentation/run-adaptive-ab-tests/)
* **Built-in Inference-Time Optimizations.**
TensorZero offers built-in inference-time optimizations (e.g. dynamic in-context learning), allowing you to optimize your inference performance.
Kong AI Gateway does not offer inference-time optimizations in this sense.
[→ Inference-Time Optimizations with TensorZero](/gateway/guides/inference-time-optimizations/)
* **Optimization Recipes.**
TensorZero offers optimization recipes (e.g. supervised fine-tuning, RLHF, GEPA) that leverage your own data to improve your LLM's performance.
Kong AI Gateway does not offer any features like this.
[→ LLM Optimization with TensorZero](/optimization/)
* **Schemas, Templates, GitOps.**
TensorZero enables a schema-first approach to building LLM applications, allowing you to separate your application logic from LLM implementation details.
This approach allows you to more easily manage complex LLM applications, benefit from GitOps for prompt and configuration management, counterfactually improve data for optimization, and more.
Kong AI Gateway does not offer a comparable schema-first approach for LLM applications.
[→ Prompt Templates & Schemas with TensorZero](/gateway/create-a-prompt-template)
* **Inference Caching.**
TensorZero offers open-source inference caching features, allowing you to cache requests to improve latency and reduce costs.
Kong offers a semantic cache plugin, but it is only available as part of AI Gateway Enterprise.
[→ Inference Caching with TensorZero](/gateway/guides/inference-caching/)
### Kong AI Gateway
* **General-Purpose API Gateway Platform.**
Kong AI Gateway is built on top of Kong Gateway, a mature API gateway and reverse proxy.
This means it inherits a rich set of platform capabilities that go far beyond LLM traffic.
TensorZero is purpose-built for LLM inference and LLMOps workflows, not general API management.
* **Enterprise Governance.**
Kong AI Gateway offers dedicated AI governance plugins (e.g. PII sanitization) that can enforce policies at the gateway layer.
TensorZero does not offer built-in guardrails or content safety plugins; you'll need to integrate with other tools.
* **Enterprise API Security Features.**
Kong offers RBAC with roles/permissions, vault-backed secret stores, and other security features on the enterprise plan.
TensorZero offers API key authentication, credential management, and custom rate limiting, but you'll need to layer specialized security and/or networking software for additional general-purpose API controls.
* **Plugin Ecosystem.**
Kong offers a large plugin marketplace with categories spanning authentication, security, traffic control, logging, and more, plus a Plugin Development Kit (PDK) for custom plugins.
TensorZero supports industry standards (e.g. OpenTelemetry, OpenInference) and offers configuration-driven extensibility, but does not have a built-in plugin marketplace.
* **Managed Service.**
Kong offers Konnect, a managed SaaS/hybrid platform for managing Kong Gateway deployments.
TensorZero is fully open-source and self-hosted.
Is TensorZero missing any features that are really important to you? Let us know on [GitHub Discussions](https://github.com/tensorzero/tensorzero/discussions), [Slack](https://www.tensorzero.com/slack), or [Discord](https://www.tensorzero.com/discord).
## Combining TensorZero and Kong AI Gateway
You can get the best of both worlds by using Kong Gateway (with or without AI Gateway features) at the edge for API management and TensorZero for LLM-specific capabilities (observability, evaluations, optimization, experimentation).
In this architecture, Kong handles platform-level concerns like authentication and enterprise governance while TensorZero handles the LLMOps loop, giving you enterprise API management alongside continuous model improvement.

---

## 

**URL:** https://www.tensorzero.com/docs/comparison/langchain.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Comparison: TensorZero vs. LangChain
> TensorZero is an open-source alternative to LangChain featuring an LLM gateway, observability, optimization, evaluations, and experimentation.
TensorZero and LangChain both provide tools for LLM orchestration, but they serve different purposes in the ecosystem.
While LangChain focuses on rapid prototyping with a large ecosystem of integrations, TensorZero is designed for production-grade deployments with built-in observability, optimization, evaluations, and experimentation capabilities.
We provide a minimal example [integrating TensorZero with LangGraph](https://github.com/tensorzero/tensorzero/tree/main/examples/integrations/langgraph).
## Similarities
* **LLM Orchestration.**
Both TensorZero and LangChain are developer tools that streamline LLM engineering workflows.
TensorZero focuses on production-grade deployments and end-to-end LLM engineering workflows (inference, observability, optimization, evaluations, experimentation).
LangChain focuses on rapid prototyping and offers complementary commercial products for features like observability.
* **Open Source.**
Both TensorZero (Apache 2.0) and LangChain (MIT) are open-source.
TensorZero is fully open-source (including TensorZero UI for observability), whereas LangChain requires a commercial offering for certain features (e.g. LangSmith for observability).
* **Unified Interface.**
Both TensorZero and LangChain offer a unified interface that allows you to access LLMs from most major model providers with a single integration, with support for structured outputs, tool use, streaming, and more.
[→ TensorZero Gateway Quickstart](/quickstart/)
* **Inference-Time Optimizations.**
Both TensorZero and LangChain offer inference-time optimizations like dynamic in-context learning.
[→ Inference-Time Optimizations with TensorZero](/gateway/guides/inference-time-optimizations/)
* **Inference Caching.**
Both TensorZero and LangChain allow you to cache requests to improve latency and reduce costs.
[→ Inference Caching with TensorZero](/gateway/guides/inference-caching/)
## Key Differences
### TensorZero
* **Separation of Concerns: Application Engineering vs. LLM Optimization.**
TensorZero enables a clear separation between application logic and LLM implementation details.
By treating LLM functions as interfaces with structured inputs and outputs, TensorZero allows you to swap implementations without changing application code.
This approach makes it easier to manage complex LLM applications, enables GitOps for prompt and configuration management, and streamlines optimization and experimentation workflows.
LangChain blends application logic with LLM implementation details, streamlining rapid prototyping but making it harder to maintain and optimize complex applications.
[→ Prompt Templates & Schemas with TensorZero](/gateway/create-a-prompt-template)
[→ Advanced: Think of LLM Applications as POMDPs — Not Agents](https://www.tensorzero.com/blog/think-of-llm-applications-as-pomdps-not-agents/)
* **Open-Source Observability.**
TensorZero offers built-in observability features (including UI), collecting inference and feedback data in your own database.
LangChain requires a separate commercial service (LangSmith) for observability.
* **Built-in Optimization.**
TensorZero offers built-in optimization features, including supervised fine-tuning, RLHF, and automated prompt engineering recipes.
With the TensorZero UI, you can fine-tune models using your inference and feedback data in just a few clicks.
LangChain doesn't offer any built-in optimization features.
[→ LLM Optimization with TensorZero](/optimization/)
* **Built-in Evaluations.**
TensorZero offers built-in evaluation functionality, including heuristics and LLM judges.
LangChain requires a separate commercial service (LangSmith) for evaluations.
[→ TensorZero Evaluations Overview](/evaluations/)
* **Automated Experimentation (A/B Testing).**
TensorZero offers built-in experimentation features, allowing you to run experiments on your prompts, models, and inference strategies.
LangChain doesn't offer any experimentation features.
[→ Run adaptive A/B tests with TensorZero](/experimentation/run-adaptive-ab-tests/)
* **Performance & Scalability.**
TensorZero is built from the ground up for high performance, with a focus on low latency and high throughput.
LangChain introduces substantial latency and memory overhead to your application.
[→ TensorZero Gateway Benchmarks](/gateway/benchmarks/)
* **Language and Platform Agnostic.**
TensorZero is language and platform agnostic; in addition to its Python client, it supports any language that can make HTTP requests.
LangChain only supports applications built in Python and JavaScript.
[→ TensorZero Gateway API Reference](/gateway/api-reference/inference/)
* **Batch Inference.**
TensorZero supports batch inference with certain model providers, which significantly reduces inference costs.
LangChain doesn't support batch inference.
[→ Batch Inference with TensorZero](/gateway/guides/batch-inference/)
* **Credential Management.**
TensorZero streamlines credential management for your model providers, allowing you to manage your API keys in a single place and set up advanced workflows like load balancing between API keys.
LangChain only offers basic credential management features.
[→ Credential Management with TensorZero](/operations/manage-credentials/)
* **Automatic Fallbacks for Higher Reliability.**
TensorZero allows you to very easily set up retries, fallbacks, load balancing, and routing to increase reliability.
LangChain only offers basic, cumbersome fallback functionality.
[→ Retries & Fallbacks with TensorZero](/gateway/guides/retries-fallbacks/)
### LangChain
* **Focus on Rapid Prototyping.**
LangChain is designed for rapid prototyping, with a focus on ease of use and rapid iteration.
TensorZero is designed for production-grade deployments, so it requires more setup and configuration (e.g. a database to store your observability data) — but you can still get started in minutes.
[→ TensorZero Quickstart — From 0 to Observability & Fine-Tuning](/quickstart/)
* **Ecosystem of Integrations.**
LangChain has a large ecosystem of integrations with other libraries and tools, including model providers, vector databases, observability tools, and more.
TensorZero provides many integrations with model providers, but delegates other integrations to the user.
* **Managed Service.**
LangChain offers paid managed (hosted) services for features like observability (LangSmith).
TensorZero is fully open-source and self-hosted.
Is TensorZero missing any features that are really important to you? Let us know on [GitHub Discussions](https://github.com/tensorzero/tensorzero/discussions), [Slack](https://www.tensorzero.com/slack), or [Discord](https://www.tensorzero.com/discord).

---

## 

**URL:** https://www.tensorzero.com/docs/comparison/langfuse.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Comparison: TensorZero vs. Langfuse
> TensorZero is an open-source alternative to Langfuse featuring an LLM gateway, observability, optimization, evaluations, and experimentation.
TensorZero and Langfuse both provide open-source tools that streamline LLM engineering workflows.
TensorZero focuses on inference and optimization, while Langfuse specializes in powerful interfaces for observability and evals.
That said, **you can get the best of both worlds by using TensorZero alongside Langfuse**.
## Similarities
* **Open Source & Self-Hosted.**
Both TensorZero and Langfuse are open source and self-hosted.
Your data never leaves your infrastructure, and you don't risk downtime by relying on external APIs.
TensorZero is fully open-source, whereas Langfuse gates some of its features behind a paid license.
* **Built-in Observability.**
Both TensorZero and Langfuse offer built-in observability features, collecting inference in your own database.
Langfuse offers a broader set of advanced observability features, including application-level tracing.
TensorZero focuses more on structured data collection for optimization, including downstream metrics and feedback.
* **Built-in Evaluations.**
Both TensorZero and Langfuse offer built-in evaluations features, enabling you to sanity check and benchmark the performance of your prompts, models, and more — using heuristics and LLM judges.
TensorZero LLM judges are also TensorZero functions, which means you can optimize them using TensorZero's optimization recipes.
Langfuse offers a broader set of built-in heuristics and UI features for evaluations.
[→ TensorZero Evaluations Overview](/evaluations/)
## Key Differences
### TensorZero
* **Unified Inference API.**
TensorZero offers a unified inference API that allows you to access LLMs from most major model providers with a single integration, with support for structured outputs, tool use, streaming, and more.
Langfuse doesn't provide a built-in LLM gateway.
[→ TensorZero Gateway Quickstart](/quickstart/)
* **Built-in Inference-Time Optimizations.**
TensorZero offers built-in inference-time optimizations (e.g. dynamic in-context learning), allowing you to optimize your inference performance.
Langfuse doesn't offer any inference-time optimizations.
[→ Inference-Time Optimizations with TensorZero](/gateway/guides/inference-time-optimizations/)
* **Optimization Recipes.**
TensorZero offers optimization recipes (e.g. supervised fine-tuning, RLHF, GEPA) that leverage your own data to improve your LLM's performance.
Langfuse doesn't offer built-in features like this.
[→ LLM Optimization with TensorZero](/optimization/)
* **Automatic Fallbacks for Higher Reliability.**
TensorZero offers automatic fallbacks to increase reliability.
Langfuse doesn't offer any such features.
[→ Retries & Fallbacks with TensorZero](/gateway/guides/retries-fallbacks/)
* **Automated Experimentation (A/B Testing).**
TensorZero offers built-in experimentation features, allowing you to run experiments on your prompts, models, and inference strategies.
Langfuse doesn't offer any experimentation features.
[→ Run adaptive A/B tests with TensorZero](/experimentation/run-adaptive-ab-tests/)
### Langfuse
* **Advanced Observability & Evaluations.**
While both TensorZero and Langfuse offer observability and evaluations features, Langfuse takes it further with advanced observability features.
Additionally, Langfuse offers a prompt playground, which TensorZero doesn't offer (coming soon!).
* **Access Control.**
Langfuse offers access control features like SSO and user management.
TensorZero supports TensorZero API key for inference, but more advanced access control requires complementary tools like Nginx or OAuth2 Proxy.
[→ Set up auth for TensorZero](/operations/set-up-auth-for-tensorzero)
* **Managed Service.**
Langfuse offers a paid managed (hosted) service in addition to the open-source version.
TensorZero is fully open-source and self-hosted.
Is TensorZero missing any features that are really important to you? Let us know on [GitHub Discussions](https://github.com/tensorzero/tensorzero/discussions), [Slack](https://www.tensorzero.com/slack), or [Discord](https://www.tensorzero.com/discord).
## Combining TensorZero and Langfuse
You can combine TensorZero and Langfuse to get the best of both worlds.
A leading voice agent startup uses TensorZero for inference and optimization, alongside Langfuse for more advanced observability and evals.

---

## 

**URL:** https://www.tensorzero.com/docs/comparison/litellm.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Comparison: TensorZero vs. LiteLLM
> TensorZero is an open-source alternative to LiteLLM featuring an LLM gateway, observability, optimization, evaluations, and experimentation.
TensorZero and LiteLLM both offer a unified inference API for LLMs, but they have different features beyond that.
TensorZero offers a broader set of features (including observability, optimization, evaluations, and experimentation), whereas LiteLLM offers more traditional gateway features (e.g. budgeting, queuing) and third-party integrations.
That said, **you can get the best of both worlds by using LiteLLM as a model provider inside TensorZero**!
## Similarities
* **Unified Inference API.**
Both TensorZero and LiteLLM offer a unified inference API that allows you to access LLMs from most major model providers with a single integration, with support for structured outputs, batch inference, tool use, streaming, and more.
[→ TensorZero Gateway Quickstart](/quickstart/)
* **Automatic Fallbacks for Higher Reliability.**
Both TensorZero and LiteLLM offer automatic fallbacks to increase reliability.
[→ Retries & Fallbacks with TensorZero](/gateway/guides/retries-fallbacks/)
* **Open Source & Self-Hosted.**
Both TensorZero and LiteLLM are open source and self-hosted.
Your data never leaves your infrastructure, and you don't risk downtime by relying on external APIs.
TensorZero is fully open-source, whereas LiteLLM gates some of its features behind an enterprise license.
* **Inference Caching.**
Both TensorZero and LiteLLM allow you to cache requests to improve latency and reduce costs.
[→ Inference Caching with TensorZero](/gateway/guides/inference-caching/)
* **Multimodal Inference.**
Both TensorZero and LiteLLM support multimodal inference.
[→ Multimodal Inference with TensorZero](/gateway/call-llms-with-image-and-file-inputs/)
## Key Differences
### TensorZero
* **High Performance.**
The TensorZero Gateway was built from the ground up in Rust 🦀 with performance in mind (\<1ms P99 latency at 10,000 QPS).
LiteLLM is built in Python, resulting in 25-100x+ latency overhead and much lower throughput.
[→ Performance Benchmarks: TensorZero vs. LiteLLM](/gateway/benchmarks/)
* **Built-in Observability.**
TensorZero offers its own observability features, collecting inference and feedback data in your own database.
LiteLLM only offers integrations with third-party observability tools like Langfuse.
* **Built-in Evaluations.**
TensorZero offers built-in evaluation functionality, including heuristics and LLM judges.
LiteLLM doesn't offer any evaluations functionality.
[→ TensorZero Evaluations Overview](/evaluations/)
* **Automated Experimentation (A/B Testing).**
TensorZero offers built-in experimentation features, allowing you to run experiments on your prompts, models, and inference strategies.
LiteLLM doesn't offer any experimentation features.
[→ Run adaptive A/B tests with TensorZero](/experimentation/run-adaptive-ab-tests/)
* **Built-in Inference-Time Optimizations.**
TensorZero offers built-in inference-time optimizations (e.g. dynamic in-context learning), allowing you to optimize your inference performance.
LiteLLM doesn't offer any inference-time optimizations.
[→ Inference-Time Optimizations with TensorZero](/gateway/guides/inference-time-optimizations/)
* **Optimization Recipes.**
TensorZero offers optimization recipes (e.g. supervised fine-tuning, RLHF, GEPA) that leverage your own data to improve your LLM's performance.
LiteLLM doesn't offer any features like this.
[→ LLM Optimization with TensorZero](/optimization/)
* **Schemas, Templates, GitOps.**
TensorZero enables a schema-first approach to building LLM applications, allowing you to separate your application logic from LLM implementation details.
This approach allows your to more easily manage complex LLM applications, benefit from GitOps for prompt and configuration management, counterfactually improve data for optimization, and more.
LiteLLM only offers the standard unstructured chat completion interface.
[→ Prompt Templates & Schemas with TensorZero](/gateway/create-a-prompt-template)
* **Access Control.**
Both TensorZero and LiteLLM support virtual (custom) API keys to authenticate requests.
LiteLLM offers advanced authentication features in its enterprise plan, whereas TensorZero requires complementary open-source tools like Nginx or OAuth2 Proxy for such use cases.
[→ Set up auth for TensorZero](/operations/set-up-auth-for-tensorzero)
### LiteLLM
* **Dynamic Provider Routing.**
LiteLLM allows you to dynamically route requests to different model providers based on latency, cost, and rate limits.
TensorZero only offers static routing capabilities, i.e. a pre-defined sequence of model providers to attempt.
[→ Retries & Fallbacks with TensorZero](/gateway/guides/retries-fallbacks/)
* **Request Prioritization.**
LiteLLM allows you to prioritize requests over others, which can be useful for high-priority tasks when you're constrained by rate limits.
TensorZero doesn't offer request prioritization, and instead requires you to manage the request queue externally (e.g. using Redis).
* **Built-in Guardrails Integration.**
LiteLLM offers built-in support for integrations with guardrails tools like AWS Bedrock.
For now, TensorZero doesn't offer built-in guardrails, and instead requires you to manage integrations yourself.
* **Managed Service.**
LiteLLM offers a paid managed (hosted) service in addition to the open-source version.
TensorZero is fully open-source and self-hosted.
Is TensorZero missing any features that are really important to you? Let us know on [GitHub Discussions](https://github.com/tensorzero/tensorzero/discussions), [Slack](https://www.tensorzero.com/slack), or [Discord](https://www.tensorzero.com/discord).
## Combining TensorZero and LiteLLM
You can get the best of both worlds by using LiteLLM as a model provider inside TensorZero.
LiteLLM offers an OpenAI-compatible API, so you can use TensorZero's OpenAI-compatible endpoint to call LiteLLM.
Learn more about using [OpenAI-compatible endpoints](/integrations/model-providers/openai-compatible/).

---

## 

**URL:** https://www.tensorzero.com/docs/comparison/openpipe.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Comparison: TensorZero vs. OpenPipe
> TensorZero is an open-source alternative to OpenPipe featuring an LLM gateway, observability, optimization, evaluations, and experimentation.
TensorZero and OpenPipe both provide tools that streamline fine-tuning workflows for LLMs.
TensorZero is open-source and self-hosted, while OpenPipe is a paid managed service (inference costs \~2x more than specialized providers supported by TensorZero).
That said, **you can get the best of both worlds by using OpenPipe as a model provider inside TensorZero**.
## Similarities
* **LLM Optimization (Fine-Tuning).**
Both TensorZero and OpenPipe focus on LLM optimization (e.g. fine-tuning, DPO).
OpenPipe focuses on fine-tuning, while TensorZero provides a complete set of tools for optimizing LLM systems (including prompts, models, and inference strategies).
[→ LLM Optimization with TensorZero](/optimization/)
* **Built-in Observability.**
Both TensorZero and OpenPipe offer built-in observability features.
TensorZero stores inference data in your own database for full privacy and control, while OpenPipe stores it themselves in their own cloud.
* **Built-in Evaluations.**
Both TensorZero and OpenPipe offer built-in evaluations features, enabling you to sanity check and benchmark the performance of your prompts, models, and more — using heuristics and LLM judges.
TensorZero LLM judges are also TensorZero functions, which means you can optimize them using TensorZero's optimization recipes.
[→ TensorZero Evaluations Overview](/evaluations/)
## Key Differences
### TensorZero
* **Open Source & Self-Hosted.**
TensorZero is fully open source and self-hosted.
Your data never leaves your infrastructure, and you don't risk downtime by relying on external APIs.
OpenPipe is a closed-source managed service.
* **No Added Cost (& Cheaper Inference Providers).**
TensorZero is free to use: your bring your own LLM API keys and there is no additional cost.
OpenPipe charges \~2x on inference costs compared to specialized providers supported by TensorZero (e.g. Fireworks AI).
* **Unified Inference API.**
TensorZero offers a unified inference API that allows you to access LLMs from most major model providers with a single integration, with support for structured outputs, tool use, streaming, and more.
OpenPipe supports a much smaller set of LLMs.
[→ TensorZero Gateway Quickstart](/quickstart/)
* **Built-in Inference-Time Optimizations.**
TensorZero offers built-in inference-time optimizations (e.g. dynamic in-context learning), allowing you to optimize your inference performance.
OpenPipe doesn't offer any inference-time optimizations.
[→ Inference-Time Optimizations with TensorZero](/gateway/guides/inference-time-optimizations/)
* **Automatic Fallbacks for Higher Reliability.**
TensorZero is self-hosted and provides automatic fallbacks between model providers to increase reliability.
OpenPipe can fallback their own models to other OpenAI-compatible APIs, but if OpenPipe itself goes down, you're out of luck.
[→ Retries & Fallbacks with TensorZero](/gateway/guides/retries-fallbacks/)
* **Automated Experimentation (A/B Testing).**
TensorZero offers built-in experimentation features, allowing you to run experiments on your prompts, models, and inference strategies.
OpenPipe doesn't offer any experimentation features.
[→ Run adaptive A/B tests with TensorZero](/experimentation/run-adaptive-ab-tests/)
* **Batch Inference.**
TensorZero supports batch inference with certain model providers, which significantly reduces inference costs.
OpenPipe doesn't support batch inference.
[→ Batch Inference with TensorZero](/gateway/guides/batch-inference/)
* **Inference Caching.**
Both TensorZero and OpenPipe allow you to cache requests to improve latency and reduce costs.
OpenPipe only caches requests to their own models, while TensorZero caches requests to all model providers.
[→ Inference Caching with TensorZero](/gateway/guides/inference-caching/)
* **Schemas, Templates, GitOps.**
TensorZero enables a schema-first approach to building LLM applications, allowing you to separate your application logic from LLM implementation details.
This approach allows your to more easily manage complex LLM applications, benefit from GitOps for prompt and configuration management, counterfactually improve data for optimization, and more.
OpenPipe only offers the standard unstructured chat completion interface.
[→ Prompt Templates & Schemas with TensorZero](/gateway/create-a-prompt-template)
### OpenPipe
* **Guardrails.**
OpenPipe offers guardrails (runtime AI judges) for your fine-tuned models.
TensorZero doesn't offer built-in guardrails, and instead requires you to manage them yourself.
Is TensorZero missing any features that are really important to you? Let us know on [GitHub Discussions](https://github.com/tensorzero/tensorzero/discussions), [Slack](https://www.tensorzero.com/slack), or [Discord](https://www.tensorzero.com/discord).
## Combining TensorZero and OpenPipe
You can get the best of both worlds by using OpenPipe as a model provider inside TensorZero.
OpenPipe provides an OpenAI-compatible API, so you can use models previously fine-tuned with OpenPipe with TensorZero.
Learn more about using [OpenAI-compatible endpoints](/integrations/model-providers/openai-compatible/).

---

## 

**URL:** https://www.tensorzero.com/docs/comparison/openrouter.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Comparison: TensorZero vs. OpenRouter
> TensorZero is an open-source alternative to OpenRouter featuring an LLM gateway, observability, optimization, evaluations, and experimentation.
TensorZero and OpenRouter both offer a unified inference API for LLMs, but they have different features beyond that.
TensorZero offers a more comprehensive set of features (including observability, optimization, evaluations, and experimentation), whereas OpenRouter offers more dynamic routing capabilities.
That said, **you can get the best of both worlds by using OpenRouter as a model provider inside TensorZero**!
## Similarities
* **Unified Inference API.**
Both TensorZero and OpenRouter offer a unified inference API that allows you to access LLMs from most major model providers with a single integration, with support for structured outputs, tool use, streaming, and more.
[→ TensorZero Gateway Quickstart](/quickstart/)
* **Automatic Fallbacks for Higher Reliability.**
Both TensorZero and OpenRouter offer automatic fallbacks to increase reliability.
[→ Retries & Fallbacks with TensorZero](/gateway/guides/retries-fallbacks/)
## Key Differences
### TensorZero
* **Open Source & Self-Hosted.**
TensorZero is fully open source and self-hosted.
Your data never leaves your infrastructure, and you don't risk downtime by relying on external APIs.
OpenRouter is a closed-source external API.
* **No Added Cost.**
TensorZero is free to use: your bring your own LLM API keys and there is no additional cost.
OpenRouter charges 5% of your inference spend when you bring your own API keys.
* **Built-in Observability.**
TensorZero offers built-in observability features, collecting inference and feedback data in your own database.
OpenRouter doesn't offer any observability features.
* **Built-in Evaluations.**
TensorZero offers built-in functionality, including heuristics and LLM judges.
OpenRouter doesn't offer any evaluation features.
[→ TensorZero Evaluations Overview](/evaluations/)
* **Automated Experimentation (A/B Testing).**
TensorZero offers built-in experimentation features, allowing you to run experiments on your prompts, models, and inference strategies.
OpenRouter doesn't offer any experimentation features.
[→ Run adaptive A/B tests with TensorZero](/experimentation/run-adaptive-ab-tests/)
* **Built-in Inference-Time Optimizations.**
TensorZero offers built-in inference-time optimizations (e.g. dynamic in-context learning), allowing you to optimize your inference performance.
OpenRouter doesn't offer any inference-time optimizations, except for dynamic model routing via NotDiamond.
[→ Inference-Time Optimizations with TensorZero](/gateway/guides/inference-time-optimizations/)
* **Optimization Recipes.**
TensorZero offers optimization recipes (e.g. supervised fine-tuning, RLHF, GEPA) that leverage your own data to improve your LLM's performance.
OpenRouter doesn't offer any features like this.
[→ LLM Optimization with TensorZero](/optimization/)
* **Batch Inference.**
TensorZero supports batch inference with certain model providers, which significantly reduces inference costs.
OpenRouter doesn't support batch inference.
[→ Batch Inference with TensorZero](/gateway/guides/batch-inference/)
* **Inference Caching.**
TensorZero offers inference caching, which can significantly reduce inference costs and latency.
OpenRouter doesn't offer inference caching.
[→ Inference Caching with TensorZero](/gateway/guides/inference-caching/)
* **Schemas, Templates, GitOps.**
TensorZero enables a schema-first approach to building LLM applications, allowing you to separate your application logic from LLM implementation details.
This approach allows your to more easily manage complex LLM applications, benefit from GitOps for prompt and configuration management, counterfactually improve data for optimization, and more.
OpenRouter only offers the standard unstructured chat completion interface.
[→ Prompt Templates & Schemas with TensorZero](/gateway/create-a-prompt-template)
### OpenRouter
* **Dynamic Provider Routing.**
OpenRouter allows you to dynamically route requests to different model providers based on latency, cost, and availability.
TensorZero only offers static routing capabilities, i.e. a pre-defined sequence of model providers to attempt.
[→ Retries & Fallbacks with TensorZero](/gateway/guides/retries-fallbacks/)
* **Dynamic Model Routing.**
OpenRouter integrates with NotDiamond to offer dynamic model routing based on input.
TensorZero supports other inference-time optimizations but doesn't support dynamic model routing at this time.
[→ Inference-Time Optimizations with TensorZero](/gateway/guides/inference-time-optimizations/)
* **Consolidated Billing.**
OpenRouter allows you to access every supported model using a single OpenRouter API key.
Under the hood, OpenRouter uses their own API keys with model providers.
This approach can increase your rate limits and streamline billing, but slightly increases your inference costs.
TensorZero requires you to use your own API keys, without any added cost.
Is TensorZero missing any features that are really important to you? Let us know on [GitHub Discussions](https://github.com/tensorzero/tensorzero/discussions), [Slack](https://www.tensorzero.com/slack), or [Discord](https://www.tensorzero.com/discord).
## Combining TensorZero and OpenRouter
You can get the best of both worlds by using OpenRouter as a model provider inside TensorZero.
Learn more about using [OpenRouter as a model provider](/integrations/model-providers/openrouter/).

---

## 

**URL:** https://www.tensorzero.com/docs/comparison/portkey.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Comparison: TensorZero vs. Portkey
> TensorZero is an open-source alternative to Portkey featuring an LLM gateway, observability, optimization, evaluations, and experimentation.
TensorZero and Portkey offer diverse features to streamline LLM engineering, including an LLM gateway, observability tools, and more.
TensorZero is fully open-source and self-hosted, while Portkey offers an open-source gateway but otherwise requires a paid commercial (hosted) service.
Additionally, TensorZero has more features around LLM optimization (e.g. advanced fine-tuning workflows and inference-time optimizations), whereas Portkey has a broader set of features around the UI (e.g. prompt playground).
## Similarities
* **Unified Inference API.**
Both TensorZero and Portkey offer a unified inference API that allows you to access LLMs from most major model providers with a single integration, with support for structured outputs, batch inference, tool use, streaming, and more.
[→ TensorZero Gateway Quickstart](/quickstart/)
* **Automatic Fallbacks, Retries, & Load Balancing for Higher Reliability.**
Both TensorZero and Portkey offer automatic fallbacks, retries, and load balancing features to increase reliability.
[→ Retries & Fallbacks with TensorZero](/gateway/guides/retries-fallbacks/)
* **Schemas, Templates.**
Both TensorZero and Portkey offer schema and template features to help you manage your LLM applications.
[→ Prompt Templates & Schemas with TensorZero](/gateway/create-a-prompt-template)
* **Multimodal Inference.**
Both TensorZero and Portkey support multimodal inference.
[→ Multimodal Inference with TensorZero](/gateway/call-llms-with-image-and-file-inputs/)
## Key Differences
### TensorZero
* **Open-Source Observability.**
TensorZero offers built-in open-source observability features, collecting inference and feedback data in your own database.
Portkey also offers observability features, but they are limited to their commercial (hosted) offering.
* **Built-in Evaluations.**
TensorZero offers built-in evaluation functionality, including heuristics and LLM judges.
Portkey doesn't offer any evaluation features.
[→ TensorZero Evaluations Overview](/evaluations/)
* **Open-Source Inference Caching.**
TensorZero offers open-source inference caching features, allowing you to cache requests to improve latency and reduce costs.
Portkey also offers inference caching features, but they are limited to their commercial (hosted) offering.
[→ Inference Caching with TensorZero](/gateway/guides/inference-caching/)
* **Open-Source Fine-Tuning Workflows.**
TensorZero offers open-source built-in fine-tuning workflows, allowing you to create custom models using your own data.
Portkey also offers fine-tuning features, but they are limited to their enterprise (\$\$\$) offering.
[→ LLM Optimization with TensorZero](/optimization/)
* **Advanced Fine-Tuning Workflows.**
TensorZero offers advanced fine-tuning workflows, including the ability to curate datasets using feedback signals (e.g. production metrics) and the ability to use RLHF for reinforcement learning.
Portkey doesn't offer similar features.
[→ LLM Optimization with TensorZero](/optimization/)
* **Automated Experimentation (A/B Testing).**
TensorZero offers advanced A/B testing features, including automated experimentation, to help your identify the best models and prompts for your use cases.
Portkey only offers simple canary and A/B testing features.
[→ Run adaptive A/B tests with TensorZero](/experimentation/run-adaptive-ab-tests/)
* **Inference-Time Optimizations.**
TensorZero offers built-in inference-time optimizations (e.g. dynamic in-context learning), allowing you to optimize your inference performance.
Portkey doesn't offer any inference-time optimizations.
[→ Inference-Time Optimizations with TensorZero](/gateway/guides/inference-time-optimizations/)
* **Programmatic & GitOps-Friendly Orchestration.**
TensorZero can be fully orchestrated programmatically in a GitOps-friendly way.
Portkey can manage some of its features programmatically, but certain features depend on its external commercial hosted service.
* **Open-Source Access Control.**
Both TensorZero and Portkey offer access control features like TensorZero API keys.
Portkey only offers them in the commercial (hosted) offering, whereas TensorZero's solution is fully open-source.
[→ Set up auth for TensorZero](/operations/set-up-auth-for-tensorzero)
### Portkey
* **Prompt Playground.**
Portkey offers a prompt playground in its commercial (hosted) offering, allowing you to test your prompts and models in a graphical interface.
TensorZero doesn't offer a prompt playground today (coming soon!).
* **Guardrails.**
Portkey offers guardrails features, including integrations with third-party guardrails providers and the ability to use custom guardrails using webhooks.
For now, TensorZero doesn't offer built-in guardrails, and instead requires you to manage integrations yourself.
* **Managed Service.**
Portkey offers a paid managed (hosted) service in addition to the open-source version.
TensorZero is fully open-source and self-hosted.

---

## 

**URL:** https://www.tensorzero.com/docs/deployment/clickhouse.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Deploy ClickHouse (optional)
> Learn how to deploy ClickHouse for TensorZero's observability features.
The TensorZero Gateway can optionally collect inference and feedback data for observability, optimization, evaluation, and experimentation.
Under the hood, TensorZero stores this data in ClickHouse, an open-source columnar database that is optimized for analytical workloads.
If you're planning to use the gateway without observability, you don't need to deploy ClickHouse.
You can also [deploy Postgres](/deployment/postgres) as the backend for observability.
We recommend using ClickHouse if you're handling >100 inferences per second.
## Deploy
### Development
For development purposes, you can run a single-node ClickHouse instance locally (e.g. using Homebrew or Docker) or a cheap Development-tier cluster on ClickHouse Cloud.
See the [ClickHouse documentation](https://clickhouse.com/docs/install) for more details on configuring your ClickHouse deployment.
### Production
#### Managed deployments
For production deployments, the easiest setup is to use a managed service like
ClickHouse Cloud
.
ClickHouse Cloud is also available through the [AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-jettukeanwrfc), [GCP Marketplace](https://console.cloud.google.com/marketplace/product/clickhouse-public/clickhouse-cloud), and [Azure Marketplace](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/clickhouse.clickhouse_cloud).
Other options for managed ClickHouse deployments include [Tinybird](https://www.tinybird.co/) (serverless) and [Altinity](https://www.altinity.com/) (hands-on support).
TensorZero tests against ClickHouse Cloud's `regular` (recommended) and `fast` release channels.
#### Self-hosted deployments
You can alternatively run your own self-managed ClickHouse instance or cluster.
**We strongly recommend using ClickHouse `lts` instead of `latest` in production.**
We test against both versions, but ClickHouse `latest` often has bugs and breaking changes.
TensorZero supports single-node and replicated deployments.
TensorZero does not currently support **sharded** self-hosted ClickHouse deployments.
See the [ClickHouse documentation](https://clickhouse.com/docs/install) for more details on configuring your ClickHouse deployment.
## Configure
### Connect to ClickHouse
To configure TensorZero to use ClickHouse, set the `TENSORZERO_CLICKHOUSE_URL` environment variable with your ClickHouse connection details.
```bash title=".env" theme={null}
TENSORZERO_CLICKHOUSE_URL="http[s]://[username]:[password]@[hostname]:[port]/[database]"
# Example: ClickHouse running locally
TENSORZERO_CLICKHOUSE_URL="http://chuser:chpassword@localhost:8123/tensorzero"
# Example: ClickHouse Cloud
TENSORZERO_CLICKHOUSE_URL="https://USERNAME:PASSWORD@XXXXX.clickhouse.cloud:8443/tensorzero"
# Example: TensorZero Gateway running in a container, ClickHouse running on host machine
TENSORZERO_CLICKHOUSE_URL="http://host.docker.internal:8123/tensorzero"
```
If you're using a self-hosted replicated ClickHouse deployment, you must also provide the ClickHouse cluster name in the `TENSORZERO_CLICKHOUSE_CLUSTER_NAME` environment variable.
### Apply ClickHouse migrations
By default, the TensorZero Gateway applies ClickHouse migrations automatically when it starts up.
This behavior can be suppressed by setting `observability.disable_automatic_migrations = true` under the `[gateway]` section of `config/tensorzero.toml`.
See [https://www.tensorzero.com/docs/gateway/configuration-reference#gateway](https://www.tensorzero.com/docs/gateway/configuration-reference#gateway).
If automatic migrations are disabled, then you must apply them manually with
`docker run --rm -e TENSORZERO_CLICKHOUSE_URL=$TENSORZERO_CLICKHOUSE_URL tensorzero/gateway:{version} --run-clickhouse-migrations`.
The gateway will error on startup if automatic migrations are disabled and any required migrations are missing.
If you're using a self-hosted replicated ClickHouse deployment, you must apply database migrations manually;
they cannot be applied automatically.

---

## 

**URL:** https://www.tensorzero.com/docs/deployment/optimize-latency-and-throughput.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Optimize latency and throughput
> Learn how to optimize the performance of the TensorZero Gateway for lower latency and higher throughput.
The TensorZero Gateway is designed from the ground up with performance in mind.
Even with default settings, the gateway is fast and lightweight enough to be unnoticeable in most applications.
The best practices below are designed to help you optimize the performance of the TensorZero Gateway for production deployments requiring maximum performance.
The TensorZero Gateway can achieve \<1ms P99 latency overhead at 10,000+ QPS. See [Benchmarks](/gateway/benchmarks/) for details.
## Best practices
### Observability data collection strategy
By default, the gateway takes a conservative approach to observability data durability, ensuring that data is persisted in the database before sending a response to the client.
This strategy provides a consistent and reliable experience but can introduce latency overhead.
For scenarios where latency and throughput are critical, the gateway can be configured to sacrifice data durability guarantees for better performance.
If latency is critical for your application, you can enable `gateway.observability.async_writes` or `gateway.observability.batch_writes`.
With either of these settings, the gateway will return the response to the client immediately and asynchronously insert data into your database.
The former will immediately insert each row individually, while the latter will batch multiple rows together for more efficient writes.
As a rule of thumb, consider the following decision matrix:
|                             | **High throughput** | **Low throughput** |
| --------------------------- | ------------------- | ------------------ |
| **Latency is critical**     | `batch_writes`      | `async_writes`     |
| **Latency is not critical** | `batch_writes`      | Default strategy   |
See the [Configuration Reference](/gateway/configuration-reference/) for more details.
### Other recommendations
* Ensure your application, the TensorZero Gateway, and database are deployed in the same region to minimize network latency.
* Initialize the client once and reuse it as much as possible, to avoid initialization overhead and to keep the connection alive.

---

## 

**URL:** https://www.tensorzero.com/docs/deployment/postgres.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Deploy Postgres (optional)
> Learn how to deploy Postgres for observability and other advanced features.
The TensorZero Gateway can optionally collect inference and feedback data for observability, optimization, evaluation, and experimentation.
Postgres is the simplest way to get started.
If you're handling >100 inferences per second, consider [deploying ClickHouse](/deployment/clickhouse) instead.
For such deployments, Postgres is only necessary for advanced features like [running adaptive A/B tests](/experimentation/run-adaptive-ab-tests) and [setting up auth for TensorZero](/operations/set-up-auth-for-tensorzero).
## Set up
You can self-host Postgres or use a managed service (e.g. AWS RDS, Supabase, PlanetScale).
Follow the deployment instructions for your chosen service.
If you find any compatibility issues, please open a detailed [GitHub Discussion](https://github.com/tensorzero/tensorzero/discussions/new?category=bug-reports).
TensorZero requires the following Postgres extensions:
* [`pg_cron`](https://github.com/citusdata/pg_cron)
* [`pg_trgm`](https://www.postgresql.org/docs/current/pgtrgm.html)
* [`pgvector`](https://github.com/pgvector/pgvector)
**You must install `pg_cron` in the logical database used by TensorZero.**
Here are example setup instructions for some popular Postgres providers.
For convenience, we publish [`tensorzero/postgres`](https://hub.docker.com/r/tensorzero/postgres) Docker images that bundle all required extensions on top of the official Postgres image.
If you prefer to set up extensions yourself, install [`pg_cron`](https://github.com/citusdata/pg_cron#installing-pg_cron), [`pgvector`](https://github.com/pgvector/pgvector#installation), and [`pg_trgm`](https://www.postgresql.org/docs/current/pgtrgm.html).
Regardless of how you install the extensions, you must set `cron.database_name` to the database used by TensorZero. You can do this via a CLI flag (`postgres -c cron.database_name=tensorzero`) or in `postgresql.conf`:
```ini title="postgresql.conf" theme={null}
shared_preload_libraries = 'pg_cron'
# Use the name of TensorZero's database,
# or 'postgres' if you use the default database for TensorZero
cron.database_name = 'tensorzero'
```
If you use a non-default Postgres user for TensorZero, connect to your database and run:
```sql theme={null}
GRANT USAGE ON SCHEMA cron TO your_username;
```
Create a custom parameter group if you haven't already. Add `pg_cron` to the `shared_preload_libraries` parameter and set `cron.database_name` to your TensorZero database name.
Connect to your database and run:
```sql theme={null}
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS vector;
-- Replace `your_username` with TensorZero's Postgres user
GRANT USAGE ON SCHEMA cron TO your_username;
```
See the [AWS documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL_pg_cron.html) for more details.
Set the `cloudsql.enable_pg_cron` [database flag](https://docs.cloud.google.com/sql/docs/postgres/flags) to `on` and set the `cron.database_name` database flag to your TensorZero database name.
Connect to your database and run:
```sql theme={null}
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS vector;
```
If TensorZero uses a non-default user, grant permissions:
```sql theme={null}
-- Replace `your_username` with TensorZero's Postgres user
GRANT USAGE ON SCHEMA cron TO your_username;
```
See the [Cloud SQL documentation](https://docs.cloud.google.com/sql/docs/postgres/extensions#pg_cron) for more details.
Enable `pg_cron`, `vector`, and `pg_trgm` from the Database Extensions page in your Supabase dashboard, or run:
```sql theme={null}
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS vector;
```
See the [Supabase documentation](https://supabase.com/docs/guides/cron) for more details.
In the PlanetScale dashboard, select your database and navigate to **Clusters**. Select the branch to configure, then go to the **Extensions** tab. Enable `pg_cron`, `vector`, and `pg_trgm`, set `cron.database_name` to your TensorZero database name, then click **Queue extension changes** and **Apply changes**.
Connect to your database and run:
```sql theme={null}
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS vector;
```
See the [PlanetScale documentation](https://planetscale.com/docs/postgres/extensions/pg_cron) for more details.
To configure TensorZero to use Postgres, set the `TENSORZERO_POSTGRES_URL` environment variable with your Postgres connection details.
```bash title=".env" theme={null}
TENSORZERO_POSTGRES_URL="postgres://[username]:[password]@[hostname]:[port]/[database]"
# Example:
TENSORZERO_POSTGRES_URL="postgres://myuser:mypass@localhost:5432/tensorzero"
```
**Supabase Free Tier**
Supabase's direct connection URL requires IPv4, which is not available on the free tier.
Use the session pooler URL instead: in your Supabase dashboard, go to `Connect → Connection string` and select `Session pooler`.
If migrations fail with `prepared statement "sqlx_s_1" already exists`, switch to port `5432` in the connection string.
See [this Supabase issue](https://github.com/supabase/supavisor/issues/239#issuecomment-2085235022) for details.
Migrations may be slower through the session pooler.
**TensorZero does not automatically apply Postgres migrations.**
You must apply migrations manually with `gateway --run-postgres-migrations`.
See [Deploy the TensorZero Gateway](/deployment/tensorzero-gateway) for more details.
If you've configured the gateway with Docker Compose, you can run the migrations with:
```bash theme={null}
docker compose run --rm gateway --run-postgres-migrations
```
In most other cases, you can run the migrations with:
```bash theme={null}
docker run --rm --network host \
-e TENSORZERO_POSTGRES_URL \
tensorzero/gateway --run-postgres-migrations
```
You should re-run this command when upgrading TensorZero from an earlier version.

---

## 

**URL:** https://www.tensorzero.com/docs/deployment/tensorzero-autopilot.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Set up TensorZero Autopilot
> Learn how to set up TensorZero Autopilot on your self-hosted TensorZero deployment.
TensorZero Autopilot is an automated AI engineer that analyzes LLM observability data, optimizes prompts and models, sets up evals, and runs A/B tests.
It's an optional complementary service that runs on top of your self-hosted TensorZero deployment.
TensorZero Autopilot is currently in a private beta. [Schedule a demo →](https://www.tensorzero.com/schedule-demo)
## Deploy
Visit [autopilot.tensorzero.com](https://autopilot.tensorzero.com/) to generate an API key.
Set the environment variable `TENSORZERO_AUTOPILOT_API_KEY` for your TensorZero Gateway:
```bash theme={null}
export TENSORZERO_AUTOPILOT_API_KEY="sk-t0-..."
```
TensorZero Autopilot requires the TensorZero Gateway, TensorZero UI, and Postgres.
Make sure the gateway has the `TENSORZERO_AUTOPILOT_API_KEY` environment variable.
Learn more about how to:
* [Deploy the TensorZero Gateway](/deployment/tensorzero-gateway)
* [Deploy the TensorZero UI](/deployment/tensorzero-ui)
* [Deploy Postgres](/deployment/postgres)
TensorZero Autopilot can edit your local TensorZero configuration.
When this feature is enabled, an "Apply changes" button appears in the UI, allowing you to easily apply any configuration changes proposed by Autopilot.
We recommend enabling this feature only in a local setup.
Do not enable this feature if the TensorZero UI is shared by multiple users.
To enable it, mount the configuration directory as a writable volume and pass the `--config-file` flag to the TensorZero UI container.
For example:
```yaml theme={null}
# ...
ui:
image: tensorzero/ui
# ...
command: --config-file /app/config/tensorzero.toml
volumes:
- ./config:/app/config
# ...
```
Visit `/autopilot` in the self-hosted TensorZero UI to use Autopilot.

---

## 

**URL:** https://www.tensorzero.com/docs/deployment/tensorzero-gateway.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Deploy the TensorZero Gateway
> Learn how to deploy and customize the TensorZero Gateway.
The TensorZero Gateway is the core component that handles inference requests and collects observability data.
It's easy to get started with the TensorZero Gateway.
## Deploy
The gateway requires one of the following command line arguments:
* `--default-config`: Use default configuration settings.
* `--config-file path/to/tensorzero.toml`: Use a custom configuration file.
`--config-file` supports glob patterns, e.g. `--config-file
/path/to/**/*.toml`.
* `--run-postgres-migrations`: Run Postgres database migrations and exit.
* `--run-clickhouse-migrations`: Run ClickHouse database migrations and exit.
There are many ways to deploy the TensorZero Gateway.
Here are a few examples:
You can easily run the TensorZero Gateway locally using Docker.
If you don't have custom configuration, you can use:
```bash title="Running with Docker (default configuration)" theme={null}
docker run \
--env-file .env \
-p 3000:3000 \
tensorzero/gateway \
--default-config
```
If you have custom configuration, you can use:
```bash title="Running with Docker (custom configuration)" theme={null}
docker run \
-v "./config:/app/config" \
--env-file .env \
-p 3000:3000 \
tensorzero/gateway \
--config-file config/tensorzero.toml
```
You can run the TensorZero Gateway using Docker Compose:
```yaml theme={null}
services:
gateway:
image: tensorzero/gateway
volumes:
- ./config:/app/config:ro
command: --config-file /app/config/tensorzero.toml
env_file:
- ${ENV_FILE:-.env}
ports:
- "3000:3000"
restart: unless-stopped
extra_hosts:
- "host.docker.internal:host-gateway"
healthcheck:
test: wget --spider --tries 1 http://localhost:3000/status
interval: 15s
timeout: 1s
retries: 2
```
Make sure to create a `.env` file with the relevant environment variables (e.g. `TENSORZERO_POSTGRES_URL` and model provider API keys).
We provide a reference Helm chart in our [GitHub repository](https://github.com/tensorzero/tensorzero/tree/main/examples/production-deployment-k8s-helm).
You can use it to run TensorZero in Kubernetes.
The chart is available on [ArtifactHub](https://artifacthub.io/packages/helm/tensorzero/tensorzero).
You can build the TensorZero Gateway from source and run it directly on your host machine using [Cargo](https://doc.rust-lang.org/cargo/).
```bash title="Building from source" theme={null}
cargo run --profile performance --bin gateway -- --config-file path/to/your/tensorzero.toml
```
See the [optimizing latency and throughput](/deployment/optimize-latency-and-throughput/) guide to learn how to configure the gateway for high-performance deployments.
## Configure
### Set up model provider credentials
The TensorZero Gateway accepts the following environment variables for provider credentials.
Unless you specify an alternative credential location in your configuration file, these environment variables are required for the providers that are used in a variant that can be sampled.
If required credentials are missing, the gateway will fail on startup.
Unless customized in your configuration file, the following credentials are used by default:
| Provider                | Environment Variable(s)                                                                                   |
| ----------------------- | --------------------------------------------------------------------------------------------------------- |
| Anthropic               | `ANTHROPIC_API_KEY`                                                                                       |
| AWS Bedrock             | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (see [details](/integrations/model-providers/aws-bedrock))   |
| AWS SageMaker           | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (see [details](/integrations/model-providers/aws-sagemaker)) |
| Azure OpenAI            | `AZURE_API_KEY`                                                                                           |
| Fireworks               | `FIREWORKS_API_KEY`                                                                                       |
| GCP Vertex AI Anthropic | `GCP_VERTEX_CREDENTIALS_PATH` (see [details](/integrations/model-providers/gcp-vertex-ai-anthropic))      |
| GCP Vertex AI Gemini    | `GCP_VERTEX_CREDENTIALS_PATH` (see [details](/integrations/model-providers/gcp-vertex-ai-gemini))         |
| Google AI Studio Gemini | `GOOGLE_AI_STUDIO_GEMINI_API_KEY`                                                                         |
| Groq                    | `GROQ_API_KEY`                                                                                            |
| Hyperbolic              | `HYPERBOLIC_API_KEY`                                                                                      |
| Mistral                 | `MISTRAL_API_KEY`                                                                                         |
| OpenAI                  | `OPENAI_API_KEY`                                                                                          |
| OpenRouter              | `OPENROUTER_API_KEY`                                                                                      |
| Together                | `TOGETHER_API_KEY`                                                                                        |
| xAI                     | `XAI_API_KEY`                                                                                             |
### Set up custom configuration
Optionally, you can use a configuration file to customize the behavior of the gateway.
See [Configuration Reference](/gateway/configuration-reference) for more details.
TensorZero collects *pseudonymous* usage analytics to help our team improve the product.
The collected data includes *aggregated* metrics about TensorZero itself, but does NOT include your application's data.
To be explicit: TensorZero does NOT share any inference input or output.
TensorZero also does NOT share the name of any function, variant, metric, or similar application-specific identifiers.
See `howdy.rs` in the GitHub repository to see exactly what usage data is collected and shared with TensorZero.
To disable usage analytics, set the following configuration in the `tensorzero.toml` file:
```toml title="tensorzero.toml" theme={null}
[gateway]
disable_pseudonymous_usage_analytics = true
```
Alternatively, you can also set the environment variable `TENSORZERO_DISABLE_PSEUDONYMOUS_USAGE_ANALYTICS=1`.
### Set up observability
Optionally, the TensorZero Gateway can collect inference and feedback data for observability, optimization, evaluations, and experimentation.
TensorZero supports both Postgres and ClickHouse as observability backends.
Postgres is the simpler choice; we recommend ClickHouse if you're handling >100 inferences per second.
If you don't provide either of the environment variables below, observability will be disabled.
We recommend setting up observability early to monitor your LLM application and collect data for future optimization, but this can be done incrementally as needed.
#### Postgres
After [deploying Postgres](/deployment/postgres), you need to configure the `TENSORZERO_POSTGRES_URL` environment variable with the connection details.
#### ClickHouse
After [deploying ClickHouse](/deployment/clickhouse), you need to configure the `TENSORZERO_CLICKHOUSE_URL` environment variable with the connection details.
### Customize the bind address and port
By default, the gateway binds to `0.0.0.0:3000`.
You can customize the bind address and port in the configuration ([`gateway.bind_address`](/gateway/configuration-reference#bind_address)), with the `--bind-address` CLI flag, or with the `TENSORZERO_GATEWAY_BIND_ADDRESS` environment variable.
Only one of these methods can be used at a time.
```toml title="tensorzero.toml" theme={null}
[gateway]
bind_address = "0.0.0.0:3000"
```
```bash theme={null}
gateway --bind-address 0.0.0.0:3000 --config-file tensorzero.toml
```
```bash theme={null}
TENSORZERO_GATEWAY_BIND_ADDRESS=0.0.0.0:3000 gateway --config-file tensorzero.toml
```
### Customize the logging format
Optionally, you can provide the following command line argument to customize the gateway's logging format:
* `--log-format`: Set the logging format to either `pretty` (default) or `json`.
### Add a status or health check
The TensorZero Gateway exposes endpoints for status and health checks.
The `/status` endpoint checks that the gateway is running successfully.
```json title="GET /status" theme={null}
{ "status": "ok" }
```
The `/health` endpoint additionally checks that it can communicate with dependencies like Postgres or ClickHouse (if enabled).
```json title="GET /health" theme={null}
{ "gateway": "ok", "postgres": "ok" }
```

---

## 

**URL:** https://www.tensorzero.com/docs/deployment/tensorzero-ui.md

> ## Documentation Index
> Fetch the complete documentation index at: https://www.tensorzero.com/docs/llms.txt
> Use this file to discover all available pages before exploring further.
# Deploy the TensorZero UI
> Learn how to deploy and customize the TensorZero UI.
The TensorZero UI is a self-hosted web application that streamlines the use of TensorZero with features like observability and optimization.
It's easy to get started with the TensorZero UI.
## Deploy
[Deploy the TensorZero Gateway](/deployment/tensorzero-gateway/) and configure `TENSORZERO_GATEWAY_URL`.
For example, if the gateway is running locally, you can set `TENSORZERO_GATEWAY_URL=http://localhost:3000`.
The TensorZero UI is available on Docker Hub as `tensorzero/ui`.
You can easily run the TensorZero UI using Docker Compose:
```yaml theme={null}
services:
ui:
image: tensorzero/ui
# Add your environment variables the .env file (make sure it contains `TENSORZERO_GATEWAY_URL`)
env_file:
- ${ENV_FILE:-.env}
# Publish the UI to port 4000
ports:
- "4000:4000"
restart: unless-stopped
```
Make sure to create a `.env` file with the relevant environment variables.
For more details, see the example `docker-compose.yml` file in the [GitHub repository](https://github.com/tensorzero/tensorzero/blob/main/ui/docker-compose.yml).
Alternatively, you can launch the UI directly with the following command:
```bash theme={null}
docker run \
--volume ./config:/app/config:ro \
--env-file ./.env \
--publish 4000:4000 \
tensorzero/ui
```
Make sure to create a `.env` file with the relevant environment variables.
We provide a reference Helm chart in our [GitHub repository](https://github.com/tensorzero/tensorzero/tree/main/examples/production-deployment-k8s-helm).
You can use it to run TensorZero in Kubernetes.
The chart is available on [ArtifactHub](https://artifacthub.io/packages/helm/tensorzero/tensorzero).
Alternatively, you can build the UI from source.
See our [GitHub repository](https://github.com/tensorzero/tensorzero/blob/main/ui/) for more details.
## Configure
### Add a health check
The TensorZero UI exposes an endpoint for health checks.
This `/health` endpoint checks that the UI is running, the associated configuration is valid, and the gateway connection is healthy.
### Customize the deployment
The TensorZero UI supports the following optional environment variables.
For certain uncommon scenarios (e.g. IPv6), you can also customize `HOST` inside the UI container.
See the Vite documentation for more details.
Set the environment variable `TENSORZERO_UI_LOG_LEVEL` to control log verbosity.
The allowed values are `debug`, `info` (default), `warn`, and `error`.
