---
title: 'Counting Sort'
weight: 8
type: docs
toc: true
sidebar:
  open: true
prev: radix-sort
next:
params:
  editURL:
---

**Counting Sort** is a non-comparison based sorting algorithm that works efficiently when you have a limited range of input values. It counts the frequency of each element and uses that information to place them in the correct sorted position. This makes it a highly efficient sorting method for scenarios with a small range of possible values.

{{< tabs items="java, python, go" >}}
{{< tab >}}
```java
import java.util.Arrays;

public class CountingSort {

    public static void countingSort(int[] arr, int max) {
        int[] countArray = new int[max + 1]; // Create a count array to store frequencies

        // Step 1: Count the frequency of each element
        for (int num : arr) {
            countArray[num]++;
        }

        // Step 2: Calculate cumulative frequencies
        for (int i = 1; i <= max; i++) {
            countArray[i] += countArray[i - 1];
        }

        int[] outputArray = new int[arr.length];

        // Step 3: Place elements in output array
        for (int num : arr) {
            outputArray[countArray[num] - 1] = num;
            countArray[num]--;
        }

        // Step 4: Copy sorted array back to original array
        System.arraycopy(outputArray, 0, arr, 0, arr.length);
    }

    public static void main(String[] args) {
        int[] arr = {4, 2, 2, 8, 3, 3, 1, 7, 6, 5, 7};

        int max = Arrays.stream(arr).max().getAsInt(); // Find maximum value in the array

        countingSort(arr, max);

        System.out.println("Sorted Array: " + Arrays.toString(arr));
    }
}

```
{{< /tab >}}
{{< tab >}}
```python
def counting_sort(arr, max_val):
    count_array = [0] * (max_val + 1)

    # Step 1: Count the frequency of each element
    for num in arr:
        count_array[num] += 1

    # Step 2: Calculate cumulative frequencies
    for i in range(1, max_val + 1):
        count_array[i] += count_array[i - 1]

    output_array = [0] * len(arr)

    # Step 3: Place elements in output array
    for num in arr:
        output_array[count_array[num] - 1] = num
        count_array[num] -= 1

    return output_array

arr = [4, 2, 2, 8, 3, 3, 1, 7, 6, 5, 7]
max_val = max(arr)

sorted_arr = counting_sort(arr, max_val)
print(f"Sorted Array: {sorted_arr}")

```
{{< /tab >}}
{{< tab >}}
```go
package main

import "fmt"

func countingSort(arr []int, max int) []int {
	countArray := make([]int, max+1)

	// Step 1: Count the frequency of each element
	for _, num := range arr {
		countArray[num]++
	}

	// Step 2: Calculate cumulative frequencies
	for i := 1; i <= max; i++ {
		countArray[i] += countArray[i-1]
	}

	outputArray := make([]int, len(arr))

	// Step 3: Place elements in output array
	for _, num := range arr {
		outputArray[countArray[num]-1] = num
		countArray[num]--
	}

	return outputArray
}

func main() {
	arr := []int{4, 2, 2, 8, 3, 3, 1, 7, 6, 5, 7}
	max := 8 // Assuming the maximum value is known

	sortedArr := countingSort(arr, max)
	fmt.Printf("Sorted Array: %v\n", sortedArr)
}

```
{{< /tab >}}
{{< /tabs >}}