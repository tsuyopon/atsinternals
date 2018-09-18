# Interface: EventProcessor

EventProcessor is responsible for starting EThread and initializing each EThread.

The event system's main thread group (eventProcessor) is the core component of the event system.

- After startup, it is responsible for creating and managing thread groups, and threads in the thread group perform user-defined asynchronous tasks periodically or at specified times.
- With a set of dispatch functions provided by it, you can have a thread tweak the specified continuation. These function calls are not blocked.
- They return an Event object and schedule a callback continuation later or after a specific time or as soon as possible or at regular intervals.

Unique instance mode

- Every executable operation is provided to the EThread group via the global instance eventProcessor
- There is no need to create duplicate EventProcessor instances, so EventProcessor is designed as a unique instance pattern
- Note: Any function/method of EventProcessor is not reentrant

Thread group (event type):

- When EventProcessor starts, the first group of threads is assigned a specific id: ET_CALL
- Depending on the complexity of the different state machines or protocols, you must create additional threads and EventProcessor, you can choose:
   - Create a separate thread via spawn_thread
      - The thread is completely independent of the thread group
      - it will not exit until the end of the continuation
      - This thread has its own event system
   - Create a thread group via spawn_event_theads
      - You will get an id or Event Type
      - You can schedule continuation on this set of threads by this id or Event Type in the future

The code for the callback event:

- UNIX
   - Callback_event parameter is not used in all dispatch functions
   - When the callback is passed, the event code passed to the continuation handler is always EVENT_IMMEDIATE
- NT (the relevant code has been chopped off when it was open source)
   - The value of the event code passed to the continuation handler is provided by the callback_event parameter.

Event allocation strategy:

- Events are allocated and released by EventProcessor
- The state machine can access the returned, non-recurring event until it is cancelled or the callback of the event is completed
- For a recurring event, the event can be accessed until it is cancelled.
- Once the event is completed or canceled, eventProcessor will release it.

Scheduling method:

- schedule_imm
   - Schedule the continuation on a specific thread group, waiting for an event or timeout.
   - Request EventProcessor to immediately dispatch a callback for this continuation, which is handled by the thread in the specified event type.
- schedule_imm_signal
   - A function similar to schedule_imm, but only sends a signal to the thread at the end.
   - This function is actually designed for network modules.
      - A thread may always block on epoll_wait, by introducing a pipe or eventfd, when a thread is scheduled to execute an event, asynchronously notifying the thread to liberate from epoll_wait
      - Another function EThread with the same function::schedule_imm_signal          
- schedule_at
   - Request EventProcessor to dispatch a callback for this continuation at the specified time (absolute time), which is handled by the thread in the specified thread type.
- schedule_in
   - Request EventProcessor to dispatch a callback for this continuation within a specified time (relative time), which is handled by the thread in the specified thread type.
- schedule_every
   - Request EventProcessor to dispatch a callback for this continuation at a specified time, which is handled by the thread in the specified event type.
   - If the interval is negative, it means that this callback is executed at any time, and EventProcessor will dispatch whenever there is an opportunity.
      - Currently this type of scheduling is only one type of epoll_wait

Comparison with Event, EThread event dispatching methods:

```
                                              event
EventProcessor::schedule_*(cont, etype)  ---------------->  EThread[etype][RoundRobin]->QUEUE

                                              event
              EThread::schedule_*(cont)  ---------------->  this->QUEUE

                                              (self)
                    Event::schedule_*()  ---------------->  this->ethread->QUEUE
```

## definition

