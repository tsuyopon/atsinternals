# core component: NetAccept

NetVConnection and its inheritance class are used to achieve the connection between FD and VIO. How is VC created?

- Can be used as a client to create a connection to the server via connect. This is another method, we will introduce later.
- Can be used as a server, when the client connects, get a new connection through accept, this is NetAccept

NetAccept is a subsystem of NetProcessor and is the guide of the NetProcessor server.

In the state machine design of ATS, the server implementation of each state machine (SM) includes:

  - an acceptor that accepts the client's connection and creates the required type of VConnection(SM)
  - A processor that provides a large number of callback functions to handle the state of each SM (NetProcessor, HttpTransaction)

# definition

```
//
// Default accept function
//   Accepts as many connections as possible, returning the number accepted
//   or -1 to stop accepting.
//
// The following two typedef definitions are for a net_accept method
// All netaccept operations will basically call this underlying net_accept method
// But there are cases where the Server::accept() method is called directly.
typedef int(AcceptFunction)(NetAccept *na, void *e, bool blockable);
typedef AcceptFunction *AcceptFunctionPtr;
AcceptFunction net_accept;

// TODO fix race between cancel accept and call back (What is this comment? Is there a big pit not solved?)
// Create a NetAcceptAction that inherits from Action and RefCountObj
// used to return an Action to the NetAccept caller, which can be cancelled at any time in the future.
// Why support reference counting? Didn't understand...
struct NetAcceptAction : public Action, public RefCountObj {
  // Pointer to the NetAccept::server member
  Server *server;

  // Provide the only method cancel this action
  void
  cancel(Continuation *cont = NULL)
  {
    Action::cancel(cont);
    // cancel operation, directly close the listen fd. Feeling good and brutal! ! !
    // This will cause an error in the NetAccept operation. When an error is encountered, NetAccept will terminate.
    server->close();
  }

  // overloaded assignment operator
  // Since inheriting from Action and RefCountObj,
  // So here choose the operator overload that inherits the Action to change the Continuation and mutex members.
  Continuation *operator=(Continuation *acont) { return Action::operator=(acont); }

  // Destructor
  // just output a debug message
  ~NetAcceptAction() { Debug("net_accept", "NetAcceptAction dying\n"); }
};


//
// NetAccept
// Handles accepting connections.
//
// Adjusted the order of member variable definitions for ease of presentation
struct NetAccept : public Continuation {
  ink_hrtime period;
  // Used only inside the net_accept method, see the internal analysis of the net_accept method for details.
  void *alloc_cache;
  // used to mark the subscript value of listen fd in the pd array, but seems to be discarded
  //     pd->pfd[ifd].df == server.fd
  int ifd;
  
  ///////////////////////////////////
  // After creating the NetAccept instance, fill in the following parameter values ​​before starting NetAccept
  // Define a Server instance, which contains listen fd, ip_family, local_ip, local_port information
  Server server;
  // Define the accept_fn pointer function
  // If accept_fn is called by acceptFastEvent, accept_fn must point to net_accept
  // But if accept_fn is called by acceptEvent, then accept_fn can point to net_accept or to a user-defined method.
  AcceptFunctionPtr accept_fn;
  // True = callback state machine after calling do_listen() to create listen fd,
  // Pass the NET_EVENT_ACCEPT_(SUCCEED|FAILED) event and the data pointer to the NetAccept instance
  bool callback_on_open;
  // True = This is an internal WebServer implementation of ATS, just for internal use. I will go back to this backdoor section in detail.
  bool backdoor;
  // Return the Action type to the caller, and also save the Continuation pointing to the upper state machine
  For <NetAcceptAction> action_;
  // In the NetAccept run, if you get a new connection,
  // The following settings will be used to set the socket property of the new connection.
  int recv_bufsize;
  int send_bufsize;
  uint32_t sockopt_flags;
  uint32_t packet_mark;
  uint32_t packet_tos;
  // This parameter determines which type of EThread the new connection will be placed in to handle
  // For example: ET_NET, ET_SSL, ...
  EventType etype;
  // The above is the parameter that needs to be set before starting NetAccept.
  ///////////////////////////////////

  // This member seems to have been abandoned
  UnixNetVConnection *epoll_vc; // only storage for epoll events

  // When accept_thread=0, it is used to add Listen FD to the corresponding EThread epoll fd. (Reference: NetAccept::init_accept_per_thread())
  // However, in the NetHandler::mainNetEvent() method, the processing of the epd->type==EVENTIO_NETACCEPT type is not seen.
  // It seems to be obsolete, but it is not obsolete! ! !
  // Because after adding to epoll fd, once a new connection comes in,
  // will cause epoll_wait to return from the blocking wait, so that the NetAccept state machine can be called back as soon as possible
  // But I don't think it's necessary to use a class member variable to handle it. Use the temporary variables inside the function.
  EventIO ep;

  // return etype member
  virtual EventType getEtype() const;
  
  // return the global variable netProcessor
  virtual NetProcessor *getNetProcessor() const;

  // Create a DEDICATED ETHREAD and run acceptLoopEvent() when calling NetAccept
  void init_accept_loop(const char *);
  // Add a NetAccept state machine for the timed callback for the specified REGULAR ETHREAD, run acceptEvent() on callback
  virtual void init_accept(EThread *t = NULL, bool isTransparent = false);
  // Add a NetAccept state machine with a timed callback for each REGULAR ETHREAD in the ET_NET group
  // If accept_fn == net_accept, then the callback:
  // Run acceptFastEvent(), but the method does not call accept_fn
  // Otherwise: run acceptEvent(), which accepts new connections via accept_fn
  // All NetAccept state machines share the same listen fd
  virtual void init_accept_per_thread(bool isTransparent);
  // clone the current NetAccept state machine, currently only init_accept_per_thread() is used
  // Create a new object with new NetAccept and then initialize the members of the new object with a member copy
  virtual NetAccept *clone() const;
  
  // If server.fd already exists, then make the necessary settings to match the requirements of listen fd
  // Otherwise, create a set listen fd
  // This method may call back the upper state machine according to the setting of callback_on_open
  // Note: If the NetAccept mutex is set in the upper state machine, it will be invalid.
  // because do_listen will set mutex to NULL after the callback is completed.
  // This is special because the NetAccept state machine has not been delivered to any REGULAR ETHREAD at this time.
  // return 0 for success
  int do_listen(bool non_blocking, bool transparent = false);

  // main callback function in DEDICATED ETHREAD
  // Continue to call do_blocking_accept(t), when its return value is less than 0, release and destruct itself, DEDICATED ETHREAD will exit
  int acceptLoopEvent(int event, Event *e);
  // do_blocking_accept is part of acceptLoopEvent
  // Call accept to get a new connection by blocking.
  // Assign a vc object to the new connection from the global memory pool
  // Set vc->from_accept_thread = true at the same time, this will return the global memory pool when the memory occupied by vc is released.
  // return 0: when the current total connection limit is reached
  // returns -1: When NetAccept is canceled, when vc object memory fails
  // return 1: when accept_till_done==false, indicating that only one new connection is accepted at a time.
  int do_blocking_accept(EThread *t);
  
  // Call accept_fn in a non-blocking manner to get a new connection:
  // The default method net_accept pointed to by accept_fn allocates vc memory from the global memory space.
  // First try to take the action_->mutex lock, which is actually the lock of the upper state machine.
  // If the situation is special, action_->mutex is NULL, then use NetAccept's own mutex
  // After getting the lock:
  // If the NetAccept is not canceled, call the accept_fn() method:
  // Greater than 0: Successfully get a new connection, return EventSystem and wait for the next call
  // equals 0: there is currently no connection, returning EventSystem for the next call
  // Less than 0:
  // If it has been canceled: cancel the callback of this method's Event, and release NetAccept itself
  // If you don't get the lock, return to EventSystem and wait for the next call.
  // The VC state machine created by this method, the first function to be called back is acceptEvent,
  // The EVENT_NONE (synchronous) or EVENT_IMMEDIATE (asynchronous) event is passed in.
  // Then call the upper state machine by acceptEvent, and then set the state machine next callback function to mainEvent
  virtual int acceptEvent(int event, void *e);
  // Different from acceptEvent:
  // This method does not handle backdoor type connections,
  // Instead of calling the accept_fn method, use socketManager.accept to get a new connection.
  // is to allocate vc memory from the thread memory space
  // When canceled, both listen fd will be closed
  // Always synchronize the callback state machine, passing the NET_EVENT_ACCEPT event and VC
  // The first function to be called back is mainEvent, which means that the method contains work inside acceptEvent
  // When setting socket options for a new connection, it also sets the relevant method of calling socketManager directly:
  //     send buffer, recv buffer
  //     TCP_NODELAY, SO_KEEPALIVE, non blocking
  // Each time it is executed, it will repeatedly call accept to create multiple new VCs.
  // will not return to EventSystem until EAGAIN is returned, or an error occurs
  virtual int acceptFastEvent(int event, void *e);
  
  // Cancel NetAccept
  // call action_->cancel(), then close listen fd
  void cancel();

  // Constructor
  // All members are initialized to 0, false, NULL, ifd is -1
  NetAccept();
  
  // Destructor
  // action_ is an automatic pointer, actually releasing the resource
  virtual ~NetAccept() { action_ = NULL; };
};

//
// General case network connection accept code
//
// This is a generic accept_fn function
// When listen fd is non blocking, blockable will pass false; otherwise pass true
int
net_accept(NetAccept *na, void *ep, bool blockable)
{
  Event *e = (Event *)ep;
  int res = 0;
  int count = 0;
  int loop = accept_till_done;
  UnixNetVConnection *vc = NULL;

  // Non-blocking IO must ensure that net_accept is also non-blocking
  // Try to lock na->action_->mutex (usually the mutex of the upper state machine)
  // Failure to return, avoid blocking
  if (!blockable)
    if (!MUTEX_TAKE_TRY_LOCK_FOR(na->action_->mutex, e->ethread, na->action_->continuation))
      return 0;
  // do-while for accepting all the connections
  // added by YTS Team, yamsat
  // The following code is mainly trying to accept as many new connections as possible in the case of non-blocking IO
  // Usually accept returns EAGAIN to indicate that there is no new connection.
  // Explain the function of alloc_cache:
  // I think the comparison of the alloc_cache design, my understanding is as follows:
  // Since na->server.accept(&vc->con) wants to use the member con in vc,
  // So I have to apply/allocate a vc first, but once server.accept() fails, I will release the vc.
  // The author thinks this method is called frequently, so save vc to alloc_cache.
  // Whenever you want to apply/allocate a vc, use alloc_cache first.
  // However, when NetAccept is destructed, the alloc_cache is not released, so there is a memory leak bug.
  // Why isn't this alloc_cache released in the destructor?
  // Because to determine the type of vc, is UnixNetVConnection, or SSLNetVConnection, or...
  // In order to call the corresponding type of Allocator.free () method, in order to release the memory
  // However, there is no way to judge in NetAccept, but soft...
  do {
    // Get a VC
    vc = (UnixNetVConnection *) na-> alloc_cache;
    if (!vc) {
      // Allocate a VC from the thread allocation pool. Note that:
      // If na is an inheritance class of NetAccept, and getNetProcessor() is overridden within the inheritance class,
      // Then the return is not necessarily the UnixNetVConnection type, just a type conversion is done here.
      vc = (UnixNetVConnection *)na->getNetProcessor()->allocate_vc(e->ethread);
      // There is also a potential bug here, that is, there is no judgment whether the VC is successfully allocated, and there is no processing for the allocation failure.
      NET_SUM_GLOBAL_DYN_STAT(net_connections_currently_open_stat, 1);
      vc->id = net_next_connection_number();
      na-> alloc_cache = vc;
    }
    // Call the na->server.accept() method to get a new connection
    if ((res = na->server.accept(&vc->con)) < 0) {
      // judge when it fails
      // If EAGAIN indicates that there is no new connection, then jump to Ldone and return the number of new connections created by the caller this time (count)
      if (res == -EAGAIN || res == -ECONNABORTED || res == -EPIPE)
        goto Ldone;
      // Other error conditions, callback to the upper state machine, indicating that an error has occurred
      // Note: If canceled, server.fd will be closed and set to NO_FD, which will cause the accept return value to be less than 0.
      if (na->server.fd != NO_FD && !na->action_->cancelled) {
        if (!blockable)
          na->action_->continuation->handleEvent(EVENT_ERROR, (void *)(intptr_t)res);
        else {
          SCOPED_MUTEX_LOCK(lock, na->action_->mutex, e->ethread);
          na->action_->continuation->handleEvent(EVENT_ERROR, (void *)(intptr_t)res);
        }
      }
      // count is set to the error code, jump to Ldone, return the caller this error code
      count = res;
      goto Ldone;
    }
    
    // Create a new connection plus 1
    count++;
    // Set to NULL because the vc saved by alloc_cache is used
    na-> alloc_cache = NULL;

    // Initialize some parameters of vc
    vc->submit_time = Thread::get_hrtime();
    ats_ip_copy(&vc->server_addr, &vc->con.addr);
    vc->mutex = new_ProxyMutex();
    vc->action_ = *na->action_;
    vc->set_is_transparent(na->server.f_inbound_transparent);
    vc->closed = 0;
    
    // Set the VC state machine callback function, note here, whether the VC is UnixNetVConnection or SSLNetVConnection,
    // The callback function is always: UnixNetVConnection::acceptEvent
    SET_CONTINUATION_HANDLER(vc, (NetVConnHandler)&UnixNetVConnection::acceptEvent);

    // If the current EThread supports the processing of this type of VC, synchronous callback VC state machine (acceptEvent)
    // For example: when ssl accept thread=0, ET_NET = ET_SSL,
    // Then the SSLUnixNetVConnection will also be run in the thread of ET_NET
    if (e->ethread->is_event_type(na->etype))
      vc->handleEvent(EVENT_NONE, e);
    // If not supported, asynchronous callback state machine, the VC state machine is packaged into an Event, placed in the EThread that can handle it
    else
      eventProcessor.schedule_imm(vc, na->etype);
  } while (loop);

Ldone:
  // Unlock, in fact, there is no need to write here, because it is automatically unlocked when leaving the valid domain.
  if (!blockable)
    MUTEX_UNTAKE_LOCK(na->action_->mutex, e->ethread);
  // return the count value
  return count;
}

```

