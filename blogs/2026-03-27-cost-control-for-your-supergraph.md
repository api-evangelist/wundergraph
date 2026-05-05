---
title: "Cost Control for Your Supergraph"
url: "https://wundergraph.com/blog/cost-control-for-your-supergraph"
date: "2026-03-27T00:00:00.000Z"
author: "Yury Smolski"
feed_url: "https://wundergraph.com/atom.xml"
---
<article><p>Cost Control prevents overly expensive operations in a Supergraph by estimating the cost of each request and rejecting operations over the set cost limit in Enforcing Mode. The router uses a default algorithm based on IBM’s GraphQL Cost Directive specification, with support for customization through <code>@cost</code> and <code>@listSize</code> directives, and each operation can be assigned a cost weight corresponding to the resources used in production. Estimated cost is computed at planning time and remains static, while actual cost is recorded from subgraph responses and is dynamic based on real response sizes.</p><hr /><p>Graphs are always growing. You may have many clients using your Supergraph, and if they send complex queries, costs can quickly get out of hand. We introduce the <strong>Cost Control</strong> feature to secure your Supergraph from overly expensive operations.</p><p>With this feature enabled, <a href="https://wundergraph.com/router-gateway">the Cosmo Router</a> computes the estimated cost for every incoming request. If the operation exceeds the set cost limit, Cosmo rejects it, preventing overload. At the same time, it measures the actual cost, which can be used by your systems in different ways.</p><h2>Why</h2><p>Imagine you have a large Supergraph, a bunch of customers, and one of these problems:</p><ol><li>You see spikes of requests from customers that put the infrastructure into a coma. Scaling resources won’t help because those spikes last only a few minutes. You want to throttle these customers, and use query complexity as a signal.</li><li>Your federation runs fine, but just a few times a day there is a sudden increase in the latency of requests. You do not know why and want to observe which requests are putting pressure on your subgraphs.</li><li>You have customers using your federation, and you want to bill them based on how complex their requests are. Each customer has its own patterns of usage, and you cannot tier them into packages. You need a flexible, pay-for-resources model.</li><li>You have a complex Supergraph with a possibility to nest lists deeply. This can explode response sizes, and it can be abused. And yet, you don’t want to exclude all the users from this flexibility, only the cases where it is abused.</li></ol><p>These problems look different, but the solution can be shared. By implementing a cost algorithm for your Supergraph, you can address all of them.</p><p>To enable Cost Control in your router config, <a href="https://cosmo-docs.wundergraph.com/router/security/cost-control">see the setup guide</a>.</p><h2>How It Works</h2><p>The Cosmo Router implements a default cost algorithm based on <a href="https://ibm.github.io/graphql-specs/cost-spec.html">IBM’s GraphQL Cost Directive specification</a>. Users can customize how costs are calculated using the <code>@cost</code> and <code>@listSize</code> directives. Each operation can be assigned a cost weight corresponding to the resources used in production.</p><p>The Router computes the estimated (static) cost for every operation at planning time. Estimations are static because list sizes are static.</p><p>There are two modes for this feature.</p><p>In <strong>Measuring Mode</strong>, the estimated cost is recorded, but nothing is rejected. The cost appears in response headers, OTEL metrics, or custom modules. This mode helps you determine limits or build custom rejection logic.</p><p><strong>Enforcing Mode</strong> rejects an operation over the set cost limit before the execution starts.</p><p>Additionally, the Cosmo Router records actual (dynamic) cost based on the response it got from subgraphs. This can be used for billing based on real usage. The value is dynamic because response sizes vary.</p><p>To learn more about Cost Control, check out our <a href="https://cosmo-docs.wundergraph.com/router/security/cost-control">documentation</a>.</p><h2>An Example</h2><p>Given this schema:</p><pre>type Query {
    products(first: Int!): [Product] @listSize(slicingArguments: [&quot;first&quot;])
}

type Product @cost(weight: 5) {
    name: String                     # default scalar weight: 0
    category: Category               # default object weight: 1
}

type Category {
    name: String @cost(weight: 2)    # overriding default weight for scalars
}
</pre><p>How are costs calculated when a user sends the following request?</p><pre>query {
    products(first: 20) {
        name
        category {
            name
        }
    }
}
</pre><h3>Estimated Cost Breakdown</h3><p>All fields have default values assigned to them. In our example, the <code>Product</code> type has the assigned weight of <code>5</code>. This value applies whenever the type is used. The <code>products</code> field returns a list. This means every child in the selection is multiplied by the expected list size. The <code>@listSize</code> directive tells the router to set the list size to the value of the <code>first</code> argument, <code>20</code> in the query. Otherwise, it would fall back to <code>estimated_list_size</code> from the router config.</p><p>The estimated cost is composed of:</p><pre>products_arguments_cost + products_multiplier * (product_cost + 
                                                 product_name_cost + 
                                                 category_cost + 
                                                 category_name_cost)
</pre><p>Substituting with numbers, the estimated cost is <code>160</code>:</p><pre>0 + 20 * ( 5 + 0 + 1 + 2 ) = 160
</pre><h3>Actual Cost Breakdown</h3><p>Actual cost is calculated the same way, but uses real list sizes from the response instead of static multipliers.</p><p>Suppose that the response has <code>16</code> products. In that case, the actual cost would be <code>128</code>:</p><pre>0 + 16 * ( 5 + 0 + 1 + 2 ) = 128
</pre><p>To learn the nuances of calculation, refer to <a href="https://ibm.github.io/graphql-specs/cost-spec.html">IBM’s GraphQL Cost specification</a> and our <a href="https://cosmo-docs.wundergraph.com/router/security/cost-control">documentation</a>.</p><hr /><hr /></article>
