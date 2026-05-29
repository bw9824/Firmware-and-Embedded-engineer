# Leetcode-in-c

## Mark Down (.md) guide
https://www.markdownguide.org/basic-syntax/

## Code speed
```c =
#include <stdio.h>
#include <time.h>
int main() {

    clock_t start = clock();
    clock_t end = clock();

    // FUNCTION CODE

    double total = (double)(end-start) / CLOCKS_PER_SEC;
    printf("Processor time taken in function: %d\n", (double)(end-start)/CLOCKS_PER_SEC);
    return 0;    
}
```
## Time & Space Complexity
When developing for embedded systems, understanding [Time Complexity](#Time_Complexity) (how to calculate the efficiency/duration of program execution) and [Space Complexity](#Space_Complexity) (how much memory your code consumes) is arguably more critical than in pure software.

### Time_Complexity
Time complexity does **not** measure the exact seconds a program takes to run (since that changes with CPU frequency). Instead, it counts the **number of basic CPU operations** executed by the algorithm.

We express both complexities using **Big O Notation** ($O$), which describes how time or space requirements grow as the input size ($n$) scales toward infinity.

#### 1. Constant Time: $O(1)$
The execution time remains identical, regardless of data size.
- Embedded Example: Reading a hardware register, or popping an element from a fixed-size buffer.

```c
int get_sensor_data(int* buffer){
    return buffer[0]; // Always takes 1 operation
}
```
#### 2. Logarithmic Time: $O(\log n)$
The problem size is cut in half with every execution step.
- Embedded Example: Binary Search inside a sorted look-up table (LUT) of values.
```c
int binary_search(int arr[], int size, int target){
    int low = 0, high = size-1;
    while (low<=high){
        int mid = low+(high-low)/2;
        if (arr[mid] == target) return mid;
        if (arr[mid]<target) low = mid+1;
        else high = mid-1;
    }
    return -1;
}
```

#### 3. Linear Time: $O(n)$
The execution time grows proportionally to the input size $n$.
- Embedded Example: Implementing a manual memcpy, calculating a checksum (CRC) over a data packet, or scanning an array to find a maximum temperature reading.
```c
void clear_buffer(char* buf, int n){
    for (int i = 0; i<n; i++){
        buf[i] = 0;
    }
}
```

#### 4. Quadratic Time: $O(n^2)$
Typically involves nested loops. Execution time quadruples if the data size doubles.
- Embedded Example: Naive sorting algorithms like Bubble Sort used on raw sensor arrays.
```c
void bubble_sort(int arr[], int n){
    for (int i = 0; i<n-1; i++){
        for (int j = 0; j<n-i-1; j++){
            if (arr[j]>arr[j+1]){
                int temp = arr[j];
                arr[j] = arr[j+1];
                arr[j+1] = temp;
            }
        }
    }
}
```

### Space_Complexity
Space complexity measures the total amount of extra memory an algorithm needs to run relative to the input size $n$.

#### 1. Constant Space: $O(1)$
The memory used by the algorithm does not change as the input data grows. This is the gold standard for embedded firmware.
- Embedded Example: An in-place string reversal or an overlapping memcpy. It only creates a couple of temporary pointers regardless of copying 10 bytes or 10,000 bytes.
```c
void reverse_array(int arr[], int n){
    int temp;
    for (int i = 0; i<n/2; i++){
        temp = arr[i];
        arr[i] = arr[n-1-i];
        arr[n-1-i] = temp;
    }
}
```
#### 2. Linear Space: $O(n)$
The algorithm requires extra memory proportional to the size of the input.
- Embedded Example: Allocating a dynamic buffer via malloc(n) to duplicate a data stream before processing it. (Highly dangerous if memory allocation fails!).
```c//
int* process_data(int* data_in, int n){
    // Dangerous if n is large: Can trigger Heap Overflow
    int* temp_buffer = (int*)malloc(n * sizeof(int)); 
    return temp_buffer;
}
```
