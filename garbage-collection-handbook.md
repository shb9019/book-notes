# 3. Mark-Compact Garbage Collection

- Fragmentation - Problem for Mark Sweep
- With compaction, easy sequential allocation.
- Types of compaction
    - Arbitrary
    - Linearising
    - Sliding

## 3.1. Two Finger Compaction

- Two pass, arbitrary order
- Best suited for regions containing fixed size objects
- Algorithm
    1. First pass, relocate,
        1. Start with two pointers at heap start (free) and heap end (scan).
        2. Increment free till free space is found.
        3. Decrement scan till live object is found.
        4. Move live object at scan to space at free.
        5. Store address of free (final address of the live object) at scan.
        6. If free and scan pass each other, end.
    2. Second pass, updateReferences,
        1. Iterate through heap, and for each pointer above the place where scan and pass intersected, use target address at the old live object’s location.
- Simple and fast, no memory overhead. Destructively updates outdated locations.
- Supports interior pointers. Predictable memory access patterns.
- The arbitrary order of compaction leads to bad mutator locality.

## 3.2. The Lisp 2 algorithm

- Three passes; but cheap
- Fastest compaction algorithm, without considering cache and paging behavior
- Disadvantage: Requires full slot in every object header for forwarding address
- Steps:
    - computeLocations - Assign forwardingAddress to each live object, having target destination.
    - updateReferences - Update heap references in all roots and inside heap itself.
    - relocate - Moves each live object to its target.
- Requires 3 full passes over heap.

# 4. Copying garbage collection

- Main Disadvantage - Reduces available heap size by half
- In tight heaps (more live data compared to heap size), mark sweep performs much better. Copying collection has to collect more times due to reduced heap size.
- In large heaps, sequential allocation of copying collection has better mutator performance than mark sweep.

## 4.1 Semispace copying collection

- Two semispaces — fromSpace and toSpace
- Objects allocated in toSpace. If full, switch semispace roles and copy live objects in new fromSpace to toSpace.
- Algorithm,
    - Copy root objects in fromSpace to toSpace, and add to worklist.
    - Store address of toSpace object in its old fromSpace address.
    - While work list is not empty,
        - Pop item from work list for scanning.
        - For each pointer to fromSpace in item, assign it to the address of toSpace counterpart.
        - If the pointed object is not copied, copy it, and add to work list.
- Does not require extra space in object headers for forwarding address.
- Work List Implementations
    - Fenichel and Yochelson — Auxiliary stack
    - Chenney
        - Use scan pointer in toSpace for FIFO queue.
        - Space between scan and free pointers is work list. free is at end of copied objects. scan is at end of scanned objects.

## 4.2 Traversal order and locality

- Optimal layout of objects is NP complete.
- Breadth First traversal
    - Done by Chenney.
    - Predictable access patterns for GC.
    - Affects mutator locality because it tends to separate parents and children.
- Sliding
    - Best order for mark-compact, as it preserves layout of objects set by allocator.
- Depth first traversal
    - Keeps parents close to children, improving locality.
    - If cost of auxiliary stack is acceptable, Fenichel and Yochelson is DFS.
- Pseudo depth first traversal
    - Moon’s approximately depth first Algorithm,
    - Does not require an auxiliary stack. Instead, uses a second pointer called partialScan in addition to scan.
    - partialScan always finishes scanning the last unscanned page in toSpace, before scan progresses.
    - Can lead to about 30% toSpace being scanned twice by scan.
    - Called Hierarchical Decomposition
    - Works okay for tree-like data structures, disappointing performance otherwise.
- Huang et al’s online reordering algorithm
    - Processes all hot fields (frequently accessed) in work list before any cold fields.
    - Better performance than static reorderings.
- Some approaches use hierarchical decomposition for hot objects. Combining the two approaches has better performance than either.
- Shuf et al’s Offline profiling
    - Identifies prolific types, and allocates adjacent space for children when parent is created. Improving cache performance.

## 4.3 Issues to Consider

