# Core component: Action & Event

Before introducing Event, first look at the Action:

- When a state machine initiates an asynchronous operation through a Processor method, the Processor will return a pointer to the Action class.
- The state machine can cancel an asynchronous operation in progress through a pointer to an Action object.
- After canceling, the state machine that initiated the operation will not receive a callback from the asynchronous operation.
- Actions or their derived classes are a common return type for Processors in the Event System, as well as methods/functions exposed by the entire IO core library.
- The canceler of the Action must be the state machine for which the operation will be called back, and the lock of the state machine is required during the cancellation process.

## Definition / Member

Event inherits from Action, first look at the Action class

`` `
class Action
{
public:
    Continuation * continuation;
    Ptr<ProxyMutex> mutex;
    // Prevent the compiler from caching the value of the variable, on the 64bits platform, the reading or setting of the value is atomic
    volatile int cancelled;

    // can be overridden by the inherited class to implement the corresponding processing in the inherited class
    // as the only interface provided by the Action to the outside
    virtual void cancel(Continuation * c = NULL) {
    if (!cancelled)
        cancelled = true;
    }

    // This method always sets the cancel operation directly on the Action base class, skipping the cancellation process of the inherited class
    // This method is specific to the Event object within the ATS code.
    void cancel_action(Continuation * c = NULL) {
    if (!cancelled)
        cancelled = true;
    }

    // overload assignment (=) operation
    // used to initialize the Action
    // acont is the state machine that is called back when the operation is complete
    // mutex is the lock of the above state machine, using Ptr<> automatic pointer management
    Continuation *operator =(Continuation * acont)
    {
        continuation = acont;
        if (acont)
            mutex = acont->mutex;
        else
            mutex = 0;
        return acont;
    }

    // Constructor
    // Initialize continuation to NULL, canceled to false
    Action():continuation(NULL), cancelled(false) {
    }

    virtual ~ Action() {
    }
};
`` `

## Processor Method implementer:

When implementing a method of a Processor, you must ensure that:

- No events are sent to the state machine after the operation is cancelled.

## Return an Action:

The Processor method is usually executed asynchronously, so you must return an Action so that the state machine can cancel the task at any time before the task is completed.
   - At this point, the state machine always gets the Action first.
   - Then I will receive a callback for this task.
   - Cancel the task at any time before the callback is received.

Since some Processor methods can be executed synchronously (reentrant), there may be cases where the state machine is first called back and the Action is returned to the state machine.
Returning the Action at this point is meaningless. To handle this situation, a special value is returned instead of the Action object to indicate that the state machine has completed the action.
   - ACTION_RESULT_DONE The Processor has completed the task and inline (synchronized) callback to the state machine
   - ACTION_RESULT_INLINE is not currently in use
   - ACTION_RESULT_IO_ERROR is not currently in use

Perhaps there is such a more complicated problem:
   - When the result is ACTION_RESULT_DONE
   - At the same time, the state machine releases itself in the synchronous callback
 
Therefore, the implementer of the state machine must:
   - When synchronizing callbacks, don't release yourself (it's not easy to tell if the type of callback is synchronous or asynchronous)
or,
   - Immediately check the Action returned by the Processor method
   - If the value is ACTION_RESULT_DONE, then no state variables of the state machine can be read or written.

Either way, check the return value (whether it is ACTION_RESULT_DONE) and handle it accordingly.


## Assign/Release Strategy:

The allocation and release of Actions follows the following strategies:

- The Action is assigned by the Processor that executes it.
  - Usually the Processor method creates a Task state machine to perform a specific task asynchronously
  - the Action object is a member object of the Task state machine
- After the Action is completed or cancelled, Processor has the responsibility and obligation to release it.
  - When the Task state machine needs to call back the state machine, 
    - Obtain and lock the mutex via Action
    - Then check the members of the action cancelled
    - If it has been canceled, destroy the Task state machine
    - Otherwise callback Action.continuation
- When the returned action has been completed, or the state machine has performed a cancel operation on an Action,
  - The state machine can no longer access the action.


## References

[I_Action.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Action.h)


#芯部件:Event

The Event class inherits from the Action class, which is a special Action type returned by EventProcessor, which is returned by EventProcessor as a result of the dispatch operation.

