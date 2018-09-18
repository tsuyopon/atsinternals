# Core part: Throttle

Throttle is not a class, but several global variables, statistical values, and a set of functions.

Throttle is implemented by calling these related functions in the following methods.

  - NetAccept::do_blocking_accept
  - NetAccept::acceptFastEvent
  - UnixNetVConnection::connectUp

ATS's Throttle design is divided into:

  - Net Throttle, this includes:
    - ACCEPT direction
    - CONNECT direction
  - Emergency Throttle, which includes:
    - Maximum file handle control section
    - Keep the file handle for the part of the backdoor

## definition

Because Throttle's implementation is not a class, it is spread across multiple source code files. In the following introduction, the order is adjusted for convenience.

### Constants and variables

The first is a few constant macro definitions and enumeration values:

```
source: P_UnixNet.h
// When the warning message is output, it needs to be separated by 24 hours or more to avoid sending a large number of warning messages repeatedly.
#define TRANSIENT_ACCEPT_ERROR_MESSAGE_EVERY HRTIME_HOURS(24)
// When the warning message is output, it needs to be separated by 10 seconds or more to avoid sending a large number of warning messages repeatedly.
#define NET_THROTTLE_MESSAGE_EVERY HRTIME_MINUTES(10)

// The number of file handles reserved is reserved for the Cache subsystem.
#define THROTTLE_FD_HEADROOM (128 + 64) // CACHE_DB_FDS + 64

// Why is ACCEPT reserved for 10%? ? ?
#define NET_THROTTLE_ACCEPT_HEADROOM 1.1  // 10%
#define NET_THROTTLE_CONNECT_HEADROOM 1.0 // 0%

// When the ACCEPT limit value occurs, the next 5 new connections are continuously discarded, and the connection is sent to the limit value.
// This only takes effect when the function "Send connection to the client reaches the limit value" is enabled. By default, this function is disabled.
// also the 'throttle connect headroom'
#define THROTTLE_AT_ONCE 5

// remaining file descriptor limit value
#define EMERGENCY_THROTTLE 16
// Remaining file descriptor emergency limit value
#define HYPER_EMERGENCY_THROTTLE 6

// Define the type of Net Throttle: ACCEPT and CONNECT
enum ThrottleType {
  ACCEPT,
  CONNECT,
};
```

Then there are several related global variables that are interpreted according to the order of initialization:

```
source: UnixNet.cc
/ / The operating system limits the number of file open (ATS hard limit on the total number of file open)
// its value is affected by the configuration item: proxy.config.system.file_max_pct, the default value is 90%
// The maximum number of file openes allowed for the system * file_max_pct
// First, call Main.cc:1535 init_system() to complete the initialization:
//     fds_limit = ink_max_out_rlimit(RLIMIT_NOFILE, true, false);
// Then call Main.cc:1538 adjust_sys_settings():
// First, adjust according to proxy.config.system.file_max_pct
// Then, re-adjust according to proxy.config.net.connections_throttle
// If connections_throttle exceeds the fds_limit - THROTTLE_FD_HEADROOM value:
/ / Try to reset the operating system's file open limit value is connections_throttle, and update fds_limit 
// Finally call Main.cc:1664 check_fd_limit():
// If fds_limit - THROTTLE_FD_HEADROOM is less than 1, it will hold the error and stop the ATS process
// In short, fds_limit is the maximum number of file handles that ATS can open.
int fds_limit = 8000;

// The maximum number of Socket handles currently allowed (excluding backdoor, cache file handles, etc.)
// its value comes from the configuration item: proxy.config.net.connections_throttle
// First, call Main.cc:1711 ink_net_init(), which calls Net.cc:43 configure_net()
// read the value from the configuration file,
// And register the handler change_net_connections_throttle() to reset this value when this configuration item changes.
// Then call Main.cc:1771 netProcessor.start(), which calls UnixNetProcessor.cc:405 change_net_connections_throttle()
// The implementation of the change_net_connections_throttle() function may have bugs.
// it ignores the value from the configuration file
// and directly calculate this value via fds_limit, so proxy.config.net.connections_throttle cannot be hot updated
int fds_throttle;

// The maximum number of connections currently allowed, whose value is affected by the configuration item: proxy.config.net.connections_throttle
// When the configuration item is -1, the default value is set using the value of fds_limit - THROTTLE_FD_HEADROOM
// When the configuration item >= 0, it will be set with the value of the configuration item, but its value still cannot exceed the value of fds_throttle
int net_connections_throttle;

// used to record the time of the last warning message output, used to limit the frequency of information output
ink_hrtime last_throttle_warning;
ink_hrtime last_shedding_warning;
ink_hrtime emergency_throttle_time;
ink_hrtime last_transient_accept_error;
```

