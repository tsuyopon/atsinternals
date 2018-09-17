# Core component: sslStartHandShake


Before introducing SSLNetVConnection, there is a lot of work to be done. The trampoline described above is only for the negotiation of NPN and ALPN.

In the SSL/TLS protocol, it is divided into a handshake process and a transmission process, and renegotiation occurs during the transmission.

Next we will look at the implementation related to the handshake process:

  - ret = sslStartHandShake(SSL_EVENT_SERVER, err);
    - sslServerHandShakeEvent
  - ret = sslStartHandShake(SSL_EVENT_CLIENT, err);
    - sslClientHandShakeEvent

These three methods are defined in SSLNetVConnection to implement the SSL handshake process:

  - When ATS accepts a client-initiated SSL connection, ATS acts as an SSL server, so sslServerHandShakeEvent is called to complete the handshake.
  - When the ATS initiates an SSL connection to the OServer, the ATS acts as the SSL Client, so the sslClientHandShakeEvent is called to complete the handshake.

## Method

### sslStartHandShake analysis

sslStartHandShake is used to initialize the members of SSLNetVC SSL *ssl;. In the SSL handshake, the SSL Client and/or SSL Server need to be initialized in different ways. If the initialization fails, it will report an error directly.

After the ssl initialization is completed, sslServerHandShakeEvent or sslClientHandShakeEvent is called according to the value of the event to complete the subsequent handshake.

`` `
int
SSLNetVConnection::sslStartHandShake(int event, int &err)
{
  // Used to record the time when the SSL handshake starts. It can be used to determine whether the handshake process times out.
  if (sslHandshakeBeginTime == 0) {
    sslHandshakeBeginTime = Thread::get_hrtime();
    // net_activity will not be triggered until after the handshake
    set_inactivity_timeout(HRTIME_SECONDS(SSLConfigParams::ssl_handshake_timeout_in));
  }
  
  // Determine the type of handshake based on event
  switch (event) {
  case SSL_EVENT_SERVER:
    // ATS as SSL Server
    if (this->ssl == NULL) {  // Member ssl is used to save SSL Session information, created by OpenSSL
      // Read the ssl_multicert.config configuration file
      SSLCertificateConfig::scoped_config lookup;
      IpEndpoint ip;
      int namelen = sizeof(ip);
      safe_getsockname(this->get_socket(), &ip.sa, &namelen);
      // According to the information of the ssl_multicert.config configuration file, find out whether there is a match with the current ip.
      SSLCertContext *cc = lookup->find(ip);
      // Output debugging information
      if (is_debug_tag_set("ssl")) {
        IpEndpoint src, etc .;
        ip_port_text_buffer ipb1, ipb2;
        int ip_len;

        safe_getsockname(this->get_socket(), &dst.sa, &(ip_len = sizeof ip));
        safe_getpeername(this->get_socket(), &src.sa, &(ip_len = sizeof ip));
        ats_ip_nptop(&dst, ipb1, sizeof(ipb1));
        ats_ip_nptop(&src, ipb2, sizeof(ipb2));
        Debug("ssl", "IP context is %p for [%s] -> [%s], default context %p", cc, ipb2, ipb1, lookup->defaultContext());
      }

      // Escape if this is marked to be a tunnel.
      // No data has been read at this point, so we can go
      // directly into blind tunnel mode
      // If the rules related to this IP are defined in the ssl_multicert.config configuration file, and action=tunnel
      if (cc && SSLCertContext::OPT_TUNNEL == cc->opt && this->is_transparent) {
        // Set to Blind Tunnel
        this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
        // Force the handshake process to complete
        sslHandShakeComplete = 1;
        // Release ssl member
        SSL_free(this->ssl);
        this->ssl = NULL;
        // Return caller
        // At this point, the SSLVC becomes a TCP Blind Tunnel, and ATS only forwards data in both directions at the TCP layer.
        return EVENT_DONE;
      }

      // If not set to Blind Tunnel:
      // Attach the default SSL_CTX to this SSL session. The default context is never going to be able
      // to negotiate a SSL session, but it's enough to trampoline us into the SNI callback where we
      // can select the right server certificate.
      // Initialize the ssl member with the default context.
      // The defaultContext() setting is not allowed to negotiate SSL sessions. -- What does this mean? ? ?
      // But SNI Callback can be triggered so that you can choose a correct server-side certificate.
      this->ssl = make_ssl_connection(lookup->defaultContext(), this);
#if !(TS_USE_TLS_SNI)
      // If the current version of OpenSSL is not supported by the basic TLS SNI Callback,
      // I need to initialize the SSLTrace state for debugging here.
      // set SSL trace
      if (SSLConfigParams::ssl_wire_trace_enabled) {
        bool trace = computeSSLTrace();
        Debug("ssl", "sslnetvc. setting trace to=%s", trace ? "true" : "false");
        setSSLTrace(trace);
      }
#endif
    }
    // If the creation of the ssl member fails, an error is reported and an error is returned.
    if (this->ssl == NULL) {
      SSLErrorVC(this, "failed to create SSL server session");
      return EVENT_ERROR;
    }

    // Finally call sslServerHandShakeEvent to handshake
    return sslServerHandShakeEvent(err);

  case SSL_EVENT_CLIENT:
    // If the member ssl is not created
    if (this->ssl == NULL) {
      // Initialize and create a member using client_ctx ssl
      this->ssl = make_ssl_connection(ssl_NetProcessor.client_ctx, this);
    }

    // If the creation of the ssl member fails, an error is reported and an error is returned.
    if (this->ssl == NULL) {
      SSLErrorVC(this, "failed to create SSL client session");
      return EVENT_ERROR;
    }
    
    // Finally call sslClientHandShakeEvent to handshake
    return sslClientHandShakeEvent(err);

  default:
    // Other cases, exception error / call
    ink_assert(0);
    return EVENT_ERROR;
  }
}
`` `

## Handshake process when ATS is used as SSL Server

### sslServerHandShakeEvent analysis

sslServerHandShakeEvent mainly implements the encapsulation of the SSL_accept method:

  - Triggering PreAcceptHook before calling SSL_accept,
  - Handling the state of staying in the PreAcceptHook,
  - After completing the PreAcceptHook, complete the handshake process via SSL_accept

When the handshake process is completed by calling SSL_accept:

  - Will trigger SNI Callback or CERT Callback,
  - Handling the state of staying in the SNI/CERT Hook,
  - After completing the SNI/CERT Hook, continue to call SSL_accept to complete the handshake process.

