# core component: SSLNetVConnection

SSLNetVConnection inherits from UnixNetVConnection and builds support for SSL sessions.

As with UnixNetVConnection, there is also a comparison of the IOCoreSSL subsystem, which is the same as EventSystem. There are also Thread, Processor and Event, but the names are different:

|  EventSystem   |         IOCoreNet         |         IOCoreSSL         |
|: --------------: |: -------------------------: |: --- ----------------------:: |
|      Event     |     UnixNetVConnection    |     SSLNetVConnection     |
| EThread | NetHandler, InactivityCop | NetHandler, InactivityCop |
| EventProcessor | NetProcessor | sslNetProcessor |

- Like Event and UnixNetVConnection, SSLNetVConnection also provides a way to the upper state machine
  - do_io_* series
  - (set|cancel)_*_timeout 系列
  - (add|remove)_*_queue series
  - There is also a part of the method for the upper state machine defined in sslNetProcessor
- SSLNetVConnection also provides methods for the underlying state machine
  - Usually called by NetHandler
  - These methods can be thought of as dedicated callback functions for the NetHandler state machine
  - I personally think that all the functions that deal with the socket should be placed in the NetHandler.
- SSLNetVConnection is also a state machine
  - So it also has its own handler (callback function)
    - SSLNetAccept calls acceptEvent
    - InactivityCop calls mainEvent
    - The constructor is initialized to startEvent, which is used to call connectUp(), which is for sslNetProcessor
  - There are roughly three call paths:
    - EThread  －－－  SSLNetAccept  －－－ SSLNetVConnection
    - EThread --- NetHandler --- SSLNetVConnection
    - EThread  －－－  InactivityCop  －－－  SSLNetVConnection

Since it is both an Event and an SM, it also adds SSL processing to UnixNetVConnection, so from a morphological point of view, SSLNetVConnection is much more complicated than Event and UnixNetVConnection.

SSLNetVConnection overloads some methods of UnixNetVConnection:

  - net_read_io
  - load_buffer_and_write
  - SSLNetVConnection
  - do_io_close
  - free
  - reenable
  - getSSLHandShakeComplete
  - setSSLHandShakeComplete
  - getSSLClientConnection
  - setSSLClientConnection

At the same time, some methods have been added:

  - enableRead
  - getSSLSessionCacheHit
  - setSSLSessionCacheHit
  - read_raw_data
  - initialize_handshake_buffers
  - free_handshake_buffers
  - sslStartHandShake
  - sslServerHandShakeEvent
  - sslClientHandShakeEvent
  - registerNextProtocolSet
  - endpoint
  - advertise_next_protocol
  - select_next_protocol
  - sslContextSet
  - callHooks
  - calledHooks
  - get_ssl_iobuf
  - set_ssl_iobuf
  - get_ssl_reader
  - isEosRcvd


## definition

In the following member methods, any new and overloaded UnixNetVConnection will be marked with a description.

