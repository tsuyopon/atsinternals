# Basic component: VIO

VIO is used to describe an ongoing IO operation.

It is returned by the do_io_read and do_io_write methods in VConnections. With VIO, the state machine monitors the progress of the operation and activates the operation when the data arrives.

## definition

```
class VIO 
{
public:
  ~VIO() {}

  /** Interface for the VConnection that owns this handle. */
  Continuation *get_continuation();
  void set_continuation(Continuation *cont);

  void done();

  int64_t ntodo();

  /////////////////////
  // buffer settings //
  /////////////////////
  void set_writer(MIOBuffer *writer);
  void set_reader(IOBufferReader *reader);
  MIOBuffer *get_writer();
  IOBufferReader *get_reader();

  inkcoreapi void reenable();

  inkcoreapi void reenable_re();

  VIO(int aop);
  VIO();

public:
  Continuation *_cont;

  int64_t nbytes;

  int64_t ndone;

  int op;

  MIOBufferAccessor buffer;

  VConnection *vc_server;

  Ptr<ProxyMutex> mutex;
};

```

## Method

get_continuation()

  - Get the Continuation associated with this VIO
  - Return member: _cont

set_continuation(Continuation *cont)

  - Set the Continuation (usually state machine SM) associated with this VIO, while inheriting Cont's mutex.
  - Notify the VConnection associated with the VIO by calling: vc_server->set_continuation(this, cont).
     - This VConnection::set_continuation() method, with the support of NT being canceled when Yahoo was open sourced, was also discarded.
  - After setting, when an event occurs on this VIO, it will call back the handler of the new Cont and pass the Event.
  - If cont==NULL, then the VIO mutex will be cleared at the same time, and will be passed to vc_server.

set_writer(MIOBuffer *writer)

  - Call: buffer.writer_for(writer);

set_reader(IOBufferReader *reader)

  - Call: buffer.reader_for(reader);

get_writer()

  - Returns: buffer.writer();

get_reader()

  - Returns: buffer.reader();

done()

  - Set VIO to complete, this VIO operation will enter the disable state
  - Set nbytes = ndone + buffer.reader()->read_avail()
  - If buffer.reader() is empty, set nbytes = ndone
  - EVENT_READ_COMPLETE or EVENT_WRITE_COMPLETE event will not be triggered**

ntodo()

  - See how much work is still not completed in this VIO
  - Return nbytes - ndone

reenable() & reenable_re()

  - Call: vc_server->reenable(this) or vc_server->reenable_re(this)
  - The state machine SM uses it to activate an I/O operation.
  - Tell VConnection that there is more data waiting to be processed, but first try to continue the previous operation
  - When subsequent operations are not possible, the I/O operation will sleep and wait for it to wake up again
     - For read operations, this means that the buffer is full.
     - For write operations, this means that the buffer is empty.
  - When the wakeup is still not possible after the wakeup, the wakeup will be ignored and no new Event will be generated.
     - This means that the next time you wake up, the incoming Event is still the last set Event.
  - Avoid unnecessary wakeups, which waste CPU resources and reduce system throughput
  - For the difference between reenable and reenable_re, you need to refer to the definition in the inherited class of VConnection.

## Member variables

_cont

  - This pointer is used to hold the call to this VConnection and pass the Continuation of the Event.
  - Usually this Cont is a state machine SM

nbytes

  - Indicates the total number of bytes that need to be processed

ndone

  - Indicates the number of bytes that have been processed
  - A lock must be taken when manipulating this value

op

  - Indicates the type of operation of this VIO

```
  enum {
    NONE = 0,
    READ,
    WRITE,
    CLOSE,
    ABORT,
    SHUTDOWN_READ,
    SHUTDOWN_WRITE,
    SHUTDOWN_READWRITE,
    SEEK,
    PREAD,
    PWRITE,
    STAT,
  };
```

buffer

  - If op is a write operation, it contains a pointer to the IOBufferReader
  - If op is a read operation, include a pointer to MIOBuffer

vc_server

  - This refers to a reverse pointer back to VIO for VConnection, which is used inside the reenable.

mutex

  - Reference to the mutex of the state machine
  - When VIO is in the disable state, it may point to the mutex of VConnection. (Example: do_io_read(NULL, 0, NULL), incoming cont==NULL)
  - Even if the state machine has closed VConnection and recycled, Processor can safely lock the operation

## Understanding VIO

When the state machine SM needs to initiate an IO operation, it informs IOCore by creating a VIO, and the IOCore calls (notifies) the state machine SM through the VIO after performing the physical IO operation.

So VIO contains three elements: BufferAccessor, VConnection, Cont(SM)

The data flow flows between the BufferAccessor and the VConnection, and the message flow is passed between IOCore and Cont(SM).

