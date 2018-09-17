# Base component: OneWayTunnel

OneWayTunnel is a complete state machine implemented using the API of the IOCore network subsystem. This is a good example of understanding and learning how to use the iocore system for development.

OneWayTunnel implements a simple data transfer function that reads data from one VC and then writes the data to another VC.

## definition

`` `
//////////////////////////////////////////////////////////////////////////////////////////////////// //////////////////////////////////////////////////
//
//      OneWayTunnel
//
//////////////////////////////////////////////////////////////////////////////////////////////////// //////////////////////////////////////////////////

#define TUNNEL_TILL_DONE INT64_MAX

#define ONE_WAY_TUNNEL_CLOSE_ALL NULL

typedef void (*Transform_fn)(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf);

/ **
  A generic state machine that connects two virtual conections. A
  OneWayTunnel is a module that connects two virtual connections, a source
  vc and a target vc, and copies the data between source and target. Once
  the tunnel is started using the init() call, it handles all the events
  from the source and target and optionally calls a continuation back when
  its done. On success it calls back the continuation with VC_EVENT_EOS,
  and with VC_EVENT_ERROR on failure.

  If manipulate_fn is not NULL, then the tunnel acts as a filter,
  processing all data arriving from the source vc by the manipulate_fn
  function, before sending to the target vc. By default, the manipulate_fn
  is set to NULL, yielding the identity function. manipulate_fn takes
  a IOBuffer containing the data to be written into the target virtual
  connection which it may manipulate in any manner it sees fit.
* /
`` `

The above comments are translated as follows:
OneWayTunnel is a general state machine that connects two VConnections. It can be used as a module to connect the source VC and the destination VC, and write the data read by the source VC to the destination VC.
Once the tunnel is started via init(), it will accept all events from the source VC and the destination VC, and can optionally call back the specified state machine after the operation is complete:

- If successful, pass the VC_EVENT_EOS state during callback
- If it fails, pass the VC_EVENT_ERROR state when the callback

You can turn a Tunnel into a filter by setting the pointer to the manual_fn function (not NULL).

- It will call summarize_fn to process all data from the source VC and send it to the destination VC.
- The Tunnel will pass to the manual_fn IOBuffer containing the data to be written to the target VC.
- manipulate_fn can handle the data contained in it in any way.
- However, the default value of manual_fn is usually NULL.