NetAccept can be a separate thread (DEDICATED ETHREAD mode)

  - The interior is an infinite loop
  - DEDICATED ETHREAD created by netProcessor.accept(AcceptorCont, fd, opt).
  - CONFIG proxy.config.accept_threads INT 1

It can also be a state machine that is periodically executed by Event, although there is only one state (REGULAR ETHREAD mode)

  - Repeated callbacks by EThread
  - Since REGULAR ETHREAD is also an infinite loop, it is actually placed in a large infinite loop thread.
  - CONFIG proxy.config.accept_threads INT 0

The following first analyzes the DEDICATED ETHREAD mode and then analyzes the Event mode.

## netProcessor.accept(AcceptorCont, NO_FD, opt) Process Analysis

Before creating a NetAccept thread, you first need to create an instance of the Acceptor, AcceptorCont, which is passed to netProcessor.accept().

Let's first look at how to create a DEDICATED ETHREAD to execute an instance of NetAccept.

The following process analysis simplifies part of the code for netProcessor.accept() to make the expression clearer:

```
   NetAccept *na = createNetAccept();
   // Potentially upgrade to SSL. 
   // Since NetProcessor is the base class of SSLNetProcessor, it is necessary to determine whether it needs to be upgraded from SSLVC to SSLVC.
   // Detailed analysis in the SSLNetProcessor section.
   upgradeEtype(upgraded_etype);

   // Created an Action
   na->action_ = new NetAcceptAction();
   // Note the value of na->action_ here
   // Since na->action_ is a Ptr<NetAcceptAction> type
   // NetAcceptAction inherits from Action and RefCountObj types,
   // The Action type overloads the assignment operator.
   // The NetAcceptAction type defines the overloading of the assignment operator as a direct call: Action's assignment operator overloading
   // At the same time due to the definition of the Ptr template,
   // This uses *na->action, which only makes the NetAcceptAction assignment operator overloaded
   // The assignment operator defined by the Ptr template overrides "no" to take effect
   // So the following is equivalent to:
   //     na->action_->continuation = AcceptorCont;
   //     na->action_->mutex = AcceptorCont->mutex;
   // Just modify the continuation and mutex members of na
   *na->action_ = AcceptorCont;
   // Port number 8080 is obtained through the opt structure
   // Create a separate Accept thread, the parameter [ACCEPT 0:8080] is the thread name
   na->init_accept_loop("[ACCEPT 0:8080]");
      // inside init_accept_loop
      // Set the instance function running inside a separate thread: NetAccept::acceptLoopEvent()
      // This means that DEDICATED ETHREAD will call back this method directly after the thread is created.
      SET_CONTINUATION_HANDLER(this, &NetAccept::acceptLoopEvent);
      // Call eventProcessor to create a thread
      eventProcessor.spawn_thread(NACont, thr_name, stacksize)
         // inside spawn_thread
         Event *e = eventAllocator.alloc();
         // Initialize the Event using the previous NetAccept Continuation instance
         e-> init (NACont, 0, 0);
         // Create a new EThread of type DEDICATED to execute this Event
         all_dthreads[n_dthreads] = new EThread(DEDICATED, e);
         // Reverse the connection of the EThread to the Event at the same time
         e->ethread = all_dthreads[n_dthreads];
         // Set mutex, Event, NACont mutex to use thread mutex by default
         e->mutex = e->continuation->mutex = all_dthreads[n_dthreads]->mutex;
         n_dthreads++;
         // Start the thread
         e->ethread->start(thr_name, stacksize);
            // The following is the code executed by the child thread
            // will call the executeProcessor's execute () method, in the execute to determine the thread is DEDICATED type
            // handleEvent function pointer points to NetAccept::acceptLoopEvent
            oneevent->continuation->handleEvent(EVENT_IMMEDIATE, oneevent);
         // The following is the code executed by the main thread.
         // spawn_thread ends
         return
      // end of init_accept_loop
      return
   // accept ends
   return
```

