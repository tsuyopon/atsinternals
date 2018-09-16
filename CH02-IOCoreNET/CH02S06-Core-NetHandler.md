# Core component: NetHandler

Specially handle network IO related operations, get the fd collection to be processed from the poll fd array, and then callback VConnection according to the state (read, write):

  - UnixNetVConnection
  - SSLNetVConnection
  - DNSConnection

There is also a special one:

  - ASYNC_SIGNAL
  - Waiting to interrupt epoll_wait

There are eight queues in NetHandler:

  - Que: open_list
    - Save all VCs managed by the current EThread
  - Dlist: cop_list
    - Traverse the open_list in the InactivityCop state machine, put the vc of vc->thread==this_ethread() into this cop_list
    - Then check the timeout of vc in cop_list one by one.
    - PS: After reviewing the code, I found that vc in open_list does not exist when vc->thread is not this_ethread().
  - Que: active_queue
    - The upper state machine, such as HttpSM, puts vc through add_to_active_queue for inactive timeout
    - In InactivityCop, clean up by manage_active_queue
    - Also provided is the remove_from_active_queue method to remove vc from the queue
    - Note: active_queue and keep_alive_queue are mutually exclusive. A NetVC cannot appear in both queues at the same time. This is guaranteed by add_(keep_alive|active)_queue.
  - Que: keep_alive_queue
    - The upper state machine, such as HttpSM, puts vc through add_to_keep_alive_queue for achieving keep alive timeout
    - In InactivityCop, clean up by manage_keep_alive_queue
    - Also provided is the remove_from_keep_alive_queue method, remove vc from the queue 
    - for active_queue and keep_alive_queue, will be analyzed in detail in InactivityCop
  - ASLLM: (read|write)_enable_list
    - When performing asynchronous reenable, put vc directly into the atomic queue
    - In EThread A, you need to reenable a VC, but this VC is managed by EThread B. At this time, it is asynchronous reenable.
  - QueM: (read|write)_ready_list
    - When NetHandler::mainNetEvent is executed, all (read|write)_enable_list will be fetched first and imported into (read|write)_ready_list
    - In addition, when performing synchronous reenable, vc will also be placed directly into this queue.
    - In EThread A, you need to reenable a VC. This VC is also managed by EThread A. At this time, it belongs to synchronous reenable.

The two queues (read|write)_enable_list are atomic queues, which can be operated without locks. The other six queues need to be locked to operate.

Both UnixNetVConnection and SSLNetVConnection may flow in the above queues:

![NetVC Routing Map](https://cdn.rawgit.com/oknet/atsinternals/master/CH02-IOCoreNET/CH02-IOCoreNet-002.svg)

## definition

```
//
// NetHandler
//
// A NetHandler handles the Network IO operations. It maintains
// lists of operations at multiples of it's periodicity.
//
class NetHandler : public Continuation
{
public:
  Event *trigger_event;

  // Read and write the vc queue of the Ready List
  // means to link UnixNetVConnection->read.ready_link to form a queue, UnixNetVConnection->read is a NetState type
  QueM(UnixNetVConnection, NetState, read, ready_link) read_ready_list;
  // means to link UnixNetVConnection->write.ready_link to form a queue, UnixNetVConnection->write is a NetState type
  QueM(UnixNetVConnection, NetState, write, ready_link) write_ready_list;

  // InactivityCop connection timeout control
  // Add vc to the open_list queue in the following method
  // NetAccept::acceptFastEvent
  // UnixNetVConnection::acceptEvent
  // UnixNetVConnection::connectUp
  // In the close_UnixNetVConnection method:
  // will remove vc from open_list, cop_list, enable list, and ready list
  // Also call the relevant method to remove vc from the keep alive and active queues
  // open_list saves every UnixNetVConnection opened by the current EThread
  Que(UnixNetVConnection, link) open_list;
  // In the InactivityCop::check_inactivity method (this method runs once a minute):
  // First traverse the open_list queue, press all vc into cop_list
  // Then pop vc from cop_list one by one, callback on timeout vc
  // Finally rearrange the active and keep alive queues
  DList (UnixNetVConnection, cop_link) cop_list;

  // Read and write the vc queue of the Enable List, support atomic operations
  // means to link UnixNetVConnection->read.enable_link to form a queue, UnixNetVConnection->read is a NetState type
  ASLLM (UnixNetVConnection, NetState, read, enable_link) read_enable_list;
  // means to link UnixNetVConnection->write.enable_link to form a queue, UnixNetVConnection->write is a NetState type
  ASLLM (UnixNetVConnection, NetState, write, enable_link) write_enable_list;

  // keep alive queue and its length
  // Managed by several methods:
  // NetHandler::add_to_keep_alive_queue()
  // NetHandler::remove_from_keep_alive_queue()
  // NetHandler::manager_keep_alive_queue()
  Que(UnixNetVConnection, keep_alive_queue_link) keep_alive_queue;
  uint32_t keep_alive_queue_size;
  // active queue and its length
  // Managed by several methods:
  // NetHandler::add_to_active_queue()
  // NetHandler::remove_from_active_queue()
  // NetHandler::manager_active_queue()
  Que(UnixNetVConnection, active_queue_link) active_queue;
  uint32_t active_queue_size;

  // Used to limit the total size of the keep alive and active queues:
  // set in NetHandler::configure_per_thread()
  // Verify in NetHandler::manage_keep_alive_queue()
  // Verify in NetHandler::manage_active_queue()
  // keep alive + active The total length of the two queues cannot exceed max_connections_per_thread_in
  // configuration parameter: proxy.config.net.max_connections_in
  // Maximum number of connections allowed per EThread = max_connections_in / threads
  uint32_t max_connections_per_thread_in;
  // configuration parameter: proxy.config.net.max_connections_active_in
  // The maximum number of active connections allowed per EThread = max_connections_active_in / threads
  uint32_t max_connections_active_per_thread_in;
  // Note: The above calculation of the number of threads, including ET_NET and ET_SSL two types of EThread

  // The following 5 values ​​are set by configuration parameters for managing active and keep alive queues
  // configuration settings for managing the active and keep-alive queues
  uint32_t max_connections_in;
  uint32_t max_connections_active_in;
  uint32_t inactive_threashold_in;
  uint32_t transaction_no_activity_timeout_in;
  uint32_t keep_alive_no_activity_timeout_in;

  // may have been abandoned
  time_t sec;
  int cycles;

  // Initialize the function, initialize the configuration variables, etc.
  int startNetEvent(int event, Event *data);
  // main handler, traversing the poll fd array, executing the callback
  int mainNetEvent(int event, Event *data);
  // no actual definition, deprecated
  int mainNetEventExt(int event, Event *data);
  // Import the Enable List into the Ready List
  void process_enabled_list(NetHandler *);

  // Manage the keep alive queue
  void manage_keep_alive_queue();
  // Manage the active queue
  bool manage_active_queue();
  // Add VC to the keep alive queue
  void add_to_keep_alive_queue(UnixNetVConnection *vc);
  // Remove VC from the keep alive queue
  void remove_from_keep_alive_queue(UnixNetVConnection *vc);
  // Add VC to the active queue
  bool add_to_active_queue(UnixNetVConnection *vc);
  // Remove VC from the active queue
  void remove_from_active_queue(UnixNetVConnection *vc);

  // set max_connections_per_thread_in and max_connections_active_per_thread_in
  void configure_per_thread();

  // Constructor
  netHandler();

private:
  // Forced to close VC
  void _close_vc(UnixNetVConnection *vc, ink_hrtime now, int &handle_event, int &closed, int &total_idle_time,
                 Int &total_idle_count);
};
```

## Enable_list and ready_list

The enable_list is a queue for temporarily saving the VC to be activated when the UnixNetVConnection::reenable() is executed and the nethandler mutex cannot be obtained.

```
struct NetState {
  // Indicates whether the current VIO is active
  volatile int enabled;
  // VIO instance
  VIO vio;
  // Define a doubly linked list node, you can connect multiple UnixNetVConnection, indirectly connect NetState
  // ready_link has two members: UnixNetVConnection *prev and UnixNetVConnection integrated from SLINK *next
  Link<UnixNetVConnection> ready_link;
  // Define a singly linked list node, you can connect multiple UnixNetVConnection, indirectly connect NetState
  // enable_link has one member: UnixNetVConnection *next
  SLink<UnixNetVConnection> enable_link;
  
  // indicates that this vc was put into the enable_list
  int in_enabled_list;
  
  // indicates that this vc is returned by epoll_wait
  int triggered;

  // Constructor
  // This uses VIO::NONE to call the VIO constructor to initialize the vio member.
  NetState() : enabled(0), vio(VIO::NONE), in_enabled_list(0), triggered(0) {}
};

class UnixNetVConnection : public NetVConnection
{
...
  // Define two network states, Read and Write, which contain two linked list nodes, ready_link and enable_link.
  NetState read;
  NetState write;

  LINK (UnixNetVConnection, cop_link);

  // Start to define the linked list operation class, does not define an instance, just define the type
  // with a representation of M
  // 1. Only define the Class type, not the instance (not ending with M, such as LINK, after defining the type, also declare an instance)
  // 2. Indicates that the linked list type of the member is defined. The Class Name is: Link_Member_LinkName
  // 3. Usually defined as: XXXM (Class, Member, LinkName)
  // 4. The definition method that is different from without M is: XXX(Class, LinkName); <-- Finally, there must be a semicolon
  // Define Class Link_read_ready_link : public Link<UnixNetVConnection>
  LINKM (UnixNetVConnection, read, ready_link)
  // Define Class Link_read_enable_link : public SLink<UnixNetVConnection>
  SLINKM (UnixNetVConnection, read, enable_link)
  // Define Class Link_write_ready_link : public Link<UnixNetVConnection>
  LINKM (UnixNetVConnection, write, ready_link)
  // Define Class Link_write_enable_link : public SLink<UnixNetVConnection>
  SLINKM (UnixNetVConnection, write, enable_link)
  // End
  // Some don't quite understand why there is only a type defined here, not an instance?

  LINK (UnixNetVConnection, keep_alive_queue_link);
  LINK (UnixNetVConnection, active_queue_link);
...
};

class NetHandler : public Continuation
{
...
  // Connect to UnixNetVConnection type, FIFO queue (enqueue/dequeue) can also be FILO (push/pop), you need to lock before you can operate
  QueM(UnixNetVConnection, NetState, read, ready_link) read_ready_list;
  // Expand as: Queue<UnixNetVConnection, UnixNetVConnection::Link_read_ready_link> read_ready_list;
  QueM(UnixNetVConnection, NetState, write, ready_link) write_ready_list;
  // Expand as: Queue<UnixNetVConnection, UnixNetVConnection::Link_write_ready_link> write_ready_list;
  // further analysis:
  // The push method of write_ready_list calls DLL<UnixNetVConnection, UnixNetVConnection::Link_write_ready_link>::push method
  // When the head inside the link is empty, the next method is called, the next pointer of the element to be inserted is obtained, the pointer is pointed to the head, and the head is pointed to the newly placed element.
  // When the head in the link is not empty, the prev method is called to get the prev pointer of the head element, then the pointer points to the new element, then the next pointer of the new element points to the head element, and then the head points to the new element.
  // The next method above is the UnixNetVConnection called::Link_write_ready_link::next_link()
  // returned is a UnixNetVConnection instance -> write.ready_link.next;
  // The above prev method is called UnixNetVConnection::Link_write_ready_link::prev_link()
  // returned is a UnixNetVConnection instance -> write.ready_link.prev;
  // where write is a NetState type ready_link is a member of its Link<UnixNetVConnection> type
...
  / / Connect UnixNetVConnection type, support LIFO unidirectional linked list of atomic operations, only operate in the head of the linked list, direct operation without locking
  ASLLM (UnixNetVConnection, NetState, read, enable_link) read_enable_list;
  // Expand as: AtomicSLL<UnixNetVConnection, UnixNetVConnection::Link_read_enable_link> read_enable_list;
  ASLLM (UnixNetVConnection, NetState, write, enable_link) write_enable_list;
  // Expand as: AtomicSLL<UnixNetVConnection, UnixNetVConnection::Link_write_enable_link> write_enable_list;
...
};
```

If the VC does not operate across threads, then the following members of the NetHandler can meet the requirements:

  - Ready_list saves all vc returned by epoll_wait
  - Traversing the ready_list will handle all network events
  - When reenable, you only need to add vc to the ready_list.

In the multi-threaded design, cross-thread resource access is difficult to avoid, but the cross-thread locking operation will be slower, but also consider the asynchronous situation of EThread execution when the lock is not available.

  - NetHandler implements enable_list, two atomic queues for read and write operations, respectively.
  - This allows you to add vc directly to the enbale_list across threads.
  - Then NetHandler then transfers vc from the enable_list to the ready_list
 
However, we noticed that NetHandler's mutex did not inherit from EThread, but created a new one via new_ProxyMutex.

  - As a result, EThread may not get the lock when calling NetHandler, then it will schedule rescheduling in process_event
  - The advantage is that another thread can get the lock if you want to access the NetHandler.

problem:

  - If other threads can get the NetHandler lock, why design an atomic queue?
  - If there is an atomic queue, why is the NetHandler mutex not inherited from EThread?

## The main function NetHandler::mainNetEvent(int event, Event *e) Process analysis:

First, call process_enabled_list(this)the enable_list dump ready_list. It is clear in the comments before the function that this step is to move the VCs enabled in other threads to the ready_list.

mainNetEvent() is the callback function of the NetHandler state machine, and there will be a NetHandler instance in each thread. The mutex of the state machine NetHandler will be locked during the callback. Only the state machine running in this thread can operate its internal data.

When we need to process an associated VC in the state machine (SM), but this VC is managed in another thread, then it involves the operation of cross-threading. At this time, because the ready_list cannot be directly manipulated, ATS IOCore implements a The middle queue enable_list.

Since the enable_list supports atomic operations, it can be operated across threads without locking. Just need to import the contents of the current enable_list first when processing the ready_list.

```
//
// Move VC's enabled on a different thread to the ready list
//
void
NetHandler::process_enabled_list(NetHandler *nh)
{
  UnixNetVConnection *vc = NULL;

  SListM(UnixNetVConnection, NetState, read, enable_link) rq(nh->read_enable_list.popall());
  while ((vc = rq.pop())) {
    vc->ep.modify(EVENTIO_READ);
    vc->ep.refresh(EVENTIO_READ);
    vc->read.in_enabled_list = 0;
    if ((vc->read.enabled && vc->read.triggered) || vc->closed)
      nh->read_ready_list.in_or_enqueue(vc);
  }

  SListM(UnixNetVConnection, NetState, write, enable_link) wq(nh->write_enable_list.popall());
  while ((vc = wq.pop())) {
    vc->ep.modify(EVENTIO_WRITE);
    vc->ep.refresh(EVENTIO_WRITE);
    vc->write.in_enabled_list = 0;
    if ((vc->write.enabled && vc->write.triggered) || vc->closed)
      nh->write_ready_list.in_or_enqueue(vc);
  }
}
```

It can be seen that only the VC's NetState property enabled and triggered is set to true, or the VC property closed is set to true to be added to the ready_list queue.

The NetState is divided into two states: read and write. The enable_list and the ready_list are also divided into two queues: read and write. Therefore, there are four queues.

Then, set the value of poll_timeout for the epoll_wait wait time and call epoll_wait

```
  if (likely(!read_ready_list.empty() || !write_ready_list.empty() || !read_enable_list.empty() || !write_enable_list.empty()))
    poll_timeout = 0; // poll immediately returns -- we have triggered stuff to process right now
  else
    poll_timeout = net_config_poll_timeout;

  PollDescriptor *pd = get_PollDescriptor(trigger_event->ethread);
  UnixNetVConnection *vc = NULL;
#if TS_USE_EPOLL
  pd->result = epoll_wait(pd->epoll_fd, pd->ePoll_Triggered_Events, POLL_DESCRIPTOR_SIZE, poll_timeout);
  NetDebug("iocore_net_main_poll", "[NetHandler::mainNetEvent] epoll_wait(%d,%d), result=%d", pd->epoll_fd, poll_timeout,
           pd->result);
```

It can be seen that the ATS is really fine to the extreme. If the ready_list is not empty, set the poll_timeout to 0, so that the connection in the ready_list can be processed as quickly as possible.

Next, the data returned by epoll_wait is processed. See the comments in the code below for a detailed analysis.
```
  vc = NULL;
  for (int x = 0; x < pd->result; x++) {
    epd = (EventIO *)get_ev_data(pd, x);
    / / First deal with ordinary VC read and write events
    if (epd->type == EVENTIO_READWRITE_VC) {
      vc = epd->data.vc;
      // If it is a read event, or there is an error
      if (get_ev_events(pd, x) & (EVENTIO_READ | EVENTIO_ERROR)) {
        // Set VC's read NetState property triggered to true
        vc->read.triggered = 1;
        if (!read_ready_list.in(vc))
          / / If not in the ready_list queue, put it in the queue
          read_ready_list.enqueue(vc);
        else if (get_ev_events(pd, x) & EVENTIO_ERROR) {
          // If in the ready_list queue, but this time there is an ERROR, output a DEBUG message
          // check for unhandled epoll events that should be handled
          debug("iocore_net_main", "Unhandled epoll event on read: 0x%04x read.enabled=%d closed=%d read.netready_queue=%d",
                Get_ev_events(pd, x), vc->read.enabled, vc->closed, read_ready_list.in(vc));
        }
      }
      vc = epd->data.vc; // This line is redundant? ? ? ? ? ?
      // If it is a write event, or there is an error
      if (get_ev_events(pd, x) & (EVENTIO_WRITE | EVENTIO_ERROR)) {
        // Set VC's write NetState property triggered to true
        vc->write.triggered = 1;
        if (!write_ready_list.in(vc))
          // If not in the ready_list queue, put it in the queue
          write_ready_list.enqueue(vc);
        else if (get_ev_events(pd, x) & EVENTIO_ERROR) {
          // If in the ready_list queue, but this time there is an ERROR, output a DEBUG message
          // check for unhandled epoll events that should be handled
          debug("iocore_net_main", "Unhandled epoll event on write: 0x%04x write.enabled=%d closed=%d write.netready_queue=%d",
                Get_ev_events(pd, x), vc->write.enabled, vc->closed, write_ready_list.in(vc));
        }
      // If there is no error in writing an event. Not all reading events here will trigger? ? ? ? ? ?
      // The following if statement has a bug, has been submitted to the official and fixed: TS-3969
      // This bug only affects the output of the debug and does not affect the ATS workflow.
      } else if (!(get_ev_events(pd, x) & EVENTIO_ERROR)) {
        // output a DEBUG message
        debug("iocore_net_main", "Unhandled epoll event: 0x%04x", get_ev_events(pd, x));
      }
    // then handle the DNS connection event
    } else if (epd->type == EVENTIO_DNS_CONNECTION) {
      if (epd->data.dnscon != NULL) {
        epd->data.dnscon->trigger(); // Make sure the DNSHandler for this con knows we triggered
#if defined(USE_EDGE_TRIGGER)
        epd->refresh(EVENTIO_READ);
#endif
      }
    }
    // then process the asynchronous signal
    else if (epd->type == EVENTIO_ASYNC_SIGNAL) {
      net_signal_hook_callback(trigger_event->ethread);
    }
    ev_next_event(pd, x);
  }

  // All epoll events are processed, and finally the event is cleared.
  pd->result = 0;
```

Next, the ready_list is processed, the read is processed first, and then the write is processed. The following takes the processing code of the epoll ET mode as an example:

```
#if defined(USE_EDGE_TRIGGER)
  // Traversing the read_ready_list queue
  while ((vc = read_ready_list.dequeue())) {
    if (vc->closed)
      // vc will be removed from both read_ready_list and write_ready_list in close_UnixNetVConnection().
      // So don't worry about repeating vc->closed when dealing with write_ready_list below.
      close_UnixNetVConnection(vc, trigger_event->ethread);
    else if (vc->read.enabled && vc->read.triggered)
      // If enabled and triggered are both true, then net_read_io is called to read.
      // In NET_read_io(), Continuation will be called and events such as READ_READY, READ_COMPLETE will be passed.
      // Note:
      //    In net_read_io() it is possible to put vc back to the end of the read_ready_list queue.
      //    If not handled properly, it may cause this while loop to run for a long time, delay will not handle the write operation,
      //    And will not return to EventSystem, which will cause this ET_NET thread to block.
      vc->net_read_io(this, trigger_event->ethread);
    else if (!vc->read.enabled) {
      // If enabled is false, then remove this vc from read_ready_list. Triggered is usually true at this time.
      // Why does this happen? ? ? ? ? ?
      //    Normally when enabled is false, remove will also be called to remove it from the queue.
      //    vc, enabled, which can usually be epoll_wait must be set to true.
      read_ready_list.remove(vc);
    }   
  }
  while ((vc = write_ready_list.dequeue())) {
    if (vc->closed)
      // In close_UnixNetVConnection(), vc will be removed from both read_ready_list and write_ready_list.
      close_UnixNetVConnection(vc, trigger_event->ethread);
    else if (vc->write.enabled && vc->write.triggered)
      // If enabled and triggered are both true, then write_to_net is called for write operation.
      // Call Continuation in write_to_net() and pass WRITE_READY, WRITE_COMPLETE and other events.
      // Note:
      //     In write_to_net() it is possible to put vc back at the end of the write_ready_list queue.
      //     If not handled properly, it may cause this while loop to run for a long time, delay will not return to EventSystem,
      //     will cause this ET_NET thread to block.
      write_to_net(this, vc, trigger_event->ethread);
    else if (!vc->write.enabled) {
      // If enabled is false, then remove this vc from write_ready_list. Triggered is usually true at this time.
      // Why does this happen? ? ? ? ? ?
      //     Normally when enabled is false, remove will also be called to remove it from the queue.
      //     vc, enabled, which can usually be epoll_wait must be set to true.
      write_ready_list.remove(vc);
    }
#else
```

## Net_read_io and write_to_net

In the net_read_io method, different processing is performed by judging the state of the MIOBuffer and calling the return value of the readv() function:

- If the MIOBuffer is full, then this read event will be ignored and the READ VIO of this VC will be disabled. Unless vio->reenable() is restored, the read event will not be triggered again on this VC.
- If MIOBuffer has space to hold new data, then readv() is called to read the data, which returns the value:
   - Equivalent to -EAGAIN or -ENOTCONN, removes the VC from the read ready queue, but still causes epoll to wait for the EVENTIO_READ event on this VC. return
   - Equal to 0 or -ECONNRESET, remove VC from read ready queue, trigger VC_EVENT_EOS event, return
   - Greater than 0 and vio.ntodo() is less than or equal to 0, triggering VC_EVENT_READ_COMPLETE, returning
   - Greater than 0 and vio.ntodo() is greater than 0, triggers the VC_EVENT_READ_READY event, then calls read_reschedule to put this VC back into the read_ready_list queue, returning

Need to pay attention to the last VC_EVENT_READ_READY event, when the VC is put back into the read_ready_list queue, then in the while loop of the mainNetEvent, the net_read_io method will be called again, as long as the VC in the read_ready_list queue continues to have data readable, then it will always be This while loop. So it does not enter the while loop of write_ready_list.

The performance in the ATS is that the ATS only receives data and does not forward the data. Unless all the data is received, it will start to send data out. This phenomenon is more obvious when the network speed is relatively fast. In the case of a bad network speed, since the data cannot be guaranteed to arrive continuously, the readv may soon encounter an -EAGAIN error, so this problem rarely occurs. Especially when implementing the tunnel, you must consider this situation.

In the write_to_net method, different processing is performed by judging the state of MIOBuffer and calling the return value of the load_buffer_and_write() function:

  - If vio.ntodo==0
    - Then this write event will be ignored, and the WRITE VIO of this VC will be disabled. Unless it is restored by vio->reenable(), the write event will not be triggered again on this VC and will be returned.
  - If this output does not complete VIO (that is, it does not trigger VC_EVENT_WRITE_COMPLETE), and MIOBuffer has room to write data:
    - The VC_EVENT_WRITE_READY event is fired first, causing the state machine to try to fill the MIOBuffer.
  - If there is still no data in the MIOBuffer, disable the WRITE VIO of this VC and return.
  - If the MIOBuffer has data for output, then load_buffer_and_write() is called to write the data, which returns the value:
    - Equivalent to -EAGAIN or -ENOTCONN, return as follows according to the value of need:
      - Need & EVENTIO_WRITE == true, removes the VC from the write ready queue, but still causes epoll to wait for the EVENTIO_WRITE event on this VC.
      - Need & EVENTIO_READ == true, removes the VC from the read ready queue, but still causes epoll to wait for the EVENTIO_READ event on this VC.
    - Equal to 0 or -ECONNRESET, trigger VC_EVENT_EOS event, return
    - Greater than 0 and vio.ntodo() is less than or equal to 0, triggering VC_EVENT_WRITE_COMPLETE, returning
    - Greater than 0 and vio.ntodo() is greater than 0,
      - If the VC_EVENT_WRITE_READY event was previously triggered and the state machine sets WBE EVENT, then WBE EVENT is triggered and returned.
      - If the VC_EVENT_WRITE_READY event has not been triggered before, the VC_EVENT_WRITE_READY event is fired, then the write_reschedule is called to put the VC back into the write_ready_list queue, returning
  - At last
    - After the write operation, if the MIOBuffer is judged to be empty again, the VC's WRITE VIO is disabled and returned.
    - According to the value of need, put vc back into the ready list queue
      - If you put back the write ready list, it will still be called by the while loop.
      - If you put back the ready list, it will be processed the next time the EventSystem is called.
      - Guess here is set for SSL to quickly complete the handshake operation. **


There is no problem with read_from_net here, because the write operation is more likely to encounter EAGAIN. However, it is not excluded that in high-speed network environments, such as 10 Gigabit NICs, the same problems as read_from_net may occur.

Re-enabling the epoll wait event will put the VC into the ready_list queue the next time EventSystem calls NetHandler::mainNetEvent, and triggered to true, but the enabled situation is not good.

## Why should we divide into two queues: read ready and write ready?

This is the communication layer of the core of iocore, to ensure that each call can be quickly returned, in order to be able to handle new events with EventSystem in a timely manner.

  - So the frequency of EThread callback NetHandler state machine is very high (as long as EThread is idle)
  - This can make the number of VCs returned by epoll_wait less, and can quickly process the callback of reading and writing two types of events.

Here, read is the producer, put the input data into the cache, write is the consumer, responsible for consuming the data in the cache.

In order to achieve high performance, producers will be designed to do their best to fill the buffer, and consumers will do their best to consume the buffer data. and so:

  - Once the buffer has free space, the producer is called to fill the buffer.
  - Once the buffer has data, the consumer is called to consume the data in the buffer.

When there are multiple producers and consumers, it is a cycle to batch process the producer's application, complete the production, and then process the consumer's application in batches, complete the consumption, and return to EventSystem. Such batch design requires a site (memory) to display the data produced by all producers, and then concentrated consumption by consumers, then the site (memory) will be occupied more.

Is there a problem with the design of ATS?

  - If mass production, the production data needs to be temporarily stacked in one place (requires memory space to store)
    - Then bulk consumption, vacating the heap (freeing memory)
    - Then you will see that the ATS memory usage should have a peak and a trough.
  - Sometimes to receive a large file, it will continue to produce
    - ATS does not limit the number of productions, causing the consumption process to be delayed

If you consume while producing?

  - There is no way to process two queues concurrently in one thread, so only producers and consumers can take turns to produce and consume.
  - Since it is also consumed during the production process, the required space is not as big as before.

But is there a problem with this approach?

  - If a couple of producers and consumers have a special temper
    - One production, one consumption
    - Reproduction, re-consumption
    - Until the producer no longer produces data
  - The result is that the occupied time of the venue becomes longer, and if the venue is rented by time, the cost is also high.
  - If the content produced by the first producer is A, and the content that the first consumer wants is B
    - The consumer will leave and will not come back unless there is a producer calling it back
    - Then the consumption efficiency is reduced
  - The result is that the last consumer left, there is still a lot of data piled up in the field, no one wants.

As mentioned earlier, the time taken by this nethandler to handle read and write operations can't be too long, otherwise EventSystem can't handle new events in time.

  - If it is a model of production while producing, the time processed in the nethandler is completely uncontrolled.
  - Therefore, this does not meet the requirements of EventSystem for the IO subsystem.

Therefore, in the IO processing of ATS, it is divided into two stages of reading and writing:

  - First read data to memory
  - Then process the data
  - Then write the data
  - Finally reclaim temporary memory space

This logic is the simplest and clearest.

Since the IO core of ATS does not limit the number of read and write triggers on a VC, in order to avoid loop processing IO read and write, in the READ READY and WRITE READY handlers:

  - READ READY handler
    - Only limited content should be copied from the read buffer (site) to the write buffer (site)
    - Realize control of read traffic while controlling data usage of memory
  - WRITE READY handler
    - Make sure the following two conditions are not met at the same time
      - Write buffer (site) is empty
      - Read buffer (site) full
    - Once satisfied at the same time, it will cause VC to be killed because:
      - The write buffer is empty and VIO Write is automatically disabled.
      - When the read buffer is full, VIO Read will be automatically banned
    - Usually read and write will share the same buffer (for example: Single Buffer Tunnel)
      - The above two conditions can never be met at the same time.
    - However, in some special designs, separate buffers are used.
      - You must pay special attention to not let the above two conditions meet at the same time.

## Reference material

![NetHandler - NetVC - VIO - UpperSM](https://cdn.rawgit.com/oknet/atsinternals/master/CH02-IOCoreNET/CH02-IOCoreNet-001.svg)

- [P_UnixNet.h]
(http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
- [P_UnixPollDescriptor.h]
(http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixPollDescriptor.h)
- [UnixNetVConnection.cc]
(http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetVConnection.cc)
- [UnixNet.cc]
(http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNet.cc)

