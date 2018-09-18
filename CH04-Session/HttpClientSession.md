# core component: HttpClientSession

HttpClientSession is an inherited class of ProxyClientSession.

Since ProxyClientSession inherits from VConnection, HttpClientSession is essentially a VConnection.

Since it is a VConnection, it is like a file handle:

  - Offered do_io _ * ()
  - provide reenable () way

In the HttpClientSession::new_connection() method, it is always required to pass in a NetVConnection type.
  - The incoming NetVConnection object is saved to the HttpClientSession::client_vc member
  - The base class of NetVConnection is also a VConnection type

Compare NetVConnection and HttpClientSession :

|     Type    |   NetVConnection   |    HttpClientSession   |
|: --------------::: ------------------::: ------------- ---------:: |
| Resources | con.fd | client_vc |
| Read Callback | read._cont | client_vc->read._cont |
| Write Callback | write._cont | client_vc->write._cont |
| Driver | NetHandler | none |

Very interesting phenomenon:

  - HttpClientSession accesses NetVConnection as a resource
    - It looks like NetVConnection encapsulates Socket FD,
    - HttpClientSession also encapsulates NetVConnection
    - So XxxSM will also treat HttpClientSession as a resource.
  - HttpClientSession transparently passes the do_io operation to NetVConnection

The previous chapters have said that NetVConnection itself is also a state machine with three states and corresponding callback functions:

  - startEvent handles connect
  - acceptEvent handle accept
  - mainEvent 处理 timeout

HttpClientSession itself is also a state machine, there are also multiple states and corresponding callback functions:

  - state_wait_for_close
  - state_keep_alive
  - state_slave_keep_alive

The specific status and function will be analyzed next.

## definition

