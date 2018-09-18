#芯部件:Thread & EThread

EThread inherits from Thread, first look at the definition of Thread:

## definition

```
class Thread
{
public:
  / * -------------------------------------------------------------- * \
  | Common Interface                            |
  \ * ------------------------------------------------------------- * /

  / **
    System-wide thread identifier. The thread identifier is represented
    by the platform independent type ink_thread and it is the system-wide
    value assigned to each thread. It is exposed as a convenience for
    processors and you should not modify it directly.

  * /
  ink_thread tid;

  / **
    Thread lock to ensure atomic operations. The thread lock available
    to derived classes to ensure atomic operations and protect critical
    regions. Do not modify this member directly.

  * /
  ProxyMutex *mutex;

  // PRIVATE
  void set_specific();
  Thread();
  virtual ~Thread();

  static ink_hrtime cur_time;
  // static variable, initialized to init_thread_key()
  inkcoreapi static ink_thread_key thread_data_key;
  Ptr<ProxyMutex> mutex_ptr;

  // For THREAD_ALLOC
  ProxyAllocator eventAllocator;
  ProxyAllocator netVCAllocator;
  ProxyAllocator sslNetVCAllocator;
  ProxyAllocator httpClientSessionAllocator;
  ProxyAllocator httpServerSessionAllocator;
  ProxyAllocator hdrHeapAllocator;
  ProxyAllocator strHeapAllocator;
  ProxyAllocator cacheVConnectionAllocator;
  ProxyAllocator openDirEntryAllocator;
  ProxyAllocator ramCacheCLFUSEntryAllocator;
  ProxyAllocator ramCacheLRUEntryAllocator;
  ProxyAllocator evacuationBlockAllocator;
  ProxyAllocator ioDataAllocator;
  ProxyAllocator ioAllocator;
  ProxyAllocator ioBlockAllocator;

private:
  // prevent unauthorized copies (Not implemented)
  Thread(const Thread &);
  Thread &operator=(const Thread &);

public:
  ink_thread start(const char *name, size_t stacksize = DEFAULT_STACKSIZE, ThreadFunction f = NULL, void *a = NULL);

  virtual void
  execute()
  {
  }

  static ink_hrtime get_hrtime();
};

extern Thread *this_thread();
```

## Method

### Thread::Thread()

  - used to initialize mutex
  - and locked
  - Threads always keep their own mutex locks
  - This way, if you reuse a thread mutex object, it is equivalent to being permanently locked by the thread, becoming a local object that belongs only to this thread.

```
Thread::Thread()
{
  mutex = new_ProxyMutex();
  mutex_ptr = mutex;
  MUTEX_TAKE_LOCK(mutex, (EThread *)this);
  mutex->nthread_holding = THREAD_MUTEX_THREAD_HOLDING;
}
```

### Thread::set_specific()

```
// source: iocore/eventsystem/P_Thread.h
TS_INLINE void
Thread::set_specific()
{
  Associate the current object (this) with the key
  ink_thread_setspecific(Thread::thread_data_key, this);
}
```

## About this_thread()

  - Returns the current Thread object
  - Since EThread inherits from Thread,
    - Calling this_thread() in EThread returns an EThread object
    - but the type of the return value is still Thread *
    - In order to provide the correct return value type, the this_ethread() method is defined in EThread

```
  // The following is a comment in I_Thread.h
  The Thread class maintains a thread_key which registers *all*
  the threads in the system (that have been created using Thread or
  a derived class), using thread specific data calls.  Whenever, you
  call this_thread() you get a pointer to the Thread that is currently
  executing you.  Additionally, the EThread class (derived from Thread)
  maintains its own independent key. All (and only) the threads created
  in the Event Subsystem are registered with this key. Thus, whenever you
  call this_ethread() you get a pointer to EThread. If you happen to call
  this_ethread() from inside a thread which is not an EThread, you will
  get a NULL value (since that thread will not be  registered with the
  EThread key). This will hopefully make the use of this_ethread() safer.
  Note that an event created with EThread can also call this_thread(),
  in which case, it will get a pointer to Thread (rather than to EThread).
  
  // Translate as follows:
  The Thread class contains a thread_key (member thread_data_key), all threads are thread-specific data related
  The call is registered to the system with thread_key (including all created Thread types and instances of their inherited classes).
  
  Any time you call this_thread() you get a pointer to the Thread instance running the current code.

  // The mechanism described in this comment is not reflected in the code, probably due to the deletion of the code, now there is no ---------
  Additionally, the EThread class (derived from Thread)
  maintains its own independent key.
  In addition, the EThread class (inherited from the Thread class) maintains a separate thread_key that belongs to itself.
  // ------------------------------------------------------------------------ --------------------------
  
  All threads created in EventSystem are registered with this (same) thread_key.
  
  // The mechanism described in this comment is not reflected in the code, probably due to the deletion of the code, now there is no ---------
  If you happen to call 
  this_ethread() from inside a thread which is not an EThread, you will
  get a NULL value (since that thread will not be  registered with the
  EThread key). This will hopefully make the use of this_ethread() safer.
  If you call this_ethread() from a thread that is not EThread, you get NULL.
  Because this thread is not registered with the EThread key.
  We hope to make the use of this_ethread() safer.
  // ------------------------------------------------------------------------ --------------------------

  Note that this_thread() can be called in EThread, but what gets is a pointer to the Thread type.
  Pointer (not the EThread type).
  Note: From the current code, you can convert to EThread by type, which is done in this_ethread().

```

Using the same key, the data obtained in different threads is different. Similarly, the bound data is only associated with the current thread.
Therefore, calls to thread specific related functions are closely related to the thread they are in.

```
TS_INLINE Thread *
this_thread()
{
  Retrieve previously associated objects by key
  return (Thread *)ink_thread_getspecific(Thread::thread_data_key);
}
```

### About this_ethread()

```
// source: iocore/eventsystem/P_UnixEThread.h
TS_INLINE EThread *
this_ethread()
{
  Call this_thread() directly and type conversion
  return (EThread *)this_thread();
}
```

### Thread::start()

Thread start

  - Before each thread is created, the set_specific() method is called to bind the thread data.

