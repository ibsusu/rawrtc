# RAWRTC

[![CircleCI build status][circleci-badge]][circleci-url]
[![Travis CI build status][travis-ci-badge]][travis-ci-url]
[![Join our chat on Gitter][gitter-icon]][gitter]

A WebRTC and ORTC library with a small footprint.

## Features

The following list represents all features that are planned for RAWRTC.
Features with a check mark are already implemented.

* ICE [[draft-ietf-ice-rfc-5245bis-08]][ice]
  - [x] Trickle ICE [[draft-ietf-ice-trickle-07]][trickle-ice]
  - [x] IPv4
  - [x] IPv6
  - [x] UDP
  - [ ] TCP
* STUN [[RFC 5389]][stun]
  - [x] UDP
  - [ ] TCP
  - [ ] TLS over TCP
  - [ ] DTLS over UDP [[RFC 7350]][stun-turn-dtls]
* TURN [[RFC 5766]][turn]
  - [ ] UDP
  - [ ] TCP
  - [ ] TLS over TCP
  - [ ] DTLS over UDP [[RFC 7350]][stun-turn-dtls]
* Data Channel
  - [x] DCEP [[draft-ietf-rtcweb-data-protocol-09]][dcep]
  - [x] SCTP-based [[draft-ietf-rtcweb-data-channel-13]][sctp-dc]
* API
  - [x] WebRTC C-API based on the [W3C WebRTC API][w3c-webrtc] and
    [[draft-ietf-rtcweb-jsep-24]][jsep]
  - [x] ORTC C-API based on the [W3C CG ORTC API][w3c-ortc]

## Threading

1. *Does this use multiple threads?*

   No, this library is single-threaded and uses an event loop underneath.

2. *Can I use it in a threaded environment?*

   Yes. Just make sure you're always calling it from the same thread the event
   loop is running on, or either [lock/unlock the event loop thread][re-lock]
   or [use the message queues provided by re][re-mqueue] to exchange data with
   the event loop thread. However, it is important that you only run one *re*
   event loop in one thread.

3. *I only want a data channel implementation for my SFU!*

   Check out [RAWRTCDC][rawrtcdc].

## Prerequisites

The following tools are required:

* [git][git]
* [ninja][ninja] >= 1.5
* [meson][meson] >= 0.48.0
* pkg-config (`pkgconf` for newer FreeBSD versions)
* SSL development libraries (`libssl-dev` on Debian, `openssl` on OSX and
  FreeBSD)

## Build

```bash
cd <path-to-rawrtc>
mkdir build
meson build
cd build
ninja
```

## Run

