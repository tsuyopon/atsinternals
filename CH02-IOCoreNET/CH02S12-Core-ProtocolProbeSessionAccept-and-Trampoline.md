# Core assembly: ProtocolProbeSessionAccept

When introducing NetAccept, it is said that after creating a UnixNetVConnection, it will call back the upper state machine in the acceptEvent callback function, passing NET_EVENT_ACCEPT to the upper state machine.

Before implementing the SPDY protocol, there is only the HTTP protocol on port 80. After the SPDY protocol and the recently supported HTTP/2 protocol, it is necessary to identify which protocol the client uses on port 80, and then Then create the upper state machine of the corresponding protocol.

ProtocolProbeSessionAccept is used to implement this recognition process, which is easy to understand literally: "Protocol Detection" SessionAccept.

Register an upper state machine for each protocol through the register method, then use the "trampoline" to read the first part of the data, analyze the data, determine whether the protocol sent by the client is SPDY or HTTP2, if not, follow HTTP To handle.

If it is not the HTTP protocol, the HttpSM implements the error handling of the non-HTTP protocol, or automatically bypass the blind tunnel.

In order to realize the identification of the protocol type first, and then create the state machine that handles the corresponding protocol, ATS designed SessionTrampoline to cooperate with SessionAccept, so that UnixNetVC jumps from SessionAccept to SessionTrampoline, like a ninja jump, with potential energy conversion. , then jump to the final destination:

`` `
                                                                              + + +
                                                ? + + + +
                                          ? + + + +
                                    *                          +         + Yeah!     + Yeah!      + Yeah!
                               * + ======== ========== ========
                           *                            +                = SPDY =    = HTTP/2 =    = HTTP =
                        * + ======== ========== ========
                     * + == == ==
                  * + + + == == ==
                * + + Ninja + == == ==
       UnixNetVC                 +      Jump   /                            ==           ==           ==
 ================= + / == == ==
 = SessionAccept =                   +       /                              ==           ==           ==
 ================= + / == == ==
   == == + / == == ==
   == == ===== + / = ======= == == ==
   == == = + = == == ==
   ==         ==                   =   Session    =                         ==           ==           ==
   ==         ==                   =  Trampoline  =                         ==           ==           ==
   == == ================ == == ==
   == == = = == == ==
   == == = = == == ==
 ================= ====================== =========== =========================

`` `

## definition

`` `
// The structure of the enumerated type, used to define the types of protocols supported. If you need to extend the supported protocols in the future, you can expand it here.
struct ProtocolProbeSessionAcceptEnums {
  /// Enumeration for related groups of protocols.
  /// There is a child acceptor for each group which
  /// handles finer grained stuff.
  enum ProtoGroupKey {
    PROTO_HTTP,    ///< HTTP group (0.9-1.1)
    PROTO_HTTP2,   ///< HTTP 2 group
    PROTO_SPDY,    ///< All SPDY versions
    N_PROTO_GROUPS ///< Size value.
  };
};

// SessionAccept is used to implement the transition phase of protocol probe
class ProtocolProbeSessionAccept : public SessionAccept, public ProtocolProbeSessionAcceptEnums
{
public:
  // constructor, set mutex=NULL, indicating no mutex protection
  // callback function, only mainEvent
  ProtocolProbeSessionAccept() : SessionAccept(NULL)
  {
    memset(endpoint, 0, sizeof(endpoint));
    SET_HANDLER(&ProtocolProbeSessionAccept::mainEvent);
  }
  ~ProtocolProbeSessionAccept() {}

  // Register a SessionAccept state machine corresponding to the specified protocol
  void registerEndpoint(ProtoGroupKey key, SessionAccept *ap);

  // no concrete implementation, just overloading the base class method
  void accept(NetVConnection *, MIOBuffer *, IOBufferReader *);

private:
  // main callback function to accept NET_EVENT_ACCEPT
  int mainEvent(int event, void *netvc);
  ProtocolProbeSessionAccept(const ProtocolProbeSessionAccept &);            // disabled
  ProtocolProbeSessionAccept &operator=(const ProtocolProbeSessionAccept &); // disabled

  /** Child acceptors, index by @c ProtoGroupKey
      We pass on the actual accept to one of these after doing protocol sniffing.
      We make it one larger and leave the last entry NULL so we don't have to
      do range checks on the enum value.
   * /
  // an array of state machines corresponding to the protocol
  // Each protocol can only correspond to one state machine
  // The number of array elements here is one more than the number of protocols, and the last one points to NULL, so that when traversing, NULL is known to arrive at the end of the array.
  SessionAccept *endpoint[N_PROTO_GROUPS + 1];

  friend struct ProtocolProbeTrampoline;
};
`` `

