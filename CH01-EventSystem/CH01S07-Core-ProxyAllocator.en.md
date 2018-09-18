# Core part: ProxyAllocator

ProxyAllocator is the thread that the thread accesses ClassAllocator. As you can see from the name, this class is a Proxy implementation that accesses ClassAllocator.

This class itself is not responsible for allocating and freeing memory, but only maintains a list of available memory blocks of the upper ClassAllocator.

## Composition

ProxyAllocator
- The structure maintains a list of available "small pieces of memory" freelist
- and a counter that indicates how many elements in the list are located


## Design thinking

ATS is a multi-threaded system. Allocator and ClassAllocator are global data. If multi-threaded accesses global data, it must be locked and unlocked, so the efficiency will be lower.

So ATS designed a ProxyAllocator, which is the internal data structure of each Thread, which performs a Proxy operation on the access of Thread to the ClassAllocator global data.

- When the Thread requests memory, it calls the THREAD_ALLOC method (actually a macro definition) to determine if there is a memory block available in the ProxyAllocator.
   - if there is not
      - Call the corresponding Alloc method of ClassAllocator to apply for a block of memory
      - This operation will take an unallocated chunk of memory from the chunks of memory
   - If there is
      - Take one directly from ProxyAllocator, freelist points to the next available memory block, allocated--
- Thread calls the THREAD_FREE method when releasing memory (actually a macro definition)
   - Directly put the memory address into the freelist in ProxyAllocator, allocated++
   - Note: At this point, this memory is not marked as assignable in ClassAllocator

From the above process we can see that ProxyAllocator is constantly taking the global space as its own.

Then there will be a bad situation

- A Thread has a large amount of global space for itself when performing special operations
- Caused an imbalance in memory allocation between Threads

In this case, ATS also handled

- 参数：thread_freelist_low_watermark 和 thread_freelist_high_watermark
- Inside THREAD_FREE, a judgment is made: if the memory of the ProxyAllocator's freelist exceeds the High Watermark value
   - Just call the thread_freeup method
      - Remove consecutive nodes from the head of the freelist until the Lowlist only has Low Watermark nodes
      - Call ClassAllocator's free or free_bulk to mark nodes removed from the freelist as allocateable memory blocks

in conclusion

- Any internal element freelist that directly operates ProxyAllocator
   - There is no need to lock and unlock, because that is the internal data of Thread
- but need ClassAllocator to intervene
   - All need to be locked and unlocked
   - When the freelist of ProxyAllocator points to NULL
   - When allocated is greater than thread_freelist_high_watermark
- Via ProxyAllocator
   - Thread can have fewer resource lock conflicts when accessing resources in the global memory pool


## ProxyAllocator in the clever realization of the freelist list

Its definition is very simple

```
struct ProxyAllocator {
  int allocated;
  void *freelist;

  ProxyAllocator() : allocated(0), freelist(0) {}
};
```

So how do you implement a linked list operation through a freelist? Please refer to the following code:

```
  if (l.freelist) {                                                                                                
    void *v = (void *)l.freelist;
    l.freelist = *(void **)l.freelist;
    --(l.allocated);
    return v;
  }
```

```
#define THREAD_FREE(_p, _a, _t)                            \
  do {                                                     \
    *(char **)_p = (char *)_t->_a.freelist;                \                                                       
    _t->_a.freelist = _p;                                  \
    _t->_a.allocated++;                                    \
    if (_t->_a.allocated > thread_freelist_high_watermark) \
      thread_freeup(::_a, _t->_a);                         \
  } while (0)
```

As we said before, only the free memory block is saved in the freelist list.

- The data in the memory space in this list is useless.

In order to save memory usage, freelist stores the pointer address of the next memory block in the linked list in the freelist[0] location.

So we saw a very strange code:

```
l.freelist = *(void **)l.freelist;

Equivalent to

l.freelist = (void *)l.freelist[0];
```

```
*(char **)_p = (char *)_t->_a.freelist;

Equivalent to

(char *)_p[0] = (char *)_t->_a.freelist;
```

## References
- [I_ProxyAllocator.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_ProxyAllocator.h)

