# Base component: PluginVC

PluginVC is a state machine that is masquerading as NetVC and is used to implement bidirectional data transfer between two "Continuations".

PluginVC is based on PluginVCCore, which contains two PluginVC members, one is active_vc and the other is passive_vc.

PluginVCCore is like a scaled-down IOCore Net Subsystem, which is only responsible for managing active_vc and passive_vc.

In the implementation of TSHttpConnectWithPluginId(),

- Establish a connection with HttpSM via PluginVCCore::connect(), passive_vc is used as ClientVC by HttpSM
- Plugin gets active_vc after the call returns
- passive_vc is set with internal request flag

In the implementation of TSHttpTxnServerIntercept(),

- HttpSM will get the member active_vc via PluginVCCore::connect_re() and use it as ServerVC
- After the PluginVCCore::connect_re() is called, the Plugin will receive the TS_EVENT_NET_ACCEPT time and get the passive_vc

## definition

PluginIdentity is the base class used to identify PluginVC identity

- Its existence is mainly used to judge whether NetVC is a real NetVC or a NetVC masqueraded by PluginVC when necessary.
- Information about the plugin that created this PluginVC can be obtained via PluginId

Since PluginVC needs to be disguised as a NetVConnection, it inherits from two classes.

`` `
class PluginVC : public NetVConnection, public PluginIdentity
{
  friend class PluginVCCore;

public:
  PluginVC(PluginVCCore *core_obj);
  ~PluginVC();

  // Start: A way to implement basic NetVC
  virtual VIO *do_io_read(Continuation *c = NULL, int64_t nbytes = INT64_MAX, MIOBuffer *buf = 0);

  virtual VIO *do_io_write(Continuation *c = NULL, int64_t nbytes = INT64_MAX, IOBufferReader *buf = 0, bool owner = false);

  virtual void do_io_close(int lerrno = -1);
  virtual void do_io_shutdown(ShutdownHowTo_t howto);

  // Reenable a given vio.  The public interface is through VIO::reenable
  virtual void reenable(VIO *vio);
  virtual void reenable_re(VIO *vio);

  // Timeouts
  virtual void set_active_timeout(ink_hrtime timeout_in);
  virtual void set_inactivity_timeout(ink_hrtime timeout_in);
  virtual void cancel_active_timeout();
  virtual void cancel_inactivity_timeout();
  virtual void add_to_keep_alive_queue();
  virtual void remove_from_keep_alive_queue();
  virtual bool add_to_active_queue();
  virtual ink_hrtime get_active_timeout();
  virtual ink_hrtime get_inactivity_timeout();

  // Pure virutal functions we need to compile
  virtual SOCKET get_socket();
  virtual void set_local_addr();
  virtual void set_remote_addr();
  virtual int set_tcp_init_cwnd(int init_cwnd);
  virtual int set_tcp_congestion_control(int);

  virtual void apply_options();

  virtual bool get_data(int id, void *data);
  virtual bool set_data(int id, void *data);
  // End
  
  // Get the PluginVC object on the other end
  virtual PluginVC *
  get_other_side()
  {
    return other_side;
  }

  // method used to implement PluginIdentity
  //@{ @name Plugin identity.
  /// Override for @c PluginIdentity.
  virtual const char *
  getPluginTag () const
  {
    return plugin_tag;
  }
  /// Override for @c PluginIdentity.
  virtual int64_t
  getPluginId () const
  {
    return plugin_id;
  }

  /// Setter for plugin tag.
  virtual void
  setPluginTag(const char *tag)
  {
    plugin_tag = tag;
  }
  /// Setter for plugin id.
  virtual void
  setPluginId(int64_t id)
  {
    plugin_id = id;
  }
  // @}

  // main callback handler
  int main_handler(int event, void *data);

private:
  void process_read_side(bool);
  void process_write_side(bool);
  void process_close();
  void process_timeout(Event **e, int event_to_send);

  void setup_event_cb(ink_hrtime in, Event **e_ptr);

  void update_inactive_time();
  int64_t transfer_bytes(MIOBuffer *transfer_to, IOBufferReader *transfer_from, int64_t act_on);

  // magic value for debugging
  uint32_t magic;
  // PluginVC type, enumeration value
  // PLUGIN_VC_UNKNOWN: Unknown (default, prevents uninitialization)
  // PLUGIN_VC_ACTIVE: active side
  // PLUGIN_VC_PASSIVE: Passive side
  PluginVC_t vc_type;
  // refers to the pointer back to PluginVCCore
  PluginVCCore *core_obj;

  // pointer to PluginVC on the other end
  PluginVC *other_side;

  // consistent with the read and write member functions in NetVC
  // The PluginVCState type is very close to the definition of the NetState type
  PluginVCState read_state;
  PluginVCState write_state;

  // mark whether process_read_side() and process_write_side() need to be called
  bool need_read_process;
  bool need_write_process;

  // Indicates whether PluginVC has been closed, consistent with NetVC::closed
  volatile bool closed;
  // Event called by main_handler that is triggered by the state machine using PluginVC
  Event *sm_lock_retry_event;
  // Event called by PluginVC / PluginVCCore that calls back to main_handler
  Event *core_lock_retry_event;

  // Can it be released?
  bool deletable;
  // Reenter count, consistent with NetVC::reentrancy_count
  int reentrancy_count;

  // active timeout
  ink_hrtime active_timeout;
  / / Callback PluginVC's Event after the active timeout
  Event *active_event;

  // inactive timeout
  ink_hrtime inactive_timeout;
  ink_hrtime inactive_timeout_at;
  / / Callback PluginVC's Event after inactive timeout
  // This Event is a periodic event. After each call to main_handler, determine whether the two variables are timed out by judging the above two variables.
  Event *inactive_event;

  // for PluginIdentify
  const char *plugin_tag;
  int64_t plugin_id;
};
`` `