```
class HttpClientSession : public ProxyClientSession
{
public:
  typedef ProxyClientSession super; ///< Parent type.
  HttpClientSession();

  // Implement ProxyClientSession interface.
  // First, define three methods declared as pure virtual functions in ProxyClientSession
  virtual void destroy();

  virtual void
  start()
  {
    new_transaction();
  }

  void new_connection(NetVConnection *new_vc, MIOBuffer *iobuf, IOBufferReader *reader, bool backdoor);

  // Implement VConnection interface.
  // Define the do_io_* and reenable methods required by NetVConnection
  virtual VIO *do_io_read(Continuation *c, int64_t nbytes = INT64_MAX, MIOBuffer *buf = 0);
  virtual VIO *do_io_write(Continuation *c = NULL, int64_t nbytes = INT64_MAX, IOBufferReader *buf = 0, bool owner = false);

  virtual void do_io_close(int lerrno = -1);
  virtual void do_io_shutdown(ShutdownHowTo_t howto);
  virtual void reenable(VIO *vio);

  // The method defined by HttpClientSession
  // used to create a new transaction
  void new_transaction();

  // Three methods for managing the semi-closed state
  void
  set_half_close_flag()
  {
    half_close = true;
  };
  void
  clear_half_close_flag()
  {
    half_close = false;
  };
  bool
  get_half_close_flag() const
  {
    return half_close;
  };
  // Called by HttpSM to separate the association between HttpSM and HttpClientSession
  // After doing this, HttpSM will no longer be triggered by read and write events on the client's NetVConnection
  virtual void release(IOBufferReader *r);

  // Next, define two methods declared as pure virtual functions in the ProxyClientSession
  virtual NetVConnection *
  get_netvc() const
  {
    return client_vc;
  };
  // release_netvc() just separates the association between NetVConnection and HttpClientSession and does not close NetVConnection
  virtual void
  release_netvc()
  {
    client_vc = NULL;
  }

  // Associate an HttpServerSession to the current HttpClientSession
  // When HttpSM processes an HTTP request, it will:
  // Pass the callback of Server NetVConnection to HttpClientSession via attach_server_session,
  // Then release HttpSM itself.
  // If the request hits the Cache and does not need to be obtained by OServer, then the Server NetVConnection will be associated with the HttpClientSession in advance.
  // At this point, since HttpSM does not complete a complete request processing, transaction_done = false is set;
  // If the incoming HttpServerSession is NULL:
  // means canceling the association on the current HttpClientSession.
  // When the next HTTP request comes in, a new HttpSM is created by HttpClientSession to handle it. When HttpSM needs to establish a connection with OServer,
  // will find HttpClientSession whether there is a previously used HttpServerSession,
  // At this point, the previously associated HttpServerSession can be retrieved via get_server_session.
  virtual void attach_server_session(HttpServerSession *ssession, bool transaction_done = true);
  // Return HttpServerSession that is already associated with HttpClientSession
  HttpServerSession *
  get_server_session() const
  {
    return bound_ss;
  };

  // Used for the cache authenticated HTTP content feature
  HttpServerSession *get_bound_ss();

  // Functions for manipulating api hooks
  // Implement the Hook-related add operation, and set the member hooks_set = 1 of HttpSM (current_reader)
  void ssn_hook_append(TSHttpHookID id, INKContInternal *cont);
  void ssn_hook_prepend(TSHttpHookID id, INKContInternal *cont);

  // This counter is used to record how many Http requests are processed on a connection due to the Keep Alive feature of the Http protocol.
  int
  get_transact_count() const
  {
    return transact_count;
  }

  // support for HTTP/2
  void
  set_h2c_upgrade_flag()
  {
    upgrade_to_h2c = true;
  }

private:
  HttpClientSession(HttpClientSession &);

  // three state handlers
  int state_keep_alive(int event, void *data);
  int state_slave_keep_alive(int event, void *data);
  int state_wait_for_close(int event, void *data);
  // Set TCP INIT CWND
  void set_tcp_init_cwnd();

  // used to indicate the status of HttpClientSession
  enum C_Read_State {
    HCS_INIT,
    HCS_ACTIVE_READER,
    HCS_KEEP_ALIVE,
    HCS_HALF_CLOSED,
    HCS_CLOSED,
  };

  // is a self-growth ID that represents the current HttpClientSession number and will be printed to the Debug log for easy tracking
  int64_t con_id;
  // Save the incoming NetVConnection resource
  NetVConnection *client_vc;
  // magic value used for memory allocation debugging
  int magic;
  // used to record how many Http requests are processed on a connection
  int transact_count;
  // Turn on TCP INIT CWND
  bool tcp_init_cwnd_set;
  // half-closed state
  bool half_close;
  // Used to help count the number of concurrent connections handled by the current ATS: http_current_client_connections_stat
  // This value is true after the current session is counted in a concurrent connection.
  // The value is false after the current Session is subtracted from the concurrent connection.
  // But after performing the subtraction operation in do_io_close, why do you want to judge again in the release method? ? ?
  bool conn_decrease;
  // HTTP / 2 support
  bool upgrade_to_h2c; // Switching to HTTP/2 with upgrade mechanism

  // Point to the associated HttpServerSession so that when Keep Alive supports it, you can get the HttpServerSession used in the last request.
  // can guarantee that the request on the same connection can be sent to the corresponding HttpServerSession
  HttpServerSession *bound_ss;

  // MIOBuffer for ka_vio
  MIOBuffer *read_buffer;
  // IOBufferReader corresponding to read_buffer
  IOBufferReader *sm_reader;
  // point to the HttpSM state machine
  HttpSM *current_reader;
  // Describe the current state of HttpClientSession
  C_Read_State read_state;

  // VIO used in the middle of two requests when Keep Alive is supported
  VIO * ka_vio;
  VIO * slave_ka_vio;

  Link<HttpClientSession> debug_link;

public:
  /// Local address for outbound connection.
  IpAddr outbound_ip4;
  /// Local address for outbound connection.
  IpAddr outbound_ip6;
  /// Local port for outbound connection.
  uint16_t outbound_port;
  /// Set outbound connection to transparent.
  bool f_outbound_transparent;
  /// Transparently pass-through non-HTTP traffic.
  bool f_transparent_passthrough;
  /// DNS resolution preferences.
  HostResStyle host_res_style;
  /// acl record - cache IpAllow::match() call
  const AclRecord *acl_record;

  // for DI. An active connection is one that a request has
  // been successfully parsed (PARSE_DONE) and it remains to
  // be active until the transaction goes through or the client
  // aborts.
  // true, indicating that the request on the current HttpClientSession has been successfully parsed by HttpSM, and then waits for the ATS to send data to the client.
  // At this point, this NetVC is considered to be an active connection, waiting for the transaction to complete, or the connection is aborted.
  bool m_active;
};
```

## Status (State)

