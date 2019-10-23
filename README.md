# Config for haproxy of blockscout.com

About:

Blockscout.com uses [haproxy](http://www.haproxy.org/) to proxy all hosted instances by BlockScout team and external instances.

If you'd like to add your instance to Blockscout.com please submit a PR following the instruction below:


## Instruction

Blockscout uses two-part path names for network urls: the first part (`org`) is used to designate organization/type of the network, the second part (`netname`) is used to name the network. For example,
  * `https://blockscout.com/eth/mainnet` - url for ETH mainnet (`org=eth`, `netname=mainnet`)
  * `https://blockscout.com/etc/mainnet` - for ETC mainnet (`org=etc`, `netname=mainnet`)
  * `https://blockscout.com/eth/kovan` - Kovan testnet (`org=eth`, `netname=kovan`)

0. If you want a new `org` part for your blockscout instance's url, you need to add a new `acl` under `Check for network type` section of the cfg file.
```
#Check for network type
...
acl is_myorg path_beg -i /myorg
```
And add a corresponding exception `!is_myorg` to `default redirect` section
```
# default redirect
redirect prefix /poa/xdai if !is_websocket !is_poa !is_etc !is_eth !is_lukso !is_rsk !is_myorg !is_cookie
```
and also to **every line** in `redirect for networks` section
```
#redirect for networks
redirect prefix /poa/core if !is_websocket !is_poa !is_etc !is_eth !is_lukso !is_rsk !is_myorg is_cookie_core
redirect prefix /poa/sokol if !is_websocket !is_poa !is_etc !is_eth !is_lukso !is_rsk !is_myorg is_cookie_sokol
# do this for every line
...
```
After this you can proceed to the next step.

1. In `Check for network name` section add a new `acl` with your `netname` similar to existing ones
```
#Check for network name
...
acl is_mynet path_beg -i /myorg/mynet
```

2. Add the following lines to set cookie and name of rpc/websockets servers
```
#Check for cookies
...
acl is_cookie_mynet hdr_sub(cookie) network=mynet

#Default backends
...
use_backend mynet if is_mynet

#WebSocket backends
...
use_backend mynet_ws if is_cookie_mynet is_websocket
```

3. Add new `backend_mynet` and `backend_mynet_ws` sections to the end of the file and provide dns names of your blockscout instances, e.g.
```
backend mynet
    #Proxy mode
    mode http

    #Backend queries should not have network prefix, so we delete it
    acl is_mynet path_beg /myorg/mynet
    reqirep ^([^\ ].*)myorg/mynet[/]?(.*) \1\2 if is_mynet

    #Setting headers to mask our query
    http-request set-header X-Forwarded-Host %[req.hdr(Host)]
    http-request set-header X-Client-IP %[src]
    http-request set-header Host my-blockscout-instance.com

    #Specifying cookie to insert
    cookie network insert

    #Server list. Format:
    #server <name> <address>:<port> cookie <network_name> check inter 60000 fastinter 1000 fall 3 rise 3 ssl verify none
    server mynet_server my-blockscout-instance.com:443 cookie mynet check inter 60000 fastinter 1000 fall 3 rise 3 ssl verify none

backend mynet_ws
    #Check if WebSocket request is correct
    acl hdr_websocket_key      hdr_cnt(Sec-WebSocket-Key)      eq 1
    acl hdr_websocket_version  hdr_cnt(Sec-WebSocket-Version)  eq 1
    http-request deny if ! hdr_websocket_key ! hdr_websocket_version

    #WebSockets should be forwarded, not http proxied
    option forwardfor

    #Setting headers to mask our query
    http-request set-header Host my-blockscout-instance.com
    http-request set-header Origin https://my-blockscout-instance.com

    #Specifying cookie to insert
    cookie network insert

    #Server list. Format:
    #server <name> <address>:<port> cookie <network_name> check inter 60000 fastinter 1000 fall 3 rise 3 check inter 60000 fastinter 1000 fall 3 rise 3 ssl verify none maxconn 30000
    server mynet_server my-blockscout-instance.com:443 cookie mynet check inter 60000 fastinter 1000 fall 3 rise 3 ssl verify none maxconn 30000
```
Note that you should change the following parts to match your case:
  * `backend mynet`:
    - `acl is_...`
    - `reqirep ...`
    - `http-request set-header Host ...`
    - `server mynet_server...` - don't forget to specify port 443 here

  * `backend mynet_ws`:
    - `http-request set-header Host...`
    - `http-request set-header Origin ...` don't forget `https` here
    - `server mynet_server...` - don't forget port 443 here
