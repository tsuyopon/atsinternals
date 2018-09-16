# Basic component: PollCont

Although the abstraction of Polling Sub-System is not so clear, it still contains the basic three elements:

|  EventSystem   |    epoll   |  Polling SubSystem  |
|:--------------:|:----------:|:-------------------:|
|      Event     |  socket fd |       EventIO       |
|     EThread    | epoll_wait |      PollCont       |
| EventProcessor |  epoll_ctl |       EventIO       |

PollCont, as a state machine in the Polling Sub-System, implements event fetching of socket fd by periodically calling epoll_wait():

- PollDescriptor
  - Can be understood as a queue containing socket fd
  - Members can be obtained from this queue via epoll_wait
- EventIO(1)
  - Can be seen as a wrapper around read/write requests on socket fd
- PollCont
  - Get members (data) from the "queue" of PollDescriptor
  - The callback is associated with the epoll fd state machine (NetHandler) and passes the obtained member (data)
  - Probably for the convenience of code implementation, NetHandler is associated with PollCont
- EventIO(2)
  - Can be seen as a PollProcessor
  - Responsible for managing members in the PollDescriptor Queue
  - Through several methods, the addition, deletion and modification of the "queue" are realized.

You can see that EventIO also assumes the functionality of the Processor. This is because EventIO always operates the current thread's PollDescriptor when it is a Processor, which is different from EventProcessor in the entire thread pool. Therefore, here is the Processor function of EventIO:

  - start() / stop()
  - modify() / refresh()
  - close()

Merge into EventIO, which is a bit like the Event class itself provides the same method for schedule_*().

Since epoll / kqueue is O(1) complexity, it is very efficient, so:

- We don't have to set up multiple epoll fd in one thread.
- Only one PollDescriptor object needs to be configured per thread.

PollCont gets the result set by calling epoll_wait:

- Usually should traverse the result set
- Call back its state machine for each result (EventIO)
- The function of the state machine is to put the NetVC contained in EventIO into the queue of NetHandler.
- Wait until the NetHandler is driven by EventSystem, then traverse its own queue

As you can see, the process of putting EventIO into the NetHandler queue one by one can be done in batches:

- ATS is designed to let NetHandler access the result set directly
- This eliminates the data replication process from the result set to the NetHandler queue.

Therefore, each thread that processes the network task only needs one PollDescriptor and its corresponding PollCont and NetHandler:

- ATS puts the result set of PollCont in PollDescriptor
- Direct access to the result set in PollDescriptor by NetHandler
- Thereby eliminating the process of passing the result set through the callback

Although PollCont does not see the callback of NetHandler from the code, NetHandler is indeed the upper state machine of PollCont, but the definition of Polling Sub-System in the system is relatively vague. In the implementation part of UDP, Polling Sub-System defines Relatively clear.

For ease of analysis, only the code related to the epoll ET mode is listed below.

## definition

```
source: iocore/net/P_UnixNet.h
struct PollCont : public Continuation {
  // 指向此PollCont的上层状态机NetHandler
  // 可以理解为 Event 中的 Continuation 成员
  //     由于一个 PollCont 中的所有 EventIO 的上层状态机都是同一个 NetHandler
  //     因此，把这个 NetHandler 的对象直接放在了 PollCont 中
  NetHandler *net_handler;
  // Poll描述符封装，主要描述符，用来保存 epoll fd 等
  PollDescriptor *pollDescriptor;
  // Poll描述符分装，但是这个主要用于UDP在EAGAIN错误时的子状态机 UDPReadContinuation
  // 在这个状态机中只使用了pfd[]和nfds
  PollDescriptor *nextPollDescriptor;

  // 由构造函数初始化，在pollEvent()方法内使用
  // 但是如果net_handler不为空，则会在每次调用pollEvent时：
  //    按一定规则重置为 0 或者 net_config_poll_timeout
  int poll_timeout;

  // 构造函数
  // 第一种构造函数用于UDPPollCont
  // 此时成员net_handler==NULL，那么pollEvent就不会重置poll_timeout
  // 在ATS所有代码中，pt参数没有显示传入过，因此总是使用默认值
  PollCont(ProxyMutex *m, int pt = net_config_poll_timeout);
  // 第二种构造函数用于PollCont
  // 此时成员net_handler被初始化为nh，
  //     但是在PollCont的使用中，并未引用过PollCont内的net_handler成员
  //     从pollEvent()的设计来看，原本应该是在NetHandler::mainNetEvent中调用pollEvent()
  //     但是抽象设计好像不太成功，在NetHandler::mainNetEvent重新实现了一遍pollEvent()的逻辑
  // 在ATS所有代码中，pt参数没有显示传入过，因此总是使用默认值
  PollCont(ProxyMutex *m, NetHandler *nh, int pt = net_config_poll_timeout);

  // 析构函数
  ~PollCont();

  // 事件处理函数
  int pollEvent(int event, Event *e);
};
```

## use

The ATS design treats the polling operation as part of the event core system and treats the processing of the fd collection returned by the polling operation as part of the network core system.

- PollCont's mutex points to ethread->mutex
- NetHandler's mutex is new_ProxyMutex()

So when NetHandler is driven by EventSystem, you can safely access PollCont and PollDescriptor.

In the implementation of the traditional I/O multiplexing system, the fd collection is traversed and processed immediately after the polling operation is completed.

- Polling operation using PollCont in ATS
- Then use NetHandler to complete the traversal and processing of the fd result set.

These two state machines are called back in each round of EThread::execute(), which ensures that every polling result can be handled by NetHandler:

- Currently only the UDP part uses this logic, splitting the polling operation (PollCont) and the processing of the fd result set (UDPNetHandler) into two parts.
- The NetHandler used in the TCP part directly calls epoll_wait internally to implement the polling operation.
  - The code logic is still the same as PollCont, and it feels like to embed PollCont in NetHandler.
  - Did not use PollCont::pollEvent to complete the polling operation
  - But still access the epoll fd and fd result sets through the member PollDescriptor of PollCont

The following understanding and understanding of PollCont comes from the implementation part of UDP (this part of the official plan is cut, but let's take a brief look, in fact, this design concept is very good):

- During UDP initialization
  - PollCont and UDPNetHandler are thrown into the implicit queue of EventSystem, the priority is: -9
  - So every round of loops is UDPNetHandler::mainNetEvent is executed first, then PollCont::pollEvent
  - The result is that the fd collection returned by pollEvent will not be processed until the next iteration.
  - But the time of each round is very short, so there is basically no problem.
- UDPNetHandler::mainNetEvent() is only responsible for traversing the array of PollDescriptor->ePoll_Triggered_Events[]
  - Determine the vc type and determine the type of event
  - Determine if the corresponding vio is activated
  - Callback vio associated state machine
- PollCont::pollEvent() is only responsible for calling epoll_wait
  - Put the fd result set from epoll_wait into the PollDescriptor->ePoll_Triggered_Events[] array
  - Update PollDescriptor->result to the return value of epoll_wait

(You can look back when you look at the UDP part)

## Reference material
- [P_UnixNet.h]
(http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