```
// source: iocore/eventsystem/Thread.cc
static void *
spawn_thread_internal(void *a) 
{
  thread_data_internal *p = (thread_data_internal *)a;

  // Set the Thread object p->me through the key to the specific data of the current thread
  p->me->set_specific();
  ink_set_thread_name(p->name);
  // If the function pointer f is not empty, then call f
  if (p->f)
    p-> f (p-> a);
  else
  // Otherwise the default run function of the execution thread
    p->me->execute();
  // Release the object after returning a
  ats_free(a);
  return NULL;
}

ink_thread
Thread::start(const char *name, size_t stacksize, ThreadFunction f, void *a) 
{
  thread_data_internal *p = (thread_data_internal *)ats_malloc(sizeof(thread_data_internal));

  // parameter f is a function pointer
  p-> f = f;
  // parameter a is the argument passed in when f is called
  p-> a = a;
  // p->me is the current Thread object
  p->me = this;
  memset(p->name, 0, MAX_THREAD_NAME_LENGTH);
  ink_strlcpy(p->name, name, MAX_THREAD_NAME_LENGTH);
  // When the thread is created, the spawn_thread_internal function is passed in, so the thread will call the function immediately.
  tid = ink_thread_create(spawn_thread_internal, (void *)p, 0, stacksize);

  return time;
}
```

  - There is a very strange place in Main.cc that also calls set_specific
    - But I didn't see any place to use main_thread afterwards.
    - From the notes, here is the support for running ATS on the win_9xMe system.
    - Optimization for calls to start_HttpProxyServer()

```
// source: proxy/Main.cc
  // This call is required for win_9xMe
  // without this this_ethread() is failing when
  // start_HttpProxyServer is called from main thread
  Thread *main_thread = new EThread;
  main_thread->set_specific();   
```

How does ## main() become [ET_NET 0]?

This_ethread()->execute() was executed at the end of Main.cc

  - The original execute() was called in spawn_thread_internal()
  - and spawn_thread_internal() is the first function to be called after the thread is created.
  - But in order to save a thread space, ATS directly calls this_ethread()->execute() in main() to start execution.
  - But the procedure that needs to call set_specific() in spawn_thread_internal() goes there?
  - How can this_ethread() return an EThread object if set_specific() is not called?

```
// source: iocore/eventsystem/UnixEventProcessor.cc
int
EventProcessor::start(int n_event_threads, size_t stacksize)
{
...
  int first_thread = 1;

  for (i = 0; i < n_event_threads; i++) {
    EThread *t = new EThread(REGULAR, i); 
    if (first_thread && !i) {
      // If i is 0, a set_specific action will be executed
      ink_thread_setspecific(Thread::thread_data_key, t); 
      global_mutex = t->mutex;
      t->cur_time = ink_get_based_hrtime_internal();
    }   
    all_ethreads[i] = t;

    eventthread[ET_CALL][i] = t;
    t->set_event_type((EventType)ET_CALL);
  }
...
  // The for loop starts at 1 and skips 0
  for (i = first_thread; i < n_ethreads; i++) {
    snprintf(thr_name, MAX_THREAD_NAME_LENGTH, "[ET_NET %d]", i);
    // that is, all_ethreads[0]->start() is not called
    ink_thread tid = all_ethreads[i]->start(thr_name, stacksize);
...
}
```

The above code shows that special handling of all_ethreads[0] is done in the EventProcessor::start() method.

  - call set_specific for all_ethreads[0]
  - do not start all_ethreads[0]

So, the last sentence of main(), this_ethread()->execute(), starts the execution of [ET_NET 0].

Revisit this process:

  - First called eventProcessor.start() at the beginning of main()
    - started all threads of [ET_NET 1] ~ [ET_NET n]
    - Initialize data for [ET_NET 0] but does not start the thread
  - then main() performs other startup processes
  - Finally main() calls this_ethread()->execute() to complete the startup of [ET_NET 0]

## References

- [I_Thread.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Thread.h)


# core component: EThread

EThread is a concrete implementation of Thread.

In ATS, only the EThread class is implemented based on the Thread class.

The EThread class is a thread type created and managed by the Event System.

It is one of the interfaces for the Event System to dispatch events (the other two are the Event and EventProcessor classes).

## Type of EThread

```
source: iocore/eventsystem/I_EThread.h
enum ThreadType {
  REGULAR = 0,
  MONITOR,
  DEDICATED,
};
```

ATS can create two types of EThread threads:

- DEDICATED type
   - This type of thread, after performing a certain continuation, dies
   - In other words, this type only processes/executes an Event, and an Event is passed when creating this type of EThread.
   - Or deal with a separate task, such as the continuation of NetAccept is performed by this type.
- REGULAR type
   - This type of thread, after the ATS is started, has been there, its thread execution function: execute() is an infinite loop
   - This type of thread maintains an event pool. When the event pool is not empty, the thread fetches the event from the event pool and executes the continuation of the event encapsulation.
   - When a thread needs to be scheduled to perform a Continuation, a Continuation is encapsulated into an Event by the EventProcessor and added to the thread's event pool.
   - This type is the core of EventSystem. It handles a lot of Events, and can pass an Event from the outside, and then let this EThread process/execute the Event.
- MONITOR type
   - There is actually a third type, but the MONITOR type is currently not used.
   - Maybe deleted when Yahoo was open sourced.

Before you create an EThread instance, you need to decide its type. Once you create it, you can't change it.


## Event Pool/Queue Type

To handle events, each EThread instance actually has four event queues:

- External queue (EventQueueExternal is declared as a member of EThread, is a protection queue)
   - Let EThread caller be able to append events to that particular thread
   - Since it can be accessed by other threads at the same time, its operation must remain atomic
- Local queue (subqueue of external queue)
   - External queues are for atomic operations and can be imported into local queues in bulk
   - Local queues can only be accessed by the current thread, atomic operations are not supported
- Internal queue (EventQueue is declared as a member of EThread, is a priority queue)
   - On the other hand, dedicated to timing events that are processed by EThread within a certain time frame
   - These events are queued internally
   - It may also contain events from external queues (eligible events in the local queue will enter the internal queue)
- implicit queues (NegativeQueue is declared as a local variable of the execute() function)
   - Because it is a local variable, it cannot be directly manipulated externally, so it is called implicit queue.
   - Usually only events with epoll_wait appear in this queue

## scheduling interface

EThread provides 8 scheduling functions (in fact, there are 9 but there is a _signal that is used outside of IOCore and is not used anywhere else).

The following method puts an Event into the outer queue of the current EThread thread.
- schedule_imm
- schedule_imm_signal
  - Designed for network modules
  - A thread may block on epoll_wait all the time. By introducing a pipe or eventfd, when a thread is scheduled to execute an event, the thread is asynchronously notified that it is liberated from epoll_wait.
- schedule_at
- schedule_in
- schedule_every

The above five methods will eventually call the public part of schedule()

The following method functions are the same as above, except that they put the Event directly into the internal queue.
- schedule_imm_local
- schedule_at_local
- schedule_in_local
- schedule_every_local

