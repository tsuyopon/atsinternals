# core component: SocksEntry

SocksEntry is the state machine that ATS uses as a Socks Client to initiate Socks requests to the Socks proxy server.

When the upper layer SM calls netProcessor.connect_re(), it determines whether the connection to the target IP and port needs to be initiated through the Socks proxy server according to the configuration of the socks, but this is transparent to the upper layer SM.

## definition

```
// Like the SSL protocol, the Socks protocol can be understood as a transport layer protocol.
// The difference is that Socks only has a handshake process, but there is no communication encryption and decryption function.
// So, here too, SocksNetVC is defined like SSLNetVConnection
// However, because the Socks protocol does not need to provide communication encryption and decryption functions,
// So, it can share net_read_io and write_to_net_io with UnixNetVConnection
typedef UnixNetVConnection SocksNetVC;

struct SocksEntry : public Continuation {
  / / used to save the Socks request sent to the Socks proxy server
  // Also used to receive Socks responses returned from the Socks proxy server
  MIOBuffer * buf;
  IOBufferReader *reader;

  // NetVC created by connect_re() to connect to the Socks proxy server
  SocksNetVC *netVConnection;

  // Changed from @a ip and @a port.
  / / Save the target's IP address and port, will access the target through the Socks proxy server
  IpEndpoint target_addr; ///< Original target address.
  // Changed from @a server_ip, @a server_port.
  // Save the IP address and port of the Socks proxy server (the English comment below is wrong)
  IpEndpoint server_addr; ///< Origin server address.

  / / used to save the current number of attempts to connect with the Socks proxy server
  int nattempts;

  // After the Socks handshake is completed, the state machine for the Action connection will be called back.
  Action action_;
  // After all attempts to establish a connection with the Socks proxy server fail, the NET_EVENT_OPEN_FAILED will be called back to the above state and the -lerrno value will be passed
  int lerrno;
  // Set the timeout by the schedule_*() method and save its return value
  // SocksEntry will receive an EVENT_INTERVAL event if it encounters a Socks timeout
  // Note: If you encounter a NetVConnection timeout, you will still receive an EVENT_*_TIMEOUT event
  Event *timeout;
  // Protocol version used when communicating with the Socks proxy server
  unsigned char version;

  // indicates that ATS is in the process of shaking hands with the Socks proxy server.
  // true: ATS has made a request to the Socks proxy server
  bool write_done;

  // Function pointer for implementing Socks5 authentication, NULL for Socks4 protocol
  SocksAuthHandler auth_handler;
  // used to indicate the Socks command word in the initiated Socks request
  // If the value is NORMAL_SOCKS, it means the CONNECT command, which is also supported by SocksEntry.
  // If the value is not NORMAL_SOCKS, ATS will establish a Blind Tunnel with the target server after sending the Socks request.
  unsigned char socks_cmd;

  // socks server selection:
  // If there are multiple Socks proxy servers available, try one by one
  ParentConfigParams *server_params;
  HttpRequestData req_data; // We dont use any http specific fields.
  ParentResult server_result;

  / / Initialize the callback function
  int startEvent(int event, void *data);
  // main processing callback function
  int mainEvent(int event, void *data);
  // When there are multiple Socks proxy servers available, select the next one via findServer
  void findServer();
  // Initialize the SocksEntry instance
  void init(ProxyMutex *m, SocksNetVC *netvc, unsigned char socks_support, unsigned char ver);
  / / Call back the upper state machine connection succeeded or failed, then release SocksEntry
  void free();

  // Constructor
  SocksEntry()
    : Continuation(NULL),
      netVConnection(0),
      nattempts (0)
      lerrno (0),
      timeout(0),
      version(5),
      write_done(false),
      auth_handler(NULL),
      socks_cmd(NORMAL_SOCKS)
  {
    memset(&target_addr, 0, sizeof(target_addr));
    memset(&server_addr, 0, sizeof(server_addr));
  }
};

typedef int (SocksEntry::*SocksEntryHandler)(int, void *);

external ClassAllocator <SocksEntry> socksAllocator;
```

## Method

SocksEntry::init(mutex, vc, socks_support, ver)

- used to initialize the SocksEntry object
  - Initialize the interactive buffer used to create a Socks session
  - Set the Socks protocol version
  - The initial callback function is startEvent
  - Get the target IP and port
  - Set a timed callback to manage the timeout of the Socks proxy server connection establishment