```
class EventProcessor : public Processor
{
public:
  / **
    Spawn an additional thread for calling back the continuation. Spawns
    a dedicated thread (EThread) that calls back the continuation passed
    in as soon as possible.

    @param cont continuation that the spawn thread will call back
      immediately.
    @return event object representing the start of the thread.

  * /
  // Create an immediate event (Conference) containing Cont
  // Then create an EThread of type DEDICATED and pass in the event (Event)
  // Then start EThread and return the event to the caller.
  // EThread will call Cont->handleEvent immediately after startup
  Event *spawn_thread(Continuation *cont, const char *thr_name, size_t stacksize = 0);

  / **
    Spawns a group of threads for an event type. Spawns the number of
    event threads passed in (n_threads) creating a thread group and
    returns the thread group id (or EventType). See the remarks section
    for Thread Groups.

    @return EventType or thread id for the new group of threads.

  * /
  // Create a thread group of number n_threads
  // The id of the thread group takes n_thread_groups and then increments its count
  EventType spawn_event_threads(int n_threads, const char *et_name, size_t stacksize);


  // The five schedule_*() methods that have been introduced above are omitted here.
  // Event *schedule_*();

  ////////////////////////////////////////////////////////////////////////////////////
  // reschedule an already scheduled event. //
  // may be called directly or called by    //
  // schedule_xxx Event member functions.   //
  // The returned value may be different    //
  // from the argument e.                   //
  ////////////////////////////////////////////////////////////////////////////////////
  // These reschedule methods have been deprecated and no calls or declarations have been seen in the entire ATS code.
  Event *reschedule_imm(Event *e, int callback_event = EVENT_IMMEDIATE);
  Event *reschedule_at(Event *e, ink_hrtime atimeout_at, int callback_event = EVENT_INTERVAL);
  Event *reschedule_in(Event *e, ink_hrtime atimeout_in, int callback_event = EVENT_INTERVAL);
  Event *reschedule_every(Event *e, ink_hrtime aperiod, int callback_event = EVENT_INTERVAL);

  // Constructor
  Event Processor ();

  / **
    Initializes the EventProcessor and its associated threads. Spawns the
    specified number of threads, initializes their state information and
    sets them running. It creates the initial thread group, represented
    by the event type ET_CALL.

    @return 0 if successful, and a negative value otherwise.

  * /
  // Initialize the unique instance eventProcessor and start the ET_CALL/ET_NET thread group
  int start(int n_net_threads, size_t stacksize = DEFAULT_STACKSIZE);

  / **
    Stop the EventProcessor. Attempts to stop the EventProcessor and
    all of the threads in each of the thread groups.

  * /
  // Stop the unique instance eventProcessor, but the function definition is empty
  virtual void shutdown();

  / **
    Allocates size bytes on the event threads. This function is thread
    safe.

    @param size bytes to be allocated.

  * /
  // Allocate space from the private data area of ​​the thread, allocated by atomic operations, so it is thread-safe.
  off_t allocate(int size);

  / **
    An array of pointers to all of the EThreads handled by the
    EventProcessor. An array of pointers to all of the EThreads created
    throughout the existence of the EventProcessor instance.

  * /
  // Each time a EThread of type REGULAR is created, a pointer to its instance is saved.
  EThread *all_ethreads[MAX_EVENT_THREADS];

  / **
    An array of pointers, organized by thread group, to all of the
    EThreads handled by the EventProcessor. An array of pointers to all of
    the EThreads created throughout the existence of the EventProcessor
    instance. It is a two-dimensional array whose first dimension is the
    thread group id and the second the EThread pointers for that group.

  * /
  // Each time a EThread of type REGULAR is created, a pointer to its instance is saved.
  // This is a two-dimensional array of pointers, grouped according to the function of EThread.
  EThread *eventthread[MAX_EVENT_TYPES][MAX_THREADS_IN_EACH_TYPE];

  // used for assign_thread, used to record the location of the last allocated thread to implement the RoundRobin algorithm
  unsigned int next_thread_for_type[MAX_EVENT_TYPES];
  // Record the current total number of each REGULAR thread group, for example: How many threads are there in the ET_CALL thread group?
  int n_threads_for_type[MAX_EVENT_TYPES];

  / **
    Total number of threads controlled by this EventProcessor.  This is
    the count of all the EThreads spawn by this EventProcessor, excluding
    those created by spawn_thread

  * /
  // There are a total of EThreads of type REGULAR, which do not contain EThread of type DEDICATED.
  int n_ethreads;

  / **
    Total number of thread groups created so far. This is the count of
    all the thread groups (event types) created for this EventProcessor.

  * /
  // How many sets of threads are currently available, for example: ET_CALL, ET_TASK, ET_SSL, etc.
  int n_thread_groups;

private:
  // prevent unauthorized copies (Not implemented)
  EventProcessor(const EventProcessor &);
  EventProcessor &operator=(const EventProcessor &);

public:
  / * ------------------------------------------------------ ------ * \
  | Unix & non NT Interface                                |
  \ * ------------------------------------------------------------------ ------ * /

  // schedule_*() the underlying implementation of the share
  Event *schedule(Event *e, EventType etype, bool fast_signal = false);
  // Select an EThread from the thread group of the specified type (etype) according to the RoundRobin algorithm, and return its pointer.
  // Usually used to allocate an EThread, and then pass the event (Event)
  EThread *assign_thread(EventType etype);

  // Manage the DEDIC private array of DEDICATED type
  // Each time a DEDIC of type DEDICATED is created, a pointer to its instance is saved in this array
  // increase the count of n_dthreads at the same time
  EThread *all_dthreads[MAX_EVENT_THREADS];
  int n_dthreads; // No. of dedicated threads

  // Record the allocation of the private data area of ​​the thread, by the use of atomic operation of allocate ()
  volatile int thread_data_used;
};

external inkcoreapi class EventProcessor eventProcessor;
```