The above four methods will eventually call the public part of schedule_local()

## definition

```
class EThread : public Thread
{
public:
  // This omits the above mentioned nine schedule_* methods, as well as two public parts schedule() and schedule_local()
  schedule_*();

  InkRand generator;

private:
  // prevent unauthorized copies (Not implemented)
  EThread(const EThread &);
  EThread &operator=(const EThread &);

public:
  // Constructor & Destructor Reference: iocore/eventsystem/UnixEThread.cc
  // Basic constructor, initialize thread_private, while member tt is initialized to REGULAR, member id is initialized to NO_ETHREAD_ID
  EThread();
  // Increase the initialization of ethreads_to_be_signalled, while the member tt is initialized to the value of att, the member id is initialized to the value of anid
  EThread(ThreadType att, int anid);
  // Dedicated to the DEDICATED type constructor, the member tt is initialized to the value of att, oneevent initialization points to e
  EThread(ThreadType att, Event *e);
  // Destructor, refresh single, and release ethreads_to_be_signalled
  virtual ~EThread();

  // has been abandoned, did not find it used in ATS
  Event *schedule_spawn(Continuation *cont);

  // Thread internal data, used to save an array such as: statistical system
  /** Block of memory to allocate thread specific data e.g. stat system arrays. */
  char thread_private[PER_THREAD_DATA];

  // Connect to the Disk Processor and the AIO queue with it
  /** Private Data for the Disk Processor. */
  DiskHandler *diskHandler;
  /** Private Data for AIO. */
  Que(Continuation, link) aio_ops;

  // External event queue: protection queue
  ProtectedQueue EventQueueExternal;
  // Internal event queue: priority queue
  PriorityEventQueue EventQueue;

  // When the schedule operation, if you set the signal to true, you need to trigger the signal to the EThread that executes the Event.
  // But when the lock of the target EThread is not available, the signal cannot be sent immediately, so the thread can only be hung here first.
  // Notify at idle times.
  // The length of this array of pointers does not exceed the total number of threads. When the total number of threads is reached, it is converted to a direct mapping table.
  EThread **ethreads_to_be_signalled;
  int n_ethreads_to_be_signalled;

  // has been deprecated, it is estimated to be used in init_accept_per_thread to save the accept event, once you do not intend to accept, you can cancel it again
  // The estimate that can be used is the accept of the ftp data channel.
  Event *accept_event[MAX_ACCEPT_EVENTS];
  int main_accept_index;

  // Thread id starting at 0, set in new EThread in EventProcessor::start
  int id;
  // event types, such as ET_NET, ET_SSL, etc., are usually used to set the EThread to handle only events of the specified type.
  // This value is only read by is_event_type, set_event_type is written
  unsigned int event_types;
  bool is_event_type(EventType et);
  void set_event_type(EventType et);

  // Private Interface

  // event handler main function
  void execute();
  // Handle the event (Event), callback Cont-> HandleEvent
  void process_event(Event *e, int calling_code);
  // Release the event (Event), recycle resources
  void free_event(Event *e);

  // Post analysis, signal hook for high priority events
  void (*signal_hook)(EThread *);
  // Related to signal_hook, netProcessor internally uses this to achieve wake and lock
#if HAVE_EVENTFD
  int evfd;
#else
  int evpipe[2];
#endif
  EventIO * ep;

  // EThread type: REGULAR, MONITOR, DEDICATED
  ThreadType tt;
  // dedicated to EDIC of DEDICATED type
  Event *oneevent; // For dedicated event thread

  // ???
  ServerSessionPool *server_session_pool;
};

// This macro definition is used to return a pointer to a specific offset position in the private data of the thread within the given thread.
#define ETHREAD_GET_PTR(thread, offset) ((void *)((char *)(thread) + (offset)))

// a pointer to get the current EThread instance
extern EThread *this_ethread();

source: iocore/eventsystem/P_UnixEThread.h
TS_INLINE EThread *
this_ethread()
{
  return (EThread *)this_thread();
}

TS_INLINE void
EThread::free_event(Event *e)
{
  ink_assert(!e->in_the_priority_queue && !e->in_the_prot_queue);
  e->mutex = NULL;
  EVENT_FREE(e, eventAllocator, this);
}
```

## About EThread::event_types

EventType is actually an int type, starting with ET_CALL(0) and having a maximum of 7, you can define 8 types. The definition of EventType is as follows:

```
source: iocore/eventsystem/I_Event.h
typedef int EventType;
const int ET_CALL = 0;
const int MAX_EVENT_TYPES = 8; // conservative, these are dynamically allocated
```

EThread member event_types is a bitwise operation state value. Since EventType can define up to 8 types, only the lower 8 bits are used.

By calling the method ```void set_event_type(EventType et)``` multiple times, you can have one EThread handle multiple types of events at the same time.

You can use the method ```bool is_event_type(EventType et)``` to determine whether the EThread supports processing of the specified type of event.

```
source: iocore/eventsystem/UnixEThread.cc
bool
EThread::is_event_type(EventType et)
{
  return !!(event_types & (1 << (int)et));
}

void
EThread::set_event_type(EventType et)
{
  event_types |= (1 << (int)et);
}
```

## EThread::process_event() analyze

```
source: iocore/eventsystem/UnixEThread.cc

void
EThread::process_event(Event *e, int calling_code)
{
  ink_assert((!e->in_the_prot_queue && !e->in_the_priority_queue));
  // Try to acquire the lock
  MUTEX_TRY_LOCK_FOR(lock, e->mutex.m_ptr, this, e->continuation);
  // Determine whether to get the lock
  if (!lock.is_locked()) {
    // If you don't get the lock, throw the event back into the external local queue.
    e->timeout_at = cur_time + DELAY_FOR_RETRY;
    EventQueueExternal.enqueue_local(e);
    // return to execute()
  } else {
    // If you get the lock, first determine if the current event has been canceled.
    if (e->cancelled) {
      // release event if it is canceled
      free_event(e);
      // return to execute()
      return;
    }
    // Prepare callback Cont->handleEvent
    Continuation *c_temp = e->continuation;
    // Note: During the callback, Cont->Mutex is locked by the current EThread
    e->continuation->handleEvent(calling_code, e);
    ink_assert(!e->in_the_priority_queue);
    ink_assert(c_temp == e->continuation);
    // Release the lock early
    MUTEX_RELEASE(lock);

    // If the event is executed periodically, add it via schedule_every
    if (e->period) {
      // Not in the protection queue, nor in the priority queue. (that is, there is no external queue or internal queue)
      // Usually dequeue from the queue before calling process_event, it should not be in the queue
      // However, other operations may be invoked in Cont->handleEvent causing the event to be put back into the queue.
      if (!e->in_the_prot_queue && !e->in_the_priority_queue) {
        if (e->period < 0)
          // Less than zero means that this is an event in a recessive queue, without recalculating the value of timeout_at
          e->timeout_at = e->period;
        else {
          // Recalculate the value of the next execution time timeout_at
          cur_time = get_hrtime();
          e->timeout_at = cur_time + e->period;
          if (e->timeout_at < cur_time)
            e->timeout_at = cur_time;
        }
        // Reset the timeout_at event into the external local queue
        EventQueueExternal.enqueue_local(e);
      }
    } else if (!e->in_the_prot_queue && !e->in_the_priority_queue)
      // Not a periodic event, nor a protection queue, nor a priority queue. (that is, there is no external queue, nor internal queue), then release the event
      free_event(e);
  }
}
```

