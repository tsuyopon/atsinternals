# Base component: OneWayMultiTunnel

Directly translate a comment in I_OneWayMultiTunnel.h as follows:

OneWayMultiTunnel is a general-purpose state machine that connects a source VC to multiple target VCs. It is very similar to OneWayTunnel.
However, it connects multiple target VCs and writes the data read by the source VC to all destination VCs.
The macro defines ONE_WAY_MULTI_TUNNEL_LIMIT, which defines the maximum number of target VCs.

Please refer to OneWayTunnel for other details.

## definition

```
/** Maximum number which can be tunnelled too */
// Since each target VC needs to allocate an IOBufferReader from MIOBuffer, the source VC also uses an IOBufferReader.
// So, the maximum number of target VCs defined here is one less than the maximum value of IOBufferReader (used by the source VC)
// The maximum value of an MIOBuffer assignable IOBufferReader defined in I_IOBuffer.h is 5
//     #define MAX_MIOBUFFER_READERS 5
// So here can only be defined as 4, if you need more target VC, you need to modify the maximum value of IOBufferReader at the same time.
#define ONE_WAY_MULTI_TUNNEL_LIMIT 4

// Inherited from OneWayTunnel
struct OneWayMultiTunnel : public OneWayTunnel {
  //
  // Public Interface
  //

  // Use these to construct/destruct OneWayMultiTunnel objects

  / **
    Allocates a OneWayMultiTunnel object.
    Static method to create a OneWayMultiTunnel object

    @return new OneWayTunnel object.

  * /
  static OneWayMultiTunnel *OneWayMultiTunnel_alloc();

  / **
    Deallocates a OneWayTunnel object.
    Static method to release the OneWayMultiTunnel object
  * /
  static void OneWayMultiTunnel_free(OneWayMultiTunnel *);
  // Constructor
  OneWayMultiTunnel();

  // Use One of the following init functions to start the tunnel.
  // The init function has multiple types. Please select the appropriate init function to start the tunnel according to your needs.
  / **
    The first init() method, which will be described and analyzed in detail below.
    This init function sets up the read (calls do_io_read) and the write
    (calls do_io_write).

    @param vcSource source VConnection. A do_io_read should not have
      been called on the vcSource. The tunnel calls do_io_read on this VC.
    @param vcTargets array of Target VConnections. A do_io_write should
      not have been called on any of the vcTargets. The tunnel calls
      do_io_write on these VCs.
    @param n_vcTargets size of vcTargets.
    @param aCont continuation to call back when the tunnel finishes. If
      not specified, the tunnel deallocates itself without calling
      back anybody.
    @param size_estimate size of the MIOBuffer to create for reading/
      writing to/from the VC's.
    @param nbytes number of bytes to transfer.
    @param asingle_buffer whether the same buffer should be used to read
      from vcSource and write to vcTarget. This should be set to true
      in most cases, unless the data needs be transformed.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the end.
      If aCont is not specified, this should be set to true.
    @param manipulate_fn if specified, the tunnel calls this function
      with the input and the output buffer, whenever it gets new data
      in the input buffer. This function can transform the data in the
      input buffer.
    @param water_mark for the MIOBuffer used for reading.

  * /
  void init(VConnection *vcSource, VConnection **vcTargets, int n_vcTargets, Continuation *aCont = NULL,
            int size_estimate = 0, // 0 == best guess
            int64_t nbytes = TUNNEL_TILL_DONE, bool asingle_buffer = true, bool aclose_source = true, bool aclose_target = true,
            Transform_fn manipulate_fn = NULL, int water_mark = 0);

  / **
    The second init() method, which will be described and analyzed in detail below.
    Use this init function if both the read and the write sides have
    already been setup. The tunnel assumes that the read VC and the
    write VCs are using the same buffer and frees that buffer when the
    transfer is done (either successful or unsuccessful).

    @param aCont continuation to call back when the tunnel finishes. If
      not specified, the tunnel deallocates itself without calling back
      anybody.
    @param SourceVio read VIO of the Source VC.
    @param TargetVios array of write VIOs of the Target VCs.
    @param n_vioTargets size of TargetVios array.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target ff true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.

  * /
  void init(Continuation *aCont, VIO *SourceVio, VIO **TargetVios, int n_vioTargets, bool aclose_source = true,
            bool aclose_target = true);

  //
  // Private
  //
  // main state handler
  int startEvent(int event, void *data);

  // Activate both source VC and target VC
  // I don’t see any examples of calls, I don’t know the specific function.
  virtual void reenable_all();
  // If vio==NULL, turn off all target VIOs and VCs
  // Otherwise, only the specified VIO and corresponding VC are closed
  virtual void close_target_vio(int result, VIO *vio = NULL);

  // indicates the number of target VCs currently managed. The default is 0.
  int n_vioTargets;
  // array, which holds the Write VIO for each target VC
  VIO *vioTargets[ONE_WAY_MULTI_TUNNEL_LIMIT];
  / / Set to write to the target VC MIOBuffer package
  MIOBufferAccessor topOutBuffer;
  // The default is false. Source VIO may have completed before entering the init function (ntodo == 0)
  bool source_read_previously_completed;
};
```