The hibernation state of the above two Hooks needs to call reenable in the Hook to trigger the re-invocation of sslServerHandShakeEvent to continue.

It should be noted that NetHandler will call back sslServerHandShakeEvent when reading events and writing events, so you should consider both the read and write callbacks when reading the code.

`` `
int
SSLNetVConnection::sslServerHandShakeEvent(int &err)
{
  // sslPreAcceptHookState is used to support the implementation of PreAcceptHook
  // Here we first determine if the PreAcceptHook phase has been completed.
  if (SSL_HOOKS_DONE != sslPreAcceptHookState) {
    // Get the first hook if we haven't started invoking yet.
    // PreAcceptHook will only fire once for each SSLNetVC.
    // There is no NET_EVENT_ACCEPT event passed to the upper state machine when calling the Hook function/plugin
    if (SSL_HOOKS_INIT == sslPreAcceptHookState) {
      // The SSL_HOOKS_INIT state indicates that the PreAcceptHook was not triggered
      // Get the Hook function / plugin, and then set the status to "initiate"
      curHook = ssl_hooks->get(TS_VCONN_PRE_ACCEPT_INTERNAL_HOOK);
      sslPreAcceptHookState = SSL_HOOKS_INVOKE;
    } else if (SSL_HOOKS_INVOKE == sslPreAcceptHookState) {
      // The SSL_HOOKS_INVOKE state indicates that the PreAcceptHook is in the initiating
      // Due to the Hook function/plugin, there may be more than one,
      // So this is possible when the first function/plugin call is completed and the second one has not yet started.
      // In addition, the Hook function/plugin needs to delay the PreAccept process for some reason and may stay in this state.
      // if the state is anything else, we haven't finished
      // the previous hook yet.
      // Get the next Hook function/plugin
      curHook = curHook->next();
    }
    if (SSL_HOOKS_INVOKE == sslPreAcceptHookState) {
      // If the state is in the initiating state
      if (0 == curHook) { // no hooks left, we're done
        // The next Hook function/plugin points to NULL (0), indicating no
        // Then all the Hook functions/plugins are executed, and the PreAcceptHook is all executed.
        // Set to SSL_HOOKS_DONE to indicate that the PreAcceptHook phase has been completed
        sslPreAcceptHookState = SSL_HOOKS_DONE;
      } else {
        // The SSL_HOOKS_ACTIVE state indicates that a callback to the Hook function/plugin is being executed.
        // In the callback operation of the Hook function/plugin, reset the sslPreAcceptHookState state by calling reenable.
        // For non-SSL_HOOKS_DONE state, unified to SSL_HOOKS_INVOKE state
        sslPreAcceptHookState = SSL_HOOKS_ACTIVE;
        // callback Hook function/plugin
        // The default SSLNetVC mutex has been locked, wrap attempts to lock the Hook function/plugin's mutex, and if successful, it performs a synchronous callback.
        // Fail to create ContWrapper asynchronous callback via EventSystem, ContWrapper's mutex share SSLNetVC's mutex
        // So the first time you lock the SSLNetVC mutex in the EventSystem callback,
        // Then try again to lock the Hook function/plugin's mutex in ContWrapper's callback function event_handler.
        // If the lock is successful, the callback Hook function/plugin is synchronized, and then the ContWrapper is released.
        // If the lock fails, re-schedule the ContWrapper and execute it again.
        ContWrapper::wrap(mutex, curHook->m_cont, TS_EVENT_VCONN_PRE_ACCEPT, this);
    
        // return SSL_WAIT_FOR_HOOK, which means that only reenable can continue the process
        return SSL_WAIT_FOR_HOOK;
      }
    } else { // waiting for hook to complete
      // is not the SSL_HOOKS_INVOKE state, for example, it may be the SSL_HOOKS_ACTIVE state,
      // Then continue with SSL_WAIT_FOR_HOOK
             /* A note on waiting for the hook. I believe that because this logic
                cannot proceed as long as a hook is outstanding, the underlying VC
                can't go stale. If that can happen for some reason, we'll need to be
                more clever and provide some sort of cancel mechanism. I have a trap
                in SSLNetVConnection::free to check for this.
             * /
      return SSL_WAIT_FOR_HOOK;
    }
  }

  /////////////
  //
  // There is an SSL bug here: https://issues.apache.org/jira/browse/TS-4075, but the patch has not been officially confirmed.
  // The situation is this:
  // If the SSL_accept procedure for the current SSLVC is suspended in the SNI/CERT Hook function callback,
  // sslHandshakeHookState == status of HANDSHAKE_HOOKS_CERT
  // At this point, if the client closes the Socket connection, then epoll_wait will find that the socket fd is readable.
  // Then NetHandler calls net_read_io and finds that the SSL handshake is not complete.
  // Then net_read_io calls ret = sslStartHandShake(SSL_EVENT_SERVER, err); 
  // Then sslStartHandShake calls sslServerHandShakeEvent , which is the current function
  // Then, when the BIO buffer is filled before the SSLAccept is called, the read_raw_data is called to find 0 bytes.
  // This indicates that the connection was interrupted and returned EVENT_ERROR to net_read_io
  // net_read_io finds that when EVENT_ERROR is encountered, SSLVC is turned off.
  // However, at this point the Plugin is still being processed, and when the Plugin needs to act on this SSLVC, the ATS crashes.
  //
  /////////////
  //
  // The fix is to return SSL_WAIT_FOR_HOOK when it is found in the sslHandshakeHookState == HANDSHAKE_HOOKS_CERT state,
  // This will delay the closing of SSLVC, so that when Plugin needs to act on this SSLVC, there will be no situation where SSLVC cannot be found.
  //
  /////////////

  // If a blind tunnel was requested in the pre-accept calls, convert.
  // Again no data has been exchanged, so we can go directly
  // without data replay.
  // Note we can't arrive here if a hook is active.
  // The SSLVC property can be set in the PreAcceptHook to make the SSLVC become the Blind Tunnel.
  // Call TSVConnTunnel inside Hook to set SSLVC to be Blind Tunnel
  // Here is the response to this setting
  if (TS_SSL_HOOK_OP_TUNNEL == hookOpRequested) {
    // Set the property to Blind Tunnel
    this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
    // release member ssl
    SSL_free(this->ssl);
    this->ssl = NULL;
    // Don't mark the handshake as complete yet,
    // Will be checking for that flag not being set after
    // we get out of this callback, and then will shuffle
    // over the buffered handshake packets to the O.S.
    // If only the case of PRE ACCEPT Hook is considered here, it can be set directly to the "handshake completed" state.
    // But here you also need to consider the SNI/CERT Hook, it is also possible to set the VC as a Blind Tunnel.
    // Because there is already data in the handshakeBuffer when the SNI/CERT Hook is triggered,
    // Need to go back to net_read_io () for processing, so this can not be set to "handshake completed" state
    // Otherwise, you will not be able to go back to the "handshake process" in net_read_io() for subsequent processing.
    return EVENT_DONE;
  } else if (TS_SSL_HOOK_OP_TERMINATE == hookOpRequested) {
    // If you need to terminate this SSLVC within the PreAcceptHook, you can set it to TS_SSL_HOOK_OP_TERMINATE
    // Directly set to the state of the handshake completion -----> This can terminate this SSLVC? ?
    sslHandShakeComplete = 1;
    return EVENT_DONE;
  }
  // Here, all the parts related to PreAcceptHook have been completed.

  int retval = 1; // Initialze with a non-error value

  // All the pre-accept hooks have completed, proceed with the actual accept.
  // Check the ssl read channel rbio, is it empty?
  if (BIO_eof(SSL_get_rbio(this->ssl))) { // No more data in the buffer
    // If there is no data, read the data through read_raw_data
    // Only a previous rbio is completely consumed, and a new one can be set via read_raw_data.
    // Because read_raw_data will replace the bio that was originally set.
    // This SSL_accept can handle these SSL handshake data.
    // Read from socket to fill in the BIO buffer with the
    // raw handshake data before calling the ssl accept calls.
    retval = this->read_raw_data();
    // return 0 for EOF
    if (retval == 0) {
      // EOF, go away, we stopped in the handshake
      SSLDebugVC(this, "SSL handshake error: EOF");
      return EVENT_ERROR;
    }
  }

  // Call SSLAccept to consume data in rbio
  // SSLAccept is a wrapper around the OpenSSL API SSL_accept
  ssl_error_t ssl_error = SSLAccept(ssl);
  bool trace = getSSLTrace();
  Debug("ssl", "trace=%s", trace ? "TRUE" : "FALSE");

  // SSL_ERROR_NONE means there is no error
  if (ssl_error != SSL_ERROR_NONE) {
    // SSLAccept call has gone wrong
    // Is it useful to save errno here? ? ?
    err = errno;
    SSLDebugVC(this, "SSL handshake error: %s (%d), errno=%d", SSLErrorName(ssl_error), ssl_error, err);

    // start a blind tunnel if tr-pass is set and data does not look like ClientHello
    // If SSLAccept is wrong, then check to see if the tr-pass flag is set.
    char *buf = handShakeBuffer->buf();
    // If the tr-pass flag is set, and the received data is not like an SSL handshake request
    if (getTransparentPassThrough() && buf && *buf != SSL_OP_HANDSHAKE) {
      SSLDebugVC(this, "Data does not look like SSL handshake, starting blind tunnel");
      // Set the Blind Tunnel property
      this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
      // Forced to set the handshake to be incomplete, it should be the same as if (TS_SSL_HOOK_OP_TUNNEL == hookOpRequested) above.
      sslHandShakeComplete = 0;
      // Return to the caller, according to the Blind Tunnel for processing
      return EVENT_CONT;
    }
  }

  // error handling based on the return value of SSLAccept
  // Reference: https: //www.openssl.org/docs/manmaster/ssl/SSL_get_error.html
  switch (ssl_error) {
  case SSL_ERROR_NONE:
    // No error
    // Output debug message with successful handshake
    if (is_debug_tag_set("ssl")) {
      X509 *cert = SSL_get_peer_certificate(ssl);

      Debug("ssl", "SSL server handshake completed successfully");
      if (cert) {
        debug_certificate_name("client certificate subject CN is", X509_get_subject_name(cert));
        debug_certificate_name("client certificate issuer CN is", X509_get_issuer_name(cert));
        X509_free(cert);
      }
    }

    // Set the handshake to complete
    sslHandShakeComplete = true;

    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake completed successfully");
    // do we want to include cert info in trace?

    // Record the time used by the handshake process
    if (sslHandshakeBeginTime) {
      const ink_hrtime ssl_handshake_time = Thread::get_hrtime() - sslHandshakeBeginTime;
      Debug("ssl", "ssl handshake time:%" PRId64, ssl_handshake_time);
      sslHandshakeBeginTime = 0;
      SSL_INCREMENT_DYN_STAT_EX(ssl_total_handshake_time_stat, ssl_handshake_time);
      SSL_INCREMENT_DYN_STAT(ssl_total_success_handshake_count_in_stat);
    }

    {
      const unsigned char *proto = NULL;
      unsigned only = 0;

// If it's possible to negotiate both NPN and ALPN, then ALPN
// is preferred since it is the server's preference.  The server
// preference would not be meaningful if we let the client
// preference have priority.
      // Use the ALPN negotiation mode to get a supported proto string
#if TS_USE_TLS_ALPN
      SSL_get0_alpn_selected(ssl, &proto, &len);
#endif /* TS_USE_TLS_ALPN */

      // When ALPN fails or does not exist,
      // Use the NPN negotiation mode to get a supported proto string
#if TS_USE_TLS_NPN
      if (len == 0) {
        SSL_get0_next_proto_negotiated(ssl, &proto, &len);
      }
#endif /* TS_USE_TLS_NPN */

      // len is the length of the description string of the supported protocol type
      if (len) {
        // len is greater than 0, indicating that the negotiation was successful.
        // If there's no NPN set, we should not have done this negotiation.
        ink_assert(this->npnSet != NULL);

        // According to npnSet to find the state machine that can handle this protocol, save to npnEndpoint
        this->npnEndpoint = this->npnSet->findEndpoint(proto, len);
        this->npnSet = NULL;

        // If npnEndpoint is empty, it means that there is no corresponding state machine to handle this protocol.
        if (this->npnEndpoint == NULL) {
          Error("failed to find registered SSL endpoint for '%.*s'", (int)len, (const char *)proto);
          return EVENT_ERROR;
        }

        Debug("ssl", "client selected next protocol '%.*s'", len, proto);
        TraceIn(trace, get_remote_addr(), get_remote_port(), "client selected next protocol'%.*s'", len, proto);
      } else {
        Debug("ssl", "client did not select a next protocol");
        TraceIn(trace, get_remote_addr(), get_remote_port(), "client did not select a next protocol");
      }
    }
    // return to the caller
    return EVENT_DONE;
    
  case SSL_ERROR_WANT_CONNECT:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_CONNECT");
    return SSL_HANDSHAKE_WANT_CONNECT;

  case SSL_ERROR_WANT_WRITE:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_WRITE");
    return SSL_HANDSHAKE_WANT_WRITE;

  case SSL_ERROR_WANT_READ:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_READ");
    if (retval == -EAGAIN) {
      // No data at the moment, hang tight
      SSLDebugVC(this, "SSL handshake: EAGAIN");
      return SSL_HANDSHAKE_WANT_READ;
    } else if (retval < 0) {
      // An error, make us go away
      SSLDebugVC(this, "SSL handshake error: read_retval=%d", retval);
      return EVENT_ERROR;
    }
    return SSL_HANDSHAKE_WANT_READ;

// This value is only defined in openssl has been patched to
// enable the sni callback to break out of the SSL_accept processing
#ifdef SSL_ERROR_WANT_SNI_RESOLVE
  case SSL_ERROR_WANT_X509_LOOKUP:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_X509_LOOKUP");
    return EVENT_CONT;
  case SSL_ERROR_WANT_SNI_RESOLVE:
    // The occurrence of SSL_ERROR_WANT_SNI_RESOLVE indicates that the SSL handshake process hangs in the SNI Callback.
    // You need to call SSL_accept again to complete the handshake process.
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_SNI_RESOLVE");
#elif SSL_ERROR_WANT_X509_LOOKUP
  case SSL_ERROR_WANT_X509_LOOKUP:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_X509_LOOKUP");
#endif
#if defined(SSL_ERROR_WANT_SNI_RESOLVE) || defined(SSL_ERROR_WANT_X509_LOOKUP)
    if (this->attributes == HttpProxyPort::TRANSPORT_BLIND_TUNNEL || TS_SSL_HOOK_OP_TUNNEL == hookOpRequested) {
      // If you set it to Blind Tunnel in Hook, you need to do a processing.
      this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
      sslHandShakeComplete = 0;
      return EVENT_CONT;
    } else {
      // If it is not set to Blind Tunnel, it is stopped in the Hook function/plugin, so you need to wait
      //  Stopping for some other reason, perhaps loading certificate
      return SSL_WAIT_FOR_HOOK;
    }
#endif

  case SSL_ERROR_WANT_ACCEPT:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_ACCEPT");
    return EVENT_CONT;

  // The following are all error handling of SSL exceptions
  case SSL_ERROR_SSL: {
    SSL_CLR_ERR_INCR_DYN_STAT(this, ssl_error_ssl, "SSLNetVConnection::sslServerHandShakeEvent, SSL_ERROR_SSL errno=%d", errno);
    char buf [512];
    unsigned long e = ERR_peek_last_error();
    ERR_error_string_n(e, buf, sizeof(buf));
    TraceIn(trace, get_remote_addr(), get_remote_port(),
            "SSL server handshake ERROR_SSL: sslErr=%d, ERR_get_error=%ld (%s) errno=%d", ssl_error, e, buf, errno);
    return EVENT_ERROR;
  }

  case SSL_ERROR_ZERO_RETURN:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_ZERO_RETURN");
    return EVENT_ERROR;
  case SSL_ERROR_SYSCALL:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_SYSCALL");
    return EVENT_ERROR;
  default:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_OTHER");
    return EVENT_ERROR;
  }
}
`` `