At this point, the creation of a separate accept thread is completed.

## NetAccept::acceptLoopEvent starts running, its internal logic and process are as follows:

```
int NetAccept::acceptLoopEvent(int event, Event *e)
   // In fact, the event *e passed in here is the event of alloc in the eventProcessor.spawn_thread function.
   // and t is also the new EThread in the eventProcessor.spawn_thread function.
   EThread *t = this_ethread();
   // Infinite loop calls do_blocking_accept
   while (do_blocking_accept(t) >= 0) ;
```

## NetAccept::do_blocking_accept(t) starts running and is nested inside the while loop

```
int NetAccept::do_blocking_accept(EThread *t)
   // Note that here we are running in the NetAccept class, so action_ represents AcceptorCont
   // Declare a Connection class to receive the new connection handle returned by the accept method
   // The accept method mentioned here is not a glibc function, but a method in the ATS SocketManager class.
   Connection con;
   do {
      res = server.accept(&con)
      // Use 'NULL' to Bypass thread allocator
      // Assign a VC, note that this is also related to SSLProcessor
      // If getNetProcessor returns sslNetProcessor, then the VC obtained here is SSLVC.
      // Otherwise it is a NetVC
      // Since the argument passed to allocate_vc is NULL, it is the memory allocated from the global memory space.
      vc = (UnixNetVConnection *)this->getNetProcessor()->allocate_vc(NULL);
      // Initialize VC
      vc-> with = with;
      vc->options.packet_mark = packet_mark;
      vc->options.packet_tos = packet_tos;
      vc->apply_options();
      // This value is true, indicating that the VC was created with DEDICATED ETHREAD
      // If false, the VC is created by Event.
      vc->from_accept_thread = true;
      // Assign a globally unique ID to this VC, this id is a self-increasing int type
      // net_next_connection_number() method guarantees atomic increment of this value in a multi-threaded environment
      vc->id = net_next_connection_number();
      // Copy IP structure data
      ats_ip_copy(&vc->server_addr, &vc->con.addr);
      // Set the transparent mode
      vc->set_is_transparent(server.f_inbound_transparent);
      // Create a Mutex for the new VC
      vc->mutex = new_ProxyMutex();
      // Since vc->action_ is an Action type (not a pointer, it is an instance member),
      // The Action type overloads the assignment operator.
      // So the following is equivalent to:
      //     vc->action_->continuation = action_;
      //     vc->action_->mutex = action_->mutex;
      // Note that since this->action_ is of type Ptr<NetAcceptAction>,
      // So here the right side of the equal sign is *action_ instead of action_ to avoid the reference count being reduced.
      // (Look at the definition of Ptr<> overloaded operation, it doesn't seem to be possible to write * because the left side is not Ptr<> type?)
      // Here, action_->mutex is initialized to AcceptorCont->mutex, which is usually created by new_ProxyMutex().
      vc->action_ = *action_;
      // Set the VC callback function
      SET_CONTINUATION_HANDLER(vc, (NetVConnHandler)&UnixNetVConnection::acceptEvent);
      // Push into the external queue of eventProcessor
      // And generate a signal that interrupts the sleep inside eventProcessor and the blocking of epoll_wait to facilitate timely processing of newly joined connections.
      eventProcessor.schedule_imm_signal(vc, getEtype());
   } while (loop)
```