### Internal method

```
source: P_UnixNet.h
// PRIVATE: This is a Throttle internal method
// Calculate the number of connections that are currently open based on the statistics.
// But for Net Throttle types, there will be a reserved value (penalty value):
// The return value for ACCEPT will be 10% more than the actual number of open connections
// Here I understand as an insurance measure:
// As long as you control the number of connections coming from ACCEPT, there will not be too many CONNECT connections.
// This will give priority to the ACCEPT action when the connection is exhausted (10% margin).
// Since ATS is a PROXY and there are no incoming connections, there will be almost no CONNECT connection.
// For CONNECT return value is the number of connections actually opened
TS_INLINE int
net_connections_to_throttle(ThrottleType t)
{
  double headroom = t == ACCEPT ? NET_THROTTLE_ACCEPT_HEADROOM : NET_THROTTLE_CONNECT_HEADROOM;
  int64_t cool = 0;

  NET_READ_GLOBAL_DYN_SUM(net_connections_currently_open_stat, sval);
  int currently_open = (int)sval;
  // deal with race if we got to multiple net threads
  if (currently_open < 0)
    currently_open = 0;
  return (int)(currently_open * headroom);
}

// PRIVATE: This is a Throttle internal method
// Currently only called by check_net_throttle()
/ / is used to determine whether the current time is limiting the connection
// return value:
// true: currently restricting new connections
// false: no new connections are currently restricted
TS_INLINE bool
emergency_throttle(ink_hrtime now)
{
  return (bool)(emergency_throttle_time > now);
}
```

### external method

```
source: P_UnixNet.h
// PUBLIC: This is the Interface provided to IOCore Net Subsystem
/ / used to detect the current Throttle state
// First determine if the network connection that has been opened exceeds the limit (weighted according to ACCEPT/CONNECT type)
// Also check if it is currently in the Throttle state.
// return value:
// true: Throttle status has been reached
// false: Throttle status not reached
TS_INLINE bool
check_net_throttle(ThrottleType t, ink_hrtime now)
{
  int connections = net_connections_to_throttle(t);

  if (connections >= net_connections_throttle)
    return true;

  if (emergency_throttle(now))
    return true;

  return false;
}

//
// Emergency throttle when we are close to exhausting file descriptors.
// Block all accepts or connects for N seconds where N
// is the amount into the emergency fd stack squared
// (e.g. on the last file descriptor we have 14 * 14 = 196 seconds
// of emergency throttle).
//
// Hyper Emergency throttle when we are very close to exhausting file
// descriptors.  Close the connection immediately, the upper levels
// will recover.
//
// PUBLIC: This is the Interface provided to IOCore Net Subsystem
/ / Used to detect if the incoming file handle is close to the exhausted value
/ / In the ATS mainly by the consumption of two types of file handles:
// Network connection (including the connection between Client and ATS and the connection between ATS and Origin Server)
// disk access (this includes 128 file handles for CACHE_DB_FDS and 64 reserved file handles)
// The restrictions on file handles are also divided into two levels:
// The first level, the number of available file handles is less than 16, for EMERGENCY_THROTTLE
// When this level is triggered, emergency_throttle_time is set so that the entire ATS will pause creating a new connection for a while.
// The second level, the number of available file handles is less than 6, which is HYPER_EMERGENCY_THROTTLE
/ / on the basis of the implementation of the first level of action, will also directly close the file handle
// This function uses the file handle to be a feature of a word-growth variable of type int to implement the decision on the remaining available file handles.
// return value:
// true: the file handle exhaustion limit is reached (the file handle may be closed)
// false: no limit
// Pause time:
// First, calculate the value of the remaining available file handles below the EMERGENCY_THROTTLE(16) value and save it to the over variable.
// For example, the currently available file handle is 10, so the over value is 16 - 10 = 6
// The pause time is over * over = 6 * 6 = 36 seconds
// Add the current time to 36 seconds and save to emergency_throttle_time. No new connections will be created until this time is reached.
TS_INLINE bool
check_emergency_throttle(Connection &con)
{
  int fd        = con.fd;
  int emergency = fds_limit - EMERGENCY_THROTTLE;
  if (fd > emergency) {
    int over                = fd - emergency;
    emergency_throttle_time = Thread::get_hrtime() + (over * over) * HRTIME_SECOND;
    RecSignalWarning(REC_SIGNAL_SYSTEM_ERROR, "too many open file descriptors, emergency throttling");
    int hyper_emergency = fds_limit - HYPER_EMERGENCY_THROTTLE;
    if (fd > hyper_emergency)
      con.close();
    return true;
  }
  return false;
}
```