### Implementation of SNI/CERT Hook

In the above sslServerHandShakeEvent analysis, I did not see the implementation part of SNI/CERT Hook, only saw the implementation of PreAccept Hook, then how is SNI/CERT Hook implemented?

Since the Certificate Callback is provided after OpenSSL 1.0.2d, but the previous version is SNI Callback, there is a certain difference. Therefore, the compatibility of two different versions of OpenSSL library is realized by macro definition in ATS.

But the earlier version, it seems that even the SNI Callback did not realize, at configure time, it detects SSL_CTX_set_tlsext_servername_callbackwhether available:

  - Available, then define TS_USE_TLS_SNI to be true (1)
  - If not available, define TS_USE_TLS_SNI as false (0)

When the client connects to the ATS SSL server to handshake, the SSL server needs to issue a certificate to the client. Therefore, the ATS requires a rule to be configured in ssl_multicert.config to indicate the certificate used:

  - Must set ssl_cert_name=<file.pem>
    - This certificate will be provided to the client when the Client accesses it.
    - When the certificate is loaded, it will match the SNI according to the domain name issued within it.
  - Other options are optional and not mandatory
  - If set dest_ip=*
    - Then the corresponding certificate is used as the default certificate.
    - This default certificate is used when the domain name contained in the SNI cannot be found in all certificates.
  - When the default certificate for dest_ip=* is not set,
    - After the configuration is loaded by the SSLParseCertificateConfiguration method, a record of dest_ip=* is created, but the record does not have a certificate.
    - Then, when the certificate cannot be found, the handshake cannot be established.