```
// Inherited from UnixNetVConnection
class SSLNetVConnection : public UnixNetVConnection
{
  // implement super , this is very interesting and very clever
  typedef UnixNetVConnection super; ///< Parent type.
public:
  // This is the entry point of an ssl handshake, depending on whether the handshake is between the Client and the ATS, or between the ATS and the OS, and then the next two methods are called.
  virtual int sslStartHandShake(int event, int &err);
  virtual void free(EThread *t);
  
  // new method
  // both read VIO and write VIO are activated
  virtual void
  enableRead()
  {
    read.enabled = 1;
    write.enabled = 1;
  };
  
  // overload method
  // Return the member sslHandShakeComplete, indicating whether the handshake process is completed
  virtual bool
  getSSLHandShakeComplete()
  {
    return sslHandShakeComplete;
  };
  // overload method
  // Set the member sslHandShakeComplete
  void
  setSSLHandShakeComplete(bool state)
  {
    sslHandShakeComplete = state;
  };
  // overload method
  // Return the member sslClientConnection to indicate if this is an SSL connection from the Client.
  virtual bool
  getSSLClientConnection()
  {
    return sslClientConnection;
  };
  // overload method
  // Set the member sslClientConnection
  virtual void
  setSSLClientConnection(bool state)
  {
    sslClientConnection = state;
  };
  // new method
  // Set the member sslSessionCacheHit
  virtual void
  setSSLSessionCacheHit(bool state)
  {
    sslSessionCacheHit = state;
  };
  // new method
  // Returns the member sslSessionCacheHit to indicate if this is a Session Reuse SSL connection
  virtual bool
  getSSLSessionCacheHit()
  {
    return sslSessionCacheHit;
  };
  
  // new method, both methods are called by sslStartHandShake
  // Used to implement the handshake between the Client and the ATS
  int sslServerHandShakeEvent(int &err);
  // Used to implement the handshake between ATS and OS
  int sslClientHandShakeEvent(int &err);

  // overload method
  // Combine with NetHandler to decrypt SSL data (net_read_io) and encrypt (load_buffer_and_write)
  virtual void net_read_io(NetHandler *nh, EThread *lthread);
  virtual int64_t load_buffer_and_write(int64_t towrite, int64_t &wattempted, int64_t &total_written, MIOBufferAccessor &buf,
                                        int &needs);
  // Implement state machine protocol flow in NPN mode
  // Register an entry for an NPN state machine
  void registerNextProtocolSet(const SSLNextProtocolSet *);
  // overload method
  virtual void do_io_close(int lerrno = -1);

  //////////////////////////////////////////////////////////////////////////////////////////////////////////////
  // Instances of NetVConnection should be allocated        //
  // only from the free list using NetVConnection::alloc(). //
  // The constructor is public just to avoid compile errors.//
  //////////////////////////////////////////////////////////////////////////////////////////////////////////////
  SSLNetVConnection();
  virtual ~SSLNetVConnection() {}

  // Point to the object used to save the SSL session
  SSL *ssl;
  // Record the time when the ssl session starts
  ink_hrtime sslHandshakeBeginTime;
  // Record the time when ssl last sent data, updated by load_buffer_and_write after data is sent
  ink_hrtime sslLastWriteTime;
  // Record the total number of bytes sent by ssl, updated by load_buffer_and_write after the data is sent
  int64_t sslTotalBytesSent;

  // Implement state machine protocol flow in NPN/ALPN mode
  // Use an entry to the NPN state machine to test the current received data, whether it can be processed by the state machine
  // set by SSL_CTX_set_next_protos_advertised_cb()
  static int advertise_next_protocol(SSL *ssl, const unsigned char **out, unsigned *outlen, void *);
  // Select the next ALPN state machine
  // set by SSL_CTX_set_alpn_select_cb()
  static int select_next_protocol(SSL *ssl, const unsigned char **out, unsigned char *outlen, const unsigned char *in,
                                  unsigned inlen, void *);

  // new method
  // Get the npn state machine. After the SSL handshake is completed, you need to set which state machine to process the received plaintext data.
  // For example, https, after completing the SSL handshake, hand it over to HttpSessionAccept to take over
  Continuation *
  endpoint() const
  {
    return npnEndpoint;
  }
  // new method
  // This method is not called, and the value of the member is directly evaluated in net_read_io.
  bool
  getSSLClientRenegotiationAbort () const
  {
    return sslClientRenegotiationAbort;
  };
  // new method
  // After the handshake is completed, if the Renegotiation initiated by the client is received, the SSL session renegotiation request is received.
  // Determine the value of ssl_allow_client_renegotiation to determine whether to call this method to set an Abort state to close this SSL connection.
  void
  setSSLClientRenegotiationAbort(bool state)
  {
    sslClientRenegotiationAbort = state;
  };
  // new method
  // return member transparentPassThrough
  bool
  getTransparentPassThrough() const
  {
    return transparentPassThrough;
  };
  // new method
  // transparentPassThrough is true when the port of ATS is set to tr-pass type
  void
  setTransparentPassThrough (boolean option)
  {
    transparentPassThrough = val;
  };
  // new method
  // Set a pointer to session_accept, but this pointer is not used in SSL
  void
  set_session_accept_pointer(SessionAccept *acceptPtr)
  {
    sessionAcceptPtr = acceptPtr;
  };
  // new method
  // return member sessionAcceptPtr
  SessionAccept *
  get_session_accept_pointer(void) const
  {
    return sessionAcceptPtr;
  };

  // Copy up here so we overload but don't override
  // 复 用 UnixNetVConnection :: reenable (VIO * saw)
  using super::reenable;

  // Reenable the VC after a pre-accept or SNI hook is called.
  // Note that the reenable here is different from the above, so SSLNetVConnection::reenable() is polymorphic
  virtual void reenable(NetHandler *nh);
  
  // Set the SSL context.
  // @note This must be called after the SSL endpoint has been created.
  // Set the context of SSL
  virtual bool sslContextSet(void *ctx);

  /// Set by asynchronous hooks to request a specific operation.
  TSSslVConnOp hookOpRequested;

  // Read undecrypted raw data directly from socket fd, put it into handShakeBuffer MIOBuffer
  int64_t read_raw_data();
  
  // Initialize the handShakeBuffer MIOBuffer, and the corresponding Reader: handShakeReader and handShakeHolder
  // handShakeBioStored is used to indicate the length of data that can be consumed in the current handShakeBuffer
  void
  initialize_handshake_buffers()
  {
    this->handShakeBuffer = new_MIOBuffer();
    this->handShakeReader = this->handShakeBuffer->alloc_reader();
    this->handShakeHolder = this->handShakeReader->clone();
    this->handShakeBioStored = 0;
  }
  // Release the handShakeBuffer MIOBuffer, and the corresponding Reader
  void
  free_handshake_buffers()
  {
    if (this->handShakeReader) {
      this->handShakeReader->dealloc();
    }
    if (this->handShakeHolder) {
      this->handShakeHolder->dealloc();
    }
    if (this->handShakeBuffer) {
      free_MIOBuffer(this->handShakeBuffer);
    }
    this->handShakeReader = NULL;
    this->handShakeHolder = NULL;
    this->handShakeBuffer = NULL;
    this->handShakeBioStored = 0;
  }
  
  // Returns true if all the hooks reenabled
  // Callback SSL Hook, return true means that all hooks are executed, allowing access to the next process
  bool callHooks(TSHttpHookID eventId);

  // Returns true if we have already called at
  // least some of the hooks
  // Has the callback process for Hook started?
  bool calledHooks(TSHttpHookID /* eventId */) { return (this->sslHandshakeHookState != HANDSHAKE_HOOKS_PRE); }

  // get iobuf
  // iobuf is used to save the decrypted data, corresponding to the MIOBuffer in VIO
  MIOBuffer *
  get_ssl_iobuf()
  {
    return iobuf;
  }

  // Set up iobuf
  void
  set_ssl_iobuf(MIOBuffer *buf)
  {
    iobuf = buf;
  }
  // This reader is the MIOBufferReader for iobuf
  IOBufferReader *
  get_ssl_reader()
  {
    return reader;
  }

  // Set eosRcvd to true if SSL is encountered.
  // This method returns this state
  bool
  isEosRcvd()
  {
    return eosRcvd;
  }

  // Get the sslTrace state
  // This is a debug debug SSL switch
  bool
  getSSLTrace() const
  {
    return sslTrace || super::origin_trace;
  };
  // Set the sslTrace state
  void
  setSSLTrace(bool state)
  {
    sslTrace = state;
  };

  // Activate sslTrace based on specific conditions
  // For example, when you need to debug SSL for a specific ip, specific domain name
  bool computeSSLTrace();

  // Get the protocol version of the SSL session
  const char *
  getSSLProtocol(void) const
  {
    if (ssl == NULL)
      return NULL;
    return SSL_get_version(ssl);
  };

  // Get the key algorithm suite for the SSL session
  const char *
  getSSLCipherSuite(void) const
  {
    if (ssl == NULL)
      return NULL;
    return SSL_get_cipher_name(ssl);
  }

  / **
   * Populate the current object based on the socket information in in the
   * con parameter and the ssl object in the arg parameter
   * This is logic is invoked when the NetVC object is created in a new thread context
   * /
  // Same as in UnixNetVConnection, this is also the new method in October 2015.
  virtual int populate(Connection &con, Continuation *c, void *arg);

private:
  SSLNetVConnection(const SSLNetVConnection &);
  SSLNetVConnection &operator=(const SSLNetVConnection &);

  // true = SSL handshake completion setting
  bool sslHandShakeComplete;
  // true = This is an SSL connection between Client and ATS
  bool sslClientConnection;
  // true = is set to be disabled during renegotiation, the SSL session is terminated
  bool sslClientRenegotiationAbort;
  // true = This SSL session for the TCP connection is a reuse of the previous SSL session.
  bool sslSessionCacheHit;
  // MIOBuffer for temporary data storage during SSL handshake
  MIOBuffer *handShakeBuffer;
  // handShakeBuffer的reader
  IOBufferReader *handShakeHolder;
  IOBufferReader *handShakeReader;
  // length of data in handShakeBuffer
  int handShakeBioStored;

  // true = tr-pass
  bool transparentPassThrough;

  /// The current hook.
  /// @note For @C SSL_HOOKS_INVOKE, this is the hook to invoke.
  // current hook point
  class APIHook *curHook;

  // This is used to mark the execution state of the PreAccept Hook
  enum {
    SSL_HOOKS_INIT,     ///< Initial state, no hooks called yet.
                        ///< initial state, no hooks have been called
    SSL_HOOKS_INVOKE,   ///< Waiting to invoke hook.
                        ///< Waiting for a callback to a hook, usually when the current hook point does not return a reenable, it stops at this state.
    SSL_HOOKS_ACTIVE,   ///< Hook invoked, waiting for it to complete.
                        ///< This is not implemented
    SSL_HOOKS_CONTINUE, ///< All hooks have been called and completed
                        ///< This is also not implemented
    SSL_HOOKS_DONE      ///< All hooks have been called and completed
                        ///< means that the PreAccept Hook is executed.
  } sslPreAcceptHookState;

  // This is used to mark the execution status of the SNI/CERT Hook
  enum SSLHandshakeHookState {
    HANDSHAKE_HOOKS_PRE, ///< initial state, no any
    HANDSHAKE_HOOKS_CERT, ///< intermediate state, only exists when openssl callback cert callback func
    HANDSHAKE_HOOKS_POST, /// <not available
    HANDSHAKE_HOOKS_INVOKE, ///< Waiting for a callback to a hook, usually when the current hook point does not return a reenable, it stops at this state.
    HANDSHAKE_HOOKS_DONE ///< means that the SNI/CERT Hook is executed.
  } sslHandshakeHookState;

  // npnSet holds the Name String and SessionAccept entries for all supported protocols
  const SSLNextProtocolSet *npnSet;
  // After the SSLaccept() handshake is successful, use the following method to make an NPN/ALPN negotiation decision:
  //     this->npnEndpoint = this->npnSet->findEndpoint(proto, len);
  // npnEndpoint will be set to the SessionAccept entry of the protocol that negotiated successfully
  // is set to NULL if the negotiation fails
  Continuation *npnEndpoint;
  // This SessionAccept doesn't seem to be used?
  SessionAccept *sessionAcceptPtr;
  // iobuf and the corresponding reader are actually duplicates of the iobuf in the trampoline.
  // Because the abstraction is not good enough, you need to define this member in SSLNetVConnection
  // And in the upper state machine, such as: HttpSM, SpdySM, H2SM need to judge NetVC,
  // If it is sslvc type, you need to get iobf from sslvc
  // If it is a unixnetvc type, use the iobuf passed in from the trampoline.
  MIOBuffer *iobuf;
  IOBufferReader *reader;
  // Did you receive eos?
  bool eosRcvd;
  // Is sslTrace turned on?
  bool sslTrace;
};

typedef int (SSLNetVConnection::*SSLNetVConnHandler)(int, void *);

extern ClassAllocator<SSLNetVConnection> sslNetVCAllocator;
```