After accepting the new connection, a new VC is created, which is bound to the UnixNetVConnection state machine (SM), which is then used by the Net handler to take over the processing of each state.

After being pushed into the external queue, it will enter the process of eventProcessor and will directly call the UnixNetVConnection::acceptEvent function.

So how do you choose one of several threads to handle? (Review the chapter on eventProcessor)

The following analyzes the code for eventProcessor.schedule_imm_signal:

```
TS_INLINE Event *
EventProcessor::schedule_imm_signal(Continuation *cont, EventType et, int callback_event, void *cookie)
{
  Event *e = eventAllocator.alloc();

  ink_assert(et < MAX_EVENT_TYPES);
#ifdef ENABLE_TIME_TRACE
  e->start_time = ink_get_hrtime();
#endif
  e->callback_event = callback_event;
  e->cookie = cookie;
  // Finally through the schedule to complete the operation of pushing into the external queue
  return schedule (e-> init (cont, 0, 0), and, true);                                                                                             
}

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

// Select ethread by polling
TS_INLINE EThread *
EventProcessor::assign_thread(EventType etype)
{
  int next;

  ink_assert(etype < MAX_EVENT_TYPES);
  if (n_threads_for_type[etype] > 1)
    next = next_thread_for_type[etype]++ % n_threads_for_type[etype];
  else
    next = 0;
  return (eventthread[etype][next]);
}
```