The loading and parsing process for ssl_multicert.config:

  - SSLNetProcessor::start(int number_of_ssl_threads, size_t stacksize)
    - SSLConfig::startup()
      - Load SSL configuration to ConfigProcessor
      - SSLConfig::reconfigure()
      - SSLConfigParams *params = new SSLConfigParams;
      - Params->initialize(); // This line is responsible for loading the SSL-related configuration in records.config
      - Save params to ConfigProcessor
    - SSLCertificateConfig::startup()
      - Load the certificate information defined in ssl_multicert.config to ConfigProcessor
      - SSLCertificateConfig::reconfigure()
        - Declare SSLConfig::scoped_config params; // Constructor loads the relevant configuration from ConfigProcessor to complete initialization
        - Declare SSLCertLookup *lookup = new SSLCertLookup();
        - Call SSLParseCertificateConfiguration(params, lookup)
        - Save lookup to ConfigProcessor

The SSLParseCertificateConfiguration method is responsible for

  - Parse the ssl_multicert.config configuration file
  - Call the ssl_store_ssl_context method to load the certificate
  - After all processing is completed, it will check if the default certificate has been set.
  - If not set, call the ssl_store_ssl_context method to add a null (NULL) certificate as the default certificate

The ssl_store_ssl_context method is responsible for

   - Add domain names and certificates to containers of type SSLCertLookup
   - If the added certificate is a default certificate, then:
     - Set this certificate to the default certificate in the SSLCertLookup type container
     - Simultaneously call the ssl_set_handshake_callbacks method to set the SNI/CERT Hook to the SSL session

