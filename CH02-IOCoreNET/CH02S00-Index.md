# Chapter 2 I/O Core: Network

According to the understanding of the event system, design a state machine, only need to receive event notifications from the event system:

  - EVENT_IMMEDIATE
  - EVENT_INTERVAL

So the state machine always receives "notifications" passively, but we know that for Sockets, there is no such notification mechanism on at least Linux systems.

The mechanism we have is "polling", which requires the application to initiate syscall to determine which Sockets have data to read and which Sockets can write data.

This requires designing a network subsystem (Net Sub-System) that converts the active "polling" into an event. This function is implemented by the NetHandler state machine.

At the same time, in a "polling"-based network subsystem, the entry of new connections is implemented by the NetAccept state machine, which can run in two modes:


  - Blocking mode
    - Call accept() in blocking mode in DEDICATED EThread to accept new connections
  - Non-blocking mode
    - Call accept() in non-blocking mode in REGULAR EThread to accept new connections

In addition, we also need to manage the timeout of the Socket connection in idle state. This function is implemented by the InactivityCop state machine.

In order to achieve the above functions, there are some necessary basic components and other components to cooperate, so the entire network subsystem (Net Sub-System) consists of the following parts:

  - Basic component
    - Polling subsystem
      - EventIO (encapsulation of the epoll/kqueue method, equivalent to PollingEvent and PollingProcessor)
      - PollDescriptor (used to abstract the handle of epoll/kqueue)
      - PollCont (state machine that periodically performs polling operations)
    - NetVConnection (used to connect Socket, buffer and state machine)
      - Inherited from the VConnection base class
    - IOBuffer (buffer for storing data)
    - VIO (used to describe Socket read and write operations)
    - Encapsulation of "Socket operation"
      - Connection
      - Server
  - Core components
    - NetAccept (implementation of state machine accepting new connections)
    - NetHandler (implementing a state machine from "polling" to event conversion)
    - InactivityCop (timeout control for implementing connections)
    - Throttle (limitation to achieve maximum connection)
    - UnixNetVConnection (NetVConnection implementation on Unix operating system)
      - Inherited from the NetVConnection base class
    - Implement protocol type detection
      - ProtocolProbeSessionAccept
      - ProtocolProbeTrampoline
  - External Interface
    - UnixNetProcessor
      - Inherited from NetProcessor base class
