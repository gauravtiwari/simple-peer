# simple-peer [![travis](https://img.shields.io/travis/feross/simple-peer.svg?style=flat)](https://travis-ci.org/feross/simple-peer) [![npm](https://img.shields.io/npm/v/simple-peer.svg?style=flat)](https://npmjs.org/package/simple-peer) [![npm downloads](https://img.shields.io/npm/dm/simple-peer.svg?style=flat)](https://npmjs.org/package/simple-peer)

#### Simple WebRTC video/voice and data channels.

[![Sauce Test Status](https://saucelabs.com/browser-matrix/feross-simple-peer.svg)](https://saucelabs.com/u/feross-simple-peer)

## features

- concise, **node.js style** API for [WebRTC](https://en.wikipedia.org/wiki/WebRTC)
- **works in node and the browser!**
- supports **video/voice streams**
- supports **data channel**
  - text and binary data
  - node.js [duplex stream](http://nodejs.org/api/stream.html) interface
- supports advanced options like:
  - enable/disable [trickle ICE candidates](http://webrtchacks.com/trickle-ice/)
  - manually set config and constraints options

This module works in the browser with [browserify](http://browserify.org/).

**Note:** If you're **NOT** using browserify, then use the included standalone file
`simplepeer.min.js`. This exports a `SimplePeer` constructor on `window`.

## install

```
npm install simple-peer
```

## usage

Let's create an html page that let's you manually connect two peers:

```html
<html>
  <body>
    <style>
      #outgoing {
        width: 600px;
        word-wrap: break-word;
      }
    </style>
    <form>
      <textarea id="incoming"></textarea>
      <button type="submit">submit</button>
    </form>
    <pre id="outgoing"></pre>
    <script src="bundle.js"></script>
  </body>
</html>
```

```js
var Peer = require('simple-peer')
var p = new Peer({ initiator: location.hash === '#1', trickle: false })

p.on('error', function (err) { console.log('error', err) })

p.on('signal', function (data) {
  console.log('SIGNAL', JSON.stringify(data))
  document.querySelector('#outgoing').textContent = JSON.stringify(data)
})

document.querySelector('form').addEventListener('submit', function (ev) {
  ev.preventDefault()
  p.signal(JSON.parse(document.querySelector('#incoming').value))
})

p.on('connect', function () {
  console.log('CONNECT')
  p.send('whatever' + Math.random())
})

p.on('data', function (data) {
  console.log('data: ' + data)
})
```

Visit `index.html#1` from one browser (the initiator) and `index.html` from another
browser (the receiver).

An "offer" will be generated by the initiator. Paste this into the receiver's form and
hit submit. The receiver generates an "answer". Paste this into the initiator's form and
hit submit.

Now you have a direct P2P connection between two browsers!

### A simpler example

This example create two peers **in the web same page**.

In a real-world application, *you would never do this*. The sender and receiver `Peer`
instances would exist in separate browsers. A "signaling server" (usually implemented with
websockets) would be used to exchange signaling data between the two browsers until a
peer-to-peer connection is established.

### data channels

```js
var SimplePeer = require('simple-peer')

var peer1 = new SimplePeer({ initiator: true })
var peer2 = new SimplePeer()

peer1.on('signal', function (data) {
  // when peer1 has signaling data, give it to peer2 somehow
  peer2.signal(data)
})

peer2.on('signal', function (data) {
  // when peer2 has signaling data, give it to peer1 somehow
  peer1.signal(data)
})

peer1.on('connect', function () {
  // wait for 'connect' event before using the data channel
  peer1.send('hey peer2, how is it going?')
})

peer2.on('data', function (data) {
  // got a data channel message
  console.log('got a message from peer1: ' + data)
})
```

### video/voice

Video/voice is also super simple! In this example, peer1 sends video to peer2.

```js
var SimplePeer = require('simple-peer')

// get video/voice stream
navigator.getUserMedia({ video: true, audio: true }, gotMedia, function () {})

function gotMedia (stream) {
  var peer1 = new SimplePeer({ initiator: true, stream: stream })
  var peer2 = new SimplePeer()

  peer1.on('signal', function (data) {
    peer2.signal(data)
  })

  peer2.on('signal', function (data) {
    peer1.signal(data)
  })

  peer2.on('stream', function (stream) {
    // got remote video stream, now let's show it in a video tag
    var video = document.querySelector('video')
    video.src = window.URL.createObjectURL(stream)
    video.play()
  })
}
```

For two-way video, simply pass a `stream` option into both `Peer` constructors. Simple!

### in node

To use this library in node, pass in `opts.wrtc` as a parameter:

```js
var SimplePeer = require('simple-peer')
var wrtc = require('wrtc')

var peer1 = new SimplePeer({ initiator: true, wrtc: wrtc })
var peer2 = new SimplePeer({ wrtc: wrtc })
```

## production apps that use `simple-peer`

- [Friends](https://github.com/moose-team/friends) - P2P chat powered by the web
- [ScreenCat](https://maxogden.github.io/screencat/) - screen sharing + remote collaboration app
- [WebCat](https://github.com/mafintosh/webcat) - P2P pipe across the web using Github private/public key for auth
- [Instant.io](https://instant.io) - Secure, anonymous, streaming file transfer
- [WebTorrent](http://webtorrent.io) - Streaming torrent client in the browser
- [PusherTC](http://pushertc.herokuapp.com) - Video chat with using Pusher. See [guide](http://blog.carbonfive.com/2014/10/16/webrtc-made-simple/).
- [lxjs-chat](https://github.com/feross/lxjs-chat) - Omegle-like video chat site [demo]
- [socket.io-p2p](https://github.com/socketio/socket.io-p2p) - P2P communication via socket.io events
- *Your app here! - send a PR!*

## api

### `peer = new SimplePeer([opts])`

Create a new WebRTC peer connection.

A "data channel" for text/binary communication is always established, because it's cheap and often useful. For video/voice communication, pass the `stream` option.

If `opts` is specified, then the default options (shown below) will be overridden.

```
{
  initiator: false,
  channelConfig: {},
  channelName: '<random string>',
  config: { iceServers: [ { url: 'stun:23.21.150.121' } ] },
  constraints: {},
  reconnectTimer: 0,
  sdpTransform: function (sdp) { return sdp },
  stream: false,
  trickle: true,
  wrtc: {} // RTCPeerConnection/RTCSessionDescription/RTCIceCandidate
}
```

The options do the following:

- `initiator` - set to true if this is the initiating peer
- `channelConfig` - custom webrtc data channel configuration (used by `createDataChannel`)
- `channelName` - custom webrtc data channel name
- `config` - custom webrtc configuration (used by `RTCPeerConnection` constructor)
- `constraints` - custom webrtc video/voice constaints (used by `RTCPeerConnection` constructor)
- `reconnectTimer` - wait __ milliseconds after ICE 'disconnect' for reconnect attempt before emitting 'close'
- `sdpTransform` - function to transform the generated SDP signaling data (for advanced users)
- `stream` - if video/voice is desired, pass stream returned from `getUserMedia`
- `trickle` - set to `false` to disable [trickle ICE](http://webrtchacks.com/trickle-ice/) and get a single 'signal' event (slower)
- `wrtc` - custom webrtc implementation, mainly useful in node to specify in the [wrtc](https://npmjs.com/package/wrtc) package

### `peer.signal(data)`

Call this method whenever the remote peer emits a `peer.on('signal')` event.

The `data` will encapsulate a webrtc offer, answer, or ice candidate. These messages help
the peers to eventually establish a direct connection to each other. The contents of these
strings are an implementation detail that can be ignored by the user of this module;
simply pass the data from 'signal' events to the remote peer and call `peer.signal(data)`
to get connected.

### `peer.send(data)`

Send text/binary data to the remote peer. `data` can be any of several types: `String`,
`Buffer` (see [buffer](https://github.com/feross/buffer)), `TypedArrayView` (`Uint8Array`,
etc.), `ArrayBuffer`, or `Blob` (in browsers that support it).

Other data types will be transformed with `JSON.stringify` before sending. This is handy
for sending object literals across like this: `peer.send({ type: 'data', data: 'hi' })`.

Note: If this method is called before the `peer.on('connect')` event has fired, then data
will be buffered.

### `peer.destroy([onclose])`

Destroy and cleanup this peer connection.

If the optional `onclose` parameter is passed, then it will be registered as a listener on the 'close' event.

### `Peer.WEBRTC_SUPPORT`

Detect native WebRTC support in the javascript environment.

```js
var Peer = require('simple-peer')

if (Peer.WEBRTC_SUPPORT) {
  // webrtc support!
} else {
  // fallback
}
```

### duplex stream

`Peer` objects are instances of `stream.Duplex`. The behave very similarly to a
`net.Socket` from the node core `net` module. The duplex stream reads/writes to the data
channel.

```js
var peer = new Peer(opts)
// ... signaling ...
peer.write(new Buffer('hey'))
peer.on('data', function (chunk) {
  console.log('got a chunk', chunk)
})
```

## events


### `peer.on('signal', function (data) {})`

Fired when the peer wants to send signaling data to the remote peer.

**It is the responsibility of the application developer (that's you!) to get this data to
the other peer.** This usually entails using a websocket signaling server. This data is an
`Object`, so  remember to call `JSON.stringify(data)` to serialize it first. Then, simply
call `peer.signal(data)` on the remote peer.

### `peer.on('connect', function () {})`

Fired when the peer connection and data channel are ready to use.

### `peer.on('data', function (data) {})`

Received a message from the remote peer (via the data channel).

`data` will be either a `String` or a `Buffer/Uint8Array` (see [buffer](https://github.com/feross/buffer)).

### `peer.on('stream', function (stream) {})`

Received a remote video stream, which can be displayed in a video tag:

```js
peer.on('stream', function (stream) {
  var video = document.createElement('video')
  video.src = window.URL.createObjectURL(stream)
  document.body.appendChild(video)
  video.play()
})
```

### `peer.on('close', function () {})`

Called when the peer connection has closed.

### `peer.on('error', function (err) {})`

Fired when a fatal error occurs. Usually, this means bad signaling data was received from the remote peer.

`err` is an `Error` object.

## connecting more than 2 peers?

The simplest way to do that is to create a full-mesh topology. That means that every peer
opens a connection to every other peer. To illustrate:

![full mesh topology](img/full-mesh.png)

To broadcast a message, just iterate over all the peers and call `peer.send`.

So, say you have 3 peers. Then, when a peer wants to send some data it must send it 2
times, once to each of the other peers. So you're going to want to be a bit careful about
the size of the data you send.

Full mesh topologies don't scale well when the number of peers is very large. The total
number of edges in the network will be ![full mesh formula](img/full-mesh-formula.png)
where `n` is the number of peers.

For clarity, here is the code to connect 3 peers together:

#### Peer 1

```js
// These are peer1's connections to peer2 and peer3
var peer2 = new SimplePeer({ initiator: true })
var peer3 = new SimplePeer({ initiator: true })

peer2.on('signal', function (data) {
  // send this signaling data to peer2 somehow
})

peer2.on('connect', function () {
  peer2.send('hi peer2, this is peer1')
})

peer2.on('data', function (data) {
  console.log('got a message from peer2: ' + data)
})

peer3.on('signal', function (data) {
  // send this signaling data to peer3 somehow
})

peer3.on('connect', function () {
  peer3.send('hi peer3, this is peer1')
})

peer3.on('data', function (data) {
  console.log('got a message from peer3: ' + data)
})
```

#### Peer 2

```js
// These are peer2's connections to peer1 and peer3
var peer1 = new SimplePeer()
var peer3 = new SimplePeer({ initiator: true })

peer1.on('signal', function (data) {
  // send this signaling data to peer1 somehow
})

peer1.on('connect', function () {
  peer1.send('hi peer1, this is peer2')
})

peer1.on('data', function (data) {
  console.log('got a message from peer1: ' + data)
})

peer3.on('signal', function (data) {
  // send this signaling data to peer3 somehow
})

peer3.on('connect', function () {
  peer3.send('hi peer3, this is peer2')
})

peer3.on('data', function (data) {
  console.log('got a message from peer3: ' + data)
})
```

#### Peer 3

```js
// These are peer3's connections to peer1 and peer2
var peer1 = new SimplePeer()
var peer2 = new SimplePeer()

peer1.on('signal', function (data) {
  // send this signaling data to peer1 somehow
})

peer1.on('connect', function () {
  peer1.send('hi peer1, this is peer3')
})

peer1.on('data', function (data) {
  console.log('got a message from peer1: ' + data)
})

peer2.on('signal', function (data) {
  // send this signaling data to peer2 somehow
})

peer2.on('connect', function () {
  peer2.send('hi peer2, this is peer3')
})

peer2.on('data', function (data) {
  console.log('got a message from peer2: ' + data)
})
```

[![js-standard-style](https://raw.githubusercontent.com/feross/standard/master/badge.png)](https://github.com/feross/standard)

## license

MIT. Copyright (c) [Feross Aboukhadijeh](http://feross.org).
