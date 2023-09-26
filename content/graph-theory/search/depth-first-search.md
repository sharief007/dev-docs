---
title: Depth First Search
type: docs
weight: 2
toc: true
sidebar:
  open: true
prev: breadth-first-search
params:
  editURL: 
---

Depth First Search (DFS) is an algorithm for traversing or searching through a tree or graph data structure. It starts at a chosen node and explores as far as possible along each branch before backtracking. This means it goes as deep as possible down one branch before moving to the next branch.

### Examples

let's consider a few different types of graphs and perform DFS on them.

#### Graph 1: Undirected Unweighted Graph

```
   0 -- 1 -- 2
   |    |    |
   3 -- 4    5
```

**Traversal Results starting from node 0**:
- DFS: 0 1 4 3 2 5

#### Graph 2: Directed Graph

```
   0 -> 1 -> 2
   |    |    |
   v    v    v
   3    4    5
```

**Traversal Results starting from node 0**:
- DFS: 0 1 3 4 2 5

#### Graph 3: Disconnected Graph

```
   0 -- 1    2
   |    |    |
   3    4    5
```

**Traversal Results starting from node 0**:
- DFS: 0 1 4 3

#### Graph 4: Disconnected Directed Graph

```
   0 -> 1    2
   |    |    |
   v    v    v
   3    4    5
```

**Traversal Results starting from node 0**:
- DFS: 0 1 3 4

#### Graph 5: Tree

```
       0
      / \
     1   2
    / \   \
   3   4   5
```

**Traversal Results starting from node 0**:
- DFS: 0 1 3 4 2 5

#### Graph 6: Disconnected Tree

```
       0         6
      / \         \
     1   2         7
    / \   \       /
   3   4   5     8
```

**Traversal Results starting from node 0**:
- DFS: 0 1 3 4 2 5

#### Graph 7: Graph with Loops

```
   0 -- 1 -- 2
   |    |    |
   3 -- 4 -- 5
```

**Traversal Results starting from node 0**:
- DFS: 0 1 4 3 5 2

#### Graph 8: Disconnected Graph with a Loop

```
   0 -- 1    2
   |    |    |
   v    v    v
   3 -- 4 -- 5
```

**Traversal Results starting from node 0**:
- DFS: 0 1 4 3

### Implementation

{{< tabs items="java, python, go" >}}
    {{< tab >}}
    ```java
    class Graph {
        // Method to perform DFS starting from a given source node
        public void dfs(List<List<Integer>> graph, int source) {
            int size = graph.size();
            boolean[] visited = new boolean[size]; // Array to keep track of visited nodes
            dfsUtil(graph, source, visited); // Start DFS from the source node
            
            // If there are unvisited nodes, perform DFS on them
            for(int i=0; i<size; i++) {
                if (!visited[i]) {
                    dfsUtil(graph, i, visited);
                }
            }
        }

        // Recursive method for performing DFS
        public void dfsUtil(List<List<Integer>> graph, int source, boolean[] visited) {
            visited[source] = true; // Mark the current node as visited
            System.out.print(source + " "); // Print the current node

            // Visit all the neighbors of the current node
            for(int neighbour: graph.get(source)) {
                if (!visited[neighbour]) {
                    dfsUtil(graph, neighbour, visited); // Recursive call for unvisited neighbors
                }
            }
        }
    }

    ```
    {{< /tab >}}
{{< /tabs >}}

### Complexity Analysis

- **Time Complexity**: The time complexity of the DFS algorithm is O(V + E), where V is the number of vertices and E is the number of edges in the graph.
- **Space Complexity**: The space complexity of the DFS algorithm is O(V), where V is the number of vertices in the graph.