## SSL/TLS Implementation Introduction

In theory, SSL/TLS is built on top of the TCP protocol, so you can use the HttpSM-like design approach to implement SSL/TLS handshake and data encryption and decryption.

However, such an implementation leads to a relationship between IOCoreNet and HttpSM, and from IOCoreNet to SSL/TLS to HttpSM, which implements https is a two-layer relationship.

In this way, when implementing HttpSM, it is necessary to consider that there is a higher-level state machine in the same level, and you cannot directly operate VIO.

In fact, in the implementation of IOCoreNet to SSL/TLS to HttpSM, SSL/TLS is more like a TransformVC. When we introduce TransformVC later, we will understand that it is very difficult to make HttpSM compatible with both NetVC and TransformVC.

Therefore, in the ATS implementation of SSL/TLS, SSL/TLS was chosen as part of IOCore instead of putting SSL/TLS into the upper state machine.

This has the advantage that HttpSM can use SSLNetVConnection when processing the HTTPS protocol, exactly the same as using UnixNetVConnection when processing the HTTP protocol.

The SSLNetVConnection was born, but the processing of SSL/TLS is actually a sub-state machine with two states: handshake state and data communication state. Overload and extend a bunch of methods on UnixNetVConnection to implement this substate machine.

Since it is a substate machine, there must be a pointcut. This pointcut is:

  - vc->net_read_io
  - write_to_net(vc), write_to_net_io(vc)
    - vc->load_buffer_and_write

Remember how the NetHandler::mainNetEvent section calls the two methods above?

```
  // UnixNetVConnection *
  while ((vc = read_ready_list.dequeue())) {
    if (vc->closed)
      close_UnixNetVConnection(vc, trigger_event->ethread);
    else if (vc->read.enabled && vc->read.triggered)
      vc->net_read_io(this, trigger_event->ethread);
    else if (!vc->read.enabled) {
      read_ready_list.remove(vc);
    }
  }
  while ((vc = write_ready_list.dequeue())) {
    if (vc->closed)
      close_UnixNetVConnection(vc, trigger_event->ethread);
    else if (vc->write.enabled && vc->write.triggered)
      write_to_net(this, vc, trigger_event->ethread);
    else if (!vc->write.enabled) {
      write_ready_list.remove(vc);
    }
  }
```

The following is the UnixNetVConnection read and write process:

  - When reading socket fd to MIOBuffer, call:
    - vc->net_read_io(this, trigger_event->ethread);
    - This is a member method of netvc
    - In fact, it directly calls read_from_net(nh, this, lthread);
    - This is not a member method of netvc
    - The read/readv method was called in read_from_net to implement the read operation
  - But when writing MIOBuffer to socket fd, call:
    - write_to_net(this, vc, trigger_event->ethread);
    - This is not a member method of netvc
    - In fact, write_to_net_io(nh, vc, thread) is called directly;
    - 在 write_to_net_io 中调用了vc->load_buffer_and_write(towrite, wattempted, total_written, buf, needs);
    - This is a member method of netvc
    - The write/writev method is called in vc->load_buffer_and_write to implement the read operation.

Then the read and write process of SSLNetVConnection:

  - When reading socket fd to MIOBuffer, call:
    - vc->net_read_io(this, trigger_event->ethread);
      - This is a member method of netvc, but it is overloaded in sslvc
    - ssl_read_from_net(this, lthread, r) is called in sslvc->net_read_io;
      - This is not a member method of netvc
    - The SSLReadBuffer method is called in ssl_read_from_net
    - SSL_read(ssl, buf, (int)nbytes), which calls the OpenSSL API in SSLReadBuffer; implements read and decrypt operations
  - When writing MIOBuffer to socket fd, call:
    - write_to_net(this, vc, trigger_event->ethread);
      - This is not a member method of netvc
      - In fact, write_to_net_io(nh, vc, thread) is called directly;
    - 在 write_to_net_io 中调用了vc->load_buffer_and_write(towrite, wattempted, total_written, buf, needs);
      - This is a member method of netvc, but it is overloaded in sslvc
    - The SSLWriteBuffer method is called in sslvc->load_buffer_and_write
    - SSL_write(ssl, buf, (int)nbytes), which calls the OpenSSL API in SSLWriteBuffer; implements the operation of sending and encrypting

We saw:

  - The data reading process has been basically reloaded by SSLNetVConnection
  - the data transmission process is only overloaded in the second half

The SSL/TLS handshake process is handled in net_read_io and write_to_net_io:

  - write_to_net_io is not a member function of netvc and cannot be overridden by sslvc, but it has already made a judgment on SSLVC at the time of writing.
  - When OpenSSL encounters an error state such as WANT_WRITE, it needs to wait for socket fd to be writable

Since the handshake process of SSL/TLS and the data transfer process are completely different, so:

  - The handshake process actually requires a state machine to handle
  - The data transfer process / encryption and decryption process is done by a method similar to TransformVC
  - In order to integrate the above two implementations, is the SSL/TLS design really complicated? clever? confusion?
  - Anyway, it’s all sort of difficult to understand, awkward, right. If you don’t understand, just read it twice and carefully.

## Method

### net_read_io(NetHandler *nh, EThread *lthread)

Net_read_io() is a member method of SSLNetVConnection and contains code for both SSL handshake and SSL transport.

In UnixNetVConnection, net_read_io directly calls the independent function read_from_net(nh, this, lthread).