### Initialization and construction

PluginVC does not appear alone, it always exists as a member of PluginVCCore.
Therefore, PluginVC is obtained by creating a PluginVCCore instance with alloc().

`` `
PluginVCCore *
PluginVCCore::alloc()
{
  PluginVCCore *pvc = new PluginVCCore;
  pvc->init();
  return pvc; 
}
`` `

Initialize by calling init()

- Create mutex
- Initialize two PluginVC members, active_vc and passive_vc
- Create MIOBuffer and IOBufferReader for bidirectional data transfer

`` `
void
PluginVCCore::init()
{
  // Use a separate mutex
  mutex = new_ProxyMutex();

  / / Initialize the PluginVC on the active side
  active_vc.vc_type = PLUGIN_VC_ACTIVE;
  active_vc.other_side = &passive_vc;
  active_vc.core_obj = this;
  active_vc.mutex = mutex;
  active_vc.thread = this_ethread();

  / / Initialize the PluginVC on the passive side
  passive_vc.vc_type = PLUGIN_VC_PASSIVE;
  passive_vc.other_side = &active_vc;
  passive_vc.core_obj = this;
  passive_vc.mutex = mutex;
  passive_vc.thread = active_vc.thread;

  / / Initialize the data transfer buffer between two PluginVC
  // These buffers can only be accessed after plugging the PluginVC and its corresponding SM
  // p_to_a:
  // Get data from the write VIO on the passive side to write p_to_a_buffer
  // Get data from p_to_a_reader to write read VIO on active side
  p_to_a_buffer = new_MIOBuffer(BUFFER_SIZE_INDEX_32K);
  p_to_a_reader = p_to_a_buffer->alloc_reader();
  // a_to_p:
  // Get data from the write VIO on the active side to write a_to_p_buffer
  // Get data from a_to_p_reader to write read VIO on the passive side
  a_to_p_buffer = new_MIOBuffer(BUFFER_SIZE_INDEX_32K);
  a_to_p_reader = a_to_p_buffer->alloc_reader();

  Debug("pvc", "[%u] Created PluginVCCore at %p, active %p, passive %p", id, this, &active_vc, &passive_vc);
}
`` `