## EThread::execute() REGULAR and DEDICATED Process Analysis

EThread::execute() is divided into multiple parts by the switch statement:

- The first part is the processing of the REGULAR type
  - It consists of code in an infinite loop that continuously scans/traverses multiple queues and calls back the handler of the Event internal Cont
- The second part is the processing of the DEDICATED type
  - Direct callback to the handler of Cont inside oneevent, which is actually a simplified version of process_event(oneevent, EVENT_IMMEDIATE)

The following is a comment and analysis of the execute() code:

```
source: iocore/eventsystem/UnixEThread.cc
//
// void  EThread::execute()
//
// Execute loops forever on:
// Find the earliest event.
// Sleep until the event time or until an earlier event is inserted
// When its time for the event, try to get the appropriate continuation
// lock. If successful, call the continuation, otherwise put the event back
// into the queue.
//

void
EThread::execute()
{
  switch (tt) {
  case REGULAR: {
    // REGULAR part starts
    Event *e;
    Que(Event, link) NegativeQueue;
    ink_hrtime next_time = 0;

    // give priority to immediate events
    // Design purpose: prioritize immediate execution of events
    for (;;) {
      // execute all the available external events that have
      // already been dequeued
      // Set cur_time, which may also update cur_time after the state machine callback is completed each time the process_event method is called.
      // If it is a recurring event, the event will be re-added to the internal queue in process_event, and the cur_time will be updated.
      cur_time = ink_get_based_hrtime_internal();
      // Traversal: external local queue
      // The dequeue_local() method takes an event from the external local queue each time.
      while ((e = EventQueueExternal.dequeue_local())) {
        // When the event is canceled asynchronously
        if (e->cancelled)
          free_event(e);
        // Immediately executed event
        else if (!e->timeout_at) { // IMMEDIATE
          ink_assert(e->period == 0);
          // Callback state machine through process_event
          process_event(e, e->callback_event);
        // Periodically executed events
        } else if (e->timeout_at > 0) // INTERVAL
          // put in the internal queue
          EventQueue.enqueue(e, cur_time);
        // negative event
        else { // NEGATIVE
          Event *p = NULL;
          Event *a = NegativeQueue.head;
          // Note that the value of timeout_at is a negative number less than 0.
          // arranged in order of largest to smallest
          // If the same value is encountered, the post-inserted event is in front
          while (a && a->timeout_at > e->timeout_at) {
            p = a;
            a = a->link.next;
          }
          // put in implicit queue (negative queue)
          if (!a)
            NegativeQueue.enqueue(e);
          else
            NegativeQueue.insert(e, p);
        }
      }
      // Traversal: internal queue (priority queue)
      bool done_one;
      do {
        done_one = false;
        // execute all the eligible internal events
        // check_ready method reorders multiple subqueues in the priority queue (reschedule)
        // Make the execution time of the events contained in each sub-queue meet the requirements of each word queue
        EventQueue.check_ready(cur_time, this);
        // The dequeue_ready method only operates on subqueue 0, and the events inside need to be executed within 5ms.
        while ((e = EventQueue.dequeue_ready(cur_time))) {
          ink_assert(e);
          ink_assert(e->timeout_at > 0);
          if (e->cancelled)
            free_event(e);
          else {
            // This loop handles an event, setting done_one to true, indicating that it continues to traverse the internal queue
            done_one = true;
            process_event(e, e->callback_event);
          }
        }
        // Every time the loop will process all the events in the 0 subqueue
        // If at least one event is processed in the traversal of subqueue 0, then
        // This process will take time, but if the event is cancelled, it will not take much time, so it is not recorded;
        // At this point, the events in the 1st subqueue may have reached the time required to execute.
        // Or, the event in the subqueue 1 has timed out.
        // So to reorganize the priority queue through the do-while (done_one) loop.
        // Then process the sub-queue 0 again to ensure that the event is executed on time.
        // If an event is not processed in this loop,
        // That means that after sorting the queue, subqueue 0 is still an empty queue.
        // This means that the most recent event in the internal queue needs to be executed after 5ms
        // Then it is no longer necessary to traverse the internal queue, ending the do-while (done_one) loop
        // Note: There may be a hypothesis here, that is, a large loop of each REGULAR ETHREAD with a running time of 5ms.
        // If the duration of a loop exceeds 5ms, the events of the internal queue may time out
      } while (done_one);
      // execute any negative (poll) events
      // Traversal: implicit queue (if the implicit queue is not empty)
      if (NegativeQueue.head) {
        // triggers signals to notify other threads of the event
        // There is a possibility of blocking in flush_signals
        if (n_ethreads_to_be_signalled)
          flush_signals(this);
        // dequeue all the external events and put them in a local
        // queue. If there are no external events available, don't
        // do a cond_timedwait.
        // If the external queue is not empty, then all events are fetched into the external local queue at one time.
        // The external queue is the only queue in which EThread gets events from the outside (crossing EThread). The operation must be atomic.
        if (!INK_ATOMICLIST_EMPTY(EventQueueExternal.al))
          EventQueueExternal.dequeue_timed(cur_time, next_time, false);
        // traverse again: external local queue
        // Purpose 1: Prioritize immediate execution of events
        // Purpose 2: There may be negative events that need to be executed
        while ((e = EventQueueExternal.dequeue_local())) {
          // First is the event that is executed immediately
          // Why didn't you first judge cancelled?
          if (!e->timeout_at)
            // Callback state machine through process_event
            process_event(e, e->callback_event);
          else {
            // When the event is canceled asynchronously
            if (e->cancelled)
              free_event(e);
            else {
              // If its a negative event, it must be a result of
              // a negative event, which has been turned into a
              // timed-event (because of a missed lock), executed
              // before the poll. So, it must
              // be executed in this round (because you can't have
              // more than one poll between two executions of a
              // negative event)
              if (e->timeout_at < 0) {
              // Negative events, according to the timeout_at value, sorted from large to small into the implicit queue
                Event *p = NULL;
                Event *a = NegativeQueue.head;
                while (a && a->timeout_at > e->timeout_at) {
                  p = a;
                  a = a->link.next;
                }
                if (!a)
                  NegativeQueue.enqueue(e);
                else
                  NegativeQueue.insert(e, p);
              // is not a negative event, it is put into the internal queue
              } else
                EventQueue.enqueue(e, cur_time);
            }
          }
        }
        // execute poll events
        // Execute the polling event in the implicit queue, take one event at a time, call the state machine through process_event, currently:
        // For TCP, the state machine is NetHandler::mainNetEvent
        // For UDP, the state machine is PollCont::pollEvent
        while ((e = NegativeQueue.dequeue()))
          process_event(e, EVENT_POLL);
        // Re-evaluate whether the external queue is empty. If the external queue is not empty, then all events are fetched into the external local queue at one time.
        if (!INK_ATOMICLIST_EMPTY(EventQueueExternal.al))
          EventQueueExternal.dequeue_timed(cur_time, next_time, false);
        // Then go back to the for loop and re-traverse the external local queue
      // The implicit queue is empty and there are no negative events
      } else { // Means there are no negative events
        // There is no negative event, that is, only periodic events need to be executed, then you need to save CPU resources to avoid idling.
        // Get the sleepable time by the difference between the time of the earliest need to execute the event in the internal queue and the current time.
        next_time = EventQueue.earliest_timeout();
        ink_hrtime sleep_time = next_time - cur_time;

        // Use the maximum allowed value if the sleep time exceeds the maximum allowed sleep time
        // Note: The last thing you get here is an absolute time.
        if (sleep_time > THREAD_MAX_HEARTBEAT_MSECONDS * HRTIME_MSECOND) {
          next_time = cur_time + THREAD_MAX_HEARTBEAT_MSECONDS * HRTIME_MSECOND;
        }
        // dequeue all the external events and put them in a local
        // queue. If there are no external events available, do a
        // cond_timedwait.
        // Because there is a possibility of blocking in the flush_signals method, the next time is passed in the dequeue_timed method.
        // If the blocking of flush_signals causes the value of next_time to have been missed, then dequeue_timed will not block.
        if (n_ethreads_to_be_signalled)
          flush_signals(this);
        // First block the wait, can be kicked out by the flush_signals of other threads, or reach the absolute time of next_time
        // Then import all events from the external queue into the external local queue
        EventQueueExternal.dequeue_timed(cur_time, next_time, true);
      }
    }
  }

  // The following is the DEDICATED mode
  case DEDICATED: {
    // coverity[lock]
    MUTEX_TAKE_LOCK_FOR(oneevent->mutex, this, oneevent->continuation);
    oneevent->continuation->handleEvent(EVENT_IMMEDIATE, oneevent);
    MUTEX_UNTAKE_LOCK(oneevent->mutex, this);
    free_event(oneevent);
    break;
  }

  default:
    ink_assert(!"bad case value (execute)");
    break;
  } /* End switch */
  // coverity[missing_unlock]
}
```

