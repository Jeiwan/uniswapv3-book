---
title: "User Interface"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# User Interface

After introducing swap paths, we can significantly simplify the internals of our web app. First of all, every swap now
uses a path since path doesn't have to contain multiple pools. Second, it's now easier to change the direction of swap:
we can simply reverse the path. And, thanks to the unified pool address generation via `CREATE2` and unique salts, we
no longer need to store pool addresses and care about tokens order.

However, we cannot integrate multi-pool swaps in the web app without adding one crucial algorithm. Ask yourself the
question: "How to find a path between two tokens that don't have a pool?"

## AutoRouter

Uniswap implements what's called *AutoRouter*, an algorithm that find shortest path between two tokens. Moreover, it also
splits one payment into multiple smaller payments to find the best average exchange rate. The profit can be as big as 
[36.84% compared to trades that are not split](https://uniswap.org/blog/auto-router-v2). This sounds great, however, we're
not going to build such an advanced algorithm. Instead, we'll build something simpler.

## A Simple Router Design

Suppose we have a whole bunch of pools:

![Scattered pools](/images/milestone_4/pools_scattered.png)

How do we find a shortest path between two tokens in such a mess?

The most suitable solution for such kind of tasks is based on a *graph*. A graph is a data structure that consists of
nodes (objects representing something) and edges (links connecting nodes). We can turn that mess of pools into a graph
where each node is a token (that has a pool) and each edge is a pool this token belongs to. So a pool represented as a
graph is two nodes connected with an edge. And the above pools become this graph:

![Pools graph](/images/milestone_4/pools_graph.png)

The biggest advantage graphs give us is the ability to traverse its nods, from one node to another, to find paths. Specifically,
we'll use [A* search algorithm](https://en.wikipedia.org/wiki/A*_search_algorithm). Feel free learning about how the
algorithm works, but, in our app, we'll use a library to make our life easier. The set of libraries we'll use is:
[ngraph.ngraph](https://github.com/anvaka/ngraph.graph) for building graphs and [ngraph.path](https://github.com/anvaka/ngraph.path)
for finding paths (it's the latter that implements A* search algorithm, as well as some others).

In the UI app, let's create a path finder. This will be a class that, when instantiated, turns a list of pairs into a
graph to later use the graph to find a shortest path between two tokens.
```javascript
import createGraph from 'ngraph.graph';
import path from 'ngraph.path';

class PathFinder {
  constructor(pairs) {
    this.graph = createGraph();

    pairs.forEach((pair) => {
      this.graph.addNode(pair.token0.address);
      this.graph.addNode(pair.token1.address);
      this.graph.addLink(pair.token0.address, pair.token1.address, pair.tickSpacing);
      this.graph.addLink(pair.token1.address, pair.token0.address, pair.tickSpacing);
    });

    this.finder = path.aStar(this.graph);
  }

  ...
```

In the constructor, we're creating an empty graph and fill it with linked nodes. Each node is a token address and links
have associated data, which is tick spacings–we'll be able to extract this information from paths found by A*. After
initializing a graph, we instantiate A* algorithm implementation.

Next, we need to implement a function that will find a path between tokens and turn it into an array of token addresses
and tick spacings:

```javascript
findPath(fromToken, toToken) {
  return this.finder.find(fromToken, toToken).reduce((acc, node, i, orig) => {
    if (acc.length > 0) {
      acc.push(this.graph.getLink(orig[i - 1].id, node.id).data);
    }

    acc.push(node.id);

    return acc;
  }, []).reverse();
}
```

`this.finder.find(fromToken, toToken)` returns a list of nodes and, unfortunately, doesn't contain the information
about edges between them (we store tick spacings in edges). Thus, we're calling `this.graph.getLink(previousNode, currentNode)`
to find edges.

Now, whenever user changes input or output token, we can call `pathFinder.findPath(token0, token1)` to build a new path.