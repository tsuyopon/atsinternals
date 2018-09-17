# Understand the design of NetHandler

Make an analysis of NetHander's design according to Sub-system's design specifications.

First compare the three basic elements:

- NetHandler(Handler)
- NetProcessor(Processor)
- NetVC (Task)

Then there are three processing flows:

- pre-Handler has two
    - Initiate a TCP connection
        - state function startEvent
        - Processing function connectUp
        - API function NetProcessor::connect\_re
    - accept TCP connection
        - state function acceptEvent
        - Processing function is not
        - API function NetProcessor::accept
- Handler procedure
	- only read and write tasks
	- Does not include: accept a new connection or initiate a new connection
	- NetHandler::mainNetEvent
- post-Handler
   - none


## NetHandler

NetHandler is a Sub-system that handles NetVC read and write tasks, so the following listings are only relevant for read and write tasks and may contain a bit of connection-related content.

NetHandler inherits from class Continuation and periodically calls back through Negative Event. It contains:

- Save NetVC queues
    - Atomic queue
        - Polling FDs
        - read\_enabled\_list
        - write\_enabled\_list
    - Local queue
        - Polling result
        - read\_ready\_list
        - write\_ready\_list
    - Polling System is a Sub-system nested inside NetHandler, which also has three basic elements of the standard.
- int NetHandler::startIO(NetVC \*p)
    - Put NetVC into NetHandler, which is responsible for performing NetVC read and write tasks.
- void NetHandler::stopIO(NetVC \*p)
    - Take NetVC out of NetHandler and unlink all connections between NetVC and NetHandler
- void NetHandler::process\_enabled\_list()
    - Traverse the atomic queue read\_enabled\_list and write\_enabled\_list respectively
    - Put NetVC, which needs to perform read and write tasks, into the corresponding local queues read\_ready\_list and write\_ready\_list
- void NetHandler::process\_ready\_list()
    - Batch processing of NetVC read and write tasks in the local queue
    - If NetVC's status is already off, it is reclaimed via NetHandler::free\_netvc(netvc)
- int NetHandler::mainNetEvent(int e, void \*data)
    - The main callback function, in the running cycle of each NetHandler, passes:
        - PollCont::pollEvent gets the Polling result. After converting to NetVC, import the corresponding local queue according to the event type.
        - NetHandler::process\_enabled\_list() imports NetVC from two atomic queues into the corresponding local queue
        - NetHandler::process\_ready\_list() Batch processing NetVC read and write tasks
- void NetHandler::free_netvc(NetVC \*p)
    - Implemented by close_UnixNetVConnection().
    - First unlink NetVC from InactivityCop.
    - Then unlink NetVC from NetHandler via NetHandler::stopIO(NetVC *p)
    - Finally, the recycling of NetVC is completed by NetVC::free(t).
    - InactivityCop is a subsystem responsible for NetVC timeouts.

## NetProcessor

NetProcessor provides a subset of APIs for users of Net Sub-system:

- NetVC * NetProcessor::allocate_vc(t)
    - used to create a NetVC object and initialize it
    - Returns: NetVC object

After the new NetVC is created, after NetVC is initialized by the accept or connect state machine, NetVC is handed over to NetHandler via NetHandler::startIO(netvc) to perform read and write tasks.

The API provided by NetProcessor to initiate a TCP connection:

- Action * NetProcessor::connect_re(Continuation *cont, ...)
    - Create a NetVC object, initialize the NetVC::action object with the incoming cont, initialize NetVC with other parameters passed in, such as NetVCOptions, etc.
    - Try to lock the NetHandler
    - If the lock is successful, directly call NetVC::connectUp(t) to complete the connected task, then callback cont
        - Return: ACTION\_RESULT\_DONE
    - If the lock fails, put NetVC into EThread with NetHandler running, try to call back NetVC::startEvent() later, and then call NetVC::connectUp(t) after locking NetHandler again.
        - Returns: the action member of NetVC

API provided by NetProcessor to accept TCP connections:

- Action * NetProcessor::accept(Continuation *cont, ...)
    - Create a NetAccept object, execute the accept() system call by the NetAccept state machine
    - Create NetVC after getting socket fd
        - Copy the NetAccept action object to the NetVC action object
        - Set NetVC::acceptEvent as the starting state function
    - Callback NetVC::acceptEvent through the event system, and then call back the success or failure event to the state machine pointed to by NetVC's action object
    - Return value: the action member of the NetAccept object