The following is a simplified logical and logical diagram:

//for (;;)

- External local queue
- internal queue
- if implicit queue has Event
  - Determine the previous schedule_*_signal call and trigger the signal asynchronously
  - if the external queue has Event --> in order to avoid blocking the following operations
    - External queue import external local queue
  - External local queue
  - implicit queue
  - if the external queue has Event --> in order to avoid blocking the following operations
    - External queue import external local queue
- else
  - Calculate the time difference between the current event time of the internal queue and the current time, as sleep time
  - Determine if the maximum sleep time is exceeded. If it is exceeded, use the longest sleep time value.
  - Determine the previous schedule_*_signal call and trigger the signal asynchronously
  - The external queue is imported into the local queue. If the external queue does not have an Event, the maximum waiting time for sleep time
  - During the wait, you can use schedule_*_signal to interrupt this wait

//end for(;;)

```
                                                                                     + ------------- +
       +-----------------------------------------------------------------------------o    Sleep    |
       | + ------ ^ ------ +
       | |
       | + --------- + + |
       |                     |  START  |                                                    |
       | + ---- o ---- + |
       | | |
       | | |
       | | |
       | | |
+ ------ V ------ + + ------ V --- V - + + - V ------- V - + + ---- --o ------ +
|   External  o------------>   External  o---------------->    Event    o------------>    Flush    |
|    Event    |            |  EventQueue |                |             |            |             |
|    Queue    o############>   (local)   o################>    Queue    |            |   Signals   |
+ ------ ^ ------ + + - ^ --- o --- ^ - + + - ^ --- ^ --- ^ - + + ---- --on ------ +
       | # #%% #% |
       | # #%% #% |
       | # #%% #% |
       | # #% + ---------------- + + # # + + ---------------- + + |
       |                      #   #   %  |   Timed Event  <%%%   #   %%%> Interval Event |  |
       | # #% + -------- o ------- + # + -------- o ------- + |
       | # #% | # | |
       | | + ---------------- + # #% + -------- V ------- + # + -------- V ------- + |
       |  | process_event  O###   #   %  | process_event  |      #   ###o process_event  |  |
       | + -------- ^ ------- + #% + -------- ^ ------- + # # + -------- -------- + + |
       | #% | # # |
       | + -------- o ------- + #% + -------- o ------- + # # |
       |  | Interval Event <%%%   #   %%%> Immediate Event<%%%   #   #                      |
       | + ---------------- + + # # ---------------- + + # # # |
       |% #% # # |
       |% #% # # |
       |% #% # # |
       | | + - V --- V ------ + + - V --- o --- V - + + ------ V ------ +
       |                   |   Negative  <################o   External  <############o   External  |
       +-------------------o             |                |  EventQueue |            |    Event    |
                           |    Queue    <----------------o   (local)   <------------o    Queue    |
                           + ------------- + + ------------- + + ------------- +

o--------->  execute route
o#########>  event route
<%%%%%%%%%>  call and return
```

From the above figure, we can see that REGULAR EThread is divided into two types according to whether the implicit queue is empty.

- Small loop: implicit queue is empty
  - Each loop starts from the local queue of the external queue
  - then the internal queue
  - then flush_signals()
  - Last blocked until the next timed event or wake up by signal()
  - After waking up, import the atomic queue of the external queue into the external local queue
  - Start the next cycle