Five state values are defined in the definition of HttpClientSession by an enumerated type to indicate the state of the HttpClientSession:

  - HCS_INIT
    - The initial value of the constructor setting is of no use
  - HCS_ACTIVE_READER
    - Indicates that data flow from NetVC is currently being taken over by HttpSM
    - Here HttpSM is called reader, because HttpClientSession itself is also a VConnection
    - One end of the VConnection is connected to a resource that can be read and written, and one end is connected to the state machine.
  - HCS_KEEP_ALIVE
    - indicates that the current HttpSM has been separated from the HttpClientSession
    - The read event currently taking over by NetVConnection by HttpClientSession::state_keep_alive
    - The ka_vio member holds the VIO returned by do_io_read
    - read_buffer member as a buffer for receiving data
    - Inactivity Timeout 为 ka_in
    - Cancel Active Timeout
    - When the connection between the Client and the ATS supports Keep Alive, after the first request is processed, it will enter this state until the next request arrives.
  - HCS_HALF_CLOSED
    - For some reason, after HttpSM has sent a response, it will no longer receive requests from the client.
    - The client should be actively disconnected at this time, and then completely closed when HttpClientSession receives EVENT_EOS
    - The read event on NetVConnection is currently taken over by HttpClientSession::state_wait_for_close
    - The ka_vio member holds the VIO returned by do_io_read
    - read_buffer member as a buffer for receiving data
    - but subsequent received information will be directly discarded
  - HCS_CLOSED
    - indicates that the current HttpClientSession is closed
    - TS_HTTP_SSN_CLOSE_HOOK will be triggered next, then NetVConnection will be closed

In the above introduction, two callback functions are introduced at the same time.

  - state_keep_alive
  - state_wait_for_close

There is also a callback function

  - state_slave_keep_alive

In ATS, HttpServerSession is designed as a slave of HttpClientSession, the corresponding HttpClientSession is master, when a request is received on the master, an HttpSM is created to resolve, and then the slave is associated with the master. After processing an HTTP request, The HttpSM object is released.

However, the relationship between master and slave still needs to be preserved, so that in the Http Keep alive support, the request from the client will continue to be sent to the TCP connection that was last used.

But there is a gap in the two HTTP requests. There is no HttpSM object in this gap to take over the read events from Client NetVConnection and Server NetVConnection, so this task is scheduled for HttpClientSession, so:

  - ka_vio is used to save the VIO returned by do_io_read on the Client NetVConnection
  - read_buffer is the buffer used to receive data
  - ka_slave is used to save the VIO returned by do_io_read on the Server NetVConnection
  - ssession points to the HttpServerSession object associated with the HttpClientSession
  - ssession->read_buffer is the buffer used to receive data

## Method

The following is an analysis of the order in which each method is called according to the runtime. The first is called by the HttpSessionAccept::accept() method: new_connection()

