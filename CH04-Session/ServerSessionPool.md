# core component: ServerSessionPool

## definition

`` `
source: proxy/http/HttpProxyAPIEnums.h
/// Server session sharing values - match
typedef enum {
  TS_SERVER_SESSION_SHARING_MATCH_NONE, // does not support getting sessions from the session pool
  TS_SERVER_SESSION_SHARING_MATCH_BOTH, // must match both IP and hostname
  TS_SERVER_SESSION_SHARING_MATCH_IP, // only matches IP
  TS_SERVER_SESSION_SHARING_MATCH_HOST // match only hostname
} TSServerSessionSharingMatchType;

/// Server session sharing values - pool
typedef enum {
  TS_SERVER_SESSION_SHARING_POOL_GLOBAL, // share the session pool across threads,
  TS_SERVER_SESSION_SHARING_POOL_THREAD, // shared session pool within the thread
} TSServerSessionSharingPoolType;
`` `

`` `
/** A pool of server sessions.

    This is a continuation so that it can get callbacks from the server sessions.
    This is used to track remote closes on the sessions so they can be cleaned up.
    Inherited from Continuation so that it can be called back by ServerSession.
    ServerSessionPool is used to track the shutdown of the source server session, so that the occupied memory can be cleaned up in time.
    
    @internal Cleanup is the real reason we will always need an IP address mapping for the
    sessions. The I/O callback will have only the NetVC and thence the remote IP address for the
    closed session and we need to be able find it based on that.
    Due to the callback of the do_io operation, only the NetVConnection is passed, and we can only get the IP of the source server from it.
    Therefore, we need to establish a mapping relationship between the source server IP address and the session, in order to complete the cleanup work.
* /
class ServerSessionPool : public Continuation
{
public:
  /// Default constructor.
  /// Constructs an empty pool.
  // constructor to create an empty session pool
  ServerSessionPool();
  /// Handle events from server sessions.
  / / Handle callbacks from the session, usually abnormal data, timeout, connection closed
  int eventHandler(int event, void *data);

protected:
  /// Interface class for IP map.
  / / IP address and session mapping table method
  struct IPHashing {
    typedef uint32_t ID;
    typedef sockaddr const *Key;
    typedef HttpServerSession Value;
    typedef DList(HttpServerSession, ip_hash_link) ListHead;

    static ID
    hash(Key key)
    {
      return ats_ip_hash(key);
    }
    static Key
    key(Value const *value)
    {
      return &value->server_ip.sa;
    }
    static bool
    equal(Key lhs, Key rhs)
    {
      return ats_ip_addr_port_eq(lhs, rhs);
    }
  };

  /// Interface class for FQDN map.
  // Method of mapping the host name to the session
  struct HostHashing {
    typedef uint64_t ID;
    typedef INK_MD5 const &Key;
    typedef HttpServerSession Value;
    typedef DList(HttpServerSession, host_hash_link) ListHead;

    static ID
    hash(Key key)
    {
      return key.fold();
    }
    static Key
    key(Value const *value)
    {
      return value->hostname_hash;
    }
    static bool
    equal(Key lhs, Key rhs)
    {
      return lhs == rhs;
    }
  };

  // Combine TSHashTable with the corresponding method through the template
  typedef TSHashTable<IPHashing> IPHashTable;     ///< Sessions by IP address.
  typedef TSHashTable<HostHashing> HostHashTable; ///< Sessions by host name.

public:
  /** Check if a session matches address and host name.
      Determine if a session matches the IP address and hostname
   * /
  static bool match(HttpServerSession *ss, sockaddr const *addr, INK_MD5 const &host_hash,
                    TSServerSessionSharingMatchType match_style);

  /** Get a session from the pool.
      Get a conversation from the session pool

      The session is selected based on @a match_style equivalently to @a match. If found the session
      is removed from the pool.
      According to the matching method specified by match_style, call the match method to select the appropriate session from the session pool.
      Once the session is selected, it is removed from the session pool.

      @return A pointer to the session or @c NULL if not matching session was found.
      Server_session points to a pointer to the selected session, or points to NULL to indicate that no suitable session was found.
      This is a function return value of type HSMresult_t, but there is no code to judge it.
      !!! In the current code, always return HSM_NOT_FOUND!!!
              
  * /
  HSMresult_t acquireSession(sockaddr const *addr, INK_MD5 const &host_hash, TSServerSessionSharingMatchType match_style,
                             HttpServerSession *&server_session);
  /** Release a session to to pool.
      Put a conversation back into the session pool
   * /
  void releaseSession(HttpServerSession *ss);

  /// Close all sessions and then clear the table.
  // Close all sessions and clear the IP mapping table and hostname mapping table
  void purge();

  // Pools of server sessions.
  // Note that each server session is stored in both pools.
  // IP mapping table (pool)
  IPHashTable m_ip_pool;
  // hostname mapping table (pool)
  HostHashTable m_host_pool;
  // Note that each session will exist in both tables
};
`` `