Note that there are two definitions of scoped_config:

   - SSLConfig::scoped_config
     - The constructor gets the structure data of type SSLConfigParams from ConfigProcessor and completes the initialization.
   - SSLCertificateConfig::scoped_config
     - The constructor gets the structure data of type SSLCertLookup from ConfigProcessor and completes the initialization.

`` `
source: P_SSLConfig.h
struct SSLConfig {
  static void startup();
  static void reconfigure();
  static SSLConfigParams *acquire();
  static void release(SSLConfigParams *params);

  typedef ConfigProcessor::scoped_config<SSLConfig, SSLConfigParams> scoped_config;

private:
  static int configid;
};

struct SSLCertificateConfig {
  static bool startup();
  static bool reconfigure();
  static SSLCertLookup *acquire();
  static void release(SSLCertLookup *params);

  typedef ConfigProcessor::scoped_config<SSLCertificateConfig, SSLCertLookup> scoped_config;

private:
  static int configid;
};
`` `

The ssl_set_handshake_callbacks method uses macro definitions to determine which OpenSSL API method to use to set the Hook function.

Note that if TS_USE_TLS_SNI is 0, then ssl_set_handshake_callbacks is an empty function.

`` `
source: SSLUtils.cc
static void
ssl_set_handshake_callbacks(SSL_CTX *ctx)
{
#if TS_USE_TLS_SNI
// Make sure the callbacks are set
#if TS_USE_CERT_CB
  SSL_CTX_set_cert_cb(ctx, ssl_cert_callback, NULL);
#else
  SSL_CTX_set_tlsext_servername_callback(ctx, ssl_servername_callback);
#endif
#endif
}
`` `

The following are the two callback functions ssl_cert_callback and ssl_servername_callback:
  - ssl_cert_callback
    - Cert Callback when the OpenSSL version is greater than or equal to 1.0.2d
  - ssl_servername_callback
    - Otherwise, use SNI Callback
  - Both callback functions need to ensure that they must be called, and only the set_context_cert method is called once.
    - Because the set_context_cert method is used to set the default certificate and communication encryption algorithm.
    - And because of the existence of the SNI/CERT Hook function, these two callback functions may be called back multiple times.
  - Then call back the SNI/CERT Hook

The set_context_cert method is the public part of the two callback functions:

  - set_context_cert
    - Called by the above two methods to set the default SSL certificate information.
    - First look up in SSLCertLookup based on Server Name,
    - If not found, look it up based on IP.

Similarly, if TS_USE_TLS_SNI is 0, the above three methods will not be defined.