Since there are a large number of operations for setting callbacks in PluginVC, setup_event_cb() was created to implement this setting.

There is a description at the head of the function. In order to avoid the lock problem, the Event pointer passed to setup_event_cb is two different events.

- sm_lock_retry_event
  - After you get the lock of the current PluginVC's VIO, you can set the callback for the event.
  - Usually after the PluginVC callback SM, when SM performs do_io, reenable, etc. operations on PluginVC, it will call back the PluginVC through the Event.
  - If the lock of VIO is not obtained, set the callback to PluginVC directly through this Event, which may cause Event leak
- core_lock_retry_event
  - After getting the lock of the current PluginVC itself, you can set the callback for the event.
  - Since PluginVCCore and two PluginVC members share the same mutex, when the two PluginVCs need to notify each other, this event can arrange each other's callbacks.

`` `
// void PluginVC::setup_event_cb(ink_hrtime in)
//
//    Setup up the event processor to call us back.
//      We've got two different event pointers to handle
//      locking issues
//
void
PluginVC::setup_event_cb(ink_hrtime in, Event **e_ptr)
{
  ink_assert(magic == PLUGIN_VC_MAGIC_ALIVE);

  // The incoming Event pointer must be NULL, otherwise
  / / Need to cancel (), then create an Event, otherwise it will lead to memory leaks
  // The design here is:
  / / Do not reset the Event, that is, do not repeat the callback
  // Create a new Event only if no callback is created
  if (*e_ptr == nullptr) {
    // We locked the pointer so we can now allocate an event
    //   to call us back
    // If in == 0 means immediate callback
    if (in == 0) {
      // need a thread of type REGULAR to perform the callback
      if (this_ethread()->tt == REGULAR) {
        *e_ptr = this_ethread()->schedule_imm_local(this);
      } else {
        *e_ptr = eventProcessor.schedule_imm(this);
      }
    // Otherwise indicates a timed callback
    } else {
      // need a thread of type REGULAR to perform the callback
      if (this_ethread()->tt == REGULAR) {
        *e_ptr = this_ethread()->schedule_in_local(this, in);
      } else {
        *e_ptr = eventProcessor.schedule_in(this, in);
      }
    }
  }
}
`` `

### PluginVC Read I/O Process Analysis

PluginVC::do_io_read() The previous code is similar to NetVC::do_io_read()

- Set MIOBuffer
- Set up VIO

Then set

- need_read_process = true means that main_handler needs to be read
- Arranging a callback to PluginVC::main_handler via setup_event_cb()

Finally, return to VIO

`` `
PluginVC::do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf)                                                                        
{
  ink_assert(!closed);
  ink_assert(magic == PLUGIN_VC_MAGIC_ALIVE);

  / / Set VIO
  if (buf) {
    read_state.vio.buffer.writer_for(buf);
  } else {
    read_state.vio.buffer.clear();
  }

  // After BUG:buffer.clear(), it may continue to set nbytes > 0
  // Note: we set vio.op last because process_read_side looks at it to
  //  tell if the VConnection is active.
  read_state.vio.mutex     = c->mutex;
  read_state.vio._cont     = c;
  read_state.vio.nbytes    = nbytes;
  read_state.vio.ndone = 0;
  read_state.vio.vc_server = (VConnection *)this;
  read_state.vio.op        = VIO::READ;

  Debug("pvc", "[%u] %s: do_io_read for %" PRId64 " bytes", core_obj->id, PVC_TYPE, nbytes);

  // Since reentrant callbacks are not allowed on from do_io
  //   functions schedule ourselves get on a different stack
  // After BUG:buffer.clear(), need_read_process should be set to false
  need_read_process = true;
  setup_event_cb(0, &sm_lock_retry_event);

  return &read_state.vio;
}
`` `

