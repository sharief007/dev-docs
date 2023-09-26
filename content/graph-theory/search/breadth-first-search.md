---
title: Breadth First Search
type: docs
weight: 1
toc: true
sidebar:
  open: true
params:
  editURL: 
---

Breadth-First Search (BFS) is a graph traversal algorithm that explores a graph level by level, starting from a chosen source node. It visits all the immediate neighbors of the source node first, then moves on to their neighbors, and so forth, until it has visited all reachable nodes.

### Examples

let's consider a few different types of graphs and perform BFS on them.

#### Graph 1: Undirected Unweighted Graph

```
   0 -- 1 -- 2
   |    |    |
   3 -- 4    5
```

**Traversal Results starting from node 0**:
- BFS: 0 1 3 4 2 5

#### Graph 2: Directed Graph

```
   0 -> 1 -> 2
   |    |    |
   v    v    v
   3    4    5
```

**Traversal Results starting from node 0**:
- BFS: 0 1 3 4 2 5

#### Graph 3: Disconnected Graph

```
   0 -- 1    2
   |    |    |
   3    4    5
```

**Traversal Results starting from node 0**:
- BFS: 0 1 3 4

#### Graph 4: Disconnected Directed Graph

```
   0 -> 1    2
   |    |    |
   v    v    v
   3    4    5
```

**Traversal Results starting from node 0**:
- BFS: 0 1 3 4

#### Graph 5: Tree

```
       0
      / \
     1   2
    / \   \
   3   4   5
```

**Traversal Results starting from node 0**:
- BFS: 0 1 2 3 4 5

#### Graph 6: Disconnected Tree

```
       0         6
      / \         \
     1   2         7
    / \   \       /
   3   4   5     8
```

**Traversal Results starting from node 0**:
- BFS: 0 1 2 3 4 5

#### Graph 7: Graph with Loops

```
   0 -- 1 -- 2
   |    |    |
   3 -- 4 -- 5
```

**Traversal Results starting from node 0**:
- BFS: 0 1 3 4 2 5

#### Graph 8: Disconnected Graph with a Loop

```
   0 -- 1    2
   |    |    |
   v    v    v
   3 -- 4 -- 5
```

**Traversal Results starting from node 0**:
- BFS: 0 1 3 4

### Implementation

{{< tabs items="java, python, go" >}}
    {{< tab >}}
    ```java
    import java.util.*;

    public class BFS {
        // Method to perform Breadth-First Search (BFS)
        public static void bfs(List<List<Integer>> graph, int source) {
            // Initialize an array to keep track of visited nodes
            boolean[] visited = new boolean[graph.size()];
            
            // Initialize a queue for BFS
            Queue<Integer> queue = new LinkedList<>();
            
            // Mark the source node as visited and enqueue it
            visited[source] = true;
            queue.offer(source);
            
            // Continue until the queue is empty
            while (!queue.isEmpty()) {
                // Dequeue a node and process it
                int current = queue.poll();
                System.out.print(current + " ");
                
                // Visit and enqueue unvisited neighbors
                for (int neighbor : graph.get(current)) {
                    if (!visited[neighbor]) {
                        visited[neighbor] = true;
                        queue.offer(neighbor);
                    }
                }
            }
        }
    }
    ```
    {{< /tab >}}
{{< /tabs >}}

### Complexity Analysis

- **Time Complexity:** In the worst case, where every node is reachable from the source, BFS visits every node once, which gives a time complexity of O(V + E), where V is the number of vertices and E is the number of edges.

- **Space Complexity:** The space complexity is O(V) for the visited array and O(V) for the queue, giving a total of O(V).