`` `
source: SSLUtils.cc
#if TS_USE_TLS_SNI
int
set_context_cert(SSL *ssl)
{
  SSL_CTX *ctx = NULL;
  SSLCertContext *cc = NULL;
  SSLCertificateConfig::scoped_config lookup;
  // Get SNI
  const char *servername = SSL_get_servername(ssl, TLSEXT_NAMETYPE_host_name);
  SSLNetVConnection *netvc = (SSLNetVConnection *)SSL_get_app_data(ssl);
  bool found = true;
  // The return value defaults to 1 for success.
  int retval = 1;

  Debug("ssl", "set_context_cert ssl=%p server=%s handshake_complete=%d", ssl, servername, netvc->getSSLHandShakeComplete());
  // set SSL trace (we do this a little later in the USE_TLS_SNI case so we can get the servername
  // Set SSL Trace
  if (SSLConfigParams::ssl_wire_trace_enabled) {
    bool trace = netvc->computeSSLTrace();
    Debug("ssl", "sslnetvc. setting trace to=%s", trace ? "true" : "false");
    netvc->setSSLTrace(trace);
  }

  // catch the client renegotiation early on
  // Handling CVE-2011-1473 client-initiated SSL re-negotiation vulnerability
  // This issue is caused by the objective factors of the SSL protocol.
  // Because the server is calculating the key, it consumes tens of times the computing resources of the client.
  // So if you can allow the client to initiate Renegotiation, it will cause a DoS attack.
  // reference:
  //     TS-1467: Disable client initiated renegotiation (SSL) DDoS by default
  //     Github: https://github.com/apache/trafficserver/commit/d43b5d685e55795a755413b92d3a8827b86c4a03
  // The fix for this issue is not limited to this one, so a full reference to the patch on github is required.
  if (SSLConfigParams::ssl_allow_client_renegotiation == false && netvc->getSSLHandShakeComplete()) {
    Debug("ssl", "set_context_cert trying to renegotiate from the client");
    retval = 0; // Error
    goto done;
  }

  // The incoming SSL_CTX is either the one mapped from the inbound IP address or the default one. If we
  // don't find a name-based match at this point, we *do not* want to mess with the context because we've
  // already made a best effort to find the best match.
  // Find the SSL_CTX setting that matches the SNI
  if (likely(servername)) {
    cc = lookup->find((char *)servername);
    if (cc && cc->ctx)
      ctx = cc->ctx;
    // Turn on the tunnel function, then directly pass through
    if (cc && SSLCertContext::OPT_TUNNEL == cc->opt && netvc->get_is_transparent()) {
      netvc->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
      netvc->setSSLHandShakeComplete(true);
      // returns -1 to suspend the SSL handshake process
      retval = -1;
      goto done;
    }
  }

  // If there's no match on the server name, try to match on the peer address.
  // When the matching SSL_CTX cannot be found through SNI, use IP to find it again
  if (ctx == NULL) {
    IpEndpoint ip;
    int namelen = sizeof(ip);

    safe_getsockname(netvc->get_socket(), &ip.sa, &namelen);
    cc = lookup->find(ip);
    if (cc && cc->ctx)
      ctx = cc->ctx;
    // There is no judgment on the Tunnel here, because here is the SNI/CERT Callback.
    // And depending on the IP, it is not necessary to be in the back position.
  }

  // If the match is successful, set SSL_CTX to the current SSL session.
  if (ctx != NULL) {
    SSL_set_SSL_CTX(ssl, ctx);
#if HAVE_OPENSSL_SESSION_TICKETS
    // Reset the ticket callback if needed
    // If you support a Session Ticket, you also need to set the Session Ticket Callback.
    SSL_CTX_set_tlsext_ticket_key_cb(ctx, ssl_callback_session_ticket);
#endif
  } else {
    found = false;
  }

  ctx = SSL_get_SSL_CTX(ssl);
  Debug("ssl", "ssl_cert_callback %s SSL context %p for requested name '%s'", found ? "found" : "using", ctx, servername);

  // If the current SSL session does not have SSL_CTX, then an error is returned
  if (ctx == NULL) {
    // return value is 0 for error
    retval = 0;
    goto done;
  }
done:
  return retval;
}

// Use the certificate callback for openssl 1.0.2 and greater
// otherwise use the SNI callback
#if TS_USE_CERT_CB
/ **
 * Called before either the server or the client certificate is used
 * Return 1 on success, 0 on error, or -1 to pause
 * /
static int
ssl_cert_callback(SSL *ssl, void * /*arg*/)
{
  SSLNetVConnection *netvc = (SSLNetVConnection *)SSL_get_app_data(ssl);
  bool reenabled;
  int retval = 1;

  // Do the common certificate lookup only once.  If we pause
  // and restart processing, do not execute the common logic again
  // Make sure it must be executed, and only execute the set_context_cert method once
  if (!netvc->calledHooks(TS_SSL_CERT_HOOK)) {
    retval = set_context_cert(ssl);
    if (retval != 1) {
      return retval;
    }
  }

  // Call the plugin cert code
  // callback SNI/CERT Hook
  reenabled = netvc->callHooks(TS_SSL_CERT_HOOK);
  // If it did not re-enable, return the code to
  // stop the accept processing
  // According to the return value to confirm whether to suspend the handshake process
  if (!reenabled) {
    retval = -1; // Pause
  }

  // Return 1 for success, 0 for error, or -1 to pause
  return retval;
}
#else
static int
ssl_servername_callback(SSL *ssl, int * /* ad */, void * /*arg*/)
{
  SSLNetVConnection *netvc = (SSLNetVConnection *)SSL_get_app_data(ssl);
  bool reenabled;
  int retval = 1;

  // Do the common certificate lookup only once.  If we pause
  // and restart processing, do not execute the common logic again
  // Make sure it must be executed, and only execute the set_context_cert method once
  if (!netvc->calledHooks(TS_SSL_CERT_HOOK)) {
    retval = set_context_cert(ssl);
    if (retval != 1) {
      goto done;
    }
  }

  // Call the plugin SNI code
  // callback SNI/CERT Hook
  reenabled = netvc->callHooks(TS_SSL_SNI_HOOK);
  // If it did not re-enable, return the code to
  // stop the accept processing
  // According to the return value to confirm whether to suspend the handshake process
  if (!reenabled) {
    retval = -1;
  }

done:
  // Map 1 to SSL_TLSEXT_ERR_OK
  // Map 0 to SSL_TLSEXT_ERR_ALERT_FATAL
  // Map -1 to SSL_TLSEXT_ERR_READ_AGAIN, if present
  // Since the return value of the earlier version of the OpenSSL API is a set of macro definitions
  // So the following code does a simple translation job
  switch (retval) {
  case 1:
    retval = SSL_TLSEXT_ERR_OK;
    break;
  case -1:
#ifdef SSL_TLSEXT_ERR_READ_AGAIN
    retval = SSL_TLSEXT_ERR_READ_AGAIN;
#else
    Error("Cannot pause SNI processsing with this version of openssl");
    retval = SSL_TLSEXT_ERR_ALERT_FATAL;
#endif
    break;
  case 0:
  default:
    retval = SSL_TLSEXT_ERR_ALERT_FATAL;
    break;
  }
  return retval;
}
#endif
#endif /* TS_USE_TLS_SNI */
`` `

### Callback for SNI/CERT Hook

In SSLVC, the callback handling of PRE ACCEPT Hook and SNI/CERT Hook is a special way.

The callback of the PRE ACCEPT Hook has been introduced in sslServerHandShakeEvent. The following is an analysis of the callback of the SNI/CERT Hook.

The call stack to the callHooks method is as follows:

  - SSLAccept()
    - SSL_accept()
      - ssl_servername_callback() / ssl_cert_callback()
        - callHooks()

`` `
bool
SSLNetVConnection::callHooks(TSHttpHookID eventId)
{
  // Only dealing with the SNI/CERT hook so far.
  // TS_SSL_SNI_HOOK and TS_SSL_CERT_HOOK are the same value
  ink_assert(eventId == TS_SSL_CERT_HOOK);

  // First time through, set the type of the hook that is currently
  // being invoked
  // Modify the state of the SNI/CERT Hook from the "initial state" to the "intermediate state"
  if (this->sslHandshakeHookState == HANDSHAKE_HOOKS_PRE) {
    this->sslHandshakeHookState = HANDSHAKE_HOOKS_CERT;
  }

  // Set the Hook function only for "intermediate state"
  // If there are multiple Plugins all Hook on SNI/CERT processing,
  // The Hook function is a linked list, which needs to be called one by one in order.
  if (this->sslHandshakeHookState == HANDSHAKE_HOOKS_CERT && eventId == TS_SSL_CERT_HOOK) {
    if (curHook != NULL) {
      // If you have already set it before, then take a Hook function
      curHook = curHook->next();
    } else {
      // If you haven't set it before, get the first Hook function
      curHook = ssl_hooks->get(TS_SSL_CERT_INTERNAL_HOOK);
    }
  } else {
    // Not in the right state, or no plugins registered for this hook
    // reenable and continue
    // The status is incorrect, or there is no plugin to register this Hook point.
    return true;
  }

  // If the curHook is not empty, then a call to the Hook function is initiated.
  bool reenabled = true;
  SSLHandshakeHookState holdState = this-> sslHandshakeHookState;
  if (curHook != NULL) {
    // Otherwise, we have plugin hooks to run
    this->sslHandshakeHookState = HANDSHAKE_HOOKS_INVOKE;
    // TS_SSL_CERT_HOOK is the most amazing Hook, it does not have its own TS_EVENT_xxxx_HOOK value
    curHook->invoke(eventId, this);
    // If TSVConnReenable(ssl_vc) is called in the Hook function, then ssl_vc->reenable(ssl_vc->nh) is called indirectly
    // If you determine that all Hook functions have been executed in reenbale, then it will be set to the state of HANDSHAKE_HOOKS_DONE
    // Only reenabled will be true at this time
    reenabled = (this->sslHandshakeHookState != HANDSHAKE_HOOKS_INVOKE);
  }
  this-> sslHandshakeHookState = holdState;
  // If reenabled is true, there is no need to suspend the SSL_accept procedure
  // Otherwise it will hang in the SSL_accept procedure until SSL_accept is called again, then callback this function
  // This function returns true for the SSL_accept process to complete.
  return reenabled;
}
`` `

