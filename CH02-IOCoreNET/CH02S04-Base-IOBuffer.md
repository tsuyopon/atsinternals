# Base component: IOBuffer

IOBuffer is the basic memory allocation and management system in ATS, just like the malloc() we use often.

In the macro definition DEFAULT_BUFFER_SIZES, ATS defines how many sizes of buf. The size of buf is defined by the following macro:

```
#define BUFFER_SIZE_INDEX_128 0
#define BUFFER_SIZE_INDEX_256 1
#define BUFFER_SIZE_INDEX_512 2
#define BUFFER_SIZE_INDEX_1K 3
#define BUFFER_SIZE_INDEX_2K 4
#define BUFFER_SIZE_INDEX_4K 5
#define BUFFER_SIZE_INDEX_8K 6
#define BUFFER_SIZE_INDEX_16K 7
#define BUFFER_SIZE_INDEX_32K 8
#define BUFFER_SIZE_INDEX_64K 9
#define BUFFER_SIZE_INDEX_128K 10
#define BUFFER_SIZE_INDEX_256K 11
#define BUFFER_SIZE_INDEX_512K 12
#define BUFFER_SIZE_INDEX_1M 13
#define BUFFER_SIZE_INDEX_2M 14
```

It can be seen that the minimum 128 bytes, the maximum 2M bytes, the above is just an index value, then how to calculate the number of bytes of buf according to the index value?

```
#define DEFAULT_BUFFER_BASE_SIZE 128
#define BUFFER_SIZE_FOR_INDEX(_i) (DEFAULT_BUFFER_BASE_SIZE * (1 << (_i)))
```
You can see that the calculation formula is: 128*2^idx, and the number of bytes corresponding to the index value is calculated by the macro BUFFER_SIZE_FOR_INDEX(_i).

In ATS, each length of buf has its own dedicated memory pool and allocator, defined by a global array ioBufAllocator[], DEFAULT_BUFFER_SIZES is the number of members of this array.

```
#define MAX_BUFFER_SIZE_INDEX 14
#define DEFAULT_BUFFER_SIZES (MAX_BUFFER_SIZE_INDEX + 1)
Inkcoreapi extern Allocator ioBufAllocator[DEFAULT_BUFFER_SIZES];
```

## Composition of the IOBuffer system

Memory management layer is implemented by IOBufferData and IOBufferBlock

  - Each IOBufferBlock has a pointer to IOBufferData
  - Each IOBufferBlock has a pointer to the next IOBufferBlock
  - When we mention IOBufferBlock, it usually refers to the linked list of the entire IOBufferBlock, not a block.

The data manipulation layer is implemented by IOBufferReader and MIOBuffer, so that there is no need to directly process the IOBufferBlock linked list and the IOBufferData structure.

  - IOBufferReader is responsible for reading the data in the IOBufferBlock (consumer)
  - MIOBuffer can write data to IOBufferBlock (producer)
  - At the same time MIOBuffer can also contain pointers to multiple IOBufferReaders
    - This allows you to record an in-memory data, multiple consumer information and consumption

The data access abstraction layer is implemented by MIOBufferAccessor

  - MIOBufferAccessor contains read and write operations for a specific IOBufferBlock
    - a pointer to IOBufferReader
    - A pointer to MIOBuffer

```
      IOBufferReader -----[entry.read]------+ 
        || || V
      Block mbuf MIOBufferAccessor
        || || |
        || MIOBuffer <----[mbuf.write]------+
        || ||
        || _writer
        || ||
      IOBufferBlock ===> IOBufferBlock ===> IOBufferBlock ===> NULL
           || || ||
         _data _data _data
           || || ||
      IOBufferData IOBufferData IOBufferData

|| or = means member reference
| or - means method path
```

in conclusion:

  - Create a MIOBuffer instance when we need a buffer to store the data
  - Then you can write data to MIOBuffer.
    - MIOBuffer will automatically create IOBufferBlock and IOBufferData
  - If you need to read data from the buffer
    - Can create IOBufferReader
    - or read directly through MIOBuffer
  - When calling do\_io\_read
    - Need to pass MIOBuffer in the past, because the data to be read is written to IOBuffer
  - When calling do\_io\_write
    - Just pass the IOBufferReader in the past, because you want to read the data inside the IOBuffer and send it out.
  - When you need to pass an IOBuffer to another method/function, but you are not sure whether the other operation is reading or writing
    - can pass MIOBufferAccessor instance
    - For example: VIO contains an instance of MIOBufferAccessor.

## Auto Pointer and Memory Management

The member ```Ptr<IOBufferBlock> _writer;``` in MIOBuffer points to the first node of the IOBufferBlock list.

Each time you write data to the MIOBuffer via the write method, the list is written. When a node is full, a node is added.

When we consume data through the IOBufferReader, the Ptr pointer automatically calls the node's free() method to release the space occupied by the node for the node that has already been consumed.

The free() method here uses IOBufferBlock::free(), which is defined as follows:

```
TS_INLINE void
IOBufferBlock::free()
{
  Dealloc();
  THREAD_FREE(this, ioBlockAllocator, this_thread());
}

TS_INLINE void
IOBufferBlock::dealloc()
{
  Clear();
}

TS_INLINE void
IOBufferBlock::clear()
{
  Data = NULL;
  IOBufferBlock *p = next;
  While (p) {
    Int r = p->refcount_dec();
    If (r)
      Break;
    Else {
      IOBufferBlock *n = p->next.m_ptr;
      P->next.m_ptr = NULL;
      P->free();
      p = n;
    }
  }
  Next.m_ptr = NULL;
  _buf_end = _end = _start = NULL;
}
```

Through the above source code, you can clearly see that the call flow is: free()-->dealloc()-->clear()-->free()-->...

Obviously this is a recursive call. In the implementation of clear, in order to avoid re-entry, it is necessary to avoid overloading the Ptr automatic pointer.

In the recursive call process, the depth of the recursion is limited by the length of the block list, and is also limited by the stack size. Once the stack space is exceeded, it will cause a Stack Overflow error.
PS: The above paragraph is an understanding and translation of the comments, but according to my analysis of the source code, this is not completely recursive.

So when we try to connect many small data blocks to the block linked list, we should consider connecting as large data blocks as possible. If we have to connect a small data block, we can consider summarizing multiple small data blocks by means of data copying. The data goes to a large block of data. This is why two write() methods are provided in MIOBuffer.
PS: Refer to row 876 and line 913 of I_IOBuffer.h.

