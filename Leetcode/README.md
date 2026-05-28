# Leetcode-in-c

#### Mark Down (.md) guide
https://www.markdownguide.org/basic-syntax/

#### Code speed
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