Through the above analysis, we confirm that when the new connection is encapsulated by Event, when it is placed in the external queue of the EThread thread, it is selected by the load algorithm that polls multiple EThreads.

So which EThread will be polled? Determined by the return value of getEtype():

  - This value is ET_NET for UnixNetVConnection type
  - This value is ET_SSL for SSLNetVConnection type

AcceptorCont is the guide for our custom higher-level state machine. UnixNetVConnection::acceptEvent will call the callback function in vc->action_

This allows AcceptorCont to initialize a higher level state machine.

## DEDICATED ETHREAD Mode Summary

In netProcessor.accept()

  - Created an instance of NetAccept via createNetAccept()
  - Create DEDICATED ETHREAD with init_accept_loop()

DEDICATED ETHREAD

  - init_accept_loop creates DEDICATED ETHREAD
    - acceptLoopEvent
      - do_blocking_accept
        - server.accept

Continuously accept and create new connections and VCs, and finally call schedule_imm_signal(vc, getEtype()) to put the VC into the REGULAR ETHREAD thread group of the EventSystem type.

REGULAR ETHREAD

  - External Local Queue
    - UnixNetVConnection::acceptEvent
      - action_->cont->handleEvent(NET_EVENT_ACCEPT, vc)
  - Event Queue
    - InactivityCop::check_inactivity
      - vc->handleEvent(EVENT_IMMEDIATE, e) ==> UnixNetVConnection::mainEvent(EVENT_IMMEDIATE, e)

