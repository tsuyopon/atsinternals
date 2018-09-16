# Core component SSLNetAccept


SSLNetAccept inherits from NetAccept, which overloads some of NetAccept's methods and implements the functionality of SSLNetAccept.

Earlier, there are two ways to create NetVC:

  - One through NetAccept
  - One way through connectUp

So for each NetVC and its subclasses, there are corresponding implementations of NetAccept and connectUp.

SSLNetAccept is for NetAccept and is used to create SSLNetVConnection.

The following explanation will focus on the methods that are overloaded, so please understand the implementation of SSLNetAccept in conjunction with the NetAccept chapter.

## definition

```
//
// NetAccept
// Handles accepting connections.
//
struct SSLNetAccept : public NetAccept {
  virtual NetProcessor *getNetProcessor() const;
  virtual EventType getEtype() const;
  virtual void init_accept_per_thread(bool isTransparent);
  virtual NetAccept *clone() const;

  SSLNetAccept(){};

  virtual ~SSLNetAccept(){};
};

typedef int (SSLNetAccept::*SSLNetAcceptHandler)(int, void *);
```

## Method

### SSLNetAccept::getNetProcessor()

This method is used to return a global type instance of NetProcessor.

```
NetProcessor *
SSLNetAccept::getNetProcessor() const
{
  return &sslNetProcessor;
}
```

### SSLNetAccept::getEtype()

In NetAccept, this method is used to return the etype member. After the etype member is used to indicate the NetVC created by this NetAccept, it will be responsible for which thread group this NetVC is assigned to.

In the design of the ATS, there is a separate thread group ET_SSL that handles connections of the SSLNetVConnection type.

But when the ET_SSL dedicated thread group is set to 0 in the configuration file:

  - Then ET_SSL is set to the same value as ET_NET
  - So UnixNetVConnection and SSLNetVConnection will be handled by the same thread group

This method fixedly returns ET_SSL.

Why not return an etype member? ? ?

```
// Virtual function allows the correct
// etype to be used in NetAccept functions (ET_SSL
// or ET_NET).
EventType
SSLNetAccept::getEtype() const
{
  return SSLNetProcessor::ET_SSL;
}
```

### SSLNetAccept::init_accept_per_thread(bool isTransparent)

Create and initialize the SSLNetAccept state machine for the ET_SSL thread group.

As mentioned earlier, NetAccept has two modes of operation, the independent thread mode and the state machine mode. This is the state machine mode, and the corresponding state machine is created in the thread group.

Period is a negative number and therefore enters an implicit queue (negative queue).

```
void
SSLNetAccept::init_accept_per_thread(bool isTransparent)
{
  int i, n;
  NetAccept *a;

  if (do_listen(NON_BLOCKING, isTransparent))
    return;
  if (accept_fn == net_accept)
    SET_HANDLER((SSLNetAcceptHandler)&SSLNetAccept::acceptFastEvent);
  else
    SET_HANDLER((SSLNetAcceptHandler)&SSLNetAccept::acceptEvent);
  period = -HRTIME_MSECONDS(net_accept_period);
  n = eventProcessor.n_threads_for_type[SSLNetProcessor::ET_SSL];
  for (i = 0; i < n; i++) {
    if (i < n - 1)
      a = clone();
    else
      a = this;
    EThread *t = eventProcessor.eventthread[SSLNetProcessor::ET_SSL][i];

    PollDescriptor *pd = get_PollDescriptor(t);
    if (ep.start(pd, this, EVENTIO_READ) < 0)
      Debug("iocore_net", "error starting EventIO");
    a->mutex = get_NetHandler(t)->mutex;
    t->schedule_every(a, period, etype);
  }
}
```

### SSLNetAccept::clone()

This method is used in the above SSLNetAccept::init_accept_per_thread(), which differs from NetAccept::clone() in that:

  - new SSLNetAccept
  - Created is the SSLNetAccept object.

```
NetAccept *
SSLNetAccept::clone() const
{
  NetAccept *na;
  na = new SSLNetAccept;
  *na = *this;
  return na;
}
```

## Reference material

- [P_SSLNetAccept.h](https://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNetAccept.h)
- [SSLNetAccept.cc](https://github.com/apache/trafficserver/tree/master/iocore/net/SSLNetAccept.cc)