```
// changed by YTS team, yamsat
void
SSLNetVConnection::net_read_io(NetHandler *nh, EThread *lthread)
{
  int right;
  int64_t r = 0;
  int64_t bytes = 0;
  NetState *s = &this->read;

  // If it is a blind tunnel, then use UnixNetVConnection::net_read_io instead
  // Equivalent to consistent, no SSL processing, only TCP Proxy forwarding
  if (HttpProxyPort::TRANSPORT_BLIND_TUNNEL == this->attributes) {
    this->super::net_read_io(nh, lthread);
    return;
  }

  // Try to get the Mutex lock on this VC's VIO
  MUTEX_TRY_LOCK_FOR(lock, s->vio.mutex, lthread, s->vio._cont);
  if (!lock.is_locked()) {
    // If you don't get the lock, reschedule it, wait until the next time you read it again.
    // read_reschedule will put this vc back to the tail of the read_ready_link queue
    // Due to the handling of NetHandler, all read operations will be completed in this EventSystem callback NetHandler.
    // Will not delay until the next EventSystem callback to NetHandler
    readReschedule(nh);
    return;
  }
  // Got closed by the HttpSessionManager thread during a migration
  // The closed flag should be stable once we get the s->vio.mutex in that case
  // (the global session pool mutex).
  // After getting the lock, first determine if the vc is closed asynchronously, because HttpSessionManager will manage a global session pool.
  if (this->closed) {
    // Why not call close_UnixNetVConnection() directly here?
    // Indirectly call close_UnixNetVConnection() by calling UnixNetVConnection::net_read_io to close sslvc
    this->super::net_read_io(nh, lthread);
    return;
  }
  // If the key renegotiation failed it's over, just signal the error and finish.
  // This state is in the configuration, SSL Renegotiation is disabled, but the client initiates an SSL Renegotiation request.
  if (sslClientRenegotiationAbort == true) {
    // Report an error directly, then close sslvc
    this->read.triggered = 0;
    readSignalError(nh, (int)r);
    Debug("ssl", "[SSLNetVConnection::net_read_io] client renegotiation setting read signal error");
    return;
  }

  // If it is not enabled, lower its priority.  This allows
  // a fast connection to speed match a slower connection by
  // shifting down in priority even if it could read.
  // Determine if the vc read operation is asynchronously disabled.
  if (!s->enabled || s->vio.op != VIO::READ) {
    read_disable(nh, this);
    return;
  }
  
  // The following is the preparation for the read operation.
  // Get the buffer, ready to write data
  MIOBufferAccessor &buf = s->vio.buffer;
  // Define the total length of the total data that needs to be read in VIO, and the length of the data read that has been completed.
  // ntodo is the remaining data length that needs to be read
  int64_t ntodo = s->vio.ntodo();
  ink_assert(buf.writer());

  // Continue on if we are still in the handshake
  // As mentioned earlier, the SSL/TLS handshake process is a substate machine embedded in net_read_io and write_to_net_io.
  // getSSLHandShakeComplete() returns the member sslHandShakeComplete, indicating whether the SSL handshake process has been completed.
  if (!getSSLHandShakeComplete()) {
    // If the SSL handshake process is not completed, then the SSL handshake process is entered.
    int err;

    // getSSLClientConnection() returns the member sslClientConnection, indicating whether this is an SSL connection initiated by ATS
    // If initiated by ATS, ATS is SSL Client, otherwise ATS is SSL Server
    // Call sslStartHandShake to handshake and pass the handshake method to indicate whether this is the handshake process of ATS as Client or Server.
    if (getSSLClientConnection()) {
      // ATS as SSL Client
      ret = sslStartHandShake(SSL_EVENT_CLIENT, err);
    } else {
      // ATS as SSL Server
      // will set the properties of this->attributes, such as Blind Tunnel
      ret = sslStartHandShake(SSL_EVENT_SERVER, err);
    }
    // If we have flipped to blind tunnel, don't read ahead
    // The following sections are mainly for judging the processing of the Blind Tunnel.
    if (this->handShakeReader && this->attributes != HttpProxyPort::TRANSPORT_BLIND_TUNNEL) {
      // If it is not a Blind Tunnel, and the IOBufferReader *handShakeReader associated with MIOBuffer *handShakeBuffer is not empty
      // need to process the data in handShakeBuffer
      // Check and consume data that has been read
      if (BIO_eof(SSL_get_rbio(this->ssl))) {
        this->handShakeReader->consume(this->handShakeBioStored);
        this->handShakeBioStored = 0;
      }
    } else if (this->attributes == HttpProxyPort::TRANSPORT_BLIND_TUNNEL) {
      // If it is a Blind Tunnel, it needs to transparently transmit data and not process it according to the SSL connection.
      //     dest_ip=1.2.3.4 action=tunnel ssl_cert_name=servercert.pem ssl_key_name=privkey.pem
      // Now in blind tunnel. Set things up to read what is in the buffer
      // Must send the READ_COMPLETE here before considering
      // forwarding on the handshake buffer, so the
      // SSLNextProtocolTrampoline has a chance to do its
      // thing before forwarding the buffers.
      // The state machine associated with SSLVC is still SSLNextProtocolTrampoline,
      // So the following READ_COMPLETE is passed to SSLNextProtocolTrampoline::ioCompletionEvent
      // Re-set the state machine for SSLVC through the trampoline, then SSLNextProtocolTrampoline is deleted
      // For the Blind Tunnel, the state machine is HttpSM
      this->readSignalDone(VC_EVENT_READ_COMPLETE, nh);

      // If the handshake isn't set yet, this means the tunnel
      // decision was make in the SNI callback.  We must move
      // the client hello message back into the standard read.vio
      // so it will get forwarded onto the origin server
      if (!this->getSSLHandShakeComplete()) {
        // If the SSL handshake is not completed, it is forced to be set to completed.
        this->sslHandShakeComplete = 1;

        // Copy over all data already read in during the SSL_accept
        // (the client hello message)
        // Then copy the raw SSL data (Raw Data) received from the client to the Read VIO of SSLVC
        NetState *s = &this->read;
        MIOBufferAccessor &buf = s->vio.buffer;
        int64_t r = buf.writer()->write(this->handShakeHolder);
        s->vio.nbytes += r;
        s-> vio.ndone + = r;

        // Clean up the handshake buffers
        // Then release the MIOBuffer *handShakeBuffer of SSL and the associated handShakeReader and handShakeHolder
        this->free_handshake_buffers();

        // If there is data in handShakeBuffer, then the data is copied above, and there is new data in Read VIO.
        // At this point, an event of READ_COMPLETE is sent to the upper state machine.
        if (r > 0) {
          // Kick things again, so the data that was copied into the
          // vio.read buffer gets processed
          this->readSignalDone(VC_EVENT_READ_COMPLETE, nh);
        }
      }
      // Since it is a Blind Tunnel, it will return directly after setting the SSL handshake state to complete.
      // After the conversion to the tunnel processing, pure TCP forwarding
      return;
    }
    
    // According to the return value of sslStartHandShake () for the corresponding processing
    // Need to consider ATS as a Server or Client two cases
    if (ret == EVENT_ERROR) {
      // Error handling, directly close the VC, callback trampoline ioCompletionEvent pass VC_EVENT_ERROR event type, lerror = err;
      // The value of err is set in sslStartHandShake, usually the error code for the syscall or openssl library.
      this->read.triggered = 0;
      readSignalError(nh, err);
    } else if (ret == SSL_HANDSHAKE_WANT_READ || ret == SSL_HANDSHAKE_WANT_ACCEPT) {
      // In order to complete the handshake process, more data needs to be read.
      if (SSLConfigParams::ssl_handshake_timeout_in > 0) {
        // If the SSL handshake timeout is set
        double handshake_time = ((double)(Thread::get_hrtime() - sslHandshakeBeginTime) / 1000000000);
        Debug("ssl", "ssl handshake for vc %p, took %.3f seconds, configured handshake_timer: %d", this, handshake_time,
              SSLConfigParams::ssl_handshake_timeout_in);
        if (handshake_time > SSLConfigParams::ssl_handshake_timeout_in) {
          // Found a timeout, the same error handling, and then close the connection, but the state of the EOS is fixed
          Debug("ssl", "ssl handshake for vc %p, expired, release the connection", this);
          read.triggered = 0;
          nh->read_ready_list.remove(this);
          readSignalError(nh, VC_EVENT_EOS);
          return;
        }
      }
      // Clear SSLVC from the NetHandler queue and wait until the next time there is data.
      // Note the difference with disable, disable will remove the socket fd corresponding to vc from epoll
      // And here is just skipping this SSLVC in the processing loop of this NetHandler, and the next time epoll_wait is triggered again
      read.triggered = 0;
      nh->read_ready_list.remove(this);
      readReschedule(nh);
    } else if (ret == SSL_HANDSHAKE_WANT_CONNECT || ret == SSL_HANDSHAKE_WANT_WRITE) {
      // In order to complete the handshake process, more data needs to be sent.
      // Usually the current buffer is full, so wait for the next buffer to be written before continuing to send
      // Also pay attention to the difference with disable
      write.triggered = 0;
      nh->write_ready_list.remove(this);
      writeReschedule(nh);
    } else if (ret == EVENT_DONE) { // When this happens, make an in-depth judgment
      // If this was driven by a zero length read, signal complete when
      // the handshake is complete. Otherwise set up for continuing read
      // operations.
      // Original comment translation: If this is a call driven by a 0 length read operation,
      // EVENT_DONE indicates that the handshake process is complete and the READ_COMPLETE event needs to be passed to the trampoline ioCompletionEvent.
      // Otherwise, you need to continue reading the data, then EVENT_DONE just means there is no error.
      
      // ntodo indicates the number of bytes that need to be read. I think I should use nbytes to determine if it is a 0-length read operation. ? ? TS-4216
      if (ntodo <= 0) {
        // Enter the 0 length read operation driven call processing
        if (!getSSLClientConnection()) {
          // When this SSLVC is for the ATS is the SSL Server side
          // Since the TCP_DEFER_ACCEPT attribute is set on accept, for a new Accept connection, it is necessary to determine if there is new data.
          // we will not see another ET epoll event if the first byte is already
          // in the ssl buffers, so, SSL_read if there's anything already..
          // So you have to do a read directly to see if there is new data.
          Debug("ssl", "ssl handshake completed on vc %p, check to see if first byte, is already in the ssl buffers", this);
          // Create an iobuf to receive data
          this->iobuf = new_MIOBuffer(BUFFER_SIZE_INDEX_4K);
          if (this->iobuf) {
            // Create iobuf successfully, assign reader
            this->reader = this->iobuf->alloc_reader();
            // Set the buffer in Read VIO to iobuf
            // After using the writer_for method to set, and then write data to vio.buffer, it is actually written in this->iobuf.
            // At the same time, vio.buffer is unreadable. If you want to read the data in this->iobuf, you need to do it with this->reader.
            // The iobuf here points to: SSLNextProtocolAccept::buffer (in order to implement the set 0 length read operation)
            // SSLNextProtocolAccept::buffer is not able to write data, so here you have to switch to the newly assigned MIOBuffer
            s->vio.buffer.writer_for(this->iobuf);
            // Perform a data read operation (because of TCP_DEFER_ACCEPT):
            // Read data from the SSLVC socket and write the data to this->iobuf via vio.buffer
            ret = ssl_read_from_net(this, lthread, r);
            // The read operation may encounter the situation of SSL EOS, and a judgment is needed. For example, the encryption algorithm of the handshake is not supported, the negotiation fails, and the like.
            if (ret == SSL_READ_EOS) {
              this->eosRcvd = true;
            }
#if DEBUG
            int pending = SSL_pending(this->ssl);
            if (r > 0 || pending > 0) {
              Debug("ssl", "ssl read right after handshake, read %" PRId64 ", pending %d bytes, for vc %p", r, pending, this);
            }
#endif
          } else {
            Error("failed to allocate MIOBuffer after handshake, vc %p", this);
          }
          // Although it has been disabled here, it will reset VIO again when it is followed by the callback trampoline.
          read.triggered = 0;
          read_disable(nh, this);
        }
        // Callback trampoline ioCompletionEvent pass READ_COMPLETE event, indicating that the handshake process is completed
        // At this point, the upper state machine will be called back by the trampoline, and the upper state machine will reset VIO to activate this SSLVC.
        readSignalDone(VC_EVENT_READ_COMPLETE, nh);
      } else {
        // This is not a 0-length read operation driven call, need to continue reading data, activate the NetVC in NetHandler
        read.triggered = 1;
        if (read.enabled)
          nh->read_ready_list.in_or_enqueue(this);
      }
    } else if (ret == SSL_WAIT_FOR_HOOK) {
      // avoid readReschedule - done when the plugin calls us back to reenable
      // Avoid calling readReschedule when the plugin callback is reenable
      // When I explain the SSL Hook later, analyze this block in detail.
    } else {
      // In other cases, call readReschedule
      // The underlying call read_reschedule (nh, vc);
      //     当 read.triggered == 1 && read.enabled == 1 的时候
      //     调用 nh->read_ready_list.in_or_enqueue(vc);
      readReschedule(nh);
    }
    // Return NetHandler directly during SSL handshake processing
    return;
  }
  // If the SSL handshake is complete
  // The following section is almost the same as UnixNetVConnection::net_read_io
  // just this is to call ssl_read_from_net instead of read

  // If there is nothing to do or no space available, disable connection
  // If VIO is turned off or Read VIO's MIOBuffer is full, it is forbidden to read
  if (ntodo <= 0 || !buf.writer()->write_avail()) {
    read_disable(nh, this);
    return;
  }

  // At this point we are at the post-handshake SSL processing
  // If the read BIO is not already a socket, consider changing it
  // After the handshake is completed:
  // Need to switch Read BIO from memory block type to Socket type
  if (this->handShakeReader) {
    // Check out if there is anything left in the current bio
    // Check if there is still data in the Read BIO of the memory block type
    if (!BIO_eof(SSL_get_rbio(this->ssl))) {
      // BIO_eof() returns 1 when encountering EOF
      // Still data remaining in the current BIO block
      // There is still data, then the handshake buffer should already be bound to the Read BIO of the current SSL session.
      // When the ssl_read_from_net is called later, the data in the handshake buffer is consumed by Read BIO.
    } else {
      // Read BIO has been read empty,
      // The last time the handShakeBioStored record was filled into the Read BIO byte is consumed from the handShakeReader
      // Consume what SSL has read so far.
      this->handShakeReader->consume(this->handShakeBioStored);

      // If we are empty now, switch over
      // After the consumption is complete, look at the data in the handShakeReader.
      if (this->handShakeReader->read_avail() <= 0) {
        // There is no data in handShakeReader, then you can directly switch Read BIO to Socket type
        // Switch the read bio over to a socket bio
        SSL_set_rfd(this->ssl, this->get_socket());
        // then release handShake buffers, which sets handShakeReader to NULL
        this->free_handshake_buffers();
      } else {
        // If there is still readable data in handShakeReader,
        // Then use the remaining data, create a Read BIO of the memory block type and bind to the SSL session.
        // Setup the next iobuffer block to drain
        char *start = this->handShakeReader->start();
        char *end = this->handShakeReader->end();
        this->handShakeBioStored = end - start;

        // Sets up the buffer as a read only bio target
        // Must be reset on each read
        BIO *rbio = BIO_new_mem_buf(start, this->handShakeBioStored);
        // The following detailed interpretation of set_mem_eof_return can be referenced:
        // https://www.openssl.org/docs/faq.html#PROG15
        // https://www.openssl.org/docs/manmaster/crypto/BIO_s_mem.html
        // Actually, I didn’t figure it out...
        BIO_set_mem_eof_return (rbio, -1);
        SSL_set_rbio(this->ssl, rbio);
      }
    }
  }
  // Otherwise, we already replaced the buffer bio with a socket bio

  // not sure if this do-while loop is really needed here, please replace
  // this comment if you know
  do {
    // Call ssl_read_from_net to read and decrypt the data,
    // The data read here may come from the memory block BIO can also come from Socket BIO
    ret = ssl_read_from_net(this, lthread, r);
    if (ret == SSL_READ_READY || ret == SSL_READ_ERROR_NONE) {
      bytes += r;
    }
    ink_assert(bytes >= 0);
  } while ((ret == SSL_READ_READY && bytes == 0) || ret == SSL_READ_ERROR_NONE);

  // If the data is read, callback VC_EVENT_READ_READY to the upper state machine
  if (bytes > 0) {
    if (ret == SSL_READ_WOULD_BLOCK || ret == SSL_READ_READY) {
      if (readSignalAndUpdate(VC_EVENT_READ_READY) != EVENT_CONT) {
        Debug("ssl", "ssl_read_from_net, readSignal != EVENT_CONT");
        return;
      }
    }
  }

  // Judge the return value of the last call to ssl_read_from_net
  switch (right) {
  case SSL_READ_ERROR_NONE:
  case SSL_READ_READY:
    readReschedule(nh);
    return;
    break;
  case SSL_WRITE_WOULD_BLOCK:
  case SSL_READ_WOULD_BLOCK:
    if (lock.get_mutex() != s->vio.mutex.m_ptr) {
      Debug("ssl", "ssl_read_from_net, mutex switched");
      if (ret == SSL_READ_WOULD_BLOCK)
        readReschedule(nh);
      else
        writeReschedule(nh);
      return;
    }
    // reset the trigger and remove from the ready queue
    // we will need to be retriggered to read from this socket again
    read.triggered = 0;
    nh->read_ready_list.remove(this);
    Debug("ssl", "read_from_net, read finished - would block");
#ifdef TS_USE_PORT
    if (ret == SSL_READ_WOULD_BLOCK)
      readReschedule(nh);
    else
      writeReschedule(nh);
#endif
    break;

  case SSL_READ_EOS:
    // close the connection if we have SSL_READ_EOS, this is the return value from ssl_read_from_net() if we get an
    // SSL_ERROR_ZERO_RETURN from SSL_get_error()
    // SSL_ERROR_ZERO_RETURN means that the origin server closed the SSL connection
    eosRcvd = true;
    read.triggered = 0;
    readSignalDone(VC_EVENT_EOS, nh);

    if (bytes > 0) {
      Debug("ssl", "read_from_net, read finished - EOS");
    } else {
      Debug("ssl", "read_from_net, read finished - 0 useful bytes read, bytes used by SSL layer");
    }
    break;
  case SSL_READ_COMPLETE:
    readSignalDone(VC_EVENT_READ_COMPLETE, nh);
    Debug("ssl", "read_from_net, read finished - signal done");
    break;
  case SSL_READ_ERROR:
    this->read.triggered = 0;
    readSignalError(nh, (int)r);
    Debug("ssl", "read_from_net, read finished - read error");
    break;
  }
}
```

