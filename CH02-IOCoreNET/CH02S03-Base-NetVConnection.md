# Basic component: NetVConnection



The VConnection abstraction in ATS is very interesting and almost corresponds to the OSI model:

| OSI Layer |       VC Class     | Comments                                                      |
|:---------:|-------------------:|:--------------------------------------------------------------|
|     6     |  SSLNetVConnection | Provides SSL session protocol support and provides members to save SSL sessions.              |
|     4     | UnixNetVConnection | Provides member to save socket fd, establishes data flow logic between socket fd and VIO |
|     3     |     NetVConnection | Have members who saved IP information but did not save members of socket fd             |
|     2     |        VConnection | Base class |

NetVConnection is a very basic, very old abstract class in the IOCoreNET part.

Essentially, NetVConnection and its derived classes are state machines for connecting an FD to a VIO (MIOBuffer).

There are many state machines in the ATS called VCs, such as: TransformVC, they inherit from VConnection, they connect objects different from NetVConnection.

## definition

NetVConnection inherits from VConnection and extends it. The following analysis only describes the extended part. For the VConnection part, please refer to the introduction of VConnection.

```
class NetVConnection : public VConnection
{
public:
  virtual VIO *do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf) = 0;
  virtual VIO *do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner = false) = 0;
  virtual void do_io_close(int lerrno = -1) = 0;
  virtual void do_io_shutdown(ShutdownHowTo_t howto) = 0;

  // Send out of band data
  // Send the data in the buffer buf of length len out-of-band.
  // Send successfully, passing the VC_EVENT_OOB_COMPLETE event when calling cont
  // The other party closes, passing the VC_EVENT_EOS event when calling cont
  // cont callback must be reentrant
  // Each VC can only have one send_OOB operation at the same time
  virtual Action *send_OOB(Continuation *cont, char *buf, int len);

  // Cancel the sending of out-of-band data
  // Cancel the out-of-band data send operation that was previously called by send_OOB, but some data may have been sent.
  // After execution, there will be no more callbacks to cont, and no more access to the Action object returned by previous send_OOB.
  // When the previous send_OOB operation is not completed and a new send_OOB operation is required, the previous operation of cancel_OOB must be performed.
  virtual void cancel_OOB();

  ////////////////////////////////////////////////////////////
  // Connection timeout setting
  //         active_timeout is used for the entire life cycle of the connection
  //     inactivity_timeout is used for the life cycle after the last read and write operation
  // These methods can be called repeatedly to reset the timeout statistics time.
  // Since these methods are not thread-safe, you can only call these functions when "callback"
  
  /*
    Definition of "timeout":

    If a timeout occurs, the state machine (SM) associated with the NetVC reader is first called back to ensure that data has been read on NetVC and the reader has not been closed.
    If both conditions are not met, NetProcessor attempts to call back the state machine (SM) associated with the write side.
    PS: The read end is turned off (read half closed, read 0 bytes). If there is no data to be written, it usually means that the connection is closed by the peer.

    Callback read state machine,
         If EVENT_DONE is returned:
             Then it will not call back the write state machine;
         If the return is not EVENT_DONE, and the write state machine and the read end are not the same state machine (compared to the pointer of the callback function):
             NetProcessor will attempt to call back the write state machine.

    Before calling back the write end, make sure the data has been written and the write end is not closed (write half closed).

    The state machine receives the TIMEOUT event, just a notification that the time has elapsed, but NetVC is still available.

    Unless the timer is reset by set_active_timeout() or set_inactivity_timeout(), subsequent timeout signals will no longer be triggered.

    The InactivityCop state machine is used to traverse the Keep alive queue and the Active queue in the ATS to detect the timeout timer and call back the state machine.
    For a more detailed history of the InactivityCop state machine, see: https://issues.apache.org/jira/browse/TS-1405
  */
  
  /**

     Notify the state machine (SM) after the specified time

     Set a time amount (nanoseconds/nanoseconds, in parts per billion), after which the callback is associated with the state machine (SM) associated with this VC and pass the VC_EVENT_ACTIVE_TIMEOUT event.
    
    The timeout is value is ignored if neither the read side nor the write side of the connection is currently active. 
    
    If the current connection is not in the active state (no read or write operations), then this value will be ignored.
    PS: In fact, it is the priority to determine the Inactivity Timeout. When the Inactivity Timeout occurs, the Active Timeout event will not be called back.
    
    Each time this function is called, the timeout is re-timed.
    This way, the state machine (SM) can be used to handle links that do not close (but have communication) for a long time.
  */
  virtual void set_active_timeout(ink_hrtime timeout_in) = 0;

  /**
    Notify the state machine (SM) if the initiated IO operation is not completed at the specified time
    
    If the set amount of time (nanoseconds/nanoseconds, in parts per billionth of a second) is reached after the last state machine has operated NetVC, NetVC is still idle on both the read and write ends, and will call back with this. NetVC associated state machine
(SM) and pass the VC_EVENT_INACTIVITY_TIMEOUT event.
    
    The timer will be reset:
         Read operation
         Write operation
         Call this function: set_inactivity_timeout()
    
    The timeout is value is ignored if neither the read side nor the write side of the connection is currently active. See section on timeout semantics above.
   */
  virtual void set_inactivity_timeout(ink_hrtime timeout_in) = 0;

  /**
    Clear Active Timeout
    
    There will be no more Active Timeout events sent until set_active_timeout() is reset.
  */
  virtual void cancel_active_timeout() = 0;

  /**
    Clear Inactivity Timeout

    There will be no more Active Timeout events sent until set_inactivity_timeout() is called.
  */
  virtual void cancel_inactivity_timeout() = 0;

  // The following four methods are the encapsulation of the corresponding methods in NetHandler, in fact, the method of calling NetHandler through member nh
  // Add NetVC to the keep alive queue
  virtual void add_to_keep_alive_queue() = 0;
  // Remove NetVC from the keep alive queue
  virtual void remove_from_keep_alive_queue() = 0;
  // Add NetVC to the active queue
  virtual bool add_to_active_queue() = 0;
  // This seems to miss the declaration of the following method, because the method is defined in UnixNetVConnection
  // virtual void remove_from_active_queue() = 0;

  // Returns the value of the current active_timeout (nanoseconds)
  virtual ink_hrtime get_active_timeout() = 0;

  // Returns the value of the current inactivity_timeout (nanoseconds)
  virtual ink_hrtime get_inactivity_timeout() = 0;

  // Special mechanism: If the write operation causes the buffer to be empty (no data can be written in MIOBuffer), use the specified event callback state machine
  /** Force an @a event if a write operation empties the write buffer.

     This mechanism is abbreviated as: WBE
     In IOCoreNet processing, if we set this special mechanism, then:
         In the next process of writing data from MIOBuffer to socket fd,
         If the data currently available for write operations in MIOBuffer is written:
             Will use the specified event callback state machine

     If the incoming event is 0, it means cancel the operation.

     As with other IO events, the VIO data type is passed during the callback.

     This operation is a single trigger. If you need to repeat the trigger, you need to set it again.

     PS: The callback logic in the write_to_net_io() method of the NetHandler callback is as follows:
             If the MIOBuffer has free space, first send a WRITE READY callback state machine to fill the MIOBuffer
             Then start writing the data in MIOBuffer to socket fd
             Save WBE to wbe_event, wbe_event = WBE;
             If MIOBuffer is written empty
                 Clear WBE status, WBE = 0;
             If the VIO is completed, send the WRITE COMPLETE callback state machine to end the write operation.
             If WRITE READY has been sent and WBE is set
                 Send the web_event callback state machine, if the callback return value is not EVENT_CONT, end the write operation
             If the WRITE READY callback state machine has not been sent before
                 Send WRITE READY callback state machine, if the return value is not EVENT_CONT, end the write operation
             If the MIOBuffer is empty, disable subsequent write events on the VC and end the write operation.
             Reschedule read and write operations
             
      The event is sent only if otherwise no event would be generated.
   */
  virtual void trapWriteBufferEmpty(int event = VC_EVENT_WRITE_READY);

  // Returns a pointer to the local sockaddr object, supporting both IPv4 and IPv6
  Sockaddr const *get_local_addr();

  / / Return Local ip, only supports IPv4, the old method is not recommended, will be abandoned
  In_addr_t get_local_ip();

  // return Local port
  Uint16_t get_local_port();

  // Returns a pointer to the Remote sockaddr object, supporting both IPv4 and IPv6
  Sockaddr const *get_remote_addr();

  // return remote ip
  In_addr_t get_remote_ip();

  // return remote port
  Uint16_t get_remote_port();

  // saved the structure set by the user
  // In the ATS, the setting information of the main object is separated, and it is applied in many places.
  // This allows you to share the same set of instances when creating a set of objects of the same set to reduce memory usage.
  NetVCOptions options;

  /** Attempt to push any changed options down */
  virtual void apply_options() = 0;

  //
  // Private
  / / The following methods are not recommended to call directly, not declared as private type in order to successfully compile through the code
  //

  // used to get the host address when the transposency of SocksProxy is turned on
  SocksAddrType socks_addr;

  / / Specify the properties of NetVC
  // More used is HttpProxyPort::TRANSPORT_BLIND_TUNNEL
  Unsigned int attributes;

  // Point to EThread holding this NetVC
  EThread *thread;

  /// PRIVATE: The public interface is VIO::reenable()
  virtual void reenable(VIO *vio) = 0;

  /// PRIVATE: The public interface is VIO::reenable()
  virtual void reenable_re(VIO *vio) = 0;

  /// PRIVATE
  virtual ~NetVConnection() {}

  /**
    PRIVATE: instances of NetVConnection cannot be created directly
    by the state machines. The objects are created by NetProcessor
    calls like accept connect_re() etc. The constructor is public
    just to avoid compile errors.

  */
  NetVConnection();

  virtual SOCKET get_socket() = 0;

  /** Set the TCP initial congestion window */
  virtual int set_tcp_init_cwnd(int init_cwnd) = 0;

  /** Set local sock addr struct. */
  virtual void set_local_addr() = 0;

  /** Set remote sock addr struct. */
  virtual void set_remote_addr() = 0;

  // for InkAPI
  bool
  get_is_internal_request() const
  {
    return is_internal_request;
  }

  void
  set_is_internal_request(bool val = false)
  {
    is_internal_request = val;
  }

  // Transparent Proxy (TPROXY) support
  // get the transparency status
  bool
  get_is_transparent() const
  {
    return is_transparent;
  }
  // Set the transparency state
  void
  set_is_transparent(bool state = true)
  {
    is_transparent = state;
  }

private:
  NetVConnection(const NetVConnection &);
  NetVConnection &operator=(const NetVConnection &);

protected:
  IpEndpoint local_addr;
  IpEndpoint remote_addr;

  bool got_local_addr;
  bool got_remote_addr;

  bool is_internal_request;
  /// Set if this connection is transparent.
  bool is_transparent;
  /// Set if the next write IO that empties the write buffer should generate an event.
  int write_buffer_empty_event;
};

inline NetVConnection::NetVConnection()
  : VConnection(NULL), attributes(0), thread(NULL), got_local_addr(0), got_remote_addr(0), is_internal_request(false),
    is_transparent(false), write_buffer_empty_event(0)
{
  ink_zero(local_addr);
  ink_zero(remote_addr);
}

inline void
NetVConnection::trapWriteBufferEmpty(int event)
{
  write_buffer_empty_event = event;
}
```