### Loading configuration items

```
// PUBLIC: This is the Interface provided to IOCore Net Subsystem
/ / used to update the net_connections_throttle value
// Reset the value of net_connections_throttle after the configuration file is updated
// But there seems to be a bug here. The value passed in is not accepted here, but is calculated based on the fds_limit value.
TS_INLINE int
change_net_connections_throttle(const char *token, RecDataT data_type, RecData value, void *data)
{
  (void)token;
  (void)data_type;
  (void)value;
  (void)data;
  int throttle = fds_limit - THROTTLE_FD_HEADROOM;
  if (fds_throttle < 0) // ??? bug ??? fds_throttle should be @value ?
    net_connections_throttle = throttle;
  else {
    net_connections_throttle = fds_throttle;  // ??? bug ??? fds_throttle should be @value ?
    if (net_connections_throttle > throttle)
      net_connections_throttle = throttle;
  }
  return 0;
}
```

### Error determination and warning message output

```
source: P_UnixNet.h
// PUBLIC: This is the Interface provided to IOCore Net Subsystem
// Determine the error status of accept()
// Parameters:
// res takes the errno value when the return value of accept() is less than 0.
// return value:
// 1: Continue to try again
// 0: Output warning, you can try again
// -1: Serious error
// 1  - transient
// 0  - report as warning
// -1 - fatal
TS_INLINE int
accept_error_seriousness(int res)
{
  switch (res) {
  case -EAGAIN:
  case -ECONNABORTED:
  case -ECONNRESET: // for Linux
  case -EPIPE:      // also for Linux
    return 1;
  case -EMFILE:
  case -ENOMEM:
#if defined(ENOSR) && !defined(freebsd)
  case -ENOSR:
#endif
    ink_assert(!"throttling misconfigured: set too high");
#ifdef ENOBUFS
  case -ENOBUFS:
#endif
#ifdef ENFILE
  case -ENFILE:
#endif
    return 0;
  case-EINTR:
    ink_assert(!"should be handled at a lower level");
    return 0;
#if defined(EPROTO) && !defined(freebsd)
  case -EPROTO:
#endif
  case -EOPNOTSUPP:
  case -ENOTSOCK:
  case -ENODEV:
  case -EBADF:
  default:
    return -1;
  }
}

// PUBLIC: This is the Interface provided to IOCore Net Subsystem
// Error message encountered when output accept() is output, and limit the output frequency of error message (24 hours)
TS_INLINE void
check_transient_accept_error(int res)
{
  ink_hrtime t = Thread::get_hrtime();
  if (!last_transient_accept_error || t - last_transient_accept_error > TRANSIENT_ACCEPT_ERROR_MESSAGE_EVERY) {
    last_transient_accept_error = t;
    Warning("accept thread received transient error: errno = %d", -res);
#if defined(linux)
    if (res == -ENOBUFS || res == -ENFILE)
      Warning("errno : %d consider a memory upgrade", -res);
#endif
  }
}

// PUBLIC: This is the Interface provided to IOCore Net Subsystem
/ / Avoid output a large number of repeated "connection has reached the overflow limit value" warning message
// * This method is not used anywhere
/ / Limit the output of the "number of connections reaching shedding limit" warning message interval of 10 seconds and above
TS_INLINE void
check_shedding_warning()
{
  ink_hrtime t = Thread::get_hrtime();
  if (t - last_shedding_warning > NET_THROTTLE_MESSAGE_EVERY) {
    last_shedding_warning = t;
    RecSignalWarning(REC_SIGNAL_SYSTEM_ERROR, "number of connections reaching shedding limit");
  }
}

// PUBLIC: This is the Interface provided to IOCore Net Subsystem
/ / Avoid output a large number of repeated "connection has reached the limit value" warning message
/ / Limit the output of "too many connections, throttling" warning message interval of 10 seconds and above
TS_INLINE void
check_throttle_warning()
{
  ink_hrtime t = Thread::get_hrtime();
  if (t - last_throttle_warning > NET_THROTTLE_MESSAGE_EVERY) {
    last_throttle_warning = t;
    RecSignalWarning(REC_SIGNAL_SYSTEM_ERROR, "too many connections, throttling");
  }
}
```

