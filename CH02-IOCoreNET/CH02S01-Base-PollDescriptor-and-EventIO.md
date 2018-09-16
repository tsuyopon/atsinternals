# Base components: PollDescriptor and EventIO

PollDescriptor is a package for I/O Poll operations on multiple platforms. At present, ATS supports epoll, kequeu, and port.

We can think of PollDescriptor as a queue:

- Put Socket FD into the poll fd / PollDescriptor queue
- Then take a batch of Socket FD from the poll fd / PollDescriptor queue through the Polling operation.
- And this is an atomic queue

EventIO is a member of the PollDescriptor queue and is the external interface provided by the PollDescriptor:

- Provide an EventIO for each Socket FD and encapsulate the Socket FD into EventIO
- Provides an interface to put EventIO itself into the PollDescriptor queue

The following only analyzes the implementation of support for the epoll ET mode.

## definition

Revisit the structure of linux pollfd, which will be used below.

```
source: man 2 poll
struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};
```

Revisit the structure of linux epoll_event, which will be used below.

```
source: man 2 epoll_wait
typedef union epoll_data {
    void    *ptr;
    int      fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;    /* Epoll events */
    epoll_data_t data;      /* User data variable */
};
```

The first is the PollDescriptor, which contains the array of handles and an array of saved result sets.

```
source: iocore/net/P_UnixPollDescriptor.h
// Define the maximum number of descriptors a PollDescriptor can handle.
// This is defined as 32K=32768 = 0x10000. If you want to modify this value, it is also recommended to use an integer multiple of 16.
// to ensure alignment of memory boundaries when storing this data.
#define POLL_DESCRIPTOR_SIZE 32768

typedef struct pollfd Pollfd;

struct PollDescriptor {
  int result; // result of poll
#if TS_USE_EPOLL
  // Used to save the epoll fd created by epoll_create
  int epoll_fd;

  // nfds and pfd are not used in the TCP section and should only be used in UDP.
  // Record how many fd are added to epoll fd
  Int nfds; // actual number
  // Each fd is saved to the pfd array before being added to epoll fd
  Pollfd pfd[POLL_DESCRIPTOR_SIZE];

   // used to save the fd state set returned by epoll_wait, the actual number of fd saved in result
   // Why use function internal variables in epoll_wait, but to allocate a memory area fixedly?
   // Since epoll_wait requires very high frequency operation, this memory area requires very high frequency allocation and release.
   // Even if it is defined as an internal array variable within a function, it will allocate space from the stack frequently.
   // So it's better to allocate a fixed memory area
  struct epoll_event ePoll_Triggered_Events[POLL_DESCRIPTOR_SIZE];
#endif
#if TS_USE_EPOLL
  // The following four macro definitions are a common set of methods, and the I/O poll operations for other systems such as kqueue are also abstracted into these four methods.
  // Get the I / O poll descriptor, for epoll is epoll fd
#define get_ev_port(a) ((a)->epoll_fd)
  // Get the current event state of the specified fd, get the event status of the returned fd through this method after epoll_wait
#define get_ev_events(a, x) ((a)->ePoll_Triggered_Events[(x)].events)
  // Get the data pointer of the specified fd binding, for epoll is the data.ptr of the epoll_event structure
#define get_ev_data(a, x) ((a)->ePoll_Triggered_Events[(x)].data.ptr)
  // Prepare to get the next fd, there is no corresponding operation for epoll
#define ev_next_event(a, x)
#endif

  // allocate an address space from pfd
  // Only used for epoll support for UDP, such as using I/O poll operations of other systems such as kqueue, returning NULL directly
  Pollfd *
  alloc()
  {
#if TS_USE_EPOLL
    // XXX : We need restrict max size based on definition.
    if (nfds >= POLL_DESCRIPTOR_SIZE) {
      nfds = 0;
    }
    return &pfd[nfds++];
#else
    return 0;
#endif
  }
private:
  // Initialize epoll fd and event array
  // return the current PollDescriptor
  PollDescriptor *
  init()
  {
    result = 0;
#if TS_USE_EPOLL
    nfds = 0;
    epoll_fd = epoll_create(POLL_DESCRIPTOR_SIZE);
    memset(ePoll_Triggered_Events, 0, sizeof(ePoll_Triggered_Events));
    memset(pfd, 0, sizeof(pfd));
#endif
    return this;
  }

public:
  // Constructor, call init() to complete initialization
  PollDescriptor() { init(); }
};
```

Then EventIO, add a file handle to poll fd, and modify the fd event status.

```
source:iocore/net/P_UnixNet.h
#define EVENTIO_READ (EPOLLIN | EPOLLET)
#define EVENTIO_WRITE (EPOLLOUT | EPOLLET)
#define EVENTIO_ERROR (EPOLLERR | EPOLLPRI | EPOLLHUP)  

struct PollDescriptor;
typedef PollDescriptor *EventLoop;

struct EventIO {
  int fd;
  EventLoop event_loop;
#define EVENTIO_NETACCEPT 1
#define EVENTIO_READWRITE_VC 2
#define EVENTIO_DNS_CONNECTION 3
#define EVENTIO_UDP_CONNECTION 4
#define EVENTIO_ASYNC_SIGNAL 5
  int type;
  union {
    Continuation *c; 
    UnixNetVConnection *vc;
    DNSConnection *dnscon;
    NetAccept *na;
    UnixUDPConnection *uc;
  } data;
  int start(EventLoop l, DNSConnection *vc, int events);
  int start(EventLoop l, NetAccept *vc, int events);
  int start(EventLoop l, UnixNetVConnection *vc, int events);
  int start(EventLoop l, UnixUDPConnection *vc, int events);

  // Underlying definition, should not be called directly
  int start(EventLoop l, int fd, Continuation *c, int events);

  // Change the existing events by adding modify(EVENTIO_READ)
  // or removing modify(-EVENTIO_READ), for level triggered I/O
  int modify(int events);

  // Refresh the existing events (i.e. KQUEUE EV_CLEAR), for edge triggered I/O
  int refresh(int events);

  int stop();
  int close();
  EventIO()
  {
    type = 0;
    data.c = 0;
  }
};
```

