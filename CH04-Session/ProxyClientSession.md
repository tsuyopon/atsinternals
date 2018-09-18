# Base component: ProxyClientSession

ProxyClientSession is the base class for all XXXClientSession.

Any XXXClientSession must define four methods: start(), new_connection(), get_netvc(), release_netvc():

  - new_connection()
    - Used to accept a call to XXXSessionAccept, passing in netvc to start a new session (Session)
  - start()
    - indicates that a new transaction (Transaction) is started
  - get_netvc()
    - used to get incoming netvc
  - release_netvc()
    - used to unlink with netvc

## definition

ProxyClientSession cannot be used directly, you must define its inheritance class to use.

```
class ProxyClientSession : public VConnection
{
public:
  ProxyClientSession();

  // Reclaim the resources of the ClientSession
  // Release the memory occupied by member api_hooks
  // Release the memory occupied by member mutex
  // Caution:
  // In the inheritance class need to define this method, used to release the memory space occupied by the object
  // You must first execute the do_io_close and release_netvc methods before calling this method.
  // Should this be declared as a pure virtual function?
  virtual void destroy();
  // start a transaction
  virtual void start() = 0;

  // Start a conversation
  virtual void new_connection(NetVConnection *new_vc, MIOBuffer *iobuf, IOBufferReader *reader, bool backdoor) = 0;

  virtual void
  ssn_hook_append(TSHttpHookID id, INKContInternal *cont)
  {
    this->api_hooks.prepend(id, cont);
  }

  virtual void
  ssn_hook_prepend(TSHttpHookID id, INKContInternal *cont)
  {
    this->api_hooks.prepend(id, cont);
  }

  // Get the netvc corresponding to the ClientSession
  virtual NetVConnection *get_netvc() const = 0;
  // Disassociated from netvc, but does not close netvc
  virtual void release_netvc() = 0;

  APIHook *
  ssn_hook_get(TSHttpHookID id) const
  {
    return this->api_hooks.get(id);
  }

  void *
  get_user_arg(unsigned ix) const
  {
    ink_assert(ix < countof(user_args));
    return this->user_args[ix];
  }

  void
  set_user_arg(unsigned ix, void *arg)
  {
    ink_assert(ix < countof(user_args));
    user_args[ix] = arg;
  }

  // Return whether debugging is enabled for this session.
  bool
  debug() const
  {
    return this->debug_on;
  }

  bool
  hooks_enabled() const
  {
    return this->hooks_on;
  }

  bool
  has_hooks() const
  {
    return this->api_hooks.has_hooks() || http_global_hooks->has_hooks();
  }

  // Initiate an API hook invocation.
  void do_api_callout(TSHttpHookID id);

  static int64_t next_connection_id();

protected:
  // XXX Consider using a bitwise flags variable for the following flags, so that we can make the best
  // use of internal alignment padding.

  // Session specific debug flag.
  bool debug_on;
  bool hooks_on;

private:
  // Hook information currently being processed
  // Hook level (global, session, transaction)
  APIHookScope api_scope;
  //     Hook ID
  TSHttpHookID api_hookid;
  // The Plugin callback function associated with this Hook ID that is currently being called back
  APIHook *api_current;
  // Correspondence between all Plugin callback functions registered to the current ClientSession and Hook ID
  HttpAPIHooks api_hooks;
  void *user_args[HTTP_SSN_TXN_MAX_USER_ARG];

  ProxyClientSession(ProxyClientSession &);                  // noncopyable
  ProxyClientSession &operator=(const ProxyClientSession &); // noncopyable

  int state_api_callout(int event, void *edata);
  void handle_api_return(int event);

  friend void TSHttpSsnDebugSet(TSHttpSsn, int);
};
```

## Method

### Debug related

debug()

  - Return member debug_on
  - Used to indicate whether the current session has the Debug function enabled.


### Custom Data

void *user_args[HTTP_SSN_TXN_MAX_USER_ARG];

  - Session provides such an array to hold custom data

set_user_arg(unsigned ix, void *arg)

  - A pointer can be associated on the Session to point to custom data
  - Ix is the index number, the maximum value is HTTP_SSN_TXN_MAX_USER_ARG-1

get_user_arg(unsigned ix)

  - Get a pointer to custom data
  - Ix is the index number, the maximum value is HTTP_SSN_TXN_MAX_USER_ARG-1

### Hook Related

hooks_enabled()

  - Return to member hooks_on
  - Used to indicate whether the current session has the Hook function enabled.
  - In the constructor, the default is initialized to true
  - Set to false only if backdoor is true in HttpClientSession

has_hooks()

  - Check if the current session has any hooks to be triggered by looking at the hook list

