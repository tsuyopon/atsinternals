# core component: HttpServerSession

HttpServerSession is similar to HttpClientSession and also contains members of NetVConnection, an extension to NetVConnection.

HttpServerSession inherits directly from the VConnection base class. There is no intermediate class inheritance class in ProxServerSession in ATS.

Same as all VConnection inheritance classes:

  - Offered do_io _ * ()
  - provide reenable () way

Passing a NetVConnection in the HttpServerSession::new_connection method requires less parameters than HttpClientSession::new_connection

  - The incoming NetVConnection object is saved to the HttpServerSession::server_vc member

What's special is that HttpServerSession is not a state machine, it does not define any event callback function.

  - But HttpClientSession::state_state_keep_alive is specifically for HttpServerSession

Therefore, HttpServerSession is subordinate to HttpClientSession

  - After the first HTTP request is parsed by HttpSM, if you need to connect to OServer
    - an HttpServerSession object is created after the NetVConnection connection is successful
  - The events on the HttpServerSession are now handled by HttpSM
  - After sending the response corresponding to the first HTTP request to the client, HttpSM will pass the HttpServerSession to the HttpClientSession for management, and HttpSM will release itself.
    - Events on Client NetVConnection are handled by HttpClientSession
    - Events on OServer NetVConnection are also handled by HttpClientSession
    - At this point on a TCP connection, after the first HTTP request is over, the second HTTP request has not yet arrived.
  - After the second HTTP request is parsed by HttpSM, if you need to connect to OServer
    - will request the HttpServerSession used before to HttpClientSession, continue to reuse if it exists
  - Then the new HttpSM will take over the events on the HttpServerSession
  - Repeat this way...

If the HttpClientSession is passively closed during the "gap", or the HttpServerSession has a Keep alive timeout

  - HttpServerSession will be taken over by ServerSessionPool
  - ServerSessionPool will set a new timeout

HttpServerSession will eventually close

  - Timeout to reach the ServerSessionPool setting
  - OServer actively closes the TCP connection

## definition