See above that acceptEvent will call back to the upper state machine, and send the NET_EVENT_ACCEPT event. Then the callback function of the VC state machine will switch to mainEvent, and cooperate with the Inactivity state machine to implement timeout control.

## REGULAR ETHREAD Mode Introduction

Create a NetAccept state machine for the specified thread group, such as ET_NET, via the NetAccept::init_accept_per_thread method

  - init_accept_per_thread
    - do_listen
    - clone
    - schedule_every(na, period, etype)

The callback function is usually set to acceptFastEvent, and the event passed in the callback is etype. If accept_fn uses a user-defined function, the callback function is acceptEvent.

Since thread->schedule_every is used here, and period is less than 0, the event is directly put into the Negative Queue.

REGULAR ETHREAD

  - Negative Queue
    - acceptFastEvent(etype, na)
      - socketManager.accept
      - nh->open_list.enqueue(vc)
      - nh->read_ready_list.enqueue(vc)
      - action_->cont->handlerEvent(NET_EVENT_ACCEPT, vc)

Calling acceptFastEvent is equivalent to including the processing part of acceptEvent, but there may be a bug here, because when you operate the nh (NetHandler) queue, you don't get the NetHandler mutex lock.

Although there is an assert() before the operation of these two queues to ensure that ATS stops running when such an exception occurs, why not support asynchronous calls?