### Cross-thread access to IOBuffer

Since the auto pointer will return the object to the freelist of the current thread when it is released, and this release is automatically performed based on the reference count.
If an IOBuffer object is operated across threads, it is likely to cause memory allocated in thread A. If the release operation is performed in thread B, then the memory is returned to thread B's freelist.

If thread A and thread B are not on the same NUMA node, this will cause a bit of performance problems with memory access.

Usually in the ATS code, the state machine that created the IOBuffer is always responsible for releasing and reclaiming it. So, when we see that MIOBuffer is created, we will (must) create an IOBufferReader, when the thread
When B needs to access the MIOBuffer, it is assigned an IOBufferReader. When thread B finishes processing the data, it returns to the state machine of thread A, and then consumes the data in MIOBuffer through the first created IOBufferReader object. The automatic pointer will release all IOBuffer objects in thread A. At this time, the state machine A is called the master state machine, and the state machine B is called the slave state machine. This is also the typical relationship between multiple state machines in the ATS. The state machine created later is always attached.

But sometimes, we need to pass data between two thread groups, and state machine A that initiates data transfer will die before state machine B that receives the data. State machine A creates the MIOBuffer and saves the data, which is then passed to state machine B, so state machine B becomes the owner of MIOBuffer. This inevitably requires the release of MIOBuffer by state machine B. This situation is also allowed in ATS, but it is not recommended. At this time, the state machine A and the state machine B are in an equal relationship, which is a rare case in the ATS. (At the moment, I have read a small part of the ATS code and have not yet encountered this pattern.)


### IOBufferBlock Recursive call analysis of the linked list release process (Is there really a Stack Overflow?)

First, suppose we have two IOBufferReaders for LA and LF respectively.

  - At this time, the reference counters of A, B, C, E, and F are all 1, and only the reference counter of D is 2.
  - The linked list of the corresponding IOBufferBlock is as follows:

```
             +-----+------+ +-----+------+ +-----++++-----+- -----+ +-----+------+
LA.block---->| A 1 | next |---->| B 1 | next |---->| C 1 | next |---->| D 2 | next |--- ->| E 1 | NULL |
             +-----+------+ +-----+------+ +-----++++-----+- -----+ +-----+------+
                                                                         ^
             +-----+------+ |
LF.block---->| F 1 | next |------------------------------------ ----------+
             +-----+------+
```

Then, we call IOBufferReader::clear() to completely release LA:

  - Block=NULL is executed in LA.clear();

Since the definition of block is: ```Ptr<IOBufferBlock> block;```

  - Therefore the equal sign in the above operation is overloaded by Ptr
  - Ptr assigns NULL to A.block
  - Ptr decrements the reference count of A to 1 and becomes 0.
  - When the reference count is 0, Ptr will call A.free()

```
LA.block---->NULL

             +-----+------+ +-----+------+ +-----++++-----+- -----+ +-----+------+
             | A 0 | next |---->| B 1 | next |---->| C 1 | next |---->| D 2 | next |---->| E 1 | NULL |
             +-----+------+ +-----+------+ +-----++++-----+- -----+ +-----+------+
                                                                         ^
             +-----+------+ |
LF.block---->| F 1 | next |------------------------------------ ----------+
             +-----+------+
```

After entering the A.free() method, it will be operated first by the A.clear() method.

  - A.data=NULL; Release IOBufferData
  - Since A.next points to B, if A is released, the reference count of B needs to be decremented by 1.
  - Then find that the reference count of B also becomes 0, and B also releases
  - Before recursively calling B.free()
     - First save C.m_ptr to n, so that you don't lose the address of C after releasing B.
     - Then point B.next to NULL, which is to break the pointer with C

```
LA.block---->NULL

             +-----+------+ +-----+------+ +-----++++-----+- -----+ +-----+------+
             | A 0 | next |---->| B 0 | NULL | | C 1 | next |---->| D 2 | next |---->| E 1 | NULL |
             +-----+------+ +-----+------+ +-----++++-----+- -----+ +-----+------+
                                                                         ^
             +-----+------+ |
LF.block---->| F 1 | next |------------------------------------ ----------+
             +-----+------+
```

After entering the B.free() method, it will first be operated by the B.clear() method.

  - B.data=NULL; Release IOBufferData
  - Since B.next has been pointed to NULL, skip the while part
  - Point B.next.m_ptr to NULL
  - Return B.free() from the B.clear() method and execute THREAD_FREE() to release B
  - Return to A.clear() to continue execution

```
LA.block---->NULL

             +-----+------+ +-----+------+ +-----++++-----+- -----+
             | A 0 | next |---->???? | C 1 | next |---->| D 2 | next |---->| E 1 | NULL |
             +-----+------+ +-----+------+ +-----++++-----+- -----+
                                                                         ^
             +-----+------+ |
LF.block---->| F 1 | next |------------------------------------ ----------+
             +-----+------+
```

Return to A.clear() again

  - Take the address of C back from n, assign it to p, so p=C, then continue the while loop
  - Actually, here you can think of A.next pointing to C, because A.clear() is the first while loop when p=A.next=B
  - Then reduce the reference to C by one, it will also become 0, and B will also release
  - Before recursively calling C.free()
     - First save D.m_ptr to n, so that you don't lose the address of D after releasing C.
     - Then point C.next to NULL, which is to break the pointer with D

```
LA.block---->NULL

             +-----+------+ +-----+------+ +-----++++-----+- -----+
             | A 0 | next |---->???? | C 0 | NULL | | D 2 | next |---->| E 1 | NULL |
             +-----+------+ +-----+------+ +-----++++-----+- -----+
                                                                         ^
             +-----+------+ |
LF.block---->| F 1 | next |------------------------------------ ----------+
             +-----+------+
```

After entering the C.free() method, it will first be operated by the C.clear() method.

  - C.data=NULL; release IOBufferData
  - Since C.next has been pointed to NULL, skip the while part
  - Point C.next.m_ptr to NULL
  - Return C.free() from the C.clear() method and execute THREAD_FREE() to release C
  - Return to A.clear() to continue execution

```
LA.block---->NULL

             +-----+------+ +-----+------+ +-----+++
             | A 0 | next |---->???? | D 2 | next |---->| E 1 | NULL |
             +-----+------+ +-----+------+ +-----+++
                                                                         ^
             +-----+------+ |
LF.block---->| F 1 | next |------------------------------------ ----------+
             +-----+------+
```