`` `
class HttpServerSession : public VConnection
{
public:
  // Constructor
  HttpServerSession()
    : VConnection(NULL), hostname_hash(), con_id(0), transact_count(0), state(HSS_INIT), to_parent_proxy(false),
      server_trans_stat(0), private_session(false), sharing_match(TS_SERVER_SESSION_SHARING_MATCH_BOTH),
      sharing_pool(TS_SERVER_SESSION_SHARING_POOL_GLOBAL), enable_origin_connection_limiting(false), connection_count(NULL),
      read_buffer(NULL), server_vc(NULL), magic(HTTP_SS_MAGIC_DEAD), buf_reader(NULL)
  {
    ink_zero(server_ip);
  }

  // destroy itself
  void destroy();
  // accept the new NetVConnection
  void new_connection(NetVConnection *new_vc);

  // Reset MIOBuffer
  void
  reset_read_buffer(void)
  {
    ink_assert(read_buffer->_writer);
    ink_assert(buf_reader != NULL);
    // release all IOBufferReader
    // In addition to the IOBufferBlock currently being written, the previously unread IOBufferBlock will be automatically released.
    read_buffer->dealloc_all_readers();
    // Release the IOBufferBlock that is currently being written
    read_buffer->_writer = NULL;
    // Reassign an IOBufferReader
    buf_reader = read_buffer->alloc_reader();
  }

  // Get IOBufferReader
  IOBufferReader *
  get_reader()
  {
    return buf_reader;
  };

  // pass to netvc's do_io_read
  virtual VIO *do_io_read(Continuation *c, int64_t nbytes = INT64_MAX, MIOBuffer *buf = 0);
  // pass to net_do_do_write
  virtual VIO *do_io_write(Continuation *c = NULL, int64_t nbytes = INT64_MAX, IOBufferReader *buf = 0, bool owner = false);
  // In addition to calling dovc_do_close, the HttpServerSession itself is destroyed.
  virtual void do_io_close(int lerrno = -1);
  // pass to the netvc do_io_shutdown
  virtual void do_io_shutdown(ShutdownHowTo_t howto);
  // passable to netvc's reenable
  virtual void reenable(VIO *vio);

  // Decouple from HttpSM or HttpClientSession and put itself into ServerSessionPool
  // If it is a private session, or if the sharing function of ServerSession is turned off, then directly call do_io_close
  void release();
  // Calculate the MD5 value for the given hostname and save it to the member hostname_hash
  void attach_hostname(const char *hostname);
  // Get member server_vc
  NetVConnection *
  get_netvc()
  {
    return server_vc;
  };

  // Keys for matching hostnames
  // the credentials used to find in the SessionPool
  // can be based on the OServer IP, the requested Hostname to find
  IpEndpoint server_ip;
  INK_MD5 hostname_hash;

  // Unique ID of HttpServerSession
  int64_t con_id;
  // The total number of transactions/requests processed on this HttpServerSession
  int transact_count;
  // The status of the HttpServerSession
  HSS_State state;

  // Used to determine whether the session is for parent proxy
  // it is session to orgin server
  // We need to determine whether a closed connection was to
  // close parent proxy to update the
  // proxy.process.http.current_parent_proxy_connections
  // Determine if this is an HttpServerSession connected to the Parent Proxy
  // Used to assist in the statistics of the current number of ATS and Parent Proxy connections
  bool to_parent_proxy;

  // Used to verify we are recording the server
  //   transaction stat properly
  // The value is 0 means HttpServerSession is not managed by HttpSM
  // The value is 1 to indicate that the HttpServerSession is being taken over by HttpSM
  // other values are abnormal
  int server_trans_stat;

  // Sessions become if authentication headers
  //  are sent over them
  // used to mark a private session, usually means that the session has HTTP authentication header information
  bool private_session;

  // Copy of the owning SM's server session sharing settings
  // Copy the session sharing settings in the current HttpSM in HttpSM::state_http_server_open
  TSServerSessionSharingMatchType sharing_match; // Matching method (None, IP, Host, Both)
  TSServerSessionSharingPoolType sharing_pool; // Use global session pool or thread local session pool
  //  int share_session;

  LINK(HttpServerSession, ip_hash_link);
  LINK(HttpServerSession, host_hash_link);

  // Keep track of connection limiting and a pointer to the
  // singleton that keeps track of the connection counts.
  // Control the TCP connection to the OServer
  bool enable_origin_connection_limiting;
  ConnectionCount *connection_count;

  // The ServerSession owns the following buffer which use
  //   for parsing the headers.  The server session needs to
  //   own the buffer so we can go from a keep-alive state
  //   to being acquired and parsing the header without
  //   changing the buffer we are doing I/O on.  We can
  //   not change the buffer for I/O without issuing a
  //   an asyncronous cancel on NT
  // HTTP Response Header used to receive OServer responses
  // used to receive possible invalid data in the "gap"
  MIOBuffer *read_buffer;

private:
  HttpServerSession(HttpServerSession &);

  NetVConnection *server_vc;
  int magic;

  // Keep the data in read_buffer will not be freely released
  // The data in read_buffer is only released when reset_read_buffer is executed
  IOBufferReader *buf_reader;
};
`` `

## Status (State)

Four state values are defined in the definition of HttpServerSession by an enumerated type to indicate the state of the HttpServerSession:

  - HSS_INIT
    - initial state, set to this state in the constructor
    - will be set to this state again in new_connection
  - HSS_ACTIVE
    - Set to this state immediately after HttpSM calls new_connection return
    - Set to this state after HttpSM gets the HttpServerSession from HttpClientSession
    - After the HttpSessionManager gets the HttpServerSession from the ServerSessionPool, it is set to this state
  - HSS_KA_CLIENT_SLAVE
    - indicates that HttpServerSession is being taken over by HttpClientSession
    - When HttpServerSession encounters a timeout, it will be placed in ServerSessionPool
  - HSS_KA_SHARED
    - indicates that HttpServerSession is being taken over by ServerSessionPool

## Method

There are fewer methods for HttpServerSession, which has a lot to do with it as a dependency of HttpClientSession.

An HttpServerSession object is managed by three state machines throughout its lifecycle:

  - HttpClientSession
  - HttpSM
  - ServerSessionPool

So, there is no event handler for itself. The interface that needs to be provided is basically the do_io series of VConnection, and the following.

Before HttpSM initiates a connection to OServer, it will check whether there is a bound HttpServerSession on HttpClientSession.
It also compares whether the current requested target OServer matches the already bound HttpServerSession. If it does not match:

  - Call HttpServerSession::release to unlink HttpServerSession from HttpSM
  - Then call HttpClientSession::attach_server_session to unbind the HttpServerSession from the HttpClientSession
  - Then initiate a connection to OServer