ssn_hook_prepend(TSHttpHookID id, INKContInternal *cont)

  - Add a callback cont on the ook this HookID
  - If there is already another callback cont on the id, put this cont at the top

ssn_hook_append(TSHttpHookID id, INKContInternal *cont)

  - Add a callback cont on the ook this HookID
  - If there is already another callback cont on the id, put this cont at the end

do_api_callout(TSHttpHookID id)

  - Get the callback cont of the specified Hook id and call state_api_callout directly to call back cont
  - Set the sessionEvent of the Session to state_api_callout

state_api_callout(int event, void *edata)

  - as a handleEvent method of the Session, used to handle the case where a Hook id is bound to multiple cont
  - When the first cont returns a CONTINUE event, it will continue to call back the next cont
  - Call handle_api_return to continue when all cont callbacks are completed

handle_api_return(int event)

  - used to return to the ATS process after a cont callback from a specific Hook id

## From Hook ID to Plugin

In the ATS, the Hook ID is used as an intersection in the ATS mainline process. It can be used to call back the Plugin, then return to this point and continue to run the ATS mainline process.

There is such a junction, but not necessarily every time, because there may be no Plugin at the end of this fork.

So when can I/require a callback to Plugin? In the ATS, three levels are designed:

  - Global (Hooks)
    - It’s an established path. When you get here, you must call back to Plugin.
    - usually does not distinguish between agreements
  - Session (Hoses)
    - It’s also an established path. When you get here, you must call back to Plugin.
    - But these Hook points are usually closely related to the protocol.
  - Transaction Hooks
    - Is a temporary set of roads that can be considered a conditional callback based on Plugin
    - After the Session Hooks callback Plugin, it will set whether to trigger the Hook callback to the Plugin after some conditional judgment in the Plugin.

Some Hook IDs can be used as a "session" level or as a "transaction" level, so the ATS convention:

  - If a Hook ID can have multiple levels of Plugin callbacks
  - Then call Global (Global Hooks) first.
  - Then call the session (Session Hooks)
  - Last call transaction (Transaction Hooks)

If there are multiple Plugins on a Hook ID waiting for a callback:

  - Follow the order in which each Plugin is registered to this Hook ID
  - The Plugin waiting for a callback on the same Hook ID is placed in a linked list
  - Register new Plugin in the header and trailer of this list with ssn_hook_prepend and ssn_hook_append

ATS defines three methods to support:

  - Hook ID trigger do_api_callout(id)
  - Callback state_api_callout(event, data) for Plugin
  - The main flow handle_api_return(id) returned from Plugin to ATS

These three methods have corresponding implementations in Global, Session and Transaction / SM.

In this set defined in the ProxyClientSession, the three methods are at the Global and Session levels:

  - do_api_callout initiates a call to the plugin function on the specified Hook ID
  - If there is no plugin hook on the specified Hook ID, call handle_api_return directly back to the main flow of ATS
  - Otherwise, callback to the plugin function is implemented via state_api_callout

```
// Initiate a call to the plugin function on the specified Hook ID
void
ProxyClientSession::do_api_callout(TSHttpHookID id)
{
  // Use the assert to limit the SSN_START and SSN_CLOSE two Hooks that only support the HTTP protocol.
  // The implementation here is not particularly good. When you want to extend ATS to support more protocols, you need to rewrite this to HttpClientSession.
  ink_assert(id == TS_HTTP_SSN_START_HOOK || id == TS_HTTP_SSN_CLOSE_HOOK);

  // three variables related to the Plugin callback
  // api_hookid is used to record which callback id is triggered this time.
  this->api_hookid = id;
  // api_scope is used to indicate the highest level to which this hook id applies. You can see that this is set to Global.
  this->api_scope = API_HOOK_SCOPE_GLOBAL;
  // api_current is used to indicate the plugin that is currently being called back
  // plugin is also a state machine that needs to be completed by multiple callbacks.
  // will then continue to call back the next plugin associated with the current hook id
  this->api_current = NULL;

  // The judgment of has_hooks() is dynamic,
  // Each time a plugin is taken from the hook list, the last hook list becomes NULL
  if (this->hooks_on && this->has_hooks()) {
    // Switch the ProxyClientSession handler
    SET_HANDLER(&ProxyClientSession::state_api_callout);
    // First callback via state_api_callout
    this->state_api_callout(EVENT_NONE, NULL);
  } else {
    // If there is no plugin on this Hook ID, then go back to the main flow of ATS directly via handle_api_return
    // The implementation here is not particularly good. When you want to extend ATS to support more protocols, you need to rewrite this to HttpClientSession.
    this->handle_api_return(TS_EVENT_HTTP_CONTINUE);
  }
}
```