## Method

### The first init() method

```
void init(VConnection *vcSource, VConnection **vcTargets, int n_vcTargets, 
         Continuation *aCont = NULL, int size_estimate = 0, // 0 == best guess
         int64_t nbytes = TUNNEL_TILL_DONE, bool asingle_buffer = true, 
         bool aclose_source = true, bool aclose_target = true,
         Transform_fn manipulate_fn = NULL, int water_mark = 0);
```
The init function will set read on the source VC (call do_io_read) and set write on all target VCs (call do_io_write).

Relative to the first OneWayTunnel::init()

- Added n_vcTargets to specify the number of vcTargets
- Cut off aMutex because there is no need to force the same mutex to implement TwoWayTunnel
- Therefore, aCont's mutex will be preferred. If aCont is not specified, then new_ProxyMutex()

Note that when asingle_buffer is set to false:

- Source VC uses a buf independently
- All target VCs share the same buf, and the IOBufferReader used by the target VC is allocated from the buf.

### The second init() method

```
void init(Continuation *aCont, VIO *SourceVio, VIO **TargetVios, int n_vioTargets, 
         bool aclose_source = true, bool aclose_target = true);
```
Init function, indicating that the source VC read and the destination VC write have been set, if there is no aCont incoming, then a new mutex will be used, the tunnel for the source VC read and the destination VC write, the caller needs to guarantee, the source VC and all destination VCs use the same buf (if not the same buf, the current code will throw an exception), and release buf after completion.

Relative to the third OneWayTunnel::init()

- Added n_vcTargets to specify the number of vcTargets.

In this case, it is possible that the previous read operation on the source VC has been completed, so during the initialization process:
```
source_read_previously_completed = (SourceVio->ntodo() == 0);
```

SourceVC is usually turned off after receiving the READ_COMPLETE event on SourceVC

- Then each TargetVC will close TargetVC after receiving the WRITE_COMPLETE event
- The entire OneWayMultiTunnel is automatically closed when the last TargetVC is closed

However, when source_read_previously_completed is true, startEvent does not receive the VC_EVENT_READ_COMPLETE event on SourceVC

- But each TargetVC will still receive the WRITE_COMPLETE event and then close TargetVC
- After the last TargetVC is closed, only SourceVC remains
- At this point, if source_read_previously_completed is true, then SourceVC is directly closed.
- Finally close the entire OneWayMultiTunnel

## startEvent Process Analysis

After the init call, the tunnel's callback function is set to OneWayMultiTunnel::startEvent, and its internal logic is as follows:

```
当 VC_EVENT_READ_READY
    // Since do_io_read is only executed on the source VC, this event is only fired when the source VC has data readable.
    // Call transform to transfer data from source VIO to temporary VIO: topOutBuffer
    transform(vioSource->buffer, topOutBuffer);
    // Activate all target VCs to complete the write operation
    for (int i = 0; i < n_vioTargets; i++)
        if (vioTargets[i])
            vioTargets[i]->reenable();
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
    // If the tunnel type is not completed until the connection is closed,
    // It is necessary to determine if there is still data in the incoming VIO. If there is, then the Tunnel has an error.
    if (!tunnel_till_done && vio->ntodo())
        goto Lerror;
    / / Determine which end is closed the connection, if it is the source VC, then write the data in the buf, and then deal with according to VC_EVENT_READ_COMPLETE
    if (vio == vioSource) {
        transform(vioSource->buffer, topOutBuffer);
        goto Lread_complete;
    // If the destination VC is closed, then it is processed according to VC_EVENT_WRITE_COMPLETE
    } else goto Lwrite_complete;

Lread_complete:
当 VC_EVENT_READ_COMPLETE
    // set write nbytes to the current buffer size
    // handle each target VC
    for (int i = 0; i < n_vioTargets; i++)
        if (vioTargets[i]) {
            // Calculate the number of write bytes that the actual target VC needs to complete = new data read by the + buffer that was previously completed
            vioTarget->nbytes = vioTarget->ndone + vioTarget->buffer.reader()->read_avail();
            // If the target VC has not been completed, activate the target VC to complete the write operation.
            vioTarget->reenable();
        }
    // Turn off the VIO of the source VC because the read operation is complete
    // passed to the close_source_vio parameter 0 indicating that the lerrno parameter passed to do_io_close is -1 (representing normal shutdown)
    close_source_vio(0);
    ret = VC_EVENT_DONE;

Lwrite_complete:
当 case VC_EVENT_WRITE_COMPLETE
    / / Close the VIO of the target VC
    close_target_vio(0, (VIO *)data);
    / / Determine the number of remaining channels
    // if there are 0 channels remaining
    // or the number of remaining channels is 1 while source_read_previously_completed is true
    // Note: The VC_EVENT_READ_COMPLETE event is not fired when source_read_previously_completed is true.
    // Therefore, the source VC cannot be actively shut down. The remaining channel here is the read channel of the source VC.
    // Compare OneWayTunnel with VC_EVENT_WRITE_COMPLETE as the unique identifier for the tunnel.
    // When VC_EVENT_WRITE_COMPLETE is received, it means that the tunnel is completed. At this time, the VIO of the source VC and the target VC are directly closed, and the connection is completed.
    if ((n_connections == 0) || (n_connections == 1 && source_read_previously_completed))
    // means that the entire tunnel is over/completed.
        goto Ldone;
    // Otherwise, if the source VC is available, the source VC is activated to read the data.
    // In this case, the VIO of the source VC should have been closed.
    // If the source VC is not closed, the source VC is activated just to let the source VC trigger VC_EVENT_EOS.
    // After VC_EVENT_EOS is triggered, all available target VCs are activated to perform write operations.
    else if (vioSource)
      vioSource->reenable();

Learner:
When an error occurs, timeout, or after the tunnel is completed
    result = -1;
Ldone:
    / / Need to close the source VC, target VC according to the init settings
    close_source_vio(result);
    close_target_vio(result);
    / / Responsible for releasing the Tunnel, if aCont is passed in init, it is responsible for callback aCont, aCont will be responsible for releasing the Tunnel
    connection_closed(result);

```

Since the transform() function inherited from OneWayTunnel is still called when the data is passed in the MIOBuffer of the source VC and the MIOBuffer of the target VC, the OneWayMultiTunnel also supports only single_buffer == true.

## Applicable scene

In the design of the Cache server, after we get the data from the source station, if this data needs to be sent to the client and saved to disk at the same time, the demand for OneWayMultiTunnel is generated.

- ServerVC is now the source VC, and ClientVC and CacheVC are the target VCs
- OneWayMultiTunnel reads data from ServerVC
- Then send data to ClientVC and CacheVC
- When ClientVC receives all the data, CacheVC also receives all the data and saves it to disk.

OneWayMultiTunnel is also required when implementing the merge back function of the proxy service.

- Both Client A and Client B initiated access to the Server
- Client A first initiated the request and the proxy server forwarded the request to the server.
- Client B's request arrives at the proxy server before the server generates a response, and the proxy server determines that it can merge with Client A's request.
- This creates a OneWayMultiTunnel with ServerVC as the source VC and Client A and Client B as the target VC.
- The data that the Server VC responds to both Client A and Client B

The tunnel state machine provides a very basic data stream forwarding function. After the transaction information processing is completed, most state machines will complete a large amount of data forwarding by using the tunnel state machine.

## References

- [I_OneWayMultiTunnel.h](http://github.com/apache/trafficserver/tree/master/iocore/utils/I_OneWayMultiTunnel.h)
- [I_OneWayTunnel.h](http://github.com/apache/trafficserver/tree/master/iocore/utils/I_OneWayTunnel.h)