## Reference material
- [I_NetVConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/net/I_NetVConnection.h)
- [I_VIO.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VIO.h)
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)

# Basic component: VConnection

VConnection is the base class for all Connection classes that provide IO functionality. It cannot implement functions by itself. It must implement specific functions through derived classes.

The VConnection class is an abstract description of a one-way or two-way data pipeline returned by the Processor.

In a sense, it is similar to the function of a file descriptor.

Basically, VConnection is the base class that defines the Stream IO processing method.

It is also a Continuation that is called back by the Processor.

VConnection defines a virtual pipeline that can be connected in series, which allows ATS to form streaming data communication between the network, state machine and disk. It is a key part of the upper layer communication mechanism.

```
                                          +------ UserSM ------+
      UpperSM Area                       /       (HTTPSM)       \
                                        /                        \
                                       /                          \
*** CS_FD ********************** CS_VIO ************************** SS_VIO ********************** SS_FD ***
          \                     /                                        \                     /
           +------ CS_VC ------+          EventSystem & IO Core           +------ SS_VC ------+

 CS_FD = Client Side File Descriptor
 SS_FD = Server Side File Descriptor
 CS_VC = Client Side VConnection
 SS_VC = Server Side VConnection
CS_VIO = Client Side VIO
SS_VIO = Server Side VIO
UserSM = User Defined SM ( HTTPSM )
****** = Separator Line
```

