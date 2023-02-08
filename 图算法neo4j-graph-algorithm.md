#图算法neo4j-graph-algorithm

图模型的建立是为了描述节点以及节点之间的关系。在复杂的图模型中，节点之间往往都涵盖复联系，为了在图模型挖掘出信息，往往需要引入相关图算法进行计算，表述。

neo4j-graph-algorithms是neo4j提供一系列执行过程的集合，这些执行过程都实现了相对应的图算法。

## 图算法分类

从图模型的使用角度出发neo4j-graph-algorithms将其图算法分类三类：

1. 路径发现和图搜索算法(Pathfinding and Graph Search Algorithms )

   路径算法依赖图的遍历，可以计算所有连通的路径，最小路径以及单源路径。还可以计算图的最小生成树。也可以对图模型进行随机漫步做统计方面相关的分析研究

2. 中心算法( Centrality Algorithms)

   中心算法依赖对每个节点以及与其关联的关系的复杂情况计算，可以发掘去图模型中最重要（最特殊，影响力最大）的节点，或者对图模型的节点进行重要度排序。著名的PageRank算法就是一种依赖中心度计算的算法。

3. 社群发现(CommunityDetection Algorithms)

   社群发现是在图模型中，发现一批一批联系密切的点，将这些联系密切的点组成一个小社群。通过对图模型中的节点进行分类，可以认为一个社群中的节点存在一些共性，这样就可以根据已有的信息挖掘出该社群中其他点的信息。

neo4j-graph-algorithms提供的图算法都涵盖在这三类中

1. 路径发现和图搜索算法Pathfinding and Graph Search Algorithms
   - 广度优先搜索Breadth First Search 
   - 深度优先搜索Depth First Search 
   - 最短路径算法Shortest Path 
   - 所有最短路径All Pairs Shortest Path 
   - 单源最短路径Single Source Shortest Path
   - 最小生成树Minimum Spanning Tree
   - 随机漫步Random Walk
2. 中心性算法( Centrality Algorithms)
   - 点度中心性Degree Centrality
   - 紧密中心性Closeness Centrality\Harmonic Centrality
   - 介数中心性Betweeness Centrality
   - 特征向量中心性Eigenvector Centrality
   - PageRank
3. 社群发现(CommunityDetection Algorithms)
   - 三角形计数Triangle Count and Clustering Coefficient 
   - 强连通分量Strongly Connected Components 
   - 连通分量Connected Components 
   - 标签传播算法Label Propagation 
   - Louvain Modularity 

## 图算法原理

*Cu*=*n*−1 *norm* ∑*n*−1*d u*,*v* 

## neo4j-graph-algorithm图算法源码

### ShortestPath

algo.bfs.stream

```cypher
CALL algo.bfs.stream(label:String, relationshipType:String, startNodeId:long, direction:Direction, {writeProperty:String, target:long, maxDepth:long, weightProperty:String, maxCost:double}) 
YIELD nodeId
```

algo.dfs.stream

```cypher
CALL algo.dfs.stream(label:String, relationshipType:String, startNodeId:long, direction:Direction, {writeProperty:String, target:long, maxDepth:long, weightProperty:String, maxCost:double}) 
YIELD nodeId
```

algo.shortestPath.stream

```cypher
CALL algo.shortestPath.stream(startNode:Node, endNode:Node, weightProperty:String, {nodeQuery:'labelName', relationshipQuery:'relationshipName', direction:'BOTH', defaultValue:1.0}) 
YIELD nodeId, cost // yields a stream of {nodeId, cost} from start to end (inclusive)
```

algo.kShortestPaths.stream

```cypher
CALL algo.kShortestPaths.stream(startNode:Node, endNode:Node, k:int, weightProperty:String {nodeQuery:'labelName', relationshipQuery:'relationshipName', direction:'OUT', defaultValue:1.0, maxDepth:42})
YIELD sourceNodeId, targetNodeId, nodeIds, costs
```

algo.allShortestPaths.stream

```cypher
CALL algo.allShortestPaths.stream(weightProperty:String, {nodeQuery:'labelName', relationshipQuery:'relationshipName', defaultValue:1.0, concurrency:4})
YIELD sourceNodeId, targetNodeId, distance // yields a stream of {sourceNodeId, targetNodeId, distance}
```

algo.shortestPath.deltaStepping.stream

```cypher
CALL algo.shortestPath.deltaStepping.stream(startNode:Node, weightProperty:String, delta:Double {label:'labelName', relationship:'relationshipName', defaultValue:1.0, concurrency:4}) 
YIELD nodeId, distance // yields a stream of {nodeId, distance} from start to end (inclusive)
```

### Spanning Tree

algo.spanningTree

```cypher
CALL algo.spanningTree(label:String, relationshipType:String, weightProperty:String, startNodeId:long, {writeProperty:String}) 
YIELD loadMillis, computeMillis, writeMillis, effectiveNodeCount
```

algo.spanningTree.minimum

```cypher
CALL algo.spanningTree.minimum(label:String, relationshipType:String, weightProperty:String, startNodeId:long, {writeProperty:String})                                                                                                         YIELD loadMillis, computeMillis, writeMillis, effectiveNodeCount
```

algo.spanningTree.maximum

```cypher
CALL algo.spanningTree.maximum(label:String, relationshipType:String, weightProperty:String, startNodeId:long, {writeProperty:String})
YIELD loadMillis, computeMillis, writeMillis, effectiveNodeCount
```

algo.spanningTree.kmin