After setting need_read_process, EventSystem will call back PluginVC::main_handler() and then call process_read_side(false).

- Implement receiving data from one PluginVC and then putting it into the data buffer on the other end
- Since process_read_side() contains processing in two cases, it is very complicated.
- So I will mark it in the code, and the unmarked part is the common code that is needed in both cases.

`` `
// void PluginVC::process_read_side()
//
//   This function may only be called while holding
//      this->mutex & while it is ok to callback the
//      read side continuation
//
//   Does read side processing
//
void
PluginVC::process_read_side(bool other_side_call)
{
  ink_assert(!deletable);
  ink_assert(magic == PLUGIN_VC_MAGIC_ALIVE);

  // TODO: Never used??
  // MIOBuffer *core_buffer;

  IOBufferReader *core_reader;

  // Select the corresponding MIOBuffer and IOBufferReader according to the current PluginVC type (Active / Passive)
  // MIOBuffer and IOBufferReader used to transfer data from the Passive side to the Active side
  //   core_obj->p_to_a_buffer
  //   core_obj->p_to_a_reader
  // MIOBuffer and IOBufferReader used to transfer data from Active to Passive
  //   core_obj->a_to_p_buffer
  //   core_obj->a_to_p_reader
  // If the current PluginVC is Active and is currently a read operation, it is transferring data from Passive to Active.
  if (vc_type == PLUGIN_VC_ACTIVE) {
    // core_buffer = core_obj->p_to_a_buffer;
    core_reader = core_obj->p_to_a_reader;
  } else {
    // The opposite is to transfer data from Active to Passive
    ink_assert(vc_type == PLUGIN_VC_PASSIVE);
    // core_buffer = core_obj->a_to_p_buffer;
    core_reader = core_obj->a_to_p_reader;
  }

//************ START: other_side_call == false ************
  // reset need_read_process
  need_read_process = false;
//************ END: other_side_call == false ************

  // BUG? For other_side_call == true, it is judged that there is a problem with the lock without lock.
  if (read_state.vio.op != VIO::READ || closed) {
    return;
  }

//************ START: other_side_call == true ************
  // This only needs to lock the read VIO when other_side_call == true (BUG? Should also lock the write VIO)
  // Otherwise, the lock operation will succeed because both read VIO and write VIO are locked in main_handler
  // The following code
  // Acquire the lock of the read side continuation
  EThread *my_ethread = mutex->thread_holding;
  ink_assert(my_ethread != NULL);
  MUTEX_TRY_LOCK(lock, read_state.vio.mutex, my_ethread);
  if (!lock.is_locked()) {
    // If the lock fails, arrange the other_side callback via core_lock_retry_event
    // read the data again during the callback
    Debug("pvc_event", "[%u] %s: process_read_side lock miss, retrying", core_obj->id, PVC_TYPE);

    need_read_process = true;
    // Note that you want to use core_lock_retry_event
    setup_event_cb(PVC_LOCK_RETRY_TIME, &core_lock_retry_event);
    // Return to local_side write process (process_write_side) from other_side processing
    return;
  }

  Debug("pvc", "[%u] %s: process_read_side", core_obj->id, PVC_TYPE);
  // Reset need_read_process again, this is very important
  // When other_side_call == true, you need to be able to set it accurately after locking.
  need_read_process = false;
  // BUG? I feel that there seems to be less judgment on closed. After locking, I should judge closed again.
//************ END: other_side_call == true ************

  // 判断 read shutdown
  // Check read_state.shutdown after the lock has been obtained.
  if (read_state.shutdown) {
    return;
  }

  / / Judgment ntodo
  // Check the state of our read buffer as well as ntodo
  int64_t ntodo = read_state.vio.ntodo();
  if (ntodo == 0) {
    return;
  }

  // Prepare to read data from PluginVCCore's internal MIOBuffer to read VIO
  int64_t bytes_avail = core_reader->read_avail();
  int64_t act_on = MIN(bytes_avail, ntodo);

  Debug("pvc", "[%u] %s: process_read_side; act_on %" PRId64 "", core_obj->id, PVC_TYPE, act_on);

  // ntodo cannot be 0, so here is actually judging bytes_avail
  if (act_on <= 0) {
    // The peer is closed, or the peer is closed.
    // callback EOS
    if (other_side->closed || other_side->write_state.shutdown) {
      read_state.vio._cont->handleEvent(VC_EVENT_EOS, &read_state.vio);
    }
    return;
  }
  // Bytes available, try to transfer from the PluginVCCore
  //   intermediate buffer
  //
  / / Read data from core_reader, write to vio buffer
  MIOBuffer *output_buffer = read_state.vio.get_writer();

  int64_t water_mark = output_buffer->water_mark;
  water_mark = MAX(water_mark, PVC_DEFAULT_MAX_BYTES);
  int64_t buf_space = water_mark - output_buffer->max_read_avail();
  / / Confirm that vio buffer has space to receive data
  if (buf_space <= 0) {
    Debug("pvc", "[%u] %s: process_read_side no buffer space", core_obj->id, PVC_TYPE);
    return;
  }
  act_on = MIN(act_on, buf_space);

  // data transmission
  int64_t added = transfer_bytes(output_buffer, core_reader, act_on);
  if (added <= 0) {
    // Couldn't actually get the buffer space.  This only
    //   happens on small transfers with the above
    //   PVC_DEFAULT_MAX_BYTES factor doesn't apply
    Debug("pvc", "[%u] %s: process_read_side out of buffer space", core_obj->id, PVC_TYPE);
    return;
  }

  read_state.vio.ndone += added;

  Debug("pvc", "[%u] %s: process_read_side; added %" PRId64 "", core_obj->id, PVC_TYPE, added);

  // Determine the callback READY or COMPLETE event based on ntodo
  if (read_state.vio.ntodo() == 0) {
    read_state.vio._cont->handleEvent(VC_EVENT_READ_COMPLETE, &read_state.vio);
  } else {
    read_state.vio._cont->handleEvent(VC_EVENT_READ_READY, &read_state.vio);
  }

  // refresh inactive timeout like net_activity()
  update_inactive_time();

  // Wake up the other side so it knows there is space available in
  //  intermediate buffer
  / / Because the core buffer data is consumed, notify the peer to fill the data as much as possible
  if (!other_side->closed) {
    if (!other_side_call) {
      // from the local PluginVC::main_handler
      // Fill the data with the process_write_side(true) of the peer
      // process_write_side(), like process_read_side(), supports calls from both ends
      other_side->process_write_side(true);
    } else {
      // From the opposite PluginVC::main_handler
      // The write_state.vio.mutex on the opposite end is locked.
      // Therefore, it is safe to execute reenable() callback via sm_lock_retry_event
      other_side->write_state.vio.reenable();
    }
  }
}
`` `

### PluginVC Write I/O Process Analysis

PluginVC::do_io_write() The previous code is similar to NetVC::do_io_write()

- Set MIOBuffer
- Set up VIO

Then set

- need_write_process = true means that main_handler needs to be written
- Arranging a callback to PluginVC::main_handler via setup_event_cb()

Finally, return to VIO

## References

- TSHttpConnectWithPluginId
- [Plugin.h](http://github.com/apache/trafficserver/tree/master/proxy/Plugin.h)
- [PluginVC.h](http://github.com/apache/trafficserver/tree/master/proxy/PluginVC.h)
- [PluginVC.cc](http://github.com/apache/trafficserver/tree/master/proxy/PluginVC.cc)