Return to A.clear() again

  - Take the address of D back from n, assign it to p, so p=D, then continue the while loop
  - Then decrement the reference of D by one, and find that the reference count of D does not become 0 but 1, indicating that there are other linked lists that use C, so they cannot continue to release C.
  - Point A.next.m_ptr to NULL
  - Return A.free() from the A.clear() method and execute THREAD_FREE() to release A

```
LA.block---->NULL

                                                                      +-----+------+ +-----+------+
                                                                      | D 1 | next |---->| E 1 | NULL |
                                                                      +-----+------+ +-----+------+
                                                                         ^
             +-----+------+ |
LF.block---->| F 1 | next |------------------------------------ ----------+
             +-----+------+
```

The example here uses a smaller list, but you can still see that the release process is a recursive process:

  - Because, after the current node is released, the reference to the next node needs to be decremented by 1
  - So, when releasing the current node, first look at the existence of the next node.
     - If present, the reference count for the next node is decremented by 1
     - If the reference count of the next node becomes 0 after decrementing by 1, then the element pointed to by the next node needs to be released first.
  - The process for releasing this element is as follows:
     - First, save the next node of the released element to n and point the next node to NULL, thus making the next node an orphan.
     - Then, release the next node via free()
     - Then, use n as the next node to loop the judgment.
  - So for a list of block=A[1]->B[1]->C[1]->D[2]->E[1]->NULL (the value of the reference counter is indicated in parentheses), The call stack when the entire linked list is free is as follows:

```
Block=NULL
A[1] ==> A[0]
A[0]->free()
   A[0]->dealloc()
      A[0]->clear()
         p=A[0].next.m_ptr=B[1].m_ptr;
         While(p) {
         B[1] ==> B[0];
         n=B[0].next.m_ptr=C[1].m_ptr;
         B[0].next.m_ptr=NULL;
         B->free()
            B->dealloc()
               B->clear()
               p=B[0].next.m_ptr=NULL;
               B[0].next.m_ptr=NULL;
            THREAD_FREE(B);
         p=n=C[1].m_ptr;
         }
         While(p) {
         C[1] ==> C[0];

         n=C[0].next.m_ptr=D[2].m_ptr;
         C[0].next.m_ptr=NULL;
         C->free()
            C->dealloc()
               C->clear()
               p=C[0].next.m_ptr=NULL;
               C[0].next.m_ptr=NULL;
            THREAD_FREE(C);
         p=n=D[2].m_ptr;
         }
         While(p) {
         D[2] ==> D[1];
         Break;
         }
         A[0].next.m_ptr=NULL;
   THREAD_FREE(A);
```

Therefore, if F.next points to D, the release process of LA.block above is B, C, A, and elements D, E will not be released.

During the release process, it is not completely recursively released. The first node is released at the end, and the remaining nodes are released one by one in the linked list order. The deepest call stack is 6 layers.

Since the three key functions all use the INLINE declaration, the actual call stack may be shallower. In the actual use of MIOBuffer, the Stack Overflow problem caused by the IOBufferBlock list is too long and the linked list is released.

## References
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)

## Basic components: IOBufferData

IOBufferData is the lowest-level memory unit that implements memory allocation and release, counting references.

  - Support for reference counters by inheriting RefCountObj.
  - To implement the reference counting function, when it is included by other classes, it is always defined as a smart pointer, such as: Ptr<IOBufferData> data;
  - The member variable \_data points to the memory space allocated by the global variable ioBufAllocator[].
  - The AllocType enumeration value defines the type of memory allocation used by objects managed by this object.
    - NO_ALLOC
    - FAST_ALLOCATED
    - XMALLOCED
    - MEMALIGNED
    - DEFAULT_ALLOC
    - CONSTANT
  - An instance can be created with new_IOBufferData()
  - An instance can be created with new_xmalloc_IOBufferData()
  - An instance can be created with new_constant_IOBufferData()
  - The memory space occupied by its instance is allocated by ioDataAllocator

### Definition

```
Class IOBufferData : public RefCountObj
{
Public:
  // return the allocated memory size
  Int64_t block_size();

  // Release the memory that was allocated by alloc before being managed by this class.
  Void dealloc();

  // According to size_index and type to allocate memory, if previously allocated, will first perform dealloc operation
  Void alloc(int64_t size_index, AllocType type = DEFAULT_ALLOC);

  // return _data member
  Char *
  Data()
  {
    Return _data;
  }

  // Overload char * operation, return _data member
  Operator char *() { return _data; }

  // Release the IOBufferData instance itself
  // First execute dealloc, then release the IOBuffeData object itself (reclaim memory resources via ioDataAllocator)
  // Therefore, after executing this method, you can no longer use and reference the object.
  Virtual void free();

  // represents the number of bytes in the memory block, as calculated by the formula 128*2^size_index
  Int64_t _size_index;

  // memory allocation type, AllocType enumeration value
  // NO_ALLOC indicates that it is currently unassigned and is used for delay allocation.
  AllocType _mem_type;

  // pointer to a memory block
  Char *_data;

#ifdef TRACK_BUFFER_USER
  Const char *_location;
#endif

  /**
    Constructor. Initializes state for a IOBufferData object. Do not use
    Use one of the functions with the 'new_' prefix instead.

  */
  // constructor, initialize IOBufferData
  // But don't use this method directly, if needed, get an instance via new_IOBufferData
  IOBufferData()
    : _size_index(BUFFER_SIZE_NOT_ALLOCATED), _mem_type(NO_ALLOC), _data(NULL)
#ifdef TRACK_BUFFER_USER
      ,
      _location(NULL)
#endif
  {
  }

Private:
  // declaration only
  IOBufferData(const IOBufferData &);
  IOBufferData &operator=(const IOBufferData &);
};

// Declare a ClassAllocator that globally assigns an instance of this type
Inkcoreapi extern ClassAllocator<IOBufferData> ioDataAllocator;
```

## Basic components: IOBufferBlock