## NetVC

NetVC inherits from class VConnection, and VConnection inherits from class Continuation, which is the body of the execution task, and contains the following:

- Action action;
    - an Action object, note that it is not a pointer
    - the state machine pointed to by the callback Action object when the pre-Handle task completes
    - If the Action object is canceled, the state machine is not called back and NetVC::free(t) is called to recycle
- int closed;
    - Indicates that NetVC's read and write tasks have been completed and can be recycled by NetHandler
- int read.enabled and write.enabled
    - Indicates whether to perform read and write tasks
- read.vio.\_cont and write.vio.\_cont
    - Callback \_cont will be called when the read and write tasks are completed
- read.vio.mutex and write.vio.mutex
    - a copy of mutex obtained from the corresponding \_cont->mutex to protect data reading for enabled and vio.\_cont
- EThread *thread;
    - a pointer to an EThread object
    - Initialized when the NetVC object is allocated, indicating that this NetVC object is bound to the EThread
    - NetHandler objects obtained in acceptEvent, startEvent and connectUp are run in this EThread
- NetHandler \*nh;
    - a pointer to a Handler object
    - When the pointer is not empty, only NetHandler can release the NetVC object
    - When the pointer is empty, you can directly call NetVC::free(t) to release the NetVC object.
- void NetVC::free(EThread \*t)
    - Used to reclaim resources occupied by NetVC objects
- void NetVC::clear()
    - Used to release and clean up individual member variables within NetVC
    - Mainly used to support freelist
- VIO \* do\_io\_read or write(Continuation \*cont, int nbytes, ...)
    - Reset the necessary NetVC members with other parameters passed in
    - Put the current NetVC into the atomic queue of the NetHandler object pointed to by NetVC::nh
- void reenable(VIO \*vio)
    - Re-place the current NetVC object into the queue of the NetHandler
- void readDisable(NetHandler \*nh) and writeDisable(NetHandler \*nh)
    - Temporarily suspend reading and writing tasks
- void set_enabled(VIO \*vio)
    - Resume read and write tasks that were paused before
- void do\_io\_close(int alerrno)
    - NetVC read and write tasks have been completed and NetVC::closed is set to non-zero
    - Next, NetVC objects will be recycled by NetHandler

NetVC is special, it contains two tasks: read and write, and it is executed repeatedly.


## InactivityCop

InactivityCop is also a Handler, inheriting from class Continuation, periodically timing callback through Period Event.


The Polling System is a subsystem nested inside a NetHandler.

```
  +-----> EThread::execute() loop
  | |
  | V
  | +-------------------+---------------+
  | | InactivityCop::check_inactivity() |
  | | | |
  | | | |
  | +-------------------+---------------+
  | |
  | V
  | +-------------------+---------------+
  | | NetHandler::mainNetEvent() |
  | | | |
  | | | |
  | +-------------------+---------------+
  | |
  | V
  +-----<----------------------+

```


First compare the three basic elements:

- InactivityCop(Handler)
- Not clear (Processor)
- NetVC (Task)

The design of InactivityCop is so simplified that there is no separate processor in the design of the basic elements, and the Task is shared with NetHandler, which is also NetVC.

Then there are three processing flows:

- pre-Handler
    - none
- Handler
    - Timeout detection for NetVC
    - Check if the number of client connections exceeds the limit and triggers a timeout event
    - InactivityCop::check_inactivity
- post-Handler
    - none

### Timeout detection

In order to implement timeout detection, it contains:

- Save NetVC queues
    - external queue
        - NetHandler::open\_list
        - This is not an atomic queue.
    - internal queue
        - NetHandler::cop\_list
    - Both queues are not defined in InactivityCop, but in NetHandler. Since InactivityCop shares the same mutex with NetHandler, this is fine.
- Handler::startIO()
    - The definition of this method does not exist, but its code can actually be seen in NetVC::acceptEvent and NetVC::connectUp
    - ```nh->open_list.enqueue(netvc);```
    - Its function is to put netvc directly into the open_list queue.
- Handler::stopIO()
    - The definition of this method does not exist, but its code can actually be seen in close_UnixNetVConnection()
    - ```nh->open_list.remove(netvc); nh->cop_list.remove(netvc)```
    - Its function is to remove netvc directly from the two queues open\_list and cop\_list.
- InactivityCop::check\_inactivity(int e, void *data)
    - This method contains a traversal of the open\_list queue and the process of importing NetVC into cop\_list
    - Also includes the functionality of Handler::process\_task()
