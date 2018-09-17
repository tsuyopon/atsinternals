# How to design a iocore sub-system

A subsystem consists of at least three parts:

- Trades
- Processor
- Task

## Trades

Handler is a state machine that inherits from class Continuation, is periodically called back by EventSystem or is called back at any time. It contains at least:

- Save the queue of the Task object
    - an atomic queue and a local queue
    - or a protection queue
- int Handler::dispatch(Task \*p)
    - Put the Task into the Handler system, and execute the corresponding task by the Handler.
    - Usually put the Task directly into the atomic queue and set Task::h to point to the Handler
    - Returns: success is 0, and -errno on failure
- int Handler::dispatch_local(Task *p)
    - Close to the function of dispatch, the difference is to put the Task directly into the local queue
    - Returns: success is 0, and -errno on failure
- void Handler::release(Task \*p)
    - Remove the Task from the Handler
    - including removing the Task from the Handler's queue, etc.
- void Handler::process_task(Task \*p)
    - Perform task tasks for Task
    - depending on the result of the execution, or the value of a specific member within the Task
        - Do you need to call Handler::free_task() 
        - Do you need to put the Task back into the queue?
        - or do nothing
- int Handler::mainEvent(int e, void \*p)
    - Periodically callback by the event system, traversing the Task queue
    - Import the atomic queue into the local queue
    - Traverse the local queue and take out the Task
    - Handling tasks within each Task
        - If task\_done in Task is true, you need to reclaim the resources occupied by the Task object via Handler::free\_task()
        - Otherwise, call Handler::precess\_task() to complete the Task task
    - Return to EventSystem after processing all or a certain number of Tasks
- void Handler::free\_task()
    - Unlink the Task and Handler (call Handler::release(Task \*p))
    - Finally call Task::free(t) to reclaim the resources occupied by the Task object

Inside the Handler, you also need to define some processing functions that help to complete the tasks described by the Task object. The member variables carried by the Task are used as input. Through these processing functions, the output of the result is realized, thus completing the task.


## Processor

The Processor provides a set of APIs to users who use the subsystem, including at least the following features:

- Task \* xxxProcessor::allocate\_task(t)
    - used to create a Task object and initialize
    - Or create a Task object via the new Task
    - Returns: Task object
- Task \* xxxProcessor::dispatch(Continuation \* callbackSM, ...)
    - Create a Task object, initialize the Task::action object with the incoming callbackSM, initialize the Task with other parameters passed in
    - Find a suitable Handler and try to lock the Handler
    - If the lock fails, put the Task into the Handler's atomic queue (call Handler::dispatch(task))
    - If the lock is successful, it is placed directly into the Handler's local queue (call Handler::dispatch\_local(task))
    - Returns a pointer to the Task object
    - Usually for situations where there is no pre-Handler processing

## Task

Task inherits from Continuation and is the main body of the task, including at least:

- int task\_done;
    - indicates whether the task of the Task object has been completed
    - When the value is 1 or true, the Handler will usually recycle the Task object
- int enabled;
    - indicates whether the task of the Task object is allowed to execute
    - If the value is 1, the Handler will execute the task it describes. If it is 0, the Handler will not execute.
- Continuation \ * account;
    - When the task is completed, it will call back cont and pass the event type and related data in the way that Task supports
- Ptr\<ProxyMutex\> cont\_mutex;
    - Increase the reference count of cont->mutex by 1 by ```cont_mutex = cont->mutex;```, ensuring that cont->mutex will not be recycled with cont
    - Lock this cont\_mutex before calling back cont
    - Must be able to read true and reliable data from task\_done and cont after locking
- EThread \*thread;
    - a pointer to an EThread object
    - Initialized when the Task object is allocated, indicating that this Task object is bound to the EThread
    - When you need to put this Task into the queue of the Handler, you need to select the Handler object running in the EThread
- Trades \ * h;
    - a pointer to a Handler object
    - When the pointer is not empty, only the Handler can release the Task object
    - When the pointer is empty, you can call Task::free(t) to release the Task object.
- void Task::init(Continuation \*c, ...) or through the constructor
    - Initialize the Task object
- void Task::free(EThread \*t)
    - used to reclaim resources occupied by Task objects
- void Task::clear()
    - optional
    - used to release and clean up the member variables in the Task
    - Mainly used to support freelist
    - Usually also called by init()
- void Task::dispatch(Continuation \*c, …)
    - optional
    - As long as the Task is in the Handler process, it can be called directly without locking the Task.
    - Reset the necessary Task members with other parameters passed in
    - Put the current Task into the atomic queue of the Handler object pointed to by Task::h
- void Task::reDispatch()
    - optional
    - As long as the Task is in the Handler process, it can be called directly without locking the Task.
    - Sometimes the Task can be executed repeatedly, and this method is used to implement multiple executions.
    - Reload the Task object into the Handler's queue and return
- void Task::pause()
    - optional
    - Sometimes the Task is executed periodically. This method temporarily stops the execution of the Task and sets Task::enabled = 0;