The constructor performs a basic initialization of the EventProcessor, which is set to 0.

```
TS_INLINE
EventProcessor::EventProcessor() : n_ethreads(0), n_thread_groups(0), n_dthreads(0), thread_data_used(0)
{
  memset(all_ethreads, 0, sizeof(all_ethreads));
  memset(all_dthreads, 0, sizeof(all_dthreads));
  memset(n_threads_for_type, 0, sizeof(n_threads_for_type));
  memset(next_thread_for_type, 0, sizeof(next_thread_for_type));
}
```

## Creating a DEDICATED type of EThread

EventProcessor provides methods specifically for creating EThreads of type DEDICATED:

```
Event *
EventProcessor::spawn_thread(Continuation *cont, const char *thr_name, size_t stacksize)
{
  ink_release_assert(n_dthreads < MAX_EVENT_THREADS);
  Event *e = eventAllocator.alloc();

  // Create an immediate event (Conference) containing Cont
  e-> init (account, 0, 0);
  // Then create an EThread of type DEDICATED and pass in the event: oneevent=e
  all_dthreads[n_dthreads] = new EThread(DEDICATED, e);
  e->ethread = all_dthreads[n_dthreads];
  e->mutex = e->continuation->mutex = all_dthreads[n_dthreads]->mutex;
  n_dthreads++;
  // Start EThread,
  e->ethread->start(thr_name, stacksize);
  // In the analysis of EThread, you can see that the EDIC of type DEDICATED will call Cont->handleEvent immediately after startup.
  // Finally return the event (Event) to the caller.
  return e;
}
```

This method can be called with the globally unique instance eventProcessor.spawn_thread(). It is usually useless to create an event (Event) returned by EDIC of type DEDICATED and does not need to be saved.

## Create EThread of type REGULAR

EventProcessor provides a method for creating an EThread group of type REGULAR. This method creates a batch of threads at a time to implement concurrent events for a certain function:

```
EventType
EventProcessor::spawn_event_threads(int n_threads, const char *et_name, size_t stacksize)
{
  char thr_name[MAX_THREAD_NAME_LENGTH];
  EventType new_thread_group_id;
  int i;

  ink_release_assert(n_threads > 0);
  ink_release_assert((n_ethreads + n_threads) <= MAX_EVENT_THREADS);
  ink_release_assert(n_thread_groups < MAX_EVENT_TYPES);

  // Assign the thread group ID
  new_thread_group_id = (EventType)n_thread_groups;

  // Create an EThread instance of type REGULAR one by one through a for loop
  for (i = 0; i < n_threads; i++) {
    // Create an EThread instance and initialize tt=REGULAR, id=n_ethreads+i
    EThread *t = new EThread(REGULAR, n_ethreads + i);
    // Save the pointer of the EThread instance to the array, unified management
    all_ethreads[n_ethreads + i] = t;
    eventthread[new_thread_group_id][i] = t;
    // Set the type of event (Event) that EThread can handle
    t->set_event_type(new_thread_group_id);
  }

  // Set the number of threads in the thread group, note
  n_threads_for_type[new_thread_group_id] = n_threads;
  // Threads within a thread group by for loop
  for (i = 0; i < n_threads; i++) {
    snprintf(thr_name, MAX_THREAD_NAME_LENGTH, "[%s %d]", et_name, i);
    eventthread[new_thread_group_id][i]->start(thr_name, stacksize);
  }

  // created, accumulate thread group count
  n_thread_groups++;
  // Accumulate the total number of threads
  n_ethreads += n_threads;
  Debug("iocore_thread", "Created thread group '%s' id %d with %d threads", et_name, new_thread_group_id, n_threads);

  // Returns the thread group ID
  return new_thread_group_id;
}
```