Unlike asynchronous operations of Action, Event is not reentrant.

  - EventProcessor always returns an Event object to the state machine,
  - Then, the state machine will receive the callback.
  - Not like the returner of the Action class, there may be situations where the synchronous callback state machine exists.

In addition to being able to cancel the event (because it is an action), you can also reschedule it after receiving its callback.

## Definition / Member

`` `
class Event : public Action 
{ 
public: 
    // Set the event (Event) type method
    void schedule_imm(int callback_event = EVENT_IMMEDIATE); 
    void schedule_at(ink_hrtime atimeout_at, int callback_event = EVENT_INTERVAL); 
    void schedule_in(ink_hrtime atimeout_in, int callback_event = EVENT_INTERVAL); 
    void schedule_every(ink_hrtime aperiod, int callback_event = EVENT_INTERVAL);

//  inherited from Action
    // Continuation * continuation;
    // Ptr<ProxyMutex> mutex;
    // volatile int cancelled;
    // virtual void cancel(Continuation * c = NULL);  

    // The ethread pointer that handles this Event is populated before ethread processes this event (that is, when it is in schedule).
    // When an Event is managed by an EThread, it cannot be handed over to other EThread management.
    EThread *ethread;

    // status and flag
    unsigned int in_the_prot_queue:1; 
    unsigned int in_the_priority_queue:1; 
    unsigned int immediate:1; 
    unsigned int globally_allocated:1; 
    unsigned int in_heap:4; 

    // Event type passed to Cont->handler
    int callback_event;

    // Combine to form four event types
    ink_hrtime timeout_at; 
    ink_hrtime period;

    // passed as data (Data) when calling Cont->handler
    void *cookie;

    // Constructor
    Event();

    // Initialize an Event
    Event *init(Continuation * c, ink_hrtime atimeout_at = 0, ink_hrtime aperiod = 0);
    // release the Event
    void free();

private:
    // use the fast allocators
    void *operator new(size_t size);

    // prevent unauthorized copies (Not implemented)
    Event(const Event &); 
    Event & operator =(const Event &);
public: 
    LINK(Event, link);

    virtual ~Event() {}
};

TS_INLINE Event *
Event::init(Continuation *c, ink_hrtime atimeout_at, ink_hrtime aperiod)
{
  continuation = c;
  timeout_at = atimeout_at;
  period = aperiod;
  immediate = !period && !atimeout_at;
  cancelled = false;
  return this;
}

TS_INLINE void
Event::free()
{
  mutex = NULL;
  eventAllocator.free(this);
}

TS_INLINE
Event::Event()
  : ethread(0), in_the_prot_queue(false), in_the_priority_queue(false), 
    immediate(false), globally_allocated(true), in_heap(false),
    timeout_at(0), period(0)
{
}

// The memory allocation of the Event does not perform a bzero() operation on the space, so all necessary values ​​are initialized in the Event::init() method.
#define EVENT_ALLOC(_a, _t) THREAD_ALLOC(_a, _t)
#define EVENT_FREE(_p, _a, _t) \
  _p->mutex = NULL;            \
  if (_p->globally_allocated)  \
    ::_a.free(_p);             \
  else                         \
  THREAD_FREE(_p, _a, _t)
`` `

## Event method

Event::init()

- Initialize an Event
- Usually used to prepare an Event and then select a new EThread thread to handle this event
- The EThread::schedule() method is usually called next.

Event::schedule_*()

- If the event already exists in EThread's ```internal queue``, it is first removed from the queue
- Then add events directly to EThread's ```local queue```
- So this method can only add events (Event) to the current EThread event pool, not to add threads

For the use of Event::schedule_*(), there are related comments in the source code, translated as follows:

- When an event is rescheduled through any of the Event class's dispatch functions, the state machine (SM) cannot call these dispatch functions in a thread other than the one that called it back (good, in fact, the state machine must be calling its thread) The rescheduled function is called), and the continuation lock must be obtained before the call.

## Event type

The events in the ATS are designed into the following four types:

- Immediate execution
   - timeout_at=0，period=0
   - Set callback_event=EVENT_IMMEDIATE via schedule_imm
- Absolutely scheduled execution
   - timeout_at>0，period=0
   - Set callback_event=EVENT_INTERVAL with schedule_at, which can be understood as executing at xx
- Relative timing execution
   - timeout_at>0，period=0
   - Set callback_event=EVENT_INTERVAL by schedule_in, which can be understood as executing in xx seconds