```
// Hook ID legality check
static bool
is_valid_hook(TSHttpHookID hookid)
{
  return (hookid >= 0) && (hookid < TS_HTTP_LAST_HOOK);
}
```

```
// Implement the callback to the plugin function
int
ProxyClientSession::state_api_callout(int event, void * /* data ATS_UNUSED */)
{
  switch (event) {
  Case EVENT_NONE: // indicates the first trigger, at this time:
                    // this->api_hookid is the triggered Hook ID
                    // this->api_scope is the triggering level: Global, Local, None respectively represent global, session, transaction, but brother level
                    // this->api_current is NULL, then it will point to the plugin callback function being executed
  Case EVENT_INTERVAL: // indicates that this is a timed callback directly from EventSystem
                        // Usually need to lock when calling Plugin, if the lock fails, it will schedule EventSystem delay to callback again.
  Case TS_EVENT_HTTP_CONTINUE: // indicates that this is a notification message from Plugin, let ATS continue to process this session
                                // can send messages to ClientSession via TSHttpSsnReenable
    // Determine if api_hookid is a legally valid Hook ID
    if (likely(is_valid_hook(this->api_hookid))) {
      // api_current == NULL indicates that there is currently no plugin callback function being executed
      // api_scope == GLOBAL indicates that you need to find the settings of the plugin callback function from the global level.
      if (this->api_current == NULL && this->api_scope == API_HOOK_SCOPE_GLOBAL) {
        // Get the global level plugin callback function corresponding to api_hookid
        // If it is not found, then return NULL, then api_current == NULL
        // here http_global_hooks is a global variable
        this->api_current = http_global_hooks->get(this->api_hookid);
        // Set the next probe level to the session level
        this->api_scope = API_HOOK_SCOPE_LOCAL;
      }

      // If the global level fails, api_current == NULL
      // Continue to get the session level plugin callback function corresponding to api_hookid
      // Skip if the global level is successful
      if (this->api_current == NULL && this->api_scope == API_HOOK_SCOPE_LOCAL) {
        // Get the session level plugin callback function corresponding to api_hookid
        // If it is not found, then return NULL, then api_current == NULL
        // where ssn_hook_get is a member function
        this->api_current = ssn_hook_get(this->api_hookid);
        // Set the next probe level to the transaction level
        this->api_scope = API_HOOK_SCOPE_NONE;
      }

      // Since session level is not supported by TS_HTTP_SSN_START_HOOK and TS_HTTP_SSN_CLOSE_HOOK
      // So there is no further judgment here, to get the session level plugin callback function

      // If the plugin callback function is successful, it may be global or session level
      if (this->api_current) {
        bool plugin_lock = false;
        APIHook *hook = this->api_current;
        // Create an automatic pointer
        Ptr<ProxyMutex> plugin_mutex;

        // Each plugin's callback function is encapsulated by Continuation, so there will be a mutex
        // If the mutex is set correctly, you need to lock the mutex.
        // However, there will also be cases where the mutex is not set, in which case the locking process is skipped.
        if (hook->m_cont->mutex) {
          plugin_mutex = hook->m_cont->mutex;
          // Lock on the plugin's Cont attempt
          plugin_lock = MUTEX_TAKE_TRY_LOCK(hook->m_cont->mutex, mutex->thread_holding);
          // Lock failed, then call state_api_callout again after 10ms
          if (!plugin_lock) {
            SET_HANDLER(&ProxyClientSession::state_api_callout);
            mutex->thread_holding->schedule_in(this, HRTIME_MSECONDS(10));
            return 0;
          }
          // Note that there is no automatic unlocking here! !
        }

        // If there are multiple plugins interested in the same hook ID, then continue to set the callback function of the next plugin corresponding to the hook ID.
        // If this is the last plugin of the Hook ID at the current level, it will return NULL
        this->api_current = this->api_current->next();
        // callback plugin's Hook function
        // For the case where the mutex is not set correctly, do not lock the plugin's mutex, then directly callback plugin is not a problem? ? ?
        // The invoke method is equivalent to directly calling the callback function set in the plugin, and converting the Hook ID to the corresponding Event ID through the eventmap array.
        hook->invoke(eventmap[this->api_hookid], this);

        // If you have successfully locked before, you need to show unlock here.
        if (plugin_lock) {
          Mutex_unlock(plugin_mutex, this_ethread());
        }

        // After a successful callback, return directly
        // Wait for Plugin to call the TSHttpSsnReenable method, which will call the handleEvent() of ClientSession again.
        // while the handleEvent points to state_api_callout
        // Plugin will pass in two types of events: TS_EVENT_HTTP_CONTINUE or TS_EVENT_HTTP_ERROR
        // I feel that the design here is not particularly good. The EVENT related to the HTTP protocol is actually placed in the base class of the ProxyClientSession.
        // Here is actually the returned EVENT_DONE = 0
        return 0;
      }
    }

    // If the Plugin associated with the Hook ID is not found at both the global and session levels, then return to the main flow of the ATS directly via handle_api_return
    handle_api_return(event);
    break;

  Case TS_EVENT_HTTP_ERROR: // Indicates that this is a notification message from Plugin, notifying ATS that there is an error in this session and that the session needs to be terminated by ATS.
                             // can send messages to ClientSession via TSHttpSsnReenable
    // Complete error handling by passing TS_EVENT_HTTP_ERROR to handle_api_return
    this->handle_api_return(event);
    break;

  // coverity[unterminated_default]
  default:
    ink_assert(false);
  }

  // Here is actually the returned EVENT_DONE = 0
  return 0;
}
```

