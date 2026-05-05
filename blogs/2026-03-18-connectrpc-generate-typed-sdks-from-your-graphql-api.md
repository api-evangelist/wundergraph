---
title: "ConnectRPC: Generate Typed SDKs from Your GraphQL API"
url: "https://wundergraph.com/blog/connectrpc-generate-typed-sdks-from-graphql"
date: "2026-03-18T00:00:00.000Z"
author: "Ahmet Soormally"
feed_url: "https://wundergraph.com/atom.xml"
---
<article><p>ConnectRPC is a protocol translation layer in the Cosmo Router. Write named GraphQL operations against your federated schema, run one CLI command, and get Protocol Buffer definitions, typed SDKs (Go, TypeScript, Swift, Kotlin, etc.), and an OpenAPI spec. The router serves them as gRPC, gRPC-Web, Connect protocol, and HTTP/JSON alongside your GraphQL endpoint. No separate API layers to build or maintain. No schema changes required. Operations are the API contract; everything else is derived.</p><h2>The Multi-Protocol Problem: Why GraphQL APIs Are Hard to Expose as REST or gRPC</h2><p>You've built a federated GraphQL API. Your internal teams use it daily. The schema is well-governed, the subgraphs are cleanly separated, and the router handles composition and execution.</p><p>Then the requests start coming in. An external partner wants REST with an OpenAPI spec. A mobile team wants typed SDKs with compile-time safety. A backend service wants <a href="https://grpc.io/">gRPC</a> with binary encoding. Some consumers have hard platform constraints — a Roku box or embedded device can't run a GraphQL client library, needs minimal payload sizes, and may only support simple HTTP/JSON or binary protocols. These are platform limitations.</p><p>Today, you maintain separate API layers for each of these consumers. Separate codebases, separate contracts, separate deployment pipelines. Each one drifts from the others. The <a href="https://wundergraph.com/blog/platform-engineering-team-bottleneck">platform team becomes a bottleneck</a> because every new consumption pattern requires a new integration layer.</p><p>AI coding assistants are making this worse, not better. It's never been easier to scaffold a new BFF, but every generated service needs to be reviewed, tested, deployed, secured, and maintained. Coding assistants accelerate BFF sprawl; they don't solve the underlying problem.</p><p>There's no centralized control over what operations are exposed through which protocols, making the governance problem just as real. There is no consistent contract enforcement across consumption channels and no single source of truth that a platform engineer can point to and say, &quot;This is the API surface.&quot;</p><h2>Protocol Choice Is a Deployment Concern, Not an Implementation Problem</h2><p>Your GraphQL schema already defines the data model, and your named operations define the access patterns. The missing piece is an intermediate representation that can target multiple protocols from that same source of truth.</p><p>Protocol Buffers are that intermediate representation, and they solve some of the hard problems: stable field numbering for wire compatibility, cross-language code generation through the Buf ecosystem, binary encoding for performance, and idempotency annotations for HTTP caching. While GraphQL solves the other half: schema governance, federation composition, and operation-level access control.</p><p>ConnectRPC bridges the two. GraphQL is the design layer where you model your data and curate your API surface, and Protocol Buffers are the compilation target that unlocks multi-protocol serving and typed SDK generation. The router handles the translation at runtime.</p><h2>No More BFF Sprawl: Why Everything Downstream Is Generated</h2><p>Everything downstream of the operations is deterministic. The same operations against the same schema always produce the same proto, the same SDKs, the same OpenAPI spec. There's no hand-written glue code, no one-off adapters, nothing to maintain. The only creative work is deciding what data to expose, in what shape, for which consumer.</p><p>That's the design problem.</p><p>This changes how you use AI coding assistants day-to-day. Instead of generating one-off BFF services that accelerate sprawl, you point them at the actual design problem: &quot;Here's my schema. I need an API surface for a mobile checkout flow — what operations should I expose?&quot; The output is a handful of <code>.graphql</code> files, not a codebase. The agent helps you think consumer-first about your API surface; the deterministic toolchain handles everything from there.</p><p>That’s the difference between using AI to generate more code and using it to design an abstraction so there’s less code to maintain.</p><h2>Introducing Cosmo ConnectRPC</h2><p>ConnectRPC is a protocol translation layer built into the Cosmo Router. It compiles GraphQL operations to Protocol Buffer definitions and serves them as multi-protocol APIs — gRPC, REST/HTTP, and typed client SDKs — turning what would be a multi-team, multi-month integration effort into a build-time generation step.</p><p>ConnectRPC is available today in Cosmo Router v0.283.0 and on Cosmo Cloud. It works with any Cosmo-managed federated graph, no schema changes or subgraph modifications required.</p><h2>How ConnectRPC Turns GraphQL Operations into gRPC, REST, and Typed SDKs</h2><p>There are two halves to this: a build-time compilation pipeline and a runtime serving layer.</p><p>At build time, you compile named GraphQL operations into Protocol Buffer definitions. The Buf toolchain then generates typed SDKs and OpenAPI specs from those protos.</p><pre>graph TD
    A[&quot;.graphql operations&quot;] --&gt; B[&quot;wgc grpc-service generate&quot;]
    B --&gt; C[&quot;service.proto&quot;]
    B --&gt; D[&quot;service.proto.lock.json&quot;]
    C --&gt; E[&quot;buf generate&quot;]
    E --&gt; F[&quot;Go SDK&quot;]
    E --&gt; G[&quot;TypeScript SDK&quot;]
    E --&gt; H[&quot;OpenAPI spec&quot;]