- Handler::free_task()
    - This method does not need to be detected during timeout
    - Timeout detection simply informs the user that the previously set time has been reached


Since no additional resources need to be allocated, you only need to put NetVC into the open_list to implement timeout management, so the Processor is not defined and the timeout is done using the control function on NetVC.

In order to integrate timeout control into NetVC, the following extensions were made to NetVC:


- ink_hrtime next\_inactivity\_timeout\_at
    - Indicates whether Inactivity Timeout detection is enabled
    - ink_hrtime inactivity\_timeout\_in to save the settings of the Inactivity Timeout
- ink_hrtime next\_activity\_timeout\_at
    - Indicates whether Activity Timeout detection is enabled
    - ink_hrtime active\_timeout\_in to save the settings of the Activity Timeout
- Action action\_;
    - Actions when the pre-Handler is reused are used to callback state machine when a timeout occurs
- netActivity()
    - Equivalent to Task::reDispatch(), automatically resets next\_inactivity\_timeout\_at after each read and write operation
- set\_inactivity\_timeout() and set\_active\_timeout()
    - Equivalent to Task::dispatch(), used to:
    - Set inactivity\_timeout\_in and next\_inactivity\_timeout\_at
    - Set active\_timeout\_in and next\_activity\_timeout\_at
- cancel\_inactivity\_timeout() and cancel\_active\_timeout()
    - Equivalent to Task::finised(), used to:
    - Clear inactivity\_timeout\_in and next\_inactivity\_timeout\_at
    - Clear active\_timeout\_in and next\_activity\_timeout\_at

### Client concurrent connection restrictions

In order to limit the client when there are too many concurrent connections to ensure that there are enough resources to initiate a connection to the source server, its internal design:

- Save NetVC queues
    - External queue 1
        - NetHandler::active_queue
    - HttpSM is placed in this queue when NetVC is in the following state
        - Waiting for an HTTP request from the client
        - Waiting for ATS to send back an HTTP response
    - Internal queue 1
        - None, traversing active_queue directly in NetHandler::manage_active_queue()
    - External queue 2
        - NetHandler::keep_alive_queue
    - HttpSM is placed in this queue when NetVC is in the following state
        - ATS has sent the HTTP response completely to the client
        - The connection between the client and the ATS supports Http Keep-alive and the connection is not closed by either party
    - Internal queue 2
        - None, traversing keep_alive_queue directly in NetHandler::manage_keep_alive_queue()
- NetHandler::add\_to\_active\_queue() and NetHandler::add\_to\_keep\_alive\_queue()
    - Equivalent to Handler::startIO
    - Its function is to put netvc directly into active\_queue or keep\_alive\_queue queue
- NetHandler::remove\_from\_active\_queue() and NetHandler::remove\_from\_keep\_alive\_queue()
    - Equivalent to Handler::stopIO()
    - Its function is to remove netvc directly from the active\_queue or keep\_alive\_queue queue
- NetHandler::manage_active_queue and NetHandler::manage_keep_alive_queue
    - Traverse the active\_queue or keep\_alive\_queue queue
    - Prioritize NetVC callback timeout events in keep_alive_queue
    - Then callback timeout event to NetVC in active\_queue
- NetHandler::_close_vc()
    - Equivalent to Handler::free\_task(Task *p)
    - Its function is mainly to callback timeout events to NetVC, but NetVC does not necessarily close, depending on the state machine's response to timeout events

The Processor is also not defined, and is encapsulated directly on NetVC for the convenience of state machine calls:

- NetVC::add\_to\_active\_queue() and NetVC::add\_to\_keep\_alive\_queue()
    - Equivalent to Task::dispatch()
    - Both directly call methods defined on NetHandler
- NetVC::remove\_from\_active\_queue() and NetVC::remove\_from\_keep\_alive\_queue()
    - Equivalent to Task::finished()
    - Both directly call methods defined on NetHandler

This allows the state machine to perform operations directly through API functions on NetVC without the need for API functions on the Processor.

## Polling System

The Polling System is a subsystem nested inside a NetHandler.

