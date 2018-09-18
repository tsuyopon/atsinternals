# core component: InactivityCop state machine

InactivityCop is an improvement to the early timeout mechanism, which reduces the pressure on EventSystem.

The original timeout control is completely dependent on the EventSystem's timed events, resulting in a large number of events in the EventSystem event queue.

For two timeout types of a NetVConnection, a 10K connection has a 20K event that is permanently stuck in the internal queue of the EventSystem.

So ATS, or earlier Inktomi, refactored the timeout management and proposed the InactivityCop state machine, which is specifically designed to support two timeouts for NetVConnection.

For some discussion of this part of the improvement, you can refer to the two Issues of the official JIRA:

  - [TS-3313 New World order for connection management and timeouts](https://issues.apache.org/jira/browse/TS-3313)
  - [TS-1405 apply time-wheel scheduler about event system](https://issues.apache.org/jira/browse/TS-1405)

For an introduction to several queues managed by InactivityCop, see the NetHandler chapter.

## definition

```
// This macro is not defined, indicating the use of a new timeout management mechanism: InactivityCop state machine
#ifndef INACTIVITY_TIMEOUT
int update_cop_config(const char *name, RecDataT data_type, RecData data, void *cookie);

// INKqa10496
// One Inactivity cop runs on each thread once every second and
// loops through the list of NetVCs and calls the timeouts
// Each thread runs an InactivityCop state machine, executing once per second
// NetVC in the convenience queue, and callback timeout NetVC state machine
class InactivityCop : public Continuation
{
public:
  // Constructor
  // Note that ProxyMutex is initialized here, and the ProxyMutex of NetHandler is passed in.
  InactivityCop(ProxyMutex *m) : Continuation(m), default_inactivity_timeout(0)
  {
    // There is only one callback function: check_inactivity
    SET_HANDLER(&InactivityCop::check_inactivity);
    REC_ReadConfigInteger(default_inactivity_timeout, "proxy.config.net.default_inactivity_timeout");
    Debug("inactivity_cop", "default inactivity timeout is set to: %d", default_inactivity_timeout);

    // The callback function for the specified configuration item when the configuration file is reloaded
    RecRegisterConfigUpdateCb("proxy.config.net.default_inactivity_timeout", update_cop_config, (void *)this);
  }

  // Inactivity Timeout handler
  // As mentioned earlier, InactivityCop will reference the ProxyMutex of NetHandler, so during the callback of this function, NetHandler is also locked.
  // Because InactivityCop will access some private members and queues of NetHandler
  int
  check_inactivity(int event, Event *e)
  {
    (void)event;
    ink_hrtime now = Thread::get_hrtime();
    NetHandler &nh = *get_NetHandler(this_ethread());

    Debug("inactivity_cop_check", "Checking inactivity on Thread-ID #%d", this_ethread()->id);
    // Copy the list and use pop() to catch any closes caused by callbacks.
    // forl_LL is a macro, similar to foreach, used to traverse the NetHandler open_list queue
    forl_LL(UnixNetVConnection, vc, nh.open_list)
    {
      // If the vc management thread is the current thread, put vc in the cop_list queue (stack)
      // However, is there really a vc that is not part of the current thread management being put into the open_list? ? ?
      if (vc->thread == this_ethread()) {
        nh.cop_list.push(vc);
      }
    }
    // Next traverse the cop_list queue (stack)
    while (UnixNetVConnection *vc = nh.cop_list.pop()) {
      // If we cannot get the lock don't stop just keep cleaning
      // Try to lock vc
      MUTEX_TRY_LOCK(lock, vc->mutex, this_ethread());
      // If the lock fails, skip this vc and continue to process the following vc.
      if (!lock.is_locked()) {
        NET_INCREMENT_DYN_STAT(inactivity_cop_lock_acquire_failure_stat);
        continue;
      }

      // If vc has been closed, call the close method to close vc, and then continue to process the following vc
      if (vc->closed) {
        close_UnixNetVConnection(vc, e->ethread);
        continue;
      }

      // set a default inactivity timeout if one is not set
      // If the Inactivity Timeout function is enabled: default_inactivity_timeout > 0
      // The current vc timeout timer is 0, then it is initialized to the default value.
      if (vc->next_inactivity_timeout_at == 0 && default_inactivity_timeout > 0) {
        Debug("inactivity_cop", "vc: %p inactivity timeout not set, setting a default of %d", vc, default_inactivity_timeout);
        vc->set_inactivity_timeout(HRTIME_SECONDS(default_inactivity_timeout));
        NET_INCREMENT_DYN_STAT(default_inactivity_timeout_stat);
      } else {
        Debug("inactivity_cop_verbose", "vc: %p now: %" PRId64 " timeout at: %" PRId64 " timeout in: %" PRId64, vc, now,
              ink_hrtime_to_sec(vc->next_inactivity_timeout_at), ink_hrtime_to_sec(vc->inactivity_timeout_in));
      }

      // For a vc that has a timer set: vc->next_inactivity_timeout_at > 0
      // and satisfy the Inactivity Timeout: vc->next_inactivity_timeout_at < now
      if (vc->next_inactivity_timeout_at && vc->next_inactivity_timeout_at < now) {
        // If this vc exists in keep_alive_queue, then a global counter is counted.
        if (nh.keep_alive_queue.in(vc)) {
          // only stat if the connection is in keep-alive, there can be other inactivity timeouts
          // 由于在net_activity()中：next_inactivity_timeout_at = get_hrtime() + inactivity_timeout_in
          // So: diff is the time difference from the last time net_activity was executed to the current time
          ink_hrtime diff = (now - (vc->next_inactivity_timeout_at - vc->inactivity_timeout_in)) / HRTIME_SECOND;
          NET_SUM_DYN_STAT(keep_alive_queue_timeout_total_stat, diff);
          NET_INCREMENT_DYN_STAT(keep_alive_queue_timeout_count_stat);
        }
        Debug("inactivity_cop_verbose", "vc: %p now: %" PRId64 " timeout at: %" PRId64 " timeout in: %" PRId64, vc, now,
              vc->next_inactivity_timeout_at, vc->inactivity_timeout_in);
        // For the timeout vc, the mainEvent is called back to handle, and the upper state opportunity receives a callback for the timeout event.
        // The callback to mainEvent here always satisfies the lock requirements for NetHandler, ReadVIO and WriteVIO in mainEvent.
        // So this callback is always done in a synchronous manner
        vc->handleEvent(EVENT_IMMEDIATE, e);
      }
    }

    // Cleanup the active and keep-alive queues periodically
    // After traversing all the current thread management vc, the following two methods are called, after analysis
    nh.manage_active_queue();
    nh.manage_keep_alive_queue();

    return 0;
  }

  // The method provided to update_cop_config to set the value of the default inactivity_timeout
  void
  set_default_timeout(const int x)
  {
    default_inactivity_timeout = x;
  }

private:
  int default_inactivity_timeout; // only used when one is not set for some bad reason
};

// This is the configuration that reads the configuration from the configuration file and updates the default inactivity_timeout of InactivityCop.
int
update_cop_config(const char *name, RecDataT data_type ATS_UNUSED, RecData data, void *cookie)
{
  InactivityCop *cop = static_cast<InactivityCop *>(cookie);
  ink_assert(cop != NULL);

  if (cop != NULL) {
    if (strcmp(name, "proxy.config.net.default_inactivity_timeout") == 0) {
      Debug("inactivity_cop_dynamic", "proxy.config.net.default_inactivity_timeout updated to %" PRId64, data.rec_int);
      cop->set_default_timeout(data.rec_int);
    }
  }

  return REC_ERR_OKAY;
}

#endif
```

## Active Queue & Keep-Alive Queue

Two queues are declared in the NetHandler: active_queue and keep_alive_queue,

- Only used to store NetVC between Client and ATS
- However, NetVC between ATS and Origin Server will not be placed in these two queues, but will be managed by ServerSessionPooll
- These two queues are designed for the HTTP protocol

Obviously these two queues appear in the members of NetHandler, which is not reasonable, and such a design requires further abstraction/optimization.

In the HTTP protocol, a session can contain multiple transactions, and there can be gaps between different transactions for a long time.

- If this session is processing a transaction, the NetVC corresponding to this session is put into active_queue
- If this session is in the gap between two transactions, the NetVC corresponding to this session is put into keep_alive_queue

When the number of available connections to the ATS is insufficient (the maximum number of connections allowed), NetVC in keep_alive_queue is forced off, even NetVC in active_queue, before accepting this new session.

## NetHandler的延伸：(add_to|remote_from)_active_queue

Add NetVC to active_queue or remove NetVC from active_queue

```
bool
NetHandler::add_to_active_queue(UnixNetVConnection *vc)
{
  Debug("net_queue", "NetVC: %p", vc);
  Debug("net_queue", "max_connections_per_thread_in: %d active_queue_size: %d keep_alive_queue_size: %d",
        max_connections_per_thread_in, active_queue_size, keep_alive_queue_size);

  // First clean up active_queue to see if the maximum connection limit is exceeded
  // if active queue is over size then close inactive connections
  if (manage_active_queue() == false) {
    // exceeds the limit and returns directly to failure
    // there is no room left in the queue
    return false;
  }

  if (active_queue.in(vc)) {
    // If it is already in the queue, it is removed from the queue, and after being added, it is placed in the header and can be processed as soon as possible.
    // already in the active queue, move the head
    active_queue.remove(vc);
  } else {
    // If it is not in the queue, try to remove it from active_queue because a NetVC cannot appear in both active_queue and keep_alive_queue
    // in the keep-alive queue or no queue, new to this queue
    remove_from_keep_alive_queue(vc);
    // Because this NetVC was not in keep_alive_queue, the number is incremented by the counter.
    ++active_queue_size;
  }
  // Put NetVC into active_queue
  active_queue.enqueue(vc);

  return true;
}

void
NetHandler::remove_from_active_queue(UnixNetVConnection *vc)
{
  Debug("net_queue", "NetVC: %p", vc);
  if (active_queue.in(vc)) {
    // Remove from the queue if it is already in the queue
    active_queue.remove(vc);
    // and decrement the counter
    --active_queue_size;
  }
}
```

## Extension of NetHandler: manage_active_queue

For the management of active_queue, only the inspection and processing of Inactivity Timeout is seen in the main process of InactivityCop. Then, where is the processing of Active Timeout completed?

The following manage_active_queue does not implement Active Timeout processing. It only actively traverses active_queue to turn off Inactivity Timeout and Active Timeout NetVC when active_queue reaches the upper limit.

By reading the code, I feel that in the implementation of InactivityCop, Active Timeout seems to be broken, can not be used.

```
bool
NetHandler::manage_active_queue()
{
  // here is used to limit the maximum number of connections
  const int total_connections_in = active_queue_size + keep_alive_queue_size;
  Debug("net_queue", "max_connections_per_thread_in: %d max_connections_active_per_thread_in: %d total_connections_in: %d "
                     "active_queue_size: %d keep_alive_queue_size: %d",
        max_connections_per_thread_in, max_connections_active_per_thread_in, total_connections_in, active_queue_size,
        keep_alive_queue_size);

  // does not process if the maximum number of connections is not reached
  if (max_connections_active_per_thread_in > active_queue_size) {
    return true;
  }

  // When the maximum number of connections is reached, the connection is actively recycled.
  // Get the current time
  ink_hrtime now = Thread::get_hrtime();

  // loop over the non-active connections and try to close them
  UnixNetVConnection *vc = active_queue.head;
  UnixNetVConnection *vc_next = NULL;
  int closed = 0;
  int handle_event = 0;
  int total_idle_time = 0;
  int total_idle_count = 0;
  // Traverse active_queue, actively close the timeout connection
  for (; vc != NULL; vc = vc_next) {
    vc_next = vc->active_queue_link.next;
    if ((vc->next_inactivity_timeout_at <= now) || (vc->next_activity_timeout_at <= now)) {
      _close_vc(vc, now, handle_event, closed, total_idle_time, total_idle_count);
    }
    // Restore to the maximum connection, then the subsequent vc is not processed
    if (max_connections_active_per_thread_in > active_queue_size) {
      return true;
    }
  }

  // Returns false means that you can no longer accept the new vc, reached the maximum value of the connection
  // In the ATS code, only add_to_active_queue() will evaluate this value.
  return false; // failed to make room in the queue, all connections are active
}
```

## Extension of NetHandler: (add_to|remove_from)_keep_alive_queue

Add NetVC to keep_alive_queue, or remove NetVC from keep_alive_queue

```
void
NetHandler::add_to_keep_alive_queue(UnixNetVConnection *vc)
{
  Debug("net_queue", "NetVC: %p", vc);

  if (keep_alive_queue.in(vc)) {
    // If it is already in the queue, it is removed from the queue, and after being added, it is placed in the header and can be processed as soon as possible.
    // already in the keep-alive queue, move the head
    keep_alive_queue.remove(vc);
  } else {
    // If it is not in the queue, try to remove it from active_queue because a NetVC cannot appear in both active_queue and keep_alive_queue
    // in the active queue or no queue, new to this queue
    remove_from_active_queue(vc);
    // Because this NetVC was not in keep_alive_queue, the number is incremented by the counter.
    ++keep_alive_queue_size;
  }
  // Put NetVC into keep_alive_queue
  keep_alive_queue.enqueue(vc);

  // Organize keep_alive_queue
  // if keep-alive queue is over size then close connections
  manage_keep_alive_queue();
}

void
NetHandler::remove_from_keep_alive_queue(UnixNetVConnection *vc)
{
  Debug("net_queue", "NetVC: %p", vc);
  if (keep_alive_queue.in(vc)) {
    // Remove from the queue if it is already in the queue
    keep_alive_queue.remove(vc);
    // and decrement the counter
    --keep_alive_queue_size;
  }
}
```

## Extension of NetHandler: manage_keep_alive_queue

In the implementation of Http/1.1, there is a feature called Keep Alive that has the following behavior:

  - After processing a request, the connection is not closed immediately and becomes IDLE state.
  - Connections in IDLE state have independent timeout settings

This has new requirements for timeout control.

ATS puts this type of vc into keep_alive_queue for processing. The following is the way to manage keep_alive_queue.

```
void
NetHandler::manage_keep_alive_queue()
{
  // here is used to limit the maximum number of connections
  uint32_t total_connections_in = active_queue_size + keep_alive_queue_size;
  // Get the current time
  ink_hrtime now = Thread::get_hrtime();

  Debug("net_queue", "max_connections_per_thread_in: %d total_connections_in: %d active_queue_size: %d keep_alive_queue_size: %d",
        max_connections_per_thread_in, total_connections_in, active_queue_size, keep_alive_queue_size);

  // No processing is done if the maximum number of connections is not reached. This is different from the decision in the active_queue part.
  // Use the sum of active_queue_size + keep_alive_queue_size to make a judgment because:
  // keep_alive_queue is an optional part just to reduce the cost of TCP rebuilding connections
  // So: It actually borrows the connection quota of active_queue. When active_queue needs it, you can't continue to increase keep_alive_queue.
  if (!max_connections_per_thread_in || total_connections_in <= max_connections_per_thread_in) {
    return;
  }

  // When the maximum number of connections is reached, the connection is actively recycled.
  // loop over the non-active connections and try to close them
  UnixNetVConnection *vc_next = NULL;
  int closed = 0;
  int handle_event = 0;
  int total_idle_time = 0;
  int total_idle_count = 0;
  // traverse keep_alive_queue to actively close some of the connections
  // Because it is a keep_alive connection, after closing, it only affects a bit of performance.
  // If active_queue runs out of all connections, then keep_alive_queue will be empty
  for (UnixNetVConnection *vc = keep_alive_queue.head; vc != NULL; vc = vc_next) {
    vc_next = vc->keep_alive_queue_link.next;
    _close_vc(vc, now, handle_event, closed, total_idle_time, total_idle_count);

    // Restore to the maximum connection, then the subsequent vc is not processed
    total_connections_in = active_queue_size + keep_alive_queue_size;
    if (total_connections_in <= max_connections_per_thread_in) {
      break;
    }
  }

  if (total_idle_count > 0) {
    Debug("net_queue", "max cons: %d active: %d idle: %d already closed: %d, close event: %d mean idle: %d\n",
          max_connections_per_thread_in, total_connections_in, keep_alive_queue_size, closed, handle_event,
          total_idle_time / total_idle_count);
  }
  // Since keep_alive_queue is an optional connection pool, there is no failure, so no value is returned.
}
```

## NetHandler extension: _close_vc

In the manage_active_queue() and manage_keep_alive_queue() methods, the _close_vc method is called to actively close some vcs.

In the main callback function of InactivityCop, when the cop_list queue is traversed, when the timeout condition is found, the execution strategy is similar to this method, but it is a little different.

Incoming parameters:

  - vc VC to be shut down
  - now is used to calculate the timeout value, indicating the value of the current time

Returns the value of multiple counters:

  - handle_event This time the value is incremented when the vc is timed out and the vc operation is closed.
  - closed This value is incremented when this process encounters vc that was originally set to the off state.
  - total_idle_time accumulates the time of this vc idle state to this counter, in seconds
  - total_idle_count This value is incremented when this process encounters the vc idle state for more than 1 second.

```
void
NetHandler::_close_vc(UnixNetVConnection *vc, ink_hrtime now, int &handle_event, int &closed, int &total_idle_time,
                      int &total_idle_count)
{
  if (vc->thread != this_ethread()) {
    return;
  }
  // Try to lock vc
  MUTEX_TRY_LOCK(lock, vc->mutex, this_ethread());
  // return if the lock fails
  if (!lock.is_locked()) {
    return;
  }
  // Calculate the time difference from the last time net_activity was executed to the current time: diff value, in seconds
  ink_hrtime diff = (now - (vc->next_inactivity_timeout_at - vc->inactivity_timeout_in)) / HRTIME_SECOND;
  if (diff > 0) {
    total_idle_time += diff;
    ++total_idle_count;
    NET_SUM_DYN_STAT(keep_alive_queue_timeout_total_stat, diff);
    NET_INCREMENT_DYN_STAT(keep_alive_queue_timeout_count_stat);
  }
  Debug("net_queue", "closing connection NetVC=%p idle: %u now: %" PRId64 " at: %" PRId64 " in: %" PRId64 " diff: %" PRId64, vc,
        keep_alive_queue_size, ink_hrtime_to_sec(now), ink_hrtime_to_sec(vc->next_inactivity_timeout_at),
        ink_hrtime_to_sec(vc->inactivity_timeout_in), diff);

  // If vc has been closed, call the close method to close vc
  if (vc->closed) {
    close_UnixNetVConnection(vc, this_ethread());
    ++closed;
  } else {
    // If vc is not closed, set to the timeout state, and then callback vc state machine according to the timeout vc processing
    vc->next_inactivity_timeout_at = now;
    // create a dummy event
    Event event;
    event.ethread = this_ethread();
    // The callback to mainEvent here always satisfies the lock requirements for NetHandler, ReadVIO and WriteVIO in mainEvent.
    // So this callback is always done synchronously, so using the temporary variable event is safe
    vc->handleEvent(EVENT_IMMEDIATE, &event);
    ++handle_event;
  }
}
```

## References

![How the InactivityCop works](https://cdn.rawgit.com/oknet/atsinternals/master/CH02-IOCoreNET/CH02-IOCoreNet-002.svg)

- [UnixNet.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNet.cc)
