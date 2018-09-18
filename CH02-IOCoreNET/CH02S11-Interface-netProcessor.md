# Interface: netProcessor

A globally unique instance of the NetProcessor type is declared in the ATS: netProcessor, which can be manipulated when we need to use network I/O operations, for example:

  - netProcessor.allocate_vc
    - Create a new NetVC
  - netProcessor.accept
    - Receive a new connection from the client
  - netProcessor.main_accept
    - Basically the same as the accept function, except that it requires the caller to pass in a listen fd that has completed the bind operation.
  - netProcessor.connect_re
    - Initiate a new connection to the server, call the state machine immediately after issuing the TCP SYN (the connection may not be successfully established at this time)
  - netProcessor.connect_s
    - Initiate a new connection to the server, but only call back the state machine after the connection is successfully established (via the CheckConnect substate machine)

netProcessor is also responsible for creating state machines such as NetHandler for each thread to handle NetVC read and write in the ET_NET thread group.

Now that we have basically introduced the various parts of the network subsystem, let's compare Net Sub-System with EventSystem and find out:

- Think of NetHandler as EThread of EventSystem
  - NetHandler manages multiple queues that save NetVC
  - EThread manages multiple queues that hold Events
- Think of NetVConnection as an Event of EventSystem
  - NetHandler traverses NetVC's queue, callback is associated with NetVC state machine
  - EThread traverses the queue of the Event, callback the state machine associated with the Event
- Think of NetProcessor as EventProcessor of EventSystem
  - NetProcessor provides the ability to create NetVC objects (EventProcessor provides the ability to create Event objects)
  - NetProcessor provides the ability to create and initialize NetHandler objects (EventProcessor provides the ability to create EThread objects)
- For scheduling functions, compare to EventSystem's schedule_\*() method
  - NetProcessor provides accept and connect to put the new NetVC into the queue of NetHandler
  - NetVConnection provides methods such as reenable() and do_io() to put NetVC into the queue of NetHandler
  - NetHandler provides methods such as ::read_disable(), ::write_disable(), class EventIO, etc. to implement operations on its queues.

## Base class NetProcessor

NetProcessor inherits from the base class Processor and is an API collection provided by IOCoreNet.

The base class NetProcessor simply defines the API, and the implementation is done through the UnixNetProcessor inheritance class.

### Definition