When the FD is in a readable or writable state, the VC is called back, the read data on the FD is produced into the VIO, or the data in the VIO is taken out and consumed and written into the FD.

When the data in the VIO changes, call the UserSM and tell the reason for the change.

Therefore, VC can be understood as a state machine responsible for moving data between FD and VIO.

## definition

VConnection inherits from Continuation, so VConnection is a continuation first, it has Cont Handler and Mutex.

```

class VConnection : public Continuation
{
public:
  virtual ~VConnection();

  virtual VIO *do_io_read(Continuation *c = NULL, int64_t nbytes = INT64_MAX, MIOBuffer *buf = 0) = 0;

  virtual VIO *do_io_write(Continuation *c = NULL, int64_t nbytes = INT64_MAX, IOBufferReader *buf = 0, bool owner = false) = 0;

  virtual void do_io_close(int lerrno = -1) = 0;

  virtual void do_io_shutdown(ShutdownHowTo_t howto) = 0;

  VConnection(ProxyMutex *aMutex);

// Private
  // Set continuation on a given vio. The public interface
  // is through VIO::set_continuation()
  virtual void set_continuation(VIO *vio, Continuation *cont);

  // Reenable a given vio.  The public interface is through VIO::reenable
  virtual void reenable(VIO *vio);
  virtual void reenable_re(VIO *vio);

  virtual bool
  get_data(int id, void *data)

  virtual bool
  set_data(int id, void *data)

public:
  int lerrno;
};

```