Callback NET_EVENT_OPEN event to HttpSM::state_http_server_open after the connection is successfully established

`` `
// Pass in the NetVConnection that has completed the TCP connection establishment
void
HttpServerSession::new_connection(NetVConnection *new_vc)
{
  ink_assert(new_vc != NULL);
  // Save NetVC to server_vc member
  server_vc = new_vc;

  // Used to do e.g. mutex = new_vc->thread->mutex; when per-thread pools enabled
  // Use NetVC's mutex to set the mutex of HttpServerSession
  mutex = new_vc->mutex;

  // Unique client session identifier.
  // Generate a unique ID of HttpServerSession, used to track in the Debug information
  con_id = ink_atomic_increment((int64_t *)(&next_ss_id), 1);

  // Set the magic value to find memory allocation error
  magic = HTTP_SS_MAGIC_ALIVE;
  // Update the current source server concurrent connections
  HTTP_SUM_GLOBAL_DYN_STAT(http_current_server_connections_stat, 1); // Update the true global stat
  // Update the total number of source server connections
  HTTP_INCREMENT_DYN_STAT(http_total_server_connections_stat);
  // Check to see if we are limiting the number of connections
  // by host
  // If the connection limit to the source server is turned on
  // Before HttpSM initiates a TCP connection to the source server, it first determines the number of connections on the specified server_ip.
  // If the maximum value origin_max_connections has been exceeded, reschedule HttpSM after a period of time
  // When the ServerSessionPool receives the timeout event of HttpServerSession, it will determine the number of connections specified by server_ip.
  // If you have exceeded origin_min_keep_alive_connections, close HttpServerSession directly
  if (enable_origin_connection_limiting == true) {
    if (connection_count == NULL)
      connection_count = ConnectionCount::getInstance();
    // The number of connections to the server_ip is incremented
    connection_count->incrementCount(server_ip);
    char addrbuf [INET6_ADDRSTRLEN];
    Debug("http_ss", "[%" PRId64 "] new connection, ip: %s, count: %u", con_id,
          ats_ip_ntop(&server_ip.sa, addrbuf, sizeof(addrbuf)), connection_count->getCount(server_ip));
  }
  // Create MIOBuffer
#ifdef LAZY_BUF_ALLOC
  read_buffer = new_empty_MIOBuffer(HTTP_SERVER_RESP_HDR_BUFFER_INDEX);
#else
  read_buffer = new_MIOBuffer(HTTP_SERVER_RESP_HDR_BUFFER_INDEX);
#endif
  // Assign IOBufferReader to MIOBuffer
  buf_reader = read_buffer->alloc_reader();
  Debug("http_ss", "[%" PRId64 "] session born, netvc %p", con_id, new_vc);
  // Set the status
  state = HSS_INIT;
}
`` `

After calling new_connection in HttpSM::state_http_server_open,

  - Continue to call HttpSM::attach_server_session
    - Set the callback function of HttpServerSession to HttpSM::state_send_server_request_header
    - Receive data via do_io_read
    - Turn off data transmission via do_io_write
  - Then call HttpSM::handle_http_server_open to complete the operation of sending the request to OServer
    - If the client is an HTTP POST request, there may be a change in the process

HttpSM will first process the response header of OServer, and then it will create the content returned by HttpTunnel, and the following events of HttpServerSession:

  - VC_EVENT_EOS
  - VC_EVENT_ERROR
  - VC_EVENT_READ_COMPLETE
  - VC_EVENT_ACTIVE_TIMEOUT
  - VC_EVENT_INACTIVITY_TIMEOUT

Take over by HttpSM::tunnel_handler_server.

The following judgment is made in HttpSM::tunnel_handler_server:

  - If you have received an EOS event,
    - Then call HttpServerSession::do_io_close to close.
  - If connection multiplexing of OServer is not allowed,
    - Then call HttpClientSession::attach_server_session to attach the HttpServerSession to the HttpClientSession.
  - If connection multiplexing is allowed,
    - Set the keep alive timeout and then call HttpServerSession::release to detach from HttpSM