IOBufferBlock is used to link multiple IOBufferData to form a larger storage unit for scalable memory management.

  - is a single linked list, member smart pointer next points to the next IOBufferBlock
  - Member smart pointer data is a memory block of type IOBufferData
  - Describe the starting position of the data in the IOBufferData memory block by member \_start
  - Describe the end position of the data in the IOBufferData memory block by member \_end
  - The end boundary of the IOBufferData memory block is described by the member \_buf\_end, and the free available space is the part between _end and _end_buf.
  - So an IOBufferBlock is a description of the use of IOBufferData, and it provides several ways to manipulate IOBufferData.
  - By connecting multiple IOBufferBlocks, you can combine data from multiple IOBufferData into larger bufs.
  - An instance of IOBufferBlock can be created with the global variable ioBlockAllocator
  - MIOBuffer By mounting an IOBufferBlock structure, you can know what is going on in the buffer (which part is in use and which part is available).
  - Cannot be shared between buffers. (The IOBufferBlock is not sharable between buffers)\*\*
  - An instance can be created with new_IOBufferBlock()
  - The memory space occupied by its instance is allocated by ioBlockAllocator

### Definition

```
Class IOBufferBlock : public RefCountObj
{
Public:
  // return a pointer to the underlying data block IOBufferData
  Char *
  Buf()
  {
    Return data->_data;
  }

  // returns a _start member variable pointing to the starting position of the data area being used
  Char *
  Start()
  {
    Return _start;
  }

  // returns a _end member variable pointing to the end of the data area being used
  Char *
  End()
  {
    Return _end;
  }

  // Returns the _buf_end member variable, a pointer to the end of the underlying data block
  Char *
  Buf_end()
  {
    Return _buf_end;
  }

  // return the value of _end - _start indicating the size of the data area being used
  Int64_t
  Size()
  {
    Return (int64_t)(_end - _start);
  }

  // returns the value of _end - _start, which indicates the length that can be read for a read operation, equivalent to size()
  Int64_t
  Read_avail()
  {
    Return (int64_t)(_end - _start);
  }

  // Returns the value of _buf_end - _end indicating the length of the write that can continue to be written, indicating the amount of space available
  Int64_t
  Write_avail()
  {
    Return (int64_t)(_buf_end - _end);
  }

  // Returns the size of the underlying data block, call the IOBufferData-> block_size () method
  Int64_t
  Block_size()
  {
    Return data->block_size();
  }

  // consume a certain amount of data from the head of the IOBufferBlock buffer, _start += len
  // After the application consumes data from the head of the IOBufferBlock buffer, this method is called to mark the memory space of this part of the buffer can be recycled.
  Void consume(int64_t len);

  // Append a certain byte of data to the end of the IOBufferBlock buffer, _end+=len
  // After the application appends the data to the end of the IOBufferBlock buffer, this method is called to mark the stock of the actual readable data in the buffer.
  // First, use end() to get the end of the current data area, then copy the data to the end, then call this method
  // Note that the length of the copied data is len <= write_avail()
  Void fill(int64_t len);

  // Reset the data area being used, _start=_end=buf(), _buf_end=buf()+block_size()
  // After reset, read_avail()==0, write_avail()==block_size()
  Void reset();

  // Clone IOBufferBlock, but the underlying data block IOBufferData will not be cloned, so the cloned IOBufferBlock instance references the same underlying data block.
  // Note that the clone_avail()==0 of the cloned IOBufferBlock instance is buf_end=end
  IOBufferBlock *clone();

  // Clear the underlying data fast, pay attention to the difference with reset ()
  // This operation only disconnects the current block from the underlying data block by data=NULL. Whether the underlying data block is released/recovered is determined by its reference count.
  // Since IOBufferBlock is a linked list, recursively reduces the reference count of next. If it is reduced to 0, it also calls free() to release the block pointed to by next.
  // Finally, data=_buf_end=_end=_start=next=NULL
  // In fact, you can think of clear as a destructor, except that you don't release the block itself.
  // If you need to reuse this block, you can redistribute the underlying data block with alloc()
  Void clear();

  // Allocate a buffer with a length index of i to data and initialize it with the reset() method.
  Void alloc(int64_t i = default_large_iobuffer_size);

  // Directly call the clear () method
  Void dealloc();

  // IOBufferData and the Block instance to establish a reference relationship
  // can specify only part of Data by len and offset
  // _start=buf()+offset,_end=_start+len,_buf_end=buf()+block_size()
  // Note: The source code has an old bug that was discovered when writing this note, TS-3754
  Void set(IOBufferData *d, int64_t len = 0, int64_t offset = 0);
  // Create an IOBufferData instance with no immediate allocation of memory blocks by internal call
  // Then assign the allocated memory pointer to the IOBufferData instance, the other is the same as the set
  Void set_internal(void *b, int64_t len, int64_t asize_index);
  
  // Copy the current data to b, then call dealloc() to release the data block, then call set_internal()
  // Finally let the size() of the new data block match the original data block: _end = _start + old_size
  Void realloc_set_internal(void *b, int64_t buf_size, int64_t asize_index);
  // Same as: realloc_set_internal(b, buf_size, BUFFER_SIZE_NOT_ALLOCATED)
  Void realloc(void *b, int64_t buf_size);
  // allocate a buffer b via ioBufAllocator[i].alloc_void() and then call realloc_set_internal()
  Void realloc(int64_t i);
  
  // xmalloc allocation mode, no analysis**
  Void realloc_xmalloc(void *b, int64_t buf_size);
  Void realloc_xmalloc(int64_t buf_size);

  // Release the IOBufferBlock instance itself
  // First call dealloc, then reclaim memory through ioBlockAllocator.
  Virtual void free();

  // Point to the first byte that can be read in the data area
  Char *_start;
  // Point to the first byte that can be written in the data area, and the next byte of the last readable byte
  Char *_end;
  // Point to the last position of the entire data area, the boundary, this position can not write data
  Char *_buf_end;

#ifdef TRACK_BUFFER_USER
  Const char *_location;
#endif

  // Point to the smart pointer of type IOBufferData, the above _start, _end, _buf_end pointer range falls within its member _data
  // If you change the pointer to data, you must reset _start, _end, _buf_end
  Ptr<IOBufferData> data;

  // To form a block list, next is a smart pointer to the next block.
  Ptr<IOBufferBlock> next;

  // constructor, initialize IOBufferBlock
  // But don't use this method directly, if necessary, get an instance via new_IOBufferBlock
  IOBufferBlock();

Private:
  IOBufferBlock(const IOBufferBlock &);
  IOBufferBlock &operator=(const IOBufferBlock &);
};

// Declare a ClassAllocator that globally assigns an instance of this type
Extern inkcoreapi ClassAllocator<IOBufferBlock> ioBlockAllocator;
```

## Basic components: IOBufferReader