This method can be called by the global unique instance eventProcessor.spawn_event_threads(). Usually, the EThread of type REGULAR is created to return the ID of the thread group, which needs to be saved.

## The start of the first EThread thread group ET_CALL/ET_NET (eventProcessor startup process)

At system startup, there is a global eventProcessor, which is the only instance of the Event System that starts up and starts running.

- A set of threads with type ET_CALL and name [ET_NET n] maintained in eventProcessor
- In ATS, thread groups of type ET_CALL or ET_NET are the same group
- The REGULAR part of EThread::execute() is executed inside each thread
- CPU affinity is set for each thread (CPU binding)
- But [ET_NET 0] is changed by the main() function at the end of thread->execute()
- The above process is implemented by eventProcessor.start(...)

```
int
EventProcessor::start(int n_event_threads, size_t stacksize)
{
  char thr_name[MAX_THREAD_NAME_LENGTH];
  int i;

  // do some sanity checking.
  // Prevent eventProcessor.start from being reentered by static variable started
  static int started = 0;
  ink_release_assert(!started);
  ink_release_assert(n_event_threads > 0 && n_event_threads <= MAX_EVENT_THREADS);
  started = 1;

  // Because it is the earliest thread created and started, it is initialized:
  // The current number of threads is n_event_threads
  // The current number of thread groups is 1
  n_ethreads = n_event_threads;
  n_thread_groups = 1;

  // Create an EThread instance, but note that the special handling when i==0 is for main() to become [ET_NET 0] at the end.
  for (i = 0; i < n_event_threads; i++) {
    EThread *t = new EThread(REGULAR, i);
    if (i == 0) {
      ink_thread_setspecific(Thread::thread_data_key, t);
      global_mutex = t->mutex;
      t->cur_time = ink_get_based_hrtime_internal();
    }
    all_ethreads[i] = t;

    eventthread[ET_CALL][i] = t;
    t->set_event_type((EventType)ET_CALL);
  }
  // Set the number of threads of type ET_CALL / ET_NET
  n_threads_for_type[ET_CALL] = n_event_threads;

  // The following process uses the HWLOC library to implement CPU affinity.
#if TS_USE_HWLOC
  // First determine the type of CPU affinity to support
  int affinity = 1;
  REC_ReadConfigInteger(affinity, "proxy.config.exec_thread.affinity");
  hwloc_obj_t obj;
  hwloc_obj_type_t obj_type;
  int obj_count = 0;
  char *obj_name;

  switch (affinity) {
  case 4: // assign threads to logical processing units
// Older versions of libhwloc (eg. Ubuntu 10.04) don't have HWLOC_OBJ_PU.
#if HAVE_HWLOC_OBJ_PU
    obj_type = HWLOC_OBJ_PU;
    obj_name = (char *)"Logical Processor";
    break;
#endif
  case 3: // assign threads to real cores
    obj_type = HWLOC_OBJ_CORE;
    obj_name = (char *)"Core";
    break;
  case 1: // assign threads to NUMA nodes (often 1:1 with sockets)
    obj_type = HWLOC_OBJ_NODE;
    obj_name = (char *)"NUMA Node";
    if (hwloc_get_nbobjs_by_type(ink_get_topology(), obj_type) > 0) {
      break;
    }
  case 2: // assign threads to sockets
    obj_type = HWLOC_OBJ_SOCKET;
    obj_name = (char *)"Socket";
    break;
  default: // assign threads to the machine as a whole (a level below SYSTEM)
    obj_type = HWLOC_OBJ_MACHINE;
    obj_name = (char *)"Machine";
  }

  // According to the type of CPU AF, determine how many logical CPUs
  obj_count = hwloc_get_nbobjs_by_type(ink_get_topology(), obj_type);
  Debug("iocore_thread", "Affinity: %d %ss: %d PU: %d", affinity, obj_name, obj_count, ink_number_of_processors());

#endif
  // Start each EThread with a for loop
  for (i = 0; i < n_ethreads; i++) {
    ink_thread tid;
    // Here is also a special treatment for [ET_NET 0]
    if (i > 0) {
      snprintf(thr_name, MAX_THREAD_NAME_LENGTH, "[ET_NET %d]", i);
      tid = all_ethreads[i]->start(thr_name, stacksize);
    } else {
      tid = ink_thread_self();
    }
    // Then set the CPU AF feature
#if TS_USE_HWLOC
    if (obj_count > 0) {
      obj = hwloc_get_obj_by_type(ink_get_topology(), obj_type, i % obj_count);
#if HWLOC_API_VERSION >= 0x00010100
      int cpu_mask_len = hwloc_bitmap_snprintf(NULL, 0, obj->cpuset) + 1;
      char * cpu_mask = (char *) alloca (cpu_mask_len);
      hwloc_bitmap_snprintf(cpu_mask, cpu_mask_len, obj->cpuset);
      Debug("iocore_thread", "EThread: %d %s: %d CPU Mask: %s\n", i, obj_name, obj->logical_index, cpu_mask);
#else
      Debug("iocore_thread", "EThread: %d %s: %d\n", i, obj_name, obj->logical_index);
#endif // HWLOC_API_VERSION
      hwloc_set_thread_cpubind(ink_get_topology(), tid, obj->cpuset, HWLOC_CPUBIND_STRICT);
    } else {
      Warning("hwloc returned an unexpected value -- CPU affinity disabled");
    }
#else
    // If there is no HWLOC library, then I have to do it.
    (Void) time;
#endif // TS_USE_HWLOC
  }

  Debug("iocore_thread", "Created event thread group id %d with %d threads", ET_CALL, n_event_threads);

  // Returns 0, accurately return ET_CALL
  // Actually the ID of the created thread group, because it is the first group, so it is 0, the same as spawn_event_threads()
  return 0;
}
```