- Large loop: recessive queue is not empty
  - Each loop starts from the local queue of the external queue
  - then the internal queue
  - then flush_signals()
  - Import the atomic queue of the external queue into an external local queue
  - Process the local queue of the external queue again
  - then an implicit queue
  - Import the atomic queue of the external queue into an external local queue
  - Start the next cycle

In a large loop, the first time the external queue is processed for the internal queue, the second time the external queue is for the implicit queue, and the EThread is not suspended at the end of each loop like a small loop.

due to:

  - Both internal and implicit queues are private members of EThread
  - Events in both queues need to be passed in from an external queue

In order to be able to process the latest events in a timely manner, each time before processing the two queues, the external queue is checked first to determine that the latest events can be processed in a timely manner.
 

External queue & local queue

- The external queue is a Free Lock queue, ensuring that multiple threads read and write queues do not need to be locked, and there is no conflict.
- The local queue is a subqueue of the external queue. It is an ordinary List structure that gives Que the operation methods, such as pop, push, insert, etc.
- Usually the new Event is directly added to the external queue. Some internal methods schedule_*_local() can directly add the Event to the local queue of the external queue.
- But the operation of Free Lock still has a price, so in each loop, all the events of the external queue will be taken out at once, and placed in the local queue of the external queue.
- Next, you only need to traverse the local queue of the external queue to determine the type of Event.
  - Immediate execution (usually NetAccept Event)
     * 由schedule_imm()添加，timeout_at=0，period=0
     * Call handler, then Free(Event)
  - Timed execution (usually Timeout Check)
     * 由schedule_at/in()添加，timeout_at>0，period=0
     * Simple understanding of the difference between at and in: at is absolute time, in is relative time
     * Call the handler when the time arrives
     * Then check if you need to "periodically execute (possibly for resource reclamation, buffer refresh, etc.)"
     * 由schedule_every()添加，timeout_at=period>0
     * Put the Event into the local queue of the external queue if it needs to be executed periodically, otherwise Free(Event)
  - Execute at any time (currently only NetHandler Event)
     * 由schedule_every()添加，timeout_at=period<0
     * The handler is called every cycle, and then the event is placed in the local queue of the external queue
     * handler for TCP events is NetHandler::mainNetEvent()

Internal queue

- In order to improve the efficiency of Event processing, especially timing and periodic events
  - In the first traversal, the timed execution of the Event will be taken out and placed in the internal queue, the internal queue is a priority queue
  - in the internal queue, sorted by the chronological order in which the Event is about to be executed
  - Divided into multiple subqueues based on how many ms the distance execution time is (<5ms, 10, 20, 40, 80, 160, 320, 640, 1280, 2560, 5120)
  - where subqueue 0 (<5ms) holds all the events that need to be executed within 5ms
  - Reorder and organize internal queues before a process of traversing internal queues independently
  - then traverse the subqueue 0 of the internal queue and execute the handler inside the Event
  - After the execution is complete, it is put back into the external local queue by process_event and waits for the next loop.

Implicit queue (negative event queue/negative queue)

- When adding an event to this queue, it is passed the schedule_every method, the time parameter is a negative number, and this negative number is treated as the sorted key in the queue.
- An Event with a value of -1 is executed first, followed by an Event with a value of -2,...
- If the values ​​are the same, for example, all are -2, then in the order of joining the queue, the first join is executed first, and the first join is executed.
- After executing the handler in the Event, it is put back into the external local queue by process_event, waiting for the next loop.
- Usually placed in this queue are Polling operations, such as:
  - NetHandler :: mainNetEvent
  - PollCont::pollEvent
- The above two handlers will call epoll_wait, then drive netProcessor to complete the receiving and sending of network data.

## signal_hook Introduction

When epoll_wait is called, there will be a blocking timeout wait time. When we introduced the Event System, we emphasized that it must be completely non-blocking.

But blocking in epoll_wait is another possible situation. How does ATS handle this special blocking situation in Event System?

The answer is ```controlled blocking```:

- The upper limit of the blocking time, determined by the timeout parameter of epoll_wait
- Let epoll_wait return by timeout before the timeout arrives by signal

It is easy to know by man epoll_wait:

- The timeout setting of epoll_wait is set in units of one thousandth of a second (ms/millisecond)
- When this value is greater than 0, then epoll_wait will not return immediately
   - Wait for the timeout specified by timeout to wait for possible events
   - But there may be no waiting time to timeout, but return in advance:
      - A new event has appeared 
      - interrupted by system interruption

System interruptions are uncontrollable, but what about new events?

- ATS creates an evfd via the eventfd() system call, wraps the fd into ep, and adds it to the epoll fd, focusing on the evfd READ event.
- If you make evfd readable, it will trigger a new event, then you only need to provide a method to make evfd readable, you can generate new events, so epoll_wait can return immediately

ATS implements this controlled blocking by adding evfd to the fd collection, which is the principle of controlled blocking.

In order to make this design more versatile, the signal_hook is added to write data to evfd. This signal_hook is initialized to net_signal_hook_function() in initialize_thread_for_net in iocore/net/UnixNet.cc, and is specified when ep is initialized. The type (ep->type) is EVENTIO_ASYNC_SIGNAL.

In NetHandler::mainNetEvent, you can see that EventIO is processed specifically for EVENTIO_ASYNC_SIGNAL type, which is to call net_signal_hook_callback() to read the data. Since this is just to let epoll_wait return from the timeout wait state, it is read. The data is useless.

In this way, ATS turns the epoll_wait timeout wait into a controllable wait.

Therefore, evfd, signal_hook and ep, these three members are one-piece, specifically designed to support controlled blocking of network IO.

In fact, the default value of this timeout wait is 10ms, that is, one hundredth of a second. ATS will interrupt this short time to process an event immediately. It can be seen that ATS is for real-time processing and realizes schedule_imm_signal(). The meaning of EVENT_IMMEDIATE!

In addition to schedule_imm_signal(), it will be called to signal_hook, and it will be called when UnixNetVConnection::reenable.

## Signal's asynchronous notification

Why is there a special method like schedule_imm_signal?

In schedule_*(), the event is placed into the external queue via ```EventQueueExternal.enqueue(e, fast_signal=true)```.