IOBufferReader

  - Do not rely on MIOBuffer
    - When multiple readers read data from the same MIOBuffer, each reader needs to mark where they started reading, how much data is read in total, how much is currently read
    - At this point you cannot directly modify the pointers in MIOBuffer, but describe them by IOBufferReader
  - Used to read a set of IOBufferBlocks.
    - When creating an IOBufferReader via MIOBuffer, it is a member of the IOBufferBlock that is copied directly from MIOBuffer
    - So it is actually reading the IOBufferBlock
  - IOBufferReader indicates where the consumer of a given buffer data starts reading data.
  - Provides a unified interface for easy access to data contained within a set of IOBufferBlocks.
  - IOBufferReader internally encapsulates the logic for automatically removing data blocks.
    - The block=block->next operation in the consume method causes the reference count of the block to change, and the automatic release of Block and Data is achieved by the automatic pointer Ptr.
  - Simply put: IOBufferReader uses multiple IOBufferBlock single-chain tables as a large buffer, providing a set of methods for reading/consuming this buffer.
  - Internal member smart pointer block points to the first element of a singly linked list of multiple IOBufferBlocks.
  - The internal member mbuf refers back to the MIOBuffer instance that created this IOBufferReader instance.

It seems that IOBufferReader can directly access the IOBufferBlock linked list, but its internal member mbuf determines that it is designed for MIOBuffer.

### Definition

```
Class IOBufferReader
{
Public:
  // Returns the starting position of the data area available for consumption (read) (assisted by the member start_offset)
  // returns NULL to indicate that there are no associated data blocks
  Char *start();

  // Returns the end position of the available (read) data in the first data block (IOBufferBlock)
  // returns NULL to indicate that there are no associated data blocks
  Char *end();

  // Returns the length of the remaining (read) data in all current data blocks
  // Traverse all the data blocks, accumulate the available data length in each data block and subtract the start_offset representing the length of the consumed data.
  Int64_t read_avail();

  // Returns whether the remaining data length of all current data blocks can be consumed (read) is greater than size
  Bool is_read_avail_more_than(int64_t size);

  // Returns the number of data blocks that can be consumed (read) in all current data blocks
  // As the data is consumed (read):
  // The member block points to the next IOBufferBlock in the linked list one by one.
  // The member start_offset will also be reset according to the information of the new block.
  Int block_count();

  // Returns the length of the data available for consumption (read) in the first data block
  Int64_t block_read_avail();

  // Skip unwanted blocks based on the value of start_offset
  // The value of start_offset must be in the range [ 0, block->size() )
  Void skip_empty_blocks();

  // Clear all member variables, IOBufferReader will not be available
  Void clear();

  // Reset the state of IOBufferReader, member mbuf and accessor will not be reset
  // Only initialize the block, start_offset, size_limit three members
  Void reset();

  // Record consumption n bytes of data, n must be less than read_avail ()
  // When you consume, the automatic pointer block will point to block->next one by one
  // This function is only to record the state of consumption, the specific data read operation, still need to access the member mbuf, or the underlying data block in the block to carry out
  Void consume(int64_t n);

  // clone the current instance, copy the current state and point to the same IOBufferBlock, and the same start_offset value
  // by calling mbuf->clone_reader(this) directly
  IOBufferReader *clone();

  // Release the current instance, then you can no longer use the instance.
  // by calling mbuf->dealloc_reader(this) directly
  Void dealloc();

  // return block member
  IOBufferBlock *get_current_block();

  // Whether the current remaining writable space is in the low limit state
  // Directly call mbuf->current_low_water()
  // The judgment rules are as follows:
  // The space that the member mbuf of the current MIOBuffer type can write without adding a block.
  // return true if it is below the water_mark value, false otherwise
  // return mbuf->current_write_avail() <= mbuf->water_mark;
  Bool current_low_water();

  // Whether the current remaining writable space is in the low limit state
  // Directly call mbuf->low_water()
  // Similar to current_low_water(), but when returning true, it means appending an empty block to the mbuf
  // return mbuf->write_avail() <= mbuf->water_mark;
  Bool low_water();

  // Is the currently readable data above the low limit?
  // return read_avail() >= mbuf->water_mark;
  Bool high_water();

  // Perform memchr on the IOBufferBlock list, but it seems that this method is not used in ATS
  // returns -1 to indicate that c was not found, otherwise it returns the offset value that c first appears in the IOBufferBlock list.
  Inkcoreapi int64_t memchr(char c, int64_t len = INT64_MAX, int64_t offset = 0);

  // Copy the len length data from the current IOBufferBlock list to buf, buf must allocate space in advance
  // If len exceeds the currently readable data length, the length of the currently readable data is used as the len value
  // Consume the block that has been read by consume()
  // Returns the actual copied data length
  Inkcoreapi int64_t read(void *buf, int64_t len);

  // From the current position of the current IOBufferBlock linked list, offset offset bytes, copy len length data to buf
  // But unlike read(), this operation does not execute the consume() operation
  // returns a pointer to the end of the data that memcpy writes to buf; if no write occurs, it is equal to buf
  // For example, when the offset exceeds the maximum value of the currently readable data, the return value is equal to buf
  Inkcoreapi char *memcpy(const void *buf, int64_t len = INT64_MAX, int64_t offset = 0);

  /**
    Subscript operator. Returns a reference to the character at the
    Specify position. You must ensure that it is within an appropriate
    Range.

    @param i positions beyond the current point of the reader. It must
      Be less than the number of the bytes available to the reader.

    @return reference to the character in that position.

  */
  // Override the subscript operator reader[i], you can use IOBufferReader just like an array
  // Need to be careful not to let i exceed the limit, otherwise it will throw an exception
  // There is no const char used to declare the type of the return value, but we should still only read the data from the IOBufferReader.
  // Although there is no hard limit to write to it, in fact, the code for reader[i] = x; should not be used here.
  Char &operator[](int64_t i);

  // return member mbuf
  MIOBuffer *
  Writer() const
  {
    Return mbuf;
  }
  // When an IOBufferReader has been associated with a MIOBuffer, it will set the mbuf to point to the MIOBuffer
  // This method to determine whether the current IOBufferReader has been associated with MIOBuffer
  // return the MIOBuffer pointer associated with it, or NULL means not associated
  MIOBuffer *
  Allocated() const
  {
    Return mbuf;
  }

  // If associated with MIOBufferAccessor, point to MIOBufferAccessor
  MIOBufferAccessor *accessor; // pointer back to the accessor

  // Point to the MIOBuffer that assigns this IOBufferReader
  MIOBuffer *mbuf;
  // The smart pointer block points to the chain header of the IOBufferBlock in the MIOBuffer at the initial situation.
  // With the use of consume(), block points to block->next one by one
  Ptr<IOBufferBlock> block;

  // start_offset is used to mark the offset position of the currently available data and the block member
  // every time block=block->next, start_offset will be recalculated
  // Normally start_offset will not be larger than the available data length of the current block
  // If it is exceeded, skip the useless block and correct the value in skip_empty_blocks()
  Int64_t start_offset;
  
  // It seems to be useless. It is set to INT64_MAX during initialization and reset().
  // Other changes to size_limit are judged not equal to INT64_MAX to modify
  Int64_t size_limit;

  // Constructor
  IOBufferReader() : accessor(NULL), mbuf(NULL), start_offset(0), size_limit(INT64_MAX) {}
};
```