- the parameter mutex is the mutex of the state machine that initiated the connection
  - that is, the mutex of the caller of netProcessor.connect_re()
- Parameter vc is the target IP and port that has been set, and SocksNetVC will be called for the connect() call.
  - This method gets the target IP and port from vc and saves it in the target_addr member
  - Then save the IP and port of the selected Socks proxy server to the server_addr member
- The parameter socks_support defaults to NORMAL_SOCKS
  - This parameter is mainly used to re-establish the connection to the Socks proxy server, for example: when the connection times out and multiple Socks proxy servers are available
  - After setting options.socks_support to NO_SOCKS, calling the connect_re() method will no longer enter SocksEntry
- The parameter ver indicates the version of the Socks protocol
  - The default is (unisgned char) 5, which means that the Socks 5 protocol is used to establish communication with the Socks proxy server.
- As the caller of the init() method, the server_addr member will be used instead of the target IP and port of vc
  - This will make a TCP connection to the Socks proxy server when the connect() call is initiated later.

```
void
SocksEntry::init(ProxyMutex *m, SocksNetVC *vc, unsigned char socks_support, unsigned char ver)
{
  // Initialize the mutex of SocksEntry and share the same mutex with the caller
  mutex = m;
  / / Create a session interaction buffer
  buf = new_MIOBuffer();
  reader = buf->alloc_reader();

  // Set the default value of socks_cmd, usually NORMAL_SOCKS(0)
  socks_cmd = socks_support;

  // Set the version of the Socks protocol used
  if (ver == SOCKS_DEFAULT_VERSION)
    version = netProcessor.socks_conf_stuff->default_version;
  else
    version = see;

  / / Set the event callback function to startEvent
  SET_HANDLER(&SocksEntry::startEvent);

  // There is a bug on the 6.0.x branch, it should be vc->get_remote_addr()
  // Get the target IP and port from vc and save it in the target_addr member
  // Construct target IP and port with target_addr member when constructing Socks request in the future
  ats_ip_copy(&target_addr, vc->get_local_addr());

// This macro is defined in I_Socks.h
#ifdef SOCKS_WITH_TS
  // Initialize req_data, used when calling the findParent() method to find the socks.config rule
  req_data.hdr = 0;
  req_data.hostname_str = 0;
  req_data.api_info = 0;
  req_data.xact_start = time(0);

  // This means that only IPv4 is supported as the target address.
  assert(ats_is_ip4(&target_addr));
  // save the target IP and port to req_data.dest_ip
  ats_ip_copy(&req_data.dest_ip, &target_addr);

  // we dont have information about the source. set to destination's
  // Initialize req_data.src_ip with the target IP and port just to ensure there is a valid message here
  ats_ip_copy(&req_data.src_ip, &target_addr);

  // Load the available Socks proxy server via the Socks configuration file
  server_params = SocksServerConfig::acquire();
#endif

  // Reset the number of attempts and start looking for available Socks proxy servers
  night punch = 0;
  findServer();

  // Create a timed callback that will call back SocksEntry::startEvent after n seconds to determine the timeout
  timeout = this_ethread()->schedule_in(this, HRTIME_SECONDS(netProcessor.socks_conf_stuff->server_connect_timeout));
  write_done = false;
}
```

SocksEntry::findServer()

- Used to find a Socks proxy server that can be used
- Socks proxy server can be specified for specific targets in socks.config so that different Socks proxy servers can be used for different purposes
- If not found in socks.config, you can also find proxy.config.socks.default_servers from records.config
- When attempting to establish a TCP connection with the Socks proxy server,
  - You can set the maximum number of retries (per_server_connection_attempts)
  - You can also set the total number of retries (connection_attempts)
    - Avoid retrying repeatedly causing connection timeouts when there are more Socks proxy servers available
    - Or set this value to: The number of available Socks proxy servers * per_server_connection_attempts to ensure that all Socks proxy servers are tried