- Regular / periodic execution
   - timeout_at=period>0
   - Set callback_event=EVENT_INTERVAL with schedule_every

There is also a special type for implicit queues:

- Execute at any time
   - timeout_at=period<0
   - Set callback_event=EVENT_INTERVAL with schedule_every
   - Fixed EVENT_POLL event when calling Cont->handler
   - For TCP connections, this type of event is added by NetHandler::startNetEvent
      - Cont->handler for TCP events is NetHandler::mainNetEvent()

### Time values:

The task scheduling function uses a time parameter of type ink_hrtime to specify a timeout or period.
This is the nanosecond value supported by libts. You should use the time functions and macros defined in ink_hrtime.h.

The difference between the timeout parameter for schedule_at and schedule_in is:
- In the former, it is absolute time, some predetermined moment in the future
- In the latter, it is an amount relative to the current time (obtained via ink_get_hrtime)


### Cancel the rules of Event

The rules for Action are the same as the following.

- The canceler of the Event must be the state machine that the task will call back to, and the lock of the state machine needs to be held during the cancellation process.
- any reference to the Event object (eg pointer) held by the state machine shall not continue to be used after the cancel operation


### Event Codes:

- After the event is completed, the state machine SM uses the continuation handler (Cont->handleEvent) to pass in the incoming Event Codes to distinguish between the Event type and the corresponding data parameters.
- When defining Event Code, state machine implementers should handle it carefully as they affect other state machines.
- For this reason, Event Code is usually assigned uniformly.

Usually the Event Code passed when calling Cont->handleEvent is as follows:

`` `
#define EVENT_NONE CONTINUATION_EVENT_NONE // 0
#define EVENT_IMMEDIATE 1
#define EVENT_INTERVAL 2
#define EVENT_ERROR 3
#define EVENT_CALL 4 // used internally in state machines
#define EVENT_POLL 5 // negative event; activated on poll or epoll
`` `

Usually Cont->handleEvent will also return an Event Callback Code.

`` `
#define EVENT_DONE CONTINUATION_DONE // 0
#define EVENT_CONT CONTINUATION_CONT // 1
#define EVENT_RETURN 5
#define EVENT_RESTART 6
#define EVENT_RESTART_DELAYED 7
`` `

PS: But there is no judgment on the return value of Cont->handleEvent in EThread::execute().

EVENT_DONE usually indicates that the Event has successfully completed the callback operation, and the Event should be released next. (Ref: ACTION_RESULT_DONE)
EVENT_CONT usually means that the event does not complete the callback operation and needs to be retained for the next callback attempt.

## Use

### Creating an Event instance in two ways

- Global allocation
   - Event *e = ::eventAllocator.alloc();
   - The defaults are all assigned globally, because the memory is not acknowledged to which thread to process when allocating memory.
   - Constructor initializes globally_allocated(true)
   - This requires a global lock
- Local distribution
   - Event *e = EVENT_ALLOC(eventAllocator, this);
   - If you know in advance that this Event will be handed over to the current thread, you can use the local allocation method.
   - When calling the EThread::schedule\_*\_local() method, globally\_allocated=false is modified
   - Does not affect global locks, is more efficient

### Put in EventSystem

- Select the next thread according to the polling rules, then put the Event into the selected thread
   - eventProcessor.schedule(e->init(cont, timeout, period));
   - EThread::schedule(e->init(cont, timeout, period));
- Put in the current thread
   - e->schedule_*();
   - this_ethread()->schedule_*_local(e);
      - Can only be used when e->ethread==this_ethread

### Release Event

- Global allocation
   - eventAllocator.free(e);
   - ::eventAllocator.free(e);
   - I can see the above two ways in the ATS code. I understand that both are a meaning because there is only one global eventAllocator.
- Automatic judgment
   - EVENT_FREE(e, eventAllocator, t);
   - judged by e->globally_allocated

### Reschedule Event

After the state machine receives a callback from EThread, void \*data points to the Event object that triggered the callback.
After a simple type conversion, you can call e->schedule_\*() to put this Event back into the current thread.
After rescheduling, the type of Event will be reset by the schedule_\*() method.


## References

- [I_Event.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Event.h)
- [P_UnixEvent.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_UnixEvent.h)
- [UnixEvent.cc](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/UnixEvent.cc)

