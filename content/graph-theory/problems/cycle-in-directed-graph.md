---
title: 'Cycle in Directed Graphs'
weight: 1
type: docs
toc: true
sidebar:
  open: true
params:
  editURL: 
---

Practice: https://practice.geeksforgeeks.org/problems/detect-cycle-in-a-directed-graph/1

### Problem

Given a Directed Graph with V vertices (Numbered from 0 to V-1) and E edges, check whether it contains any cycle or not.

### Same Path Visited



{{< tabs items="java, python, go" >}}
{{< tab >}}
```java    
class Solution {
    // Function to detect cycle in a directed graph.
    public boolean isCyclic(int n, ArrayList<ArrayList<Integer>> adj) {
        // code here
        boolean[] visited = new boolean[n];
        boolean[] samepath = new boolean[n];
        
        // there could be multiple components
        for (int i=0; i<n; i++) {
            // if any component not visited
            if(!visited[i]) {
                // check for cycle
                if(isCyclic(i, adj, visited, samepath)) {
                    return true;
                }
            }
        }
        
        // no cycles found in graph
        return false;
    }
    
    private boolean isCyclic(int index, ArrayList<ArrayList<Integer>> adj, boolean[] visited, boolean[] samePath) {
        // this vertex is visited and
        // that visit happened in the same path
        // therefore, there exists a cycle ih this path
        if (visited[index] && samePath[index]) {
            return true;
        }
    
        // if already visited, the rest of the path is already visited
        if(visited[index]) {
            return false;
        }
        
        // mark this vertex as visited
        visited[index] = true;
        samePath[index] = true;
        
        // foreach neighbour, check for cycle
        for (int neigh: adj.get(index)) {
            boolean result = isCyclic(neigh, adj, visited, samePath);
            if(result) {
                return true;
            }
        }
        
        // no cycle found in this path, so remove vertex from path
        samePath[index] = false;
        
        // if no cycles found return false
        return false;
    }
}
```
{{< /tab >}}
{{< tab >}}
```python
class Solution:
  # Function to detect cycle in a directed graph.
  def isCyclic(self, n, adj):
      # code here
      visited = [False] * n
      samepath = [False] * n
      
      # there could be multiple components
      for i in range(n):
          # if any component not visited
          if not visited[i]:
              # check for cycle
              if self.isCyclicUtil(i, adj, visited, samepath):
                  return True
      
      # no cycles found in graph
      return False
  
  def isCyclicUtil(self, index, adj, visited, samePath):
      # this vertex is visited and
      # that visit happened in the same path
      # therefore, there exists a cycle in this path
      if visited[index] and samePath[index]:
          return True
  
      # if already visited, the rest of the path is already visited
      if visited[index]:
          return False
      
      # mark this vertex as visited
      visited[index] = True
      samePath[index] = True
      
      # for each neighbor, check for cycle
      for neigh in adj[index]:
          result = self.isCyclicUtil(neigh, adj, visited, samePath)
          if result:
              return True
      
      # no cycle found in this path, so remove vertex from path
      samePath[index] = False
      
      # if no cycles found return false
      return False
```
{{< /tab >}}
{{< tab >}}
```go
package main

// Function to detect cycle in a directed graph.
func isCyclic(n int, adj [][]int) bool {
	// code here
	visited := make([]bool, n)
	samePath := make([]bool, n)

	// there could be multiple components
	for i := 0; i < n; i++ {
		// if any component not visited
		if !visited[i] {
			// check for cycle
			if isCyclicUtil(i, adj, visited, samePath) {
				return true
			}
		}
	}

	// no cycles found in graph
	return false
}

func isCyclicUtil(index int, adj [][]int, visited, samePath []bool) bool {
	// this vertex is visited and
	// that visit happened in the same path
	// therefore, there exists a cycle in this path
	if visited[index] && samePath[index] {
		return true
	}

	// if already visited, the rest of the path is already visited
	if visited[index] {
		return false
	}

	// mark this vertex as visited
	visited[index] = true
	samePath[index] = true

	// for each neighbor, check for cycle
	for _, neigh := range adj[index] {
		result := s.isCyclicUtil(neigh, adj, visited, samePath)
		if result {
			return true
		}
	}

	// no cycle found in this path, so remove vertex from path
	samePath[index] = false

	// if no cycles found return false
	return false
}
```
{{< /tab >}}
{{< /tabs >}}

### Optimized

{{< tabs items="java, python, go" >}}
{{< tab >}}
```java
class Solution {
    public boolean isCyclic(int n, ArrayList<ArrayList<Integer>> adj) {
        int[] state = new int[n];
        
        for (int i = 0; i < n; i++) {
            if (state[i] == 0 && isCyclic(i, adj, state)) {
                return true;
            }
        }
        
        return false;
    }
    
    private boolean isCyclic(int index, ArrayList<ArrayList<Integer>> adj, int[] state) {
        if (state[index] == 2) {
            return true;
        }
        
        if (state[index] == 1) {
            return false;
        }
        
        state[index] = 2;
        
        for (int neigh : adj.get(index)) {
            if (isCyclic(neigh, adj, state)) {
                return true;
            }
        }
        
        state[index] = 1;
        
        return false;
    }
}
```
{{< /tab >}}
{{< tab >}}
```python
class Solution:
    def isCyclic(self, n, adj):
        state = [0] * n
        
        for i in range(n):
            if state[i] == 0 and self.isCyclicUtil(i, adj, state):
                return True
        
        return False
    
    def isCyclicUtil(self, index, adj, state):
        if state[index] == 2:
            return True
        
        if state[index] == 1:
            return False
        
        state[index] = 2
        
        for neigh in adj[index]:
            if self.isCyclicUtil(neigh, adj, state):
                return True
        
        state[index] = 1
        
        return False
```
{{< /tab >}}
{{< tab >}}
```go
package main

type Solution struct{}

func (s Solution) isCyclic(n int, adj [][]int) bool {
	state := make([]int, n)

	for i := 0; i < n; i++ {
		if state[i] == 0 && s.isCyclicUtil(i, adj, state) {
			return true
		}
	}

	return false
}

func (s Solution) isCyclicUtil(index int, adj [][]int, state []int) bool {
	if state[index] == 2 {
		return true
	}

	if state[index] == 1 {
		return false
	}

	state[index] = 2

	for _, neigh := range adj[index] {
		if s.isCyclicUtil(neigh, adj, state) {
			return true
		}
	}

	state[index] = 1

	return false
}
```
  {{< /tab >}}
{{< /tabs >}}