## Basic components: MIOBuffer

MIOBuffer

  - It is a single write (producer), multiple read (consumer) memory buffer.
  - Is the center of all IOCore data transmission.
  - It is a data buffer for VConnection receive and send operations.
  - It points to a set of IOBufferBlocks, which in turn point to the IOBufferData structure containing the actual data.
  - MIOBuffer allows one producer and multiple consumers.
  - The speed at which it writes (produces) data depends on the slowest data read (consumer).
  - It supports automatic flow control between multiple (speed) readers of different speeds.
  - The data in the IOBuffer is immutable and cannot be modified once written.
  - One-time release after all reading (consumption) is completed.
  - Since data (IOBufferData) can be shared between buffers
    - Multiple IOBufferBlocks may reference the same data (IOBufferData),
    - But only one has ownership and can write data. If you allow data in the IOBuffer to be modified, it will cause confusion.
  - Member IOBufferReader readers[MAX_MIOBUFFER_READERS] defines multiple reads (consumers) with a default maximum of 5.
  - An instance can be created with new\_MIOBuffer() and free\_MIOBuffer(mio) destroys an instance.
  - The memory space occupied by its instance is allocated by ioAllocator

### Definition

```
#define MAX_MIOBUFFER_READERS 5

Class MIOBuffer
{
Public:
  // MIOBuffer write operation

  // Expand the len bytes of the data area currently in use (increasing the amount of readable data and reducing the writable space)
  // Directly operate the IOBufferBlock linked list _writer member, if the current remaining writable space is insufficient, will automatically append empty blocks
  Void fill(int64_t len);

  // Append block to IOBufferBlock list _writer member
  // Block *b must be current MIOBuffer writable, other MIOBuffer can not be written
  Void append_block(IOBufferBlock *b);

  // Append an empty block of the specified size to the IOBufferBlock list _writer member
  Void append_block(int64_t asize_index);

  // Append the current MIOBuffer default size empty block to the IOBufferBlock list _writer member
  // directly called append_block(size_index)
  Void add_block();

  /**
    Adds by reference len bytes of data pointed to by b to the end
    Of the buffer. b MUST be a pointer to the beginning of block
    Allocated from the ats_xmalloc() routine. The data will be deallocated
    By the buffer once all readers on the buffer have consumed it.

  */
  Void append_xmalloced(void *b, int64_t len);

  /**
    Adds by reference len bytes of data pointed to by b to the end of the
    Buffer. b MUST be a pointer to the beginning of block allocated from
    ioBufAllocator of the corresponding index for fast_size_index. The
    Data will be deallocated by the buffer once all readers on the buffer
    Have consumed it.

  */
  Void append_fast_allocated(void *b, int64_t len, int64_t fast_size_index);

  // Write data of length nbytes bytes in rbuf to IOBufferBlock member_writer
  // The return value is the number of bytes actually written.
  // Watermark and size_limit are not detected, but the block is appended when the remaining writable space is insufficient.
  // If flow control is required, the caller needs to implement it himself. This method does not implement flow control.
  // PS: write is a polymorphic definition, there is also a definition of write below
  Inkcoreapi int64_t write(const void *rbuf, int64_t nbytes);

#ifdef WRITE_AND_TRANSFER
  // Basically the same as write() below
  // But the write permission of the last cloned block is transferred from the MIOBuffer referenced by IOBufferReader *r to the current MIOBuffer
  // PS: Each time you append to the block list, the _writer member will point to the newly added block because the _writer member always points to the first writable block.
  Inkcoreapi int64_t write_and_transfer_left_over_space(IOBufferReader *r, int64_t len ​​= INT64_MAX, int64_t offset = 0);
#endif

  // After skipping the data of the offset byte started by IOBufferReader, copy the current block by clone().
  // Append the new block to the IOBufferBlock member_writer via append_block(),
  // Until the processing of the length len data (or the remaining data /INT64_MAX) is completed.
  // The return value is the number of bytes actually written.
  // The following is a direct translation of the comments in the source code:
  // Watermark and size_limit are not detected, but the block is appended when the remaining writable space is insufficient.
  // If flow control is required, the caller needs to implement it himself. This method does not implement flow control.
  // Even if only 1 byte of data is available in a block, the entire block is cloned.
  // Therefore, care must be taken when transferring data between two MIOBuffers,
  // Especially when the source MIOBuffer data is read from the network, because the received data may be small data blocks.
  // Since the release of the block is a recursive process, when there are too many blocks in the linked list, if the release may cause the call stack to overflow (there should be an error here, see the above analysis)
  // When using the write() method, it is recommended that the caller use flow control to avoid too many blocks in the linked list, and to control the minimum number of bytes used to transfer data using clones.
  // If you are experiencing the transfer of a large number of small blocks, it is recommended to use the first write() method to copy the data instead of using the clone block.
  // PS: Does not modify the data consumption of IOBufferReader *r.
  Inkcoreapi int64_t write(IOBufferReader *r, int64_t len ​​= INT64_MAX, int64_t offset = 0);

  // Transfer the block list in IOBufferReader *r to the current MIOBuffer
  // IOBufferReader *r The original block link will be emptied
  // Returns the length of all transferred data
  // Basically think this is a concat operation
  Int64_t remove_append(IOBufferReader *);

  // return the first writable block
  // Usually _writer points to the first writable block,
  // But in a very special case, the current block is filled, then block->next is the first writable block
  // If NULL is returned, there is no writable block
  IOBufferBlock *
  First_write_block()
  {
    If (_writer) {
      If (_writer->next && !_writer->write_avail())
        Return _writer->next;
      Ink_assert(!_writer->next || !_writer->next->read_avail());
      Return _writer;
    } else
      Return NULL;
  }

  // return the buf() of the first writable block
  Char *
  Buf()
  {
    IOBufferBlock *b = first_write_block();
    Return b ? b->buf() : 0;
  }
  // return the buf_end() of the first writable block
  Char *
  Buf_end()
  {
    Return first_write_block()->buf_end();
  }
  // return the start() of the first writable block
  Char *
  Start()
  {
    Return first_write_block()->start();
  }
  // return the end() of the first writable block
  Char *
  End()
  {
    Return first_write_block()->end();
  }

  // Returns the remaining writable space of the first writable block
  Int64_t block_write_avail();

  // Returns the remaining writable space of all blocks
  // This operation does not append an empty block to the end of the list
  Int64_t current_write_avail();

  // Returns the remaining writable space of all blocks
  // If: the readable data is less than the watermark value and the remaining writable space is less than or equal to the watermark value
  // Then automatically append an empty block to the end of the list
  // The return value also contains the space of the newly added block
  Int64_t write_avail();

  // Returns the default size used by the MIOBuffer to request a block
  // This value is passed in by the constructor when it is created in MIOBuffer and is stored in the member size_index.
  Int64_t block_size();

  // Same as block_size, it seems to be deprecated, not seen in the code.
  Int64_t
  Total_size()
  {
    Return block_size();
  }

  // Returns true if the readable data length exceeds the watermark value
  Bool
  High_water()
  {
    Return max_read_avail() > water_mark;
  }

  // return true if the writable space is less than or equal to the water_mark value
  // This method uses write_avail to get the size of the current remaining writable space, so an empty block may be appended
  Bool
  Low_water()
  {
    Return write_avail() <= water_mark;
  }

  // return true if the writable space is less than or equal to the water_mark value
  // This method does not append an empty block
  Bool
  Current_low_water()
  {
    Return current_write_avail() <= water_mark;
  }
  
  // According to the value of size to choose a minimum value, used to set the member size_index
  // E.g:
  // When size=128, select size_index to 0, and the calculated block size=128
  // When size=129, select size_index to be 1, and the calculated block size=256
  Void set_size_index(int64_t size);


  // MIOBuffer read operation
  
  // Assign an available IOBufferReader from the readers[] member, and set the accessor member of the Reader to point to anAccessor
  // A Reader must be associated with MIOBuffer, if a MIOBufferAccessor is associated with MIOBuffer,
  // Then the IOBufferReader associated with MIOBuffer must also be associated with MIOBufferAccessor
  IOBufferReader *alloc_accessor(MIOBufferAccessor *anAccessor);

  // Assign an available IOBufferReader from the readers[] member
  // Unlike alloc_accessor, this method points the accessor member to NULL
  // When a MIOBuffer is created, at least one IOBufferReader must be created.
  // If the IOBufferReader is created after the data is filled in the MIOBuffer, the IOBufferReader may not be able to read from the beginning of the data.
  // PS: Since the automatic pointer Ptr is used, _writer always points to the first writable block.
  // Then with the movement of the _writer pointer, the earliest created block will lose its referrer and will be automatically released by Ptr
  // So after creating MIOBuffer, first create an IOBufferReader,
  // to ensure that the earliest created block is referenced, so that it will not be automatically released by Ptr
  IOBufferReader *alloc_reader();

  // Assign an available IOBufferReader from the readers[] member
  // Then copy from r member block, start_offset, size_limit to the newly allocated IOBufferReader
  // But note that r must be one of the readers[] members, otherwise there will be problems with accessing the block.
  IOBufferReader *clone_reader(IOBufferReader *r);

  // release IOBufferReader, but e must be one of the readers[] members
  // If e is associated with an accessor, then e->accessor->clear() is called, so the accessor cannot read or write to MIOBuffer.
  // Then call e->clear(), you must set mbuf=NULL in clear
  Void dealloc_reader(IOBufferReader *e);

  // release each member of readers[] one by one
  Void dealloc_all_readers();


  // MIOBuffer settings / initialization section
  
  // Initialize MIOBuffer with a pre-allocated data block
  // and point the already assigned IOBufferReader to this data block as well
  Void set(void *b, int64_t len);
  Void set_xmalloced(void *b, int64_t len);
  // Create an empty memory block with the specified size_index value, initialize MIOBuffer
  // and point the already allocated IOBufferReader to this empty memory block as well
  // will also update the MIOBuffer member size_index with the new size_index value
  Void alloc(int64_t i = default_large_iobuffer_size);
  Void alloc_xmalloc(int64_t buf_size);
  // append IOBufferBlock *b to _writer->next and let _writer point to b
  // Note that this operation will cause the reference count of _writer->next to decrease.
  // If b->next is not NULL, iterate over _writer until _writer->next == NULL
  Void append_block_internal(IOBufferBlock *b);
  // Write the string s ending in \0 or \n to the current data block, the maximum length len, the write content includes \0 or \n
  // Returns 0, if the current data block has insufficient remaining writable space
  // returns -1 if \0 or \n is still not found within the len length
  // Returns the length of the actual write, when successful
  // PS: len contains 1 byte of \0 or \n
  Int64_t puts(char *buf, int64_t len);


  // The following are for internal use only

  // Determine whether it is empty MIOBuffer, there is no block
  Bool
  Empty()
  {
    Return !_writer;
  }
  Int64_t max_read_avail();

  // Traverse the number of blocks per reader in the readers[] and find the maximum value
  Int max_block_count();
  
  // According to the water_mark value, determine whether you need to add a block
  // append if needed
  // Related logic, see: Understanding water_mark
  Void check_add_block();

  // Get the first writable block
  IOBufferBlock *get_current_block();

  // Call block and all reader's reset() to reset MIOBuffer
  Void
  Reset()
  {
    If (_writer) {
      _writer->reset();
    }
    For (int j = 0; j < MAX_MIOBUFFER_READERS; j++)
      If (readers[j].allocated()) {
        Readers[j].reset();
      }
  }

  // Initialize the assigned reader
  // Point the reader's block to the IOBufferBlock list referenced by _writer
  Void
  Init_readers()
  {
    For (int j = 0; j < MAX_MIOBUFFER_READERS; j++)
      If (readers[j].allocated() && !readers[j].block)
        Readers[j].block = _writer;
  }

  // release block and all readers
  Void
  Dealloc()
  {
    _writer = NULL;
    Dealloc_all_readers();
  }

  // Clear the status of all members, first call dealoc (),
  // Then set size_index to BUFFER_SIZE_NOT_ALLOCATED, water_mark = 0
  Void
  Clear()
  {
    Dealloc();
    Size_index = BUFFER_SIZE_NOT_ALLOCATED;
    Water_mark = 0;
  }

  // Reassign a larger memory block to the current block
  Void
  Realloc(int64_t i)
  {
    _writer->realloc(i);
  }
  
  // Reassign a larger memory block to the current block,
  // Then copy the contents of the memory block pointed to by b to the newly allocated memory block
  Void
  Realloc(void *b, int64_t buf_size)
  {
    _writer->realloc(b, buf_size);
  }
  Void
  Realloc_xmalloc(void *b, int64_t buf_size)
  {
    _writer->realloc_xmalloc(b, buf_size);
  }
  Void
  Realloc_xmalloc(int64_t buf_size)
  {
    _writer->realloc_xmalloc(buf_size);
  }

  // Record the index value of the current default memory block size
  Int64_t size_index;

  // Decide when to stop writing and reading
  // For example: How many bytes need to be read at least to trigger the upper state machine
  // Need more free space to allocate new blocks, etc.
  // PS: needs to be implemented through the upper state machine
  Int64_t water_mark;

  // Point to the first IOBufferBlock that can write data
  Ptr<IOBufferBlock> _writer;
  // Save the IOBufferReader that can read this MIOBuffer. The default value is up to 5
  IOBufferReader readers[MAX_MIOBUFFER_READERS];

#ifdef TRACK_BUFFER_USER
  Const char *_location;
#endif

  // Constructor
  // Initialize the MIOBuffer with a pre-allocated memory block and set the water_mark value, the size_index value is set to the unassigned state
  MIOBuffer(void *b, int64_t bufsize, int64_t aWater_mark);
  // Only set the size_index value, but do not immediately allocate the underlying block
  // When writing data, it automatically allocates blocks according to the size_index value.
  MIOBuffer(int64_t default_size_index);
  // size_index value is set to unassigned state
  MIOBuffer();
  
  // Destructor
  // its definition is equal to dealloc(), releasing block and all readers
  ~MIOBuffer();
};
```