```
/ **
  This is the heart of the Net system. Provides common network APIs,
  like accept, connect etc. It performs network I/O on behalf of a
  state machine.
* /
class NetProcessor : public Processor
{
public:
  /** Options for @c accept.
    Define Options for the accept method
   * /
  struct AcceptOptions {
    typedef AcceptOptions self; ///< Self reference type.

    /// Port on which to listen.
    /// 0 => don't care, which is useful if the socket is already bound.
    int local_port;
    /// Local address to bind for accept.
    /// If not set -> any address.
    IpAddr local_ip;
    /// IP address family.
    /// @note Ignored if an explicit incoming address is set in the
    /// the configuration (@c local_ip). If neither is set IPv4 is used.
    int ip_family;
    /// Should we use accept threads? If so, how many?
    int accept_threads;
    /// Event type to generate on accept.
    EventType etype;
    /** If @c true, the continuation is called back with
        @c NET_EVENT_ACCEPT_SUCCEED
        or @c NET_EVENT_ACCEPT_FAILED on success and failure resp.
    * /
    bool f_callback_on_open;
    /** Accept only on the loopback address.
        Default: @c false.
     * /
    bool localhost_only;
    /// Are frequent accepts expected?
    /// Default: @c false.
    bool frequent_accept;
    bool backdoor;

    /// Socket receive buffer size.
    /// 0 => OS default.
    int recv_bufsize;
    /// Socket transmit buffer size.
    /// 0 => OS default.
    int send_bufsize;
    /// Socket options for @c sockopt.
    /// 0 => do not set options.
    uint32_t sockopt_flags;
    uint32_t packet_mark;
    uint32_t packet_tos;

    /** Transparency on client (user agent) connection.
        @internal This is irrelevant at a socket level (since inbound
        transparency must be set up when the listen socket is created)
        but it's critical that the connection handling logic knows
        whether the inbound (client / user agent) connection is
        transparent.
    * /
    bool f_inbound_transparent;

    /// Default constructor.
    /// Instance is constructed with default values.
    AcceptOptions() { this->reset(); }
    /// Reset all values to defaults.
    self &reset();
  };

  / **
    Accept connections on a port.
    Callbacks:
      - cont->handleEvent( NET_EVENT_ACCEPT, NetVConnection *) is
        called for each new connection
      - cont->handleEvent(EVENT_ERROR,-errno) on a bad error
    Re-entrant callbacks (based on callback_on_open flag):
      - cont->handleEvent(NET_EVENT_ACCEPT_SUCCEED, 0) on successful
        accept init
      - cont->handleEvent(NET_EVENT_ACCEPT_FAILED, 0) on accept
        init failure
    @param cont Continuation to be called back with events this
      continuation is not locked on callbacks and so the handler must
      be re-entrant.
    @param opt Accept options.
    @return Action, that can be cancelled to cancel the accept. The
      port becomes free immediately.
   * /
  inkcoreapi virtual Action *accept(Continuation *cont, AcceptOptions const &opt = DEFAULT_ACCEPT_OPTIONS);

  / **
    Accepts incoming connections on port. Accept connections on port.
    Accept is done on all net threads and throttle limit is imposed
    if frequent_accept flag is true. This is similar to the accept
    method described above. The only difference is that the list
    of parameter that is takes is limited.
    Callbacks:
      - cont->handleEvent( NET_EVENT_ACCEPT, NetVConnection *) is called for each new connection
      - cont->handleEvent(EVENT_ERROR,-errno) on a bad error
    Re-entrant callbacks (based on callback_on_open flag):
      - cont->handleEvent(NET_EVENT_ACCEPT_SUCCEED, 0) on successful accept init
      - cont->handleEvent(NET_EVENT_ACCEPT_FAILED, 0) on accept init failure
    @param cont Continuation to be called back with events this
      continuation is not locked on callbacks and so the handler must
      be re-entrant.
    @param listen_socket_in if passed, used for listening.
    @param opt Accept options.
    @return Action, that can be cancelled to cancel the accept. The
      port becomes free immediately.
  * /
  virtual Action *main_accept(Continuation *cont, SOCKET listen_socket_in, AcceptOptions const &opt = DEFAULT_ACCEPT_OPTIONS);

  // The Options used by the Connect method are defined in I_NetVConnection.h
  / **
    Open a NetVConnection for connection oriented I/O. Connects
    through sockserver if netprocessor is configured to use socks
    or is socks parameters to the call are set.
    Re-entrant callbacks:
      - On success calls: c->handleEvent(NET_EVENT_OPEN, NetVConnection *)
      - On failure calls: c->handleEvent(NET_EVENT_OPEN_FAILED, -errno)
    @note Connection may not have been established when cont is
      call back with success. If this behaviour is desired use
      synchronous connect connet_s method.
    @see connect_s()
    @param cont Continuation to be called back with events.
    @param addr target address and port to connect to.
    @param options @see NetVCOptions.
  * /

  inkcoreapi Action *connect_re(Continuation *cont, sockaddr const *addr, NetVCOptions *options = NULL);

  / **
    Open a NetVConnection for connection oriented I/O. This call
    is simliar to connect method except that the cont is called
    back only after the connections has been established. In the
    case of connect the cont could be called back with NET_EVENT_OPEN
    event and OS could still be in the process of establishing the
    connection. Re-entrant Callbacks: same as connect. If unix
    asynchronous type connect is desired use connect_re().
    @param cont Continuation to be called back with events.
    @param addr Address to which to connect (includes port).
    @param timeout for connect, the cont will get NET_EVENT_OPEN_FAILED
      if connection could not be established for timeout msecs. The
      default is 30 secs.
    @param options @see NetVCOptions.
    @see connect_re()
  * /
  Action *connect_s(Continuation *cont, sockaddr const *addr, int timeout = NET_CONNECT_TIMEOUT, NetVCOptions *opts = NULL);

  / **
    After the EventSystem is started, netProcessor.start() is called by the main() function to install a state machine such as NetHandler for each EThread.
    You can then use the other APIs provided by netProcessor to implement specific network I/O operations.
    Starts the Netprocessor. This has to be called before doing any
    other net call.
    @param number_of_net_threads is not used. The net processor
      uses the Event Processor threads for its activity.
      This parameter is used to specify the number of ET_NET, but the number of ET_NET is always the same as the number of EThreads.
  * /
  virtual int start(int number_of_net_threads, size_t stacksize) = 0;

  inkcoreapi virtual NetVConnection *allocate_vc(EThread *) = 0;

  /** Private constructor. */
  netprocess is () {};

  /** Private destructor. */
  virtual ~NetProcessor(){};

  /** This is MSS for connections we accept (client connections). */
  static int accept_mss;

  //
  // The following are required by the SOCKS protocol:
  //
  // Either the configuration variables will give your a regular
  // expression for all the names that are to go through the SOCKS
  // server, or will give you a list of domain names which should *not* go
  // through SOCKS. If the SOCKS option is set to false then, these
  // variables (regular expression or list) should be set
  // appropriately. If it is set to TRUE then, in addition to supplying
  // the regular expression or the list, the user should also give the
  // the ip address and port number for the SOCKS server (use
  // appropriate defaults)

  /* shared by regular netprocessor and ssl netprocessor */
  // The connect method supports the connection to the destination IP through the socks server.
  // This static variable is used to save the Socks configuration information used globally.
  static socks_conf_struct *socks_conf_stuff;

  /// Default options instance.
  // This static variable is used to hold the default AcceptOptions
  static AcceptOptions const DEFAULT_ACCEPT_OPTIONS;

private:
  /** @note Not implemented. */
  virtual int
  stop()
  {
    ink_release_assert(!"NetProcessor::stop not implemented");
    return 1;
  }

  NetProcessor(const NetProcessor &);
  NetProcessor &operator=(const NetProcessor &);
};


/ **
  Global NetProcessor singleton object for making net calls. All
  net processor calls like connect, accept, etc are made using this
  object.
  @code
    netProcesors.accept(my_cont, ...);
    netProcessor.connect_re(my_cont, ...);
  @endcode
* /
External InkCoreapi NetProcessor & NetProcessor;
```

