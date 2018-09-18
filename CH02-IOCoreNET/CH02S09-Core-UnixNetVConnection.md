# core component: UnixNetVConnection

In fact, the IOCoreNet subsystem is the same as the EventSystem. There are also Thread, Processor and Event, but the names are different:

|  EventSystem   |    epoll   |  Polling SubSystem  |        NetSubSystem       |
|: --------------::: ----------::: -------------------- -: |: -------------------------: |
|      Event     |  socket fd |       EventIO       |     UnixNetVConnection    |
|     EThread    | epoll_wait |      PollCont       | NetHandler，InactivityCop |
| EventProcessor |  epoll_ctl |       EventIO       |        NetProcessor       |

- Like Event, UnixNetVConnection also provides a way to the upper state machine
  - do_io_* series
  - (set|cancel)_*_timeout 系列
  - (add|remove)_*_queue series
  - There is also a part of the method for the upper state machine defined in NetProcessor
- UnixNetVConnection also provides methods for the underlying state machine
  - Usually called by NetHandler
  - These methods can be thought of as dedicated callback functions for the NetHandler state machine
  - I personally think that all the functions that deal with the socket should be placed in the NetHandler.
- UnixNetVConnection is also a state machine
  - So it also has its own handler (callback function)
    - NetAccept calls acceptEvent
    - InactivityCop calls mainEvent
    - The constructor is initialized to startEvent, which is used to call connectUp(), which is for NetProcessor.
  - There are roughly three call paths:
    - EThread  －－－  NetAccept  －－－ UnixNetVConnection
    - EThread --- NetHandler --- UnixNetVConnection
    - EThread  －－－  InactivityCop  －－－  UnixNetVConnection

Since it is both an Event and an SM, UnixNetVConnection is more complex than Event.

Since SSLNetVConnection inherits from UnixNetVConnection, some of the content is reserved for supporting SSL in the definition of UnixNetVConnection.

## Definition & Method

The following is a partial adjustment of the order of the lines of code for the convenience of description.