###方法

Describe in detail two write methods to avoid confusion (most of the content comes from direct translation of source comments)

  - inkcoreapi int64_t write(const void *rbuf, int64_t nbytes);
    - Copy nbytes bytes from rbuf to the current IOBufferBlock
    - If the current IOBufferBlock is full, add one more to the end and continue to write nbytes
  - inkcoreapi int64_t write(IOBufferReader *r, int64_t len = INT64_MAX, int64_t offset = 0);
    - Copy the available data contained in r, copy it by clone method, and append it to the current IOBufferBlock list.
    - have to be aware of is:
      - The IOBufferBlock obtained by the clone method directly references IOBufferData, and specifies the referenced data in IOBufferData via \_start and \_end.
      - Even if only 1 byte is referenced, a complete IOBufferBlock instance will be cloned and it will not be able to write any data.
      - So if you continue to append data, a new block is created and appended to the end of the list.
    - This method iterates through all the readable blocks contained in the IOBufferBlock list in r, and clones one by one, but does not modify the read status of r.
    - Because it is a recursive call process when releasing the IOBufferBlock linked list,
      - This will cause the overlay of the call stack. If the IOBufferBlock list is too long, it will cause the call stack to overflow**
    - So when using the write method,
      - When writing a large number of smaller bytes of data blocks, IOBufferReader should be avoided as much as possible, as this will result in an unreasonable increase in the IOBufferBlock list.
      - When you expect to write multiple consecutive lengths of data, consider using the first write method to reduce the length of the IOBufferBlock list.

