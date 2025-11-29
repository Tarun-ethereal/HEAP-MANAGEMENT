# Fibonacci-Buddy Heap Management System

This repository presents a C implementation of a custom heap memory management system based on the **Fibonacci-Buddy System**. This project serves as an educational exploration into theoretical memory allocation models, demonstrating a sophisticated approach to managing dynamic memory with a focus on efficient allocation and deallocation, particularly through the merging of adjacent free blocks.

## Project Background

This project was an experimental endeavor to implement a highly theoretical model for heap management. The goal was to deeply understand the intricacies of memory allocation beyond standard library functions. It required extensive thought and careful implementation to translate a complex theoretical concept into a functional system. The Fibonacci-Buddy System, with its unique block splitting and merging rules based on Fibonacci numbers, offered a fascinating challenge that pushed the boundaries of my understanding of data structures and algorithms. This hands-on experience provided invaluable insights into low-level memory operations, debugging complex pointer logic, and optimizing performance in a constrained environment. The project, though academic, significantly deepened my appreciation for the fundamental principles of operating systems and memory management.

## Key Features

  * **Fibonacci-Buddy System:** Implements a memory allocation strategy where memory blocks are sized according to Fibonacci numbers. This system provides a unique alternative to traditional power-of-2 buddy systems, leading to different fragmentation characteristics.
  * **Simulated Malloc (`simulate_malloc`):** A custom memory allocation function that mimics `malloc`. It finds the smallest available Fibonacci-sized block that can accommodate the requested size, splitting larger blocks if necessary.
  * **Simulated Free (`simulate_free`):** A custom memory deallocation function that mimics `free`. It marks a block as free and attempts to merge it with its adjacent "buddy" block if both are free and form a larger Fibonacci-sized block.
  * **Doubly Linked Free List:** Maintains a doubly linked list of free memory blocks, sorted by memory address, to facilitate efficient searching for suitable blocks during allocation and seamless merging during deallocation.
  * **Dynamic Fibonacci Table Generation:** The system dynamically generates a Fibonacci sequence up to a predefined `HEAP_SIZE`, ensuring that all available block sizes conform to the Fibonacci series.
  * **Coalescing of Free Blocks:** A crucial aspect of the buddy system is the ability to merge adjacent free blocks (buddies) into a larger free block, combating external fragmentation and maximizing memory utilization.

## Technical Details & Implementation Nitpicks

### Memory Model and Constants

  * **`HEAP_SIZE`**: Defined as `1836311903` (a large Fibonacci number, $F\_{46}$), representing the total size of the simulated heap in bytes. This large size allows for a comprehensive test of the allocation and deallocation mechanisms.
  * **`MAX_FIB_COUNT`**: Set to `50`, which is the maximum number of Fibonacci numbers to pre-calculate, ensuring that the Fibonacci table covers all possible block sizes within the `HEAP_SIZE`.
  * **`heap_start`**: A global `void*` pointer that acts as the base address of the simulated memory region. This is where all allocations and deallocations occur.
  * **`simulated_heap_size`**: The actual usable size of the heap, which is the largest Fibonacci number less than or equal to `HEAP_SIZE`.

### BlockHeader Structure

Each memory block, whether free or allocated, is prefixed with a `BlockHeader` struct. This header contains critical metadata for managing the block:

```c
typedef struct BlockHeader {
    size_t size;          // Total size of the block (including header)
    size_t req_size;      // The original size requested by the user (excluding header)
    int fib_index;        // Index in the Fibonacci table corresponding to `size`
    int is_free;          // Flag: 1 if free, 0 if allocated
    struct BlockHeader *next; // Pointer to the next free block in the free list
    struct BlockHeader *prev; // Pointer to the previous free block in the free list
} BlockHeader;
```

  * **`size` vs. `req_size`**: It's important to distinguish between `size` (the actual Fibonacci-aligned size of the block, including the `BlockHeader`) and `req_size` (the size the user initially requested). This allows for internal fragmentation tracking and proper deallocation.
  * **`fib_index`**: This field is crucial for the Fibonacci-Buddy system. It indicates which Fibonacci number corresponds to the block's `size`, enabling the system to identify potential buddies for merging and to determine appropriate split sizes.

### Core Functions

1.  **`init_fib_table()`**:

      * Generates a sequence of Fibonacci numbers up to `HEAP_SIZE`.
      * The sequence starts with $F\_0 = 1$ and $F\_1 = 2$ (or $F\_1=1, F\_2=2$ depending on convention, but in this code, the first two are 1 and 2, and subsequent numbers are sums of the previous two).
      * This table is fundamental for determining valid block sizes and for the splitting and merging logic.

2.  **`init_heap()`**:

      * Allocates a large chunk of memory from the system using `malloc` to serve as the simulated heap.
      * Initializes a single `BlockHeader` at the beginning of this `heap_start` pointer, representing the entire available heap space.
      * This initial block is marked as free and added to the `free_list_head`. Its size is set to the largest Fibonacci number less than or equal to `HEAP_SIZE`.

3.  **`insert_free_block(BlockHeader *block)`**:

      * Adds a newly freed or split block into the `free_list_head`.
      * The free list is maintained in ascending order of memory addresses, which is vital for efficient buddy finding and merging.
      * It correctly handles insertion at the head, middle, or tail of the list.

4.  **`remove_free_block(BlockHeader *block)`**:

      * Removes a block from the `free_list_head`. This is called when a block is allocated or when it's merged into a larger block.
      * Properly updates `next` and `prev` pointers of adjacent blocks to maintain list integrity.

