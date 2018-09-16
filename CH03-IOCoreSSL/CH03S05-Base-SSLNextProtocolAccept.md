# Basic component: SSLNextProtocolAccept

The ProtocolProbeSessionAccept introduced in the IOCoreNET section actually refers to the implementation of SSLNextProtocolAccept.

SSLNextProtocolAccept reads the type passed in the NPN / ALPN protocol, and then jumps to the corresponding state machine through the "trampoline". ProtocolProbeSessionAccept reads the first request message sent by the client to determine what type of message is. Then, jump to the corresponding state machine through the "trampoline".

Register an upper state machine for each protocol through the register method, then use the "trampoline" to determine the type passed in the NPN / ALPN protocol. After completing the SSL handshake, jump to the corresponding state machine through the "trampoline".

You can see that the functionality combined with ProtocolProbeSessionAccept and ProtocolProbeSessionTrampoline is very close.

## definition

```
class SSLNextProtocolAccept : public SessionAccept
{
public:
  // Constructor:
  // Initialize mutex to NULL, create buffer using new_empty_MIOBuffer(),
  // Initialize the endpoint with the incoming state machine and initialize the member with the incoming transparent_passthrough
  // Set the callback function to mainEvent
  SSLNextProtocolAccept(Continuation *, bool);
  // Destructor: used to release member buffer
  ~SSLNextProtocolAccept();

  // Not implemented, can not be called
  void accept(NetVConnection *, MIOBuffer *, IOBufferReader *);

  // Register handler as an endpoint for the specified protocol. Neither
  // handler nor protocol are copied, so the caller must guarantee their
  // lifetime is at least as long as that of the acceptor.
  // Register a SessionAccept state machine corresponding to the specified protocol
  // The underlying is directly called protoset.registerEndpoint(protocol, handler)
  bool registerEndpoint(const char *protocol, Continuation *handler);

  // Unregister the handler. Returns false if this protocol is not registered
  // or if it is not registered for the specified handler.
  // Unregister a SessionAccept state machine corresponding to the specified protocol
  // This method is not used in ATS. In TSAPI, TSNetAcceptNamedProtocol will call the unregister method after calling register failure, but only for SSL.
  // So there is no unregister method in ProtocolProbeSessionAccept
  // The underlying is directly called protoset.unregisterEndpoint(protocol, handler)
  bool unregisterEndpoint(const char *protocol, Continuation *handler);

  SLINK(SSLNextProtocolAccept, link);

private:
  // Main callback function
  int mainEvent(int event, void *netvc);
  SSLNextProtocolAccept(const SSLNextProtocolAccept &);            // disabled
  SSLNextProtocolAccept &operator=(const SSLNextProtocolAccept &); // disabled

  MIOBuffer *buffer; // XXX do we really need this?
  // endpoint points to the default processing mechanism when NPN / ALPN cannot match.
  // For SSLNextProtocolAccept, passed in by the caller at creation time, set by constructor,
  // The ProtocolProbeSessionAccept object is passed in when SSLNextProtocolAccept is created in HttpProxyServerMain.cc.
  Continuation *endpoint;
  // Stores the correspondence between all currently registered protocols and upper state machine for the NPN / ALPN protocol.
  SSLNextProtocolSet protoset;
  // tr-pass flag
  bool transparent_passthrough;

  friend struct SSLNextProtocolTrampoline;
};
```

## Method

```
// 由于不同的event传入的可能是netvc，可能是vio
// 这个方法对于不同的event，总是返回sslvc
static SSLNetVConnection *
ssl_netvc_cast(int event, void *edata)
{
  union {
    VIO *vio;
    NetVConnection *vc;
  } ptr;

  switch (event) {
  case NET_EVENT_ACCEPT:
    ptr.vc = static_cast<NetVConnection *>(edata);
    return dynamic_cast<SSLNetVConnection *>(ptr.vc);
  case VC_EVENT_INACTIVITY_TIMEOUT:
  case VC_EVENT_READ_COMPLETE:
  case VC_EVENT_ERROR:
    ptr.vio = static_cast<VIO *>(edata);
    return dynamic_cast<SSLNetVConnection *>(ptr.vio->vc_server);
  default:
    return NULL;
  }
}

int
SSLNextProtocolAccept::mainEvent(int event, void *edata)
{
  // Get SSLVC
  SSLNetVConnection *netvc = ssl_netvc_cast(event, edata);
  // There should be an assert here to judge netvc!=NULL

  Debug("ssl", "[SSLNextProtocolAccept:mainEvent] event %d netvc %p", event, netvc);

  switch (event) {
  case NET_EVENT_ACCEPT:
    // Usually should only have NET_EVENT_ACCEPT
    ink_release_assert(netvc != NULL);

    // Set the tr-pass status of sslvc
    netvc->setTransparentPassThrough(transparent_passthrough);

    // Register our protocol set with the VC and kick off a zero-length read to
    // force the SSLNetVConnection to complete the SSL handshake. Don't tell
    // the endpoint that there is an accept to handle until the read completes
    // and we know which protocol was negotiated.
    // Register the currently supported protocol with the upper state machine (protoset) to SSLVC
    netvc->registerNextProtocolSet(&this->protoset);
    // Set the "boring machine" for the current SSLVC, ready to jump to the matching upper state machine
    // Set a Read VIO here, read 0 bytes, and this->buffer is created in the constructor.
    // The last parameter 0 is not useful for VIO::READ
    netvc->do_io(VIO::READ, new SSLNextProtocolTrampoline(this, netvc->mutex), 0, this->buffer, 0);
    // Set sessionAcceptPtr but it doesn't seem to be used.
    netvc->set_session_accept_pointer(this);
    return EVENT_CONT;
  default:
    // If it is another event, just close SSLVC directly
    // There may be a problem here, netvc may be NULL, if there is an assert after the ssl_netvc_cast call, it is ok
    netvc->do_io(VIO::CLOSE);
    return EVENT_DONE;
  }
}
```