```
ink_assert(vc->nh->mutex->thread_holding == this_ethread());
```

  - Negative Queue
    - acceptEvent(etype, na)
      - accept_fn ==> net_accept default
        - na->server.accept
        - 同步 vc->handleEvent(EVENT_NONE, e) ==> UnixNetVConnection::acceptEvent(EVENT_NONE, e)
          - action_->cont->handleEvent(NET_EVENT_ACCEPT, vc)
        - 异步 eventProcessor.schedule_imm(vc, na->etype)
  - External Local Queue
    - UnixNetVConnection::acceptEvent(EVENT_IMMEDIATE, e)
      - action_->cont->handleEvent(NET_EVENT_ACCEPT, vc)

If you use the acceptEvent callback function, you still go with the acceptEvent. Because of the lock on the NetHandler in the acceptEvent and the rescheduling of the lock failure, there is no bug of acceptFastEvent.

Both synchronous and asynchronous modes are supported when the NET_EVENT_ACCEPT callback is performed on the upper state machine.

## Creating a separate NetAccept state machine

Sometimes, we don't need to create a NetAccept state machine for every thread in the thread group, so what should we do?

If you pay attention, there is a NetAccept method in the above introduction: init_accept()

When we need to create a NetAccept state machine in the specified REGULAR ETHREAD, we only need to first create a NetAccept instance of the specified type.

For example: if you want to accept SSL after receiving a new connection, then:

  - sslna = new SSLNetAccept
  - sslna->init_accept(thread, isTransparent)

It should be noted that the thread here must be of the type corresponding to na->etype, or you can be lazy, use NULL instead of thread, and then find a suitable thread (assign_thread) according to the etype value.

At this point, a periodically running event is generated, the NetAccept state machine is placed in the thread, and the callback function is set to NetAccept::acceptEvent.

If you only want to accept a connection, you can call sslna->cancel() to cancel the NetAccept state machine after receiving the NET_EVENT_ACCEPT event.

## Introducer (Acceptor)

The introducer is a series of simple classes, usually:

  - There are only one or two methods inside
  - Each type of guide (Acceptor) corresponds to only one state machine (SM)

The director is mainly responsible for:

  - Create an instance of a state machine (SM)
  - After the creation is successful, use the state machine (SM) processor as the callback function of the new VC

Since the director is responsible for creating the state machine (SM), after the importer (Acceptor) is created,

  - usually not released
  - ATS may crash even if a serious error is encountered

The processor is responsible for releasing and destroying the state machine.

Multiple director

  - When the TCP link is established, it is necessary to determine when the upper state machine (SM) to be created has multiple types.
  - Need to read a user's request, depending on the type of request to determine which upper state machine (SM) needs to be created
  - At this point, AcceptorCont points to the second re-director, which simply analyzes the data and then creates the corresponding upper-level state machine (SM).
  - Can refer to the related design of ProtocolProbeSessionAccept

When the server accepts a new connection as a server side, it passively creates the state machine's pre-state machine. If it needs to be the client side (Client Side) to initiate an external connection within the ATS, how is it handled? Please see the introductory chapter of UnixNetVConnection.

## References

- [P_NetAccept.h](https://github.com/apache/trafficserver/tree/master/iocore/net/P_NetAccept.h)
- [UnixNetAccept.cc](https://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetAccept.cc)