`` `
struct OneWayTunnel : public Continuation {
  //
  //  Public Interface
  //

  //  Copy nbytes from vcSource to vcTarget.  When done, call
  //  aCont back with either VC_EVENT_EOS (on success) or
  //  VC_EVENT_ERROR (on error)
  //

  // Use these to construct/destruct OneWayTunnel objects

  / **
    Allocates a OneWayTunnel object.
    Static method to create a OneWayTunnel object

    @return new OneWayTunnel object.

  * /
  static OneWayTunnel *OneWayTunnel_alloc();

  /** Deallocates a OneWayTunnel object. 
    Static method to release the OneWayTunnel object
  * /
  static void OneWayTunnel_free(OneWayTunnel *);

  // Set TwoWayTunnel to associate two OneWayTunnel
  static void SetupTwoWayTunnel(OneWayTunnel *east, OneWayTunnel *west);
  // Constructor
  OneWayTunnel();
  // Destructor
  virtual ~OneWayTunnel();

  // Use One of the following init functions to start the tunnel.
  // The init function has multiple types. Please select the appropriate init function to start the tunnel according to your needs.
  / **
    The first init() method, which will be described and analyzed in detail below.
    This init function sets up the read (calls do_io_read) and the write
    (calls do_io_write).

    @param vcSource source VConnection. A do_io_read should not have
      been called on the vcSource. The tunnel calls do_io_read on this VC.
    @param vcTarget target VConnection. A do_io_write should not have
      been called on the vcTarget. The tunnel calls do_io_write on this VC.
    @param aCont continuation to call back when the tunnel finishes. If
      not specified, the tunnel deallocates itself without calling back
      anybody. Otherwise, its the callee's responsibility to deallocate
      the tunnel with OneWayTunnel_free.
    @param size_estimate size of the MIOBuffer to create for
      reading/writing to/from the VC's.
    @param aMutex lock that this tunnel will run under. If aCont is
      specified, the Continuation's lock is used instead of aMutex.
    @param nbytes number of bytes to transfer.
    @param asingle_buffer whether the same buffer should be used to read
      from vcSource and write to vcTarget. This should be set to true in
      most cases, unless the data needs be transformed.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.
    @param manipulate_fn if specified, the tunnel calls this function
      with the input and the output buffer, whenever it gets new data
      in the input buffer. This function can transform the data in the
      input buffer
    @param water_mark watermark for the MIOBuffer used for reading.

  * /
  void init(VConnection *vcSource, VConnection *vcTarget, Continuation *aCont = NULL, int size_estimate = 0, // 0 = best guess
            ProxyMutex *aMutex = NULL, int64_t nbytes = TUNNEL_TILL_DONE, bool asingle_buffer = true, bool aclose_source = true,
            bool aclose_target = true, Transform_fn manipulate_fn = NULL, int water_mark = 0);

  / **
    The second init() method, which will be described and analyzed in detail below.
    This init function sets up only the write side. It assumes that the
    read VConnection has already been setup.

    @param vcSource source VConnection. Prior to calling this
      init function, a do_io_read should have been called on this
      VConnection. The tunnel uses the same MIOBuffer and frees
      that buffer when the transfer is done (either successful or
      unsuccessful).
    @param vcTarget target VConnection. A do_io_write should not have
      been called on the vcTarget. The tunnel calls do_io_write on
      this VC.
    @param aCont The Continuation to call back when the tunnel
      finishes. If not specified, the tunnel deallocates itself without
      calling back anybody.
    @param SourceVio VIO of the vcSource.
    @param reader IOBufferReader that reads from the vcSource. This
      reader is provided to vcTarget.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.
  * /
  void init(VConnection *vcSource, VConnection *vcTarget, Continuation *aCont, VIO *SourceVio, IOBufferReader *reader,
            bool aclose_source = true, bool aclose_target = true);

  / **
    The third init() method, which will be described and analyzed in detail below.
    Use this init function if both the read and the write sides have
    already been setup. The tunnel assumes that the read VC and the
    write VC are using the same buffer and frees that buffer
    when the transfer is done (either successful or unsuccessful)
    @param aCont The Continuation to call back when the tunnel finishes. If
    not specified, the tunnel deallocates itself without calling back
    anybody.

    @param SourceVio read VIO of the Source VC.
    @param TargetVio write VIO of the Target VC.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.

    * /
  void init(Continuation *aCont, VIO *SourceVio, VIO *TargetVio, bool aclose_source = true, bool aclose_target = true);

  //
  // Private
  //
  // This method is currently not supported, there is assert in this function
  OneWayTunnel(Continuation *aCont, Transform_fn manipulate_fn = NULL, bool aclose_source = false, bool aclose_target = false);

  // main state handler
  int startEvent(int event, void *data);

  // Transfer data from in_buf to out_buf
  // Since this only supports single buffer, this function doesn't really make any sense.
  virtual void transform(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf);

  /** Result is -1 for any error. */
  / / Close the source VC, release MIOBuffer
  void close_source_vio(int result);

  / / Close the target VC, release MIOBuffer
  virtual void close_target_vio(int result, VIO *vio = ONE_WAY_TUNNEL_CLOSE_ALL);

  // Close the tunnel
  / / Directly release the Tunnel object, or callback to specify the state machine, the state machine releases the Tunnel object
  void connection_closed(int result);

  // Activate both source VC and target VC
  // I don’t see any examples of calls, I don’t know the specific function.
  virtual void reenable_all();

  // true means that only the last VC channel is left in the current tunnel.
  bool last_connection();

  // Save the source VC vio
  VIO * vioSource;
  / / Save the target VC vio
  VIEW * vioTarget;
  Continuation * account;
  / / Used to achieve data transfer between the source buf and the target buf, but currently only supports the single buffer mode, so there is no meaning
  Transform_fn manipulate_fn;
  // The number of channels (VCs) held by the current tunnel, the possible values ​​are: 0, 1, 2
  int n_connections;
  // Error code from the VC with the error
  int lerrno;

  // Whether the source buf and the target buf are the same buf, can only be set to true at present, otherwise an exception will be thrown.
  bool single_buffer;
  // Incoming at initialization, indicating whether to close the source VC and target VC when the tunnel is completed.
  bool close_source;
  bool close_target;
  // true means that the tunnel will not end until the source VC is closed (EOS).
  bool tunnel_till_done;

  /** Non-NULL when this is one side of a two way tunnel. */
  // Introduced in TwowayTunnel
  OneWayTunnel *tunnel_peer;
  bool free_vcs;

private:
  OneWayTunnel(const OneWayTunnel &);
  OneWayTunnel &operator=(const OneWayTunnel &);
};
`` `

## Method

### The first init() method

`` `
void init (VConnection * vcSource, VConnection * vcTarget, 
         Continuation *aCont = NULL, int size_estimate = 0, // 0 = best guess
         ProxyMutex *aMutex = NULL, 
         int64_t nbytes = TUNNEL_TILL_DONE, bool asingle_buffer = true, 
         bool aclose_source = true, bool aclose_target = true, 
         Transform_fn manipulate_fn = NULL, int water_mark = 0);
`` `