```
void
HttpClientSession::new_connection(NetVConnection *new_vc, MIOBuffer *iobuf, IOBufferReader *reader, bool backdoor)
{
  ink_assert(new_vc != NULL);
  ink_assert(client_vc == NULL);
  // save new_vc to HttpClientSession
  // Equivalent to binding NetVC to HttpClientSession and unbinding using the release_netvc() method
  client_vc = new_vc;
  // Set the magic value for debugging memory out of bounds and other issues
  magic = HTTP_CS_MAGIC_ALIVE;
  // HttpClientSession shared NetVC mutex
  mutex = new_vc->mutex;
  // Try to lock
  // Since the mutex is shared with NetVC, it should be, must be locked successfully
  MUTEX_TRY_LOCK(lock, mutex, this_ethread());
  // The assert here is definitely not triggered. If it is triggered, it must be a problem.
  ink_assert(lock.is_locked());

  // Disable hooks for backdoor connections.
  // Do not allow backdoor to be hooked
  this->hooks_on = !backdoor;

  // Unique client session identifier.
  // Generate a unique ID for HttpClientSession, which will be printed in the debug log for easy tracking and analysis.
  con_id = ProxyClientSession::next_connection_id();

  // increment the HTTP current concurrent client transaction counter
  HTTP_INCREMENT_DYN_STAT(http_current_client_connections_stat);
  conn_decrease = true;
  // increment the HTTP client total connection counter
  HTTP_INCREMENT_DYN_STAT(http_total_client_connections_stat);
  // If it is an HTTPS request, you need to increment the HTTPS client total connection counter.
  if (static_cast<HttpProxyPort::TransportType>(new_vc->attributes) == HttpProxyPort::TRANSPORT_SSL) {
    HTTP_INCREMENT_DYN_STAT(https_total_client_connections_stat);
  }

  /* inbound requests stat should be incremented here, not after the
   * header has been read */
  // The total number of connections used to count HTTP incoming, the difference from http_total_client_connections_stat is...? ? ?
  HTTP_INCREMENT_DYN_STAT(http_total_incoming_connections_stat);

  // check what type of socket address we just accepted
  // by looking at the address family value of sockaddr_storage
  // and logging to stat system
  // Count the number of HTTP client connections for ipv4 and ipv6 respectively
  switch (new_vc->get_remote_addr()->sa_family) {
  case AF_INET:
    HTTP_INCREMENT_DYN_STAT(http_total_client_connections_ipv4_stat);
    break;
  case AF_INET6:
    HTTP_INCREMENT_DYN_STAT(http_total_client_connections_ipv6_stat);
    break;
  default:
    // don't do anything if the address family is not ipv4 or ipv6
    // (there are many other address families in <sys/socket.h>
    // but we don't have a need to report on all the others today)
    break;
  }

#ifdef USE_HTTP_DEBUG_LISTS
  ink_mutex_acquire(&debug_cs_list_mutex);
  debug_cs_list.push(this, this->debug_link);
  ink_mutex_release(&debug_cs_list_mutex);
#endif

  DebugHttpSsn("[%" PRId64 "] session born, netvc %p", con_id, new_vc);

  // used to take over the remaining unread data content in SSLNetVC
  if (!iobuf) {
    SSLNetVConnection *ssl_vc = dynamic_cast<SSLNetVConnection *>(new_vc);
    if (ssl_vc) {
      iobuf = ssl_vc->get_ssl_iobuf();
      sm_reader = ssl_vc->get_ssl_reader();
    }
  }

  // If there is no data left in SSLNetVC, create a new MIOBuffer ready to receive requests from the client
  read_buffer = iobuf ? iobuf : new_MIOBuffer(HTTP_HEADER_BUFFER_SIZE_INDEX);
  // Create the corresponding IOBufferReader
  // There is a situation with the SSLNetVC part that is out of sync, which may cause a bug. In the latest version, the iobuff in SSLNetVC has been canceled.
  sm_reader = reader ? reader : read_buffer->alloc_reader();

  // INKqa11186: Use a local pointer to the mutex as
  // when we return from do_api_callout, the ClientSession may
  // have already been deallocated.
  // Since the return from do_api_callout, the HttpClientSession may have been released.
  // So here a local variable is created to hold the mutex, guaranteeing normal unlocking after returning from do_api_callout.
  EThread *ethis = this_ethread();
  Ptr<ProxyMutex> lmutex = this->mutex;
  MUTEX_TAKE_LOCK(lmutex, ethis);
  // Since the do_api_callout method is not defined in HttpClientSession,
  // So here is the do_api_callout method that directly calls the base class ProxyClientSession.
  do_api_callout(TS_HTTP_SSN_START_HOOK);
  MUTEX_UNTAKE_LOCK(lmutex, ethis);
  // lmutex is an automatic pointer. Can you not write this sentence here?
  lmutex.clear();
}
```

For the sake of analysis, we analyze the SSN START here without any Plugin Hook.

  - Then do_api_callout will call handle_api_return(TS_EVENT_HTTP_CONTINUE) directly
  - then handle_api_return will call the start() method
  - HttpClientSession::start() directly calls the new_transaction() method

```
void
HttpClientSession::new_transaction()
{
  ink_assert(current_reader == NULL);
  PluginIdentity *pi = dynamic_cast<PluginIdentity *>(client_vc);

  // used to determine whether client_vc is a plugin created vc
  // If not, add client_vc to the active queue of the NetHandler
  // ATS uses active queue to limit the maximum number of concurrent connections.
  // If the addition fails, it indicates that the maximum number of concurrent connections has been reached, then directly call do_io_close() to close HttpClientSession
  if (!pi && client_vc->add_to_active_queue() == false) {
    // no room in the active queue close the connection
    this->do_io_close();
    return;
  }

  // Defensive programming, make sure nothing persists across
  // connection re-use
  // Preventive design to ensure that connection reuse does not occur
  half_close = false;

  // Set the state of HttpClientSession to HttpSM takeover
  read_state = HCS_ACTIVE_READER;
  // Create an HttpSM object
  current_reader = HttpSM::allocate();
  // Initialize the HttpSM object
  current_reader->init();
  // Http transaction counter accumulation
  transact_count++;
  DebugHttpSsn("[%" PRId64 "] Starting transaction %d using sm [%" PRId64 "]", con_id, transact_count, current_reader->sm_id);

  // Bind HttpClientSession with HttpSM
  // At this point HttpSM has pointed the I/O event callback of client_vc to the HttpSM state machine by calling the do_io method of HttpClientSession.
  current_reader->attach_client_session(this, sm_reader);
  // Extra processing for VCs created by Plugin, skip here, do not analyze for now
  if (pi) {
    // it's a plugin VC of some sort with identify information.
    // copy it to the SM.
    current_reader->plugin_tag = pi->getPluginTag();
    current_reader->plugin_id = pi->getPluginId();
  }
}
```