## Method

### start

When you need to add a file descriptor and file descriptor extension (NetVC, UDPVC, DNSConn, NetAccept) to the PollDescriptor, you can use the start method provided by EventIO:

- start(EventLoop l, DNSConnection *vc, int events)
   - type = EVENTIO_DNS_CONNECTION
- start(EventLoop l, NetAccept *vc, int events)
   - type = EVENTIO_NETACCEPT
- start(EventLoop l, UnixNetVConnection *vc, events)
   - type = EVENTIO_READWRITE_VC
- start(EventLoop l, UnixUDPVConnection *vc, events)
   - type = EVENTIO_UDP_CONNECTION

The above four methods will set the value of the member type, and finally call the basic method:

- start(EventLoop l, int afd, Continuation *vc, events)
- This basic method is to support multiple IO Poll platforms, such as: epoll, kqueue, port
- This method should usually not be called directly because this method does not set the type.
- Parameter Description:
   - EventLoop is actually a packaged PollDescriptor for poll fd.
   - Continuation is used for callbacks
   - Events indicates the event of interest, usually should use one of: EVENTIO_READ, EVENTIO_WRITE, EVENTIO_ERROR or their "or" value

```
TS_INLINE int
EventIO::start(EventLoop l, int afd, Continuation *c, int e)
{
  data.c = c;
  fd = afd;
  event_loop = l;
#if TS_USE_EPOLL
  struct epoll_event ev;
  memset(&ev, 0, sizeof(ev));
  ev.events = e;
  ev.data.ptr = this;
#ifndef USE_EDGE_TRIGGER
  events = e;
#endif
  // Finally, call epoll_ctl to add ev to the event pool of epoll fd, and then get the result via epoll_wait
  // A pointer to the EventIO instance is included in the ev, and there is a member of the EventIO type in each UnixNetVConnection.
  return epoll_ctl(event_loop->epoll_fd, EPOLL_CTL_ADD, fd, &ev);
#endif
}
```

### stop

The corresponding start method is the stop method, the stop method is only the implementation of epoll and port, there is no kqueue processing (I am not familiar with kqueue, I don't know why there is no processing for kqueue here):

```
TS_INLINE int
EventIO::stop()
{
  if (event_loop) {
    int retval = 0;
#if TS_USE_EPOLL
    struct epoll_event ev;
    memset(&ev, 0, sizeof(struct epoll_event));
    ev.events = EPOLLIN | EPOLLOUT | EPOLLET;
    // Remove fd from epoll fd via epoll_ctl
    retval = epoll_ctl(event_loop->epoll_fd, EPOLL_CTL_DEL, fd, &ev);
#endif
    // Set event_loop to NULL, because all operations must be done via epoll fd. Setting event_loop to NULL here is equivalent to no epoll fd.
    event_loop = NULL;
    return retval;
  }
  return 0;
}
```

### modify & refresh

Start and stop respectively add and delete, and then define two methods of modification and refresh, but for epoll ET mode, both methods are empty methods:

- modify
   - Modify, there are related code definitions for epoll LT, kqueue LT, port, so I will not analyze it here.
- refresh
   - Refresh, there are related code definitions for kqueue LT, port, so I will not analyze it here.

### close

Finally, there is a close method for closing fd. First, call the stop method to stop polling. Then, according to the type of type, call the close method of this type to complete the closing of fd.

However, the close method is only responsible for closing the DNS, NetAccept, and VC types. For UDP, you cannot call the close method. // Why? ? ?

```
TS_INLINE int
EventIO::close()
{
  stop();
  switch (type) {
  default:
    ink_assert(!"case");
  case EVENTIO_DNS_CONNECTION:
    return data.dnscon->close();
    break;
  case EVENTIO_NETACCEPT:
    return data.na->server.close();
    break;
  case EVENTIO_READWRITE_VC:
    return data.vc->con.close();
    break;
  }
  return -1;
}
```

The close method seems to have only the dns system call, and other systems don't see any calls to this method:

- In close_UnixNetVConnection, the stop method is called first, and then vc->con.close() is called by itself.
  - In fact, you can modify the call to the stop method to call the close method.
- In the processing of NetAccept, because the listen fd needs to be deleted from the epoll fd only when an error is encountered, this piece is rarely used.
  - Server.close() is called directly at the Lerror tag of acceptFastEvent. The listen fd is not deleted from epoll fd. It looks like a small bug???
  - In the cancel, server.close() is also called directly, and the listen fd is not deleted from the epoll fd.

## 参考资料

- man 2 poll
- man 2 epoll_wait
- [P_UnixPollDescriptor.h]
(http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixPollDescriptor.h)
- [P_UnixNet.h]
(http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