Note that the SSLNetVConnection::reenable() method is polymorphic. The following is the one called by TSAPI TSVConnReenable(ssl_vc):

`` `
void
SSLNetVConnection :: reenable (NetHandler * nh)
{
  if (this->sslPreAcceptHookState != SSL_HOOKS_DONE) {
    this->sslPreAcceptHookState = SSL_HOOKS_INVOKE;
    this->readReschedule(nh);
  } else {
    // Reenabling from the handshake callback
    //
    // Originally, we would wait for the callback to go again to execute additinonal
    // hooks, but since the callbacks are associated with the context and the context
    // can be replaced by the plugin, it didn't seem reasonable to assume that the
    // callback would be executed again.  So we walk through the rest of the hooks
    // here in the reenable.
    if (curHook != NULL) {
      curHook = curHook->next();
      if (curHook != NULL) {
        // Invoke the hook
        curHook->invoke(TS_SSL_CERT_HOOK, this);
      }
    }
    if (curHook == NULL) {
      this->sslHandshakeHookState = HANDSHAKE_HOOKS_DONE;
      this->readReschedule(nh);
    }
  }
}
`` `

## Handshake process when ATS is used as SSL Client


First, let's talk briefly about SSLInitClientContext() , in this method:

  - Set the version of the SSL protocol
    - SSL_CTX_set_options(client_ctx, params->ssl_client_ctx_protocols)
  - Encryption algorithm
    - SSL_CTX_set_cipher_list(client_ctx, params->client_cipherSuite)
  - Set the verify_callback function
    - SSL_CTX_set_verify(client_ctx, SSL_VERIFY_PEER, verify_callback)
    - Verification by this verify_callback when the OServer certificate needs to be verified
    - SSL_CTX_set_verify_depth(client_ctx, params->client_verify_depth)
    - When the certificate chain exists, you can also specify the depth of verification.
  - If the function of providing a client certificate to OServer is set
    - SSL_CTX_use_certificate_chain_file(client_ctx, params->clientCertPath)
    - SSL_CTX_use_PrivateKey_file(client_ctx, clientKeyPtr, SSL_FILETYPE_PEM)
    - SSL_CTX_check_private_key(client_ctx)
    - Load the client certificate, load the client private key, and verify the match between the client certificate and the private key.
  - Initialized ssl_client_data_index
    - Used to request a memory block in the SSL session descriptor. This index represents the number of the memory block.
    - For data that can be stored/fetched by this index value
    - There are multiple such memory blocks in the SSL session descriptor that can store data at the application layer.
    - Reference: https://www.mail-archive.com/openssl-users@openssl.org/msg52326.html

SSLInitClientContext() is called by SSLNetProcessor::start(), so the initialization is done before ET_SSL starts.

### sslClientHandShakeEvent analysis

The main implementation of sslClientHandShakeEvent is:

  - ATS acts as an SSL Client to initiate an SSL Handshake to OServer
  
`` `
int
SSLNetVConnection::sslClientHandShakeEvent(int &err)
{
#if TS_USE_TLS_SNI
  // Support for SNI features
  if (options.sni_servername) {
    if (SSL_set_tlsext_host_name(ssl, options.sni_servername)) {
      Debug("ssl", "using SNI name '%s' for client handshake", options.sni_servername.get());
    } else {
      Debug("ssl.error", "failed to set SNI name '%s' for client handshake", options.sni_servername.get());
      SSL_INCREMENT_DYN_STAT(ssl_sni_name_set_failure);
    }
  }
#endif

  // In ATS, use the ex_data function to store the SSLVC instance memory address (this) in the SSL session descriptor.
  SSL_set_ex_data(ssl, get_ssl_client_data_index(), this);
  // Initiate SSL Handshake to OServer, SSLConnect is a wrapper around SSL_connect
  ssl_error_t ssl_error = SSLConnect(ssl);
  // SSL trace debug status
  bool trace = getSSLTrace();
  Debug("ssl", "trace=%s", trace ? "TRUE" : "FALSE");

  // Error handling based on SSLConnect's return value ssl_error
  switch (ssl_error) {
  // SSL_ERROR_NONE means there is no error
  case SSL_ERROR_NONE:
    // Output debugging information
    if (is_debug_tag_set("ssl")) {
      X509 *cert = SSL_get_peer_certificate(ssl);

      Debug("ssl", "SSL client handshake completed successfully");
      // if the handshake is complete and write is enabled reschedule the write
      // Why do writeReschedule when debugging information is enabled? ? ?
      if (closed == 0 && write.enabled)
        writeReschedule(nh);
      if (cert) {
        debug_certificate_name("server certificate subject CN is", X509_get_subject_name(cert));
        debug_certificate_name("server certificate issuer CN is", X509_get_issuer_name(cert));
        X509_free(cert);
      }
    }
    SSL_INCREMENT_DYN_STAT(ssl_total_success_handshake_count_out_stat);

    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake completed successfully");
    // do we want to include cert info in trace?

    // Set the handshake to complete
    sslHandShakeComplete = true;
    return EVENT_DONE;

  // The following is the handling of other error conditions
  case SSL_ERROR_WANT_WRITE:
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_WANT_WRITE");
    SSL_INCREMENT_DYN_STAT(ssl_error_want_write);
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake ERROR_WANT_WRITE");
    return SSL_HANDSHAKE_WANT_WRITE;

  case SSL_ERROR_WANT_READ:
    SSL_INCREMENT_DYN_STAT(ssl_error_want_read);
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_WANT_READ");
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake ERROR_WANT_READ");
    return SSL_HANDSHAKE_WANT_READ;

  case SSL_ERROR_WANT_X509_LOOKUP:
    SSL_INCREMENT_DYN_STAT(ssl_error_want_x509_lookup);
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_WANT_X509_LOOKUP");
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake ERROR_WANT_X509_LOOKUP");
    break;

  case SSL_ERROR_WANT_ACCEPT:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake ERROR_WANT_ACCEPT");
    return SSL_HANDSHAKE_WANT_ACCEPT;

  case SSL_ERROR_WANT_CONNECT:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake ERROR_WANT_CONNECT");
    break;

  case SSL_ERROR_ZERO_RETURN:
    SSL_INCREMENT_DYN_STAT(ssl_error_zero_return);
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, EOS");
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake EOS");
    return EVENT_ERROR;

  case SSL_ERROR_SYSCALL:
    err = errno;
    SSL_INCREMENT_DYN_STAT(ssl_error_syscall);
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, syscall");
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake Syscall Error: %s", strerror(errno));
    return EVENT_ERROR;
    break;

  case SSL_ERROR_SSL:
  default: {
    err = errno;
    // FIXME -- This triggers a retry on cases of cert validation errors....
    Debug("ssl", "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_SSL");
    SSL_CLR_ERR_INCR_DYN_STAT(this, ssl_error_ssl, "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_SSL errno=%d", errno);
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_SSL");
    char buf [512];
    unsigned long e = ERR_peek_last_error();
    ERR_error_string_n(e, buf, sizeof(buf));
    TraceIn(trace, get_remote_addr(), get_remote_port(),
            "SSL client handshake ERROR_SSL: sslErr=%d, ERR_get_error=%ld (%s) errno=%d", ssl_error, e, buf, errno);
    return EVENT_ERROR;
    }
    break;
  }
  return EVENT_CONT;
}
`` `