`` `
// Close and release HttpServerSession
void
HttpServerSession::do_io_close(int alerrno)
{
  // If it is initiated from HttpSM
  if (state == HSS_ACTIVE) {
    // Update the number of concurrent transactions for the current source server. This value is incremented in HttpSM::attach_server_session
    HTTP_DECREMENT_DYN_STAT(http_current_server_transactions_stat);
    // The number of references to HttpServerSession is reduced
    this->server_trans_stat--;
  }

  Debug("http_ss", "[%" PRId64 "] session closing, netvc %p", con_id, server_vc);

  // close NetVConnection
  server_vc->do_io_close(alerrno);
  // Unlink from NetVConnection
  server_vc = NULL;

  // Update the number of concurrent connections to the current source server
  HTTP_SUM_GLOBAL_DYN_STAT(http_current_server_connections_stat, -1); // Make sure to work on the global stat
  // Count the average number of transactions completed on each source server
  HTTP_SUM_DYN_STAT(http_transactions_per_server_con, transact_count);

  // Check to see if we are limiting the number of connections
  // by host
  // If the connection limit to the source server is turned on
  if (enable_origin_connection_limiting == true) {
    if (connection_count->getCount(server_ip) > 0) {
      // The number of connections to the server_ip is reduced
      connection_count->incrementCount(server_ip, -1);
      char addrbuf [INET6_ADDRSTRLEN];
      Debug("http_ss", "[%" PRId64 "] connection closed, ip: %s, count: %u", con_id,
            ats_ip_ntop(&server_ip.sa, addrbuf, sizeof(addrbuf)), connection_count->getCount(server_ip));
    } else {
      Error("[%" PRId64 "] number of connections should be greater than zero: %u", con_id, connection_count->getCount(server_ip));
    }
  }

  // If it is connected to the Parent Proxy
  if (to_parent_proxy) {
    // Update the current number of concurrent connections to the Parent Proxy, which is incremented in HttpSM::state_http_server_open
    HTTP_DECREMENT_DYN_STAT(http_current_parent_proxy_connections_stat);
  }
  // Release resources, recycle object memory
  destroy();
}
`` `

As you can see, as long as HttpServerSession::do_io_close is called, the NetVConnection will be closed and the object will be released. If you need to leave the HttpServerSession for reuse, you need to pass:

  - HttpServerSession::release
    - Put in ServerSessionPool
  - HttpClientSession::attach_server_session
    - Associated to HttpClientSession

`` `
void
HttpServerSession::release()
{
  Debug("http_ss", "Releasing session, private_session=%d, sharing_match=%d", private_session, sharing_match);
  // Set our state to KA for stat issues
  state = HSS_KA_SHARED;

  // Private sessions are never released back to the shared pool
  // If it is a private session, or the ServerSessionPool function is turned off in the configuration file
  // Then call do_io_close directly to close
  if (private_session || TS_SERVER_SESSION_SHARING_MATCH_NONE == sharing_match) {
    this->do_io_close();
    return;
  }

  // Put HttpServerSession into ServerSessionPool via HttpSessionManager
  HSMresult_t r = httpSessionManager.release_session(this);

  // According to the return value to determine whether the successful placement
  if (r == HSM_RETRY) {
    // Session could not be put in the session manager
    //  due to lock contention
    // Since the lock of ServerSessionPool was not taken, the insert failed and needs to be retried again
    // FIX:  should retry instead of closing
    // But there is currently no mechanism to implement retry, so call do_io_close directly to close it.
    // The logic of retrying needs to be implemented in the future.
    this->do_io_close();
  } else {
    // The session was successfully put into the session
    //    manager and it will manage it
    // successfully placed into the ServerSessionPool
    // (Note: should never get HSM_NOT_FOUND here)
    // r should now equal HSM_DONE
    ink_assert(r == HSM_DONE);
  }
}
`` `

The destroy method for releasing and reclaiming resources

`` `
void
HttpServerSession::destroy()
{
  ink_release_assert(server_vc == NULL);
  ink_assert(read_buffer);
  ink_assert(server_trans_stat == 0);
  // set magic
  magic = HTTP_SS_MAGIC_DEAD;
  // release MIOBuffer
  if (read_buffer) {
    free_MIOBuffer(read_buffer);
    read_buffer = NULL;
  }

  // clean up mutex
  mutex.clear();
  // Reclaim the memory occupied by the object
  if (TS_SERVER_SESSION_SHARING_POOL_THREAD == sharing_pool)
    THREAD_FREE(this, httpServerSessionAllocator, this_thread());
  else
    httpServerSessionAllocator.free(this);
}
`` `

## References

- [HttpServerSession.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpServerSession.h)
- [HttpServerSession.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpServerSession.cc)
