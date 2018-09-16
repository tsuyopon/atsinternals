# Core component: HttpSessionAccept


Before reading this chapter, please make sure you have read and understood it completely:

  - [CH02: ProtocolProbeSessionAccept and Trampoline](https://github.com/oknet/atsinternals/blob/master/CH02-IOCoreNET/CH02S12-Core-ProtocolProbeSessionAccept-and-Trampoline.md)
  - [CH03: SSLNextProtocolAccept](https://github.com/oknet/atsinternals/blob/master/CH03-IOCoreSSL/CH03S05-Base-SSLNextProtocolAccept.md)
  - [CH03: SSLNextProtocolTrampoline](https://github.com/oknet/atsinternals/blob/master/CH03-IOCoreSSL/CH03S04-Base-SSLNextProtocolTrampoline.md)
  - [CH03: SessionAccept](https://github.com/oknet/atsinternals/blob/master/CH03-IOCoreSSL/CH03S02-Base-SessionAccept.md)

This chapter will focus on the Session processing of the Http protocol, and introduce how NetVC comes from IOCore to the state machine of each protocol.

## definition

```
/**
   The continuation mutex is NULL to allow parellel accepts in NT. No
   state is recorded by the handler and values are required to be set
   during construction via the @c Options struct and never changed. So
   a NULL mutex is safe.

   The comment specifies that the mutex that initializes Continuation in the constructor with: SessionAccept(NULL) is NULL:
      This is to support the acceptance of new sessions in parallel in the NT environment. (The same is true in Linux/Unix)
   because:
      The handler does not set or record any state;
      The value initialized in the constructor via the Options structure will never change afterwards (read-only)
   Therefore, it is safe to set mutex to NULL here.
   In fact, in ATS, all XXXSessionAccept mutex can be set to NULL

   Most of the state is simply passed on to the @c ClientSession after
   an accept. It is done here because this is the least bad pathway
   from the top level configuration to the HTTP session.
   
   Most of the settings are passed to the XXXClientSession in the method XXXSessionAccept::accept().
   In order to pass the configuration item to the Http Session, this is not a bad way.
*/

class HttpSessionAccept : public SessionAccept, private detail::HttpSessionAcceptOptions
{
private:
  typedef HttpSessionAccept self; ///< Self reference type.
public:
  /** Construction options.
      Provide an easier to remember typedef for clients.
  */
  typedef detail::HttpSessionAcceptOptions Options;

  /** Default constructor.
      @internal We don't use a static default options object because of
      initialization order issues. It is important to pick up data that is read
      from the config file and a static is initialized long before that point.
  */
  // Constructor
  // I have already explained this, here the base class SessionAccept is initialized;
  // Initialize each configuration item with the passed opt.
  HttpSessionAccept(Options const &opt = Options()) : SessionAccept(NULL), detail::HttpSessionAcceptOptions(opt) // copy these.
  {
    SET_HANDLER(&HttpSessionAccept::mainEvent);
    return;
  }

  ~HttpSessionAccept() { return; }

  // Every XXXSessionAccept needs to define this accept method.
  // In the declaration of the base class SessionAccept, accept is defined as a pure virtual function.
  void accept(NetVConnection *, MIOBuffer *, IOBufferReader *);
  // Usually when the SSLNextProtocolTrampoline bed callback is used, the accept method may be called indirectly via the Handler.
  // Because the type of communication protocol that is going to be made can be directly obtained through NPN / ALPN, so there is no need to read a piece of content for analysis (Probe)
  // So, there is no need to pass MIOBuffer *, IOBufferReader * to XXXSessionAccept;
  // Conversely, if you analyze through Probe, you can get the type of protocol that will communicate.
  // Then you need to call the accept method to pass MIOBuffer *, IOBufferReader *.
  int mainEvent(int event, void *netvc);

private:
  HttpSessionAccept(const HttpSessionAccept &);
  HttpSessionAccept &operator=(const HttpSessionAccept &);
};
```

## Method

```
void
HttpSessionAccept::accept(NetVConnection *netvc, MIOBuffer *iobuf, IOBufferReader *reader)
{
  sockaddr const *client_ip = netvc->get_remote_addr();
  const AclRecord *acl_record = NULL;
  ip_port_text_buffer ipb;
  IpAllow::scoped_config ipallow;

  // The backdoor port is now only bound to "localhost", so no
  // reason to check for if it's incoming from "localhost" or not.
  if (backdoor) {
    // backdoor is a service inside ATS. It is basically cut in the latest 6.0.x branch, leaving only one heartbeat detection function with traffic_manage.
    // If you are interested in this, you can look at the older code and study it yourself.
    acl_record = IpAllow::AllMethodAcl();
  } else if (ipallow && (((acl_record = ipallow->match(client_ip)) == NULL) || (acl_record->isEmpty()))) {
    // ipallow corresponds to the ip_allow.config function, which is outside the scope of this chapter.
    // If you are interested in this, you can read the relevant code yourself, or if I have time, I will do an analysis.
    ////////////////////////////////////////////////////
    // if client address forbidden, close immediately //
    ////////////////////////////////////////////////////
    // If do not pass the ip_allow check, execute do_io_close()
    // ??memleak?? The incoming MIOBuffer is not released?
    // Bug confirmed: https://issues.apache.org/jira/browse/TS-4697
    Warning("client '%s' prohibited by ip-allow policy", ats_ip_ntop(client_ip, ipb, sizeof(ipb)));
    netvc->do_io_close();

    return;
  }

  // Set the transport type if not already set
  // Set the properties of netvc, which is the transfer type.
  if (HttpProxyPort::TRANSPORT_NONE == netvc->attributes) {
    netvc->attributes = transport_type;
  }

  // Output debug information
  if (is_debug_tag_set("http_seq")) {
    Debug("http_seq", "[HttpSessionAccept:mainEvent %p] accepted connection from %s transport type = %d", netvc,
          ats_ip_nptop(client_ip, ipb, sizeof(ipb)), netvc->attributes);
  }

  // Create an HttpClientSession
  HttpClientSession *new_session = THREAD_ALLOC_INIT(httpClientSessionAllocator, this_ethread());

  // copy over session related data.
  // Copy most of the settings into the HttpClientSession
  new_session->f_outbound_transparent = f_outbound_transparent;
  new_session->f_transparent_passthrough = f_transparent_passthrough;
  new_session->outbound_ip4 = outbound_ip4;
  new_session->outbound_ip6 = outbound_ip6;
  new_session->outbound_port = outbound_port;
  new_session->host_res_style = ats_host_res_from(client_ip->sa_family, host_res_preference);
  new_session->acl_record = acl_record;

  // Transfer control of netvc to HttpClientSession
  new_session->new_connection(netvc, iobuf, reader, backdoor);

  return;
}

int
HttpSessionAccept::mainEvent(int event, void *data)
{
  ink_release_assert(event == NET_EVENT_ACCEPT || event == EVENT_ERROR);
  ink_release_assert((event == NET_EVENT_ACCEPT) ? (data != 0) : (1));

  // Is a shell that calls the accept method
  if (event == NET_EVENT_ACCEPT) {
    this->accept(static_cast<NetVConnection *>(data), NULL, NULL);
    return EVENT_CONT;
  }

  /////////////////
  // EVENT_ERROR //
  /////////////////
  if (((long)data) == -ECONNABORTED) {
    /////////////////////////////////////////////////
    // Under Solaris, when accept() fails and sets //
    // errno to EPROTO, it means the client has    //
    // sent a TCP reset before the connection has  //
    // been accepted by the server...  Note that   //
    // in 2.5.1 with the Internet Server Supplement//
    // and also in 2.6 the errno for this case has //
    // changed from EPROTO to ECONNABORTED.        //
    /////////////////////////////////////////////////

    // FIX: add time to user_agent_hangup
    HTTP_SUM_DYN_STAT(http_ua_msecs_counts_errors_pre_accept_hangups_stat, 0);
  }

  MachineFatal("HTTP accept received fatal error: errno = %d", -((int)(intptr_t)data));
  return EVENT_CONT;
}
```


## Reference material

- [HttpSessionAccept.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionAccept.h)
- [HttpSessionAccept.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionAccept.cc)