The next step will be transferred to the HttpSM process.

  - If HttpSM needs to establish a connection with the OS, if the sharing function of HttpServerSession is not enabled
    - will call httpSessionManager.acquire_session() in the HttpSM::do_http_server_open() method
    - In this method, the bound HttpServerSession is obtained by get_bound_ss() and saved to existing_ss
    - Then call HttpClientSession::attach_server_session(NULL) to clear the binding relationship
    - Then verify the HttpServerSession obtained before, and then call HttpSM::attach_server_session(existing_ss) as the ServerSession of HttpSM after verification.
    - Unauthenticated, call HttpClientSession::attach_server_session(NULL) to clear the binding relationship

Then HttpSM will create an HttpTunnel after the HttpTunnel has passed all the data:

  - In HttpSM::tunnel_handler_server():
    - Determine if the OS supports Keep alive,
    - Judge that there is no EOS, TIMEOUT event on the server side,
    - Determine that the client's data stream does not abort,
    - If the attach_server_session_to_client function is enabled, HttpClientSession::attach_server_session(server_session) will be called.
    - If this feature is not enabled, call server_session->release() to put the HttpServerSession back into the connection pool.
  - 在 HttpSM::release_server_session() 中：
    - If this is provided by the Cache content response
    - then bind via HttpClientSession::attach_server_session(server_session, false)

Therefore, the function of HttpClientSession::attach_server_session is:

  - After HttpSM has processed a transaction,
  - If the current HttpServerSession is needed to continue processing the next transaction from HttpClientSession
  - Then bind the current HttpServerSession to the HttpClientSession
  - Need to pay attention to the difference with HttpSM::attach_server_session.

```
void
HttpClientSession::attach_server_session(HttpServerSession *ssession, bool transaction_done)
{
  if (ssession) {
    // The incoming ssession is not empty, indicating that you want to bind ssession to the current HttpClientSession
    ink_assert(bound_ss == NULL);
    // Set the state of HttpServerSession to have been bound to HttpClientSession
    ssession->state = HSS_KA_CLIENT_SLAVE;
    // Save the HttpServerSession to the bound_ss member
    bound_ss = ssession;
    DebugHttpSsn("[%" PRId64 "] attaching server session [%" PRId64 "] as slave", con_id, ssession->con_id);
    ink_assert(ssession->get_reader()->read_avail() == 0);
    ink_assert(ssession->get_netvc() != client_vc);

    // handling potential keep-alive here
    // This should only take effect for calls from HttpSM::tunnel_handler_server()
    // Is there a bug here for a call from httpSessionManager.acquire_session()?
    if (m_active) {
      m_active = false;
      HTTP_DECREMENT_DYN_STAT(http_current_active_client_connections_stat);
    }
    // Since this our slave, issue an IO to detect a close and
    //  have it call the client session back.  This IO also prevent
    //  the server net conneciton from calling back a dead sm
    // Can be associated with HttpServerSession, indicating support for the Keep alive feature, so directly set by state_keep_alive take over HttpServerSession
    SET_HANDLER(&HttpClientSession::state_keep_alive);
    slave_ka_vio = ssession->do_io_read(this, INT64_MAX, ssession->read_buffer);
    // Here is a bit strange, why should the state machine of HttpServerSession point to HttpClientSession?
    ink_assert (slave_ka_vio! = ka_vio);

    // Transfer control of the write side as well
    // Close the HttpServerSession data transmission, I feel this should be do_io_write (NULL, 0, NULL)
    ssession->do_io_write(this, 0, NULL);

    if (transaction_done) {
      // The default is true, which means that a transaction has been processed when the method is called.
      // Next, I will wait for the client to start the next transaction.
      // So set the inactivity timeout for the OS side keep alive timeout
      ssession->get_netvc()->set_inactivity_timeout(
        HRTIME_SECONDS(current_reader->t_state.txn_conf->keep_alive_no_activity_timeout_out));
      // cancel active timeout at the same time
      ssession->get_netvc()->cancel_active_timeout();
    } else {
      // we are serving from the cache - this could take a while.
      // Only when returning content from the Cache that requires user authentication to access
      // Since the time is returned from the Cache, the time required is uncontrollable, so temporarily cancel the NetVC timeout detection on the HttpServerSession
      // Since ATS has a caching mechanism, on a keep alive connection, there may be transactions (requests) that can hit the Cache, and some cannot hit.
      // Therefore, when hitting the Cache, the HttpServerSession needs to be maintained during the content provided by the Cache, and the timeout control must be temporarily turned off.
      ssession->get_netvc()->cancel_inactivity_timeout();
      ssession->get_netvc()->cancel_active_timeout();
    }
  } else {
    // The incoming ssession is empty, indicating that the information bound to the current HttpClientSession is to be cleared.
    ink_assert(bound_ss != NULL);
    bound_ss = NULL;
    slave_ka_vio = NULL;
  }
}
```