The init function will set read on the source VC (call do_io_read), set write on the target VC (call do_io_write), automatically create a MIOBuffer for internal use, and release buf upon completion. The parameters are as follows:

- vcSource
   You
   - Tunnel will call do_io_read on this VC, so if do_io_read is called on this VC before, it may fail or other problems may occur.
- vcTarget
   - Target VC
   - Tunnel will call do_io_write on this VC, so if you have previously called do_io_write on this VC, it may fail or other problems may occur.
- aCont = NULL
   - This Continuation will be called back when the Tunnel is completed.
   - If not specified (NULL), the Tunnel will release itself directly.
   - Otherwise, the callback function should call OneWayTunnel_free() to complete the release of the tunnel.
- size_estimate = 0
   - The length of MIOBuffer, which acts as a buffer for reading and writing data between VCs.
   - When set to 0, it means adaptive, set to default_large_iobuffer_size in init.
- aMutex = NULL
   - The lock that the tunnel is running.
   - If aCont is specified, then aMutex=aCont->mutex.
- nbytes = TUNNEL_TILL_DONE = INT64_MAX
   - The number of bytes transferred.
- asingle_buffer = true
   - Whether the buf used to read data from the source VC is the same as the buf written to the target VC.
   - Most of the same (true), can be set to false unless manual_fn is not NULL
   - When set to false, an additional MIOBuffer will be created in the init method, and data movement will be required in the manual_fn
   - If this value is set to false if manual_fn is not set, an exception will be thrown because the implementation of transfer_data is incomplete and the function is officially blocked.
- aclose_source = true
   - Whether to turn off the source VC when the tunnel is completed, the default is off (true)
- aclose_target = true
   - Whether to close the destination VC when the tunnel is completed, the default is off (true)
- manipulate_fn = NULL
   - When not empty, this function is called whenever the tunnel gets new data, with input buf and output buf as arguments.
   - Can be used to modify the data in the data buf in the tunnel transmission and send it to the destination VC.
   - 定义：void (*Transform_fn)(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf);
- water_mark = 0
   - Watermark for setting the MIOBuffer used by the source VC
   - See the IOBuffer chapter for more information.

### The second init() method

`` `
void init (VConnection * vcSource, VConnection * vcTarget, 
          Continuation *aCont, 
          VIO *SourceVio, IOBufferReader *reader,
          bool aclose_source = true, bool aclose_target = true);
`` `

Init function, only set write on the target VC (call do_io_write), you need to ensure that the source VC has completed the read settings (call do_io_read), if there is no aCont incoming, then a new mutex will be used, the tunnel for the source VC read and purpose VC writes the same buf, which is the incoming reader, and releases the buf when it is finished. The parameters are as follows:

- SourceVio
   - Source VC's VIO
   - Switch state machine by calling SourceVio->set_continuation(this)
- reader
   - an IOBufferReader that reads data from the MIOBuffer of the source VC, which is passed to the destination VC as the input source for the data.

Note: Although read(do_io_read) is not set for the source VC, SourceVio->set_continuation(this) is called, so when the source VC has data coming in, it will call OneWayTunnel::startEvent to process the read data. . Calls using this method can inherit the previous IOBuffer, especially if there is already data in the previous IOBuffer.

### The third init() method

`` `
void init(Continuation *aCont, VIO *SourceVio, VIO *TargetVio, 
          bool aclose_source = true, bool aclose_target = true);
`` `
- SourceVio
   - Source VC's VIO
   - Switch state machine by calling SourceVio->set_continuation(this)
- TargetVio
   - Target VC's VIO
   - Switch state machine by calling TargetVio->set_continuation(this)

This call the init function, indicating that the source VC read and the destination VC write have been set, if there is no aCont incoming, then a new mutex will be used, the tunnel for the source VC read and the destination VC write, the caller needs to guarantee Both the source VC and the destination VC use the same buf (if not the same buf, the current code will throw an exception) and release the buf upon completion.

### The internal implementation of the default manually_fn: transfer_data

There is an incomplete internal summary_fn implementation in the code that can be used for reference.

Note: There is an assert in this transfer_data function, marked as "Function not fully implemented", so it cannot be called.

The transfer_data function is called when it is determined in OneWayTunnel::transform that the source buf is different from the address of the target buf.
Therefore, you can assume that OneWayTunnel does not support single_buffer = false.