```
void
SocksEntry::findServer()
{
  nattempts ++;

#ifdef SOCKS_WITH_TS
  if (nattempts == 1) {
    ink_assert(server_result.r == PARENT_UNDEFINED);
    // Look for an available Socks proxy server,
    server_params->findParent(&req_data, &server_result);
  } else {
    socks_conf_struct *conf = netProcessor.socks_conf_stuff;
    // If it’s not the first attempt, then determine if the maximum number of retries has been reached
    // If the limit is not reached, return directly, this will try to establish a connection with the Socks proxy server again.
    // reference:  proxy.config.socks.per_server_connection_attempts
    if ((nattempts - 1) % conf->per_server_connection_attempts)
      return; // attempt again

    // reached the maximum number of retries
    / / Set the Socks proxy server to the down state
    server_params->markParentDown(&server_result);

    // reference:  proxy.config.socks.connection_attempts
    if (nattempts > conf->connection_attempts)
      server_result.r = PARENT_FAIL;
    else
      server_params->nextParent(&req_data, &server_result);
  }

  // The analysis related to ParentSelection is not introduced here.
  // server_result.r == PARENT_DIRECT: indicates that the rule is not matched, and default_servers is not set
  // server_result.r == PARENT_SPECIFIED: indicates that an available Socks proxy server was found
  // server_result.r == PARENT_FAIL: Indicates that although a rule in socks.config is matched, the rule does not have a valid Socks proxy server configured.
  switch (server_result.r) {
  case PARENT_SPECIFIED:
    // Original was inet_addr, but should hostnames work?
    // ats_ip_pton only supports numeric (because other clients
    // explicitly want to avoid hostname lookups).
    // When configuring the rules of the Socks proxy server in socks.config, if the domain name is filled in instead of IP,
    // In the process of loading rules in setup_socks_servers(), the domain name will be resolved to an IP address.
    // When the parsing fails, the hostname is set to 255.255.255.255.
    // So here the pton method is used to write the hostname directly to server_addr,
    // The pton method should not fail due to the above error handling strategy when loading the configuration
    // Note 1: In ats_ip_pton, the address for IPv4 is converted to struct in_addr by inet_aton,
    // inet_aton considers 255.255.255.255 to be a valid IPv4 address
    // Note 2: In setup_socks_servers(), the IP address of the Socks proxy server is accepted as IPv6.
    // But SocksEntry is designed to only proxy requests with a destination IP address of IPv4.
    if (0 == ats_ip_pton(server_result.hostname, &server_addr)) {
      ats_ip_port_cast(&server_addr) = htons(server_result.port);
    } else {
      Debug("SocksParent", "Invalid parent server specified %s", server_result.hostname);
    }
    break;

  default:
    ink_assert(!"Unexpected event");
  case PARENT_DIRECT:
  case PARENT_FAIL:
    // Clear server_addr to zero when no available Socks proxy server is found, at times ats_is_ip() returns false
    memset(&server_addr, 0, sizeof(server_addr));
  }
#else
  if (nattempts > netProcessor.socks_conf_stuff->connection_attempts)
    memset(&server_addr, 0, sizeof(server_addr));
  else
    ats_ip_copy(&server_addr, &g_socks_conf_stuff->server_addr);
#endif // SOCKS_WITH_TS

  char buff[INET6_ADDRSTRLEN];
  Debug("SocksParents", "findServer result: %s:%d", ats_ip_ntop(&server_addr.sa, buff, sizeof(buff)),
        ats_ip_port_host_order(&server_addr));
}
```

SocksEntry::free()

- After establishing Socks communication with the Socks proxy server or encountering any errors, notify the state machine by calling the free() method, and then freeing the memory occupied by the SocksEntry.
- need to cancel the timeout event
- need to close netVC when an error occurs
- Need to consider the state machine has been closed (such as the state machine has been closed will call the cancel method through the Action object)
- need to reset the do_io operation before releasing the SocksEntry object

