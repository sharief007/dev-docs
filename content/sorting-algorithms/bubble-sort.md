---
title: 'Bubble Sort'
type: docs
toc: true
sidebar:
  open: true
prev: sorting-algorithms
next: selection-sort
params:
  editURL:
---

**Bubble sort** is a simple comparison-based sorting algorithm. It works by repeatedly swapping adjacent elements if they are in the wrong order. The pass through the list is repeated until the list is sorted.

### Example

Consider a sample array:  `[5, 4, 3, 2, 1]`:

```mermaid
graph LR
    A[5] --> B[4]
    B --> C[3]
    C --> D[2]
    D --> E[1]
```

Now, let's go through one pass of the bubble sort algorithm:

```mermaid
graph LR
    A[5] --> B[4]
    B --> C[3]
    C --> D[2]
    D --> E[1]

    style A fill: grey
    style B fill: grey
```

```mermaid
graph LR
    A[4] --> B[5]
    B --> C[3]
    C --> D[2]
    D --> E[1]

    style B fill: grey
    style C fill: grey
```

```mermaid
graph LR
    A[4] --> B[3]
    B --> C[5]
    C --> D[2]
    D --> E[1]

    style C fill: grey
    style D fill: grey
```

```mermaid
graph LR
    A[4] --> B[3]
    B --> C[2]
    C --> D[5]
    D --> E[1]

    style D fill: grey
    style E fill: grey
```

**After the first pass:**

```mermaid
graph LR
    A[4] --> B[3]
    B --> C[2]
    C --> D[1]
    D --> E[5]

    style E fill: green
```

**After the second pass:**

```mermaid
graph LR
    A[3] --> B[2]
    B --> C[1]
    C --> D[4]
    D --> E[5]

    style D fill: green
    style E fill: green
```

**After the third pass:**

```mermaid
graph LR
    A[2] --> B[1]
    B --> C[3]
    C --> D[4]
    D --> E[5]

    style C fill: green
    style D fill: green
    style E fill: green
```

**After the fourth pass:**

```mermaid
graph LR
    A[1] --> B[2]
    B --> C[3]
    C --> D[4]
    D --> E[5]

    style B fill: green
    style C fill: green
    style D fill: green
    style E fill: green
```

**After the fifth pass (final pass):**

```mermaid
graph LR
    A[1] --> B[2]
    B --> C[3]
    C --> D[4]
    D --> E[5]

    style A fill: green
    style B fill: green
    style C fill: green
    style D fill: green
    style E fill: green
```

### Flow chart

```mermaid
graph TB
  A[Start] --> B1((i=0, j=0))
  B1 --> B2{j < n-i-1}
  B2 -->|Yes| B3("Swap if arr[j] > arr[j+1]")
  B2 -->|No| B4(Increment j)
  B3 --> B4
  B4 -->|j < n-i-1| B2
  B4 -->|j = n-i-1| B5(Increment i, Reset j)
  B5 -->|i < n-1| B1
  B5 -->|i = n-1| C[End]

```

### Implementation

{{< tabs items="java, python, go" >}}
    {{< tab >}}
    ```java
    public class BubbleSort {
        public static void bubbleSort(int arr[]) {
            int n = arr.length;
            boolean swapped;
            
            for (int i = 0; i < n-1; i++) {
                swapped = false;
                for (int j = 0; j < n-i-1; j++) {
                    if (arr[j] > arr[j+1]) {
                        // Swap arr[j] and arr[j+1]
                        int temp = arr[j];
                        arr[j] = arr[j+1];
                        arr[j+1] = temp;
                        swapped = true;
                    }
                }
                // If no two elements were swapped, break the loop
                if (!swapped)
                    break;
            }
        }

        public static void main(String args[]) {
            int arr[] = {64, 34, 25, 12, 22, 11, 90};
            bubbleSort(arr);
            System.out.println("Sorted array:");
            for (int i : arr) {
                System.out.print(i + " ");
            }
        }
    }
    ```
    {{< /tab >}}
    {{< tab >}}
    ```python
    def bubble_sort(arr):
        n = len(arr)
        for i in range(n):
            # Initialize swapped to False
            swapped = False
            for j in range(0, n-i-1):
                if arr[j] > arr[j+1]:
                    # Swap arr[j] and arr[j+1]
                    arr[j], arr[j+1] = arr[j+1], arr[j]
                    swapped = True
            # If no two elements were swapped, break the loop
            if not swapped:
                break

    arr = [64, 34, 25, 12, 22, 11, 90]
    bubble_sort(arr)
    print("Sorted array:")
    print(arr)
    ```
    {{< /tab >}}
    {{< tab >}}
    ```go
    package main

    import "fmt"

    func bubbleSort(arr []int) {
        n := len(arr)
        for i := 0; i < n-1; i++ {
            swapped := false
            for j := 0; j < n-i-1; j++ {
                if arr[j] > arr[j+1] {
                    // Swap arr[j] and arr[j+1]
                    arr[j], arr[j+1] = arr[j+1], arr[j]
                    swapped = true
                }
            }
            // If no two elements were swapped, break the loop
            if !swapped {
                break
            }
        }
    }

    func main() {
        arr := []int{64, 34, 25, 12, 22, 11, 90}
        bubbleSort(arr)
        fmt.Println("Sorted array:")
        fmt.Println(arr)
    }
    ```
    {{< /tab >}}
{{< /tabs >}}


These implementations use an optimized version of the Bubble Sort algorithm with a flag (`swapped`) that helps break out of the loop early if the array is already sorted. This reduces the time complexity in cases where the array is mostly sorted.

### Complexity Analysis

For the optimized Bubble Sort implementation:

**Time Complexity**
- **Worst-case Time Complexity**: O(n^2)
- **Best-case Time Complexity**: O(n)
- **Average-case Time Complexity**: O(n^2)

**Space Complexity**

- **Space Complexity**: O(1)