### OServer certificate verification process

If OServer certificate chain verification is enabled in the ATS records.config:

`` `
CONFIG proxy.config.ssl.client.verify.server INT 1
`` `

Then, after obtaining the certificate of OServer, ATS will verify the certificate chain, and the verification result of the certificate chain is saved to preverify_ok:

  - 0 means there is a problem with the certificate chain
  - 1 means certificate chain verification passed

Then call adjust_callback() to verify the domain name match.

`` `
source: SSLClientUtils.cc
int
verify_callback(int preverify_ok, X509_STORE_CTX *ctx)
{
  X509 * true;
  int depth;
  int err;
  SSL *ssl;

  SSLDebug("Entered verify cb");
  // Get the location of the error in the certificate chain
  // If it is 0, it means the end of the certificate
  // If it is 1, 2,... indicates that the parent certificate that issued this certificate has an error.
  depth = X509_STORE_CTX_get_error_depth(ctx);
  // Get the current certificate
  cert = X509_STORE_CTX_get_current_cert(ctx);
  // Get the error message
  // If err = X509_V_OK , then depth = 0 means the end of the certificate verification passed
  err = X509_STORE_CTX_get_error(ctx);

  // Enter the parameter preverify_ok. If 0, the verification of the certificate chain has failed.
  // So there is no need to verify SNI or IP
  if (! preverify_ok) {
    // Don't bother to check the hostname if we failed openssl's verification
    SSLDebug("verify error:num=%d:%s:depth=%d", err, X509_verify_cert_error_string(err), depth);
    return verify_ok;
  }
  if (depth != 0) {
    // Not server cert....
    // If this happens: Although there is no certificate validation error, there is no end certificate
    return verify_ok;
  }
  / *
   * Retrieve the pointer to the SSL of the connection currently treated
   * and the application specific data stored into the SSL object.
   * /
  // Get the SSL object from the SSL_CTX object
  ssl = static_cast<SSL *>(X509_STORE_CTX_get_ex_data(ctx, SSL_get_ex_data_X509_STORE_CTX_idx()));
  // Get the SSLVC object from the SSL object
  SSLNetVConnection *netvc = static_cast<SSLNetVConnection *>(SSL_get_ex_data(ssl, ssl_client_data_index));
  // Only certificate verification for SSLNetVConnection
  if (netvc != NULL) {
    // Match SNI if present
    // If SNI is filled in when a request is made to OServer, then the match between SNI and certificate needs to be verified.
    if (netvc->options.sni_servername) {
      char *matched_name = NULL;
      // The function validate_hostname() is defined in lib/ts/X509HostnameValidator.cc
      if (validate_hostname(cert, reinterpret_cast<unsigned char *>(netvc->options.sni_servername.get()), false, &matched_name)) {
        SSLDebug("Hostname %s verified OK, matched %s", netvc->options.sni_servername.get(), matched_name);
        ats_free(matched_name);
        // Verify by returning
        return verify_ok;
      }
      SSLDebug("Hostname verification failed for (%s)", netvc->options.sni_servername.get());
    }
    // Otherwise match by IP
    // If you do not fill in the SNI, then verify the match between the IP and the certificate.
    else {
      char buff[INET6_ADDRSTRLEN];
      // Convert IP to a string
      ats_ip_ntop(netvc->server_addr, buff, INET6_ADDRSTRLEN);
      if (validate_hostname(cert, reinterpret_cast<unsigned char *>(buff), true, NULL)) {
        SSLDebug("IP %s verified OK", buff);
        // Verify by returning
        return verify_ok;
      }
      SSLDebug("IP verification failed for (%s)", buff);
    }
    // SNI and IP verification did not succeed, returning 0 means verification failed
    return 0;
  }
  
  // If not SSLVC, directly return verification success
  return verify_ok;
}
`` `

# reference

  - [OpenSSL::SSL_accept](https://www.openssl.org/docs/manmaster/ssl/SSL_accept.html)
  - [OpenSSL::SSL_get_rbio](https://www.openssl.org/docs/manmaster/ssl/SSL_get_rbio.html)
  - [OpenSSL::SSL_set_bio](https://www.openssl.org/docs/manmaster/ssl/SSL_set_bio.html)
  - [OpenSSL::SSL_connect](https://www.openssl.org/docs/manmaster/ssl/SSL_connect.html)
  - [OpenSSL::SSL_get_error](https://www.openssl.org/docs/manmaster/ssl/SSL_get_error.html)
  - [OpenSSL::SSL_CTX_set_verify](https://www.openssl.org/docs/manmaster/ssl/SSL_CTX_set_verify.html)
  - [SSLUtils.cc](https://github.com/apache/trafficserver/blob/master/iocore/net/SSLUtils.cc)
  - [SSLClientUtils.cc](https://github.com/apache/trafficserver/blob/master/iocore/net/SSLClientUtils.cc)