```
void
SocksEntry::free()
{
  MUTEX_TRY_LOCK(lock, action_.mutex, this_ethread());
  // There is an exclamation point in front of the lock in the 6.0.x version.
  // This bug was generated when 53+56ffc7499648bc096d63f9c0f9da0ea9212ba repaired TS-3156
  if (!lock.is_locked()) {
    // This assumes that the lock of action_.mutex must already be held before calling the free method.
    // If the lock cannot be successfully locked immediately, an exception occurs.
    // I think this should be the early code. There are other bugs that cause problems. Lock here to avoid problems.
    // So I think there should be no need to lock at all, just judge mutex->thread_holding == this_ethread()
    // Socks continuation share the user's lock
    // so acquiring a lock shouldn't fail
    ink_assert(0);
    return;
  }

  // cancel the previously set timeout
  // This should be timed after the cancel, assigning timeout to NULL
  if (timeout)
    timeout->cancel(this);

#ifdef SOCKS_WITH_TS
  // If there are no errors, set the state of this Socks proxy server to be available
  if (!lerrno && netVConnection && server_result.retry)
    server_params->recordRetrySuccess(&server_result);
#endif

  // The logic below is a bit confusing and should be optimized.
  // Turn off netVC if it is canceled or if something goes wrong
  // This should set netVC to NULL after closing netVC
  if ((action_.cancelled || lerrno) && netVConnection)
    netVConnection->do_io_close();

  // If it is not canceled, then the callback state machine is needed.
  if (!action_.cancelled) {
    // If netVC is set to NULL above, it is entirely possible to call back different events for the state machine based on whether NetVC is NULL or not.
    if (lerrno || !netVConnection) {
      // If the error or netVC is NULL, callback OPEN_FAILED and pass the error message
      Debug("Socks", "retryevent: Sent errno %d to HTTP", lerrno);
      NET_INCREMENT_DYN_STAT(socks_connections_unsuccessful_stat);
      action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)(intptr_t)(-lerrno));
    } else {
      // Otherwise callback OPEN and pass netVC to the state machine
      // Description successfully established Socks communication
      // There is a problem with the do_io reset here, you should not use this, you should use NULL
      netVConnection->do_io_read(this, 0, 0);
      netVConnection->do_io_write(this, 0, 0);
      netVConnection->action_ = action_; // assign the original continuation
      // The guess here is to copy target_addr back to netVC->server_addr for a completely transparent connection via SocksEntry
      // But here is written as server_addr
      ats_ip_copy(&netVConnection->server_addr, &server_addr);
      Debug("Socks", "Sent success to HTTP");
      NET_INCREMENT_DYN_STAT(socks_connections_successful_stat);
      action_.continuation->handleEvent(NET_EVENT_OPEN, netVConnection);
    } 
  }
  // Reclaim a reference to a resource
#ifdef SOCKS_WITH_TS
  SocksServerConfig::release(server_params);
#endif

  // free resources
  free_MIOBuffer(buf);
  / / Recall the reference count of action_-> mutex
  action_ = NULL;
  / / Recycle the reference count of this->mutex
  mutex = NULL;
  // Recycle the SocksEntry object
  socksAllocator.free(this);
}
```

SocksEntry::startEvent(event, data)
- used to complete the TCP connection with the Socks proxy server
- Switch to mainEvent after connection is established
- If a timeout or connection failure occurs, call findServer() to determine:
  - Try reconnecting to the current Socks proxy server
  - or choose the next available Socks proxy server
- Do_io_close() will be executed on the previous netVC before reconnecting

```
int
SocksEntry::startEvent(int event, void *data)
{
  if (event == NET_EVENT_OPEN) {
    // The connection is successfully established.
    netVConnection = (SocksNetVC *)data;

    // If you choose to establish a connection using the Socks v5 protocol, you need to process the authentication message.
    if (version == SOCKS5_VERSION)
      auth_handler = &socks5BasicAuthHandler;

    / / Switch to mainEvent
    SET_HANDLER((SocksEntryHandler)&SocksEntry::mainEvent);
    // Pass NET_EVENT_OPEN to mainEvent
    // I think it would be better to put the processing of NET_EVENT_OPEN in startEvent, the mainEvent is too long.
    mainEvent(NET_EVENT_OPEN, data);
  } else {
    // Connection establishment failed or timed out
    if (timeout) {
      timeout->cancel(this);
      timeout = NULL;
    }

    char buff[INET6_ADDRPORTSTRLEN];
    Debug("Socks", "Failed to connect to %s", ats_ip_nptop(&server_addr.sa, buff, sizeof(buff)));

    // Find the next available Socks server
    // or reconnect to the current Socks server
    findServer();

    // If the IP address of the next Socks server is not valid,
    / / As a connection error, call the upper state machine through free (),
    if (!ats_is_ip(&server_addr)) {
      Debug("Socks", "Unable to open connection to the SOCKS server");
      lerrno = ESOCK_NO_SOCK_SERVER_CONN;
      free();
      return EVENT_CONT;
    }

    // redundant code
    if (timeout) {
      timeout->cancel(this);
      timeout = 0;
    }

    / / Close the netVC failed to connect, prepare for reconnection
    if (netVConnection) {
      netVConnection->do_io_close();
      netVConnection = 0;
    }

    / / Set the timeout to re-establish the connection
    timeout = this_ethread()->schedule_in(this, HRTIME_SECONDS(netProcessor.socks_conf_stuff->server_connect_timeout));

    write_done = false;

    NetVCOptions options;
    // NO_SOCKS tells connect_re to connect directly to the target IP and port
    // If NO_SOCKS is not set, connect_re will use the Socks proxy server to establish a connection with the target IP and port.
    options.socks_support = NO_SOCKS;
    netProcessor.connect_re(this, &server_addr.sa, &options);
  }

  return EVENT_CONT;
}
```