```cypher
CALL algo.spanningTree.kmin(label:String, relationshipType:String, weightProperty:String, startNodeId:long, k:int, {writeProperty:String})
YIELD loadMillis, computeMillis, writeMillis, effectiveNodeCount
```

### Random Walk

algo.randomWalk.stream

```cypher
CALL algo.randomWalk.stream(start:null=all/[ids]/label, steps, walks, {graph: 'heavy/cypher', nodeQuery:nodeLabel/query, relationshipQuery:relType/query, mode:random/node2vec, return:1.0, inOut:1.0, path:false/true concurrency:4, direction:'BOTH'})
YIELD nodes, path // computes random walks from given starting points
```

### Centrality

algo.degree.stream

```cypher
CALL algo.degree.stream(label:String, relationship:String, {weightProperty: null, concurrency:4})
YIELD node, score // calculates degree centrality and streams results
```

algo.betweenness.stream

```cypher
CALL algo.betweenness.stream(label:String, relationship:String, {direction:'out', concurrency :4})
YIELD nodeId, centrality // yields centrality for each node
```

algo.closeness.stream 

```cypher
CALL algo.closeness.stream(label:String, relationship:String{concurrency:4})   							 YIELD nodeId, centrality // yields centrality for each node
```

algo.closeness.dangalchev.stream

```cypher
CALL algo.closeness.dangalchev.stream(label:String, relationship:String{concurrency:4}) 
YIELD nodeId, centrality // yields centrality for each node
```

algo.closeness.harmonic.stream

```cypher
CALL algo.closeness.harmonic.stream(label:String, relationship:String{concurrency:4}) 
YIELD nodeId, centrality // yields centrality for each node
```

algo.eigenvector.stream

```cypher
CALL algo.eigenvector.stream(label:String, relationship:String, {weightProperty: null, concurrency:4}) 
YIELD node, score // calculates eigenvector centrality and streams results
```

### PageRank

algo.pageRank.stream

```cypher
CALL algo.pageRank.stream(label:String, relationship:String, {iterations:20, dampingFactor:0.85, weightProperty: null, concurrency:4}) 
YIELD node, score // calculates page rank and streams results
```

### Triangles

algo.triangle.stream

```cypher
CALL algo.triangle.stream(label, relationship, {concurrency:4})
YIELD nodeA, nodeB, nodeC // yield nodeA, nodeB and nodeC which form a triangle
```

algo.triangleCount.stream

```cypher
CALL algo.triangleCount.stream(label, relationship, {concurrency:8})
YIELD nodeId, triangles // yield nodeId, number of triangles
```

### Connected Components

algo.scc.stream

```cypher
CALL algo.scc.stream(label:String, relationship:String, config:Map<String, Object>) 
YIELD loadMillis, computeMillis, writeMillis, setCount, maxSetSize, minSetSize
```

algo.scc.recursive.tunedTarjan.stream

```cypher
CALL algo.scc.recursive.tunedTarjan.stream(label:String, relationship:String, config:Map<String, Object>) 
YIELD nodeId, partition
```

algo.scc.iterative.steam

```cypher
CALL algo.scc.iterative.stream(label:String, relationship:String, config:Map<String, Object>) 
YIELD nodeId, partition
```

algo.scc.multistep.steam

```cypher
CALL algo.scc.multistep.stream(label:String, relationship:String, {write:true, concurrency:4, cutoff:100000}) 
YIELD nodeId, partition
```

algo.scc.forwardBackward.stream

```cypher
CALL algo.scc.forwardBackward.stream(long startNodeId, label:String, relationship:String, {write:true, concurrency:4}) 
YIELD nodeId, partition
```

algo.unionFind.stream

```cypher
CALL algo.unionFind.stream(label:String, relationship:String, {weightProperty:'propertyName', threshold:0.42, defaultValue:1.0)
YIELD nodeId, setId // yields a setId to each node id
```

algo.unionFind.queue.stream

```cypher
CALL algo.unionFind.queue.stream(label:String, relationship:String, {property:'propertyName', threshold:0.42, defaultValue:1.0, concurrency:4})
YIELD nodeId, setId // yields a setId to each node id
```

### Louvain

algo.louvain.steam

```cypher
CALL algo.louvain.stream(label:String, relationship:String, {weightProperty:'propertyName', defaultValue:1.0, concurrency:4, communityProperty:'propertyOfPredefinedCommunity', innerIterations:10, communitySelection:'classic')
YIELD nodeId, community // yields a setId to each node id
```

### LabelPropagation

algo.labelPropagation.stream

```cypher
CALL algo.labelPropagation.stream(label:String, relationship:String, config:Map<String, Object>) 
YIELD nodeId, label
```

## 说明

> 在neo4j4.x版本时期，neo4j-graph-algorithms被合并到neo4j的[Graph Data Science(GDS) library](https://github.com/neo4j/graph-data-science/)中，
>
> 在neo4j-graph-algorithms包的基础之上，GDS增加了一些额外有用的算法，包括:
>
> - 相似性算法(Similarity algorithms)
> - 链接预测算法(Link Prediction Algorithms)
>
> 同时还提供了一些图计算框架：
>
> - Auxiliary procedures - neo4j提供的一些有用的工作流工具包
> - Pregel API - 对谷歌Pregel计算框架实现的neo4j的api

虽然GDS包增加了更多的功能以及额外的算法，但主流的图算法已经在neo4j-graph-algorithms包中都有了完整实现，因此neo4j-graph-algorithms代码阅读已经可以覆盖到图算法的基础。

再深入阅读GDS的代码可以对PregelAPI和neo4j提供的算法工作流工具包有进一步的理解，也对大型图模型计算的工作流程有更深的认识。