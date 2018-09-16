# Basic component: SSLNextProtocolTrampoline


Trampoline is a "trampoline" in English, which is a very graphic representation of the function of this component.

When we introduced NetAccept in the previous section, we mentioned the relationship between Acceptor and NetVC and SM. We need the Acceptor to accept the new NetVC, then create the SM object and associate it with NetVC.

In the implementation of SSL, a step is added. After the Acceptor, there is a "trampoline", and the trampoline jumps to the corresponding SessionAcceptor according to the negotiation of NPN/ALPN.

A number of different SessionAcceptors are registered in the trampoline and managed via SSLNextProtocolSet.

The SessionAcceptor here is actually part of the upper state machine, and the SessionAcceptor is responsible for creating the corresponding state machine.

In the design of ProtocolTrampoline, you need to consider two types of clients that are supported and not supported by the NPN/ALPN protocol:

  - SSLNetVC established for clients that support the NPN/ALPN protocol
    - Confirm the type of the application layer protocol by parsing the NPN/ALPN protocol, and jump directly to the corresponding state machine.
  - SSLNetVC established for clients that do not support the NPN/ALPN protocol
    - Jump directly to ProbeSessionAccept,
    - Then determine the protocol type of the application layer through ProbeSessionTrampoline, and then jump to the corresponding state machine.

This design is like a ninja with different skills. When you can't reach the target directly, you can achieve the ultimate goal with the aid of the “boring machine” through the “Ninja Jump”:

  - Some can jump to the end point
  - Some need to use two jumps to reach the end

The design of the double trampoline is shown below:

```
                                                    ?       ?                        #     #     #     #
                                            ?                                #      #     #                   #
                                      ?                               #     #      #            #                  #
                                  ?                            #                         #            #               #
                              *                           #     ALPN                          #            #            #
                           *                           #    or                                  #         + #  +    +    #
                        *                           #  NPN                                        #  +   +   #         + #
                      *                          #  h                                             +#         +           +
                    *                          #  t                                             +  + Yeah!   + Yeah!     + Yeah!
                  *                          #  i                                             +    ========  ==========  ========
                * + +      Ninja           #  W                                             +      = SPDY =  = HTTP/2 =  = HTTP =
        SSLNetVC      +        Jump      #                                                +        ========  ==========  ========
 ==================     +               #                                               +             ==         ==         ==
 = ProtocolAccept =       +           #            (Without NPN/ALPN)                  +              ==         ==         ==
 ==================        +         #    #  #  #                         Ninja       +               ==         ==         ==
   ==          ==           +       #  #            # .. .. .. +  +  +        Jump   /                ==         ==         ==
   ==          ==            +     #             =================      +           /                 ==         ==         ==
   ==          ==             +   /              = SessionAccept =        +        /                  ==         ==         ==
   ==          ==          ====+ /=======        =================          +     /                   ==         ==         ==
   ==          ==          =    +       =          ==         ==             +   /                    ==         ==         ==
   ==          ==          =  Protocol  =          ==         ==          ====+ /=======              ==         ==         ==
   ==          ==          = Trampoline =          ==         ==          =    +       =              ==         ==         ==
   ==          ==          ==============          ==         ==          =  Session   =              ==         ==         ==
   ==          ==          =            =          ==         ==          = Trampoline =              ==         ==         ==
   ==          ==          =            =          ==         ==          ==============              ==         ==         ==
   ==          ==          =            =          ==         ==          =            =              ==         ==         ==
   ==          ==          =            =          ==         ==          =            =              ==         ==         ==
 ==================     ====================     =================     ====================     ====================================

```

## definition