### Universal Event Code

VC_EVENT_INACTIVITY_TIMEOUT

  - Indicates that within a certain period of time: No data is read from the connection / No data is written to the connection
  - For read operations, the buffer may be empty, and the buffer may be full for write operations.

VC_EVENT_ACTIVITY_TIMEOUT

  - read/write operation timed out

VC_EVENT_ERROR

  - An error occurred during the read/write process


## method

Since VConnection is just a base class, the following methods need to be redefined in their inheritance classes.

However, in order to ensure the unification of the definition, in the inheritance class of VConnection, we must implement these methods according to the following basic conventions.

### do_io_read

Ready to receive data from VConnection

  - The state machine (SM) calls it to read data from VConnection.
  - When the processor implements this read function, it first acquires the lock, then puts the new data into buf, calls Continuation, and then releases the lock.
  - For example: Let the state machine handle the Data Transfer Protocol (NNTP) with special characters as the end of the transaction.

Possible Event Code (VConn may use these values as Event Code when the state machine calls Connection)

  - VC_EVENT_READ_READY
    - The data has been added to the buffer, or the buffer is full
  - VC_EVENT_READ_COMPLETE
    - nbytes bytes of data have been read into the buffer
  - VC_EVENT_EOS
    - The reader of Stream closes the connection