RAWRTC provides a lot of tools that can be used for quick testing purposes and
to get started. Let's go through them one by one. If you just want to check out
data channels and browser interoperation, skip to the
[`peer-connection` tool section](#peer-connection) which uses the WebRTC API or
to the [`data-channel-sctp` tool section](#data-channel-sctp) which uses the
ORTC API.

Most of the tools have required or optional arguments which are shared among
tools. Below is a description for the various shared arguments:

#### offering

Whether the peer is going to create an offer. Provide `1` to create an offer
immediately or `0` to create an answer once the remote offer has been
processed.

Only used by WebRTC API tools.

#### ice-role

Determines the ICE role to be used by the ICE transport, where `0` means
*controlled* and `1` means *controlling*.

Only used by ORTC API tools.

#### sctp-port

The port number the internal SCTP stack is supposed to use. Defaults to `5000`.

Note: It doesn't matter which port you choose unless you want to be able to
debug SCTP messages. In this case, it's easier to distinguish the peers by
their port numbers.

Only used by ORTC API tools.

#### ice-candidate-type

If supplied, one or more specific ICE candidate types will be enabled and all
other ICE candidate types will be disabled. Can be one of the following
strings:

* *host*
* *srflx*
* *prflx*
* *relay*

Note that this has no effect on the gathering policy. The candidates will be
gathered but they will simply be ignored by the tool.

If not supplied, all ICE candidate types are enabled.

### ice-gatherer

API: ORTC

The ICE gatherer tool gathers and prints ICE candidates. Once gathering is
complete, the tool exits.

Usage:

    ice-gatherer

### ice-transport-loopback

API: ORTC

The ICE transport loopback tool starts two ICE transport instances which
establish an ICE connection. Once you see the following line for both clients
*A* and *B*, the ICE connection has been established:

    (<client>) ICE transport state: connected

Usage:

    ice-transport-loopback [<ice-candidate-type> ...]

### dtls-transport-loopback

API: ORTC

The DTLS transport loopback tool starts two DTLS transport instances which
work on top of an established ICE transport connection.

To verify that the DTLS connection establishes, wait for the following line for
both clients *A* and *B*:

    (<client>) DTLS transport state change: connected

Usage:

    dtls-transport-loopback [<ice-candidate-type> ...]

### sctp-redirect-transport

API: ORTC

The SCTP redirect transport tool starts an SCTP redirect transport on top of an
established DTLS transport to relay SCTP messages from and to a third party.
This tool has been developed to be able to test data channel implementations
without having to write the required DTLS and ICE stacks. An example of such a
testing tool is [dctt][dctt] which uses the kernel SCTP stack of FreeBSD.

Building:

This tool is not built by default. You can enable building it in the following
way:

```bash
cd <path-to-rawrtc>/build
meson configure -Dsctp_redirect_transport=true
ninja
```

Usage:

    sctp-redirect-transport <0|1 (ice-role)> <redirect-ip> <redirect-port>
                            [<sctp-port>] [<maximum-message-size>]
                            [<ice-candidate-type> ...]

Special arguments:

* `redirect-ip`: The IP address on which the external SCTP stack is listening.
* `redirect-port` The port on which the external SCTP stack is listening.
* `maximum-message-size`: The maximum message size of a data channel message
  the external SCTP stack is able to handle. `0` indicates that messages of
  arbitrary size can be handled. Defaults to `0`.

### data-channel-sctp-loopback

API: ORTC

The data channel SCTP loopback tool creates several data channels on top of an
abstracted SCTP data transport. As soon as a data channel is open, a message
will be sent to the other peer. Furthermore, another message will be sent on a
specific channel after a brief timeout.

To verify that a data channels opens, wait for the following line:

    (<client>) Data channel open: <channel-label>
    
The tool will send some large (16 MiB) test data to the other peer depending on
the ICE role. We are able to do this because RAWRTC handles data channel
messages correctly and does not have a maximum message size limitation compared
to most other implementations (check out
[this article][demystifying-webrtc-dc-size-limit] for a detailed explanation).

Usage:

    data-channel-sctp-loopback [<ice-candidate-type> ...]

### data-channel-sctp

API: ORTC

The data channel SCTP tool creates several data channels on top of an
abstracted SCTP data transport:

* A pre-negotiated data channel with the label `cat-noises` and the id `0`
  that is reliable and ordered. In the WebRTC JS API, the channel would be
  created by invoking:
   
   ```js
   peerConnection.createDataChannel('cat-noises', {
       ordered: true,
       id: 0
   });
   ```

* A data channel with the label `bear-noises` that is reliable but unordered.
  In the WebRTC JS API, the channel would be created by invoking:
   
   ```js
   peerConnection.createDataChannel('bear-noises', {
       ordered: false,
       maxRetransmits: 0
   });
   ```

To establish a connection with another peer, the following procecure must be
followed:

1. The JSON blob after `Local Parameters:` must be pasted into the other peer
   you want to establish a connection with. This can be either a browser
   instance that uses the 
   [WebRTC-ORTC browser example tool][webrtc-ortc-example] or another instance
   of this tool.

2. The other peer's local parameters in form of a JSON blob must be pasted into
   this tool's instance.

3. Once you've pasted the local parameters into each other's instance, the peer
   connection can be established by pressing *Enter* in both instances (press
   the *Start* button in the browser).

The tool will send some test data to the other peer depending on the ICE role.
However, the browser tool behaves a bit differently. Check the log output of
the tool instances (console output in the browser) to see what data has been
sent and whether it has been received successfully.

In the browser, you can use the created data channels by accessing
`peer.dc['<channel-name>']`, for example:

```js
peer.dc['example-channel'].send('RAWR!')
```

Usage:

    data-channel-sctp <0|1 (ice-role)> [<sctp-port>] [<ice-candidate-type> ...]

### data-channel-sctp-streamed

API: ORTC

The data channel SCTP streamed tool is the counterpart to the *normal* data
channel SCTP tool but uses the streaming mode. **Be aware this tool and the
streaming mode is currently experimental and incomplete.**

The necessary peer connection establishment steps are identical to the ones
described for the [data-channel-sctp](#data-channel-sctp) tool.

Usage:

    data-channel-sctp-streamed <0|1 (ice-role)> [<sctp-port>]
                               [<ice-candidate-type> ...]

### data-channel-sctp-echo

API: ORTC

The data channel SCTP echo tool behaves just like any other echo server: It
echoes received data on any data channel back to the sender.

The necessary peer connection establishment steps are identical to the ones
described for the [data-channel-sctp](#data-channel-sctp) tool.

Usage:

    data-channel-sctp-echo <0|1 (ice-role)> [<sctp-port>]
                           [<ice-candidate-type> ...]
    
### data-channel-sctp-throughput

API: ORTC

The data channel SCTP throughput tool allows you to test throughput by sending
one or more message. It will report the amount of seconds elapsed and the
throughput in Mbit/s.

The necessary peer connection establishment steps are identical to the ones
described for the [data-channel-sctp](#data-channel-sctp) tool. However,
be aware that this tool has no browser counterpart at the moment, so it only
makes sense to use two instances of this tool for throughput testing.

Usage:

    data-channel-sctp-throughput <0|1 (ice-role)> <message-size> [<n-times>]
                                 [<sctp-port>] [<ice-candidate-type> ...]

Special arguments:

* `message-size`: Is the message size in bytes used for throughput testing. The
  controlling peer will determine the message size for both peers, so this
  argument is being ignored for the controlled peer.
* `n-times`: Is the amount of times the message will be sent. Again, this
  value is being ignored for the controlled peer.

### peer-connection

API: WebRTC

The peer connection tool creates a peer connection instance and several data
channels:

* A pre-negotiated data channel with the label `cat-noises` and the id `0`
  that is reliable and ordered. In the JS API, the channel would be created
  by invoking:
   
   ```js
   peerConnection.createDataChannel('cat-noises', {
       ordered: true,
       id: 0
   });
   ```

* A data channel with the label `bear-noises` that is reliable but unordered.
  In the WebRTC JS API, the channel would be created by invoking:
   
   ```js
   peerConnection.createDataChannel('bear-noises', {
       ordered: true,
       maxRetransmits: 0
   });
   ```

To establish a connection with another peer, the following procecure must be
followed:

1. If the peer is taking the *offering* role, the generated JSON blob that
   contains the *offer SDP* must be pasted into the other peer you want to
   establish a connection with. This can be either a browser instance that uses
   the [WebRTC browser example tool][webrtc-example] or another instance of
   this tool. In case it is a browser instance, press the *Start* button and
   paste the data directly into the text area below
   `Paste remote description:`. In case it is another instance of this tool,
   paste the data into the other peer's console and press *Enter*.

2. The peer who takes the *answering* role now generates a JSON blob as well
   that contains the *answer SDP*. It must be pasted into the other browser
   instance or tool instance as described in the previous step.

3. The peer connection should be established automatically once *offer* and
   *answer* have been exchanged and applied.

The tool will send some test data to the other peer depending on whether or not
it took the *offering* role. However, the browser tool behaves a bit
differently. Check the log output of the tool instances (in the browser, either
open the console log or check out the live log on the right side) to see what
data has been sent and whether it has been received successfully.

In the browser, you can use the created data channels by accessing
`pc.dcs['<channel-name>']` in the console log, for example:

```js
pc.dcs['cat-noises'].send('RAWR!')
```

Usage:

    peer-connection <0|1 (offering)> [<ice-candidate-type> ...]

## Contributing

When creating a pull request, it is recommended to run `format-all.sh` to
apply a consistent code style.



[circleci-badge]: https://circleci.com/gh/rawrtc/rawrtc.svg?style=shield
[circleci-url]: https://circleci.com/gh/rawrtc/rawrtc
[travis-ci-badge]: https://travis-ci.org/rawrtc/rawrtc.svg?branch=master
[travis-ci-url]: https://travis-ci.org/rawrtc/rawrtc
[gitter]: https://gitter.im/rawrtc/Lobby
[gitter-icon]: https://badges.gitter.im/rawrtc/Lobby.svg

[ice]: https://tools.ietf.org/html/draft-ietf-ice-rfc5245bis-08
[trickle-ice]: https://tools.ietf.org/html/draft-ietf-ice-trickle-07
[stun]: https://tools.ietf.org/html/rfc5389
[turn]: https://tools.ietf.org/html/rfc5766
[stun-turn-dtls]: https://tools.ietf.org/html/rfc7350
[dcep]: https://tools.ietf.org/html/draft-ietf-rtcweb-data-protocol-09
[sctp-dc]: https://tools.ietf.org/html/draft-ietf-rtcweb-data-channel-13
[jsep]: https://tools.ietf.org/html/draft-ietf-rtcweb-jsep-19
[w3c-webrtc]: https://www.w3.org/TR/webrtc/
[w3c-ortc]: https://draft.ortc.org

[re-lock]: http://www.creytiv.com/doxygen/re-dox/html/re__main_8h.html#ad335fcaa56e36b39cb1192af1a6b9904
[re-mqueue]: http://www.creytiv.com/doxygen/re-dox/html/re__mqueue_8h.html
[rawrtcdc]: https://github.com/rawrtc/rawrtc-data-channel

[git]: (https://git-scm.com)
[meson]: https://mesonbuild.com
[ninja]: https://ninja-build.org

[webrtc-ortc-example]: https://rawgit.com/rawrtc/rawrtc/master/htdocs/ortc/index.html
[webrtc-example]: https://rawgit.com/rawrtc/rawrtc/master/htdocs/webrtc/index.html
[dctt]: https://github.com/nplab/dctt
[demystifying-webrtc-dc-size-limit]: https://lgrahl.de/articles/demystifying-webrtc-dc-size-limit.html