### ssl_read_from_net

Ssl_read_from_net is not a member method of sslvc:

  - it is a wrapper around SSLReadBuffer,
    - and SSLReadBuffer is a wrapper around the OpenSSL API function SSL_read

```
static int
ssl_read_from_net(SSLNetVConnection *sslvc, EThread *lthread, int64_t &ret)
```

The return value of ssl_read_from_net is a direct mapping to the return value of SSL_read:

|           SSL_read           |        ssl_read_from_net        |
|: ----------------------------::: ------------------- --------------: |
|  SSL_ERROR_NONE              |  SSL_READ_ERROR_NONE(0)         |
|  SSL_ERROR_SYSCALL           |  SSL_READ_ERROR(1)              |
|                              |  SSL_READ_READY(2)              |
|                              |  SSL_READ_COMPLETE(3)           |
|  SSL_ERROR_WANT_READ         |  SSL_READ_WOULD_BLOCK(4)        |
|  SSL_ERROR_WANT_X509_LOOKUP  |  SSL_READ_WOULD_BLOCK(4)        |
|  SSL_ERROR_SYSCALL           |  SSL_READ_EOS(5)                |
|  SSL_ERROR_ZERO_RETURN       |  SSL_READ_EOS(5)                |
|  SSL_ERROR_WANT_WRITE        |  SSL_WRITE_WOULD_BLOCK(10)      |