## Method

`` `
int
ProtocolProbeSessionAccept::mainEvent(int event, void *data)
{
  // only handle NET_EVENT_ACCEPT events
  if (event == NET_EVENT_ACCEPT) {
    // data is vc, not NULL
    ink_assert(data);

    I SAW * saw;
    // Convert data to netvc
    NetVConnection *netvc = static_cast<NetVConnection *>(data);
    // Try to convert netvc to sslvc. If it is from HTTP/SPDY/HTTP2 protocol after SSL negotiation, the bottom layer is sslvc
    // Otherwise the conversion fails, sslvc==NULL
    SSLNetVConnection *ssl_vc = dynamic_cast<SSLNetVConnection *>(netvc);
    // Default initialization buf and reader are NULL
    MIOBuffer *buf = NULL;
    IOBufferReader *reader = NULL;
    if (ssl_vc) {
      // If it is SSLVC, get iobuf and reader from SSLVC
      // The decrypted content of the first part of the data read after the handshake is completed is stored in the ibuf of SSLVC.
      buf = ssl_vc->get_ssl_iobuf();
      reader = ssl_vc->get_ssl_reader();
    }
    // Create a trampoline for protocol detection, ready to jump/migrate vc to the target state machine
    ProtocolProbeTrampoline *probe = new ProtocolProbeTrampoline(this, netvc->mutex, buf, reader);

    // XXX we need to apply accept inactivity timeout here ...

    if (!probe->reader->is_read_avail_more_than(0)) {
      // This is usually for netvc processing, sslvc may also run here
      // For example, if the client of sslvc completes the handshake and does not send the request in time, the iobuf of sslvc is passed in, but it is empty.
      // If there is no data to read in the buffer, there is no way to judge
      Debug("http", "probe needs data, read..");
      vio = netvc->do_io_read(probe, BUFFER_SIZE_FOR_INDEX(ProtocolProbeTrampoline::buffer_size_index), probe->iobuf);
      // Therefore through the do_io_read operation associated with the trampoline, EventSystem callback trampoline to complete the judgment, reenable and then handed over to EventSystem to handle
      saw-> reenable ();
    } else {
      // This is usually for sslvc processing, netvc usually does not run here
      // Because the data is readable just after Accept, that means buf and reader from the iobuf of sslvc
      // If there is already data that can be read in the buffer, then you can use this data to determine what type of protocol is.
      Debug("http", "probe already has data, call ioComplete directly..");
      // cancel the read operation
      vio = netvc->do_io_read(NULL, 0, NULL);
      // Synchronous callback trampoline to complete the judgment
      probe->ioCompletionEvent(VC_EVENT_READ_COMPLETE, (void *)vio);
      // After the synchronous callback, the trampoline has already deleted itself, so no member of the trampoline can be referenced by the probe afterwards.
    }
    return EVENT_CONT;
  }

  // Other events are reported, but continue to run?
  MachineFatal("Protocol probe received a fatal error: errno = %d", -((int)(intptr_t)data));
  return EVENT_CONT;
}
`` `

## References