Parameters (Continuation *c, int64_t nbytes, MIOBuffer *buf)

  - c
    - After the read operation is completed, the Continuation will be called back, and the Event Code will be passed.
  - nbytes
    - The number of bytes you want to read, if you are not sure how much to read, you must set it to INT64_MAX
  - buf
    - The data read will be placed here

return value

  - VIO type representing scheduled IO operations

note

  - When c=NULL, nbytes=0, buf=NULL means canceling the do_io_read operation set before.
  - When no data is readable, or buf is full, VIO will be disabled and need to call reenable to continue reading data.

### do_io_write

Preparing to write data to VConnection

  - The state machine (SM) calls it to write data to the VConnection.
  - When the processor implements this read function, it first acquires the lock, then writes the data from the buf, calls the Continuation, and then releases the lock.

Possible Event Code (VConn may use these values as Event Code when the state machine calls Continuation)

  - VC_EVENT_WRITE_READY
    - The data from the buffer has been written, or the buffer is empty
  - VC_EVENT_WRITE_COMPLETE
    - nbytes of bytes from the buffer have been written

Parameters (Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner)

  - c
    - After the read operation is completed, the Continuation will be called back, and the Event Code will be passed.
  - nbytes
    - The number of bytes you want to write, if you are not sure how much to write, you must set it to INT64_MAX
  - buf
    - the data source, which will read the data from here and then write
  - owner
    - Not used, default false, if set to true, may cause assert??

return value

  - VIO type representing scheduled IO operations

note

  - When c=NULL, nbytes=0, buf=NULL means cancel the do_io_write operation set before.
  - When writing data fails, or buf is empty, VIO will be disabled, and need to call reenable to continue writing data.

### set_continuation

To set Continuation for a given VIO, this method should normally be called by VIO::set_continuation().

Parameters (VIO *vio, Continuation *cont)

   - vio
     - VIO will be set up for Continuation
   - cont
     - Continuation to be linked to VIO

### reenable & reenable_re

To activate the specified VIO, this method should normally be called by VIO::reenable().

   - Let the previously set VIO resume running, EventSystem will call the processor.

Parameter (VIO *vio)

   - vio
     - Prepare the activated VIO

### do_io_close

Indicates that this VConnection is no longer needed.

   - When the state machine uses a VConnection, this function must be called to indicate that it can be recycled.
   - after calling close
     - VConnection and the underlying Processor can no longer send any events associated with this VConnection to the state machine.
     - The state machine is also unable to access the VConnection and the VIOs obtained/returned from it.

Parameter (int lerrno = -1)

   - lerrno
     - Used to indicate if this is a normal shutdown or an abnormal shutdown
     - The difference between the two closures is determined by the underlying VConnection type
     - Normal shutdown, lerrno = -1, set vc->closed = 1
     - Abnormally closed, lerrno != -1, set vc->.closed = -1, vc->lerrno = lerrno
     - In most cases, ATS does not distinguish between normal shutdown and abnormal shutdown. It seems that only the difference is treated in ClusterVC.

### do_io_shutdown

Terminate one-way or two-way transmission of VConnection

   - After calling this method, I/O operations in the relevant direction will be disabled
   - Processor can't send any associated EVENT (even Timeout Event) to the state machine
   - The state machine cannot use VIO from the Shutdown direction
   - Even if the two-way transfer is Shutdown, when the state machine needs to reclaim the VConnection, the recycle operation must still be completed by calling do_io_close.