First look at [NetState] (https://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetState.h),

```
struct NetState {
  volatile int enabled;
  VIO saw;
  Link<UnixNetVConnection> ready_link;
  SLink<UnixNetVConnection> enable_link;
  int in_enabled_list;
  int triggered;

  NetState() : enabled(0), vio(VIO::NONE), in_enabled_list(0), triggered(0) {}
};

```

Then UnixNetVConnection,

```
// Calculate the memory offset of the VIO member in the NetState instance
#define STATE_VIO_OFFSET ((uintptr_t) & ((NetState *)0)->vio)
// return a pointer to a NetState instance containing the specified VIO
#define STATE_FROM_VIO(_x) ((NetState *)(((char *)(_x)) - STATE_VIO_OFFSET))

// Turn off the read and write operations of the specified VC, actually modify the specified VC NetState instance read or write enabled member is 0
#define disable_read(_vc) (_vc)->read.enabled = 0
#define disable_write(_vc) (_vc)->write.enabled = 0
// Open the specified VC read and write operations, the actual modification of the specified VC NetState instance read or write enabled member is 1
#define enable_read(_vc) (_vc)->read.enabled = 1
#define enable_write(_vc) (_vc)->write.enabled = 1
```

The above macro definitions are used to manipulate the NetState structure, which will be used below.

```
class UnixNetVConnection : public NetVConnection
{
public:
  // method for the upper state machine
  // Set the read, write, close, half-close operation
  virtual VIO *do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf);
  virtual VIO *do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner = false);
  virtual void do_io_close(int lerrno = -1);
  virtual void do_io_shutdown(ShutdownHowTo_t howto);

  // The method provided specifically for the TS API
  // Called by TSVConnReadVIOGet, TSVConnWriteVIOGet, TSVConnClosedGet, TSTransformOutputVConnGet
  virtual bool get_data(int id, void *data);

  // Send sub-state machine with out-of-band data and corresponding cancel method
  virtual Action *send_OOB(Continuation *cont, char *buf, int len);
  virtual void cancel_OOB();

  // method to support SSL function
  virtual void
  setSSLHandshakeWantsRead(bool /* flag */)
  { return; }
  virtual bool
  getSSLHandshakeWantsRead()
  { return false; }
  virtual void
  setSSLHandshakeWantsWrite(bool /* flag */)
  { return; }
  virtual bool
  getSSLHandshakeWantsWrite()
  { return false; }


  //////////////////////////////////////////////////////////////////////////////////////////////////////////////
  // Set the timeouts associated with this connection.      //
  // active_timeout is for the total elasped time of        //
  // the connection.                                        //
  // inactivity_timeout is the elapsed time from the time   //
  // a read or a write was scheduled during which the       //
  // connection  was unable to sink/provide data.           //
  // calling these functions repeatedly resets the timeout. //
  // These functions are NOT THREAD-SAFE, and may only be   //
  // called when handing an  event from this NetVConnection,//
  // or the NetVConnection creation callback.               //
  //////////////////////////////////////////////////////////////////////////////////////////////////////////////
  // Timeout management for upper state machine
  virtual void set_active_timeout(ink_hrtime timeout_in);
  virtual void set_inactivity_timeout(ink_hrtime timeout_in);
  virtual void cancel_active_timeout();
  virtual void cancel_inactivity_timeout();
  // Connection pool management for upper state machine
  virtual void add_to_keep_alive_queue();
  virtual void remove_from_keep_alive_queue();
  virtual bool add_to_active_queue();
  virtual void remove_from_active_queue();

  // When the upper state machine calls VIO::reenable(), it is called by it and should not be called by the upper state machine.
  // VIO to activate read or write operations
  // In fact, there are two VIOs inside each VC, especially for read and write operations.
  // The public interface is VIO::reenable()
  virtual void reenable(VIO *vio);
  virtual void reenable_re(VIO *vio);

  virtual SOCKET get_socket();

  virtual ~UnixNetVConnection();

  //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  // instances of UnixNetVConnection should be allocated         //
  // only from the free list using UnixNetVConnection::alloc().  //
  // The constructor is public just to avoid compile errors.     //
  //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  // UnixNetVConnection instances can only be allocated via the alloc() method
  // In fact, the distribution is done through ProxyAllocator or global space allocator
  // There is an instance of UnixNetVConnection in the global space allocator, which will run this constructor to complete the initialization.
  // The instance allocated by ProxyAllocator or global space allocator is directly copying the internal space of the internal instance to complete the initialization.
  UnixNetVConnection();

private:
  UnixNetVConnection(const NetVConnection &);
  UnixNetVConnection &operator=(const NetVConnection &);

public:
  //////////////////////////////////////
  // UNIX implementation //
  //////////////////////////////////////
  
  // Set the enabled member of the NetState instance containing the VIO instance to 1
  // and reset the timer of inactivity timeout at the same time
  void set_enabled(VIO *vio);

  // has been abandoned
  // Look at the function name, guess is to get the structure of the local Socket Address
  void get_local_sa();

  // these are not part of the pure virtual interface.  They were
  // added to reduce the amount of duplicate code in classes inherited
  // from NetVConnection (SSL).
  virtual int
  sslStartHandShake(int event, int &err)
  {
    (void)event;
    (void)err;
    return EVENT_ERROR;
  }
  virtual bool
  getSSLHandShakeComplete()
  {
    return (true);
  }
  virtual bool
  getSSLClientConnection()
  {
    return (false);
  }
  virtual void
  setSSLClientConnection(bool state)
  {
    (void)state;
  }
  
  // Read operation, read data from the socket, production to the MIOBuffer in Read VIO, directly call read_from_net () to achieve
  virtual void net_read_io(NetHandler *nh, EThread *lthread);
  // Write operation, called by write_to_net_io, consume data from MIOBufferAccessor in Write VIO, then write to socket, return the number of bytes actually consumed
  virtual int64_t load_buffer_and_write(int64_t towrite, int64_t &wattempted, int64_t &total_written, MIOBufferAccessor &buf,
                                        int &needs);

  // Close the current VC Read VIO, directly call the read_disable () method to achieve
  // Cancel the timeout control, remove from the NetHandler read_ready_list, delete from epoll
  void readDisable(NetHandler *nh);
  // Note that there is no writeDisable in ATS!!!

  // This method is called when the VIO read operation encounters an error, and the upper state opportunity receives an ERROR event.
  // Set vc-> lerrno = err, directly call read_signal_done (VC_EVENT_ERROR) method to achieve
  void readSignalError(NetHandler *nh, int err);
  // Note that there is no writeSignalError method in the UnixNetVConnection class, but there is a write_signal_error() function

  // This method is called when the VIO read operation is completed, and the upper state machine receives a READ_COMPLETE or EOS event.
  // Directly call the read_signal_done (event) method to achieve
  // If the upper state machine does not turn off the VC in the callback,
  // Then the corresponding VC will be removed from the NetHandler's read_ready_list via read_reschedule
  // If the VC is closed, the close_UnixNetVConnection is responsible for removing the VC from the NetHandler's read and write queues.
  int readSignalDone(int event, NetHandler *nh);
  // Note that there is no writeSignalDone method in the UnixNetVConnection class, but there is a write_signal_done() function

  // This method is called when a VIO read is completed, and the upper state machine receives a READ_READY event.
  // Directly call the read_signal_and_update (event) method to achieve
  int readSignalAndUpdate(int event);
  // Note that there is no writeSignalAndUpdate method in the UnixNetVConnection class, but there is a write_signal_and_update() function

  // Reschedule the current VC read operation
  // Directly call the read_reschedule () method to achieve
  // If both read.triggered and read.enabled are true, they will be re-added to the NetHandler's read_ready_list queue.
  // Otherwise, remove the corresponding vc from the NetHandler's read_ready_list queue
  void readReschedule (NetHandler * nh);
  // Reschedule the current VC write operation
  // Directly call the write_reschedule () method to achieve
  // If both write.triggered and write.enabled are true, they will be re-added to the NetHandler's write_ready_list queue.
  // Otherwise, remove the corresponding vc from the NetHandler's write_ready_list queue
  void writeReschedule (NetHandler * nh);
  
  // Update the inactivity timeout timer, this method is called to update the timer after each read and write operation
  // Directly call the net_activity () method to achieve
  void netActivity(EThread *lthread);
  
  / **
   * If the current object's thread does not match the t argument, create a new
   * NetVC in the thread t context based on the socket and ssl information in the
   * current NetVC and mark the current NetVC to be closed.
   * /
  // Migrate VC from the original EThread to the current EThread.
  // This is a new method that was added in October 2015, implemented by the Yahoo team. Reference:
  // Official JIRA: TS-3797
  //     官方Github：https://github.com/apache/trafficserver/commit/c181e7eea93592fa496247118f72e8323846fd5a
  //     官方Wiki：https://cwiki.apache.org/confluence/display/TS/Threading+Issues+And+NetVC+Migration
  UnixNetVConnection *migrateToCurrentThread(Continuation *c, EThread *t);

  // point to the upper state machine
  Action action_;
  // Indicates if the current VC is closed
  volatile int closed;
  // Read and write two states, each containing a VIO instance, corresponding to read and write operations
  NetState read;
  NetState write;

  // Define four types of links, but no instances are defined here. Instances of these links are declared in NetHandler.
  LINKM(UnixNetVConnection, read, ready_link)
  SLINKM(UnixNetVConnection, read, enable_link)
  LINKM(UnixNetVConnection, write, ready_link)
  SLINKM(UnixNetVConnection, write, enable_link)

  // Define the linked list node used inside InactivityCop
  LINK(UnixNetVConnection, cop_link);
  // Define the linked list node used to implement the support of the keepalive connection pool
  LINK(UnixNetVConnection, keep_alive_queue_link);
  // Define the linked list node used to implement the timeout operation
  LINK(UnixNetVConnection, active_queue_link);

  // Define inactivity timeout, I understand IDLE Timeout, is the maximum waiting time when there is no data transmission on this connection
  // Reset the IDLE Timeout timer every time it reads or writes
  ink_hrtime inactivity_timeout_in;
  // Define active timeout, I understand NetVC LifeCycle Timeout, is how long a NetVC can survive
  // Reset can extend the life of NetVC
  ink_hrtime active_timeout_in;
#ifdef INACTIVITY_TIMEOUT
  Event *inactivity_timeout;
  Event *activity_timeout;
#else
  ink_hrtime next_inactivity_timeout_at;
  ink_hrtime next_activity_timeout_at;
#endif

  // descriptor for operating epoll
  // used in reenable, reenable_re, acceptEvent, connectUp
  EventIO ep;
  // point to the NetHandler that manages this connection
  NetHandler * nh;
  // The unique ID of this connection, self-growth, assigned by UnixNetProcessor.cc::net_next_connection_number()
  unsigned int id;
  // the address of the peer side
  // When ATS accepts a client connection, fill in the client's IP address with this variable.
  // When the ATS is connected to a server, fill in the server IP address with this variable.
  // So this is equivalent to the result of get_remote_addr(), don't be stunned by the literal meaning of server_addr.
  // AMC God made a comment here, I think this is a duplicate definition, you can use remote_addr or con.addr instead.
  // I submitted a patch, using get_remote_addr() instead of server_addr
  // amc - what is this for? Why not use remote_addr or con.addr?
  IpEndpoint server_addr; /// Server address and port.

  // can be a flag, or used to mark a semi-closed state
  union {
    unsigned int flags;
#define NET_VC_SHUTDOWN_READ 1
#define NET_VC_SHUTDOWN_WRITE 2
    struct {
      unsigned int got_local_addr : 1;
      unsigned int shutdown : 2;
    } f;
  };

  // The underlying socket description structure corresponding to the current VC instance, which contains the file descriptor
  Connection con;
  
  // Count the reentry of UnixNetVConnection as a state machine
  int recursion;
  // Record the current instance creation time
  ink_hrtime submit_time;
  // callback method pointer for out-of-band data
  OOB_callback * oob_ptr;
  
  // true=The current VC instance is created by NetAccept in DEDICATED EThread mode
  // Indicates that the memory of the VC is allocated by the global space. When released, you need to directly call the global space to release.
  bool from_accept_thread;

  // used to debug and track data transfers
  // es - origin_trace associated connections
  bool origin_trace;
  const sockaddr *origin_trace_addr;
  int origin_trace_port;

  // callback method as a state machine
  // startEvent is usually set by connectUp
  int startEvent(int event, Event *e);
  // acceptEvent is usually set by NetAccept after creating VC
  int acceptEvent(int event, Event *e);
  // mainEvent is usually called by InactivityCop to implement timeout control
  // startEvent and acceptEvent will be transferred to mainEvent after the execution is completed.
  int mainEvent(int event, Event *e);
  
  // As a client initiates a socket connection, the state machine callback function is set to startEvent
  virtual int connectUp(EThread *t, int fd);
  
  / **
   * Populate the current object based on the socket information in in the
   * with parameter.
   * This is logic is invoked when the NetVC object is created in a new thread context
   * /
  // This is a new method that was added in October 2015, implemented by the Yahoo team, see above: migrateToCurrentThread
  virtual int populate(Connection &con, Continuation *c, void *arg);
  
  // Release the VC, will not release the memory, just return the memory to the freelist memory pool or the global memory pool
  // At the same time will release the mutex used by the vc, the mutex used by the upper state machine, read and write the mutex used by VIO
  virtual void free(EThread *t);

  // return member: inactivity_timeout_in
  virtual ink_hrtime get_inactivity_timeout();
  // return member: active_timeout_in
  virtual ink_hrtime get_active_timeout();

  // Set the local address, local_addr.sa
  virtual void set_local_addr();
  // Set the remote address, remote_addr
  virtual void set_remote_addr();
  
  // Set the INIT CWND parameter of TCP
  virtual int set_tcp_init_cwnd(int init_cwnd);
  
  // Set some parameters of TCP
  // Directly call con.apply_options(options)
  virtual void apply_options();

  // Write operation, in order to be compatible with the SSL part, actually call load_buffer_and_write to complete the socket write operation
  friend void write_to_net_io(NetHandler *, UnixNetVConnection *, EThread *);

  void
  setOriginTrace(bool t)
  {
    origin_trace = t;
  }

  void
  setOriginTraceAddr(const sockaddr *addr)
  {
    origin_trace_addr = addr;
  }

  void
  setOriginTracePort(int port)
  {
    origin_trace_port = port;
  }
};

extern ClassAllocator<UnixNetVConnection> netVCAllocator;

typedef int (UnixNetVConnection::*NetVConnHandler)(int, void *);

```

## NetHandler extension: from Socket to MIOBuffer

How does the data read from the Socket to the MIOBuffer?

When we introduced NetAccept, we know that:

  - NetAccept is responsible for accepting a new connection and creating a VC
  - Then pass the EVENT_NET_ACCEPT event to the Acceptor of the upper state machine
  - Create the corresponding upper state machine from the Acceptor of the upper state machine
  - Then the NetHandler will adjust the upper state machine back and forth according to the read and write VIO registered by the upper state machine and the received read and write events.

Therefore, the answer to this question should be found from NetHandler.

Looking back at the analysis of NetHandler::mainNetEvent, we know that:

  - NetHandler discovers whether the underlying socket has data to read via epoll_wait
  - Then put the data-readable vc into read_ready_list
  - then iterate over the read_ready_list
  - For vc with Read VIO active, call vc->net_read_io(this, trigger_event->ethread) to receive data
  - Then call back the upper state machine
  - For the result of the reception: success, failure, whether to complete VIO, pass the different events to the upper state machine, and pass VC or VIO

The net_read_io function is analyzed in detail below:

```
// net_read_io is a wrapper function that actually calls read_from_net()
// In the implementation of ssl, this method will be overridden, calling ssl_read_from_net()
void
UnixNetVConnection::net_read_io(NetHandler *nh, EThread *lthread)
{
  read_from_net(nh, this, lthread);
}

// Read the data for a UnixNetVConnection.
// Rescheduling the UnixNetVConnection by moving the VC
// onto or off of the ready_list.
// Had to wrap this function with net_read_io for SSL.
// This is the way to actually read the socket and put the read data into MIOBuffer
static void
read_from_net(NetHandler *nh, UnixNetVConnection *vc, EThread *thread)
{
  NetState *s = &vc->read;
  ProxyMutex *mutex = thread->mutex;
  int64_t r = 0;

  // Try to get the Mutex lock on this VC's VIO
  MUTEX_TRY_LOCK_FOR(lock, s->vio.mutex, thread, s->vio._cont);
  if (!lock.is_locked()) {
    // If you don't get the lock, reschedule it, wait until the next time you read it again.
    // read_reschedule will put this vc back to the tail of the read_ready_link queue
    // Due to the handling of NetHandler, all read operations will be completed in this EventSystem callback NetHandler.
    // Will not delay until the next EventSystem callback to NetHandler
    read_reschedule(nh, vc);
    return;
  }

  // It is possible that the closed flag got set from HttpSessionManager in the
  // global session pool case.  If so, the closed flag should be stable once we get the
  // s->vio.mutex (the global session pool mutex).
  // After getting the lock, first determine if the vc is closed asynchronously, because HttpSessionManager will manage a global session pool.
  if (vc->closed) {
    close_UnixNetVConnection(vc, thread);
    return;
  }
  // if it is not enabled.
  // Determine if the vc read operation is asynchronously disabled.
  if (!s->enabled || s->vio.op != VIO::READ) {
    read_disable(nh, vc);
    return;
  }

  // The following is the preparation for the read operation.
  // Get the buffer, ready to write data
  MIOBufferAccessor &buf = s->vio.buffer;
  // If the buffer does not exist, it will report an exception and crash
  ink_assert(buf.writer());

  // if there is nothing to do, disable connection
  // Define the total length of the total data that needs to be read in VIO, and the length of the data read that has been completed.
  // ntodo is the remaining data length that needs to be read
  int64_t ntodo = s->vio.ntodo();
  if (ntodo <= 0) {
    // ntodo is 0, indicating that the read operation defined by VIO has been completed before.
    // Therefore directly prohibit this VC read operation
    read_disable(nh, vc);
    return;
  }
  
  // View the remaining writable space of the MIOBuffer buffer, as the length of the data to be read this time
  int64_t toread = buf.writer()->write_avail();
  // If toread is greater than ntodo, adjust the value of toread
  // Note: toread may be equal to 0
  if (toread > ntodo)
    toread = ntodo;

  // read data
  // read the data
  // Because the underlying MIOBuffer may have multiple IOBufferBlocks, the readv system call is used to register multiple data blocks to the IOVec vector for read operations.
  // total_read indicates the length of the data actually read
  int64_t rattempted = 0, total_read = 0;
  int niov = 0;
  IOVec tiovec [NET_MAX_IOV];
  // Only true read operations are performed when toread is greater than 0
  if (toread) {
    IOBufferBlock *b = buf.writer()->first_write_block();
    do {
      // The niov value is set to the number of IOVec vectors, indicating that several memory blocks are used to hold the data.
      n = 0;
      // The rattempted value is set to the length of the data read by this plan
      rattempted = 0;
      
      // Build an IOVec vector array based on the underlying IOBufferBlock
      // But the maximum number of IOVec vectors per build is NET_MAX_IOV
      // If you want to read more data, it is divided into multiple calls readv
      while (b && niov < NET_MAX_IOV) {
        int64_t a = b->write_avail();
        if (a > 0) {
          tiovec[niov].iov_base = b->_end;
          int64_t togo = toread - total_read - rattempted;
          if (a > togo)
            a = togo;
          tiovec [niov] .iov_len = a;
          rattempted += a;
          nii ++;
          if (a >= togo)
            break;
        }
        b = b->next;
      }

      // If there is only one data block, call the read method, otherwise call the readv method
      if (niov == 1) {
        r = socketManager.read(vc->con.fd, tiovec[0].iov_base, tiovec[0].iov_len);
      } else {
        r = socketManager.readv(vc->con.fd, &tiovec[0], niov);
      }
      // Update the number of read operations counter
      NET_INCREMENT_DYN_STAT(net_calls_to_read_stat);

      // Data read debug trace
      if (vc->origin_trace) {
        char origin_trace_ip[INET6_ADDRSTRLEN];

        ats_ip_ntop(vc->origin_trace_addr, origin_trace_ip, sizeof(origin_trace_ip));

        if (r > 0) {
          TraceIn((vc->origin_trace), vc->get_remote_addr(), vc->get_remote_port(), "CLIENT %s:%d\tbytes=%d\n%.*s", origin_trace_ip,
                  vc->origin_trace_port, (int)r, (int)r, (char *)tiovec[0].iov_base);

        } else if (r == 0) {
          TraceIn((vc->origin_trace), vc->get_remote_addr(), vc->get_remote_port(), "CLIENT %s:%d closed connection",
                  origin_trace_ip, vc->origin_trace_port);
        } else {
          TraceIn((vc->origin_trace), vc->get_remote_addr(), vc->get_remote_port(), "CLIENT %s:%d error=%s", origin_trace_ip,
                  vc->origin_trace_port, strerror(errno));
        }
      }

      // update the total_read counter
      total_read += rattempted;
    // If the data is read successfully (r == rattempted) and the VIO task is not completed (total_read < toread), continue to read the data in the do-while loop
    } while (rattempted && r == rattempted && total_read < toread);
    // If the read fails (r != rattempted), the do-while loop is jumped out and the read operation is stopped. At this point, some data may be read.

    // if we have already moved some bytes successfully, summarize in r
    // Adjust the total_read counter based on the value of r
    if (total_read != rattempted) {
      if (r <= 0)
        r = total_read - rattempted;
      else
        r = total_read - rattempted + r;
    }
    // check for errors
    // When r<=0 means an error has occurred
    if (r <= 0) {
      // EAGAIN means that the buffer is empty, there is no data, then it will be read when the next epoll_wait is triggered.
      if (r == -EAGAIN || r == -ENOTCONN) {
        NET_INCREMENT_DYN_STAT(net_calls_to_read_nodata_stat);
        vc->read.triggered = 0;
        nh->read_ready_list.remove(vc);
        return;
      }

      // If the connection is closed, trigger EOS to notify the upper state machine
      if (!r || r == -ECONNRESET) {
        vc->read.triggered = 0;
        nh->read_ready_list.remove(vc);
        read_signal_done(VC_EVENT_EOS, nh, vc);
        return;
      }
      
      // In other cases, trigger ERROR to notify the upper state machine
      vc->read.triggered = 0;
      read_signal_error(nh, vc, (int)-r);
      return;
    }
    
    // Read successfully, update data read counter
    NET_SUM_DYN_STAT(net_read_bytes_stat, r);

    // Add data to buffer and signal continuation.
    // Update the MIOBuffer counter to increase the amount of data available for consumption
    buf.writer()->fill(r);
#ifdef DEBUG
    if (buf.writer()->write_avail() <= 0)
      Debug("iocore_net", "read_from_net, read buffer full");
#endif
    // Update the VIO counter to increase the amount of data that has been completed
    s-> vio.ndone + = r;
    // refresh timeout timer
    net_activity(vc, thread);
  } else
    r = 0;
  // If the buffer is full, no real read occurs, set r to 0

  // Signal read ready, check if user is not done
  // When r>0, it means that the data has been read this time.
  // Then call back the upper state machine, but need to determine what kind of event should be passed
  if (r) {
    // If there are no more bytes to read, signal read complete
    ink_assert(ntodo >= 0);
    if (s->vio.ntodo() <= 0) {
      // If the VIO requirements are all completed, then callback READ_COMPLETE to the upper state machine
      read_signal_done(VC_EVENT_READ_COMPLETE, nh, vc);
      Debug("iocore_net", "read_from_net, read finished - signal done");
      return;
    } else {
      // If the VIO request is not completed, then callback READ_READY to the upper state machine
      // If the return value is not EVENT_CONT, then it means that the upper state machine has closed vc, so there is no need to continue processing.
      if (read_signal_and_update(VC_EVENT_READ_READY, vc) != EVENT_CONT)
        return;

      // change of lock... don't look at shared variables!
      // If the upper state machine replaced the VIO mutex, then our previous lock has expired.
      // So you can't continue to operate the VIO anymore, you can only reschedule vc, put vc back into read_ready_list and wait for the next NetHandler call.
      if (lock.get_mutex() != s->vio.mutex.m_ptr) {
        read_reschedule(nh, vc);
        return;
      }
    }
  }
  
  // If here are is no more room, or nothing to do, disable the connection
  // in case:
  // MIOBuffer has no space to write (no real read operation occurs)
  // or:
  // read the data, but did not complete all VIO operations, and the upper state machine did not close the VC, nor modified Mutex
  // Then, if:
  // The upper state machine may cancel the operation of VIO: s->vio.ntodo() <= 0
  // The upper state machine may have disabled reading VIO: s->enabled == 0
  // The upper state machine may run out of MIOBuffer: buf.writer()->write_avail() == 0
  if (s->vio.ntodo() <= 0 || !s->enabled || !buf.writer()->write_avail()) {
    // Then close the read operation of this VC.
    read_disable(nh, vc);
    return;
  }

  // Other cases: re-schedule the read operation of vc. At this time, the following conditions must be met:
  //     s->vio.ntodo() > 0
  //     s->enabled == 1
  //     buf.writer()->write_avail() > 0
  read_reschedule(nh, vc);
}
```

## NetHandler extension: from MIOBuffer to Socket

How is the data written to the Socket from MIOBuffer?

Looking back at the analysis of NetHandler::mainNetEvent, we know that:

  - NetHandler discovers whether the underlying socket is writable by epoll_wait
  - Then put the data-readable vc into write_ready_list
  - then iterate over write_ready_list
  - For vc where Write VIO is active, call write_to_net(this, vc, trigger_event->ethread) to send the data.
  - Then call back the upper state machine
  - For the result of the transmission: success, failure, whether to complete VIO, pass the different events to the upper state machine, and pass VC or VIO

The write_to_net function is analyzed in detail below:

```
// write_to_net is a wrapper function that actually calls write_to_net_io()
// Note that both write_to_net() and write_to_net_io() are not internal methods of UnixNetVConnection.
// For SSLVC, these two methods are also called
void
write_to_net(NetHandler *nh, UnixNetVConnection *vc, EThread *thread)
{
  // this line can be deleted, obsolete code
  ProxyMutex *mutex = thread->mutex;

  NET_INCREMENT_DYN_STAT(net_calls_to_writetonet_stat);
  NET_INCREMENT_DYN_STAT(net_calls_to_writetonet_afterpoll_stat);

  write_to_net_io(nh, vc, thread);
}

//
// Write the data for a UnixNetVConnection.
// Rescheduling the UnixNetVConnection when necessary.
//
// write_to_net_io is not responsible for sending data, it needs to call load_buffer_and_write() to complete the data transmission.
// But it is responsible for calling back the upper state machine before and after the data is sent.
// For SSLNetVConnection, encrypt the data by overloading load_buffer_and_write
void
write_to_net_io(NetHandler *nh, UnixNetVConnection *vc, EThread *thread)
{
  NetState *s = &vc->write;
  ProxyMutex *mutex = thread->mutex;

  // Try to get the Mutex lock on this VC's VIO
  MUTEX_TRY_LOCK_FOR(lock, s->vio.mutex, thread, s->vio._cont);
  if (!lock.is_locked() || lock.get_mutex() != s->vio.mutex.m_ptr) {
    // If you don't get the lock,
    // Or after getting the lock, the Vute's Mutex is changed asynchronously.
    // (This is a judge more than read_net_io, it feels amazing, I don't know how this happens.)
    // Reschedule, wait for the next read
    // write_reschedule will put this vc back to the end of the queue of write_ready_link
    // Due to the handling of NetHandler, all write operations will be completed in this EventSystem callback NetHandler until the kernel's write buffer is full.
    // Will not delay until the next EventSystem callback to NetHandler
    write_reschedule(nh, vc);
    return;
  }

  // This function will always return true unless
  // vc is an SSLNetVConnection.
  // Determine if the current vc is an SSLVC, skip it here.
  if (!vc->getSSLHandShakeComplete()) {
    // The code for SSLVC processing is omitted here.
    // ...
  }
  
  // There is no way to determine if the VC is closed asynchronously? ? ? Vc->closed? ? ?
  
  // If it is not enabled,add to WaitList.
  // If the vc write operation is asynchronously disabled
  // Or, the operation method of VIO is not a write operation (here is the judgment condition more than the Read part)
  if (!s->enabled || s->vio.op != VIO::WRITE) {
    write_disable(nh, vc);
    return;
  }
  
  // If there is nothing to do, disable
  // Define the total length of the total data to be sent in VIO, and the length of the data that has been completed.
  // ntodo is the remaining data length that needs to be sent
  int64_t ntodo = s->vio.ntodo();
  if (ntodo <= 0) {
    // ntodo is 0, indicating that the VIO defined write operation has been completed before.
    // So directly disable the write operation of this VC
    write_disable(nh, vc);
    return;
  }

  // The following is the preparation for the write operation.
  // Get the buffer, ready to get the data to send
  MIOBufferAccessor &buf = s->vio.buffer;
  // If the buffer does not exist, it will report an exception and crash
  ink_assert(buf.writer());
  
  // Compare the read part, is to first determine whether the buffer exists, and then judge ntodo
  // Therefore, Write VIO MIOBuffer may be empty when Write VIO is disabled.
  // The official JIRA: https://issues.apache.org/jira/browse/TS-4055 describes the bug because there is no judgment that the Write VIO's MIOBuffer is empty.

  // Calculate amount to write
  // View the data available for reading in the MIOBuffer buffer as the length of the data to be sent this time
  int64_t towrite = buf.reader()->read_avail();
  // If towrite is greater than ntodo, adjust the value of towrite
  // Note: towrite may be equal to 0
  if (towrite > ntodo)
    towrite = ntodo;

  // Mark whether the callback has passed the upper state machine in this call.
  // Since the upper layer state machine may be called back first before the write operation
  // After the write operation, whether to call back the upper state machine again, you need to check whether it has been recalled before.
  // So, record it with a temporary variable.
  int signalled = 0;

  // signal write ready to allow user to fill the buffer
  // If towrite < ntodo : then this time will not call back WRITE_COMPLETE to the upper state machine
  // and there is still space left in the buffer to fill in more data
  if (towrite != ntodo && buf.writer()->write_avail()) {
    // Callback to the upper state machine, passing the WRITE_READY event
    // Let the upper state machine fill the buffer before the write operation
    if (write_signal_and_update(VC_EVENT_WRITE_READY, vc) != EVENT_CONT) {
      // If an error occurs in the upper state machine, it will return directly
      return;
    }
    // Recalculate the value of VIO's ntodo, because VIO may be reset in the upper state machine
    method = s-> vio.n all ();
    // If VIO is set to completed, disable vc's Write VIO and return directly
    if (ntodo <= 0) {
      write_disable(nh, vc);
      return;
    }
    
    // Mark the upper state machine has been called back
    signalled = 1;
    
    // recalculate towrite
    // Recalculate amount to write
    towrite = buf.reader()->read_avail();
    if (towrite > ntodo)
      towrite = ntodo;
  }
  
  // if there is nothing to do, disable
  ink_assert(towrite >= 0);
  // At this point, towrite may still be 0, so if towrite is 0, disable vc's Write VIO and return directly.
  if (towrite <= 0) {
    write_disable(nh, vc);
    return;
  }

  // Call vc->load_buffer_and_write to complete the actual write operation
  // wattempted indicates the amount of data to try to write, total_written indicates the amount of data that was actually written successfully
  int64_t total_written = 0, wattempted = 0;
  // Use the needs to indicate whether the corresponding flag is set by the OR operation after the write operation is completed, and whether the tag needs to reschedule the read and write operations:
  // Used to implement the multiple read and write processes that SSLNetVC needs during the handshake process.
  // For UnixNetVC, always set EVENTIO_WRITE when VIO is not completed and the source MIOBuffer has data readable
  int needs = 0;
  // r is the return value of the last write operation, greater than 0 indicates the actual written byte, and less than or equal to 0 indicates the error code
  int64_t r = vc->load_buffer_and_write(towrite, wattempted, total_written, buf, needs);

  // if we have already moved some bytes successfully, summarize in r
  // If the partial write is successful, adjust the r value to indicate the amount of data that was actually sent successfully.
  if (total_written != wattempted) {
    if (r <= 0)
      r = total_written - wattempted;
    else
      r = total_written - wattempted + r;
  }
  // check for errors
  // When r<=0 means no data is sent because there is no successful write operation
  if (r <= 0) { // if the socket was not ready,add to WaitList
    // Determine the situation of EAGAIN, indicating that the write buffer is full
    if (r == -EAGAIN || r == -ENOTCONN) {
      NET_INCREMENT_DYN_STAT(net_calls_to_write_nodata_stat);
      // Remove from write_ready_list, wait for the next epoll_wait event and continue writing
      if ((needs & EVENTIO_WRITE) == EVENTIO_WRITE) {
        vc->write.triggered = 0;
        nh->write_ready_list.remove(vc);
        write_reschedule(nh, vc);
      }
      // For UnixNetVConnection, needs only sets EVENTIO_WRITE in load_buffer_and_write
      // The following judgment is for SSLNetVConnection
      if ((needs & EVENTIO_READ) == EVENTIO_READ) {
        vc->read.triggered = 0;
        nh->read_ready_list.remove(vc);
        read_reschedule(nh, vc);
      }
      return;
    }
    
    // If r==0 or the connection is closed, the callback upper state machine passes the EOS event
    if (!r || r == -ECONNRESET) {
      vc->write.triggered = 0;
      write_signal_done(VC_EVENT_EOS, nh, vc);
      return;
    }
    
    // In other cases, the callback upper state machine passes the ERROR event and passes -r as data
    vc->write.triggered = 0;
    write_signal_error(nh, vc, (int)-r);
    return;
  } else {
    // When r >= 0, it means that the data is sent successfully, at least part of the data is sent out.
    //
    // WBE = Write Buffer Empty
    // This is a special mechanism. When the write operation completely consumes the MIOBuffer of the data source, it will call back the upper state machine again and pass the wbe_event event.
    // This mechanism is valid once and every time it is triggered, it must be reset in the upper state machine. Otherwise, this mechanism will not be triggered after the next write operation is completed.
    // But if VIO is completed at the same time, then the upper state machine will not be called back.
    int wbe_event = vc->write_buffer_empty_event; // save so we can clear if needed.

    NET_SUM_DYN_STAT(net_write_bytes_stat, r);

    // Remove data from the buffer and signal continuation.
    // Consumption of data in MIOBuffer based on actual data sent
    ink_assert(buf.reader()->read_avail() >= r);
    buf.reader()->consume(r);
    // Simultaneously update the amount of data sent to the VIO counter
    ink_assert(buf.reader()->read_avail() >= 0);
    s-> vio.ndone + = r;

    // If the empty write buffer trap is set, clear it.
    // If the MIOBuffer of the data source is completely consumed, then reset wbe_event to 0.
    if (!(buf.reader()->is_read_avail_more_than(0)))
      vc->write_buffer_empty_event = 0;

    // refresh timeout timer
    net_activity(vc, thread);
    
    // If there are no more bytes to write, signal write complete,
    // Here: Since ntodo=s->vio.ntodo() is the assignment made before the data is sent,
    // So, ntodo indicates the total amount of data that needs to be sent in VIO. It should be greater than 0 when running here.
    // However, s->vio.ntodo() is the value obtained after sending in real time. If it is 0, it means that VIO is completed.
    ink_assert(ntodo >= 0);
    if (s->vio.ntodo() <= 0) {
      // If the VIO requirements are all completed, then call WRITE_COMPLETE to the upper state machine and return
      write_signal_done(VC_EVENT_WRITE_COMPLETE, nh, vc);
      return;
    // If the VIO request has not been completed:
    } else if (signalled && (wbe_event != vc->write_buffer_empty_event)) {
      // @a signalled means we won't send an event, and the event values differing means we
      // had a write buffer trap and cleared it, so we need to send it now. 
      // ! ! ! The comment information here is inconsistent with the code, and the comment should be wrong! ! !
      // Interpreted as follows:
      // If the previous state machine was called back and wbe_event is set, then wbe_event is called back to the upper state machine
      // If the return value is not EVENT_CONT, then it means that the upper state machine has closed vc, so there is no need to continue processing.
      if (write_signal_and_update(wbe_event, vc) != EVENT_CONT)
        return;
      // ! ! ! There is a problem here, the upper state machine may modify the mutex, and if it continues to run here, there may be problems! ! !
    } else if (!signalled) {
      // If there is no callback to the upper state machine, then call WRITE_READY to the upper state machine
      // If the return value is not EVENT_CONT, then it means that the upper state machine has closed vc, so there is no need to continue processing.
      if (write_signal_and_update(VC_EVENT_WRITE_READY, vc) != EVENT_CONT) {
        return;
      }
      
      // change of lock... don't look at shared variables!
      // If the upper state machine replaced the VIO mutex, then our previous lock has expired.
      // So you can't continue to operate the VIO anymore, you can only reschedule vc, put vc back to write_ready_list and wait for the next NetHandler call.
      if (lock.get_mutex() != s->vio.mutex.m_ptr) {
        write_reschedule(nh, vc);
        return;
      }
    }
    
    // The current VIO requirements have not been completed, if:
    // Callback to the upper state machine before the write operation, but no wbe_event is set, and the upper state machine is not called back after the write operation
    // or:
    // After the write operation, the upper state machine is called back, but the upper state machine does not close the VC and does not modify the Mutex.
    // Then, if:
    // The data in the source MIOBuffer has all been sent: buf.reader()->read_avail() == 0
    if (!buf.reader()->read_avail()) {
      // Then close the VC write operation and return
      write_disable(nh, vc);
      return;
    }

    // When the data in the source MIOBuffer is not sent, the read and write operations are rescheduled according to the needs value.
    // For UnixNetVC, the needs are always set to EVENTIO_WRITE
    if ((needs & EVENTIO_WRITE) == EVENTIO_WRITE) {
      write_reschedule(nh, vc);
    }
    // For SSLNetVC, EVENTIO_READ may be set at the same time
    if ((needs & EVENTIO_READ) == EVENTIO_READ) {
      read_reschedule(nh, vc);
    }
    return;
  }
}

// This code was pulled out of write_to_net so
// I could overwrite it for the SSL implementation
// (SSL read does not support overlapped i/o)
// without duplicating all the code in write_to_net.
// consume data of specified length towrite in MIOBuffer through MIOBufferAccessor buf and send it out
// The following parameters must be initialized to 0 before the call:
// wattempted is an address, set to the length of the data that is attempted to be sent when the last call to write/writev is returned on return
// total_written is an address, set to the length of the data to be sent on return, and the caller needs to calculate the length of the data successfully sent based on the return value.
//                    r  < 0 : total_written - wattempted;
//                    r >= 0 : total_written - wattempted + r;
// needs is an address, set to the next step to be tested when returning, can be marked by reading (EVENTIO_READ), writing (EVENTIO_WRITE) by or operation
// return value:
// The last time the return value of write/writev was called.
//
// Encrypted SSL data is sent over SSLNetVConnection by overloading this method
int64_t
UnixNetVConnection::load_buffer_and_write(int64_t towrite, int64_t &wattempted, int64_t &total_written, MIOBufferAccessor &buf,
                                          int &needs)
{
  int64_t r = 0;

  // XXX Rather than dealing with the block directly, we should use the IOBufferReader API.
  // Get the data at the beginning of the MIOBuffer and the first data block
  int64_t offset = buf.reader()->start_offset;
  IOBufferBlock *b = buf.reader()->block;

  // Run the write/writev operation at least once through a do-while loop
  // If the write operation succeeds, continue the loop and stop the loop when any error is encountered.
  do {
    // In order to support multiple IOBufferBlock, through writev to complete the write operation, you need to build an IOVec vector array
    IOVec tiovec [NET_MAX_IOV];
    int niov = 0;
    // total_written_last is the total_written value after the last write/writev call completed
    int64_t total_written_last = total_written;
    while (b && niov < NET_MAX_IOV) {
      // check if we have done this block
      int64_t l = b->read_avail();
      l -= offset;
      if (l <= 0) {
        offset = -l;
        b = b->next;
        continue;
      }
      // check if to amount to write exceeds that in this buffer
      int64_t wavail = towrite - total_written;
      if (l > wavail)
        l = wavail;
      if (!l)
        break;
      total_written += l;
      // build an iov entry
      tiovec [niov] .iov_len = l;
      tiovec[niov].iov_base = b->start() + offset;
      nii ++;
      // on to the next block
      offset = 0;
      b = b->next;
    }
    // Note that total_written has been added to the length of the data to be sent after the IOVec vector array is constructed.
    // Set the length of the data that is attempted to be sent when the current is looped / this time write/writev is called
    wattempted = total_written - total_written_last;
    // If there is only one data block, use write, otherwise use writev
    if (niov == 1)
      r = socketManager.write(con.fd, tiovec[0].iov_base, tiovec[0].iov_len);
    else
      r = socketManager.writev(con.fd, &tiovec[0], niov);

    // Debug data transmission
    if (origin_trace) {
      char origin_trace_ip[INET6_ADDRSTRLEN];
      ats_ip_ntop(origin_trace_addr, origin_trace_ip, sizeof(origin_trace_ip));

      if (r > 0) {
        TraceOut(origin_trace, get_remote_addr(), get_remote_port(), "CLIENT %s:%d\tbytes=%d\n%.*s", origin_trace_ip,
                 origin_trace_port, (int)r, (int)r, (char *)tiovec[0].iov_base);

      } else if (r == 0) {
        TraceOut(origin_trace, get_remote_addr(), get_remote_port(), "CLIENT %s:%d closed connection", origin_trace_ip,
                 origin_trace_port);
      } else {
        TraceOut(origin_trace, get_remote_addr(), get_remote_port(), "CLIENT %s:%d error=%s", origin_trace_ip, origin_trace_port,
                 strerror(errno));
      }
    }

    // Update the number of write operations counter
    ProxyMutex *mutex = thread->mutex;
    NET_INCREMENT_DYN_STAT(net_calls_to_write_stat);
    // If the data is read successfully (r == rattempted) and the send task is not completed (total_written < towrite), continue to read the data in the do-while loop
  } while (r == wattempted && total_written < towrite);
  // If the read fails (r != rattempted):
  // Then jump out of the do-while loop and stop the read operation. At this point, some data may be sent.
  // It may be that the send task is completed (total_written == towrite)

  // Set the flag that needs to be written later
  needs |= EVENTIO_WRITE;

  // Returns the return value of the last write / writev
  // The caller needs to correct the total_written value based on the r value
  return (r);
}
```

## NetHandler extension: callback processing for the upper state machine

If read and write VIO is set, then in the read and write operations, a callback to the upper state machine will be generated:

  - 传递COMPLETE事件：read_signal_done，write_signal_done
  - 传递ERROR事件：read_signal_error，write_signal_error
  - 传递READY事件：read_signal_and_update，write_signal_and_update

Both signal_error and signal_done call signal_and_update as the underlying implementation, so the following is done with read_signal_and_update (the flow of write_signal_and_update is exactly the same).

```
static inline int
read_signal_and_update(int event, UnixNetVConnection *vc)
{
  // increase vc reentrant counter
  vc->recursion++;
  if (vc->read.vio._cont) {
    // If the upper state machine is set to VIO, then call the VIO associated upper state machine
    vc->read.vio._cont->handleEvent(event, &vc->read.vio);
  } else {
    // If the upper state machine does not have VIO set, then the default processing is performed.
    switch (event) {
    case VC_EVENT_EOS:
    case VC_EVENT_ERROR:
    case VC_EVENT_ACTIVE_TIMEOUT:
    case VC_EVENT_INACTIVITY_TIMEOUT:
      // EOS appears, error, timeout, set vc off
      // Where: EOS and ERROR are generated for IOCoreNET, and Timeout is generated by InactivityCop
      Debug("inactivity_cop", "event %d: null read.vio cont, closing vc %p", event, vc);
      vc->closed = 1;
      break;
    default:
      Error("Unexpected event %d for vc %p", event, vc);
      ink_release_assert(0);
      break;
    }
  }
  // Only perform the operation of closing vc when the reentrant counter is 1, and vc is set to off.
  if (!--vc->recursion && vc->closed) {
    /* BZ  31932 */
    ink_assert(vc->thread == this_ethread());
    close_UnixNetVConnection(vc, vc->thread);
    return EVENT_DONE;
  } else {
    return EVENT_CONT;
  }
}

```

## NetHandler extension: Reentrant for VC callbacks

In the implementation of Proxy, a server is composed of ClientSide's ServerVC and ServerSide's ClientVC:

  - ServerVC accepts data from the client and then sends it to Origin Server via ClientVC
  - ServerVC is active at this time, and ClientVC is passive.
  - ClientVC accepts data from Origin Server and then sends it to Client via ServerVC
  - ClientVC is active at this time, and ServerVC is passive.

Consider a possible scenario:

  - If ServerVC is managed by ET_NET 0, ClientVC is managed by ET_NET 1
  - ServerVC received data from the client and is writing to ClientVC
  - At this time, ClientVC also received data from Origin Server and is reading data.
  - Of course, these two operations are responsible for two different state machines, but VCs are all the same: ClientVC

PS: This scene is my imagination. I haven't gone to the code to find the callback reentrant scene. If you can provide the code related to this in ATS, please submit the pull request, thank you!

Then for ClientVC, there are concurrent read and write operations, if one of the parties thinks the VC needs to be closed:

  - You must wait for all state machines that use this VC to end
  - At this point, I encountered a callback reentrant problem for VC.

ATS solves this problem by setting a reentrant counter to the VC.

Question: Why is the vc mutex locked when the state machine is called back?

  - Using mutex should follow a principle: when accessing an object's internal resources, the mutex of the object is locked, so:
    - Lock the vio.mutex before the nethandler performs read and write operations because it wants to manipulate the MIOBuffer in vio
    - When the nethandler calls back to the upper state machine before or after the read and write operations, it also calls vio._cont to call back the upper state machine.
  - When calling the state machine, there is no access to the internal resources of any vc object, so there is no need to lock the mutex of vc

## NetHandler Extension: Section

In NetHandler::mainNetEvent, the following nested calls are made:

  - UnixNetVConnection::net_read_io(this, trigger_event->ethread)
    - read_from_net(nh, this, lthread)
  - write_to_net(nh, vc, thread)
    - write_to_net_io(nh, vc, thread)
      - load_buffer_and_write

The data stream is passed between the socket and the buffer, so I feel that read_from_net, write_to_net and write_to_net_io are part of NetHandler, and only net_read_io, load_buffer_and_write is the UnixNetVConnection method, because the SSLNetVConnection inheriting from it needs to be overloaded. Methods to achieve data encryption and decryption.

## Timeout Control and Management

For Active Timeout and Inactivity Timeout, ATS provides the following methods:

  - get_(inactivity|active)_timeout
  - set_(inactivity|active)_timeout
  - cancel_(inactivity|active)_timeout

```
// The get method is relatively simple, directly return the value of the member
// The suffix _in and the suffix of schedule_in are a meaning, relative to the current time, relative timeout
TS_INLINE ink_hrtime
UnixNetVConnection::get_active_timeout()
{
  return active_timeout_in;
}

TS_INLINE ink_hrtime
UnixNetVConnection::get_inactivity_timeout()
{
  return inactivity_timeout_in;
}

// The set method needs to be determined by the macro definition according to the way of the timeout mechanism.
// But in either case you need to set the value of inactivity_timeout_in
TS_INLINE void
UnixNetVConnection::set_inactivity_timeout(ink_hrtime timeout)
{
  Debug("socket", "Set inactive timeout=%" PRId64 ", for NetVC=%p", timeout, this);
  inactivity_timeout_in = timeout;
#ifdef INACTIVITY_TIMEOUT
  // If you have previously set inactivity_timeout, you must first cancel the previous settings.
  if (inactivity_timeout)
    inactivity_timeout->cancel_action(this);

  // Then determine if you want to set a new timeout control, inactivity_timeout_in == 0 means cancel the timeout control
  if (inactivity_timeout_in) {
    if (read.enabled) {
      // Read operation is activated, setting the timeout event of the read operation
      ink_assert(read.vio.mutex->thread_holding == this_ethread() && thread);
      // is divided into synchronous scheduling and asynchronous scheduling, setting the mainEvent of the VC state machine after the inactivity_timeout_in time
      if (read.vio.mutex->thread_holding == thread)
        inactivity_timeout = thread->schedule_in_local(this, inactivity_timeout_in);
      else
        inactivity_timeout = thread->schedule_in(this, inactivity_timeout_in);
    } else if (write.enabled) {
      // judge if the read operation is not set
      // Write operation is activated, set the timeout event of the write operation
      ink_assert(write.vio.mutex->thread_holding == this_ethread() && thread);
      // is divided into synchronous scheduling and asynchronous scheduling, setting the mainEvent of the VC state machine after the inactivity_timeout_in time
      if (write.vio.mutex->thread_holding == thread)
        inactivity_timeout = thread->schedule_in_local(this, inactivity_timeout_in);
      else
        inactivity_timeout = thread->schedule_in(this, inactivity_timeout_in);
    } else {
      // The read and write operations are not set, the timeout setting does not take effect.
      // Clear the pointer to the event
      inactivity_timeout = 0;
    }
  } else {
    // Clear the pointer to the event
    inactivity_timeout = 0;
  }
#else
  // Set the absolute value of the timeout time corresponding to inactivity_timeout
  next_inactivity_timeout_at = Thread::get_hrtime() + timeout;
#endif
}

// The set method needs to be determined by the macro definition according to the way of the timeout mechanism.
// But in either case you need to set the value of active_timeout_in
// The process is exactly the same as set_inactivity_timeout
TS_INLINE void
UnixNetVConnection::set_active_timeout(ink_hrtime timeout)
{
  Debug("socket", "Set active timeout=%" PRId64 ", NetVC=%p", timeout, this);
  active_timeout_in = timeout;
#ifdef INACTIVITY_TIMEOUT
  if (active_timeout)
    active_timeout->cancel_action(this);
  if (active_timeout_in) {
    if (read.enabled) {
      ink_assert(read.vio.mutex->thread_holding == this_ethread() && thread);
      if (read.vio.mutex->thread_holding == thread)
        active_timeout = thread->schedule_in_local(this, active_timeout_in);
      else
        active_timeout = thread->schedule_in(this, active_timeout_in);
    } else if (write.enabled) {
      ink_assert(write.vio.mutex->thread_holding == this_ethread() && thread);
      if (write.vio.mutex->thread_holding == thread)
        active_timeout = thread->schedule_in_local(this, active_timeout_in);
      else
        active_timeout = thread->schedule_in(this, active_timeout_in);
    } else
      active_timeout = 0;
  } else
    active_timeout = 0;
#else
  next_activity_timeout_at = Thread::get_hrtime() + timeout;
#endif
}

// cancel the timeout setting
// The cancel method also needs to be determined by the macro definition according to the way of the timeout mechanism.
// But either way needs to clear the value of inactivity_timeout_in
TS_INLINE void
UnixNetVConnection::cancel_inactivity_timeout()
{
  Debug("socket", "Cancel inactive timeout for NetVC=%p", this);
  inactivity_timeout_in = 0;
#ifdef INACTIVITY_TIMEOUT
  // cancel the event
  if (inactivity_timeout) {
    Debug("socket", "Cancel inactive timeout for NetVC=%p", this);
    inactivity_timeout->cancel_action(this);
    inactivity_timeout = NULL;
  }
#else
  // Set the inactivity_timeout corresponding to the timeout value of the absolute value of 0
  next_inactivity_timeout_at = 0;
#endif
}

// The cancel method also needs to be determined by the macro definition according to the way of the timeout mechanism.
// But either way needs to clear the value of active_timeout_in
// The process is exactly the same as cancel_inactivity_timeout
TS_INLINE void
UnixNetVConnection::cancel_active_timeout()
{
  Debug("socket", "Cancel active timeout for NetVC=%p", this);
  active_timeout_in = 0;
#ifdef INACTIVITY_TIMEOUT
  if (active_timeout) {
    Debug("socket", "Cancel active timeout for NetVC=%p", this);
    active_timeout->cancel_action(this);
    active_timeout = NULL;
  }
#else
  next_activity_timeout_at = 0;
#endif
}
```

Whether Inactivity Timeout or Active Timeout requires activation of read and write operations, what is the difference between these two Timeouts?

In the iocore/net/NetVCTest.cc file:

```
void
NetVCTest::start_test()
{
  test_vc->set_inactivity_timeout(HRTIME_SECONDS(timeout));
  test_vc->set_active_timeout(HRTIME_SECONDS(timeout + 5));

  read_buffer = new_MIOBuffer();
  write_buffer = new_MIOBuffer();

  reader_for_rbuf = read_buffer->alloc_reader();
  reader_for_wbuf = write_buffer->alloc_reader();

  if (nbytes_read > 0) {
    read_vio = test_vc->do_io_read(this, nbytes_read, read_buffer);
  } else {
    read_done = true;
  }

  if (nbytes_write > 0) {
    write_vio = test_vc->do_io_write(this, nbytes_write, reader_for_wbuf);
  } else {
    write_done = true;
  }
}
```

The active_timeout is set to be 5 seconds longer, so you can guess that active_timeout is longer than inactivity_timeout.

The comment in iocore/net/I_NetVConnection.h illustrates the difference (with a simple translation):

```
  / **
    After the state machine uses this NetVC for a certain period of time, it receives a VC_EVENT_ACTIVE_TIMEOUT event.
    If both read and write are not activated on this NetVC, this value will be ignored.
    This feature prevents the state machine from keeping the connection open for a longer period of time.
    
    Sets time after which SM should be notified.
    Sets the amount of time (in nanoseconds) after which the state
    machine using the NetVConnection should receive a
    VC_EVENT_ACTIVE_TIMEOUT event. The timeout is value is ignored
    if neither the read side nor the write side of the connection
    is currently active. The timer is reset if the function is
    called repeatedly This call can be used by SMs to make sure
    that it does not keep any connections open for a really long
    time.
    ...
   * /
  virtual void set_active_timeout(ink_hrtime timeout_in) = 0;
    
  / **
    When the state machine requests that the IO operation performed by this NetVC is not completed,
    The status opportunity receives a VC_EVENT_INACTIVITY_TIMEOUT event after the read and write are in the IDLE state for a certain period of time.
    Any read or write operation will cause the timer to be reset.
    If both read and write are not activated on this NetVC, this value will be ignored.
    
    Sets time after which SM should be notified if the requested
    IO could not be performed. Sets the amount of time (in nanoseconds),
    if the NetVConnection is idle on both the read or write side,
    after which the state machine using the NetVConnection should
    receive a VC_EVENT_INACTIVITY_TIMEOUT event. Either read or
    write traffic will cause timer to be reset. Calling this function
    again also resets the timer. The timeout is value is ignored
    if neither the read side nor the write side of the connection
    is currently active. See section on timeout semantics above.
   * /
  virtual void set_inactivity_timeout(ink_hrtime timeout_in) = 0;
```

and so,

 - Active Timeout, which is to set a maximum lifetime of NetVC
 - Inactivity Timeout is to set the longest IDLE time

The above describes the method of obtaining the timeout setting, setting the timeout period, and canceling the timeout control. Then, who implements the timeout function in the ATS?

Please continue reading the analysis of the mainEvent callback function below.

## UnixNetVConnection state machine callback function

UnixNetVConnection is polymorphic. It has a function similar to the Event type for passing events, and it is also a state machine with its own callback function.

### Accept new link (acceptEvent)

The callback function acceptEvent is an extension of NetAccept in NetVConnection.

When the ATS sets up a separate accept thread, after a new socket connection is established,

  - Create a new vc associated with this socket,
  - and set the callback function of this vc to acceptEvent,
  - then find a thread from the corresponding thread group according to the rotation training algorithm,
  - Give this vc to the thread management,
  - When the first callback is called, acceptEvent is called

Let's take a look at the process analysis of acceptEvent:

```
int
UnixNetVConnection::acceptEvent(int event, Event *e)
{
  // Set the thread of vc, there will be this thread to manage this vc, until vc close
  thread = e->ethread;

  // Try to lock the NetHandler
  MUTEX_TRY_LOCK(lock, get_NetHandler(thread)->mutex, e->ethread);
  // If the lock fails
  if (!lock.is_locked()) {
    // The net_accept method is called in NetAccept::acceptEvent(), and the callback vc state machine is synchronized in the net_accept method.
    // When event == EVENT_NONE means this is a synchronous callback from the net_accept method
    // Usually only Management, the Cluster part will use NetAccept::init_accept to create a NetAccept instance using the net_accept method
    if (event == EVENT_NONE) {
      // Here we use the ethread dispatch function to reschedule the event, as this might be a cross-thread call, ie: e->ethread != this_ethread()
      // So use the dispatch method provided in the ethread that supports atomic operations. At this point, put this vc into the external queue of EThread.
      thread->schedule_in(this, HRTIME_MSECONDS(net_retry_delay));
      // Since it is a synchronous callback, tell the caller that the call has been completed, because the event has been put into the thread that manages this vc, and the thread takes over the follow-up work.
      return EVENT_DONE;
    } else {
      // If it is a callback from eventsystem, it should be passed the EVENT_IMMEDIATE event
      // Because vc is first placed in the thread after creation, it is passed the schedule_imm method
      // Since it is from eventsystem, ie: e->ethread == this_ethread()
      // only need to reschedule the event by passing in the event and then callback again after net_retry_delay time
      // At this point, this vc is placed in the local queue of the EThread external queue.
      e->schedule_in(HRTIME_MSECONDS(net_retry_delay));
      return EVENT_CONT;
    }
  }

  // If it is successfully locked
  
  // Determine whether it is canceled
  if (action_.cancelled) {
    free(thread);
    return EVENT_DONE;
  }

  // Set the callback function to mainEvent, after which IOCoreNet callback for this vc is mainEvent until vc is closed
  SET_HANDLER((NetVConnHandler)&UnixNetVConnection::mainEvent);

  // Put vc into the monitor of epoll_wait via PollDescriptor and EventIO
  nh = get_NetHandler(thread);
  PollDescriptor *pd = get_PollDescriptor(thread);
  if (ep.start(pd, this, EVENTIO_READ | EVENTIO_WRITE) < 0) {
    // error handling
    Debug("iocore_net", "acceptEvent : failed EventIO::start\n");
    close_UnixNetVConnection(this, e->ethread);
    return EVENT_DONE;
  }

  // Add to the open_list managed by NetHandler, all open vc will be put into open_list, accept timeout management
  nh->open_list.enqueue(this);

  // If using EDGE mode, of course, we use epoll is this mode
#ifdef USE_EDGE_TRIGGER
  // Set the vc as triggered and place it in the read ready queue in case there is already data on the socket.
  Debug("iocore_net", "acceptEvent : Setting triggered and adding to the read ready queue");
  // Because we may have set TCP_DEFER_ACCEPT on the socket before, so there may already be data readable here.
  // Directly simulate the operation when activated by epoll_wait, set triggered, and put vc into ready_list
  read.triggered = 1;
  nh->read_ready_list.enqueue(this);
#endif

  // Set the inactivity timeout
  if (inactivity_timeout_in) {
    UnixNetVConnection::set_inactivity_timeout(inactivity_timeout_in);
  }

  // Set active timeout
  if (active_timeout_in) {
    UnixNetVConnection::set_active_timeout(active_timeout_in);
  }

  // Callback upper state is low, send NET_EVENT_ACCEPT event
  action_.continuation->handleEvent(NET_EVENT_ACCEPT, this);
  return EVENT_DONE;
}
```

### Initiate a new link (startEvent)

When you need to initiate a connection to the outside, first create a vc, and then set the relevant parameters, you can put vc into the eventsystem.

When startEvent is returned, connectUp will be called, and subsequent operations will be completed by connectUp. The connectUp method will be introduced in the following sections.

```
int
UnixNetVConnection::startEvent(int /* event ATS_UNUSED */, Event *e)
{
  // Try to lock the NetHandler
  MUTEX_TRY_LOCK(lock, get_NetHandler(e->ethread)->mutex, e->ethread);
  if (!lock.is_locked()) {
    // Reschedule if the lock fails
    e->schedule_in(HRTIME_MSECONDS(net_retry_delay));
    return EVENT_CONT;
  }
  // call connectUp when it is not canceled
  if (!action_.cancelled)
    connectUp(e->ethread, NO_FD);
  else
    free(e->ethread);
  return EVENT_DONE;
}
```

### Main handler (mainEvent)

The mainEvent is mainly used to implement timeout control, and the Inactivity Timeout and Active Timeout controls are here.

In ATS, the processing of TIMEOUT is divided into two modes:

  - Completed with two events, active_timeout and inactivity_timeout
    - Is to encapsulate each vc's timeout control into a timed execution of the Event into the EventSystem to handle
    - This will cause a vc to generate two events. The processing pressure on EventSystem is very large, so ATS designed InactivityCop.
    - This method is used in ATS 5.3.x and earlier
  - Completed by InactivityCop
    - This is a state machine that manages connection timeouts. I will introduce this state machine later.
    - ATS 6.0.0 started this way

Prior to 6.0.0, defined in [P_UnixNet.h] (http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)

```
#define INACTIVITY_TIMEOUT
```

This is a timeout control implemented using a mechanism based on the EventSystem timed event. Two members are also defined in the UnixNetVConnection to work with:

```
#ifdef INACTIVITY_TIMEOUT
  Event *inactivity_timeout;
  Event *activity_timeout;
#else
  ink_hrtime next_inactivity_timeout_at;
  ink_hrtime next_activity_timeout_at;
#endif
```

However, starting at 6.0.0, this line is commented out, and a separate state machine InactivityCop is used to implement timeout control.

In this chapter, we introduce the early timeout processing mechanism. In the following chapters, we introduce the InactivityCop state machine.


```
//
// The main event for UnixNetVConnections.
// This is called by the Event subsystem to initialize the UnixNetVConnection
// and for active and inactivity timeouts.
//
int
UnixNetVConnection::mainEvent(int event, Event *e)
{
  // assert judgment
  // EVENT_IMMEDIATE callback from InactivityCop to Inactivity Timeout
  // EVENT_INTERVAL callback from Activesystem to Active Timeout
  ink_assert(event == EVENT_IMMEDIATE || event == EVENT_INTERVAL);
  // Only the current ethread is the thread that manages the vc, can call this method
  ink_assert(thread == this_ethread());

  // Because it is a timeout control, so try to add a bunch of locks, to lock all aspects of the use of vc
  // The various parts of these associations include: NetHandler, Read VIO, Write VIO
  // Usually VIO's mutex will reference the mutex of the upper state machine that registered VIO
  MUTEX_TRY_LOCK(hlock, get_NetHandler(thread)->mutex, e->ethread);
  MUTEX_TRY_LOCK(rlock, read.vio.mutex ? (ProxyMutex *)read.vio.mutex : (ProxyMutex *)e->ethread->mutex, e->ethread);
  MUTEX_TRY_LOCK(wlock, write.vio.mutex ? (ProxyMutex *)write.vio.mutex : (ProxyMutex *)e->ethread->mutex, e->ethread);
  // Determine whether the above three parts are locked successfully
  if (!hlock.is_locked() || !rlock.is_locked() || !wlock.is_locked() ||
  // Determine that Read VIO has not been changed
      (read.vio.mutex.m_ptr && rlock.get_mutex() != read.vio.mutex.m_ptr) ||
  // Determine that Write VIO has not been changed
      (write.vio.mutex.m_ptr && wlock.get_mutex() != write.vio.mutex.m_ptr)) {
    // If the above judgment fails, it needs to be rescheduled.
#ifdef INACTIVITY_TIMEOUT
    // but only re-scheduled by the active_timeout event callback
    // Why does the callback of the inactivity_timeout event return directly after the lock fails, without rescheduling? ? ?
    // Because there is a lock failure, it means that there is an operation to reset the inactivity_timeout timer, so there is no need to reschedule.
    if (e == active_timeout)
#endif
      e->schedule_in(HRTIME_MSECONDS(net_retry_delay));
    return EVENT_CONT;
  }

  // All locks succeeded and VIO has not been changed
  // Determine if the Event is canceled
  if (e->cancelled) {
    return EVENT_DONE;
  }

  // Next to determine whether it is Active Timeout or Inactivity Timeout
  int signal_event;
  Continuation *reader_cont = NULL;
  Continuation *writer_cont = NULL;
  ink_hrtime *signal_timeout_at = NULL;
  Event *t = NULL;
  // signal_timeout is used to support the old timeout control mode, fixed in the new mode.
  Event **signal_timeout;
  signal_timeout = &t;

#ifdef INACTIVITY_TIMEOUT
  // When using EventSystem to implement timeout control:
  // The incoming e should be inactivity_timeout or active_timeout
  // According to the value of e to determine the type of timeout occurs, set the parameters
  if (e == inactivity_timeout) {
    signal_event = VC_EVENT_INACTIVITY_TIMEOUT;
    signal_timeout = &inactivity_timeout;
  } else {
    ink_assert(e == active_timeout);
    signal_event = VC_EVENT_ACTIVE_TIMEOUT;
    signal_timeout = &active_timeout;
  }
#else
  // When using InactivityCop to implement timeout control:
  if (event == EVENT_IMMEDIATE) {
    // If it is a callback from InactivityCop, the incoming event is fixed to EVENT_IMMEDIATE
    // This indicates that an Inactivity Timeout has occurred
    // Inactivity Timeout
    /* BZ 49408 */
    // ink_assert(inactivity_timeout_in);
    // ink_assert(next_inactivity_timeout_at < ink_get_hrtime());
    // in case:
    // Inactivity Timeout is not set (inactivity_timeout_in==0)
    //     没有出现Inactivity Timeout (next_inactivity_timeout_at > Thread::get_hrtime())
    // return directly to EVENT_CONT
    if (!inactivity_timeout_in || next_inactivity_timeout_at > Thread::get_hrtime())
      return EVENT_CONT;
    // Inactivity Timeout appears, so set the event type of the callback upper state machine to INACTIVITY_TIMEOUT
    signal_event = VC_EVENT_INACTIVITY_TIMEOUT;
    // The pointer points to the timeout period, so that there is no need to judge the timeout period based on the timeout type.
    signal_timeout_at = &next_inactivity_timeout_at;
  } else {
    // When the incoming event is EVENT_INTERVAL
    // This indicates that Active Timeout has occurred
    // Usually only eventSystem callbacks for timed, periodically executed events are passed to EVENT_INTERVAL
    // But I don't see any place to do the schedule. If the EventSystem callback, then where is the incoming event created? ? ?
    // event == EVENT_INTERVAL
    // Active Timeout
    // Active Timeout appears, so set the event type of the callback upper state machine to ACTIVE_TIMEOUT
    signal_event = VC_EVENT_ACTIVE_TIMEOUT;
    // pointer points to timeout
    signal_timeout_at = &next_activity_timeout_at;
  }
#endif

  // Reset the timeout value
  *signal_timeout = 0;
  // Reset the absolute value of the timeout
  *signal_timeout_at = 0;
  // record writer_cont
  writer_cont = write.vio._cont;

  // If the vc has been closed, directly call close_UnixNetVConnection to close the vc
  if (closed) {
    close_UnixNetVConnection(this, thread);
    return EVENT_DONE;
  }

  // From here on, the following code logic is in line according to iocore/net/I_NetVConnection.h: 380 ~ 393
  // Implemented in the description in the Timeout symantics section.

  // in case:
  // Set Read VIO, the op value might be: VIO::READ or VIO::NONE
  // And there is no half-close to the read end, that is, the Socket corresponding to the VC is in a readable state.
  if (read.vio.op == VIO::READ && !(f.shutdown & NET_VC_SHUTDOWN_READ)) {
    // record reader_cont
    reader_cont = read.vio._cont;
    // Up to the state machine callback timeout event, at which point the reader_cont has been called back.
    if (read_signal_and_update(signal_event, this) == EVENT_DONE)
      return EVENT_DONE;
  }

  // The first three conditions mainly determine whether the timeout control is reset again when the reader_cont callback is just changed, and possibly close vc (this seems redundant?):
  // The first condition determines if the timeout is set: !*signal_timeout
  // The second condition determines if the timeout is set: !*signal_timeout_at
  // The third condition determines whether vc is closed: !closed, previously judged, if VC is turned off when callback reader_cont, then EVENT_DONE will be returned above
  // in case:
  // Set Write VIO, the op value might be: VIO::WRITE or VIO::NONE
  // And there is no half-close to the sender, that is, the Socket corresponding to the VC is writable.
  // and the state machine corresponding to Write VIO is not the reader_cont that has just been called back.
  // and the current Write VIO is still the previously recorded writer_cont, because the Write VIO state machine may be reset when the reader_cont is previously called back
  if (!*signal_timeout && !*signal_timeout_at && !closed && write.vio.op == VIO::WRITE && !(f.shutdown & NET_VC_SHUTDOWN_WRITE) &&
      reader_cont != write.vio._cont && writer_cont == write.vio._cont)
    // Up to the state machine callback timeout event
    if (write_signal_and_update(signal_event, this) == EVENT_DONE)
      return EVENT_DONE;

  // Finally always returns EVENT_DONE
  return EVENT_DONE;
}
```

## Method

### Creating a connection to Origin Server (connectUp)

How do I create a connection to the source server?

```
int
UnixNetVConnection::connectUp(EThread *t, int fd)
{
  int res;

  // Set the management thread of the new vc to t
  thread = t;
  
  // If the maximum number of connections allowed is exceeded, the failure will be declared immediately
  // Callback to create this vc state machine, the state machine needs to close this vc
  if (check_net_throttle(CONNECT, submit_time)) {
    check_throttle_warning();
    action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)-ENET_THROTTLING);
    free(t);
    return CONNECT_FAILURE;
  }

  // Force family to agree with remote (server) address.
  options.ip_family = server_addr.sa.sa_family;

  //
  // Initialize this UnixNetVConnection
  //
  // Print debugging information
  if (is_debug_tag_set("iocore_net")) {
    char addrbuf [INET6_ADDRSTRLEN];
    Debug("iocore_net", "connectUp:: local_addr=%s:%d [%s]\n",
          options.local_ip.isValid() ? options.local_ip.toString(addrbuf, sizeof(addrbuf)) : "*", options.local_port,
          NetVCOptions::toString(options.addr_binding));
  }

  // If this is getting called from the TS API, then we are wiring up a file descriptor
  // provided by the caller. In that case, we know that the socket is already connected.
  // If you didn't create the underlying socket fd, create one
  if (fd == NO_FD) {
    res = con.open(options);
    if (res != 0) {
      goto fail;
    }
  } else {
    // Usually, from the TS API call, the underlying socket fd has been created.
    int len = sizeof(con.sock_type);

    // This call will fail if fd is not a socket (e.g. it is a
    // eventfd or a regular file fd.  That is ok, because sock_type
    // is only used when setting up the socket.
    // only need to set the members of the con object
    safe_getsockopt(fd, SOL_SOCKET, SO_TYPE, (char *)&con.sock_type, &len);
    safe_nonblocking(fd);
    con.fd = fd;
    con.is_connected = true;
    con.is_bound = true;
  }

  // Must connect after EventIO::Start() to avoid a race condition
  // when edge triggering is used.
  // Put vc into the monitor of epoll_wait via EventIO::start, READ & WRITE
  // The comment here says that when using the EPOLL ET mode, you must execute start and then execute connect to avoid a race condition.
  // But I didn't understand it. What competitive conditions are there? ? ?
  if (ep.start(get_PollDescriptor(t), this, EVENTIO_READ | EVENTIO_WRITE) < 0) {
    lerrno = errno;
    Debug("iocore_net", "connectUp : Failed to add to epoll list\n");
    action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)0); // 0 == res
    // con.close() is called by the destructor of the Connection class to close fd
    free(t);
    return CONNECT_FAILURE;
  }

  // If the underlying socket fd is not created, you need to call connect to initiate the connection.
  // Note that this is non-blocking IO, so it is necessary to judge the socket fd to be written in the state machine.
  // Here again to see the race conditions mentioned above, if socket fd has been created, then there is a situation before connect is in EventIO::start
  // So what is the competitive condition here? ? ?
  if (fd == NO_FD) {
    res = con.connect(&server_addr.sa, options);
    if (res != 0) {
      // con.close() is called by the destructor of the Connection class to close fd
      goto fail;
    }
  }

  check_emergency_throttle(con);

  // start up next round immediately

  // Switch vc handler to mainEvent
  SET_HANDLER(&UnixNetVConnection::mainEvent);

  // put in the open_list queue
  nh = get_NetHandler(t);
  nh->open_list.enqueue(this);

  ink_assert(!inactivity_timeout_in);
  ink_assert(!active_timeout_in);
  this->set_local_addr();
  // Callback create this vc state machine, NET_EVENT_OPEN, create this vc state opportunity to continue to call back the upper state machine
  // In the upper state machine, you can set VIO, then the upper state machine can receive the READ|WRITE_READY event.
  // Note that there is no way to determine if the non-blocking connect method successfully opened the connection.
  action_.continuation->handleEvent(NET_EVENT_OPEN, this);
  return CONNECT_SUCCESS;

fail:
  // failed processing, saving the value of errno
  lerrno = errno;
  // Callback to create this vc state machine, the state machine needs to close this vc
  action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)(intptr_t)res);
  free(t);
  return CONNECT_FAILURE;
}
```

### 激活 VIO & VC（reenable & reenable_re）

Two methods, reenable and reenable_re, are provided in UnixNetVConnection, corresponding to the VIO defined in [P_VIO.h] (http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_VIO.h) Two methods of reenable and reenable_re.

```
TS_INLINE void
VIO :: reenable ()
{
  if (vc_server)
    vc_server->reenable(this);
}
TS_INLINE void
VIO::reenable_re()
{
  if (vc_server)
    vc_server->reenable_re(this);
}
```

We know that one end of VIO is VC, the other end is MIOBuffer, and VC is always active, driven by IOCore, so the reenable to VIO is VC's reenable.

So what is the difference between reenable and reenable_re? In what scenario is it used separately? With questions, let's comment and analyze the two methods first.

```
//
// Function used to reenable the VC for reading or
// writing.
//
void
UnixNetVConnection :: reenable (VIO * saw)
{
  // Get the NetState object containing this VIO
  // The enabled NetState object is true, indicating that it is already active.
  if (STATE_FROM_VIO(vio)->enabled)
    return;
  // Set enabled to true, while resetting inactivity timeout
  set_enabled(vio);
  // The thread member is defined in the base class NetVConnection and is used to point to the EThread that manages the vc.
  // Initialize thread to e->ethread in NetAccept or acceptEvent
  // If the vc has not been delivered to an EThread for management, the description has not been managed by the NetHandler and therefore cannot be activated.
  if (!thread)
    return;
  
  // Must be locked to vio before executing reenable (the thread that locks the vio must be the thread that called this method)
  EThread *t = vio->mutex->thread_holding;
  ink_assert(t == this_ethread());
  // Can not perform a reenable operation on the closed vc
  ink_assert(!closed);
  // If nethandler is already locked by the current thread
  if (nh->mutex->thread_holding == t) {
    // Determine whether vio is read vio or writevio
    // according to triggered, put into the ready_list
    if (vio == &read.vio) {
      ep.modify(EVENTIO_READ);
      ep.refresh(EVENTIO_READ);
      if (read.triggered)
        nh->read_ready_list.in_or_enqueue(this);
      else
        nh->read_ready_list.remove(this);
    } else {
      ep.modify(EVENTIO_WRITE);
      ep.refresh(EVENTIO_WRITE);
      if (write.triggered)
        nh->write_ready_list.in_or_enqueue(this);
      else
        nh->write_ready_list.remove(this);
    }
  } else {
    // may not be locked, may be locked by other threads
    // So try to lock the nethandler in the current thread
    MUTEX_TRY_LOCK(lock, nh->mutex, t);
    if (!lock.is_locked()) {
      // Locked failed, put into enable_list and wait for the next time nethandler is processed by eventsystem callback
      if (vio == &read.vio) {
        if (!read.in_enabled_list) {
          read.in_enabled_list = 1;
          nh->read_enable_list.push(this);
        }
      } else {
        if (!write.in_enabled_list) {
          write.in_enabled_list = 1;
          nh->write_enable_list.push(this);
        }
      }
      // If there is a Signal Event that needs to be processed as soon as possible, call the signal_hook notification to the corresponding thread.
      if (nh->trigger_event && nh->trigger_event->ethread->signal_hook)
        nh->trigger_event->ethread->signal_hook(nh->trigger_event->ethread);
    } else {
      // Locked successfully
      // According to triggered, put into ready_list
      if (vio == &read.vio) {
        ep.modify(EVENTIO_READ);
        ep.refresh(EVENTIO_READ);
        if (read.triggered)
          nh->read_ready_list.in_or_enqueue(this);
        else
          nh->read_ready_list.remove(this);
      } else {
        ep.modify(EVENTIO_WRITE);
        ep.refresh(EVENTIO_WRITE);
        if (write.triggered)
          nh->write_ready_list.in_or_enqueue(this);
        else
          nh->write_ready_list.remove(this);
      }
    }
  }
}

void
UnixNetVConnection::reenable_re(VIO *vio)
{
  // Compared to reenable, there is less judgment that enabled is true
  // The thread member is defined in the base class NetVConnection and is used to point to the EThread that manages the vc.
  // Initialize thread to e->ethread in NetAccept or acceptEvent
  // If the vc has not been delivered to an EThread for management, the description has not been managed by the NetHandler and therefore cannot be activated.
  if (!thread)
    return;

  // must be locked to vio before executing reenable_re (the thread that locks the vio must be the thread that called this method)
  EThread *t = vio->mutex->thread_holding;
  ink_assert(t == this_ethread());
  // Compared with reenable, there is no judgment vc->closed
  // If nethandler is already locked by the current thread
  if (nh->mutex->thread_holding == t) {
    // Set enabled to true, while resetting inactivity timeout
    set_enabled(vio);
    // Determine whether vio is read vio or writevio
    // If the triggered is true, the read and write operations are triggered directly.
    // Otherwise removed from ready_list
    if (vio == &read.vio) {
      ep.modify(EVENTIO_READ);
      ep.refresh(EVENTIO_READ);
      if (read.triggered)
        net_read_io(nh, t);
      else
        nh->read_ready_list.remove(this);
    } else {
      ep.modify(EVENTIO_WRITE);
      ep.refresh(EVENTIO_WRITE);
      if (write.triggered)
        write_to_net(nh, this, t);
      else
        nh->write_ready_list.remove(this);
    }
  } else {
    // may not be locked, may be locked by other threads
    // call reenable to complete
    reenable (saw);
  }
}
```

Compare reenable with reenable_re:

  - reenable_re can immediately trigger vio read and write operations, can achieve synchronous operation, and can repeatedly trigger read and write operations in the enabled state
  - reenable just put vc into enable_list or ready_list, you need to wait for NetHandler to process in the next traversal

So when you need to trigger read and write operations immediately, call reenable_re

  - But there may still be asynchronous operations where NetHandler has been locked by other threads.
  - After calling reenable_re, there may be re-entry of the upper state machine (I think this is the true meaning of the re suffix).

scenes to be used:

  - When we want a VIO operation to complete immediately
  - In the READY event, call reenable_re to allow the operation to continue immediately
  - Until the state machine receives the COMPLETE event, then returns layer by layer
  - But there will be asynchronous situations in the middle, and reenable_re just guarantees as much synchronization as possible.

Since this method can cause potential blocking problems, there are only a few places in the ATS that use the method reenable_re.

Summary of the lock:

  - First lock the VIO before calling reenable. Usually the lock of VIO is the lock of the upper state machine, basically it will be locked.
  - Lock the NetHandler when the reenable is executed, because:
    - In the A thread may be reenable a vc managed in the B thread
    - At this time, NetHandler may be calling processor_enabled_list to batch process all vc managed in B thread.
    - This will result in a competitive condition for resource access.

### Close & Free (close & free)

All NetVC and its inheritance classes will call the close_UnixNetVConnection function, which is not a member function of a NetVC class.

Therefore, this function is called for the shutdown of SSLNetVConnection, but since the free method is a member function, ```vc->free(t)``` executes the release operation corresponding to the NetVConnection inheritance type.

```
//
// Function used to close a UnixNetVConnection and free the vc
//
// Pass in two variables, one is the vc you want to close, the other is the current EThread
void
close_UnixNetVConnection(UnixNetVConnection *vc, EThread *t)
{
  NetHandler * nh = vc-> nh;
  
  // Cancel the sending state machine for out-of-band data
  vc-> cancel_OOB ();
  // Let epoll_wait stop monitoring this vc socket fd
  vc->ep.stop();
  // close socket fd
  vc->con.close();

  // There is no asynchronous processing strategy, it must be guaranteed that the thread that manages this vc initiates the call.
  // In fact, here t == this_ethread(), you can check the place where the close_UnixNetVConnection is called in the ATS code to confirm
  ink_release_assert(vc->thread == t);

  // clean up the timeout counter
  vc->next_inactivity_timeout_at = 0;
  vc->next_activity_timeout_at = 0;
  vc->inactivity_timeout_in = 0;
  vc->active_timeout_in = 0;

  // The previous version did not determine if nh is empty, there is a bug
  // When nh is not empty, you must remove vc from several queues.
  if (nh) {
    // four local queues
    nh->open_list.remove(vc);
    nh->cop_list.remove(vc);
    nh->read_ready_list.remove(vc);
    nh->write_ready_list.remove(vc);
    
    // two enable_list are atomic queues
    // But the operation of in_enable_list is not synchronized with the atomic operation. I think there is a problem here! ! !
    if (vc->read.in_enabled_list) {
      nh->read_enable_list.remove(vc);
      vc->read.in_enabled_list = 0;
    }
    if (vc->write.in_enabled_list) {
      nh->write_enable_list.remove(vc);
      vc->write.in_enabled_list = 0;
    }
    
    // two are only used for the InactivityCop local queue
    vc->remove_from_keep_alive_queue();
    vc->remove_from_active_queue();
  }
  
  // Finally call vc free method
  vc->free(t);
}
```

After vc is closed, resources need to be reclaimed. Since vc's memory resources are allocated through the allocate_vc function, the memory is also returned to the memory pool during reclaim.

Each type of NetVC inheritance class will have a matching NetProcessor inheritance class to provide the allocate_vc function and its own free function.

```
void
UnixNetVConnection::free(EThread *t)
{
  // Only the current thread can release the vc resource
  // Here t == thread, which is the same as close_UnixNetVConnection.
  ink_release_assert(t == this_ethread());
  NET_SUM_GLOBAL_DYN_STAT(net_connections_currently_open_stat, -1);
  
  // clear variables for reuse
  // release vc mutex
  this->mutex.clear();
  // Release the mutex of the upper state machine
  action_.mutex.clear();
  got_remote_addr = 0;
  got_local_addr = 0;
  attributes = 0;
  // Release vio mutex, may be equal to the mutex of the upper state machine, there is a judgment, will not repeatedly release the memory
  read.vio.mutex.clear();
  write.vio.mutex.clear();
  // Reset half-closed state
  flags = 0;
  // Reset callback function
  SET_CONTINUATION_HANDLER(this, (NetVConnHandler)&UnixNetVConnection::startEvent);
  // NetHandler is empty
  nh = NULL;
  // Reset NetState 
  read.triggered = 0;
  write.triggered = 0;
  read.enabled = 0;
  write.enabled = 0;
  // Reset VIO
  read.vio._cont = NULL;
  write.vio._cont = NULL;
  read.vio.vc_server = NULL;
  write.vio.vc_server = NULL;
  // reset options
  options.reset();
  // reset the close state
  closed = 0;
  // Must not be in ready_link and enable_link, removed in close_UnixNetVConnection
  ink_assert(!read.ready_link.prev && !read.ready_link.next);
  ink_assert(!read.enable_link.next);
  ink_assert(!write.ready_link.prev && !write.ready_link.next);
  ink_assert(!write.enable_link.next);
  ink_assert(!link.next && !link.prev);
  // socket fd has been closed, calling con.close() in close_UnixNetVConnection to close it
  ink_assert(con.fd == NO_FD);
  ink_assert(t == this_ethread());

  // Determine whether the allocate_vc method allocates memory for vc, is it global memory allocation or thread local allocation
  // Select the corresponding method to return memory resources
  if (from_accept_thread) {
    netVCAllocator.free(this);
  } else {
    THREAD_FREE(this, netVCAllocator, t);
  }
}
```

Since close_UnixNetVConnection and free are not reentrant, in many places, we have seen the use of vc's reentrant counter to determine the reentrant.

For example, in do_io_close:

```
void
UnixNetVConnection::do_io_close(int alerrno /* = -1 */)
{
  disable_read(this);
  disable_write(this);
  read.vio.buffer.clear();
  read.vio.nbytes = 0;
  read.vio.op = VIO::NONE;
  read.vio._cont = NULL;
  write.vio.buffer.clear();
  write.vio.nbytes = 0;
  write.vio.op = VIO :: NONE;
  write.vio._cont = NULL;

  EThread *t = this_ethread();
  // By re-entry counter to determine whether you can directly call close_UnixNetVConnection
  bool close_inline = !recursion && (!nh || nh->mutex->thread_holding == t);

  INK_WRITE_MEMORY_BARRIER;
  if (alerrno && alerrno != -1)
    this->lerrno = alerrno;
  if (alerrno == -1)
    closed = 1;
  else
    closed = -1;

  if (close_inline)
    close_UnixNetVConnection(this, t);
  // There is no operation here for the case where vc cannot be closed immediately, and there is no rescheduling to delay closing vc.
  // Then when is the close_UnixNetVConnection called to close vc? The answer is: will be processed in InactivityCop
}
```

### Memo on the atomicity of the in_enable_list operation

Directly determine in_enable_list and set 0 in close_UnixNetVConnection

  - If the NetHandler is not locked, then it cannot be synchronized with the atomic operation.
  - So, is there a lock on NetHandler? The answer is locked!
  - But where is it locked?

Hint: Why is NetAccept's mutex set to ```get_NetHandler(t)->mutex``` in init_accept_per_thread?

## References

- [P_UnixNetState.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetState.h)
- [P_UnixNet.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
- [P_UnixNetVConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetVConnection.h)
- [UnixNetVConnection.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetVConnection.cc)