## SSL session creation process

Since SSL is located in the presentation layer of OSI layer 6, it needs to interface with the application layer of OSI layer 7 after completing the encryption and decryption operation. Therefore, ATS designed a "trampoline" structure to realize the jump from SSL to SPDY, HTTP2, HTTP. The transfer process, so the implementation of ATS is as follows:

  - First, create a ProtocolProbeSessionAccept object to support the "trampoline" structure of 80-port HTTP, SPDY, and HTTP2.
    - But this method is to read the first part of the plaintext (unencrypted) data to determine the request
    - It is possible to judge incorrectly
  - Then, create an SSLNextProtocolAccept to support the 443 port SSL protocol encryption of HTTP, SPDY, HTTP2 "boring machine" structure
    - Register http, spdy, http/2 and the corresponding state machine using NPN / ALPN
    - Also pass in the ProtocolProbeSessionAccept state machine
    - In this way, the client that does not support the NPN / ALPN protocol can determine the protocol type of the application layer in the original way.

## About the new_empty_MIOBuffer method

```
TS_INLINE MIOBuffer *
new_empty_MIOBuffer_internal(
  int64_t size_index)
{
  MIOBuffer *b = THREAD_ALLOC(ioAllocator, this_thread());
  b->size_index = size_index;
  return b;
}
```

Compare the new_MIOBuffer method

```
TS_INLINE MIOBuffer *
new_MIOBuffer_internal(
  int64_t size_index)
{
  MIOBuffer *b = THREAD_ALLOC(ioAllocator, this_thread());
  b->alloc(size_index);
  return b;
}
```

Can see the difference is whether the internal method of alloc is executed

  - The alloc method allocates memory (IOBufferBlock and IOBufferData) for MIOBuffer as required by size_index
  - If you do not call the alloc method, just set the value of size_index, it means that the memory allocation is done when the data is written to MIOBuffer for the first time.

Therefore, new_empty_MIOBuffer is a structure that creates only one MIOBuffer, but does not associate IOBufferBlock and IOBufferData.

## Free Mutex

Like the ProtocolProbeSessionAccept state machine, SSLNextProtocolAccept is set to NULL in the constructor, indicating that the state machine is called without a lock and can be called concurrently.

Although the destructor for SSLNextProtocolAccept is defined, since the SSLNextProtocolAccept is not released, the destructor is not called when the system is running.

SSLNextProtocolAccept::buffer

  - When calling do_io to set up Read VIO, you need to pass in a MIOBuffer
  - But SSLNextProtocolAccept is just a state machine before the SSL handshake, so this MIOBuffer is not useful.
  - Just to pass a parameter that meets the requirements when calling do_io
  - So the buffer here uses new_empty_MIOBuffer in the constructor to complete the initialization.
  - And all SSLNextProtocolTrampoline share the same buffer
    - netvc->do_io(VIO::READ, new SSLNextProtocolTrampoline(this, netvc->mutex), 0, this->buffer, 0);
    - In fact, the iobuf in each sslvc replaces the buffer.

Finally, the callback for the SSLNextProtocolTrampoline state machine is always asynchronous.

## Reference material

- [P_SSLNextProtocolSet.h](https://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNextProtocolSet.h)
- [P_SSLNextProtocolAccept.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNextProtocolAccept.h)
- [SSLNextProtocolAccept.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNextProtocolAccept.cc)
- [P_IOBuffer.h](https://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_IOBuffer.h)