SocksEntry::mainEvent
- If using the Socks v5 protocol, based on RFC1928 implementation
  - Send authentication negotiation header
  - Receive the authentication method returned by the Socks server
  - If the user name and password are used for authentication, the user name and password are sent to complete the authentication, which is based on RFC1929.
  - Enter the Socks command sending stage after successful authentication
  `` `
  + ----- + ----- + ------- + ------ + ---------- + ------------ +
  | VER | CMD |  RSV  | ATYP | DST.ADDR | DST.PORT |
  + ----- + ----- + ------- + ------ + ---------- + ------------ +
  |  1  |  1  | X'00' |  1   | Variable |    2     |
  + ----- + ----- + ------- + ------ + ---------- + ------------ +
  `` `

- If using the Socks v4 protocol
  - No authentication process, directly into the Socks command sending phase
  `` `
  + ---- + ---- + ---- + ---- + ---- + ---- + ---- + ---- + ---- + ---- + .... + ---- +
  | VN | CD | DSTPORT |      DSTIP        | USERID       |NULL|
  + ---- + ---- + ---- + ---- + ---- + ---- + ---- + ---- + ---- + ---- + .... + ---- +
  |  1 |  1 |    2    |         4         | variable|    |  1 |
  + ---- + ---- + ---- + ---- + ---- + ---- + ---- + ---- + ---- + ---- + .... + ---- +
  `` `

- The various fields of the Socks command are explained as follows:
  - VER / VN: Socks protocol version number, 0x04 or 0x05
  - CMD / CD: Socks command word
    - 0x01：CONNECT
    - 0x02：BIND
    - 0x03：UDP Associate
  - RSV: reserved bytes
  - ATYP: Target IP Type
    - 0x01: IPv4
    - 0x03：Domain Name
    - 0x04：IPv6
  - DST.ADDR: Depending on the type of ATYP, it may be IPv4, IPv6 address or [len]"domain name"
  - DST.PORT / DSTPORT: Destination port (network byte order)
  - DSTIP: IPv4 address
  - USERID: Username, length 0 bytes means no username
  - NULL: its value is 0x00, which is used to indicate the end of the username.

