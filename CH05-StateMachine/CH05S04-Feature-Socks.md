# New Features / Function: Socks

The Socks feature is described below, and the Socks feature includes:

- Socks Proxy
  - It is the easiest proxy feature in ATS, it takes advantage of OneWayTunnel and TwoWayTunnel
  - It allows ATS to parse requests from Socks Client
  - We can learn how to use the Tunnel state machine by analyzing the Socks Proxy code.
- Socks Parent
  - It allows ATS to initiate access to the source server through the Socks proxy server
  - It makes ATS available as a Socks Client

## SOCKS_WITH_TS

First, introduce a macro definition SOCKS_WITH_TS, which is defined in I_Socks.h. There are a lot of places in the Socks code to determine whether this macro is defined to apply different code designs.
After defining this macro, there will be some new enhancements, so when we do code analysis later, we will perform a comprehensive analysis based on the code with new features.
```
Source: I_Socks.h
/*When this is being compiled with TS, we enable more features the use
  Non modularized stuff.
  Ip_ranges and multiple socks server support.
*/
#define SOCKS_WITH_TS
```
## socks_conf_struct

Next, we'll look at a structure that stores the global configuration items for the Socks feature.

```
Source: P_Socks.h
Struct socks_conf_struct {
  // The following configuration items are used to serve the Socks Parent function.
  Int socks_needed; // true: all ATS connections to the origin server need to pass through the Socks proxy server
                                      // This configuration is also the main switch for the entire Socks function
  Int server_connect_timeout; // timeout for connecting to the Socks proxy server
  Int socks_timeout; // communication timeout after establishing a connection with the Socks proxy server
  Unsigned char default_version; // The Socks protocol version used by default when communicating with the Socks proxy server
  Char *user_name_n_passwd; // Pointer to username and password: [len(short int)] + "username" + [len(short int)] + "password"
  Int user_name_n_passwd_len; // Bytes of username and password +2 bytes

  Int per_server_connection_attempts; // The number of attempts to reconnect for each Socks proxy server when multiple Socks proxy servers are specified
  Int connection_attempts; // total number of attempts

  // the following ports are used by SocksProxy
  // The following configuration items are used to serve the Socks Proxy feature.
  Int accept_enabled; // true: enable parsing of Socks Client requests
                            // Ask socks_needed to be true, otherwise the resolved Socks request can't be sent to the Socks proxy server.
  Int accept_port; // Parse the request from the Socks Client on the specified port
                            // After enabling accept_enabled, you must set up a service port to receive Socks requests.
  Unsigned short http_port; // If the Socks Client request is CONNECT and the destination port is the specified port, then the request is forwarded to HttpSM for processing.

  IpMap ip_map; // When socks_needed == true, the target IP address contained in this table will not pass through the Socks proxy server.

  Socks_conf_struct() // configuration file default
    : socks_needed(0), // false(0) (required, true=minimum configuration item)
      Server_connect_timeout(0), // 10 seconds
      Socks_timeout(100), // 100 seconds
      Default_version(5), // Version 4
      User_name_n_passwd(NULL), // NULL (non-required)
      User_name_n_passwd_len(0), // 0 (non-required)
      Per_server_connection_attempts(1), // 1
      Connection_attempts(0), // 4
      Accept_enabled(0), // false(0) (required, true=minimum configuration item)
      Accept_port(0), // 1080
      Http_port(1080) // 80, here is a small bug, but it doesn't matter, the default value is correct when the configuration file is loaded.
  {
  }
};
```

## Configuring Socks

There are few configuration files for the Socks function in ATS.

- First, configure the default Socks Server, separated by semicolons between multiple Socks Servers.

```
CONFIG proxy.config.socks.default_servers STRING "xxxx:1080;xxxx:1080"
```

- Then, enable ATS's Socks Proxy function so that ATS can use Socks Server to access the source server.

```
CONFIG proxy.config.socks.socks_needed INT 1
```

- If you want ATS to filter out HTTP requests in Socks requests and serve them

```
CONFIG proxy.config.socks.accept_enabled INT 1
```

- There are also some socks rules that need to be configured. You can specify the default filename of the socks.config through the configuration item.

```
CONFIG proxy.config.socks.socks_config_file STRING socks.config
```

- Mainly configure the following three functions in socks.config:
  - Username and password required to connect to the Socks 5 proxy server
  - Do not use the target IP of the Socks proxy server (Socks exception)
  - Specify different Socks proxy servers for different target IPs (Socks rules)

### About accept_enabled

After accept_enabled, ATS acts as the pre-agent of Socks Server, requiring the client to use ATS as Socks Server. All Socks requests are sent to ATS first. ATS transfers the connection with destination port 80 to HttpSessionAccept for processing.
To modify the default HTTP port (80):

```
CONFIG proxy.config.socks.http_port INT 80
```

### About socks_needed

After opening socks_needed, the ATS external request will probably pass the Socks Server proxy.

- if no_socks is specified in socks.config
  - Connections will not be established through the socks proxy server for the specified target IP
- If the socks rule is specified in socks.config
  - Match the connection of the rule, establish a connection using the socks proxy server specified by the socks rule
  - If the socks rule is empty or does not match any of the rules, look at default_servers
- if default_servers is configured in records.config
  - All outbound requests match the socks rule first, and those that do not match will connect via default_servers

Two typical application scenarios are as follows:

1. All externally initiated connections pass through the specified Socks proxy server, and only specific target IP addresses do not pass through the Socks proxy server.
   - Configure default_servers for records.config to the specified Socks proxy server
   - Configure no_socks for socks.config to exclude specific target IPs
2. Only the specific target IP address passes through the Socks proxy server.
   - Configure default_servers for records.config to be empty
   - Configure the socks rules for socks.config to set a specific target IP through the specified Socks proxy server

## Configuring socks.config

Set the username and password required for the Socks 5 proxy server

- When configuring multiple Socks 5 agents, all agents will use the same username and password
- auth u <user_name> <pasword>

Set the Socks exception rule

- If you want to initiate a request for a specific target IP, instead of passing through the Socks Server proxy, the ATS initiates a connection directly to the OS.
- The target IP can be specified in the socks.config using the no_socks configuration item
- Can specify a single IP
  - no_socks x1.x2.x3.x4
- IP range can be specified
  - no_socks x1.x2.x3.x4 - y1.y2.y3.y4
- Multiple rules can be separated by commas
  - no_socks 123.14.84.1 - 123.14.89.4, 109.32.15.2

Set up Socks rules

- If you want to apply a different Socks Server proxy server to different target IPs,
- You can use configuration items similar to those in parent.config in socks.config
- but only the dest_ip configuration item is supported in socks.config
- and two dependent configuration items parent, round_robin
- E.g:
  - For the target IP address range: 216.32.0.0-216.32.255.255
  - Provide proxy with two socks servers: socks1:4080 and socks2:1080
  - The above two socks servers use the strict mode of round robin (the first socks proxy requests through socks1:4080 and the second through socks2:1080)

```
Dest_ip=216.32.0.0-216.32.255.255 parent="socks1:4080; socks2:1080" round_robin=strict
```

If the target IP is not included in the above rules, the Socks proxy server specified by proxy.config.socks.default_servers is used.

## References

- [I_Socks.h] (http://github.com/apache/trafficserver/tree/master/iocore/net/I_Socks.h)
- [P_Socks.h] (http://github.com/apache/trafficserver/tree/master/iocore/net/P_Socks.h)
- [mgmt/RecordsConfig.cc] (http://github.com/apache/trafficserver/tree/master/mgmt/RecordsConfig.cc)