- Then netProcessor starts, creating a NetAccept independent thread for each service port
  - Independent threads are implemented by the DEDICATED part of execute()

- NetAccept independent thread creates a netVC for each received sockfd
  - First set the netVC handler to acceptEvent
  - Then encapsulate netVC as an immediate execution of the Event type into the external queue of the eventProcessor


## Event scheduling between threads

When an event is placed in a thread group through the eventProcessor.schedule_*() method, how do you decide which thread (EThread) to handle?

This mechanism is relatively simple to implement. Each time an event is created, assign_thread() is called to get a pointer to EThread, and then the thread (EThread) handles the event. The specific implementation is as follows:

```
TS_INLINE EThread *
EventProcessor::assign_thread(EventType etype)
{
  int next;

  ink_assert(etype < MAX_EVENT_TYPES);
  // Remember the last allocated thread with the next_thread_for_type[] array
  // then select the next adjacent thread
  if (n_threads_for_type[etype] > 1)
    next = next_thread_for_type[etype]++ % n_threads_for_type[etype];
  else
    next = 0;
  // Get the pointer to the next EThread via the eventthread[etype][] array
  return (eventthread[etype][next]);
}
```

There is no lock on the operation of the next_thread_for_type[] array, which may cause competition problems, but it will not cause the program to crash. Since it is not very strict, the task allocation between threads is completely balanced, so there are not too many requirements here. .

## Thread private data space allocation management

Inside each EThread there is a thread_private array that stores the private data of the thread.

However, to ensure that each EThread is synchronized, all EThread threads must be synchronized. If you enter each EThread for allocation, you must call the allocation process multiple times. EventProcessor provides a method for allocating this space.

All thread_private areas of EThread are the same size, so you only need to specify an offset for each allocation. When each EThread is operated internally, accessing the thread_private area through this offset ensures one allocation. All EThreads are allocated. In order to ensure memory access efficiency, the method also performs memory alignment when allocating.