- Advantages: Fast allocation & No fragmentation
- Uses only half of available heap for allocation.
- Sequential allocation enables hardware/software prefetch optimizations.
- The cost difference between bump-a-pointer and free-list allocation may not be significant. As allocation counts for less than 10% of object creation time.
- Compacted heaps have good spacial locality.
- Copying collectors will perform more collections because it has only half the heap space available for allocation.
- In some environments (eg., with lack of type accuracy), copying is not possible.
- Non-moving collection needs to find at least one reference to a live object. Moving collection needs to find every pointer and rewrite them.

# 5. Reference counting

- An object is presumed to be live if and only if the no. of references to that object is greater than 0.
- Operates directly on objects instead of tracing and finding live objects.
- Naive algorithm
    - atomic write barrier (src, i, ref)
        - src → object, i → index of pointer in src, ref → where src[i] should point to
        - Increase reference count of ref
        - Reduce reference count of src[i]
        - If reference count of src[i] is 0, free itself and all children recursively

## 5.1 Advantages and disadvantages of reference counting

- Advantages
    - Promptness. Immediately collected after becoming garbage.
    - Distributed memory management cost. Objects are freed when they becomes garbage.
    - Does not need headroom in heap like tracing collectors.
    - Locality will not become worse.
- Disadvantages
    - Time overhead on mutator. Frequent writes lead to low throughput.
    - Reference count and pointer manipulations should be atomic. To prevent races.
    - Read only operations require stores to memory i.e., updating reference counts. (e.g., using thread variable to point to heap object). Pollutes cache.
    - Cannot reclaim cyclic garbage.
    - Reference count field must be pointer sized (to handle cases where all pointers in heap point to one object)
    - Recursive deletion of object may introduce pauses.

## 5.2 Improving efficiency

- Deferred reference counting - Defer identification of some garbage to a reclamation phase.
- Coalesced reference counting - Ignore all but first and last modification to a pointer field in a period.
- Buffered reference counting - Buffers all reference count increments and decrements for later processing.

## 5.3 Deferred reference counting

- Majority of pointer loads are to registers and stack variables (roots).
- If root pointer is being manipulated, unbarriered write i.e., remove ref count updates when pointers from roots are changing.
- Otherwise, update ref counts.
- If decrementing ref count in the latter case leads to ref count of 0, add object to zero count table (zct). These may still be alive due to unaccounted pointers from roots.
- Delete object for zct, if a new reference to the object is made.
- If allocate fails, stop-the-world collect.
    - ZCT object can only be live if there is direct reference from any root.
    - Increment ref count of each heap object directly pointed by roots.
    - Iterate over each object in zct. If an object still has ref count = 0, delete and free itself and its children.
    - Decrement ref count of heap objects directly point by roots.
- Can reduce pointer manipulation cost by 80% compared to Naive.

## 5.4 Coalesced reference counting

- In deferred, reference counts still need to be updated atomically for object fields.
- In any period for any object field, only the values at the start and end of the period need to be considered.
- Copy objects to thread local log before first modification in an epoch.
- When pointer of clean object is updated,
    - Save its address and values of pointer fields to local update buffer.
    - Mark object as dirty. Pointer to log entry is stored in object header.
    - New objects are also marked as dirty.
- Duplicates of same objects in multiple thread logs are allowed.
- At the end of epoch, stop all threads and collect,
    - Collect all thread local update buffers into collector log.
    - For each entry in collector log, if object is dirty,
        - Mark object as clean
        - Increment ref count of whichever objects the pointer fields are currently pointing to.
        - Decrement ref count of whichever objects the pointer fields were pointing to at the start of epoch, using the log entry.
        - If decremented ref count of object reaches 0, add to zero count table.
- Advantages — Removes most of mutator overhead compared to naive approach.
- Disadvantages — Has stop-the-world pauses, usually smaller than tracing collectors. Requires extra space for zct and update buffers.

## 5.5 Cycling reference counting