`` `
inline void
transfer_data(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf)
{
  ink_release_assert(!"Not Implemented.");

  int64_t n = in_buf.reader()->read_avail();
  int64_t o = out_buf.writer()->write_avail();

  if (n > o)
    n = o;
  if (!n)
    return;
  memcpy(in_buf.reader()->start(), out_buf.writer()->end(), n);
  in_buf.reader()->consume(n);
  out_buf.writer()->fill(n);
}

void
OneWayTunnel::transform(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf)
{
  if (manipulate_fn)
    manipulate_fn(in_buf, out_buf);
  // Here the writer() method is called to get the pointer to the MIOBuffer buffer of the two bufs.
  // If equal, the asingle_buffer is true
  / / If not equal, you need to call transfor_data to achieve data copy between the two buffers
  // But the implementation of transfer_data may have problems, and the official block it will cause an exception to be thrown once called.
  else if (in_buf.writer() != out_buf.writer())
    transfer_data(in_buf, out_buf);
}
`` `

### connection_closed function

BUG: When calling the callback function, the second parameter cont passed should be this, so that the instance of the Tunnel can be passed into the callback function, and then the callback function calls OneWayTunnel_free to release the Tunnel.

`` `
void
OneWayTunnel::connection_closed(int result)
{
  if (cont) {
#ifdef TEST
    cout << "OneWayTunnel::connection_closed() ... calling cont" << endl;
#endif
    Cont->handleEvent(result ? VC_EVENT_ERROR : VC_EVENT_EOS, cont); // The last cont should be this
  } else {
    OneWayTunnel_free(this);
  }
}

`` `

### startEvent Process Analysis

After the init call, the Tunnel's callback function is set to OneWayTunnel::startEvent, and its internal logic is as follows:

`` `
当 VC_EVENT_READ_READY
    // Since do_io_read is only executed on the source VC, this event is only fired when the source VC has data readable.
    transform(vioSource->buffer, vioTarget->buffer);
    // Activate the target VC to complete the write operation
    vioTarget->reenable();
    ret = VC_EVENT_CONT;

当 VC_EVENT_WRITE_READY
    // Since do_io_write is only executed on the destination VC, this event is only fired when the destination VC buffer is writable.
    // But because it is a tunnel, when the destination VC can be written, it indicates that the VIO of the target VC has free space.
    // At this point, the source VC is activated to provide data to fill the write buffer of the destination VC.
    // So if the source VC is available, activate the source VC to read the data.
    if (vioSource) vioSource->reenable();
    ret = VC_EVENT_CONT;

当 VC_EVENT_EOS
    // usually indicates that the connection is closed
    / / Determine which end is closed the connection, if it is the source VC, then write the data in the buf, and then deal with according to VC_EVENT_READ_COMPLETE
    if (vio == vioSource) {
        transform(vioSource->buffer, vioTarget->buffer);
        goto Lread_complete;
    // If the destination VC is closed, then you don’t have to worry about it, just when the tunnel is finished/completed.
    } else goto Ldone;

Lread_complete:
当 VC_EVENT_READ_COMPLETE
    // set write nbytes to the current buffer size
    // Calculate the number of write bytes that the actual target VC needs to complete = new data read by the + buffer that was previously completed
    vioTarget->nbytes = vioTarget->ndone + vioTarget->buffer.reader()->read_avail();
    // If the data of the target VC is also finished, the entire tunnel is also finished/completed.
    if (vioTarget->nbytes == vioTarget->ndone)
        goto Ldone;
    // If the target VC has not been completed, activate the target VC to complete the write operation.
    vioTarget->reenable();
    // If you do not use the SetupTwoWayTunnel to set the associated tunnel, you can close the source VC first.
    // passed to the close_source_vio parameter 0 indicating that the lerrno parameter passed to do_io_close is -1 (representing normal shutdown)
    if (!tunnel_peer) close_source_vio(0);

Learner:
当 VC_EVENT_ERROR
    lerrno = ((VIO *) date) -> vc_server-> lerrno;
当 VC_EVENT_INACTIVITY_TIMEOUT:
当 VC_EVENT_ACTIVE_TIMEOUT:
    result = -1;
    // Here both ERROR and TIMEOUT are classified as errors and the Tunne processing is complete
Ldone:
当 VC_EVENT_WRITE_COMPLETE
    // At this point, it means that the entire tunnel is over/completed.
    // If the associated tunnel is set using SetupTwoWayTunnel, then an event is sent to the associated tunnel to notify it to close.
    if (tunnel_peer) tunnel_peer->startEvent(ONE_WAY_TUNNEL_EVENT_PEER_CLOSE, data);
    / / Need to close the source VC, target VC according to the init settings
    close_source_vio(result);
    close_target_vio(result);
    / / Responsible for releasing the Tunnel, if aCont is passed in init, it is responsible for callback aCont, aCont will be responsible for releasing the Tunnel
    connection_closed(result);

`` `

## References

- [I_OneWayTunnel.h](http://github.com/apache/trafficserver/tree/master/iocore/utils/I_OneWayTunnel.h)