```
// SSLNextProtocolTrampoline is the receiver of the I/O event generated when we perform a 0-length read on the new SSL
// connection. The 0-length read forces the SSL handshake, which allows us to bind an endpoint that is selected by the
// NPN extension. The Continuation that receives the read event *must* have a mutex, but we don't want to take a global
// lock across the handshake, so we make a trampoline to bounce the event from the SSL acceptor to the ultimate session
// acceptor.

// Direct translation of the source code comments as follows:
// When a read operation of length 0 bytes is executed on an SSL connection, EventSystem will call back SSLNextProtocolTrampoline to accept this I/O event.
// Note: The length of 0 in the 0-length read operation refers to the length of the decrypted data. For example: for the handshake process, it is 0 length.
// The focus of the 0-byte read operation is the SSL handshake process, which implements the end of the protocol selected by binding an NPN extension.
// The "continuation" of this read event callback must have a mutex lock, but we don't want to remain locked throughout the handshake process.
// So the trampoline was designed to bounce this event from the SSL Acceptor to the final Session Acceptor.

struct SSLNextProtocolTrampoline : public Continuation {
  // Constructor
  // Initialize mutex and npnParent members, and set the callback function to ioCompletionEvent
  explicit SSLNextProtocolTrampoline(const SSLNextProtocolAccept *npn, ProxyMutex *mutex) : Continuation(mutex), npnParent(npn)
  {
    SET_HANDLER(&SSLNextProtocolTrampoline::ioCompletionEvent);
  }

  // This state machine is created by the SSLNextProtocolAccept via the new method, and then popped into the subsequent SessionAccept by the "trampoline" method.
  // Therefore, after the SSLNetVC is bounced off, the object itself is released via the delete method.
  int
  ioCompletionEvent(int event, void *edata)
  {
    VIO *vio;
    Continuation *plugin;
    SSLNetVConnection *netvc;

    // Since it is an I/O event callback for a 0-length read operation, edata points to the read.vio member of SSLNetVC, which is a VIO type.
    vio = static_cast<VIO *>(edata);
    // Parse out the SSLNetVC contained in VIO
    netvc = dynamic_cast<SSLNetVConnection *>(vio->vc_server);
    // If it is not an SSLNetVConnection type, then it is asserted.
    ink_assert(netvc != NULL);

    // Since it is a 0-length read operation, only READ_COMPLETE is passed to indicate success, and READ_READY does not occur because READ_COMPLETE has a higher priority.
    // Other situations are always wrong
    switch (event) {
    case VC_EVENT_EOS:
    case VC_EVENT_ERROR:
    case VC_EVENT_ACTIVE_TIMEOUT:
    case VC_EVENT_INACTIVITY_TIMEOUT:
      // Cancel the read before we have a chance to delete the continuation
      // When the connection timeout occurs, log out the read operation, close SSLNetVC, and recycle the "trampoline" itself.
      netvc->do_io_read(NULL, 0, NULL);
      netvc->do_io(VIO::CLOSE);
      delete this;
      return EVENT_ERROR;
    case VC_EVENT_READ_COMPLETE:
      // Encounter the READ_COMPLETE event we need, jump out of the switch
      break;
    default:
      // I don’t think there should be any possibility of running here.
      // Is it necessary to get an assert here? Otherwise, the memory leaks.
      return EVENT_ERROR;
    }

    // Cancel the read before we have a chance to delete the continuation
    // Log out the read operation because the next step is to recycle the trampoline itself.
    netvc->do_io_read(NULL, 0, NULL);
    // Get the npnEndpoint member through the SSLNetVConnection endpoint method
    plugin = netvc->endpoint();
    if (plugin) {
      // priority "bounce" npnEndpoint state machine
      // plugin is the state machine determined according to the NPN/ALPN protocol. If it is NULL, it means:
      // The client does not support the NPN/ALPN protocol, or the protocol is not registered in the negotiation process.
      send_plugin_event(plugin, NET_EVENT_ACCEPT, netvc);
    } else if (npnParent->endpoint) {
      // If the npnEndpoint state machine does not exist, it "bounces" to the npnParent->endpoint state machine
      // npnParent is the incoming SSLNextProtocolAccept instance when the "boring machine" is initialized.
      // npnParent->endpoint For SSLNextProtocolAccept,
      // is pointed to the ProtocolProbeSessionAccept object in HttpProxyServerMain.cc.
      // In ProtocolProbeSessionAccept, the protocol type will be confirmed based on the data of the read application layer.
      // Route to the default endpoint
      send_plugin_event(npnParent->endpoint, NET_EVENT_ACCEPT, netvc);
    } else {
      // If it does not exist, close SSLNetVC directly.
      // No handler, what should we do? Best to just kill the VC while we can.
      netvc->do_io(VIO::CLOSE);
    }

    // After returning from the NET_EVENT_ACCEPT event handler, SSLNetVC's Read VIO and/or Write VIO will be reset and associated with the upper state machine.
    // So you can safely delete the "trampoline" itself and return
    delete this;
    return EVENT_CONT;
  }

  const SSLNextProtocolAccept *npnParent;
};

// Use the method of adjusting the state machine back and forth
// plugin is the state machine that will be called back
// event is a callback event
// edata is NetVC
static void
send_plugin_event(Continuation *plugin, int event, void *edata)
{
  if (plugin->mutex) {
    // With mutex, then callback after locking
    EThread *thread(this_ethread());
    MUTEX_TAKE_LOCK(plugin->mutex, thread);
    plugin->handleEvent(event, edata);
    MUTEX_UNTAKE_LOCK(plugin->mutex, thread);
  } else {
    // No mutex, direct callback
    plugin->handleEvent(event, edata);
  }
}
```

## Reference material

- [SSLNextProtocolAccept.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNextProtocolAccept.cc)
- [HttpProxyServerMain.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpProxyServerMain.cc)