The caller of ssl_read_from_net needs to process the return value above:

  - SSL_READ_WOULD_BLOCK 和 SSL_WRITE_WOULD_BLOCK
    - indicates that you need to wait for the next NetHandler callback
  - SSL_READ_ERROR_NONE 和 SSL_READ_READY
    - Indicates that some data has been successfully read, but there is still data in the Kernel TCP/IP Buffer, and you can continue reading.
  - SSL_READ_COMPLETE
    - indicates that the data read length of the VIO setting has been completed.
  - SSL_READ_EOS
    - indicates that the connection is broken
  - SSL_READ_ERROR
    - indicates that the read operation encountered an error

In the entire SSL implementation, the following mappings are also used:

|       SSL_accept/connect     |  sslServer/ClientHandShakeEvent |
|: ----------------------------::: ------------------- --------------: |
|  SSL_ERROR_WANT_READ         |  SSL_HANDSHAKE_WANT_READ(6)     |
|  SSL_ERROR_WANT_WRITE        |  SSL_HANDSHAKE_WANT_WRITE(7)    |
|  SSL_ERROR_WANT_ACCEPT       |  SSL_HANDSHAKE_WANT_ACCEPT(8)   |
|  SSL_ERROR_WANT_CONNECT      |  SSL_HANDSHAKE_WANT_CONNECT(9)  |
|                              |  SSL_WAIT_FOR_HOOK(11)          |

These values are used during the SSL handshake, all of which are macros defined in the header of SSLNetVConnection.cc.

### write_to_net_io (write_to_net)

Write_to_net_io is not a member method of sslvc, and SSLNetVConnection shares this function with UnixNetVConnection.

Both netVC types of SSLNetVConnection and UnixNetVConnection are considered in write_to_net_io.

However, when the data is actually sent, vc->load_buffer_and_write is called to complete the data encryption transmission on the SSL session.