## Method

### ServerSessionPool::ServerSessionPool

`` `
ServerSessionPool::ServerSessionPool() : Continuation(new_ProxyMutex()), m_ip_pool(1023), m_host_pool(1023)
{
  SET_HANDLER(&ServerSessionPool::eventHandler);
  m_ip_pool.setExpansionPolicy(IPHashTable::MANUAL);
  m_host_pool.setExpansionPolicy(HostHashTable::MANUAL);
}
`` `

### initialize_thread_for_http_sessions

`` `
source: proxy/http/HttpSessionManager.cc
// Initialize a thread to handle HTTP session management
void
initialize_thread_for_http_sessions(EThread *thread, int /* thread_index ATS_UNUSED */)
{
  thread->server_session_pool = new ServerSessionPool;
}
`` `

In UnixNetProcessor::start(), initialize_thread_for_http_sessions is called to create a ServerSessionPool object for each thread.

`` `
int
UnixNetProcessor::start(int, size_t)
{
  EventType etype = ET_NET;

  netHandler_offset = eventProcessor.allocate (sizeof (NetHandler));
  pollCont_offset = eventProcessor.allocate(sizeof(PollCont));

  // etype is ET_NET for netProcessor
  // and      ET_SSL for sslNetProcessor
  upgradeEtype(etype);

  n_netthreads = eventProcessor.n_threads_for_type[etype];
  netthreads = eventProcessor.eventthread[etype];
  for (int i = 0; i < n_netthreads; ++i) {
    initialize_thread_for_net(netthreads[i]);                                                                                        
    extern void initialize_thread_for_http_sessions(EThread * thread, int thread_index);
    initialize_thread_for_http_sessions(netthreads[i], i); 
  }
...
}
`` `

This can actually be understood as the use of netProcessor to do httpProcessor, but httpProcessor is not defined in ATS.


### ServerSessionPool::match

`` `
// used to determine if a ServerSession matches the specified Origin Server IP and hostname information
bool
ServerSessionPool::match(HttpServerSession *ss, sockaddr const *addr, INK_MD5 const &hostname_hash,
                         TSServerSessionSharingMatchType match_style)
{
  return TS_SERVER_SESSION_SHARING_MATCH_NONE != match_style && // if no matching allowed, fail immediately.
         // The hostname matches if we're not checking it or it (and the port!) is a match.
         (TS_SERVER_SESSION_SHARING_MATCH_IP == match_style ||
          (ats_ip_port_cast(addr) == ats_ip_port_cast(ss->server_ip) && ss->hostname_hash == hostname_hash)) &&
         // The IP address matches if we're not checking it or it is a match.
         (TS_SERVER_SESSION_SHARING_MATCH_HOST == match_style || ats_ip_addr_port_eq(ss->server_ip, addr));
}
`` `

Use match_style to represent the matching rule and convert the above content to the syntax of if-else:

`` `
  if (TS_SERVER_SESSION_SHARING_MATCH_NONE == match_style) {
    // If nothing matches, return false directly
    return false;
  }
  
  // If it is the matching mode of IP,
  if (TS_SERVER_SESSION_SHARING_MATCH_IP == match_style ) {
    / / Determine the IP and PORT are equal, return true, otherwise return flase
    if (ats_ip_addr_port_eq(ss->server_ip, addr)) {
      return true;
    } else {
      return false;
    }
  // Otherwise (not the IP match mode, it might be: MATCH_HOST, MATCH_BOTH)
  } else if (ats_ip_port_cast(addr) == ats_ip_port_cast(ss->server_ip) && ss->hostname_hash == hostname_hash) {
    / / If the match is the port number + host name hash value (has already met the conditions of MATCH_HOST)
    // ats_ip_port_cast(addr) returns the port number
    // The IP address matches if we're not checking it or it is a match.
    / / Determine the matching condition is MATCH_HOST, then return true directly
    if (TS_SERVER_SESSION_SHARING_MATCH_HOST == match_style) {
      return true;
    // Otherwise it is the case of MATCH_BOTH, you need to continue to judge whether IP and PORT are equal.
    } else if (ats_ip_addr_port_eq(ss->server_ip, addr)) {
      // equal, return true if the condition of MATCH_BOTH is met, false otherwise
      return true;
    } else {
      return false;
    }
  // is neither an IP match mode nor a condition for MATCH_HOST
  // So, definitely can't match, return false directly
  } else {
    return false;
  }
`` `