When a transaction (request) ends, HttpSM will call HttpClientSession::release() or HttpClientSession::do_io_close() depending on the situation.

  - If the client supports keep alive, then release() will be called.
  - Otherwise, do_io_close() will be called. 
  - Can refer to the code of HttpSM::tunnel_handler_ua() for analysis

```
// The release method simply disconnects the association between HttpClientSession and HttpSM
// But the relationship between HttpClientSession and HttpServerSession remains the same
// and will not actively close the session, but because the keep alive connection queue is full, it may result in forced shutdown
void
HttpClientSession::release(IOBufferReader *r)
{
  // The release call must be initiated by the upper state machine after completing a transaction.
  // So when calling release, the state of HttpClientSession must be HCS_ACTIVE_READER
  ink_assert(read_state == HCS_ACTIVE_READER);
  ink_assert(current_reader != NULL);
  // Get the timeout setting of the client's keep alive from HttpSM
  MgmtInt ka_in = current_reader->t_state.txn_conf->keep_alive_no_activity_timeout_in;

  DebugHttpSsn("[%" PRId64 "] session released by sm [%" PRId64 "]", con_id, current_reader->sm_id);
  // Unlink HttpClientSession and HttpSM
  current_reader = NULL;

  // handling potential keep-alive here
  // If the connection is active connection, then switch to inactive
  // Decrement the HTTP current active client connection counter at the same time
  if (m_active) {
    m_active = false;
    HTTP_DECREMENT_DYN_STAT(http_current_active_client_connections_stat);
  }
  // Make sure that the state machine is returning
  //  correct buffer reader
  // Fault tolerance check, requires that the correct IOBufferReader must be passed in
  ink_assert(r == sm_reader);
  if (r != sm_reader) {
    // Otherwise reduce the impact of this exception/problem by immediately calling do_io_close() to close the session
    this->do_io_close();
    return;
  }

  // Decrement the HTTP current concurrent client transaction counter
  HTTP_DECREMENT_DYN_STAT(http_current_client_transactions_stat);

  // Clean up the write VIO in case of inactivity timeout
  // Close the write operation on client_vc
  this->do_io_write(NULL, 0, NULL);

  // Check to see there is remaining data in the
  //  buffer.  If there is, spin up a new state
  //  machine to process it.  Otherwise, issue an
  //  IO to wait for new data
  if (sm_reader->read_avail() > 0) {
    // Check whether there is request data received from client_vc in the read buffer,
    // For example, in pipeline mode, there may already be another request in the read buffer.
    // Directly call new_transaction() to start processing the next request.
    DebugHttpSsn("[%" PRId64 "] data already in buffer, starting new transaction", con_id);
    new_transaction();
  } else {
    // If there is no data
    DebugHttpSsn("[%" PRId64 "] initiating io for next header", con_id);
    // HttpClientSession enters the HCS_KEEP_ALIVE state
    read_state = HCS_KEEP_ALIVE;
    // take over the data read on client_vc by state_keep_alive
    SET_HANDLER(&HttpClientSession::state_keep_alive);
    ka_vio = this->do_io_read(this, INT64_MAX, read_buffer);
    ink_assert (slave_ka_vio! = ka_vio);
    // Set the wait timeout (obtained by HttpSM)
    client_vc->set_inactivity_timeout(HRTIME_SECONDS(ka_in));
    // 取消 active timeout
    client_vc->cancel_active_timeout();
    // Put client_vc into the keep_alive connection queue of NetHandler
    client_vc->add_to_keep_alive_queue();
    // If the connection queue is full, client_vc will be closed immediately, state_keep_alive() will receive the TIMEOUT event, then call do_io_close()
  }
}
```

