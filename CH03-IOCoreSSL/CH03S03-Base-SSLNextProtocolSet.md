# Base component: SSLNextProtocolSet

In order to implement NPN and ALPN, ATS defines SSLNextProtocolSet to support the processing of multiple protocols.

SSLNextProtocolSet is mainly used to generate a list of supported protocols.

   - Add and remove supported protocols via register and unregister methods
   - After each addition, deletion (bug?), the list of supported protocols npn is generated according to the requirements of NPN and/or ALPN, and the length is npnsz.
   - Use advertise to get a list of generated protocols for use when interacting with NPN and/or ALPN
   - Find the state machine corresponding to a specific protocol by find

## 定义

```
class SSLNextProtocolSet
{
public:
  // Constructor
  // Initialize npn=NULL, npnsz=0
  SSLNextProtocolSet();
  // Destructor
  // Release the npn memory, and then traverse the endpoints linked list, one by one delete the elements on the linked list
  ~SSLNextProtocolSet();

  // Register a state machine and supported protocol strings
  // Cannot double registration for the same protocol
  // When registering a new protocol, it will be a new NextProtocolEndpoint and placed in the endpoints list.
  // true=Successful registration, traversing endpoints to generate the latest npn and npnsz
  // false=registration failed, the protocol has been registered or the protocol string exceeds 255 bytes
  bool registerEndpoint(const char *, Continuation *);
  // Unregister a state machine and supported protocol strings
  // Traverse the endpoints, compare the protocol string, find the to delete, directly call the remove method of the endpoints list to delete
  // true=Successfully logged out
  // false=logout failed, no protocol string was found to be logged out
  // There is a memory leak: after removing the element from the linked list, the memory occupied by the element is not freed by the delete method.
  // There is a bug here: no refresh npn and npnsz
  // However, looking at the entire ATS code, it seems that no component has called this logout method~
  bool unregisterEndpoint(const char *, Continuation *);
  // Get the current npn and npnsz and store them in *out and *len respectively
  // true=success, false=currently no npn and npnsz
  // followed by a const keyword indicating that this function does not modify the value of the class member
  bool advertiseProtocols(const unsigned char **out, unsigned *len) const;
  // Traverse the endpoints list and compare the protocol names one by one
  // returns the endpoint member in the object of the NextProtocolEndpoint type that exactly matches the list
  // returns NULL to indicate that there is no matching NextProtocolEndpoint object in the list.
  // followed by a const keyword indicating that this function does not modify the value of the class member
  Continuation *findEndpoint(const unsigned char *, unsigned) const;

  struct NextProtocolEndpoint {
    // NOTE: the protocol and endpoint are NOT copied. The caller is
    // responsible for ensuring their lifetime.
    // Constructor
    // member protocol points to the passed parameter protocol
    // member endpoint points to the passed in parameter endpoint
    // Since there is no copy operation, the caller must ensure that the object pointed to by the passed argument is not released.
    NextProtocolEndpoint(const char *protocol, Continuation *endpoint);
    // Destructor
    // There is no empty function for any operation, because there is no need to recycle resources, etc.
    ~NextProtocolEndpoint();

    const char *protocol;
    Continuation *endpoint;
    LINK(NextProtocolEndpoint, link);

    typedef DLL<NextProtocolEndpoint> list_type;
  };

private:
  SSLNextProtocolSet(const SSLNextProtocolSet &);            // disabled
  SSLNextProtocolSet &operator=(const SSLNextProtocolSet &); // disabled

  // The mutable keyword seems to be in order to correspond to the two methods advertiseProtocols and findEndpoint
  // But it doesn't feel necessary?
  mutable unsigned char *npn;
  mutable size_t npnsz;

  // Define a doubly linked list
  // Here is no need for DLL<NextProtocolEndpoint> endpoints; ? Do you have to use a typedef?
  NextProtocolEndpoint::list_type endpoints;
};
```

## Reference material

- [P_SSLNextProtocolSet.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNextProtocolSet.h)
- [SSLNextProtocolSet.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNextProtocolSet.cc)