```
TS_INLINE Event *
EventProcessor::schedule(Event *e, EventType etype, bool fast_signal)
{
  ink_assert(etype < MAX_EVENT_TYPES);
  // assign_thread(etype) is responsible for returning the ethread thread pointer of the specified etype type from the thread list
  e->ethread = assign_thread(etype);
  // If Cont has its own mutex, Event inherits Cont's mutex, otherwise Event and Cont inherit ethread's mutex
  if (e->continuation->mutex)
    e->mutex = e->continuation->mutex;
  else
    e->mutex = e->continuation->mutex = e->ethread->mutex;

  // Push Event into the external queue of the assigned ethread thread
  e->ethread->EventQueueExternal.enqueue(e, fast_signal);
  return e;
}
```

When reading the code of the enqueue() method, be sure to look at the above schedule() method and find the most awake moment of the brain, otherwise it will be easy to faint...

EventQueueExternal is a ProtectedQueue, then its enqueue method is as follows:

```
source:iocore/eventsystem/ProtectedQueue.cc
void
ProtectedQueue::enqueue(Event *e, bool fast_signal)
{
  ink_assert(!e->in_the_prot_queue && !e->in_the_priority_queue);
  // doEThread is the thread where the task initiator is located, runEThread is the thread where the task executor is, and the task description (state machine) is passed through the Event object.
  // Usually an event object is created by doEThread, but the member ethread member of the object is NULL.
  // Then, assign a runEThread to the Event via eventProcessor.schedule_*() and then call this method.
  // Since doEThread may be in the same thread pool as RunEThread, doEThread may be the same as runEThread.
  // So, here you need to consider that e->ethread might be equal to doEThread, and doEThread is this_ethread().
  EThread *e_ethread = e->ethread;
  e->in_the_prot_queue = 1;
  // ink_atimiclist_push performs atomic operations to push e into the head of al, and the return value is the head before pressing
  // So was_empty is true to indicate that al is empty before pressing
  bool was_empty = (ink_atomiclist_push(&al, e) == NULL);

  if (was_empty) {
    // Empty if the protection queue is pushed into the new event
    // inserting_thread is the queue that initiated the insert operation
    // For example: insert event into ET_NET from DEDICATED ACCEPT THREAD,
    // Then inserting_thread is DEDICATED ACCEPT THREAD
    EThread *inserting_thread = this_ethread();
    // queue e->ethread in the list of threads to be signalled
    // inserting_thread == 0 means it is not a regular EThread
    // If doEThread and runEThread are the same EThread, then no special processing is needed here.
    // At this point, the EThread that initiated the insert is the EThread that will handle the Event. This is called an internal insert.
    // If doEThread is different from runEThread, the following processing flow is required.
    // At this point, the EThread that initiated the insert is not the EThread that will handle the event, simply inserting it from outside the thread.
    // This event is created by the current EThread (inserting_thread / doEThread) and is inserted into another EThread (e_ethread / runEThread)
    // The way to call this method must be through e_ethread->ExternalEventQueue.enqueue()
    if (inserting_thread != e_ethread) {
      // If the ethread that initiated the insert is not of type REGULAR.
      // For example: DEDICATED type (the type of ethreads_to_be_signalled is NULL)
      if (!inserting_thread || !inserting_thread->ethreads_to_be_signalled) {
        // Since doEThread does not have ethreads_to_be_signalled, the delay notification mechanism cannot be implemented. Only the blocking mode can be used to send a notification to runEThread.
        // After issuing the notification, the ink_cond_timedwait blocked in ProtectedQueue::dequeue_timed will return immediately.
        // The following signal is directly notified to the xxxQueue held by e->ethread
        signal();
        if (fast_signal) {
          // If you need to trigger signal immediately, try to notify EThread that generated this Event.
          if (e_ethread->signal_hook)
            // Call the signal_hook method of the EThread that generates the event to achieve asynchronous notification
            // Currently in the NetHandler through the signal_hook can let epoll_wait block waiting for interrupt and return
            e_ethread->signal_hook(e_ethread);
        }
      } else {
      // If the current EThread is of type REGULAR, and ethreads_to_be_signalled is not empty (that is, support for queued signaling)
#ifdef EAGER_SIGNALLING
        // The meaning of the macro definition switch here: send the signal more promptly. (Is this good? See the analysis below)
        // Since the event has been inserted into the queue, it is necessary to send a signal to the queue holding this event.
        // and this queue must be e->ethread->EventQueueExternal
        // This should be abbreviated as if (try_signal()) because the currently called enqueue() method is passed:
        // e->ethread->EventQueueExternal.enqueue() is initiated.
        // Try to signal now and avoid deferred posting.
        if (e_ethread->EventQueueExternal.try_signal())
          return;
        // Try to initiate a signal directly before pushing into the signal queue.
        // Return directly if successful, and continue to push into the signal queue if it fails.
#endif
        if (fast_signal) {
          // If you need to trigger the signal immediately
          if (e_ethread->signal_hook)
            // Call the event_hook method of the EThread holding the event to achieve asynchronous notification
            e_ethread->signal_hook(e_ethread);
        }
        // signal queue operation, adding a to-be-notified list to the signal queue in the EThread that initiated the insert operation
        int &t = inserting_thread->n_ethreads_to_be_signalled;
        EThread **sig_e = inserting_thread->ethreads_to_be_signalled;
        if ((t + 1) >= eventProcessor.n_ethreads) {
          // If it is added to the queue, it will exceed or reach the upper limit of the queue.
          // we have run out of room
          if ((t + 1) == eventProcessor.n_ethreads) {
            // When it is added to the queue, when the queue is full, convert this queue to a directly mapped array, using EThread::id as the array subscript
            // convert to direct map, put each ethread (sig_e[i]) into
            // the direct map loation: sig_e[sig_e[i]->id]
            // Traversing the members of the current queue, the algorithm is as follows:
            // 1. Take out a member cur
            // 2. Save sig_e[cur->id] to next
            // 3. If cur==next, the order of this member meets the requirements of the mapping table, skip to step 5.
            // 4. Put cur to sig_e[cur->id], then assign the previously saved next to cur, skip to step 2
            // Processing in the 2 ~ 4 step loop, encountered sig_e [cur-> id] is NULL,
            // At this point sig_e[i] has been moved to the position of the fight, but the pointer to the old position has not been emptied.
            // 5. Determine whether sig_e[i]->id is i, whether it meets the requirements of the mapping table, and if it is not satisfied, point sig_e[i] to NULL(0)
            // 6. Select the next member and jump to step 1. 
            for (int i = 0; i < t; i++) {
              EThread *cur = sig_e[i]; // put this ethread
              while (cur) {
                EThread *next = sig_e[cur->id]; // into this location
                if (next == cur)
                  break;
                sig_e [cur-> id] = cur;
                cur = next;
              }
              // if not overwritten
              if (sig_e[i] && sig_e[i]->id != i)
                sig_e [i] = 0;
            }
            t++;
          }
          // At this point, the signal queue must already be the mode of the mapping table, so insert the EThread waiting to be notified into the list according to the mapping table.
          // we have a direct map, insert this EThread
          sig_e[e_ethread->id] = e_ethread;
        } else
          // Queue insertion mode
          // insert into vector
          sig_e[t++] = e_ethread;
      }
    }
  }
}
```