```
int
SocksEntry::mainEvent(int event, void *data)
{
  int ret = EVENT_DONE;
  int n_bytes = 0;
  unsigned char *p;

  switch (event) {
  case NET_EVENT_OPEN:
    buf->reset();
    unsigned short ts;
    p = (unsigned char *)buf->start();
    ink_assert(netVConnection);

    if (auth_handler) {
      // socks5BasicAuthHandler is responsible for populating the authentication header
      // No authentication: 0x05 0x01 0x00
      //   0x05: Version
      // 0x01: 1 Methrods
      // 0x00: Method=0 No authentication
      // There is certification: 0x05 0x02 0x00 0x02
      //   0x05: Version
      // 0x02: 2 Methods
      // 0x00: Method=0 No authentication
      // 0x02: Method=2 username and password authentication
      // After reading the authentication header, read two bytes from the Socks server
      // No authentication: 0x05 0x00
      //   0x05: Version
      // 0x00: no authentication
      // Then auth_handler is set to NULL and enter the following Socks command processing stage
      // There is certification: 0x05 0x02
      //   0x05: Version
      // 0x02: Username password authentication, if the second byte is 0xff, the authentication mode negotiation fails.
      // Then switch to socks5PasswdAuthHandler to fill in the username and password:
      //     0x01 [ulen] "username" [plen] "password"
      // 0x01: Version, this is not the version of the Socks protocol, but the version of the username and password transfer protocol.
      // ulen: username length (1 ~ 255)
      // username: username
      // plen: password length (1 ~ 255)
      // password: password
      // Then socks5PasswdAuthHandler is responsible for verifying the authentication result:
      // 0x01 0x00
      // 0x01: Version, this is not the version of the Socks protocol, but the version of the username and password transfer protocol.
      // 0x00: Authentication succeeds, other values ​​fail authentication, TCP connection must be closed
      // After the authentication is successful, auth_handler is set to NULL and enters the following Socks command processing stage.
      n_bytes = invokeSocksAuthHandler(auth_handler, SOCKS_AUTH_OPEN, p);
    } else {
      // Socks command construction stage
      // Debug("Socks", " Got NET_EVENT_OPEN to SOCKS server\n");

      p[n_bytes++] = version;
      p[n_bytes++] = (socks_cmd == NORMAL_SOCKS) ? SOCKS_CONNECT : socks_cmd;
      // BUG1: Since the port should be filled in according to the network byte order, ntohs() should not be called.
      // BUG2: This should get the target port information from target_addr to fill in. A commit that supports IPv6 in ATS causes this problem to occur.
      ts = ntohs(ats_ip_port_cast(&server_addr));

      if (version == SOCKS5_VERSION) {
        // Socks5 protocol IP address first
        p[n_bytes++] = 0; // Reserved
        if (ats_is_ip4(&server_addr)) {
          p[n_bytes++] = 1; // IPv4 addr
          memcpy(p + n_bytes, &server_addr.sin.sin_addr, 4);
          n_bytes += 4;
        } else if (ats_is_ip6(&server_addr)) {
          // In init(), there is an assert that the target IP must be IPv4
          // Obviously IPv6 is supported under the Socks v5 protocol.
          p[n_bytes++] = 4; // IPv6 addr
          memcpy(p + n_bytes, &server_addr.sin6.sin6_addr, TS_IP6_SIZE);
          n_bytes += TS_IP6_SIZE;
        } else {
          Debug("Socks", "SOCKS supports only IP addresses.");
        }
      }

      // fill in the port
      memcpy(p + n_bytes, &ts, 2);
      n_bytes += 2;

      if (version == SOCKS4_VERSION) {
        // Socks4 protocol IP address is behind
        if (ats_is_ip4(&server_addr)) {
          // for socks4, ip addr is after the port
          memcpy(p + n_bytes, &server_addr.sin.sin_addr, 4);
          n_bytes += 4;

          p[n_bytes++] = 0; // NULL
        } else {
          Debug("Socks", "SOCKS v4 supports only IPv4 addresses.");
        }
      }
      // BUG2: End, you can get the repaired version from 6.2.x
    }

    buf->fill(n_bytes);

    // BUG3: The connection timeout was set before. The current setting should be the socks communication timeout. You should cancel the previous timeout and reset the new timeout.
    if (!timeout) {
      /* timeout would be already set when we come here from StartEvent() */
      timeout = this_ethread()->schedule_in(this, HRTIME_SECONDS(netProcessor.socks_conf_stuff->socks_timeout));
    }

    // Send authentication negotiation, authentication information, CONNECT command
    netVConnection->do_io_write(this, n_bytes, reader, 0);
    // Debug("Socks", "Sent the request to the SOCKS server\n");

    ret = EVENT_CONT;
    break;

  case VC_EVENT_WRITE_READY:

    ret = EVENT_CONT;
    break;

  case VC_EVENT_WRITE_COMPLETE:
    // The processing here is a bit fuzzy
    // WRITE_COMPLETE Case possible Display:
    // 1. Send authentication negotiation request (Socks v5)
    // 2. Send username and password (Socks v5)
    // 3. Send the Socks command
    / / The first write operation, may be one of 1, 3, the first cancellation is Socks connect timeout
    // Therefore, if any of 1, 3 is sent, the Socks connection is considered established.
    if (timeout) {
      timeout->cancel(this);
      timeout = NULL;
      write_done = true;
    }

    // heavy buffer
    buf->reset(); // Use the same buffer for a read now

    // auth_handler is not NULL to indicate that it is in the authentication process, and then to receive the authentication information returned by Socks Server
    // Otherwise, it means it is in the Socks command flow, and then it will receive the command execution returned by Socks Server.
    if (auth_handler)
      n_bytes = invokeSocksAuthHandler(auth_handler, SOCKS_AUTH_WRITE_COMPLETE, NULL);
    // If it is a CONNECT command
    else if (socks_cmd == NORMAL_SOCKS)
      // Calculate the length of information that needs to be received from the Socks Server based on the Socks protocol version
      // SOCKS4_REP_LEN is 8 bytes, fixed length
      // SOCKS5_REP_LEN is 262 bytes, which is the maximum information length under the Socks v5 protocol.
      n_bytes = (version == SOCKS5_VERSION) ? SOCKS5_REP_LEN : SOCKS4_REP_LEN;
    // If it is not a CONNECT command
    else {
      / / Transparent transmission of the connection through the tunnel
      Debug("Socks", "Tunnelling the connection");
      // let the client handle the response
      // Call the state machine NET_EVENT_OPEN via free() and pass VC
      // The state machine here should be SocksProxy and the handler is SocksProxy::mainEvent
      // SocksProxy sets its state before calling connect_re() state = SERVER_TUNNEL
      free();
      // It will be better to write return here.
      break;
    }

    // Here is the Socks communication timeout, for example:
    // 1. Wait for Socks Server to return the authentication negotiation result
    // 2. Wait for Socks Server to return the authentication result of the username and password.
    // 3. Wait for Socks Server to return the execution result of the Socks command
    timeout = this_ethread()->schedule_in(this, HRTIME_SECONDS(netProcessor.socks_conf_stuff->socks_timeout));

    // Prepare to receive information returned by Socks Server
    netVConnection->do_io_read(this, n_bytes, buf);

    ret = EVENT_DONE;
    break;

  case VC_EVENT_READ_READY:
    ret = EVENT_CONT;

    // If you are reading the command execution result of Socks v5
    // auth_handler is NULL to indicate that authentication has been completed
    if (version == SOCKS5_VERSION && auth_handler == NULL) {
      // Due to the implementation of Socks v5, it is divided into IPv4, IPv6 and FQHN.
      // In three cases, the data length of the returned result is different.
      // Determine and correct the total length of the data to be read based on the contents of the first 5 bytes.
      VIO * vio = (VIO *) data;
      p = (unsigned char *)buf->start();

      if (vio-> ndone> = 5) {
        int reply_len;

        switch (p[3]) {
        case SOCKS_ATYPE_IPV4:
          reply_len = 10;
          break;
        case SOCKS_ATYPE_FQHN:
          reply_len = 7 + p[4];
          break;
        case SOCKS_ATYPE_IPV6:
          Debug("Socks", "Who is using IPv6 Addr?");
          reply_len = 22;
          break;
        default:
          reply_len = INT_MAX;
          Debug("Socks", "Illegal address type(%d) in Socks server", (int)p[3]);
        }

        // The data that has been read so far has reached the required length
        if (vio->ndone >= reply_len) {
          vio-> nbytes = vio-> ndone;
          ret = EVENT_DONE;
        }
      }
    }

    // if:
    //   1. Socks v4
    // 2. Socks v5 and in the process of receiving authentication information
    // 3. Socks v5 waits for the command execution result, but does not complete the data read
    // then return to continue receiving data
    if (ret == EVENT_CONT)
      break;
  // Fall Through
  case VC_EVENT_READ_COMPLETE:
    // cancel the previously set timeout
    if (timeout) {
      timeout->cancel(this);
      timeout = NULL;
    }
    // Debug("Socks", "Successfully read the reply from the SOCKS server\n");
    p = (unsigned char *)buf->start();

    if (auth_handler) {
      / / Processing authentication logic
      SocksAuthHandler temp = auth_handler;

      if (invokeSocksAuthHandler(auth_handler, SOCKS_AUTH_READ_COMPLETE, p) < 0) {
        lerrno = ESOCK_DENIED;
        free();
      } else if (auth_handler != temp) {
        // here either authorization is done or there is another
        // stage left.
        mainEvent(NET_EVENT_OPEN, NULL);
      }

    } else {
      / / Handle the Socks command logic
      bool success;
      if (version == SOCKS5_VERSION) {
        success = (p[0] == SOCKS5_VERSION && p[1] == SOCKS5_REQ_GRANTED);
        Debug("Socks", "received reply of length %" PRId64 " addr type %d", ((VIO *)data)->ndone, (int)p[3]);
      } else
        success = (p[0] == 0 && p[1] == SOCKS4_REQ_GRANTED);

      // ink_assert(*(p) == 0);
      / / According to the results returned by the Socks Server to determine whether the command was successfully executed
      // set lerrno if it fails
      if (!success) { // SOCKS request failed
        Debug("Socks", "Socks request denied %d", (int)*(p + 1));
        lerrno = ESOCK_DENIED;
      } else {
        Debug("Socks", "Socks request successful %d", (int)*(p + 1));
        lerrno = 0;
      }
      // Call the state machine via free() according to the value of lerrno
      free();
    }

    break;

  // The following timeouts and error handling, please see the analysis of the following two paragraphs
  case EVENT_INTERVAL:
    timeout = NULL;
    if (write_done) {
      lerrno = ESOCK_TIMEOUT;
      free();
      break;
    }
  /* else
     This is server_connect_timeout. So we treat this as server being
     down.
     Should cancel any pending connect() action. Important on windows

     fall through
   * /
  case VC_EVENT_ERROR:
    /*This is mostly ECONNREFUSED on Unix */
    SET_HANDLER(&SocksEntry::startEvent);
    startEvent(NET_EVENT_OPEN_FAILED, NULL);
    break;

  // callback state machine OPEN_FAILED for EOS / INACTIVITY_TIMEOUT / ACTIVE_TIMEOUT
  / / Therefore the value of Inactivity Timeout will affect the establishment of the entire Socks connection
  case VC_EVENT_EOS:
  case VC_EVENT_INACTIVITY_TIMEOUT:
  case VC_EVENT_ACTIVE_TIMEOUT:
    Debug("Socks", "VC_EVENT error: %s", get_vc_event_name(event));
    lerrno = ESOCK_NO_SOCK_SERVER_CONN;
    free();
    break;
  default:
    // BUGBUG:: could be active/inactivity timeout ...
    ink_assert(!"bad case value");
    Debug("Socks", "Bad Case/Net Error Event");
    lerrno = ESOCK_NO_SOCK_SERVER_CONN;
    free();
    break;
  }

  return right;
}
```