```
// Go back to the main process of ATS
void
ProxyClientSession::handle_api_return(int event)
{
  // save the current Hook ID
  TSHttpHookID hookid = this->api_hookid;
  // Set the callback function to state_api_callout
  // This is a protective setting, but will not be called back in the app.
  // If it is unfortunately called back, it will trigger assert
  SET_HANDLER(&ProxyClientSession::state_api_callout);

  // Reset the Hook status of ClientSession
  this->api_hookid = TS_HTTP_LAST_HOOK;
  this->api_scope = API_HOOK_SCOPE_NONE;
  this->api_current = NULL;

  // Perform the corresponding operation based on the previously saved Hook ID
  switch (hookid) {
  case TS_HTTP_SSN_START_HOOK:
    // If the Hook ID is SSN START
    if (event == TS_EVENT_HTTP_ERROR) {
      // If you passed ERROR before TSHttpSsnReenable, execute the operation to close ClientSession
      this->do_io_close();
    } else {
      // If it is CONTINUE, start a transaction
      this->start();
    }
    break;
  case TS_HTTP_SSN_CLOSE_HOOK: {
    // If the Hook ID is SSN CLOSE
    NetVConnection *vc = this->get_netvc();
    if (vc) {
      // Since it is SSN CLOSE HOOK, the ClientSession is closed regardless of any errors, so there is no need to judge the event.
      // close netvc
      vc->do_io_close();
      // set the member variable holding netvc to NULL
      this->release_netvc();
    }
    // First, release the member object of the inherited class
    // Then, release the api_hooks and mutex of the ProxyClientSession
    // Finally, reclaim the memory of the ClientSession object
    this->destroy();
    break;
  }
  default:
    Fatal("received invalid session hook %s (%d)", HttpDebugNames::get_api_hook_name(hookid), hookid);
    break;
  }
}
```
It can be seen that in the base class of ProxyClientSession, only the processing of returning the two Hook points of SSN START and SSN CLOSE to the ATS main flow is realized.

But when SSN START encounters an error and closes ClientSession, the code to close ClientSession after SSN CLOSE is completely different. Why?

  - After calling this->do_io_close(), you still need to trigger SSN CLOSE Hook
  - After SSN CLOSE Hook, directly close NetVC and release the resources of ClientSession.

and so:

  - this->do_io_close() is designed to be used
    - Perform a shutdown of ClientSession, just set the state to off, but do not recycle any resources
    - Then trigger SSN CLOSE Hook
  - After returning from Plugin, come to the processing point of SSN CLOSE Hook of handle_api_return
    - Here is the last time to close NetVC
    - then reclaim the resources of the ClientSession

## Relationship with SessionAccept

XxxSessionAccept::accept()

  - cs = THREAD_ALLOC(XxxClientSession)
  - cs->new_connection()

XxxClientSession::new_connection()

  - do_api_callout(TS_HTTP_SSN_START_HOOK)
  - handle_api_return(TS_HTTP_SSN_START_HOOK)
  - start()

XxxClientSession::start()

  - Start transaction processing
  - The Http protocol is relatively simple, calling new_transaction() in start()
  - The H2 protocol is relatively complex and requires more operations. New_transaction() is not defined.

XxxxSessionAccept is usually used to create XxxxClientSession, for example:

  - HttpSessionAccept creates HttpClientSession
  - Http2SessionAccept creates Http2ClientSession

## References

- [ProxyClientSession.h](http://github.com/apache/trafficserver/tree/master/proxy/ProxyClientSession.h)
- [ProxyClientSession.cc](http://github.com/apache/trafficserver/tree/master/proxy/ProxyClientSession.cc)