Look at signal and try_signal again.

- signal
   - First get the lock in blocking mode
   - then trigger cond_signal
   - Last release lock
- try_signal is a non-blocking version of signal
   - Try to acquire the lock. If the lock is obtained, it is the same as signal, and then returns 1, indicating successful execution of the signal operation.
   - Returns 0 if no lock is obtained

If try_signal gets the lock and successfully sends a notification, it means:

- There is currently no special blocking scenario such as epoll_wait().
- There is no need to send a notification to a special component via signal_hook().

Instead, it means:

- Target EThread is not currently in cond_wait() waiting
- The target EThread is currently executing the task of the Event tag, and there may be blocking caused by special components.
- Need to interrupt the blocking of special components by signal_hook()

The relevant code is as follows:

```
source: iocore/eventsystem/P_ProtectedQueue.h
TS_INLINE void
ProtectedQueue::signal()
{
  // Need to get the lock before you can signal the thread
  ink_mutex_acquire(&lock);
  ink_cond_signal(&might_have_data);
  ink_mutex_release(&lock);
}

TS_INLINE int 
ProtectedQueue::try_signal()
{
  // Need to get the lock before you can signal the thread
  if (ink_mutex_try_acquire(&lock)) {
    ink_cond_signal(&might_have_data);
    ink_mutex_release(&lock);
    return 1;
  } else {
    return 0;
  }
}
```

When the notification is placed in the signal queue, it will be judged in the REGULAR mode of EThread::execute(). If the signal queue is found to have elements, flush_signals(this) will be called for notification. The relevant code is as follows:

```
void
flush_signals(EThread *thr)
{
  ink_assert(this_ethread() == thr);
  int n = thr->n_ethreads_to_be_signalled;
  if (n > eventProcessor.n_ethreads)
    n = eventProcessor.n_ethreads; // MAX
  int i;

// Since the lock is only there to prevent a race in ink_cond_timedwait
// the lock is taken only for a short time, thus it is unlikely that
// this code has any effect.
#ifdef EAGER_SIGNALLING
  // First complete some of the signal operations in non-blocking mode by try_signal.
  for (i = 0; i < n; i++) {
    // Try to signal as many threads as possible without blocking.
    if (thr->ethreads_to_be_signalled[i]) {
      if (thr->ethreads_to_be_signalled[i]->EventQueueExternal.try_signal())
        thr->ethreads_to_be_signalled[i] = 0;
    }
  }
#endif
  // Then use signal() to complete the remaining signal operations in blocking mode.
  // If an EThread Negative Queue is empty, then when no events need to be executed,
  // Need to hang EThread for a while through ProtectedQueue::timed_wait()
  // During this time, if the EThread needs to process an Event immediately, it can wake it up via signal()
  // If an EThread Negative Queue is not empty, then there is a Negative Event that needs to be executed at any time.
  // At this point, the EThread never needs to hang because it needs to constantly execute the tasks specified by the Negative Event as soon as it has a chance.
  // and so,
  // For EThreads with Negative Events, you usually only need to call signal_hook().
  // For EThreads that do not have a Negative Event, you need to consider both "blocking in the state machine" or "EThread is suspended".
  // E.g:
  // NetHandler is a state machine driven by Negative Event, which has a blocking condition when calling epoll_wait.
  // Therefore, the NetHandler registers the signal_hook function to implement the function of aborting epoll_wait blocking and returning to EventSystem.
  for (i = 0; i < n; i++) {
    if (thr->ethreads_to_be_signalled[i]) {
      thr->ethreads_to_be_signalled[i]->EventQueueExternal.signal();
      if (thr->ethreads_to_be_signalled[i]->signal_hook)
        thr->ethreads_to_be_signalled[i]->signal_hook(thr->ethreads_to_be_signalled[i]);
      thr->ethreads_to_be_signalled[i] = 0;
    }   
  }
  // Queue length is cleared
  // Since the queue length is converted to a mapping table when it reaches the maximum value, and there is no logic for reverse conversion, all elements in the table are completely processed each time.
  thr->n_ethreads_to_be_signalled = 0;
}
```

## About EAGER_SIGNALLING

This is described in [ProtectedQueue.cc] (http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/ProtectedQueue.cc):

```
 36 // The protected queue is designed to delay signaling of threads
 37 // until some amount of work has been completed on the current thread
 38 // in order to prevent excess context switches.
 39 //
 40 // Defining EAGER_SIGNALLING disables this behavior and causes                                                                      
 41 // threads to be made runnable immediately.
 42 //
 43 // #define EAGER_SIGNALLING
```

The translation is as follows:

  - The protection queue is designed to:
    - Notify the thread when the current thread has completed a certain amount of work
    - Delayed notifications can be used to prevent/reduce excessive context switching
  - But you can define the macro EAGER_SIGNALLING to turn off the above behavior and let the notification be sent immediately

explain:

  - If the notification needs to be initiated immediately for a specific Event, the target thread will be woken up immediately
    - After the target thread is woken up, the CPU needs to load the state of the register before it sleeps, thereby causing context incision
    - After processing the specified Event, the target thread enters the temple again, causing the upper and lower questions to be cut out.
    - If you have to cut in and out once for each Event, then the number of context switches will be amazing.
  - If a delay notice is used,
    - in each loop of EThread::execute(),
      - Save the target thread that needs to be notified in a list of the current thread
      - At the end of the loop, iterate over the list and notify all threads that need to wake up at once
    - At this point, there may be multiple events in a target thread that need to run
    - The same context-cut operation, you can process multiple events and then cut out
    - This reduces the number of context switches

## References

![EventQueue - EThread - Signals](https://cdn.rawgit.com/oknet/atsinternals/master/CH01-EventSystem/CH01-EventSystem-001.svg)

- [I_EThread.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_EThread.h)
- [P_UnixEThread.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_UnixEThread.h)
- [ProtectedQueue.cc](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/ProtectedQueue.cc)
- [EventProcessor](CH01S10-Interface-EventProcessor.zh.md)
- [Event](CH01S08-Core-Event.zh.md)