Timeout management in ### SocksEntry::mainEvent()

Events related to timeout processing:

- NET_EVENT_OPEN
  - Execute do_io_write() and set the timeout to socks_timeout
  - The timeout period for sending messages to the Socks Server is set.

- WRITE_COMPLETE
  - Execute do_io_read() and set the timeout to socks_timeout
  - This is the timeout for waiting for the Socks Server to respond.

- READ_COMPLETE
  - If it is found that it is necessary to continue sending data to the Socks Server, the mainEvent is called recursively into the processing flow of NET_EVENT_OPEN.

- Cancel the previously set timeout before setting a new timeout
  - The previous timeout settings are canceled in both READ_COMPLETE and WRITE_COMPLETE
  - Since NET_EVENT_OPEN will only set a new timeout if no timeout is set, there is no need to cancel the previous timeout setting first.

Through the above analysis, you can clearly see that socks_timeout is the timeout for each Socks request and Socks response.

### Retrying after connecting with Socks Server

In the processing of WRITE_COMPLETE,
  - will set write_done to true
  - Used to indicate that a Socks request has been sent and is waiting to receive a Socks response.
In the processing of EVENT_INTERVAL,
  - If find_done is true, it usually means socks_timeout, then callback state machine OPEN_FAILED
  - If write_done is false, it usually means server_connect_timeout, and restarts startEvent() to re-establish TCP connection.

That is, in SocksEntry,

- If the connection to the Socks server times out, it will be retried
- If timeout occurs during Socks communication, it will directly call back state machine OPEN_FAILED


## References

- [P_Socks.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_Socks.h)
- [Socks.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/Socks.cc)
- [ParentSelection.cc](http://github.com/apache/trafficserver/tree/master/proxy/ParentSelection.cc)
- SOCKS on Wikipedia
  - [中文] (https://zh.wikipedia.org/wiki/SOCKS)
  - [English](https://en.wikipedia.org/wiki/SOCKS)
- RFCs
  - [Socks v4](https://www.openssh.com/txt/socks4.protocol)
  - [Socks v5](https://tools.ietf.org/html/rfc1928)
  - [Socks v5 username/password authentication](https://tools.ietf.org/html/rfc1929)
- [Sample code](http://howtofix.pro/c-establish-connections-through-socks45-using-winsock2/)