### References

- [I_NetProcessor.h](https://github.com/apache/trafficserver/tree/master/iocore/net/I_NetProcessor.h)

## Inheriting UnixNetProcessor

UnixNetProcessor is a concrete implementation of NetProcessor on a Unix-like system, while NetProcessor simply provides an interface definition to external users.

### Definition

```
struct UnixNetProcessor : public NetProcessor {
public:
  // internal methods are used to implement the accept method
  virtual Action *accept_internal(Continuation *cont, int fd, AcceptOptions const &opt);

  // Internal method: used to implement the connect_re method
  Action *connect_re_internal(Continuation *cont, sockaddr const *target, NetVCOptions *options = NULL);
  // Internal method: used to initiate a connection
  Action *connect(Continuation *cont, UnixNetVConnection **vc, sockaddr const *target, NetVCOptions *opt = NULL);

  // Virtual function allows etype to be upgraded to ET_SSL for SSLNetProcessor.  Does
  // nothing for NetProcessor
  // Internal method: This is used to provide SSL support. This method will be overridden in SSLNetProcessor.
  virtual void upgradeEtype(EventType & /* etype ATS_UNUSED */){};

  // Internal method: Create a NetAccept object
  // Actually it should be creating a UnixNetAccept object and then returning the base class NetAccept type
  // Since the support for the Windows version has been chopped off, there is no NTNetAccept object
  virtual NetAccept *createNetAccept();
  // Create a UnixNetVConnection object, but return the type of the base class
  virtual NetVConnection *allocate_vc(EThread *t);

  // Create an ET_NET thread group and for each ET_NET thread group
  // Create a NetHandler state machine for all EThreads and peripheral components that work with NetHandler
  virtual int start(int number_of_net_threads, size_t stacksize);

  // This member should be populated somewhere, and when the connection limit is reached, the padding information is sent to the client.
  // But there is no operation to fill the information in the current code. The guess should be to read a configuration item in start().
  char *throttle_error_message;
  // This member is not currently using it.
  // In the code for branch 2.0.x, UnixNetProcessor also has an atomic queue for NetAccept:
  //     ASLL(NetAccept, link) accepts_on_thread
  // Feel that all NetAccept objects on this queue will be processed while a NetAccept is running
  // Guess: In the earlier version, there was ET_ACCEPT. If you are interested, you can look at the code of 2.0.x.
  // This should be the Event that holds the super NetAccept state machine dedicated to handling the NetAccept queue.
  Event *accept_thread_event;

  // offsets for per thread data structures
  // Record the offset of the netHandler and pollCont objects in the thread stack
  off_t netHandler_offset;
  off_t pollCont_offset;

  // we probably wont need these members
  // The following two variables are only used in the start() function and can actually be removed from the class members.
  // The number of threads in the current ET_NET thread group
  int n_netthreads;
  // pointer array to the ET_NET thread group object in eventProcessor
  EThread **netthreads;
};
```

### References

- [P_UnixNetProcessor.h](https://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetProcessor.h)
- [UnixNetProcessor.cc](https://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetProcessor.cc)