### Understanding water_mark

It can be used

  - Perform an action when you need to read a certain amount of data or if the available data is below a certain amount
  - For example, when the ``` readable data amount ``` and the ``` writable space ``` are both smaller than this value, a new block is added to the MIOBuffer. Reference: check_add_block()
  - For example, we can set the data to be read from the disk when the available data in the buf is lower than the water_mark value. This can reduce the disk I/O. Otherwise, the amount of data read each time is very small, which is very wasteful of disk I/O. Scheduling.
  - The default is 0

## Basic components: MIOBufferAccessor

MIOBufferAccessor

  - IOBuffer read (consumer), write (producer) package
  - It encapsulates MIOBuffer and IOBufferReader

### Definition

```
Struct MIOBufferAccessor {
  // return IOBufferReader
  IOBufferReader *
  Reader()
  {
    Return entry;
  }

  // return MIOBuffer
  MIOBuffer *
  Writer()
  {
    Return mbuf;
  }

  // Returns the default size of the MIOBuffer when creating the block (read-only)
  Int64_t
  Block_size() const
  {
    Return mbuf->block_size();
  }

  // Same as block_size()
  Int64_t
  Total_size() const
  {
    Return block_size();
  }

  // Use IOBufferReader * abuf to initialize the current accessor read and write
  Void reader_for(IOBufferReader *abuf);
  // Use MIOBuffer * abuf to initialize the current accessor read and write
  Void reader_for(MIOBuffer *abuf);
  // Use MIOBuffer *abuf to initialize the current accessor's write
  // The current accessor is unreadable
  Void writer_for(MIOBuffer *abuf);

  // Clear the association with MIOBuffer and IOBufferReader
  Void
  Clear()
  {
    Mbuf = NULL;
    Entry = NULL;
  }

  // Constructor
  // function with clear()
  MIOBufferAccessor()
    :
#ifdef DEBUG
      Name(NULL),
#endif
      Mbuf(NULL), entry(NULL)
  {
  }

  // Destructor
  // empty function
  ~MIOBufferAccessor();

#ifdef DEBUG
  Const char *name;
#endif

Private:
  MIOBufferAccessor(const MIOBufferAccessor &);
  MIOBufferAccessor &operator=(const MIOBufferAccessor &);

  // Point to MIOBuffer for write operations
  MIOBuffer *mbuf;
  // point to the IOBufferReader for read operations
  IOBufferReader *entry;
};
```

## References
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)
- [P_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_IOBuffer.h)
- [IOBuffer.cc]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/IOBuffer.cc)