### ServerSessionPool::acquireSession

`` `
// Incoming the IP information and hostname information of the Origin Server, and the mode you want to match
// Returns the ServerSession found from the session pool by to_return according to the specified match pattern
// Return value: always HSM_NOT_FOUND
HSMresult_t
ServerSessionPool::acquireSession(sockaddr const *addr, INK_MD5 const &hostname_hash, TSServerSessionSharingMatchType match_style,
                                  HttpServerSession *&to_return)
{
  HSMresult_t zret = HSM_NOT_FOUND;
  if (TS_SERVER_SESSION_SHARING_MATCH_HOST == match_style) {
    // MATCH_HOST mode
    // This is broken out because only in this case do we check the host hash first.
    // look directly from m_host_pool
    HostHashTable::Location loc = m_host_pool.find(hostname_hash);
    / / Get the port number from the IP data structure
    in_port_t port = ats_ip_port_cast(addr);
    // Since the same hostname may have multiple ports, a ServerSession session matching the port must be found in the hash bucket
    while (loc && port != ats_ip_port_cast(loc->server_ip))
      ++loc; // scan for matching port.
    // If found
    if (loc) {
      // return the found ServerSession
      to_return = loc;
      // Remove the ServerSession from the session pool at the same time
      // Note: here are two session pools
      m_host_pool.remove(loc);
      m_ip_pool.remove(m_ip_pool.find(loc));
    }
  } else if (TS_SERVER_SESSION_SHARING_MATCH_NONE != match_style) { // matching is not disabled.
    // Otherwise, judge the mode that is not MATCH_NONE, it may be: MATCH_IP, MATCH_BOTH
    // Look directly from m_ip_pool, where the match is based on both IP and port
    IPHashTable::Location loc = m_ip_pool.find(addr);
    // If we're matching on the IP address we're done, this one is good enough.
    // Otherwise we need to scan further matches to match the host name as well.
    // Note we don't have to check the port because it's checked as part of the IP address key.
    // If it is MATCH_BOTH mode
    if (TS_SERVER_SESSION_SHARING_MATCH_IP != match_style) {
      // Continue to determine if the hostname_hash value matches the found ServerSession
      // because there may be more than one domain name on an IP
      while (loc && loc->hostname_hash != hostname_hash)
        ++ place;
    }
    // If found
    if (loc) {
      // return the found ServerSession
      to_return = loc;
      // Remove the ServerSession from the session pool at the same time
      // Note: here are two session pools
      m_ip_pool.remove(loc);
      m_host_pool.remove(m_host_pool.find(loc));
    }
  }
  // always returns HSM_NOT_FOUND
  return zret;
}
`` `

### ServerSessionPool::releaseSession

`` `
// Put the ServerSession into the session pool
void
ServerSessionPool::releaseSession(HttpServerSession *ss)
{
  // Set the state to indicate that the session was taken over by the session pool
  ss->state = HSS_KA_SHARED;
  // Now we need to issue a read on the connection to detect
  //  if it closes on us.  We will get called back in the
  //  continuation for this bucket, ensuring we have the lock
  //  to remove the connection from our lists
  // Receive EOS events via do_io_read
  ss->do_io_read(this, INT64_MAX, ss->read_buffer);

  // Transfer control of the write side as well
  // Also take over the write operation
  ss->do_io_write(this, 0, NULL);

  // we probably don't need the active timeout set, but will leave it for now
  / / Set the timeout, there may be no need to set active timeout, but temporarily reserved
  ss->get_netvc()->set_inactivity_timeout(ss->get_netvc()->get_inactivity_timeout());
  ss->get_netvc()->set_active_timeout(ss->get_netvc()->get_active_timeout());
  // put it in the pools.
  // put two session pools at the same time
  m_ip_pool.insert(ss);
  m_host_pool.insert(ss);

  / / Output the successful debugging information
  Debug("http_ss", "[%" PRId64 "] [release session] "
                   "session placed into shared pool",
        ss->con_id);
}
`` `

### ServerSessionPool::eventHandler