### Output warning message to the client

```
source: UnixNetAccept.cc
// PUBLIC: This is the Interface provided to IOCore Net Subsystem
// When the throttle_error_message function is enabled (the code to enable this feature is gone), when the Throttle state is encountered:
// ATS will continue accept() up to THROTTLE_AT_ONCE connections
// Proactively send information within the throttle_error_message to these connections
// Finally close these connections
/ / Block the accept operation proxy.config.net.throttle_delay milliseconds at the same time, the default value is: 50 milliseconds
// If not enabled (current default)
// will not call this method, but simply block the accept operation
//
// Send the throttling message to up to THROTTLE_AT_ONCE connections,
// delaying to let some of the current connections complete
//
static int 
send_throttle_message(NetAccept *na)
{
  struct pollfd afd;
  Connection con[100];
  char dummy_read_request[4096];

  afd.fd     = na->server.fd;
  afd.events = POLLIN;

  int n = 0;
  while (check_net_throttle(ACCEPT, Thread::get_hrtime()) && n < THROTTLE_AT_ONCE - 1 && (socketManager.poll(&afd, 1, 0) > 0)) {
    int res = 0;
    if ((res = na->server.accept(&con[n])) < 0)
      return res;
    n ++;
  }
  safe_delay(net_throttle_delay / 2); 
  int i = 0;
  for (i = 0; i < n; i++) {
    socketManager.read(con[i].fd, dummy_read_request, 4096);
    socketManager.write(con[i].fd, unix_netProcessor.throttle_error_message, strlen(unix_netProcessor.throttle_error_message));
  }
  safe_delay(net_throttle_delay / 2); 
  for (i = 0; i < n; i++)
    con[i].close();
  return 0;
}
```

## Throttle's implementation

Since a fixed number of file handles are always used in the cache subsystem, ATS reserves 64 file handles for other systems (such as logs) (the use of these 64 reserved file handles needs to be reconfirmed), which may be used Only a network subsystem is available to a large number of file handles.

Throttle is implemented in two parts in the network subsystem:

  - NetAccept judges Throttle after accepting a new connection
  - connectUp judges Throttle before creating a file descriptor for connect()

This part of the implementation, there is a bug on the 6.0.x branch: TS-4879, because check_emergency_throttle() will close the file handle directly in the limit case, but this condition is not handled in the code, and continue to use the file that has been closed The handle creates NetVC, which causes the NetVC to be managed only by the default timeout control, causing NetVC to not shut down for a longer period of time.

The following uses the repaired code for analysis:

## Implementation in NetAccept

NetAccept has several modes of operation. First, it analyzes the operation of blocking accept in Dedicated EThread:

```
int
NetAccept::do_blocking_accept(EThread *t) 
{
  int res = 0;
  int loop               = accept_till_done;
  UnixNetVConnection *vc = NULL;
  Connection con;

  // do-while for accepting all the connections
  // added by YTS Team, yamsat
  do {
    ink_hrtime now = Thread::get_hrtime();

    // Throttle accepts
    // There are no restrictions on the backdoor service.
    // Since backdoor is used to implement the heartbeat between traffic_server and traffic_manager, the availability of its services must be guaranteed.
    // Then determine if the ACCEPT direction has reached the connection limit or is in the state of the file descriptor limit
    while (!backdoor && check_net_throttle(ACCEPT, now)) {
      // has reached the limit
      // output warning message
      check_throttle_warning();
      // throttle_error_message is not set (default)
      if (!unix_netProcessor.throttle_error_message) {
        // blocking net_throttle_delay milliseconds
        // Since do_blocking_accept() runs in Dedicated EThread, it does not block
        safe_delay(net_throttle_delay);
      } else if (send_throttle_message(this) < 0) {
        // Otherwise, send throttle_error_message
        goto Lerror;
      }
      // update the now value for check_net_throttle
      now = Thread::get_hrtime();
    }   

    if ((res = server.accept(&con)) < 0) {
    Learner:
      // Limit the output frequency of error messages when accepting a new connection failure
      int seriousness = accept_error_seriousness(res);
      // encountered a serious error, such as EAGAIN, etc.
      if (seriousness >= 0) { // not so bad
        // If you need to send a warning message
        if (!seriousness)     // bad enough to warn about
          check_transient_accept_error(res);
        // Block net_throttle_delay milliseconds to save CPU usage
        safe_delay(net_throttle_delay);
        return 0;
      }   
      if (!action_->cancelled) {
        SCOPED_MUTEX_LOCK(lock, action_->mutex, t);
        action_->continuation->handleEvent(EVENT_ERROR, (void *)(intptr_t)res);
        Warning("accept thread received fatal error: errno = %d", errno);
      }
      return -1;
    }

    / / Because the blocking method accept () connection limit value judged before accept () may have been a long time ago
    // So when we get a reasonable connection we still need to determine if it exceeds the file descriptor limit based on the value of FD
    // The con.fd may exceed the limitation of check_net_throttle() because we do blocking accept here.
    if (check_emergency_throttle(con)) {
      // If the limit is exceeded, you also need to check if the FD has been closed.
      // The `con' could be closed if there is hyper emergency
      if (con.fd == NO_FD) {
        return 0;
      }
    }

    // Use 'NULL' to Bypass thread allocator
    vc = (UnixNetVConnection *)this->getNetProcessor()->allocate_vc(NULL);