</pre><p>At runtime, the Cosmo Router serves a ConnectRPC endpoint alongside your GraphQL endpoint. It accepts requests over gRPC, gRPC-Web, <a href="https://connectrpc.com/">Connect protocol</a>, and plain HTTP/JSON — all resolving to the same underlying GraphQL execution. The router maps proto messages to GraphQL variables, executes the operation against your federated graph, and returns the response in whatever wire format the client requested.</p><pre>sequenceDiagram
    participant Client as Client (gRPC / REST / Connect)
    participant Router as Cosmo Router
    participant Subgraphs as Federated Subgraphs

    Client-&gt;&gt;Router: Request (proto message)
    Router-&gt;&gt;Router: Proto → GraphQL variable mapping
    Router-&gt;&gt;Subgraphs: GraphQL execution
    Subgraphs--&gt;&gt;Router: GraphQL response
    Router-&gt;&gt;Router: GraphQL → Proto response mapping
    Router--&gt;&gt;Client: Response (wire format)
</pre><h2>How the GraphQL-to-Proto Compilation Pipeline Works</h2><h3>Operations as API Contracts</h3><p>API surfaces are defined as named GraphQL operations. There is one per file, each version-controlled as a persisted operation (also known as named operations or trusted documents). This is API design: each operation is an intentional choice about what data to expose, in what shape, and for which consumers. Consumers can only call these curated operations; there's no way to construct arbitrary queries or discover the underlying schema.</p><pre># GetEmployeeById.graphql
query GetEmployeeById($id: Int!) {
  employee(id: $id) {
    id
    tag
    details {
      forename
      surname
    }
    currentMood
  }
}
</pre><pre># UpdateEmployeeMood.graphql
mutation UpdateEmployeeMood($id: Int!, $mood: Mood!) {
  updateEmployeeMood(employeeId: $id, mood: $mood) {
    id
    currentMood
  }
}
</pre><p>Each file contains a single named operation. The operation name becomes the RPC method name. The operation variables become the request message fields. The selection set becomes the response message.</p><p>You're not limited to a single service. Multiple ConnectRPC services can be defined from the same federated graph, with each exposing a different domain or capability. A <code>ProductService</code>, <code>OrderService</code>, and <code>RecommendationService</code> can each have their own operations, proto package, and generated SDKs — all backed by the same federated graph. Alternatively, you can organize services per consumer: one for a mobile app, another for an external partner, each with its own curated API surface.</p><h3>From Operations to Proto Definitions</h3><p>A single CLI command compiles operations and schema into a Protocol Buffer service definition.</p><pre>wgc grpc-service generate \
  --with-operations ./services \
  --input ./schema.graphqls \
  --output ./services \
  --package-name employees.v1 \
  HRService
</pre><p>This produces <code>service.proto</code> and <code>service.proto.lock.json</code>. The generated proto for the operations above:</p><pre>syntax = &quot;proto3&quot;;
package employees.v1;

import &quot;google/protobuf/wrappers.proto&quot;;

service HRService {
  rpc GetEmployeeById(GetEmployeeByIdRequest) returns (GetEmployeeByIdResponse) {
    option idempotency_level = NO_SIDE_EFFECTS;
  }
  rpc UpdateEmployeeMood(UpdateEmployeeMoodRequest) returns (UpdateEmployeeMoodResponse) {}
}

message GetEmployeeByIdRequest {
  int32 id = 1;
}

