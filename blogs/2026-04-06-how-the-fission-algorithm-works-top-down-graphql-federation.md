---
title: "How the Fission Algorithm Works: Top-Down GraphQL Federation Design"
url: "https://wundergraph.com/blog/fission-algorithm-deep-dive"
date: "2026-04-06T00:00:00.000Z"
author: "Jens Neuse"
feed_url: "https://wundergraph.com/atom.xml"
---
<article><p>Fission inverts GraphQL Federation. Instead of composing subgraphs into a supergraph, you design the supergraph first and Fission decomposes it into subgraph specs, handling entity keys, directives, and validation automatically.</p><p>In a <a href="https://wundergraph.com/blog/graphql-federation-was-built-backwards">previous post</a>, I argued that Federation's composition model is backwards. It builds the supergraph from the bottom up when it should be designed from the top down.</p><p>This post is a technical deep dive into how the Fission algorithm actually does that.</p><p>How are teams governing schema changes, handling production traffic, and measuring Federation success? Share your experience and get early access to the full report. For every valid survey completed, we'll donate $30 to <a href="https://www.unicef.org/">UNICEF</a>.</p><h2>The Problem Fission Solves</h2><p>To understand Fission, first you need to understand what makes schema changes expensive in Federation.</p><p>Consider a federated graph with three subgraphs:</p><pre># Subgraph: Users
type User @key(fields: &quot;id&quot;) {
  id: ID!
  name: String!
  email: String!
}

type Query {
  user(id: ID!): User
  users: [User!]!
}
</pre><pre># Subgraph: Orders
type User @key(fields: &quot;id&quot;) {
  id: ID!
  orders: [Order!]!
}

type Order @key(fields: &quot;id&quot;) {
  id: ID!
  total: Float!
  status: OrderStatus!
  createdAt: DateTime!
}

enum OrderStatus {
  PENDING
  SHIPPED
  DELIVERED
}
</pre><pre># Subgraph: Products
type Order @key(fields: &quot;id&quot;) {
  id: ID!
  items: [OrderItem!]!
}

type OrderItem {
  product: Product!
  quantity: Int!
}

type Product @key(fields: &quot;id&quot;) {
  id: ID!
  name: String!
  price: Float!
}
</pre><p>Composition produces a supergraph where you can query:</p><pre>query {
  user(id: &quot;1&quot;) {
    name
    orders {
      total
      status
      items {
        product {
          name
          price
        }
      }
    }
  }
}
</pre><p>Now a frontend team wants to add shipping tracking. This is their <a href="https://wundergraph.com/blog/dream-query-design-apis-from-consumer-out">dream query</a>:</p><pre>query {
  user(id: &quot;1&quot;) {
    orders {
      shipping {
        carrier
        trackingNumber
        estimatedDelivery
        currentLocation
      }
    }
  }
}
</pre><p>In the traditional workflow, the frontend team has to figure out:</p><ul><li><code>shipping</code> should be a field on <code>Order</code>, but which subgraph owns <code>Order</code>?</li><li>Both the Orders subgraph and the Products subgraph define <code>Order</code> as an entity.</li><li><code>carrier</code> and <code>trackingNumber</code> come from the shipping provider.</li><li><code>estimatedDelivery</code> might come from logistics.</li><li><code>currentLocation</code> requires real-time tracking data.</li><li>Should this be a new subgraph? An extension of Orders? Both?</li></ul><p>The frontend team can't answer these questions, so the platform team needs to be brought in. Meetings are scheduled and the process begins.</p><p>Fission automates the hardest parts of this process.</p><h2>What Fission Actually Is</h2><p>Before we walk through an example, it's important to understand what Fission is and what it isn't.</p><p>Fission is not just the opposite of composition, a batch algorithm that takes a supergraph and splits it into subgraphs. It's a <strong>stateful schema graph engine</strong>, an in-memory system that maintains two synchronized views of your federated graph at all times:</p><ol><li><strong>The supergraph view</strong> — the unified, consumer-facing schema. This is the canonical source of truth for what the API looks like.</li><li><strong>The subgraph views</strong> — one per service, representing what each team is responsible for implementing.</li></ol><p>These two views are kept in sync. When you make a change to either one, Fission will automatically propagate the consequences: renaming a type cascades to every subgraph and every field reference. Adding a type to a subgraph pulls in all its transitive dependencies. Entity <code>@key</code> directives propagate automatically. <code>@shareable</code> directives are added bidirectionally when fields exist in multiple subgraphs.</p><p>Every edit follows the same pattern: <strong>validate first, mutate second, update indexes, and return a structured result.</strong> If validation fails, no state changes happen. This ensures you never have a half-applied edit.</p><p>The key insight is the synchronization contract:</p><blockquote><p>The supergraph view represents the union of all subgraph views, plus any types and fields not yet assigned to a subgraph.</p></blockquote><p>This means the architect can design the complete API shape first and then decide how to distribute the implementation across subgraphs. The supergraph doesn't have to be assembled from the bottom up. It can be designed from the top down.</p><h2>How It Works in Practice</h2><p>Let's walk through the shipping example step by step.</p><h3>Step 1: Design on the Supergraph</h3><p>The architect adds the new types and fields directly on the supergraph:</p><pre># New additions to the supergraph
type Order @key(fields: &quot;id&quot;) {
  id: ID!
  shipping: ShippingInfo!   # new field
}

