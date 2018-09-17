# core component: SocksProxy

SocksProxy is the state machine that ATS receives as a Socks Proxy from Socks Client requests.

In SocksProxy, after identifying the Socks request that satisfies the following conditions, it will be transferred to HttpSM for processing:

- The Socks command is CONNECT
- The target IP is IPv4
- The target port is 80

For those that do not meet the conditions, pass the Socks request transparently according to the configuration in socks.config and records.config (Tunnel)

## definition

## References

- [SocksProxy.cc](http://github.com/apache/trafficserver/tree/master/proxy/SocksProxy.cc)
