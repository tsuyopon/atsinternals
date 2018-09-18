# core component: HttpSessionManager

Single instance, only one object declared by the HttpSessionManager class httpSessionManager:

httpSessionManager is the manager of ServerSessionPool, which actually has similarities to Processor.

Because ServerSessionPool is designed in two layers in ATS:

  - One layer is in the thread, there is a local ServerSessionPool in each thread
  - Also included in the httpSessionManager is a global ServerSessionPool

```
HttpSessionManager httpSessionManager;
```

However, it should be noted that in fact, ServerSessionPool has a three-layer design:

  - Global ServerSessionPool
  - Thread ServerSessionPool
  - ClientSession 的 Slave ServerSession

The implementation of the above three-layer design will be seen below in the acquire_session method.

## definition

```
class HttpSessionManager
{
public:
  // constructor, initialize the global session pool to NULL
  HttpSessionManager() : m_g_pool(NULL) {}

  ~HttpSessionManager() {}

  // Select a ServerSession from the session pool and then associate this ServerSession with the specified HttpSM
  HSMresult_t acquire_session(Continuation *cont, sockaddr const *addr, const char *hostname, HttpClientSession *ua_session,
                              HttpSM *sm);
  / / Return the specified ServerSession to the session pool
  HSMresult_t release_session(HttpServerSession *to_release);
  // Close and clear all sessions in the global session pool
  void purge_keepalives();
  // Create an object for the global session pool
  void init();
  // undefined, may not be complete
  int main_handler(int event, void *data);

private:
  /// Global pool, used if not per thread pools.
  /// @internal We delay creating this because the session manager is created during global statics init.
  // global session pool
  // The system is initializing the global statistics system when the SessionManager is created.
  // Therefore, we do not initialize the global session pool in the constructor, but instead call the init method to initialize the global session pool after the global statistics system has been initialized.
  ServerSessionPool *m_g_pool;
};
```

## Method

### HttpSessionManager::init

```
void
HttpSessionManager::init()
{
  m_g_pool = new ServerSessionPool;
}
```

### HttpSessionManager::purge_keepalives

```
// TODO: Should this really purge all keep-alive sessions?
// Does this make any sense, since we always do the global pool and not the per thread?
// From the comments point of view, there are still some doubts here, because this method only closes and releases the session in the global session pool.
// This method is only called in HttpSM::do_http_server_open.
// This method is called when it is determined that the largest server connection (server_max_connections) is exceeded.
void
HttpSessionManager::purge_keepalives()
{
  EThread *ethread = this_ethread();

  MUTEX_TRY_LOCK(lock, m_g_pool->mutex, ethread);
  if (lock.is_locked()) {
    m_g_pool->purge();
  } // should we do something clever if we don't get the lock?
}
```

### HttpSessionManager::acquire_session