message GetEmployeeByIdResponse {
  message Employee {
    int32 id = 1;
    google.protobuf.StringValue tag = 2;
    message Details {
      string forename = 1;
      string surname = 2;
    }
    Details details = 3;
    Mood current_mood = 4;
  }
  Employee employee = 1;
}

message UpdateEmployeeMoodRequest {
  int32 id = 1;
  Mood mood = 2;
}

message UpdateEmployeeMoodResponse {
  message UpdateEmployeeMood {
    int32 id = 1;
    Mood current_mood = 2;
  }
  UpdateEmployeeMood update_employee_mood = 1;
}

enum Mood {
  MOOD_UNSPECIFIED = 0;
  MOOD_HAPPY = 1;
  MOOD_SAD = 2;
}
</pre><p>The mapping is deterministic and follows <a href="https://protobuf.dev/programming-guides/proto3/">proto3</a> conventions.</p><ul><li>GraphQL types become proto messages</li><li>Fields get snake_case naming</li><li>Nullable scalars use wrapper types, so nullability semantics are preserved</li><li>Enums get a type prefix and an <code>_UNSPECIFIED</code> zero value.</li></ul><p>The result is a proto that feels idiomatic to gRPC engineers.</p><p>Query operations are annotated with <code>NO_SIDE_EFFECTS</code>, which tells the Connect protocol to serve them via HTTP GET, ensuring they are cacheable by CDNs and reverse proxies.</p><p>The lock file tracks field number assignments, so regenerating after schema changes never breaks wire compatibility with existing clients.</p><p>API evolution is where most multi-protocol systems break. Proto field numbers are part of the wire format — if they shift, deployed clients silently receive wrong data. The lock file makes this a non-issue: commit it alongside your proto, and field numbers stay stable no matter how your operations evolve.</p><p>The generation step validates operations against the current schema — if an operation references a removed or renamed field, generation fails with a specific error before any proto is produced. The lock file detects breaking changes to field number assignments and surfaces them as warnings rather than silently reordering. Problems are caught at build time, in CI, before anything reaches production.</p><h3>Router Configuration</h3><p>Enabling ConnectRPC in the Cosmo Router is a configuration change. You point it at your services directory, and it will serve a ConnectRPC endpoint alongside your existing GraphQL endpoint. Both listeners run inside the same router process. When operations are updated and protos regenerated, the router picks up changes atomically.</p><p>See the <a href="https://cosmo-docs.wundergraph.com/router/configuration">configuration reference</a> for the full setup.</p><h3>SDK Generation via Buf</h3><p>With a proto in hand, the <a href="https://buf.build/">Buf toolchain</a> generates typed clients for any language.</p><pre># buf.gen.yaml
version: v2
managed:
  enabled: true
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen/go
  - remote: buf.build/connectrpc/go
    out: gen/go
  - remote: buf.build/bufbuild/es
    out: gen/ts
  - remote: buf.build/connectrpc/es
    out: gen/ts
  - local: protoc-gen-connect-openapi
    out: gen/openapi
</pre><pre>buf generate ./services
</pre><p>One command. Go client, TypeScript client, and OpenAPI spec — all generated from the same proto.</p><p>In practice, most teams run <code>wgc grpc-service generate</code> and <code>buf generate</code> as CI steps. The <code>.graphql</code> operation files and the lock file are committed to version control. The generated proto and SDKs are build artifacts. The generation is deterministic, so the output is reproducible across environments.</p><h3>What Consumers See</h3><p>Consumers pick their protocol. The router handles the rest.</p><p><strong>TypeScript SDK:</strong></p><pre>import { createClient } from &quot;@connectrpc/connect&quot;;
import { createConnectTransport } from &quot;@connectrpc/connect-web&quot;;
import { HrService } from &quot;@my-org/sdk/employees/v1/service_connect&quot;;

const transport = createConnectTransport({
  baseUrl: &quot;http://localhost:5026&quot;,
});

const client = createClient(HrService, transport);
const response = await client.getEmployeeById({ id: 1 });
</pre><p><strong>Go SDK:</strong></p><pre>client := employeesv1connect.NewHrServiceClient(
    http.DefaultClient,
    &quot;http://localhost:5026&quot;,
)
req := connect.NewRequest(&amp;employeesv1.GetEmployeeByIdRequest{Id: 1})
res, err := client.GetEmployeeById(context.Background(), req)
</pre><p><strong>HTTP/JSON (no SDK required):</strong></p><pre>curl -X POST http://localhost:5026/employees.v1.HrService/GetEmployeeById \
  -H &quot;Content-Type: application/json&quot; \
  -H &quot;Connect-Protocol-Version: 1&quot; \
  -d '{}'