Parameter (ShutdownHowTo_t howto)

   - howto value and meaning
     - IO_SHUTDOWN_READ = 0
       - Close the reader, indicating that the VConn should no longer generate a Read Event
     - IO_SHUTDOWN_WRITE = 1
       - Close the write end, indicating that the VConn should not generate a Write Event
     - IO_SHUTDOWN_READWRITE = 2
       - Bidirectional shutdown, indicating that the VConn should no longer generate Read and Write Event

### get_data & set_data

This function is used in state machines to transfer information from/to a VConnection without breaking the abstraction of VConnection

   - Its behavior depends on which type of VConnection is used
   - Access custom data structures in derived classes

Parameters (int id, void *data)

   - id
     - for the enumerated type TSApiDataType
   - data
     - Corresponding data

return value

- Successfully returned True

```
  /** Used in VConnection::get_data(). */
  enum TSApiDataType {
    TS_API_DATA_READ_VIO = VCONNECTION_API_DATA_BASE,
    TS_API_DATA_WRITE_VIO,
    TS_API_DATA_OUTPUT_VC,
    TS_API_DATA_CLOSED,
    TS_API_DATA_LAST ///< Used by other classes to extend the enum values.
  };
```

## Callback for TIMEOUT event

In the ATS code, we can see the two usages of do_io_read and do_io_write :

  - do_io_read(NULL, 0, NULL)
  - do_io_read(Cont, 0, NULL)

The first parameter indicates the Continuation of the callback after reading the data on this NetVC, usually a state machine, the meaning of the last two parameters:

   - read 0 bytes
   - The data receive buffer is NULL

So here it is obvious that you no longer care about the data received on this NetVC in the future.

So, it seems that this state machine will never be called back, so there seems to be no difference between the two methods.

Do an analysis with the code of do_io_read:

```
VIO *
UnixNetVConnection::do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf)
{
  // If c is NULL, then nbytes must be 0 (which is one of the ways we see the read and write do_io calls)
  // However, nbytes is 0, and c can be any value (that is, the two types of closed read and write do_io calls that we can see earlier)
  // nbytes is not 0, c can't be NULL (the length of the data to be read is not 0, naturally it will adjust the state machine back and forth, so c must not be NULL)
  ink_assert(c || 0 == nbytes);
  if (closed) {
    Error("do_io_read invoked on closed vc %p, cont %p, nbytes %" PRId64 ", buf %p", this, c, nbytes, buf);
    return NULL;
  }
  read.vio.op = VIO::READ;
  // Here also judges that c is NULL, obviously it is allowed to set c to NULL
  read.vio.mutex = c ? c->mutex : this->mutex;
  read.vio._cont = c; 
  read.vio.nbytes = nbytes;
  read.vio.ndone = 0; 
  read.vio.vc_server = (VConnection *)this;
  if (buf) {
    read.vio.buffer.writer_for(buf);
    if (!read.enabled)
      read.vio.reenable();
  } else {
    // When buf is NULL, the reading is turned off, which is the same as we have understood before. As long as buf is NULL, the read operation is closed.
    read.vio.buffer.clear();
    disable_read(this);
  }
  return &read.vio;
}
```

According to the above code, ATS is designed to allow these two types of calls to exist. The so-called existence is reasonable, so what is the difference between the two methods?

Analyze the code for UnixNetVConnection::mainEvent :

