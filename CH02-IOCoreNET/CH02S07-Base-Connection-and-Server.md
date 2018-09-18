# Base component: Connection

Used to define a network connection and client side.

# definition

```
struct Connection {
  SOCKET fd; ///< The fd descriptor of the connection is actually an int type
  IpEndpoint addr; ///< associated address information
  Bool is_bound; ///< True= has been done with the local IP address, usually refers to the Connection::open () method, rather than the bind () operation when listen
  Bool is_connected; ///< True=A link has been established, usually referring to the Connection::connect() method. For non-blocking IO, the connection may still be established.
  Int sock_type; ///< If it is a UDP connection: SOCK_DGRAM, if it is a TCP link: SOCK_STREAM

  / / Create and initialize a socket handle / description word
  // You can pass a parameter of type NetVCOptions to set some features of this socket.
  // Pass the 'same' NetVCOptions type parameter when passing to the connect() method
  //     ALL：TPROXY，sendbuf，revcbuf，call apply_options(opt)
  // return 0 for success,
  // Less than 0 means failure, its value is -ERRNO
  int open(NetVCOptions const &opt = DEFAULT_OPTIONS ///< Socket options.
           );

  / / Use the socket handle / description word to initiate a connection to the specified IP:PORT
  // Execute the target IP and PORT to connect by the sockaddr type parameter
  // Set the connection to be 'blocking' or 'non-blocking' by using the NetVCOptions type parameter.
  // Pass the 'same' NetVCOptions type parameter when passing to the open() method
  // return 0 for success,
  // Less than 0 means failure, its value is -ERRNO
  int connect(sockaddr const *to,                       ///< Remote address and port.
              NetVCOptions const &opt = DEFAULT_OPTIONS ///< Socket options
              );

  / / Set the socket address structure member used internally
  // For ICP only
  void
  setRemote(sockaddr const *remote_addr ///< Address and port.
            )
  {
    ats_ip_copy(&addr, remote_addr);
  }

  / / Set the Send of MultiCast
  int setup_mc_send(sockaddr const *mc_addr, sockaddr const *my_addr, bool non_blocking = NON_BLOCKING, unsigned char mc_ttl = 1,
                    bool mc_loopback = DISABLE_MC_LOOPBACK, Continuation *c = NULL);

  // Set the Receive of MultiCast
  int setup_mc_receive(sockaddr const *from, sockaddr const *my_addr, bool non_blocking = NON_BLOCKING, Connection *sendchan = NULL,
                       Continuation *c = NULL);

  // Close the socket handle/descriptor created by open()
  // return 0 for success,
  // Less than 0 means failure, its value is -ERRNO
  int close(); // 0 on success, -errno on failure

  / / Set special parameters
  //     TCP：SOCK_OPT_NO_DELAY，SOCK_OPT_KEEP_ALIVE，SOCK_OPT_LINGER_ON
  //     ALL：SO_MARK，IP_TOS/IPV6_TCLASS
  void apply_options(NetVCOptions const &opt);

  // Destructor
  // call close() to close the socket
  // 重置成员：fd=NO_FD，is_bound=false，is_connected=false
  virtual ~Connection();
  
  // Constructor
  // Initialize members: fd=NO_FD, is_bound=false, is_connected=false, sock_type=0, addr memory area padding 0
  Connection();

  / / NetVCOptions type default value (static constant, read only)
  // The member is called by the constructor to call NetVCOptions::reset() to complete the initialization of the default value (P_UnixNetVConnection.h)
  static NetVCOptions const DEFAULT_OPTIONS;

protected:
  // Cleanup function: used with the cleaner class template
  // call close() to close the socket
  void _cleanup();
};
```

## About the cleaner template in UnixConnection.cc