It should be noted that the BufferAccessor in VIO is the operator pointing to MIOBuffer, and the MIOBuffer is created by Cont(SM).

In addition, the IOCore callback Cont (SM) is carried out by the _cont pointer stored in the VIO, not through the Cont in the vc_server.

```
+-------+                 +--------------------+    +-----------------------------------------+
|       |  +-----------+  |                    |    |  +-----------+                          |
|       |  |           |  |                    |    |  |           |                          |
|       ===> vc_server ==========> read  ==============> MIOBuffer |                          |
|       |  |           |  |                    |    |  |           |                          |
|       |  +-----------+  |                    |    |  +-----------+                          |
|       |                 |  vc->read.vio._cont----->handleEvent(READ_READY, &vc->read.vio)   |
|       |                 |                    |    |                                         |
|  NIC  |                 |       IOCore       |    |             VIO._Cont (SM)              |
|       |  +-----------+  |                    |    |  +-----------+                          |
|       |  |           |  |                    |    |  |           |                          |
|       <=== vc_server <========== write <============== MIOBuffer |                          |
|       |  |           |  |                    |    |  |           |                          |
|       |  +-----------+  |                    |    |  +-----------+                          |
|       |                 | vc->write.vio._cont----->handleEvent(WRITE_READY, &vc->write.vio) |
|       |                 |                    |    |                                         |
+-------+                 +--------------------+    +-----------------------------------------+

For TCP, IOCore here refers to NetHandler and UnixNetVConnection.
```

For example, when the state machine (SM) wants to receive 1 Mbyte of data and forward it to the client, we can do so in the following way:

- Create a MIOBuffer to hold temporarily received data
- Create a read VIO on SourceVC via do_io_read, set the read length to 1M bytes, and pass in the MIOBuffer to receive temporary data.
- IOCore will generate data into VIO when it receives data on SourceVC (temporarily stored in MIOBuffer)
   - Then call SM->handler(EVENT_READ_READY, SourceVIO)
   - In the handler, you can consume SourceVIO (read the data temporarily stored in MIOBuffer) and get some data read this time.
- There is a counter in VIO, when the total production (reading) data reaches 1Mbyte
   - Then IOCore calls SM->handler(EVENT_READ_COMPLETE, SourceVIO)
   - In the handler, you need to first consume (read) SourceVIO, then turn off SourceVIO.
- This completes the process of receiving 1 Mbyte of data.

It can be seen that ATS describes a task containing several IO operations to IOCore through VIO. The MIOBuffer for VIO operation can be small, only need to save the data required for IO operation, and then SM treats VIO and MIOBuffer.

Alternatively, you can think of VIO as a PIPE, one end is the underlying IO device, and one end is MIOBuffer. A VIO should be selected before it is created. It cannot be modified after creation. For example, reading VIO is from the underlying IO device (vc_server). Read data to MIOBuffer; write VIO, which is to write data from MIOBuffer to the underlying IO device (vc_server).

```
+---------------------------------------------------------------+
|                 op=READ, nbytes=X, ndone=Y                    |
|                                                               |
|     VConnection *vc_server ----> MIOBufferAccessor buffer     |
|                                                               |
|  if( ndone < nbytes )                                         |
|      Continuation *_cont->handler(EVENT_READ_READY, thisVIO)  |
|  if( ndone == nbytes )                                        |
|                 _cont->handler(EVENT_READ_COMPLETE, thisVIO)  |
+---------------------------------------------------------------+

+---------------------------------------------------------------+
|               op=WRITE, nbytes=X, ndone=Y                     |
|                                                               |
|     VConnection *vc_server <---- MIOBufferAccessor buffer     |
|                                                               |
|  if( ndone < nbytes )                                         |
|      Continuation *_cont->handler(EVENT_WRITE_READY, thisVIO) |
|  if( ndone == nbytes )                                        |
|                 _cont->handler(EVENT_WRITE_COMPLETE, thisVIO) |
|  else if( wbeEvent )                                          |
|                 _cont->handler(wbeEvent, thisVIO)             |
+---------------------------------------------------------------+
In some VC implementations, the wbeEvent event is fired again when the empty buffer is written, usually EVENT_WRITE_READY
```

VIO operations contain multiple types of operations that can be determined by the 'op' member variable. The optional values are as follows:

  - READ indicates read operation
  - WRITE indicates write operation
  - CLOSE indicates that the request to close VConnection
  - ABORTION
  - SHUTDOWN_READ
  - SHUTDOWN_WRITE
  - SHUTDOWN_READWRITE
  - SEEK
  - PREAD
  - PWRITE
  - STAT

## Reference material
- [I_VIO.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VIO.h)
- [I_VConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VConnection.h)
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)