type ShippingInfo {          # new type
  carrier: String!
  trackingNumber: String!
  estimatedDelivery: DateTime!
  currentLocation: String
}
</pre><p>At this point, <code>ShippingInfo</code> exists in the supergraph but isn't assigned to any subgraph yet. The <code>shipping</code> field on <code>Order</code> is defined but no team is responsible for implementing it.</p><p>The supergraph is the API you want.</p><h3>Step 2: Assign to Subgraphs</h3><p>The architect assigns the new type to an existing subgraph, or if necessary, creates a new one.</p><p>Let's say they assign <code>ShippingInfo</code> and the <code>shipping</code> field on <code>Order</code> to a new &quot;Shipping&quot; subgraph.</p><p>When this happens, Fission automatically does several things:</p><p><strong>Transitive dependency propagation:</strong> <code>ShippingInfo</code> references <code>String</code>, <code>DateTime</code>, and <code>Boolean</code>. These are all scalars that are already available. But if <code>ShippingInfo</code> had a field like <code>warehouse: Warehouse!</code>, and the <code>Warehouse</code> type doesn't exist in the Shipping subgraph yet, Fission will automatically pull it in, along with anything <code>Warehouse</code> references. This propagation is cycle-safe; mutual type references don't cause infinite loops.</p><p><strong>Entity key propagation:</strong> <code>Order</code> is an entity with <code>@key(fields: &quot;id&quot;)</code> in the Orders and Products subgraphs. When the Shipping subgraph extends <code>Order</code>, Fission automatically adds <code>@key(fields: &quot;id&quot;)</code> to the Shipping subgraph's <code>Order</code> definition. The entity remains resolvable across subgraph boundaries.</p><p><strong>Shareable propagation:</strong> If the <code>shipping</code> field ends up defined in multiple V2 subgraphs, <code>@shareable</code> is added bidirectionally to <em>all</em> subgraphs containing that field, not just the new one. This ensures the rendered SDL is always valid for composition.</p><h3>Step 3: Continuous Validation</h3><p>Every edit is validated before it's applied.</p><p>Fission enforces a comprehensive set of invariants:</p><ul><li><strong>Reference integrity:</strong> you cannot remove a type that's still referenced by other fields. The error includes the exact coordinates that still reference it.</li><li><strong>Last-child protection:</strong> you cannot remove the last field from a type that's still referenced elsewhere.</li><li><strong>Key field validity:</strong> you cannot rename or change the type of a field that participates in a <code>@key</code> field set. The key selection tree would break.</li><li><strong>Type-kind compatibility:</strong> you cannot have <code>User</code> as an Object in one subgraph and an Interface in another.</li><li><strong>Interface consistency:</strong> when a type implements an interface, it must have all the interface's fields with compatible types.</li></ul><p>If any validation fails, the edit is rejected and the state remains unchanged. The error message includes enough information for the Hub UI to show exactly what went wrong and suggest a fix.</p><h3>Cascading Renames</h3><p>Renaming is the most complex operation, and it's where Fission's reference tracking really shines.</p><p>When a type is renamed at the supergraph level — say <code>Order</code> becomes <code>CustomerOrder</code> — Fission cascades the rename:</p><ol><li>Every subgraph containing <code>Order</code> has its node renamed</li><li>Every field typed as <code>Order</code> or <code>[Order!]!</code> is updated</li><li>Every <code>@key</code> field set referencing <code>Order</code> is regenerated from its selection tree</li><li>Interface implementations and union memberships are updated</li><li>All reverse indexes are re-keyed</li></ol><p>The SDL is not stored as text, it's regenerated from structured state. This means <code>@key(fields: &quot;id&quot;)</code> always reflects the current field names. If you rename the <code>id</code> field to <code>orderId</code>, the key directive automatically becomes <code>@key(fields: &quot;orderId&quot;)</code>.</p><h2>A Multi-Team Example</h2><p>Let's trace through a more complex scenario to see how these steps work together.</p><p><strong>Starting state:</strong> the three subgraphs from above (Users, Orders, Products).</p><p><strong>Dream query:</strong></p><pre>query OrderDetails($orderId: ID!) {
  order(id: $orderId) {
    total
    status
    items {
      product {
        name
        price
        inventory {          # new
          inStock
          warehouseLocation
        }
      }
    }
    shipping {               # new
      carrier
      trackingNumber
      estimatedDelivery
    }
    payment {                # new
      method
      last4
      receiptUrl
    }
  }
}
</pre><p>Three new capabilities: inventory, shipping, and payment.</p><p><strong>Step 1: Design on the supergraph</strong></p><p>The architect adds all new types and fields to the supergraph:</p><ul><li><code>Product.inventory: InventoryInfo!</code> (new field on existing entity)</li><li><code>InventoryInfo</code> with <code>inStock</code> and <code>warehouseLocation</code> (new type)</li><li><code>Order.shipping: ShippingInfo!</code> (new field on existing entity)</li><li><code>ShippingInfo</code> with <code>carrier</code>, <code>trackingNumber</code>, <code>estimatedDelivery</code> (new type)</li><li><code>Order.payment: PaymentInfo!</code> (new field on existing entity)</li><li><code>PaymentInfo</code> with <code>method</code>, <code>last4</code>, <code>receiptUrl</code> (new type)</li><li><code>Query.order(id: ID!): Order</code> (new root field)</li></ul><p>At this point, the supergraph describes the ideal API. None of these new types are assigned to subgraphs yet.</p><p><strong>Step 2: Assign to subgraphs</strong></p><p>The architect assigns each new type to a subgraph on the Hub canvas:</p><table><thead><tr><th>New item</th><th>Assigned to</th><th>What Fission automates</th></tr></thead><tbody><tr><td><code>InventoryInfo</code> + <code>Product.inventory</code></td><td>New Inventory subgraph</td><td><code>Product @key(fields: &quot;id&quot;)</code> is propagated to the Inventory subgraph. Product entity is automatically resolvable.</td></tr><tr><td><code>ShippingInfo</code> + <code>Order.shipping</code></td><td>New Shipping subgraph</td><td><code>Order @key(fields: &quot;id&quot;)</code> is propagated. If <code>ShippingInfo</code> referenced other types, they'd be pulled in transitively.</td></tr><tr><td><code>PaymentInfo</code> + <code>Order.payment</code></td><td>New Payments subgraph</td><td><code>Order @key(fields: &quot;id&quot;)</code> is propagated.</td></tr><tr><td><code>Query.order</code></td><td>Orders subgraph</td><td>Orders already owns the <code>Order</code> entity. Field is added, references are tracked.</td></tr></tbody></table><p>The architect decides that <code>Product.inventory</code> goes to a new Inventory subgraph because the Products team doesn't have warehouse data.</p><p><strong>Continuous validation</strong></p><p>Every assignment is validated as it happens:</p><ul><li>Entity keys are resolvable across the new subgraph boundaries</li><li>No type-kind conflicts (e.g., <code>Order</code> is an Object in all subgraphs)</li><li>Naming conventions pass (Hub's governance layer flags <code>inStock</code> — should it be <code>isInStock</code>? Architect approves the exception)</li><li>Reference integrity is maintained</li></ul><p>If the architect tried to assign <code>ShippingInfo</code> to a subgraph in a way that would break composition, the edit would be rejected with a specific error before any state changes.</p><h2>From Design to Subgraph Specs</h2><p>At any point, Fission can render valid GraphQL SDL from structured state. This means the output reflects every cascading update, renamed field, and propagated directive.</p><p><strong>Subgraph SDL Specifications</strong></p><p>For each affected subgraph, Fission renders the exact SDL that the team needs to implement:</p><pre># New subgraph: Shipping
type Order @key(fields: &quot;id&quot;) {
  id: ID!
  shipping: ShippingInfo!
}

type ShippingInfo {
  carrier: String!
  trackingNumber: String!
  estimatedDelivery: DateTime!
}
</pre><pre># New subgraph: Payments
type Order @key(fields: &quot;id&quot;) {
  id: ID!
  payment: PaymentInfo!
}

type PaymentInfo {
  method: String!
  last4: String!
  receiptUrl: String!
}
</pre><pre># New subgraph: Inventory
type Product @key(fields: &quot;id&quot;) {
  id: ID!
  inventory: InventoryInfo!
}

type InventoryInfo {
  inStock: Boolean!
  warehouseLocation: String!
}
</pre><pre># Modified subgraph: Orders (new root field)
type Query {
  order(id: ID!): Order  # new
}
</pre><p>Each new subgraph automatically includes <code>@key(fields: &quot;id&quot;)</code> on the entities it extends, because Fission propagated these from the existing subgraphs.</p><p><strong>Supergraph Preview</strong></p><p>Fission can also render the full supergraph SDL at any time. The architect can verify that the dream query will actually work before any code is written. Because the supergraph is the source of truth, not a composition result.</p><h2>Design Decisions</h2><p>A few architectural choices in Fission that are worth explaining:</p><p><strong>The supergraph is the source of truth.</strong> In traditional Federation, the supergraph is a side effect of composition. In Fission, it's the canonical representation. Removing a type from a subgraph does not remove it from the supergraph. You can design the complete API shape first, then decide how to distribute implementation. This is backwards from composition.</p><p><strong>Fission automates consequences, not decisions.</strong> The architect decides where types and fields belong. Fission automates everything that follows: transitive dependency propagation, entity key propagation, <code>@shareable</code> management, reference tracking, and cascading renames. The goal is to automate the mechanical work and surface the judgment calls to humans.</p><p><strong>Validate first, mutate second.</strong> Every edit validates all preconditions before making any state change. If validation fails, the state is unchanged. Because invalid states are caught at design time, the &quot;composition fails after three teams implement their parts&quot; scenario is prevented.</p><p><strong>Federation directives are structured state, not text.</strong> <code>@key</code>, <code>@shareable</code>, <code>@inaccessible</code>, <code>@override</code>, and <code>@provides</code> are modeled as structured state with their own selection trees, indexes, and validation rules. When you rename a field that participates in <code>@key(fields: &quot;id name&quot;)</code>, the key field set is automatically updated.</p><p><strong>Fission tracks the full lifecycle.</strong> From proposal to review to implementation to composition, the entire workflow is tracked in one place. When a subgraph team ships their implementation, it's checked against the original specification. The proposal isn't complete until all specifications are met and the final composition succeeds.</p><h2>The Bigger Picture</h2><p>Fission is the algorithmic counterpart to Hub's collaboration model.</p><p>Hub provides the canvas, the collaboration workflow, and the governance layer. Fission provides the ability to take a high-level design intent and translate it into concrete, actionable subgraph specifications.</p><p>Together, they make it possible to <a href="https://wundergraph.com/blog/design-like-a-monolith-implement-as-microservices">design like a monolith and implement as microservices</a>.</p><p>The supergraph is designed as a coherent whole. The implementation is distributed across teams. The algorithm handles the translation between the two.</p><p>If you want to see Fission in action, <a href="https://wundergraph.com/events/virtual/introducing-fission">watch the webinar</a> or <a href="https://wundergraph.com/contact/sales">schedule a walkthrough</a>.</p><hr /><hr /></article>
