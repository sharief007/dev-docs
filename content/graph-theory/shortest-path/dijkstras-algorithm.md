---
title: Dijkstra's Algorithm
type: docs
weight: 1
toc: true
sidebar:
  open: true
next: depth-first-search
params:
  editURL: 
---

Dijkstra's Algorithm is a method used to find the shortest path from a specified starting point (source) to all other points (nodes) in a weighted graph, where the edge weights represents `non-negative distances` between nodes. It ensures that the shortest path to each node is determined accurately, considering all possible routes through the graph, and is commonly employed in navigation and network routing applications.

### Examples

1. **Simple Weighted Graph**:

```
       A
      / \
     2   3
    /     \
   B-------C
      \   /
        1
        D
```

Initial distances: A: ∞, B: ∞, C: ∞, D: ∞

Applying Dijkstra's Algorithm:

1. Start from A.
2. Update distances: A: 0, B: 2, C: 3, D: ∞
3. Move to B.
4. Update distances: A: 0, B: 2, C: 3, D: 3
5. Move to D.
6. Update distances: A: 0, B: 2, C: 3, D: 3
7. Move to C.
8. Update distances: A: 0, B: 2, C: 3, D: 3

Final distances: A: 0, B: 2, C: 3, D: 3

2. **Graph with Negative Weight**:

```
       A
      / \
     1  -3
    /     \
   B-------C
      \   /
      -2
        D
```

This graph has negative weights, so Dijkstra's Algorithm may not work correctly. It's designed for non-negative weights.

3. **Disconnected Graph**:

```
   A        B       C
   |        |       |
   1        2       3
   |        |       |
   D        E       F
```

If you start from any vertex, you won't be able to reach the other vertices. Dijkstra's Algorithm would report infinity for all unreachable vertices.

4. **Graph with Loop**:

```
   A--1--B
   |     |
   2     3
   |     |
   C--4--D
```

In this case, you have a loop between A, B, C, and D. This can lead to infinite loops in Dijkstra's Algorithm.

5. **Graph with Duplicate Edges**:

```
   A--1--B
   |     |
   2     3
   |     |
   C--4--D
   |     |
   1     5
   |     |
   E--2--F
```

Duplicate edges can be handled, but it's important to make sure they are handled correctly in the implementation of Dijkstra's Algorithm.

Remember, Dijkstra's Algorithm is designed for non-negative weighted graphs without negative cycles. If your graph violates these conditions, the algorithm may not produce correct results.

### Implementation

{{< tabs items="java, python, go" >}}
    {{< tab >}}
    ```java
    import java.util.*;

    public class DijkstraAlgorithm {

        // Define a class to represent a node in the graph
        static class Node implements Comparable<Node> {
            int id;
            int distance;

            public Node(int id, int distance) {
                this.id = id;
                this.distance = distance;
            }

            @Override
            public int compareTo(Node other) {
                return Integer.compare(this.distance, other.distance);
            }
        }

        static void dijkstra(int[][] graph, int source) {
            int n = graph.length;
            Set<Integer> visited = new HashSet<>();
            int[] distances = new int[n];
            Arrays.fill(distances, Integer.MAX_VALUE);

            // Create priority queue
            PriorityQueue<Node> pq = new PriorityQueue<>();
            pq.add(new Node(source, 0));
            distances[source] = 0;

            while (!pq.isEmpty()) {
                Node current = pq.poll();
                int currentId = current.id;

                if (visited.contains(currentId)) continue;

                visited.add(currentId);

                for (int neighbor = 0; neighbor < n; neighbor++) {
                    if (graph[currentId][neighbor] != 0) {
                        int distanceThroughCurrent = distances[currentId] + graph[currentId][neighbor];
                        if (distanceThroughCurrent < distances[neighbor]) {
                            distances[neighbor] = distanceThroughCurrent;
                            pq.add(new Node(neighbor, distanceThroughCurrent));
                        }
                    }
                }
            }

            // Print the distances from source to each node
            for (int i = 0; i < n; i++) {
                System.out.println("Distance from source to node " + i + " is " + distances[i]);
            }
        }

        public static void main(String[] args) {
            int[][] graph = {
                {0, 10, 0, 0, 0},
                {10, 0, 20, 0, 0},
                {0, 20, 0, 10, 0},
                {0, 0, 10, 0, 50},
                {0, 0, 0, 50, 0}
            };

            int source = 0;
            dijkstra(graph, source);
        }
    }
    ```
    {{< /tab >}}
{{< /tabs >}}

**Edge Cases Covered:**

**Directed Graph:** The provided code handles directed graphs where edges may only go in one direction.

**Weighted Edges:** The graph is represented using an adjacency matrix, allowing for weighted edges.

**Disconnected Nodes:** The algorithm can handle cases where some nodes are not reachable from the source.

**Non-Negative Weights:** Dijkstra's algorithm assumes non-negative edge weights. If there are negative weights, you'd need to use a different algorithm like Bellman-Ford.

**Handling Infinite Distances:** The initial distances are set to Integer.MAX_VALUE, representing infinity. When a valid path is found, the distance is updated.

### Complexity Analysis

**Time Complexity:**
- **Worst Case:** O((V + E) log V)
- **Best Case (dense graph):** O((V^2) log V)
- **Best Case (sparse graph):** O((V + E) log V)
- **Average Case:** O((V + E) log V)

**Space Complexity:**
- **Visited Set:** O(V)
- **Distances Array:** O(V)
- **Priority Queue (Binary Heap):** O(V)
- **Adjacency Matrix:** O(V^2)

The total space complexity is O(V^2) due to the adjacency matrix. In some implementations, you might use an adjacency list instead, which can reduce space complexity to O(V + E).

Here, V is the number of vertices.E is the number of edges.

**Note:** If you're using a Fibonacci heap or a more efficient data structure for the priority queue, the time complexity could potentially be improved. However, the space complexity may also increase.