```
HSMresult_t
HttpSessionManager::acquire_session(Continuation * /* cont ATS_UNUSED */, sockaddr const *ip, const char *hostname,
                                    HttpClientSession *ua_session, HttpSM *sm)
{
  / / Find the matching ServerSession will be saved to the to_return variable
  HttpServerSession *to_return = NULL;
  // Get the matching method from HttpSM (MATCH_NONE, MATCH_IP, MATCH_HOST, MATCH_BOTH)
  TSServerSessionSharingMatchType match_style =
    static_cast<TSServerSessionSharingMatchType>(sm->t_state.txn_conf->server_session_sharing_match);
  // save the hash of the passed in hostname
  INK_MD5 hostname_hash;
  // Indicates if a matching ServerSession is found, as a function return value
  HSMresult_t retval = HSM_NOT_FOUND;

  / / Calculate the hash value of the incoming hostname
  ink_code_md5((unsigned char *)hostname, strlen(hostname), (unsigned char *)&hostname_hash);

  // First check to see if there is a server session bound
  //   to the user agent session
  // First, check if there is already an associated ServerSession on the ClientSession
  // Note: There is no operation to get the ClientSession lock! ! !
  // Because, acquire_session is only called in HttpSM::do_http_server_open,
  // HttpSM, ClientSession, and ServerSession are designed to share the mutex of the Client NetVConnection.
  // Therefore, there is no need to lock when accessing the incoming HttpSM and HttpClientSession objects.
  to_return = ua_session->get_server_session();
  if (to_return != NULL) {
    // There is an associated ServerSession, then take the ServerSession from the ClientSession
    // At this point, yes, ClientSession is used as a session pool that can only hold one ServerSession.
    ua_session->attach_server_session(NULL);

    // Then use the match method to make the decision:
    // Whether the ServerSession previously associated with the ClientSession can match the input criteria
    if (ServerSessionPool::match(to_return, ip, hostname_hash, match_style)) {
      // match successfully
      Debug("http_ss", "[%" PRId64 "] [acquire session] returning attached session ", to_return->con_id);
      // Set the ServerSession state to be taken over by the state machine
      to_return->state = HSS_ACTIVE;
      // Call HttpSM::attach_server_session to complete the binding of ServerSession to HttpSM
      sm->attach_server_session(to_return);
      // return match complete (success)
      return HSM_DONE;
    }
    // Release this session back to the main session pool and
    //   then continue looking for one from the shared pool
    // If there is no match, it succeeds
    Debug("http_ss", "[%" PRId64 "] [acquire session] "
                     "session not a match, returning to shared pool",
          to_return->con_id);
    // Then put the ServerSession that was previously associated with the ClientSession into the session pool for future use.
    to_return->release();
    // Clear to_return, then store the results found in the session pool
    // Note: I feel like I should put this line outside the braces
    to_return = NULL;
  }

  // Now check to see if we have a connection in our shared connection pool
  // If the appropriate ServerSession is not found on the ClientSession, then it is looked up from the session pool.
  EThread *ethread = this_ethread();

  // Depending on how the current HttpSM uses the session pool, choose whether to use the global session pool or the thread local session pool.
  // Get the lock for the selected session pool.
  ProxyMutex *pool_mutex = (TS_SERVER_SESSION_SHARING_POOL_THREAD == sm->t_state.http_config_param->server_session_sharing_pool) ?
                             ethread->server_session_pool->mutex :
                             m_g_pool->mutex;
  // Try to lock the session pool
  MUTEX_TRY_LOCK(lock, pool_mutex, ethread);
  if (lock.is_locked()) {
    // successfully locked
    if (TS_SERVER_SESSION_SHARING_POOL_THREAD == sm->t_state.http_config_param->server_session_sharing_pool) {
      // If you are using a thread local session pool
      / / Call the corresponding session pool's acquireSession method to get a matching ServerSession
      retval = ethread->server_session_pool->acquireSession(ip, hostname_hash, match_style, to_return);
      Debug("http_ss", "[acquire session] thread pool search %s", to_return ? "successful" : "failed");
    } else {
      // If you are using a global session pool
      / / Call the corresponding session pool's acquireSession method to get a matching ServerSession
      retval = m_g_pool->acquireSession(ip, hostname_hash, match_style, to_return);
      Debug("http_ss", "[acquire session] global pool search %s", to_return ? "successful" : "failed");
    }
    if (to_return) {
      // Found a matching ServerSession
      Debug("http_ss", "[%" PRId64 "] [acquire session] return session from shared pool", to_return->con_id);
      // Set the ServerSession state to be taken over by the state machine
      to_return->state = HSS_ACTIVE;
      // Holding the pool lock and the sm lock
      // the attach_server_session will issue the do_io_read under the sm lock
      // Must be careful to transfer the lock for the read vio because
      // the server VC may be moving between threads TS-3266
      // Call HttpSM::attach_server_session to complete the binding of ServerSession to HttpSM
      sm->attach_server_session(to_return);
      // return match complete (success)
      retval = HSM_DONE;
    }
  } else {
    // Locked failed
    // return needs to be retried
    retval = HSM_RETRY;
  }
  return retval;
}
```

ServerSessionPool::acquireSession 方法：

  - will only return the found ServerSession
  - The read and write events that occur on the NetServer Connection of the ServerServer will still call back the state machine of the ServerSessionPool

Therefore, after obtaining the ServerSession, you must:

  - Immediately reset the I/O callback for the ServerServer via the do_io_read and do_io_write methods
  - Then you can release the lock of the ServerSessionPool