`` `
// Session pool event handler
int
ServerSessionPool::eventHandler(int event, void *data)
{
  NetVConnection *net_vc = NULL;
  HttpServerSession *s = NULL;

  switch (event) {
  case VC_EVENT_READ_READY:
  // The server sent us data.  This is unexpected so
  //   close the connection
  // Received data from Origin Server, this is exception data and should close the connection.
  // There is no immediate shutdown here, it is normal, continue to look down.
  /* Fall through */
  case VC_EVENT_EOS:
  case VC_EVENT_ERROR:
  case VC_EVENT_INACTIVITY_TIMEOUT:
  case VC_EVENT_ACTIVE_TIMEOUT:
    // Get NetVConnection
    net_vc = static_cast<NetVConnection *>((static_cast<VIO *>(data))->vc_server);
    break;

  default:
    ink_release_assert(0);
    return 0;
  }

  sockaddr const *addr = net_vc->get_remote_addr();
  // Get the http_config_params object
  HttpConfigParams *http_config_params = HttpConfig::acquire();
  bool found = false;

  // Since the hash table of the IP address session pool is looked up according to IP+ port or ServerSession,
  // But there is only a NetVC object here, so I can only find it based on the IP+ port.
  // But there may be more connections to the same Origin Server, so you need to traverse to find the ServerSession associated with this NetVC
  for (ServerSessionPool::IPHashTable::Location lh = m_ip_pool.find(addr); lh; ++lh) {
    // If you find a ServerSession corresponding to NetVC
    if ((s = lh)->get_netvc() == net_vc) {
      // if there was a timeout of some kind on a keep alive connection, and
      // keeping the connection alive will not keep us above the # of max connections
      // to the origin and we are below the min number of keep alive connections to this
      // origin, then reset the timeouts on our end and do not close the connection
      // If it is a timeout event (going out of the timeout event, others are experiencing serious errors, need to close NetVC)
      // and ServerSession is currently managed by the session pool
      // and the session has source port connection restrictions turned on
      if ((event == VC_EVENT_INACTIVITY_TIMEOUT || event == VC_EVENT_ACTIVE_TIMEOUT) && s->state == HSS_KA_SHARED &&
          s->enable_origin_connection_limiting) {
        / / Determine whether the current server is below the minimum keep alive connection to maintain the limit
        bool connection_count_below_min =
          s->connection_count->getCount(s->server_ip) <= http_config_params->origin_min_keep_alive_connections;

        // If the minimum keep alive connection is below the minimum limit
        // I need to keep this connection, even if it times out, it won't close it.
        // After resetting the timeout, keep it in the session pool
        if (connection_count_below_min) {
          Debug("http_ss", "[%" PRId64 "] [session_bucket] session received io notice [%s], "
                           "reseting timeout to maintain minimum number of connections",
                s->con_id, HttpDebugNames::get_event_name(event));
          s->get_netvc()->set_inactivity_timeout(s->get_netvc()->get_inactivity_timeout());
          s->get_netvc()->set_active_timeout(s->get_netvc()->get_active_timeout());
          // set to already found, then jump out of the for loop
          found = true;
          break;
        }
      }
      // If it is not a timeout event, it is a serious error or the other party closes the connection.
      // Or, although it is a timeout event, a keep alive connection is enough
      // Then, remove this NetVConnection from the session pool and then close it
      // We've found our server session. Remove it from
      //   our lists and close it down
      Debug("http_ss", "[%" PRId64 "] [session_pool] session %p received io notice [%s]", s->con_id, s,
            HttpDebugNames::get_event_name(event));
      ink_assert(s->state == HSS_KA_SHARED);
      // Out of the pool! Now!
      m_ip_pool.remove(lh);
      m_host_pool.remove(m_host_pool.find(s));
      // Drop connection on this end.
      s->do_io_close();
      // set to already found, then jump out of the for loop
      found = true;
      break;
    }
    // If NetVC does not correspond to ServerSession, continue for loop traversal
  }

  // release the http_config_params object
  HttpConfig::release(http_config_params);
  if (!found) {
    // If the ServerSession is not found, there is a connection leak.
    // We failed to find our session.  This can only be the result
    //  of a programming flaw
    Warning("Connection leak from http keep-alive system");
    ink_assert(0);
  }
  return 0;
}
`` `

### ServerSessionPool::purge

`` `
void
ServerSessionPool::purge()
{
  for (IPHashTable::iterator last = m_ip_pool.end(), spot = m_ip_pool.begin(); spot != last; ++spot) {
    spot->do_io_close();
  }
  m_ip_pool.clear();
  m_host_pool.clear();
}
`` `

## References

- [HttpProxyAPIEnums.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpProxyAPIEnums.h)
- [HttpSessionManager.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.h)
- [HttpSessionManager.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.cc)