[Extension of nethandler: from miobuffer to socket] in CH02-09-Core-UnixNetVConnection of CH02-IOCoreNet (https://github.com/oknet/atsinternals/blob/master/CH02-IOCoreNET/CH02S09-Core-UnixNetVConnection.md #nethandler extends from miobuffer to socket) The write_to_net_io has been parsed, but the SSL part has been skipped. Let's go back and look at the SSL part of the code.

```
void
write_to_net_io(NetHandler *nh, UnixNetVConnection *vc, EThread *thread)
{
  // ...... omit part of the code
  
  // This function will always return true unless
  // vc is an SSLNetVConnection.
  // If the type of vc is SSLNetVConnection, then return the status value of whether the SSL handshake is complete,
  // If the type of vc is not SSLNetVConnection, the return value is fixed to true, and the inverse is false.
  // Since we are analyzing SSLVC, we assume that the type of vc is SSLNetVConnection.
  if (!vc->getSSLHandShakeComplete()) {
    // did not complete the SSL handshake
    int err right

    // Confirm the direction of sslvc as SSL Client or SSL Server?
    // here is the same as the part of net_read_io
    if (vc->getSSLClientConnection())
      ret = vc->sslStartHandShake(SSL_EVENT_CLIENT, err);
    else
      ret = vc->sslStartHandShake(SSL_EVENT_SERVER, err);

    // According to the return value of sslStartHandShake () for the corresponding processing
    // Need to consider ATS as a Server or Client two cases
    if (ret == EVENT_ERROR) {
      // encountered an error, close the write, call back to the upper state machine
      vc->write.triggered = 0;
      write_signal_error(nh, vc, err);
    } else if (ret == SSL_HANDSHAKE_WANT_READ || ret == SSL_HANDSHAKE_WANT_ACCEPT) {
      // need to read more data, but the buffer is empty
      // So reschedule the read operation, wait for the next callback of NetHandler::mainNetEvent
      vc->read.triggered = 0;
      nh->read_ready_list.remove(vc);
      read_reschedule(nh, vc);
    } else if (ret == SSL_HANDSHAKE_WANT_CONNECT || ret == SSL_HANDSHAKE_WANT_WRITE) {
      // need to send more data, but the buffer is full
      // So reschedule the write operation, wait for the next callback of NetHandler::mainNetEvent
      vc->write.triggered = 0;
      nh->write_ready_list.remove(vc);
      write_reschedule(nh, vc);
    } else if (ret == EVENT_DONE) {
      // Completed the handshake operation
      // Activate the write operation, let the data transfer part to determine the situation of Write VIO
      // But note that NetHandler::mainNetEvent will only call back when both triggered and enabled are set to 1.
      // enabled means that do_io() was called before.
      // triggered indicates that epoll_wait has found data activity on this vc socket fd
      vc->write.triggered = 1;
      if (vc->write.enabled)
        nh->write_ready_list.in_or_enqueue(vc);
    } else
      // 其它返回值，例如：SSL_WAIT_FOR_HOOK，SSL_READ_WOULD_BLOCK，SSL_WRITE_WOULD_BLOCK 等
      // Reschedule the write operation directly, wait for the next callback of NetHandler::mainNetEvent
      write_reschedule(nh, vc);
    return;
  }
  
  // ...... omit part of the code
}
```

### load_buffer_and_write

The function of load_buffer_and_write is to send the towrite byte data in buf through the SSL encrypted channel.

  - total_written is used to indicate the number of bytes successfully sent
  - wattempted is used to indicate the number of bytes attempted to be sent
  - needs indicates that after returning to the caller, the caller needs to activate read and/or write again

return value

  - If the towrite bytes are all successfully sent, then the return value is equal to total_written
  - If the towrite byte part is successfully sent, then:
    - The return value is greater than 0, indicating that the length of data that can be sent in buf may be less than the towrite byte.
    - The return value is less than 0, indicating the error value returned on the last transmission

```
int64_t
SSLNetVConnection::load_buffer_and_write(int64_t towrite, int64_t &wattempted, int64_t &total_written, MIOBufferAccessor &buf,
                                         int &needs)
{
  ProxyMutex *mutex = this_ethread()->mutex;
  int64_t r = 0;
  int64_t l = 0;
  uint32_t dynamic_tls_record_size = 0;
  ssl_error_t err = SSL_ERROR_NONE;

  // XXX Rather than dealing with the block directly, we should use the IOBufferReader API.
  int64_t offset = buf.reader()->start_offset;
  IOBufferBlock *b = buf.reader()->block;

  // When SSL is sent, a plaintext block is encrypted and sent. After encryption, it is called TLS record.
  // The receiving end must receive a full TLS record to decrypt, but is limited by the transmission of the TCP connection.
  // The size of the TCP packets that can be sent each time is different. If the length of the TLS record needs to span multiple TCP packets,
  // The receiving end must merge multiple TCP packets to get a complete TLS record before decrypting.
  // A method of dynamically adjusting the plaintext block size is adopted in the ATS, so that the encrypted TLS record is transmitted in a TCP message as much as possible.
  // In this way, every time the receiving end receives a TCP packet, it can get a complete TLS record, which can be decrypted immediately and obtain the plaintext content.
  // Dynamic TLS record sizing
  ink_hrtime now = 0;
  if (SSLConfigParams::ssl_maxrecord == -1) {
    now = Thread::get_hrtime_updated();
    int msec_since_last_write = ink_hrtime_diff_msec(now, sslLastWriteTime);

    if (msec_since_last_write > SSL_DEF_TLS_RECORD_MSEC_THRESHOLD) {
      // reset sslTotalBytesSent upon inactivity for SSL_DEF_TLS_RECORD_MSEC_THRESHOLD
      sslTotalBytesSent = 0;
    }
    Debug("ssl", "SSLNetVConnection::loadBufferAndCallWrite, now %" PRId64 ",lastwrite %" PRId64 " ,msec_since_last_write %d", now,
          sslLastWriteTime, msec_since_last_write);
  }

  // Determine the situation of the Blind Tunnel, through UnixNetVConnection::load_buffer_and_write ()
  // It feels better to handle this before Dyn TLS record sizing.
  if (HttpProxyPort::TRANSPORT_BLIND_TUNNEL == this->attributes) {
    return this->super::load_buffer_and_write(towrite, wattempted, total_written, buf, needs);
  }

  // for SSL trace debugging
  bool trace = getSSLTrace();
  Debug("ssl", "trace=%s", trace ? "TRUE" : "FALSE");

  do {
    // Prepare the data block for sending and calculate its length
    // check if we have done this block
    l = b->read_avail();
    l -= offset;
    if (l <= 0) {
      offset = -l;
      b = b->next;
      continue;
    }
    // check if to amount to write exceeds that in this buffer
    int64_t wavail = towrite - total_written;

    if (l > wavail) {
      l = wavail;
    }

    // still for Dyn TLS record sizing
    // TS-2365: If the SSL max record size is set and we have
    // more data than that, break this into smaller write
    // operations.
    int64_t orig_l = l;
    if (SSLConfigParams::ssl_maxrecord > 0 && l > SSLConfigParams::ssl_maxrecord) {
      l = SSLConfigParams::ssl_maxrecord;
    } else if (SSLConfigParams::ssl_maxrecord == -1) {
      if (sslTotalBytesSent < SSL_DEF_TLS_RECORD_BYTE_THRESHOLD) {
        dynamic_tls_record_size = SSL_DEF_TLS_RECORD_SIZE;
        SSL_INCREMENT_DYN_STAT(ssl_total_dyn_def_tls_record_count);
      } else {
        dynamic_tls_record_size = SSL_MAX_TLS_RECORD_SIZE;
        SSL_INCREMENT_DYN_STAT(ssl_total_dyn_max_tls_record_count);
      }
      if (l > dynamic_tls_record_size) {
        l = dynamic_tls_record_size;
      }
    }

    if (!l) {
      break;
    }

    // Set the number of bytes sent by the desired output for this time
    wattempted = l;
    // Set total_writen to the total number of bytes sent after this successful transmission.
    // Is this setting strange? ?
    total_written += l;
    Debug("ssl", "SSLNetVConnection::loadBufferAndCallWrite, before SSLWriteBuffer, l=%" PRId64 ", towrite=%" PRId64 ", b=%p", l,
          towrite, b);
    // Call SSLWriteBuffer for data encryption and transmission
    // ssl is the SSL CTX descriptor
    // b->start() + offset is the starting address of the plaintext data content to be sent
    // l is the number of bytes expected to be sent
    // r is the number of bytes successfully sent
    // err is non-zero when an error occurs
    err = SSLWriteBuffer(ssl, b->start() + offset, l, r);

    // Display debug information for SSL as needed
    if (!origin_trace) {
      TraceOut((0 < r && trace), get_remote_addr(), get_remote_port(), "WIRE TRACE\tbytes=%d\n%.*s", (int)r, (int)r,
               b->start() + offset);
    } else {
      char origin_trace_ip[INET6_ADDRSTRLEN];
      ats_ip_ntop(origin_trace_addr, origin_trace_ip, sizeof(origin_trace_ip));
      TraceOut((0 < r && trace), get_remote_addr(), get_remote_port(), "CLIENT %s:%d\ttbytes=%d\n%.*s", origin_trace_ip,
               origin_trace_port, (int)r, (int)r, b->start() + offset);
    }

    if (r == l) {
      // This data is sent successfully, set wattempted to the total number of bytes sent.
      wattempted = total_written;
    }
    
    // Determine whether the current IOBufferBlock transmission is affected by Dyn TLS record sizing and send the fragment
    if (l == orig_l) {
      // No fragmentation, continue to send the next IOBufferBlock
      // on to the next block
      offset = 0;
      b = b->next;
    } else {
      // If fragmentation occurs, continue to send the rest of the current IOBufferBlock
      offset += l;
    }

    Debug("ssl", "SSLNetVConnection::loadBufferAndCallWrite,Number of bytes written=%" PRId64 " , total=%" PRId64 "", r,
          total_written);
    NET_INCREMENT_DYN_STAT(net_calls_to_write_stat);
    // in case:
    // This round is sent successfully: r == l
    // Not all sent tasks: total_written < towrite
    // There is also an IOBufferBlock that holds the data: b
    // Then continue to the next loop
  } while (r == l && total_written < towrite && b);

  if (r > 0) {
    // I didn't encounter an error when jumping out of while
    sslLastWriteTime = now;
    sslTotalBytesSent += total_written;
    if (total_written != wattempted) {
      // did not complete all data transmission, at this time b==NULL
      // Need to call back the upper state machine WRITE_READY, refill the data, and then reschedule the write operation,
      // Wait for the next NetHandler callback and continue to complete the data transmission.
      Debug("ssl", "SSLNetVConnection::loadBufferAndCallWrite, wrote some bytes, but not all requested.");
      // I'm not sure how this could happen. We should have tried and hit an EAGAIN.
      // Inform the caller (write_to_net_io), need to reschedule the write operation
      needs |= EVENTIO_WRITE;
      // Returns the number of bytes successfully sent last time
      return (r);
    } else {
      // Completed all data transmission, at this time total_written >= towrite
      // !!! Here you need to pay attention!!!
      // If total amount of data available for sending in IOBufferBlock is greater than towrite value, total_written > towrite will occur
      // Is this a bug? ?
      Debug("ssl", "SSLNetVConnection::loadBufferAndCallWrite, write successful.");
      // Returns the number of bytes actually sent successfully
      return (total_written);
    }
  } else {
    // I encountered an error when jumping out of while
    // Determine the type of error based on the return value of the last call to SSLWriteBuffer
    switch (err) {
    case SSL_ERROR_NONE:
      // no error
      Debug("ssl", "SSL_write-SSL_ERROR_NONE");
      break;
    case SSL_ERROR_WANT_READ:
      // need to read the data
      needs |= EVENTIO_READ;
      r = -EAGAIN;
      SSL_INCREMENT_DYN_STAT(ssl_error_want_read);
      Debug("ssl.error", "SSL_write-SSL_ERROR_WANT_READ");
      break;
    case SSL_ERROR_WANT_WRITE:
      // need to send data
    case SSL_ERROR_WANT_X509_LOOKUP: {
      // need to query X509
      if (SSL_ERROR_WANT_WRITE == err) {
        SSL_INCREMENT_DYN_STAT(ssl_error_want_write);
      } else if (SSL_ERROR_WANT_X509_LOOKUP == err) {
        SSL_INCREMENT_DYN_STAT(ssl_error_want_x509_lookup);
        TraceOut(trace, get_remote_addr(), get_remote_port(), "Want X509 lookup");
      }

      needs |= EVENTIO_WRITE;
      r = -EAGAIN;
      Debug("ssl.error", "SSL_write-SSL_ERROR_WANT_WRITE");
      break;
    }
    case SSL_ERROR_SYSCALL:
      // encountered an error while calling the system API in the OpenSSL API
      TraceOut(trace, get_remote_addr(), get_remote_port(), "Syscall Error: %s", strerror(errno));
      r = -errno;
      SSL_INCREMENT_DYN_STAT(ssl_error_syscall);
      Debug("ssl.error", "SSL_write-SSL_ERROR_SYSCALL");
      break;
    // end of stream
    case SSL_ERROR_ZERO_RETURN:
      // usually indicates that the connection is broken
      // Different from the read operation, does not return EOS, because the Write VIO callback VC_EVENT_EOS will not be called
      TraceOut(trace, get_remote_addr(), get_remote_port(), "SSL Error: zero return");
      r = -errno;
      SSL_INCREMENT_DYN_STAT(ssl_error_zero_return);
      Debug("ssl.error", "SSL_write-SSL_ERROR_ZERO_RETURN");
      break;
    case SSL_ERROR_SSL:
      // indicates that an internal SSL error has been encountered
    default: {
      char buf [512];
      unsigned long e = ERR_peek_last_error();
      ERR_error_string_n(e, buf, sizeof(buf));
      TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL Error: sslErr=%d, ERR_get_error=%ld (%s) errno=%d", err, e, buf,
              errno);
      r = -errno;
      SSL_CLR_ERR_INCR_DYN_STAT(this, ssl_error_ssl, "SSL_write-SSL_ERROR_SSL errno=%d", errno);
    } break;
    }
    // return to the caller
    return (r);
  }
}
```


## References

- [P_SSLNetVConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNetVConnection.h)
- [SSLNetVConnection.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNetVConnection.cc)
- [SSLUtils.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLUtils.cc)

  