HttpSM::attach_server_session does just that. If you don't want to do any reading or writing for a while, you should at least execute:

  - do_io_read(sm, 0, NULL);
  - do_io_write(sm, 0, NULL);

### HttpSessionManager::release_session

```
HSMresult_t
HttpSessionManager::release_session(HttpServerSession *to_release)
{
  EThread *ethread = this_ethread();
  // Select a matching session pool based on the mode in which the session was originally assigned (default is the global session pool)
  ServerSessionPool *pool =
    TS_SERVER_SESSION_SHARING_POOL_THREAD == to_release->sharing_pool ? ethread->server_session_pool : m_g_pool;
  bool released_p = true;

  // The per thread lock looks like it should not be needed but if it's not locked the close checking I/O op will crash.
  // Try to lock the session pool
  MUTEX_TRY_LOCK(lock, pool->mutex, ethread);
  if (lock.is_locked()) {
    // Locked successfully
    // put back into the session pool via releaseSession
    // which resets the I/O callback via do_io_read and do_io_write
    pool->releaseSession(to_release);
  } else {
    // Locked failed
    Debug("http_ss", "[%" PRId64 "] [release session] could not release session due to lock contention", to_release->con_id);
    released_p = false;
  }

  // return different results depending on whether the ServerSession was successfully placed in the session pool
  return released_p ? HSM_DONE : HSM_RETRY;
}
```

## ALL

By analyzing the code of the HttpSessionManager, I feel that this is a semi-finished product:

  - Declared the callback function int main_handler(int event, void *data);
    - but it is not inherited from the Continuation class
  - No self-rescheduling when a lock failure occurs,
    - instead returning HSM_RETRY prompts the caller to retry
  - After calling the HttpSessionManager method in HttpSM and HttpServerSession,
    - There will always be something like "TODO" and "FIXME", suggesting that further improvement is needed.

## ServerSession and NetVConnection for the use of mutex

First look at a piece of code from the HttpSM::attach_server_session method

```
void
HttpSM::attach_server_session(HttpServerSession *s)
{
  hsm_release_assert(server_session == NULL);
  hsm_release_assert(server_entry == NULL);
  hsm_release_assert(s->state == HSS_ACTIVE);
  server_session = s; 
  server_session->transact_count++;

  // Set the mutex so that we have something to update
  //   stats with
  server_session->mutex = this->mutex;
...
}
```

When this code is executed:

  - ServerSession just removed from the session pool
  - I/O callback for ServerSession still points to the session pool
  - The session pool is already locked, so the I/O callback for ServerSession will not occur

HttpSM changes the lock of the ServerSession to the HttpSM lock. At this point, the following instances share the same lock:

  - Client UnixNetVConnection
  - ClientSession
  - HttpSM
  - ServerSession

The locks used by Server UnixNetVConnection alone are different.

Who will lock the ServerSession?

  - We know that there is no callback function on HttpServerSession, so NetHandler will not lock it
  - But if ServerSession always shares locks for HttpSM and ClientSession
    - NetHandle locks when calling HttpSM and ClientSession, which is equivalent to locking the lock of ServerSession at the same time.
  - When the HttpServerSession is placed in the session pool, the lock is not switched to the lock of the session pool
    - Then the HttpServerSession is actually unlocked when the session pool is called by the NetHandler
    - But at this point the HttpServerSession has been detached from HttpSM and HttpClientSession and will not be accessed by other objects.

Looking back, HttpServerSession exists as a slave of HttpClientSession, so you can determine:

  - ServerSession is a subordinate to ClientSession or an affiliate of SM
  - In order to securely access members of ServerSession, ServerSession is required to share the locks of its owner
  - This way, when its owner is called back, its owner is naturally locked, and at the same time ServerSession is also locked because it shares the same lock.

When the owner (HttpClientSession or HttpSM) is called back, you can directly access the members of the HttpServerSession.

## References

- [HttpProxyAPIEnums.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpProxyAPIEnums.h)
- [HttpSessionManager.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.h)
- [HttpSessionManager.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.cc)