...
  } while (loop);

  return 1;
}
```

Then analyze the non-blocking accept mode in Regular EThread:

```
// Through the periodic Event, acceptFastEvent is periodically called back
int   
NetAccept::acceptFastEvent(int event, void *ep)
{   
  Event *e = (Event *)ep;
  (void)event;
  (void)e;
  int bufsz, res = 0;
  Connection con;
        
  PollDescriptor *pd     = get_PollDescriptor(e->ethread);
  UnixNetVConnection *vc = NULL;
  int loop               = accept_till_done;
    
  do {
    // There are no restrictions on the backdoor service.
    // Since backdoor is used to implement the heartbeat between traffic_server and traffic_manager, the availability of its services must be guaranteed.
    // Then determine if the ACCEPT direction has reached the connection limit
    // Note that the determination of the file descriptor limit status may not be accurate enough here.
    if (!backdoor && check_net_throttle(ACCEPT, Thread::get_hrtime())) {
      // ifd this member seems useless
      ifd = NO_FD;
      // Return directly to EventSystem when the connection is restricted
      return EVENT_CONT;
    }

    socklen_t sz = sizeof(con.addr);
    int fd       = socketManager.accept4(server.fd, &con.addr.sa, &sz, SOCK_NONBLOCK | SOCK_CLOEXEC);
    con.fd       = fd;
...
    // check return value from accept()
    if (res < 0) {
      res = -errno;
      if (res == -EAGAIN || res == -ECONNABORTED
#if defined(linux)
          || nothing == -EPIPE
#endif
          ) {
        goto Ldone;
      } else if (accept_error_seriousness(res) >= 0) {
        // Limit the output frequency of error messages when accepting a new connection failure
        check_transient_accept_error(res);
        // There is no blocking here. After jumping to Ldone, return directly to EventSystem and wait for the next callback.
        // Each time you are called back, you will first check if it is in the connection-limited state.
        goto Ldone;
      }
      if (!action_->cancelled)
        action_->continuation->handleEvent(EVENT_ERROR, (void *)(intptr_t)res);
      goto Lerror;
    }
    // Before the VC is created, the status FD is not judged again by check_emergency_throttle.
    // This is because:
    // We have a very short time between the decision to connect to the accept() call in the non-blocking accept() call, there is no great need
    // Although in a multi-threaded environment, all NetAccepts that make an accept() call to the same port share the same mutex
    // Every time a new connection is accepted, the connection limit is checked
    // But there are also hidden dangers: there will be other ports that make accept() calls at the same time.
    // Since check_emergency_throttle is not called:
    // After accepting the new connection, the FD is not checked and the emergency_throttle_time cannot be updated.
    // Therefore, the update of emergency_throttle_time is only updated in connectUp
    // Since check_net_throttle has a 10% reservation for the ACCEPT direction, therefore:
    // When accept() returns an FD, there is a 10% margin from the limit of the file descriptor.
    vc = (UnixNetVConnection *)this->getNetProcessor()->allocate_vc(e->ethread);
...
  } while (loop);

Ldone:
  return EVENT_CONT;

Learner:
  server.close();
  e->cancel();
  if (vc)
    vc->free(e->ethread);
  NET_DECREMENT_DYN_STAT(net_accepts_currently_open_stat);
  delete this;
  return EVENT_DONE;
}
```

### Implementation in connectUp

```
int
UnixNetVConnection::connectUp(EThread *t, int fd)
{
  int res; 

  thread = t;
  / / First determine whether the CONNECT direction has reached the connection limit, or in the state of the file descriptor limit
  if (check_net_throttle(CONNECT, submit_time)) {
    // has reached the limit
    // output warning message
    check_throttle_warning();
    // Return the state machine NET_EVENT_OPEN_FAILED event
    action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)-ENET_THROTTLING);
    free(t);
    return CONNECT_FAILURE;
  }