A cleaner template class is defined in [UnixConnection.cc] (https://github.com/apache/trafficserver/tree/master/iocore/net/UnixConnection.cc):

```
// This cleaner template can only be used in UnixConnection.cc because of the definition of an unnamed namespace
namespace
{
 /* This is a structure to facilitate resource cleanup
    Usually method is a method called to clean up and release resources when object is destructed.
    However, the cleaner::reset method can be used to avoid calling the method method during destructuring.
    This feature may not be useful in the process of assigning, checking, and returning processes.
    But it is useful for the following situations:
      - There are multiple resources (each can have its own resource cleanup method)
      - Multiple checkpoints
    At this point, you can create a cleaner at the time of allocation, so you don't have to check the resources in all the places you need.
        If the creation method is successful, the reset method is called. This branch usually has only one place in the code.
        Otherwise, the normal destructor is executed, releasing resources.
    example:
    self::some_method (...) {
      // Allocating resources
      cleaner<self> clean_up(this, &self::cleanup);
      // Modify or check resources
      if (fail)
        Return FAILURE; // Failed, automatically calls cleanup(), the resource is released
      // Success
      Clean_up.reset(); // Call the reset() method to prevent the destruction, and cleanup() will not be called.
      return SUCCESS;
  * /
template <typename T> struct cleaner {
  T *obj;                      ///< Object instance.
  typedef void (T::*method)(); ///< Method signature.
  method m;

  cleaner(T *_obj, method _method) : obj(_obj), m(_method) {}
  ~cleaner()
  {
    if (obj)
      (obj->*m)();
  }
  void
  reset()
  {
    obj = 0;
  }
};
}
```


#基组件:Server

Inherited from the Connection base class, used to define a server side.

# definition

```
struct Server : public Connection {
  // Prepare to accept the local IP address of the client connection
  IpEndpoint accept_addr;

  // True = incoming connections are transparent, corresponding to ATS tr-in
  bool f_inbound_transparent;

  // True = 使用kernel的HTTP accept filter
  // This is an advanced filter on the BSD system. Linux only has DEFER_ACCEPT similar to it, so on Linux systems, this value is usually false.
  // Its function is probably (I haven't used it, not 100% sure)
  // 1. Complete the TCP handshake in the kernel.
  // 2. Then read the first request packet,
  // 3. is an HTTP request, returning from the accept() call
  // 4. Not an HTTP request, close the TCP connection, wait for the next connection, or accept() returns an error
  bool http_accept_filter;

  //
  // Use this call for the main proxy accept
  //
  // This is not defined and is not used in the ATS code.
  int proxy_listen(bool non_blocking = false);

  // accept a new connection
  // Use this->fd, which was created before the listen fd to execute accept()
  // The newly connected descriptor is stored in c->fd
  // Set FD_CLOEXEC, nonblocking, SEND_BUF_SIZE for the new link
  // return 0 for success
  // Less than 0 means failure, its value is -ERRNO
  int accept(Connection *c);

  // Create a listen fd that can be used for accept()
  // populate the addr member with accept_addr
  // Create a TCP socket, SOCK_STREAM, set to non blocking
  // bind to IP and PORT via bind()
  // Next, you can set some necessary parameters with setup_fd_for_listen() and then call accept() to accept the new connection.
  int listen(bool non_blocking = false, int recv_bufsize = 0, int send_bufsize = 0, bool transparent = false);
  
  / / Set the parameters of the member fd
  //     send buf size, recv buf size, FD_CLOEXEC, SO_LINGER, IPV6_V6ONLY for ipv6
  //     SO_REUSEADDR, TCP_NODELAY, SO_KEEPALIVE, SO_TPROXY, TCP_MAXSEG, non blocking
  // Prepare to call the accept() method
  / / Before you first call listen () to create a socket
  int setup_fd_for_listen(bool non_blocking = false, int recv_bufsize = 0, int send_bufsize = 0,
                          bool transparent = false ///< Inbound transparent.
                          );

  // Constructor
  // Initialize the member via Connection(): fd=NO_FD, is_bound=false, is_connected=false, sock_type=0, addr memory area is filled with 0
  / / Initialize the member f_inbound_transparent = false, accept_addr memory area is filled with 0
  Server() : Connection(), f_inbound_transparent(false) { ink_zero(accept_addr); }
};
```

# References

- [P_Connection.h]
(https://github.com/apache/trafficserver/tree/master/iocore/net/P_Connection.h)
- [Connection.cc]
(https://github.com/apache/trafficserver/tree/master/iocore/net/Connection.cc)
- [UnixConnection.cc]
(https://github.com/apache/trafficserver/tree/master/iocore/net/UnixConnection.cc)