```
  +-----> EThread::execute() loop
  | |
  | V
  | +-------------------+---------------+
  | | InactivityCop::check_inactivity() |
  | | | |
  | | | |
  | +-------------------+---------------+
  | |
  | V
  | +-------------------+---------------+
  | | NetHandler::mainNetEvent() |
  | | | |
  | | V |
  | | +---------+-------------+ |
  | | | PollCont::pollEvent() | |
  | | +---------+-------------+ |
  | | | |
  | +-------------------+---------------+
  | |
  | V
  +-----<----------------------+

```

First compare the three basic elements:

- PollCont + sys poll(Handler)
- NetHandler(Processor)
- EventIO(Task)

Then there are three processing flows:

- pre-Handler
    - none
- Handler procedure
    - PollCont::pollEvent
- post-Handler
    - none

Since the Polling System is directly linked to the operating system, the Linux epoll will be used as an example for analysis.

### PollCont and Sys Poll

PollCont and sys poll together form a Handler:

- Save EventIO's queue
    - Atomic queue
        - epoll fd
        - Created by epoll\_create(), which adds elements to the atomic queue via epoll\_ctl(EPOLL\_ADD, ...)
    - Local queue
        - pollDescriptor->ePoll\_Triggered\_Events array
        - Imported by epoll\_wait on epoll fd after operation like ```popall()```
- Handler::dispatch()
    - PollCont does not define this method
    - Can be replaced with epoll\_ctl(EPOLL\_CTL\_ADD, ...)
- Handler::release()
    - PollCont does not define this method
    - Can be replaced with epoll_ctl(EPOLL\_CTL\_DEL, ...)
- PollCont::pollEvent(int e, void *data)
    - NetHandler::mainNetEvent() callback that is run periodically
    - Provide NetEvent with all EventIOs in the entire local queue
    - Do not handle these EventIO objects themselves
- Handler::free_task()
    - Since the result returned by epoll\_wait is stored in the pollDescriptor->ePoll_Triggered_Events array,
    - The next time you call epoll_wait, the last content is overwritten, so there is no need to release and clean up

### NetHandler as Processor

Since the Polling System is a subsystem nested inside the NetHandler, the NetHandler is the only legal access to this subsystem, so the NetHandler will act as the Processor of the Polling System, providing the following API functions for this:

- NetHandler::startIO(NetVC *p)
    - Equivalent to Processor::dispatch 
    - Put NetVC member EventIO objects into the Polling System
- NetHandler::stopIO(NetVC *p)
    - Equivalent to Handler::release(). Since PollCont shares the same mutex with NetHandler, in order to simplify the design, change PollCont::stopIO to NetHandler::stopIO
    - Remove NetVC member EventIO objects from the Polling System
- Processor::allocate_task(t)
    - none
    - Since EventIO is defined in most cases as a member of NetVC, no separate creation is required
    - Very few cases created by new EventIO
    - Therefore, the API function of allocate is not provided in the processor of the polling system.

### EventIO

EventIO provides the ability to adapt to multiple VConnection types. To simplify the analysis, we take NetVC as an example.

- EventLoop event_loop;
    - when nullptr indicates that the task has been completed
    - But EventIO does not have a separate recycling method, so the task is not immediately reclaimed when it is completed, but is a member of NetVC and is recycled with NetVC.
- get_PollCont(t)
    - Equivalent to Handler *h in the Task object;
    - There is only one NetHandler and one PollCont state machine in each EThread
- Continuation \*c
    - Pointer to a NetVC object that is used by NetHandler after converting EventIO to NetVC
- int type
    - The above c points to the NetVC object only if type == EVENTIO\_READWRITE\_VC
- int events
    - used to describe the tasks to be performed to the polling system
    - Set by bit, which can be described as: read task, write task, read and write compound task, etc.
- Ptr\<ProxyMutex\> cont_mutex;
    - none
    - Since it always exists as a member of NetVC, it is not required
- Initialization
    - completed by constructor
    - event_loop = nullptr; type = 0; data.c = nullptr;
- start(EventLoop l, int fd, Continuation \*c, int event)
    - Originally defined on PollCont, implement Handler::dispatch(Task *p)
    - Here to simplify the design, define it in EventIO
- modify(int event)
    - When event > 0, it is equivalent to Task::resume()
    - When event < 0, equivalent to Task::pause()
- refresh()
    - Equivalent to Task::reDispatch()
- stop()
    - Equivalent to Task::finished()
- close()
    - Originally defined on PollCont, implement Handler::free_task(Task *p)
    - But EventIO::free() is not defined, and the Event object is not released by PollCont.
    - Therefore, this special close() method is defined, which is not used when EventIO is a NetVC service.