```
int
UnixNetVConnection::mainEvent(int event, Event *e)
{
...
  // Force the following two values to be NULL and 0, here should be a hack
  // Because there is a code that calculates these two values in detail here.
  *signal_timeout = 0;
  *signal_timeout_at = 0;
  // First put the WRITE vio callback state machine to the local temporary variable
  Writer_cont = write.vio._cont;

  // If the NetVC has been set to off, then the resource is reclaimed and then returned
  if (closed) {
    close_UnixNetVConnection(this, thread);
    return EVENT_DONE;
  }

  // If READ vio is a read operation and there is no half-close read
  If (read.vio.op == VIO::READ && !(f.shutdown & NET_VC_SHUTDOWN_READ)) {
    / / Save the READ vio callback state machine to the local temporary variables
    Reader_cont = read.vio._cont;
    // signal_event is set to VC_EVENT_INACTIVITY_TIMEOUT or VC_EVENT_ACTIVE_TIMEOUT in the previous section
    // callback state machine timeout signal
    If (read_signal_and_update(signal_event, this) == EVENT_DONE)
      // If the NetVC is closed, it will return immediately
      Return EVENT_DONE;
  }

  // If WRITE vio is a write operation and there is no half-close write
  // - !*signal_timeout && !*signal_timeout_at must be true because the above is forcibly set
  // - !closed This should also be true, because the READ vio and WRITE vio mutex are obtained at the beginning of the mainEvent, usually vc->mutex is reused.
  // - reader_cont != write.vio._cont This means that if the previously retrieved READ vio state machine is different from the WRITE vio state machine
  // - writer_cont == write.vio._cont This checks if the WRITE vio callback state machine was modified during the READ vio callback state machine.
  If (!*signal_timeout && !*signal_timeout_at && !closed && write.vio.op == VIO::WRITE && !(f.shutdown & NET_VC_SHUTDOWN_WRITE) &&
      Reader_cont != write.vio._cont && writer_cont == write.vio._cont)
    // If all of the above conditions are passed, then the WRITE vio state machine can be called back.
    If (write_signal_and_update(signal_event, this) == EVENT_DONE)
      // If the NetVC is closed, it will return immediately
      Return EVENT_DONE;
  // Return after all is completed, I feel that I should return EVENT_CONT because NetVC is not closed after all.
  Return EVENT_DONE;
}
```

You can see the callbacks for READ vio and WRITE vio in mainEvent:

   - Both are timeout events
   - Do not judge the status of vio, even if vio has been disabled, it will still generate a callback
   - First callback READ vio state machine
   - If NetVC is not shut down by the READ vio state machine, it will continue to call back the WRITE vio state machine
     - However, the WRITE vio state machine will not repeat the callback if it is the same as the READ vio state machine.

See here, the answer has been drawn:

   - After READ vio and WRITE vio are turned off, if cont is not NULL, the timeout event will still be accepted
   - but the premise must be that the timeout has been set

If the READ vio or WRITE vio is turned off after the timeout period is set, what happens if the error setting cont is NULL?

   - Continue to look at the code for read_signal_and_update

```
static inline int
read_signal_and_update(int event, UnixNetVConnection *vc)
{
  vc->recursion++;
  // If cont is not NULL
  if (vc->read.vio._cont) {
    // Callback state machine
    vc->read.vio._cont->handleEvent(event, &vc->read.vio);
  } else {
    // If cont is NULL
    switch (event) {
    case VC_EVENT_EOS:
    case VC_EVENT_ERROR:
    // For two timeout events
    case VC_EVENT_ACTIVE_TIMEOUT:
    case VC_EVENT_INACTIVITY_TIMEOUT:
      Debug("inactivity_cop", "event %d: null read.vio cont, closing vc %p", event, vc);
      // Just set closed to 1
      vc->closed = 1;
      break;
    default:
      Error("Unexpected event %d for vc %p", event, vc);
      ink_release_assert(0);
      break;
    }
  }
  if (!--vc->recursion && vc->closed) {
    /* BZ  31932 */
    ink_assert(vc->thread == this_ethread());
    // Then close NetVC here
    close_UnixNetVConnection(vc, vc->thread);
    return EVENT_DONE;
  } else {
    return EVENT_CONT;
  }
}
```

As you can see, IOCore's network subsystem sets the default mechanism for this situation, but in this case, the state machine will not receive a timeout event.

## References
- [I_VConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VConnection.h)
- [I_VIO.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VIO.h)
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)
- [UnixNetVConnection.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetVConnection.cc)