- [ProtocolProbeSessionAccept.h](http://github.com/apache/trafficserver/tree/master/proxy/ProtocolProbeSessionAccept.h)
- [ProtocolProbeSessionAccept.cc](http://github.com/apache/trafficserver/tree/master/proxy/ProtocolProbeSessionAccept.cc)

#芯组件:ProtocolProbeTrampoline

"Protocol Detection Trampoline"

## definition

`` `
struct ProtocolProbeTrampoline : public Continuation, public ProtocolProbeSessionAcceptEnums {
  static const size_t minimum_read_size = 1;
  // In iocore/net/P_Net.h, defined:
  //     static size_t const CLIENT_CONNECTION_FIRST_READ_BUFFER_SIZE_INDEX = BUFFER_SIZE_INDEX_4K;
  static const unsigned buffer_size_index = CLIENT_CONNECTION_FIRST_READ_BUFFER_SIZE_INDEX;
  IOBufferReader *reader;

  // Constructor
  // mutex inherits from netvc's mutex
  // probeParent points to the ProtocolProbeSessionAccept object that created this trampoline instance
  explicit ProtocolProbeTrampoline(const ProtocolProbeSessionAccept *probe, ProxyMutex *mutex, MIOBuffer *buffer,
                                   IOBufferReader *reader)
    : Continuation(mutex), probeParent(probe)
  {
    // If the SSLVC client does not support the NPN/ALPN protocol, then the specific protocol type will be determined here.
    // For sslvc, the incoming buffer and reader are both members of sslvc, inheriting buffer and reader
    // For netvc, the incoming buffer and reader are NULL, then create a separate buffer and reader
    // buffer and reader are used to read the first request data from the client, so that you can determine what protocol the client uses.
    this->iobuf = buffer ? buffer : new_MIOBuffer(buffer_size_index);
    this->reader = reader ? reader : iobuf->alloc_reader(); // reader must be allocated only on a new MIOBuffer.
    // Set the callback function to ioCompletionEvent
    SET_HANDLER(&ProtocolProbeTrampoline::ioCompletionEvent);
  }

  int
  ioCompletionEvent(int event, void *edata)
  {
    I SAW * saw;
    NetVConnection *netvc;
    ProtoGroupKey key = N_PROTO_GROUPS; // use this as an invalid value.

    vio = static_cast<VIO *>(edata);
    netvc = static_cast<NetVConnection *>(vio->vc_server);

    // Only VC_EVENT_READ_READY / VC_EVENT_READ_COMPLETE is the correct event
    // This means that the content has been read, and then it can be judged based on what was read.
    switch (event) {
    case VC_EVENT_EOS:
    case VC_EVENT_ERROR:
    case VC_EVENT_ACTIVE_TIMEOUT:
    case VC_EVENT_INACTIVITY_TIMEOUT:
      // Error ....
      netvc->do_io_close();
      goto done;
    case VC_EVENT_READ_READY:
    case VC_EVENT_READ_COMPLETE:
      break;
    default:
      return EVENT_ERROR;
    }

    // There must be netvc, because here is the processing of the READ_READY / READ_COMPLETE event
    ink_assert(netvc != NULL);

    // If the amount of data is insufficient, the default is to read at least 1 byte.
    if (!reader->is_read_avail_more_than(minimum_read_size - 1)) {
      // Not enough data read. Well, that sucks.
      // no longer try the second time, close directly
      netvc->do_io_close();
      goto done;
    }

    // SPDY clients have to start by sending a control frame (the high bit is set). Let's assume
    // that no other protocol could possibly ever set this bit!
    // First judge spdy
    if (proto_is_spdy(reader)) {
      key = PROTO_SPDY;
    // Then judge http2
    } else if (proto_is_http2(reader)) {
      key = PROTO_HTTP2;
    } else {
    // Last default http
      key = PROTO_HTTP;
    }

    // Since the protocol has been released by the probe, there is no need to read the data again, close Read VIO
    netvc->do_io_read(NULL, 0, NULL); // Disable the read IO that we started.

    // Find the state machine of the corresponding protocol registered in ProtocolProbeSessionAccept by probeParent
    // If the corresponding state machine is not set, then this type of state machine is not supported.
    if (probeParent->endpoint[key] == NULL) {
      Warning("Unregistered protocol type %d", key);
      // print the output information, then close netvc
      netvc->do_io_close();
      goto done;
    }

    // Directly invoke the session acceptor, letting it take ownership of the input buffer.
    // Directly callback the accept method of the state machine of the selected protocol
    // The upper method of the accept method will set Read VIO / Write VIO, etc., to take over the subsequent IO operations
    // will return quickly after the setting is completed
    probeParent->endpoint[key]->accept(netvc, this->iobuf, reader);
    // Release the state machine itself
    delete this;
    return EVENT_CONT;

  done:
    // Exception handling (since SSLVC also uses this state machine to determine the protocol type of the application layer when the client does not support NPN/ALPN, so it is necessary to judge and process SSLVC)
    // in case:
    // The incoming iobuf is provided by SSLVC, but not the internal ibuf of SSLVC
    // or
    // netvc is not an SSLVC
    // iobuf needs to be freed because the iobuf is used for the probe operation to temporarily store the first part of the data received.
    SSLNetVConnection *ssl_vc = dynamic_cast<SSLNetVConnection *>(netvc);
    if (!ssl_vc || (this->iobuf != ssl_vc->get_ssl_iobuf())) {
      free_MIOBuffer(this->iobuf);
    }
    this->iobuf = NULL;
    // Release the state machine itself
    delete this;
    return EVENT_CONT;
  }

  MIOBuffer *iobuf;
  // probeParent points to the ProtocolProbeSessionAccept state machine that created this "boringbed" instance
  const ProtocolProbeSessionAccept *probeParent;
};
`` `

## Method

`` `
static bool
proto_is_spdy(IOBufferReader *reader)
{
  // Determine the spdy protocol, the first byte is 0x80
  // SPDY clients have to start by sending a control frame (the high bit is set). Let's assume
  // that no other protocol could possibly ever set this bit!
  return ((uint8_t)(*reader)[0]) == 0x80u;
}

static bool
proto_is_http2(IOBufferReader *reader)
{
  // Determine the http2 protocol, need at least 4 bytes
  char buf[HTTP2_CONNECTION_PREFACE_LEN];
  char * end;
  ptrdiff_t nbytes;

  end = reader->memcpy(buf, sizeof(buf), 0 /* offset */);
  nbytes = end - buf;

  // Client must send at least 4 bytes to get a reasonable match.
  if (nbytes < 4) {
    return false;
  }

  ink_assert(nbytes <= (int64_t)HTTP2_CONNECTION_PREFACE_LEN);
  return memcmp(HTTP2_CONNECTION_PREFACE, buf, nbytes) == 0;
}
`` `

## No lock design / Mutex Free

The protocolProbeSessionAccept state machine's mutex is set to NULL in the constructor, indicating that the state machine is not required to be locked when it is called back, and can be called concurrently.

  - When calling a state machine from NetAccept, callback ProtocolProbeSessionAccept::mainEvent in the following way
    - action_.continuation->handleEvent(NET_EVENT_ACCEPT, vc);
  - Since the mutex of ProtocolProbeSessionAccept is NULL, it can be called concurrently

In ProtocolProbeSessionAccept::mainEvent

  - Create a ProtocolProbeTrampoline state machine for each new netvc, share the netvc mutex
    - ProtocolProbeTrampoline *probe = new ProtocolProbeTrampoline(this, netvc->mutex, buf, reader);
  - If there is no data in the iobuf, set Read VIO and wait for the callback to ProtocolProbeTrampoline
  - If there is data in the iobuf, cancel Read VIO, sync callback ProtocolProbeTrampoline
    - probe->ioCompletionEvent(VC_EVENT_READ_COMPLETE, (void *)vio);

After the ProtocolProbeTrampoline state machine is created, it is initialized by the constructor:

  - For non-SSLVC: need to allocate memory for iobf via new_MIOBuffer
  - For SSLVC: iobuf points to the ibuf of SSLVC
  - reader corresponds to iobuf
  - Set probeParent to ProtocolProbeSessionAccept to get ProtocolProbeSessionAccept::endpoint to complete the jump
  - Set the callback function to: ioCompletionEvent

The callback function ProtocolProbeTrampoline::ioCompletionEvent accepts two event types:

  - VC_EVENT_READ_READY
  - VC_EVENT_READ_COMPLETE

Then, according to the read information, it is judged whether the protocol of the application layer is SPDY or HTTP2, and if not, it is processed according to the HTTP protocol.

According to the type of the protocol, the following method is used to call back the Accept method of the upper state machine's SessionAccept.

  - probeParent->endpoint[key]->accept(netvc, this->iobuf, reader);

The accept method of the upper state machine is used to:

  - Take over netvc,
  - Reset Read/Write VIO,
  - Take over the MIOBuffer pointed to by iobuf, so you don't need to release it in ProtocolProbeTrampoline after returning
  - So return quickly after the call.

At this point, ProtocolProbeTrampoline completes the jump from NetAccept to the SessionAccept of the upper state machine, releasing itself:

  - delete this;
  - return EVENT_CONT;

If ProtocolProbeTrampoline does not successfully bounce netvc to the upper layer protocol's SessionAccept state machine, you need to consider whether to release iobuf.

  - Considering that iobuf points directly to the iobuf of sslvc in sslvc, so make a judgment before releasing

`` `
  SSLNetVConnection *ssl_vc = dynamic_cast<SSLNetVConnection *>(netvc);
  if (!ssl_vc || (this->iobuf != ssl_vc->get_ssl_iobuf())) {
    free_MIOBuffer(this->iobuf);
  }
`` `

to sum up:

  - ProtocolProbeSessionAccept is:
    - Create one per TCP Port
    - no Mutex protection
  - ProtocolProbeTrampoline is:
    - Each NetVC will create one
    - The life cycle is very short and will be recycled after jumping to the upper layer protocol's SessionAccept state machine
    - Mutex sharing NetVC