```
// do_io_close first disconnects the association between HttpClientSession and HttpServerSession
// Then put the HttpServerSession back into the connection pool and then perform the session shutdown process
void
HttpClientSession::do_io_close(int alerrno)
{
  // If the call is made after an HTTP request has been processed by HttpSM, the status value should be HCS_ACTIVE_READER
  // But the do_io_close call may come from inside the HttpClientSession, for example:
  // Received an EOS or TIMEOUT event while in the HCS_KEEP_ALIVE state
  if (read_state == HCS_ACTIVE_READER) {
    // Decrement the HTTP current concurrent client transaction counter
    HTTP_DECREMENT_DYN_STAT(http_current_client_transactions_stat);
    // If the connection is active connection, then switch to inactive
    // Decrement the HTTP current active client connection counter at the same time
    if (m_active) {
      m_active = false;
      HTTP_DECREMENT_DYN_STAT(http_current_active_client_connections_stat);
    }
  }

  // Prevent double closing
  // prevent multiple closed asserts
  ink_release_assert(read_state != HCS_CLOSED);

  // If we have an attached server session, release
  //   it back to our shared pool
  // Recycle HttpServerSession to the Session Pool
  if (bound_ss) {
    bound_ss->release();
    bound_ss = NULL;
    slave_ka_vio = NULL;
  }

  if (half_close && this->current_reader) {
    // Determine the execution of a half-close operation
    read_state = HCS_HALF_CLOSED;
    // handled by state_wait_for_close half closed
    SET_HANDLER(&HttpClientSession::state_wait_for_close);
    DebugHttpSsn("[%" PRId64 "] session half close", con_id);

    // We want the client to know that that we're finished
    //  writing.  The write shutdown accomplishes this.  Unfortuantely,
    //  the IO Core semantics don't stop us from getting events
    //  on the write side of the connection like timeouts so we
    //  need to zero out the write of the continuation with
    //  the do_io_write() call (INKqa05309)
    // Send a half-close to the client, the client can return a value of 0 by read (), determine the shutdown event
    client_vc->do_io_shutdown(IO_SHUTDOWN_WRITE);

    // Pay attention to the data on subsequent client_vc, but the data is directly discarded in state_wait_for_close
    ka_vio = client_vc->do_io_read(this, INT64_MAX, read_buffer);
    ink_assert (slave_ka_vio! = ka_vio);

    // [bug 2610799] Drain any data read.
    // If the buffer is full and the client writes again, we will not receive a
    // READ_READY event.
    // Directly discard the remaining unprocessed data from client_vc
    sm_reader->consume(sm_reader->read_avail());

    // Set the active timeout to the same as the inactive time so
    //   that this connection does not hang around forever if
    //   the ua hasn't closed
    // Set a timeout to prevent the client from shutting down for a long time
    client_vc->set_active_timeout(HRTIME_SECONDS(current_reader->t_state.txn_conf->keep_alive_no_activity_timeout_out));
  } else {
    // Otherwise perform a full shutdown 
    read_state = HCS_CLOSED;
    // clean up ssl's first byte iobuf
    // Clean up the iobuf of SSLNetVC
    SSLNetVConnection *ssl_vc = dynamic_cast<SSLNetVConnection *>(client_vc);
    if (ssl_vc) {
      ssl_vc->set_ssl_iobuf(NULL);
    }
    // Switch to HTTP/2, do not analyze
    if (upgrade_to_h2c) {
      Http2ClientSession *h2_session = http2ClientSessionAllocator.alloc();

      h2_session->set_upgrade_context(&current_reader->t_state.hdr_info.client_request);
      h2_session->new_connection(client_vc, NULL, NULL, false /* backdoor */);
      // Handed over control of the VC to the new H2 session, don't clean it up
      this->release_netvc();
      // TODO Consider about handling HTTP/1 hooks and stats
    } else {
      DebugHttpSsn("[%" PRId64 "] session closed", con_id);
    }
    // Count the average number of HTTP transactions per client connection
    HTTP_SUM_DYN_STAT(http_transactions_per_client_con, transact_count);
    // Decrement the HTTP current concurrent client connection counter
    HTTP_DECREMENT_DYN_STAT(http_current_client_connections_stat);
    conn_decrease = false;
    // trigger SSN CLOSE HOOK
    do_api_callout(TS_HTTP_SSN_CLOSE_HOOK);
  }
}
```

For the sake of analysis, we analyze it here without any Plugin Hook at SSN CLOSE:

  - Execute client_vc->do_io_close() to close NetVC
  - Execute release_netvc() to unlink HttpClientSession from client_vc
  - Call HttpClientSession::destroy() to release the object

```
void
HttpClientSession::destroy()
{
  DebugHttpSsn("[%" PRId64 "] session destroy", con_id);

  ink_release_assert(upgrade_to_h2c || !client_vc);
  ink_release_assert(bound_ss == NULL);
  ink_assert(read_buffer);

  // Set the magic value, indicating that the memory space occupied by the current object is no longer used
  magic = HTTP_CS_MAGIC_DEAD;
  // If NetVC has not been taken over by HttpSM, it will cause the read_buffer to be released here when it is closed in advance.
  if (read_buffer) {
    free_MIOBuffer(read_buffer);
    read_buffer = NULL;
  }

#ifdef USE_HTTP_DEBUG_LISTS
  ink_mutex_acquire(&debug_cs_list_mutex);
  debug_cs_list.remove(this, this->debug_link);
  ink_mutex_release(&debug_cs_list_mutex);
#endif

  // I feel that this should not be run...
  // must call do_io_close before calling release, which has already been processed in do_io_close.
  if (conn_decrease) {
    HTTP_DECREMENT_DYN_STAT(http_current_client_connections_stat);
    conn_decrease = false;
  }

  // transfer ProxyClientSession :: destroy ()
  super::destroy();
  // Release the memory space occupied by the object
  THREAD_FREE(this, httpClientSessionAllocator, this_thread());
}
```