- Use cases: doubly linked lists, circular buffers, recursion.
- Simple approach is to combine reference counting with occasional tracing collection.
- Unsafe algorithm by several authors. Uses Strong Pointers → Never allowed to form cycles. Weak Pointers → Cycle closing pointers than form cycles. Algorithm may reclaim prematurely.
- Most widely adopted algo is Trial Deletion, Recycler algorithm.
    - Any garbage pointer structure can only have references from objects within the structure.
    - Garbage cycles can arise only from a pointer deletion that leaves a ref count > 0.
- Deleting any object such than ref count > 0 can lead to garbage cycle.
- Object is suspected to be garbage based on the 2nd point. By partial tracing, ref counts are recursively decremented for all references from object. We are trial deleting these pointers.
- If there is any object in structure with ref count > 0, pointer from outside structure. Not garbage. Restore ref counts by incrementing recursively.
- In algorithm,
    - black → not garbage / free space
    - purple → suspected to be garbage
    - grey → visited objects while partial tracing
    - white → garbage
- Complexity is O(N + E), where N is no. of nodes, and E is no. of edges.

## 5.6 Limited-field reference counting

- Common for some objects to be popular, require reference count field of pointer size.
- One option to set a maximum reference count that is sticky i.e., once it reaches max, it cannot be incremented or decrement.
- Backup tracing collector can be used to delete these references.

## 5.7 Issues to consider

- Works for environments where simple small objects are managed explicitly. Large complex objects are managed by reference counting.
- Programmer must ensure races between pointer modifications and reference count updates are avoided.
- Advanced reference counting algorithms require stop-the-world pauses, auxiliary data structures or need a backup tracing collector.
- However, these advanced solutions scale well and cost is proportional to pointer writes.

# 6. Comparing Garbage Collectors

- For each collector, there was at least one benchmark that would have been at least 15% faster with a more appropriate collector. There is no “best” collector for all circumstances.
- Relative performance of collectors varies by heap space available. Excessively large heaps may disperse temporally related objected, leading to worsened locality.

## 6.1 Throughput

- GC should account for only a small fraction of overall execution time.
- Explicit cost to mutator — Read and write barriers
- Implicit cost to mutator — Reference counting affecting cold objects, cache behavior of objects moved by GC.
- Reference counting
    - Has to synchronize all reference count updates. Alleviated by deferred and coalesced counting.
    - Touches cold objects for count updates.
- Mark-Sweep
    - Cost of tracing + sweep. Tracing is for live objects. Sweeping is for entire heap.
- Copying
    - Requires only tracing, but number of instructions per live object is higher than mark-sweep.
- Cost of chasing pointers is highest in mark-sweep and copying.

## 6.2 Pause time

- Extent to which GC interrupts program execution.
- Reference counting distributes cost of pause times throughout the program execution. However, removal of last reference leads to recursive freeing.

## 6.3 Space

- Reference counting has a per-object overhead to store reference count. It prevents garbage accumulation by immediately freeing. Unable to collect cycles, incomplete collector.
- Semispace copying collectors need additional heap space for a copy reserve.
- Non-moving collectors face the problem of fragmentation.
- Tracing collectors require marking stacks, mark bitmaps or other auxiliary data structures.
- Tracing collectors need sufficient room for garbage to avoid thrashing.
- Complete collector - Reclaims all dead objects eventually. Prompt collector - Reclaims all dead objects at each collection cycle.

## 6.4 Implementation

- Errors made by collector manifest when mutator follows an invalid reference.
- Tracing collectors have a simple interface between collector and mutator: call collector when allocation exhausts. Complex only to identify roots.
- A moving collector must identify *every* root and update the reference accordingly. A non-moving collector need *at least one* reference to each live object.
- Reference counting is tightly coupled to the mutator. Libraries exist for programmers to decide which objects should be managed by reference counting.
- Read/write barries and GC’s inner loops needs to be inlined without leading to large generated code size. Higher instructions can affect performance if they cannot fit in the instruction cache.

## 6.5 Adaptive Systems

- Some runtimes can switch collectors depending on available heap size.
- Hotspot collector tunes performance against user-supplied throughput and max pause time goals, adjusting the size of spaces within the heap.
- Measure application behavior, and lifetime distributions of the objects. Then experiment with different collector configurations.
