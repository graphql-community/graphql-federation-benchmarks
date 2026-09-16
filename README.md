# GraphQL Federation Benchmarks

We compare gateway efficiency under controlled workloads for **GraphQL Federation** and **Apollo Federation**. The aim is to help users understand how gateways perform with their subgraph stack: **Hot Chocolate (.NET), Spring Boot (Java), or Apollo Server (Node.js)**.

Every core workload runs against all three stacks, using equivalent data and operations. Results are shown separately for each stack.

## What we test

- **Subgraph latency:** add 0, 10, 50, 100 or 200 ms of delay to see how gateways handle slower dependencies.
- **Response size:** vary the data returned to measure parsing, merging, and serialization costs.
- **Query structure:** exercise parallel fetches, dependent fetches, entity batches, and conditional fields.
- **Planning:** compare repeated queries with varied operation sets and cold versus warm plan caches.
- **Deduplication:** measure the cost and benefit of sharing work between requests in separate runs.

Incremental delivery and failure recovery get separate tests with measures suited to those features.

## How we keep comparisons fair

Benchmarks run on dedicated bare-metal hardware, with the same CPU and memory budget for each gateway. Versions, settings, and datasets are fixed and published. We check that gateways return equivalent results and record their subgraph requests to explain differences in the work performed.

Subgraphs use the actual frameworks and have enough capacity to expose gateway differences. We monitor both subgraphs and the load generator to detect bottlenecks outside the gateway. Each request carries a distinct Authorization header, which the gateway forwards to the subgraphs.

## What users see

For each workload, we report throughput, latency, errors, gateway CPU usage, and memory. We compare latency at the same request rate and measure how much traffic each gateway can sustain within a stated latency target. Requests arrive independently of response time.

Runs include warmup and repeated measurements. We publish variability and raw results, including failed or unsupported cases. There is no single overall score.

## Categories

Each category focuses on one aspect of gateway performance. We vary that aspect while keeping the other conditions stable. Each case uses a selected combination of query structure, payload size, and subgraph latency.

### Subgraph latency

Subgraphs take time to respond. This category measures how efficiently gateways handle that waiting time: scheduling independent fetches, following dependencies, and managing more requests in flight as responses get slower.

We run the same query with **0, 10, 50, 100, and 200 ms of added delay** on every subgraph. Data and response sizes stay the same. The 0 ms case has no artificial delay; normal execution and network time still apply.

We use a deeply nested query across all latency tiers. Increasing subgraph latency exposes how efficiently gateways plan and coordinate fetches. We record subgraph request counts and execution plans alongside response latency.

An additional case gives **one subgraph a 500 ms delay and the others 10 ms**. This shows whether independent work proceeds while the slow subgraph is still responding.

Each subgraph adds a fixed, asynchronous delay once per HTTP request, including a request containing multiple operations. We measure the actual delay and count both HTTP requests and operations so batching remains visible.

### Response size

Larger responses give the gateway more data to parse, merge, and serialize.

We execute the same nested query while varying the response size. We test responses around **2 KB, 100 KB, 1 MB, and 8 MB**.

### Query structure

The structure of a query determines which subgraph fetches can run in parallel, which must wait, and which can be grouped together.

- **Parallel fetches:** increase the number of independent roots served by different subgraphs.
- **Dependent fetches:** increase the number of sequential steps needed to answer a query.
- **Entity batches:** request related fields for increasing numbers of entities, using both distinct and repeated entity keys.
- **Conditional fields:** vary `@skip` and `@include` values and check that omitted branches cause no unnecessary fetches.

We compare response times, the number of subgraph fetches, and query plan efficiency in these tests.

### Planning and plan caching

Creating an execution plan takes time. Reusing cached plans saves that work, but a varied workload can fill the cache and force gateways to plan operations again.

We compare a repeated operation, a small set of operations that fits in the plan cache, and a larger set that puts pressure on it. We include simple queries and queries with more planning choices.

We measure cold and warm cache behavior separately, documenting cache settings and how each state is established. Where gateways expose planning time, we report it alongside total request latency, CPU usage, and memory. Subgraph responses stay small to make planning costs easier to see.

### Deduplication

Concurrent requests can need identical data from a subgraph. Deduplication lets them share a fetch, reducing subgraph calls at the cost of tracking and coordinating that work.

We compare requests with distinct Authorization headers against requests sharing the same header, query, and variables. The shared-header case deliberately allows matching subgraph requests to be combined. It is separate from the normal per-request identity used elsewhere.

### Incremental delivery

Deferred fragments let the gateway send initial data while it continues fetching the rest of the response.

We compare the same query with and without `@defer`. A fast branch supplies the initial data, while a slower branch requires additional subgraph work.

We measure two timings from the start of the request:

- **Time to initial payload:** until the client has received the complete first payload.
- **Time to complete response:** until the client has received all deferred data and the response is complete.

These measurements show how much sooner the client gets the initial data and whether deferring changes total completion time. We verify that the assembled result matches the query without `@defer`. Unsupported features are shown explicitly.

### Failure recovery

Subgraphs can return errors, respond too slowly, or go offline. Gateways need to handle these failures and recover when the subgraph becomes available again.

We introduce subgraph errors, responses that exceed a configured timeout, and a temporary subgraph outage. Unaffected branches continue to serve data.

We use repeatable fault schedules and publish timeout and retry settings. We measure client errors, partial results, recovery time, memory, and additional subgraph calls caused by retries. Expected failures are evaluated against the case's expected behavior rather than treated as successful throughput.
