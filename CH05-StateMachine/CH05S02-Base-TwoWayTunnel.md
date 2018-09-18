# Base component: TwoWayTunnel

OneWayTunnel was designed with the data in mind as a one-way flow. For a pair of VCs, data will be transferred from eastVC to westVC, and data will be transferred from westVC to eastVC.

By linking the two OneWayTunnels together, it is very simple to implement TwoWayTunnel.

##implementation

Suppose we are currently in a state machine, the state machine has got a successfully established clientVC, and the state machine has already started a serverVC.

When this state machine receives the NET_EVENT_OPEN event for this serverVC, it means that serverVC has successfully established a TCP connection with the server.

At this point we need to establish a TwoWayTunnel for the two VCs on the client and server side to achieve their interoperability.

First, create two OneWayTunnel

```
OneWayTunnel *c_to_s = OneWayTunnel::OneWayTunnel_alloc();
OneWayTunnel *s_to_c = OneWayTunnel::OneWayTunnel_alloc();

// Since we have done the do_io_read operation for clientVC, we have applied for MIOBuffer and the corresponding IOBufferReader is reader.
// So, call the second init() to initialize:
// only do_io_write on serverVC
// The incoming state machine is empty, at this time mutex=new()
c_to_s->init(clientVC, serverVC, NULL, clientVIO, reader);

// Then, call the first init initialization:
// execute do_io_read on serverVC
// execute do_io_write on clientVC
// Pass the cex of c_to_s into it, let the two OneWayTunnel share the same mutex
s_to_c->init(serverVC, clientVC, NULL, 0 , c_to_s->mutex);

// Finally, associate two tunnels
OneWayTunnel::SetupTwoWayTunnel(c_to_s, s_to_c);
```

At this point, the TwoWayTunnel is built. If the clientVC has data, it will be read into the internal MIOBuffer and then sent out when the serverVC is writable.
Similarly, when serverVC has data readable, it will be read into the internal MIOBuffer and then sent out when clientVC is writable.

It can be thought of as two separate OneWayTunnels working, but since the two tunnels share the same mutex, the data transfer in both directions always alternates.
In fact, clientVC and serverVC are in the same ET_NET thread, they are always called back, so the two OneWayTunnel will not be called back at the same time.

The key to two-way communication is that when a VC receives an EOS or completes a given number of transmitted bytes and needs to be closed, it must consider both tunnels simultaneously:

- When clientVC receives EOS as the source VC in tunnel c_to_s, you need to check whether serverVC has sent all the contents of MIOBuffer.
  - If there is still data to send, activate serverVC to complete the sending of the remaining data.
- When clientVC receives ERROR as the target VC in Tunnel s_to_c, it needs to notify Tunnel c_to_s that the current tunnel is about to close.
  - After the tunnel c_to_s receives the notification, it will close the readVIO of clientVC and the writeVIO of serverVC, and release the tunnel, but the two VCs will not be closed.
  - Then Tunnel s_to_c will close clientVC's writeVIO and serverVC readVIO, close both VCs, and release the tunnel.

As you can see, TwoWayTunnel is closed based on the target VIO completion.

- Any OneWayTunnel receives a WRITE_COMPLETE event and closes another OneWayTunnel
  - Then close both VCs and finally close yourself
- However, when EOS is received, the VIO of the target VC is set to complete the transmission of data in the remaining MIOBuffer.
  - Then wait for the target VC to complete the data transmission, OneWayTunnel receives the WRITE_COMPLETE event

In TwoWayTunnel, one direction is turned off and the other direction is turned off, even if another OneWayTunnel is still communicating normally.
For example, a half-shutdown via shutdown will cause a OneWayTunnel to be closed, but the OneWayTunnel in the other direction will still be able to communicate and will be forced to close.

## Members

Members are defined in OneWayTunnel:

- tunnel_peer
  - Point to the associated OneWayTunnel
- free_vcs
  - The default is true, which means that VC is turned off after VIO is turned off.
  - However, when OneWayTunnel receives the close notification of the peer, it will not continue to close the VC after closing VIO, but the party that initiated the notification completes the VC shutdown.

## Method

In the startEvent, the notification from the peer OneWayTunnel is determined, and after the WRITE_COMPLETE, the notification is sent to the peer OneWayTunnel:

```
  switch (event) {
  case ONE_WAY_TUNNEL_EVENT_PEER_CLOSE:
    /* This event is sent out by our peer */
    ink_assert(tunnel_peer);
    tunnel_peer = NULL;
    free_vcs    = false;
    goto Ldone;

...

  Ldone:
  case VC_EVENT_WRITE_COMPLETE:
    if (tunnel_peer) {
      // inform the peer:
      tunnel_peer->startEvent(ONE_WAY_TUNNEL_EVENT_PEER_CLOSE, data);
    }
    close_source_vio(result);
    close_target_vio(result);
    connection_closed(result);
    break;
```


After closing VIO, according to the value of free_vcs, determine whether to continue to close VC
```
void
OneWayTunnel::close_source_vio(int result)
{
  if (vioSource) {
    if (last_connection() || !single_buffer) {
      free_MIOBuffer(vioSource->buffer.writer());
      vioSource->buffer.clear();
    }
    if (close_source && free_vcs) {
      vioSource->vc_server->do_io_close(result ? lerrno : -1);
    }
    vioSource = NULL;
    n_connections--;
  }
}

void
OneWayTunnel::close_target_vio(int result, VIO *vio)
{ 
  (Void) pattern;
  if (vioTarget) {
    if (last_connection() || !single_buffer) {
      free_MIOBuffer(vioTarget->buffer.writer());
      vioTarget->buffer.clear();
    } 
    if (close_target && free_vcs) {
      vioTarget->vc_server->do_io_close(result ? lerrno : -1);
    }
    vioTarget = NULL;
    n_connections--;
  }
}
```

Complete the association of two OneWayTunnels via SetupTwoWayTunnel:

```
void
OneWayTunnel::SetupTwoWayTunnel(OneWayTunnel *east, OneWayTunnel *west)
{
  // make sure the both use the same mutex
  ink_assert(east->mutex == west->mutex);

  east->tunnel_peer = west;
  west->tunnel_peer = east;
}
```
## state machine design pattern

We see that TwoWayTunnel is made up of two OneWayTunnels that share a mutex and are designed with events specifically for collaboration.

Each OneWayTunnel manages two VCs, but for any one of them:

- OneWayTunnel is either responsible for receiving data or for sending data
- but will not both receive and send data

This means that OneWayTunnel only manages half of the functionality of these two VCs.

In TwoWayTunnel:

- When a VC's data is received and managed by a OneWayTunnel,
- This VC's data transmission is managed by another OneWayTunnel

TwoWayTunnel is designed to fully manage two VCs, but TwoWayTunnel is not a state machine. It implements two OneWayTunnels through a notification mechanism between two OneWayTunnels and sharing the same mutex in two OneWayTunnels. Collaboration.

But we also see a problem in TwoWayTunnel from the implementation of the code, that is, does not support "TCP half-close", when one VC initiates shutdown, then TwoWayTunnel will close both VCs at the same time, so we see the Tunnel The state machine can do very simple things. It is completely no problem to use it to implement pure TCP proxy and its auxiliary functions. However, if you need to implement advanced protocol processing based on TCP communication, the Tunnel state machine will be difficult to implement.

Therefore, TCP proxy and stream processing are the strengths of the Tunnel state machine.

At the same time, we see that the dual MIOBuffer function has not been implemented in the code, we can do it through the dual MIOBuffer

- Implement modification of the data flow in the tunnel
- Implement flow control and other functions in the tunnel

But when we need to implement a complex state machine design, we will let a state machine manage the data reception and transmission of a VC at the same time, for example:

- Receive a request and find that the request is illegal, we can send the error message directly
- But when the request is legal, the data flow can be completed through the Tunnel state machine.

At this point, we need to establish a state machine that can interact with ClientVC, and then transfer the control of the VC to the Tunnel state machine. After the tunnel state machine is processed, it will return to the interactive state machine.

This nested design is expressed in the ATS using a master-slave relationship:

- The state machine that interacts with ClientVC is called the "master state machine" (Master SM)
- The state machine that interacts with ServerVC is called the "slave state machine" (Slave SM)
- The Tunnel state machine that connects ClientVC and ServerVC is also called "slave SM" (Slave SM)

A Master SM can associate multiple Slave SMs when a specific subtask needs to be performed:

- the master state machine creates a slave state machine and calls the constructor from the state machine to hand over control of the VC to the slave state machine,
- When the task is completed from the state machine, the primary state machine is called back, the master state machine regains control of the VC, and the slave state machine is destroyed.

When we introduce the HttpSM state machine later, we will see the design of the Tunnel state machine. The design of the nested small state machine in this large state machine is common in ATS.

## References
- [I_OneWayTunnel.h](http://github.com/apache/trafficserver/tree/master/iocore/utils/I_OneWayTunnel.h)
