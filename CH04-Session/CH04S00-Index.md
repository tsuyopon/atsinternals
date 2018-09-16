# Overview

EventSystem and Net Sub-System are the infrastructure of ATS.
  - Ability to accept connections from Clients via NetAccept
  - Ability to initiate a connection to the server via NetProcessor
  - Ability to read and write data via NetHandler

So how do you support various communication protocols?

In TCP communication, it is necessary to first determine which protocol is used in TCP communication, and then select the state machine of the corresponding protocol to handle this TCP communication.

In the previous chapters, two important components were introduced:

  - SSLNextProtocolTrampoline
    - Obtain the type of communication protocol to be committed in the SSL channel through the NPN / ALPN protocol
  - ProtocolProbeTrampoline
    - After reading the first data content sent by the client and analyzing it, the type of the upcoming communication protocol is obtained.

Note: There are some protocols (for example: SMTP protocol), and the protocol type cannot be judged by ProtocolProbeTrampoline.

  - After the client connects to the server, the client does not send any information.
  - Instead, the server first sends the content to the client.

Through the above two components, you can know the type of protocol to be communicated next, and then follow the following process:

  - Pass netvc to the XXXSessionAccept class instance corresponding to the protocol (eg HttpSessionAccept)
  - The XXXSessionAccept class instance creates an instance of the XXXClientSession class (eg HttpClientSession)
  - The XXXClientSession class instance creates an instance of the XXXSM class (eg HttpSM) and passes netvc to XXXSM
  - The XXXSM class instance initiates a connection to OServer and creates a corresponding netvc
  - The XXXSM class instance creates an instance of the XXXServerSession class (eg HttpServerSession) and passes in the newly created netvc

The above various types of objects will share the mutex of the client side netvc:

  - XXXClientSession
  - XXXSM
  - Server end netvc
  - XXXServerSession

After completing a communication:

  - XXXSM will strip XXXServerSession (release / deattach)
    - According to whether OServer supports connection multiplexing, it is decided whether to close the netvc connected to OServer at the same time.
  - XXXSM then passes netvc back to XXXClientSession and then deletes itself
  - Then XXXClientSession will wait for the next request to come, or the connection is closed.

Next, a detailed introduction to XXXSessionAccept and XXXClientSession will be given.

  - The introduction to XXXServerSession will be left to the section of SessionPool and then detailed description.
  - For the introduction of XXXSM, there will be separate chapters for analysis.