5.  **`simulate_malloc(size_t size)`**:

      * **Size Adjustment**: Calculates the `total_req` by adding `sizeof(BlockHeader)` to the user's requested `size`.
      * **Fibonacci Alignment**: Finds the smallest Fibonacci number (`fib_numbers[target_index]`) that is greater than or equal to `total_req`. This ensures that all allocated blocks are Fibonacci-sized.
      * **Best Fit Search**: Iterates through the `free_list_head` to find a free block whose `fib_index` exactly matches the `target_index`.
      * **Block Splitting**: If an exact match is not found, it searches for a larger free block (`current->fib_index > target_index`). If such a block is found, it is recursively split into two smaller Fibonacci-sized blocks (using the identity $F\_n = F\_{n-1} + F\_{n-2}$).
          * The left part (size $F\_{n-1}$) remains at the original address.
          * The right part (size $F\_{n-2}$) is placed immediately after the left part.
          * Both newly formed blocks are added back to the `free_list_head` (or one is allocated, and the other added to the free list). This process continues until a block of the `target_index` size is obtained or no suitable larger block can be split.
      * **Allocation**: Once a suitable block is found (either directly or through splitting), it's removed from the free list, marked as `is_free = 0`, its `req_size` is set, and a pointer to the user-usable memory (after the `BlockHeader`) is returned.

6.  **`simulate_free(void *ptr)`**:

      * **Header Retrieval**: Converts the user-provided `ptr` back to a `BlockHeader*` by subtracting `sizeof(BlockHeader)`.
      * **Mark as Free**: Sets `block->is_free = 1`.
      * **Buddy Coalescing (`try_merge`)**: This is the most complex part. It attempts to merge the freed block with its adjacent "buddy" block.
          * **Buddy Identification**: The buddy of a block is determined by its address and its Fibonacci size. For a block at address `A` with size $F\_n$, its buddy will be at either `A - F_{n-1}` or `A + F_{n-1}` (if it's the right buddy) or `A + F_{n-2}` (if it's the left buddy), depending on its position within the larger $F\_{n+1}$ block from which it was split.
          * **Merging Logic**: If the buddy is also free and has the correct Fibonacci index to form a larger Fibonacci block, both blocks are removed from the free list, and a new, larger `BlockHeader` is created at the lower address, representing the merged block. This new merged block is then re-inserted into the free list. This process is recursive, allowing multiple merges to occur until no more coalescing is possible.

7.  **`print_free_list()`**:

      * A utility function to display the current state of the free list, showing the address, size, and Fibonacci index of each free block. This is invaluable for debugging and visualizing memory state.

### Implementation Nitpicks & Considerations

  * **Pointer Arithmetic**: The code heavily relies on pointer arithmetic to navigate memory blocks and their headers. Careful attention is paid to casting `void*` to `BlockHeader*` and vice-versa, along with adding/subtracting `sizeof(BlockHeader)` to get to the actual user data region.
  * **Fibonacci Sequence Start**: The Fibonacci sequence used (1, 2, 3, 5, 8...) is a common variant for buddy systems. Consistency in its generation and use across allocation and deallocation is critical.
  * **Buddy Identification Complexity**: Identifying the correct "buddy" for merging in a Fibonacci-Buddy system is more complex than in a power-of-2 system. The `try_merge` function needs to deduce the buddy's address based on the current block's address and its Fibonacci index, considering whether it was the left or right part of a previous split.
  * **Edge Cases**: The code handles various edge cases, such as `simulate_malloc` failing due to insufficient memory or `simulate_free` being called on an invalid pointer (though robust error messages for invalid pointers are not explicitly shown in the provided snippet, they would be crucial in a production system).
  * **Overhead**: Each allocation carries the overhead of `sizeof(BlockHeader)`. For very small allocations, this overhead can be significant, leading to internal fragmentation.
  * **External Fragmentation**: While the coalescing mechanism helps reduce external fragmentation, it's not entirely eliminated. The Fibonacci-Buddy system aims to minimize it by ensuring that free blocks are merged whenever possible.
  * **Debugging**: The `print_free_list` function is a vital debugging tool for understanding the state of the heap and verifying that allocations, deallocations, and merges are occurring as expected.

## Technologies Used

  * **C:** The entire memory management system is implemented in C, providing low-level control over memory.
  * **Pointers and Structures:** Extensive use of pointers and custom `struct` definitions to manage memory blocks and linked lists.
  * **Algorithms:** Implementation of Fibonacci sequence generation, and custom allocation/deallocation algorithms.

## Getting Started

### Prerequisites

You need a C compiler (e.g., `gcc` or `clang`) to compile and run the source code.

### Installation and Compilation

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/your-username/fibonacci-buddy-heap.git
    cd fibonacci-buddy-heap
    ```
2.  **Compile the source code:**
    ```bash
    gcc -o heap_manager HEAP_MANAGEMENT_FINAL.C -std=c11
    ```

### Usage

1.  **Run the compiled executable:**
    ```bash
    ./heap_manager
    ```
2.  The program will prompt you to enter sizes for several memory allocations (`a`, `b`, `c`, `d`, `e`).
3.  It will then display the memory addresses of the allocated blocks and the state of the free list.
4.  Finally, it will demonstrate freeing these allocations and show the free list after deallocation, illustrating the merging process.

## License

This project is licensed under the MIT License - see the `LICENSE.md` file for details.