```
TS_INLINE off_t
EventProcessor::allocate(int size)
{
  // The offsetof(EThread, thread_private) method returns the relative offset of the thread_private member in class EThread
  // Align upwards with INK_ALIGN
  static off_t start = INK_ALIGN(offsetof(EThread, thread_private), 16);
  // Because memory alignment will waste a small amount of space at the beginning of thread_private, calculate the size of this space
  static off_t loss = start - offsetof(EThread, thread_private);
  // Then use INK_ALIGN to align the size of the space size to be allocated upwards
  size = INK_ALIGN(size, 16); // 16 byte alignment

  // Next is the atomic operation, allocate space
  // Use thread_data_used to record the memory space that has been allocated
  int old;
  do {
    old = thread_data_used;
    if (old + loss + size > PER_THREAD_DATA)
      // returns -1 when there is no space to allocate
      return -1;
  } while (!ink_atomic_cas(&thread_data_used, old, old + size));

  // The assignment is successful, returning the offset, at this time:
  // thread_data_used already points to the first address of the unallocated area
  // start is a static variable that always points to the aligned thread_private first address
  // old saves the value of the last thread_data_used
  return (off_t)(old + start);
}
```

The offset address is there, how to access data in this location?

- A pointer can be obtained by the macro definition ETHREAD_GET_PTR(thread, offset).
- Then initialize the address where the pointer is located
- Be sure to ensure the same size of the data type as when applying this memory area during initialization

Let's explain this process by taking the process of space allocation in the thread of the NetHandler instance as an example:

```
UnixNetProcessor::start(int, size_t)
{
...
  // allocate memory by allocate, get the offset
  netHandler_offset = eventProcessor.allocate (sizeof (NetHandler));
...
}

P_UnixNet.h::get_NetHandler(EThread *t)

// Encapsulate the operation to get NetHandler specifically
static inline NetHandler *
get_NetHandler(EThread *t)
{
  // Use the macro definition to construct a pointer
  return (NetHandler *)ETHREAD_GET_PTR(t, unix_netProcessor.netHandler_offset);                                                           
}

UnixNet.cc :: initialize_thread_for_net
void
initialize_thread_for_net(EThread *thread)
{
  // Here is a special new usage: new ( Buffer ) ClassName(Parameters), which means:
  // at the specified address: returned by get_NetHandler(thread)
  // Then call the constructor in NetHandler() to complete the initialization
  // ink_dummy_for_new is defined as an empty class in I_EThread.h
  new ((ink_dummy_for_new *)get_NetHandler(thread)) NetHandler();
...
}
// Any time you need to access, you can:
NetHandler * nh = get_NetHandler (ethread);
```

The data stored in the private data area of ​​the thread usually needs to be allocated once and then will not be released, for example:

- NetHandler
- PollCont
- udpNetHandler
- PollCont(udpNetInternal)
- RecRawStat

## References
[I_EventProcessor.h](http://github.com/opensource/trafficserver/tree/master/iocore/eventsystem/I_EventProcessor.h)

# 基类 Processor

EventProcessor inherits from Processor. In ATS's IO Core, Processor is defined for each function. These processors are inherited from the Processor base class, such as:

- tasksProcessor
- netProcessor
- sslProcessor
- dnsProcessor
- cacheProcessor
- clusterProcessor
- hostDBProcessor
- UDPNetProcessor(udpNet)

## definition

```
class Processor
{
public:
  virtual ~Processor();

  / **
    Returns a Thread appropriate for the processor.

    Returns a new instance of a Thread or Thread derived class of
    a thread which is the thread class for the processor.

    @param thread_index reserved for future use.

  * /
  virtual Thread *create_thread(int thread_index);

  / **
    Returns the number of threads required for this processor. If
    the number is not defined or not used, it is equal to 0.

  * /
  virtual int get_thread_count();

  / **
    This function attemps to stop the processor. Please refer to
    the documentation on each processor to determine if it is
    supported.

  * /
  virtual void
  shutdown()
  {
  }

  / **
    Starts execution of the processor.

    Attempts to start the number of threads specified for the
    processor, initializes their states and sets them running. On
    failure it returns a negative value.

    @param number_of_threads Positive value indicating the number of
        threads to spawn for the processor.
    @param stacksize The thread stack size to use for this processor.

  * /
  virtual int
  start(int number_of_threads, size_t stacksize = DEFAULT_STACKSIZE)
  {
    (void)number_of_threads;
    (void)stacksize;
    return 0;
  }

protected:
  Processor();

private:
  // prevent unauthorized copies (Not implemented)
  Processor(const Processor &);
  Processor &operator=(const Processor &);
};
```

## References
[I_Processor.h](http://github.com/opensource/trafficserver/tree/master/iocore/eventsystem/I_Processor.h)
