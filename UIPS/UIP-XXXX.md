---
uip: "XXXX"
title: %mach
description: TCP vane
author: ~bonbud-macryg
status: Draft
type: Standards Track
category: Kernel
created: 2026-10-15
---

## Abstract

A new vane, Mach, serves as a Transmission Control Protocol (TCP) client for Gall agents. Mach is a TCP client in its first release, and may be upgraded to function as a TCP server after its first release. Gall agents can use Mach to open and close communication channels with servers such as Bitcoin nodes, running locally or remotely, without need for a complex sidecar integration. A general-purpose TCP client/server vane completes the system of Urbit networking: where Ames is ship-to-ship and [Lick](https://github.com/urbit/UIPs/blob/main/UIPS/UIP-0101.md) ship-to-host, Mach opens the way to ship-to-server communication across any application-level protocol built on top of TCP/IP.

## Status

Mach is planned for release on Groundwire’s Arvo distribution in a 407k version bump. Mach PRs will target the upstream `urbit/urbit` and `urbit/vere` repos, to be merged as the Core Guild sees fit.

## Motivation

Groundwire needs to read the Bitcoin blockchain in order to populate its Urbit PKI. This requires at minimum a Bitcoin light client running on the ship, which we have at the `gwbtc/node` repo. In our current implementation this desk’s `%bitcoin-client` agent talks to another agent, `%tcp-sidecar`, which is itself just a passthrough ￼￼￼via Lick to a TCP sidecar written in C. In practice getting this sidecar installed for end-users, keeping it alive across Vere lifecycles, and securing this pipeline is unacceptably complex.

A TCP client vane will allow an agent like `%bitcoin-client` to communicate directly with some known Bitcoin node and handle the lifecycle in its own application logic.

### TCP Server

Mach may later be extended to serve incoming connections, which would let TCP-based protocols (SSH, IMAP/SMTP, WebSockets) be implemented as ordinary Gall agents. A server would introduce port binding to Vere and create a significant new attack surface to Vere, and it’s beyond what Groundwire needs from Arvo right now.

## Specification

In its first release, Mach SHALL be a minimal interface to a TCP driver in the runtime.

### Interface

Mach assumes a `$fief` — an IP address or domain name — which is included in Groundwire’s Lull.

```hoon
+$  fief
  $%  [%turf p=(list turf) q=@ud]
      [%if p=@if q=@ud]
      [%is p=@is q=@ud]
  ==
```

#### Tasks, gifts

```hoon
++  mach  ^?
  |%
  +$  wye
    $?  %tls                                            ::  failed tls
        %dead                                           ::  peer not answering keepalive
        %reset                                          ::  connection reset
        %reborn                                         ::  connection rest on new unix process
        %refused                                        ::  peer refused us
        %timeout                                        ::  connect timed out
        %overflowed                                     ::  exceeded queue
        %unresolved                                     ::  address did not resolve
    ==
  ::
  +$  task
    $~  [%vega ~]
    $%  $>(%born vane-task)                             ::  new unix process
        $>(%trim vane-task)                             ::  trim state
        $>(%vega vane-task)                             ::  report upgrade
        [%connect secure=? =fief]                       ::  connect to address
        [%send =octs]                                   ::  send on this connection
        [%close ~]                                      ::  close this connection
        [%opened id=@ud]                                ::  vere opened conn.
        [%received id=@ud =octs]                        ::  vere received bytes
        [%closed id=@ud p=(unit wye)]                   ::  vere closed this connection
    ==
  ::
  +$  gift
    $%  [%connected ~]                                  ::  connection opened
        [%received =octs]                               ::  received bytes
        [%closed p=(unit wye)]                          ::  connection closed
        [%connect id=@ud secure=? =fief]                ::  vere, open connection
        [%send id=@ud =octs]                            ::  vere, send bytes
        [%close id=@ud]                                 ::  vere, close connection
    ==
  --
```

#### Scries

* `/ma/connections` — list open connections
* `/md/owner/<connection>` — get agent that opened this connection
* `/mu/<connection>` — does this connection exist

### Lifecycle

Upon first installation, Mach MUST initialize a default `//mach` duct as bunt `unix-duct=_[//mach ~]`, as Lick does.

On `%born`, Mach MUST drop all outgoing connections, as Iris and Lick do, and tell the Gall agents that own them about these dropped connections with `%reborn`. Once Gall resource tracking is implemented, Gall SHOULD tell Mach to clean up an agent’s outgoing connections when the agent is nuked.

### Runtime

In its first release the Mach vane is a minimal interface with the TCP driver. All congestion control, bindings, timeouts, etc. MUST be specified as hard-coded constants in the driver. The driver MUST limit the size of each receive event, and maintain per-connection queues where one `%received` gift is in transit per connection until Arvo has committed it to the event log.

The driver MUST enforce the following constraints:
* `%received` events MUST carry no more than 1 MiB
* Per-connection queues MUST be capped at 64 MiB, failing with `%overflow`
* Connections MUST be closed with a 30 second drain limit
* Outgoing connection attempts MUST have a timeout of 30 seconds
* Peers MAY remain idle for 120 seconds before the driver sends a keepalive probe, with libuv enforcing probe intervals and max. attempts consistently across platforms; Gall agents MAY implement their own keepalive pings within that 120-second window if they wish

The above constraints protect from resource exhaustion and make Arvo and Vere behaviour consistent across host platforms. Conversely, the driver SHALL run with `TCP_NODELAY` on, such that batching is left to the agents and peers that understand whichever protocol they’re communicating over.

The driver SHALL connect to peers over h2o, which is already a Vere dependency in `http.c` and `cttp.c`. For a plain connection, the driver resolves the address with `uv_getaddrinfo`, tries each result in order with `h2o_socket_connect`, and reads and writes through the appropriate `h2o_socket_*` functions. For a `secure=&` connection, the driver calls `h2o_socket_ssl_handshake` with the hostname after connecting, with h2o verifying the TLS certificate for us.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

## Rationale

WebSocket support was added to Groundwire’s distribution in March 2026 following ~[UIP-0125](https://github.com/urbit/UIPs/blob/main/UIPS/UIP-0125.md)~: the client in Iris and `cttp.c`, the server in Eyre and `http.c` (~[gwbtc/urbit\#23](https://github.com/gwbtc/urbit/pull/23)~, ~[gwbtc/vere\#4](https://github.com/gwbtc/vere/pull/4)~). That split a single protocol across two vanes and two drivers. A TCP vane would have contained it in a userspace.

This vane should not set a precedent that unsupported protocols should be candidates for support in either the kernel or the `%base` desk. The entire class of protocols that builds on top of TCP should be, by default, excluded from those codebases *because* this vane enables them to be implemented in userspace. HTTP/HTTPS is unique in that Arvo builds on it for user authentication in the browser, the `/~debug` dashboard, and streaming logs to the runtime.

## Backwards Compatibility

A Kelvin decrement will be necessary.

The main backwards compatibility issue is between the `gwbtc/{urbit,vere}` repos and upstream `urbit/{urbit, vere}` repos.

This will require some careful scoping of branches and upstream PRs on Groundwire’s part, but releasing a 407k to Groundwire users while mainstream Urbit remains on 408k is no stranger than the “alt-408k” we’ve already been distributing for around eight months. A prerequisite for this will be rebasing Groundwire’s release from `urbit/urbit:next/kelvin/407`.

Our goals are to ensure as much continuity as possible between Groundwire Alpha releases, and to make it as easy as possible to eventually upstream Groundwire’s kernel changes to `urbit/urbit`. Network compatibility between Groundwire comets and Azimuth ships is already achieved by the Suite C changes outlined in [UIP-0136](https://github.com/urbit/UIPs/blob/main/UIPS/UIP-0136.md). We have explicitly disclaimed continuity of Urbit data and Groundwire identity between our Alpha release and a future Beta / production release.

## Reference Implementation

{XX - will be updated with links to PRs in `gwbtc/...` repos}

## Security Considerations

Mach’s initial release will only allow Gall agents to connect to outside servers, never the other way around. The runtime driver’s tasks are only accepted from the `//mach` duct. We defer to the Gall Working Group’s userspace permissions system to manage which agents and desks can pass tasks to Mach.

Mach will not include Transport Layer Security (TLS) in its first release, as this is not necessary for its initial goal of talking to a Bitcoin fullnode. Putting TLS in the driver will be a requirement for talking to most Earth servers that have not been built with Mach in mind. TLS will be implemented before Mach’s upstream PRs are opened. The driver will rely on Vere’s h2o dependency for TLS.

TLS connections will be verified against the certificates embedded in Vere, the same Iris uses for HTTPS. TLS connections (`secure=&`) will only be valid if the given `$fief` is a `%turf`, and Mach will reject `%connect` tasks from userspace on this basis.

As with Iris, any Gall agent can reach any TCP endpoint the host OS can, including loopback. This is a userspace security question and out-of-scope for this UIP.

The TCP driver’s hard-coded resource limits protect the ship from DoS attacks. Building this vane out into a TCP server will require more security concerns around ports, auth, etc. which are out-of-scope for this UIP.

## Copyright

Copyright and related rights waived via [CC0](/urbit/UIPs/blob/main/LICENSE.md).
