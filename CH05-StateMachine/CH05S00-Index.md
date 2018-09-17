# state machine

The state machine introduced in this chapter is actually a state machine that connects ClientVC and ServerVC.

The "boring machine" and "ClientSession" introduced earlier are actually a state machine that can handle some events, but these state machines can only handle events from a VConnection.

This chapter will begin with a description of the state machine used to implement the proxy function, which will:

  - Connect multiple VConnections
  - Receive events from these VConnections
  - Pass data between these VConnections

For data communication between two VConnections, it can be divided into:

  - One way (OneWayTunnel)
    - read data from one VC and then send data through another VC
  - Two-way (TwoWayTunnel)
    - read data from one VC and then send data through another VC
    - simultaneous data transfer in the opposite direction
    - Equivalent to combining two one-way

For data communication between multiple VConnections, ATS can implement:

  - One-way multi-out (OneWayMultiTunnel)
    - Read data from one VC and then send data to multiple VCs simultaneously
    - And can take into account the speed of data consumption of multiple target VCs
 
For the simple implementation of data transfer, the three state machines described above are sufficient, but sometimes we have to do some more detailed design to meet business needs, such as:

  - After the received data is verified, it is sent to the target VC.
    - If the verification fails, the source VC data is also notified that there is an error and needs to be resent
    - If the verification is successful, please inform the source VC data is being processed, please wait
  - Split the data returned by the target VC, as multiple results may be returned
    - Send the split results to the corresponding source VC

At this time, it is not simple to implement data transfer only, so the architecture of a state machine that can meet the business needs should be like this:

```
                +----------+------------------------------+------- ---+
                | | | |
--------------> | +===>>=== OneWayTunnel ===>>===+ | -------------- >
                | | | |
                ClientVC | StateMachine | ServerVC |
                | | | |
<-------------- | +===<<=== OneWayTunnel ===<<===+ | <------------- -
                | | | |
                +----------+------------------------------+------- ---+
```

The state machine that implements the business function controls the OneWayTunnel:

  - The state machine is responsible for managing ClientVC and ServerVC when it needs to parse the business.
  - When the data needs to be transferred directly, the state machine hands over the management to OneWayTunnel, and when the data transfer is completed, it is returned to the state machine.
  - Tunnel is like an assistant to the state machine, it can do some simple and easy work.

So, how does the state machine receive and process events from both VCs simultaneously?

  - OneWayTunnel
    - ClientVC shares the same IOBuffer with ServerVC
    - So whoever gets the lock of the OneWayTunnel state machine, who can operate this IOBuffer
    - ClientVC is responsible for writing data to IOBuffer, ServerVC always consumes data from IOBuffer
    - If any party can't get the lock, wait for the next time.
    - If the IOBuffer is full, notify ServerVC to hurry to spend
    - If the IOBuffer is read empty, notify ClientVC to hurry to produce
    - ClientVC received the EOS event and still needs to write the remaining data in IOBuffer to ServerVC.
    - ServerVC received the EOS event and the entire tunnel was terminated.
  - TwoWayTunnel
    - Combine two OneWayTunnel
    - If one of the OneWayTunnels is terminated, then another OneWayTunnel should be notified to terminate.
  - OneWayMultiTunnel
    - Basically the same as OneWayTunnel
    - When TargetVC receives an EOS event, only one TargetVC will be closed.
    - Only the entire TargetVC is closed and the entire tunnel will be terminated
    - When ClientVC receives an EOS event, it still needs to write the remaining data in the IOBuffer to all surviving TargetVCs.

It can be seen that for the Tunnel, since all VCs call back the same state machine and share the same IOBuffer in the state machine, there is no problem in the whole process, even if ClientVC and ServerVC are not in the same thread.

For the state machine that implements complex services, the most important point is how to safely operate ServerVC when receiving callbacks from ClientVC.

  - Calling server_vc->reenable() is safe
  - Any other operations need to lock server_vc->mutex

Similarly, when receiving a callback from ServerVC, the operation of ClientVC also follows the above principles.