- void Task::resume()
    - optional
    - used to restore the Task being pause(), setting Task::enabled = 1;
- void Task::finished(int err)
    - optional
    - As long as the Task is in the Handler process, it can be called directly without locking the Task.
    - When the Task is repeatedly executed, mark this Task has been executed, and then the Task object will be recycled by the Handler

Special attention needs to be paid to the following differences:

- Handler::dispatch(Task *p) 
    - Is a Task managed by the Handler, or bound to the Handler
    - Also put the Task in the queue of the Handler
    - usually only need to be called once
- Task::dispatch(Continuation \*c, …) 
    - At this point Task has been bound to Tash::h this Handler
    - This action simply puts this Task into the Handler's queue
    - Repeatable call
- Task::reDispatch()
    - Usually no parameters are passed in, the task content will not be modified
    - for repeating Task tasks


## Task's life cycle

From creation to recycling, the Task object can be divided into the following three processes in order to complete the given task:

- pre-Handler processing
- Handler handler / in-Handler handler
- post-Handler processing

Before a Task is delivered to a Handler or returned from a Handler, some additional processing steps may be required to complete the entire task, which may or may not exist for different sub-systems.

For pre-Handler processing, you need to:

- Define the corresponding API function in the Processor, and set the corresponding state function and processing function in Task.
    - The handler will be called by the API function and the state function.
    - Usually these three functions should be defined together.
- Lock the Task before calling the state function.
- Lock the Handler before calling the handler.
- At any time, after the Handler is locked, you can make the Task into the Handler process by calling Handler::dispatch(Task *p).
- When entering the Handler process from the pre-Handler process, it needs to be known to the user of the sub-system in a callback manner.

In the process of pre-Handler execution, various errors may be encountered, so that the Handler process is not performed. Therefore, in order for the user of the sub-system to know clearly whether a Task has entered the Handler process, it is necessary to:

- In the pre-Handler procedure, if you encounter an error and cannot enter the Handler process,
    - Callback ERROR event and error message to the user
    - Then recycle the Task
- At any time, after calling Handler::dispatch(Task *p) successfully,
    - Call the user back to the SUCCESS event and the Task itself
    - After entering the Handler process, the Handler is responsible for reclaiming the Task after the task is completed, unless proceeding to the post-Handler process


The API corresponding to the pre-Handler() function can be provided in the Processor and called by the user of the sub-system:

- Action \* Processor::doTask(Continuation \*callbackSM, …)
    - optional
    - Create a Task object, initialize the Task::action object with the incoming callbackSM, initialize the Task with other parameters passed in
    - Pick a Handler and lock it
    - If the lock is successful, call Task::preHandler() directly, then callback callbackSM, return: ACTION\_RESULT\_DONE
    - If the lock fails, set Task::preEvent(int e, void \*data) corresponding to Task::preHandler() as the state handler, and complete the pre-Handler flow through the event system callback Task.
    - Returns: the action member of the Task

In ATS, callbacks are divided into synchronous and asynchronous. Referring to the general way inside ATS, the Action object or ACTION\_RESULE\_DONE is returned to the caller, so you need to add an Action object to the Task:

- Action Task::action;
    - an Action object, note that it is not a pointer
    - the state machine pointed to by the callback Action object when the pre-Handle task completes
    - If the Action object is canceled, the state machine is not called back and Task::free(t) is called to recycle

For the Handler / in-Handler process,

- In addition to Task::finished(), Task::dispatch(), Task::reDispatch(), the operation of the Task needs to lock the Handler where the Task is located.
- When the user of the sub-system receives the notification from the Handler, he can choose:
    - Let Task repeat it with Task::reDispatch()
    - Pause Task execution with Task::pause(),
    - Set the Task to completion by Task::finished(int err) and let the Handler complete the recovery of the Task.
    - Let Task go from the Handler process to the post-Handler process via Handler::release(Task *p).
- During the Handler processing, you can call Task::finished(int err) to terminate the execution of the Task and let the Handler reclaim the Task object.

For post-Handler processing,

- Initiated by the user of the sub-system. When the Handler calls back the user, the user separates the Task from the Handler by Handler::release(Task *p), and the Task enters the post-Handler procedure.
- The user will then attempt to lock the Task, then call Task::postHandler or set Task::postEvent(int e, void \*data) to the state function to reschedule the Task through the event system.
- Finally, Task::free(t) is always called by the user of the sub-system to reclaim the Task.

## shared Handler processing

When the Handler processing of the two Tasks is the same, but the pre-Handler and post-Handler processes are different, you can merge the two Tasks.

- A Task will have
   - pre-red-Handler(), pre-blue-Handler(), etc.
   - redEvent(int e, void \*data), blueEvent(int e, void \*data) and other state functions
   - Multiple API functions such as Processor::doRead(), Processor::doBlue()
- depending on the information passed in when the Task is initialized, one of the multiple state functions is selected as the starting state function.

This allows a Task to have different pre-Handler processes, but share the same Hander process.



