# Right retention

For any forwarding and citation of the content of this project, please indicate the address of this project: http://github.com/oknet/atsinternals

# Apache Traffic Server Source Code Analysis

This project is mainly the analysis result of my source code of Apache Traffic Server 6.0 (hereinafter referred to as ATS).

It is my personal understanding of its internal implementation and architectural design, which is limited by personal ability and may be understood and analyzed.

Part of the content is directly translated from the comments in the ATS source code, may be somewhat fluent, can be read and understood against the source code.

Due to the large changes in ATS 6.0, if the source code you are using/referencing is not version 6.0, please read the source code changes in the official git repository to help understand.

# Contact author

If you don't understand the content, or think it is wrong, you can leave a message to me directly on github.

# Source code directory structure

The source code of ATS consists of multiple parts, each with a separate directory to distinguish:

  - iocore
    - Implement various underlying I/O interactions
    - Contains multiple subsystems
  - proxy

## IOCORE

There are multiple subsystems within IOCore, each subsystem is separated by a separate directory:

  - EventSystem
    - Provides a basic event system to implement the basic components of the Continuation programming model
  - Net
    - Provide basic network operations, TCP, UDP, OOB, Polling and other network I / O operations
  - DNS
    - Provide DNS resolution service
  - HostDB
    - Cache/query service that provides DNS resolution results 
  - AIO
    - Provide disk I / O operations 
  - Cache
    - Provide caching service 
  - Cluster
    - Provide cluster services 

In addition, the Utils directory is a public part, not a subsystem.

Within each subsystem, there are two types of header file prefixes:

  - I\_xxxx.h
    - Provide external interface definition, I means Interface
    - Can include I\_xxxx.h, but can't include P\_xxxx.h
  - P\_xxxx.h
    - the definition used inside the subsystem, P means Private
    - Can include I\_xxxx.h or include P\_xxxx.h

### EventSystem Subsystem

Mainly contains the definition of the Interface base type:

  - Continuation
  - Action
  - Thread
  - Processor
  - ProxyMutex
  - IOBufferData
  - IOBufferBlock
  - IOBufferReader
  - MIOBuffer
  - MIOBufferAccessor
  - VIO

Mainly contains the definition of the Interface extension type

  - VConnection <- Continuation
  - Event <- Action
  - EThread <- Thread
  - EventProcessor <- Processor

It can be seen that the EventSystem subsystem is basically a definition of the Interface type, because it is the basic component of the Continuation programming mode.

### Net Subsystem

Mainly contains the Interface definition:

  - NetVCOptions
  - NetVConnection <- VConnection
  - NetProcessor <- Processor
  - SessionAccept <- Continuation
  - UDPConnection <- Continuation
  - UDPPacket
  - UDPNetProcessor <- Processor

Mainly contains the basic definition of Private:

  - PollDescriptor
  - PollCont <- Continuation
  - Connection
  - Server <- Connection
  - NetState

TCP related:

  - UnixNetVConnection <- NetVConnection
  - NetHandler <- Continuation
  - NetAccept <- Continuation
  - UnixNetProcessor <- NetProcessor

SSL related:

  - SSLNetVConnection <- UnixNetVConnection
  - SSLNetAccept <- NetAccept
  - SSLNetProcessor <- UnixNetProcessor
  - SSLSessionID
  - SSLNextProtocolAccept <- SessionAccept
  - SSLNextProtocolTrampoline <- Continuation
  - SSLNextProtocolSet

Socks related:

  - SocksEntry <- Continuation

UDP related:

  - UDPIOEvent <- Event
  - UDPConnectionInternal <- UDPConnection
  - UnixUDPConnection <- UDPConnectionInternal
  - UDPPacketInternal <- UDPPacket
  - PacketQueue
  - UDPQueue
  - UDPNetHandler <- Continuation

It can be seen that the Net subsystem has begun to distinguish between the definition of the Interface and Private types. For the definition of the Private type, it should be used directly only within the Net subsystem.

Note: Since the ATS code is developed by many of the community's developer features, it may break the logic above when writing code, so don't worry too much when reading the code.

#### NetVConnection inheritance relationship

In the early days, this is very long. As the predecessor of ATS open source, Inktomi Cache Server supports Windows operating system. From the current ATS code, you can still see some commented out code. Some of the classes used start with NT. And you can find the corresponding class definition that starts with Unix.

So for the NetVConnection inheritance class UnixNetVConnection:

  - Can be boldly guessing that NTNetVConnection is also an inheritance class of NetVConnection
  - Then we see that the definition of NetVConnection is in I_NetVConnection.h, so NetVConnection is an externally provided Interface.
  - The definition of UnixNetVConnection appears in P_UnixNetVConnection.h, so UnixNetVConnection is only used within the Net subsystem.
  - Then look at all the header files containing Unix keywords, all starting with P\_

So we can draw the following conclusions:

  - NetVConnection is an interface provided by the Net subsystem to the outside. All methods that can be called externally need to be placed in a virtual method.
  - UnixNetVConnection and NTNetVConnection redefine the virtual method of the base class NetVConnection on different platforms, using different implementations
  - To achieve NetVConnection support for multiple operating system platforms
  - The external system does not need to care about the specific underlying implementation of NetVConnection is UnixNetVConnection or NTNetVConnection


Based on the understanding of the above inheritance relationship, then look at the SSLNetVConnection:

  - SSLNetVConnection is inherited from UnixNetVConnection
  - The definition of SSLNetVConnection does not contain Unix or NT keywords
  - *.h and *.cc file names that implement SSLNetVConnection also do not contain Unix or NT keywords
  - But all header file names that contain the SSL keyword start with P\_

You can boldly guess that the implementation of SSLNetVConnection is distinguished by macro definitions:

  - On Unix operating systems, SSLNetVConnection inherits from UnixNetVConnection
  - In WinNT operating system, SSLNetVConnection inherits from NTNetVConnection

But then the NT part of the code was cut off, so the inheritance relationship I see now is:

  - NetVConnection
    - UnixNetVConnection
      - SSLNetVConnection

However, in the implementation, there is no way to let the users of the Net subsystem know what type of NetVConnection is, so a large amount of code in the proxy directory directly refers to the dynamic conversion of UnixNetVConnection and SSLNetVConnection types to determine a NetVConnection. What type is it.

Since the implementation of the SSL layer is based on the use of the OpenSSL library, and when the actual business logic is implemented, some SSL attributes must be set. Therefore, SSLNetVConnection needs to provide some interfaces to the users of the Net subsystem, but these are SSLNetVConnection. The method is not defined in NetVConnection. This is probably because there is a UnixNetVConnection between SSLNetVConnection and NetVConnection. Some are inconvenient, but there are some ugliness.

Therefore, users of the Net subsystem need to dynamically convert the NetVConnection to the SSLNetVConnection type when using SSL-related functions, which breaks the good design of the original system.

Note: The above analysis is only personally understood because it contains a lot of guesses.