</pre><p><strong>gRPC:</strong></p><pre>grpcurl -plaintext \
  -proto ./services/service.proto \
  -d '{&quot;id&quot;: 1}' \
  localhost:5026 \
  employees.v1.HrService/GetEmployeeById
</pre><p>The examples above show Go, TypeScript, HTTP/JSON, and gRPC, but the Buf ecosystem supports any language with a protoc or Connect plugin. Add the plugin to your <code>buf.gen.yaml</code> and you have another typed client. The <a href="https://github.com/wundergraph/connectrpc-tutorial">tutorial repository</a> walks through several of these, including HTTP GET for cacheable queries.</p><h2>Why ConnectRPC Replaces Separate API Layers</h2><p><strong>Persisted operations as the API design surface.</strong> ConnectRPC treats named GraphQL operations as the API contract — not the schema, not an auto-generated CRUD layer. Each operation is an intentional design decision: which data to expose, in what shape, for which consumer. This is both a security property and a design discipline. The platform team controls the API surface the same way a library author controls a public API: by choosing what to export.</p><p><strong>Lock file for backward-compatible evolution.</strong> The <code>service.proto.lock.json</code> file tracks field number assignments across regenerations. When you modify an operation's selection set, existing field numbers are preserved and removed fields have their numbers reserved. Deployed clients keep working.</p><p><strong>Idempotency annotations for HTTP caching.</strong> Query operations automatically get <code>idempotency_level = NO_SIDE_EFFECTS</code> in the proto definition. This tells the Connect protocol to use HTTP GET for these methods, making them cacheable by CDNs and HTTP caches without any additional configuration.</p><p><strong>Atomic hot reload.</strong> The router watches the services directory and picks up proto and operation changes without restarts. Update an operation, regenerate the proto, drop the files in, and the router switches over atomically.</p><p><strong>One source of truth.</strong> Your GraphQL schema is the data model. Your named operations are the contracts. The proto is derived, the SDKs are derived, and the OpenAPI spec is derived. There is no drift because there is nothing to drift from.</p><p>Here's what changes concretely:</p><table><thead><tr><th></th><th>Without ConnectRPC</th><th>With ConnectRPC</th></tr></thead><tbody><tr><td>Protocol surfaces</td><td>Separate codebase per protocol</td><td>One set of operations, multiple outputs</td></tr><tr><td>Contract drift</td><td>Manual synchronization across layers</td><td>Derived from single source — impossible to drift</td></tr><tr><td>API evolution</td><td>Coordinate changes across REST, gRPC, and GraphQL layers</td><td>Update operations, regenerate, lock file preserves compatibility</td></tr><tr><td>Team bottleneck</td><td>Platform team builds each adapter</td><td>Platform team curates operations; SDKs are generated</td></tr><tr><td>Security surface</td><td>Each layer has its own auth and validation</td><td>One execution path, one authorization model</td></tr></tbody></table><h2>Use Cases: Partner APIs, Mobile SDKs, and Platform Governance</h2><h3>External partner API</h3><p>Your platform team manages a federated GraphQL API internally. External partners need REST with an OpenAPI spec and versioning guarantees. With ConnectRPC, you author a set of operations that define the partner API surface, generate the proto, and hand partners an OpenAPI spec and HTTP endpoints. The partner never sees GraphQL. The platform team never maintains a separate REST layer. When the API evolves, you update the operations, regenerate, and the lock file ensures backward compatibility.</p><h3>Mobile team with typed SDKs</h3><p>Your mobile engineers want compile-time type safety and binary encoding for bandwidth-sensitive environments. They don't want to adopt a GraphQL client library. With ConnectRPC, you generate a Swift or Kotlin SDK from the proto (via Buf plugins) and hand them a typed client that talks binary protobuf over HTTP/2. Request and response types are generated, autocomplete works, and the mobile team never writes a GraphQL query.</p><h3>Platform governance across teams</h3><p>Your organization runs 50 microservices behind a federated graph. Each domain team owns its subgraphs, but the platform team controls what's exposed externally. With ConnectRPC, the platform team authors and version-controls the operation files that define the external API surface. Domain teams evolve their subgraphs freely; the exposed API only changes when the platform team updates the operations and regenerates. There's one place to audit what's exposed, one lock file that enforces backward compatibility, and one CI pipeline that validates the entire external contract. Individual teams don't need to know ConnectRPC exists; they just maintain their subgraphs.</p><h2>Performance Characteristics of ConnectRPC</h2><p>ConnectRPC runs on Protocol Buffers and the Connect protocol — both have well-understood performance characteristics.</p><p><strong>Binary encoding.</strong> Protobuf payloads are smaller than equivalent JSON and faster to serialize/deserialize — typically 2-5x smaller on the wire with lower serialization overhead. For high-throughput services or bandwidth-constrained mobile clients, the difference is significant.</p><p><strong>HTTP GET for queries.</strong> Because query operations are annotated with <code>NO_SIDE_EFFECTS</code>, the Connect protocol serves them via HTTP GET. This makes query responses cacheable by CDNs, reverse proxies, and browser caches — with no application-level caching logic.</p><p><strong>In-process transcoding.</strong> Protocol translation happens inside the router process — no external sidecar, no additional network hop. This was a deliberate design choice to avoid the latency and operational complexity of a separate transcoding service.</p><p><strong>No runtime reflection.</strong> The proto definitions are compiled and loaded at startup. The router does not use runtime reflection to resolve proto types or field mappings.</p><h2>Design Principle: Protocol Translation Belongs in Infrastructure</h2><p>ConnectRPC is built on a specific belief: protocol translation belongs in infrastructure, not in application code. Teams shouldn't build and maintain separate API layers to serve the same data over different protocols. The platform should handle that.</p><p>This is the same separation of concerns that made API gateways successful for REST. ConnectRPC applies it to multi-protocol serving: GraphQL governs the schema and operations, Protocol Buffers provide the compilation target, and the router handles everything in between. Individual teams pick the protocol that fits their consumers, and the platform ensures consistency.</p><p>Standalone transcoding proxies require you to author and maintain proto definitions manually, separate from your API logic. ConnectRPC derives protos from your existing GraphQL operations. The operations <em>are</em> the contract, and the proto is a compilation artifact. There's nothing to keep in sync because there's only one source.</p><p>Not every consumer wants or can adopt GraphQL. Some have platform constraints, such as streaming devices, embedded systems, or partner integrations locked to REST. ConnectRPC meets them where they are without asking anyone to change their stack.</p><p>The generated artifacts are standard Protocol Buffers and standard ConnectRPC/gRPC clients. Nothing in the output is proprietary to Cosmo. If you move away from the Cosmo Router, the protos, SDKs, and OpenAPI specs remain valid and usable, because they're standard artifacts produced by the Buf toolchain.</p><h2>What's on the ConnectRPC Roadmap</h2><p>We're exploring several directions based on early adopter feedback:</p><ul><li><strong>Streaming support.</strong> Map GraphQL subscriptions to server-streaming RPCs for real-time data over gRPC and Connect.</li><li><strong>Per-method authorization.</strong> Apply fine-grained access control at the RPC method level, building on the same <code>@requiresScopes</code> directives used in your GraphQL schema.</li><li><strong>Per-method rate limiting.</strong> Apply rate limits at the RPC method level, with policies defined centrally in the router configuration.</li><li><strong>Observability integration.</strong> Surface per-RPC-method metrics, traces, and access logs in Cosmo Studio alongside existing GraphQL analytics.</li><li><strong>API reference documentation.</strong> Auto-generate human-readable API reference sites from the proto definitions — method descriptions, type tables, and usage examples — so consumers get documentation alongside their SDKs.</li><li><strong>Proto registry integration.</strong> Publish generated protos to a Buf Schema Registry or internal registry as part of the generation pipeline.</li></ul><p>If there's a capability you'd like to see, <a href="https://github.com/wundergraph/cosmo/issues">open an issue</a> or reach out on <a href="https://discord.gg/QRf4dhJp">Discord</a>.</p><h2>Get Started with ConnectRPC</h2><p>ConnectRPC is available now. The documentation covers configuration, CLI reference, and SDK generation in detail.</p><ul><li><a href="https://cosmo-docs.wundergraph.com/connect-rpc/overview">ConnectRPC documentation</a></li><li><a href="https://github.com/wundergraph/connectrpc-tutorial">Tutorial repository</a></li><li><a href="https://cosmo-docs.wundergraph.com/router/configuration">Cosmo Router configuration reference</a></li><li><a href="https://cosmo-docs.wundergraph.com/cli/grpc-service/generate"><code>wgc grpc-service generate</code> CLI reference</a></li></ul><hr /><hr /></article>