...
  // Next, determine if the file descriptor is already open.
  // If it is a request initiated from the TS API, it should pass in an already opened file descriptor
  // If this is getting called from the TS API, then we are wiring up a file descriptor
  // provided by the caller. In that case, we know that the socket is already connected.
  if (fd == NO_FD) {
    // If not open, create one now
    // Because it is a multi-threaded environment, even after judging it, after opening a file descriptor, there may still be problems beyond the limit.
    // Due to multi-threads system, the fd returned from con.open() may exceed the limitation of check_net_throttle().
    res = con.open(options);
    if (res != 0) {
      goto fail;
    }
  } else {
    int len = sizeof(con.sock_type);

    // This call will fail if fd is not a socket (e.g. it is a
    // eventfd or a regular file fd.  That is ok, because sock_type
    // is only used when setting up the socket.
    safe_getsockopt(fd, SOL_SOCKET, SO_TYPE, (char *)&con.sock_type, &len);
    safe_nonblocking(fd);
    con.fd           = fd;
    con.is_connected = true;
    con.is_bound     = true;
  }

  // Therefore, you need to call check_emergency_throttle again to verify that the file descriptor is out of bounds
  if (check_emergency_throttle(con)) {
    // If true is returned, the limit value is exceeded and the penalty time is set, at which point accept will be suspended for a while.
    // We need to further determine if the FD is closed
    // The `con' could be closed if there is hyper emergency
    if (con.fd == NO_FD) {
      // We need to decrement the stat because close_UnixNetVConnection only decrements with a valid connection descriptor.
      NET_SUM_GLOBAL_DYN_STAT(net_connections_currently_open_stat, -1);
      // Set errno force to EMFILE (reached limit for open file descriptors)
      errno = EMFILE;
      res   = -errno;
      goto fail;
    }
    // If the limit has been reached, but the connection is not closed, you can still continue to use the connection
  }

  // Must connect after EventIO::Start() to avoid a race condition
  // when edge triggering is used.
  if (ep.start(get_PollDescriptor(t), this, EVENTIO_READ | EVENTIO_WRITE) < 0) {
    res = -errno;
    Debug("iocore_net", "connectUp : Failed to add to epoll list : %s", strerror(errno));
    goto fail;
  }

  if (fd == NO_FD) {
    res = con.connect(NULL, options);
    if (res != 0) {
      goto fail;
    }
  }
...
}
```

You can see that there is no safe_delay() operation in the CONNECT direction because:

  - safe_delay() is a blocking operation
  - When the state machine encounters NET_EVENT_OPEN_FAILED, the reschedule operation should be performed when the error code is -ENET_THROTTLING

### THROTTLE_FD_HEADROOM 限定

This is the file descriptor that ATS retains for non-network communication. The comments explain:

  - 128 from CACHE_DB_FDS
  - 64 No explanation, here I think it is Other

I looked up the definition of CACHE_DB_FDS:

```
cache/I_CacheDefs.h:#define CACHE_DB_FDS 128
```

But I haven't found any place to use this macro after discussing it with other developers in the community:

  - 8 file descriptors per hard disk
    - Proxy.config.cache.threads_per_disk INT 8 in the default configuration file records.config released with the source code
    - But the default is 12 in the source code. If the configuration item is not defined in records.config, it is 12
  - Early hardware environment was SCSI hard disk
    - Up to 16 disks can be connected per SCSI channel

Therefore, the number of file descriptors for CACHE_DB is: 8 * 16 = 128.

For the purpose of the other 64 file descriptors, it is still unclear. The guess may be related to the DNS and the log system.

## References

- [P_UnixNet.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
- [UnixNet.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNet.cc)
- [Net.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/Net.cc)
- [Main.cc](http://github.com/apache/trafficserver/tree/master/proxy/Main.cc)
- [UnixNetAccept.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetAccept.cc)
- [UnixNetVConnection.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetVConnection.cc)