## Understanding HttpClientSession with NetVConnection

Both HttpClientSession and NetVConnection inherit from the VConnection base class, and NetVConnection is a member of the HttpClientSession.

You can see that HttpClientSession basically transparently passes the do_io operation to the NetVConnection member.

If HttpClientSession is inherited from NetVConnection, is it possible to create an HttpClientSession directly when NetAccept accepts a new socket connection?

  - If you follow this design, ATS needs to provide a NetAccept inheritance class for each protocol, then it becomes:
    - HttpNetAccept，HttpClientSession，HttpSM
  - The current design is:
    - NetAccept, ProtocolSessionProbeAccept, ProtocolProbeTrampoline, HttpClientSession，HttpSM
  - It can be seen that the existing design can support multiple protocols more flexibly
  - Support for multiple operating system support for NetVConnection (UnixNetVConnection)

Therefore, HttpClientSession is a more specific, protocol-related NetVConnection that facilitates collaboration with HttpSM.

## TCP_INIT_CWND

CWND is the abbreviation of Congestion Window, and INIT CWND represents the initial congestion window.

Prior to Linux 3.0, the kernel's default initcwnd was small, calculated according to the value defined by RFC3390, by the value of MSS:

  - initcwnd = 4
    - MSS <= 1095
  - initcwnd = 3
    - 1095 < MSS <= 2190
  - initcwnd = 2
    - MSS> 2190

Typically, our MSS is 1460, so the initial congestion control window is 3.

```
/* Convert RFC3390 larger initial window into an equivalent number of packets. 
 * This is based on the numbers specified in RFC 6861, 3.1. 
 * /  
static inline u32 rfc3390_bytes_to_packets(const u32 smss)  
{  
    return smss <= 1095 ? 4 : (smss > 2190 ? 2 : 3);  
}  
```

After Linux 3.0, Google's advice was adopted to adjust the initial congestion control window to 10.

Google's advice：[An Argument for Increasing TCP's Initial Congestion Window](http://code.google.com/speed/articles/tcp_initcwnd_paper.pdf)

  - The recommended value of initcwnd is 10*MSS.

Originally proposed in October 2010 [IETF DRAFT] (http://tools.ietf.org/html/draft-ietf-tcpm-initcwnd-00), after nearly 10 revisions, in April 2013 Become [RFC 6928] (https://tools.ietf.org/html/rfc6928).

However, as a TCP Server, it is not advisable to set the initcwnd to 10 for users with different delays and bandwidths. Because many clients accessing the Internet through mobile phones have narrow bandwidth, they will cause serious network data. congestion.

Therefore, ATS created [TS-940] (https://issues.apache.org/jira/browse/TS-940) to implement a configurable parameter:

  - proxy.config.http.server_tcp_init_cwnd

At the same time, in order to support condition-based, different values ​​of initcwnd are set for different clients, and remap is also used to dynamically modify each connection.

The TS-940 code submission is divided into two parts:

  - https://github.com/apache/trafficserver/commit/0e4814487769a203ea3aa0522374798ec60dfe4c
    - Set the value of initcwnd for listen fd in UnixNetProcessor::accept_internal
    - This setting should only be supported for Solaris 10+ versions
    - This setting is valid for all accept connections
  - https://github.com/apache/trafficserver/commit/341a2c5aeac4a3073c5c84ea410455194cc909ab
    - This is the value of initcwnd set for the current socket connection before the first write operation in HttpClientSession::do_io_write
    - Before the first do_io_write call, the HTTP request has been read from the client, so remap has already been executed.
    - Dynamic modification of the currently connected initcwnd via the remap plugin conf_remap
    - Related InkAPI:
      - TSHttpTxnConfigIntSet(TSHttpTxn txnp, TS_CONFIG_HTTP_SERVER_TCP_INIT_CWND, value)


## References

- [HttpClientSession.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpClientSession.h)
- [HttpClientSession.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpClientSession.cc)
- [TCP_INIT_CWND related] (http://kb.cnblogs.com/page/209101/)
