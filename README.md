# JSONgle

## Introduction

**JSONgle** is a JavaScript library that proposes an agnostic transport implementation based and adapted from the Jingle signaling protocol for WebRTC calls using JSON messages.

Goal is that an application can use its existing server to exchange the signaling messages between peers but relies on **JSONgle** for the content of these messages.

The exemple provided here is using **Socket.io** as the transport layer. But an abstraction is done to map your existing transport layer to **JSONgle**.

## Signaling and WebRTC

WebRTC needs a signaling way to negotiate with the remote peer about the media and the best path to follow.

In fact, only few information need to be exchanged: a **SDP** and some **ICE Candidates**.

So what **JSONgle** does is to ask for a local SDP in one side and its associated candidates and send them to the remote peer and by asking to that remote peer in a same maner his local description and some candidates that are given back to the initial sender. And that all!

Additionnaly to that, **JSONgle** computes internally a **Call State** machine that can be retrieved throught some events and generate at the end of the communication a **log ticket** that summarizes the call progress and information.

## WebRTC Adapter

Don't forget to install and use [**WebRTC adapter**](https://github.com/webrtcHacks/adapter) in order to help you managing WebRTC on different browsers (JavaScript API).

## Configuration

In order to adapt **JSONgle** to your own server, you need to do some configuration.

### Defining the transport layer

The first thing you have to do is to define your transport wrapper by using the following pattern:

```js
const transportWrapper = (transport) => {
    return {
        in: (callback) => {
            transport.on("jsongle", (msg) => {
                callback(msg);
            });
        },
        out: (msg) => {
            transport.emit("jsongle", msg);
        },
    };
};
```

The transport wrapper is in fact a function that embeds your own transport (here socket.io) and that returns an object with 2 properties `in` and `out`:

-   **in**: This property which is a function is used when receiving a message from your transport layer to give it back to **JSONgle** when it should be. Here, we listen to the event name `jsongle` and we execute the callback given with the message received.

-   **out**: This property which is a function too is used when **JSONgle** needs to send a message using your transport layer. This function is called by **JSONgle** with the message to send as an argument. Just taken the message and send it using your transport layer.

In that previous sample, the `transport` parameter is in fact an instance of **Socket.IO**. `transport.on` and `transport.emit` are function from **Socket.IO**.

_Note_: If your transport layer allows to use custom event name, it is better to send all the **JSONgle** messages in a separate queue to avoid mixing them with your own events.

Once your wrapper has been defined, you can configure your transport.

```js
const io = socketio("<your_host>");

const transportCfg = {
    name: "socket.io",
    transport: transportWrapper(io),
};
```

### Defining the user identity

As now, the identity of the user is used to allow the recipient to identify the caller.

You can use any kinds of unique `id` such as the user database identifier. It will be up to your application to identify that user from your database.

```js
const peerCfg = {
    id: "43eed341123123",
};
```

_Note_: This `id` is used by JSONgle when generating messages. All messages will have a `from` and `to` field that will contain the `id` of the caller and the callee.

### Initialize JSONgle

Once the configurations are ok, you can initialize **JSONgle**

```js
const jsongle = new JSONGle({
    transport: transportCfg,
    peer: peerCfg,
});
```

## API Methods

### Call

This method calls a user.

```js
const jsongle = new JSONGle({...});

jsongle.oncall = (call) => {
    // Do something when the call has been initiated
};

// Initiate a new audio call
jsongle.call(id, JSONGle.MEDIA.AUDIO);
```

The mandatory parameter is the identifier of the recipient. Depending on how your server dispatch the message it can be the user id or any information that allows to contact the right recipient. This information will be used to fill the field `to` in the message sent. The `from` will contain your id as defined in the user identity paragraph.

The method accepts a second optional parameter which is the media used. This is used to alert the recipient about the kind of call you want to initiate. If not provided, the default media used is `MEDIA.AUDIO`.

```js
jsongle.call(id, JSONGle.MEDIA.AUDIO);
```

### Decline

When call is ringing (`state` === `ringing`) and initiated from someone else (`direction` === `JSONgle.DIRECTION.INCOMING`), you have the possibility to decline it.

```js
jsongle.oncallended = (hasBeenInitiated) => {
    // Do something when the call has been ended
};

jsongle.decline();
```

A message will be sent to the initiator and the call will be ended (`state` === `ended`).

### Proceed

In the same manner, when the call is ringing (`state`=== `ringing`) and initiated from someone else (`direction` === `JSONgle.DIRECTION.INCOMING`), you have the possibility to proceed it which means you want to answer the call.

```js
jsongle.oncallstatechanged = (call) => {
    // Do something when the call state has changed
};

jsongle.proceed();
```

A message will be sent to the initiator and the call will move to state `proceeded` that will trigger the negotiation step one step further.

### Send and receive offer

If the call has been proceeded, the WebRTC negotiation starts and the **JSONgle** library will send the event `onofferneeded` when it needs the local SDP to send it to the remote peer.

Once you have to do, is to obtain that SDP (aka **local description**) from your WebRTC stack and to give it to **JSONgle** by using the method `sendOffer` as follow

```js
// Your Peer Connection with the correct configuration
const pc = new RTCPeerConnection({...});

// Your camera/mic constraints
const constraints = {...};

jsongle.onofferneeded = async (call) => {
    // Got the local stream from your camera/mic
    const stream = await navigator.mediaDevices.getUserMedia(constraints);

    for (const track of stream.getTracks()) {
        pc.addTrack(track, stream);
    }

    const localDescription = await pc.setLocalDescription();
    jsongle.sendOffer(localDescription);
};
```

In the same way, when the remote recipient sends his SDP (his local description), **JSONGle** fires an event with that description in order for your application to give it to the WebRTC stack. Here is the minimum to do

```js

jsongle.onofferreceived = (remoteDescription) {
    pc.setRemoteDescription(remoteDescription);
}

```

### Send and receive ICE candidates

ICE candidates should be exchanged the same way between the two peers.

The first part of the job is when the `RTCPeerConnection` generates new ICE candidates, you need to give them to **JSONgle** by calling the method `sendCandidate` by doing something like that:

```js
pc.onicecandidate = ({ candidate }) => {
    if (candidate) {
        jsongle.sendCandidate(candidate);
    }
};
```

**JSONgle** will send each ICE candidate to the remote peer.

In the opposite, when the remote peer sends to you an ICE candidate, you need to listen to the event `oncandidatereceived` to get that candidate and give it to the WebRTC stack like that:

```js
jsongle.oncandidatereceived = async (candidate) => {
    await pc.addIceCandidate(candidate);
};
```

### Set call as active

In order to inform the recipient that everything is ok on your side, you have to send a `session-info` message with a `reason=active`. This can be done by calling the method `setAsActive()` when the WebRTC call is established.

```js
pc.onconnectionstatechange = () => {
    if (pc.connectionState === "connected") {
        jsongle.setAsActive();
    }
};
```

In the same way, your application will receive a `session-info` with a `reason=active` from your recipient. This will trigger the event `oncallstatechanged`.

### End

At anytime, an initiated call can be ended by the issuer or the responder. When the call not yet active, the issuer can **retract** it. When the call is active, both can **end** that call.

From the application point of view, only one method is provided that retracts or ends the call depending on its internal state.

```js
jsongle.oncallended = (hasBeenInitiated) => {
    // Do something when the call has been ended
};

// End or retract a call
jsongle.end();
```

### Ticket

For each call done, a ticket is generated and can be retrieved through the getter `ticket` or by listening to the event `onticket`. The event is fired once the call has ended.

```js
// Got a ticket on a call in progress at any time
const ticket = jsongle.ticket;

jsongle.onticket = (ticket) => {
    //Get the generated ticket once the call has ended
};
```

## API Events

You can subscribe to the following events on the **JSONgle** instance

| Events                | Description                                                                                                                                                                                                                                 |
| :-------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `oncall`              | Fired when a new call has been received or when a call is initiated.<br>The event contains the `Call`                                                                                                                                       |
| `oncallstatechanged`  | Fired each time there is an update on the current call.<br>The event contains the `Call`                                                                                                                                                    |
| `oncallended`         | Fired when a call has ended.<br>The event contains a boolean indicating if the call has been ended locally (`true`) or from the remote peer (`false`)                                                                                       |
| `onofferneeded`       | Fired when a call needs a SDP offer.<br>The event contains the `Call`<br>The application should get the local description (SDP) and answer as soon as possible by calling the method `sendOffer` with the offer generated from the browser. |
| `onofferreceived`     | Fired when a call received a SDP offer.<br>The event contains the `RTCSessionDescription` received from the recipient.<br>The application should give that offer to the `RTCPeerConnection`.                                                |
| `oncandidatereceived` | Fired when a call received an ICE candidate.<br>The event contains the `RTCIceCandidate` received from the recipient.<br>The application should give that candidate to the `RTCPeerConnection`.                                             |
| `onticket`            | Fired when the call has ended.<br>The event contains a sum-up of all call information.                                                                                                                                                      |

Here is an exemple of registering to an event

```js
jsongle.oncallstatechanged = (call) => {
    // The call state has changed. Do something if needed
};
```

## Server side

You can send additionnal messages from your server to the issuer for handling specific cases

### Unreachable recipient

When the issuer sent the **session-propose** message, once received, the server could check if the recipient exists and is connected.

If the recipient can't be reachable, a **session-info** message with `reason=unreachable` could be sent to the issuer to inform him that the call can't be proceeded and to change the call information to the `state=ended` with `reason=unreachable`.

See paragraph **Messages exchanged** to have the description of the message to send.

```js
// Example using socket-io on server side
socket.on("jsongle", (message) => {
    // Check that the recipient exists and is connected
    if (!(message.to in users)) {
        const abortMsg = {
            id: "<your_random_message_id>",
            from: "server",
            to: message.from,
            jsongle: {
                sid: message.jsongle.sid,
                action: "session-info",
                reason: "unreachable",
                initiator: message.jsongle.initiator,
                responder: message.jsongle.responder,
                description: {
                    ended: new Date().toJSON(),
                },
            },
        };

        // Send a try to issuer
        socket.emit("jsongle", abortMsg);
        return;
    }
});
```

### Trying

When the issuer sent the **session-propose** message and once the server has found the recipient, the server could send a **session-info** message with `reason=trying` to the issuer to inform him that the call is routing to the recipient.

See paragraph **Messages exchanged** to have the description of the message to send.

```js
// Example using socket-io on server side
socket.on("jsongle", (message) => {
    // Check the recipient and the message
    if (message.to in users && message.jsongle.action === "session-propose") {
        const abortMsg = {
            id: "<your_random_message_id>",
            from: "server",
            to: message.from,
            jsongle: {
                sid: message.jsongle.sid,
                action: "session-info",
                reason: "trying",
                initiator: message.jsongle.initiator,
                responder: message.jsongle.responder,
                description: {
                    tried: new Date().toJSON(),
                },
            },
        };

        // Send a try to issuer
        socket.emit("jsongle", abortMsg);
        return;
    }
});
```

## Call State

A `Call` can have the following states:

| **State**   | **Description**                                                                                                      | **Reason**                                                       |
| :---------- | :------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------- |
| `new`       | Call has just been created                                                                                           |                                                                  |
| `trying`    | Call has been received by the server and is being routed to the remote recipient.<br>Only for the issuer of the call |                                                                  |
| `ringing`   | Call has been received by the remote peer and is being presented<br>Only for the issuer                              |                                                                  |
| `proceeded` | Call has been accepted by the responder                                                                              |                                                                  |
| `offering`  | Call has been accepted by the remote peer and is being negotiated                                                    | `have-offer`<br>`have-answer`<br>`have-both`                     |
| `active`    | Call is active                                                                                                       | `is-active-local`<br>`is-active-remote`<br>`is-active-both-side` |
| `releasing` | Call is releasing by a peer                                                                                          |                                                                  |
| `ended`     | Call is ended                                                                                                        | `retracted`<br>`declined`<br>`terminated`<br>`unreachable`       |

### Call lifecycle from the caller point of view

On the caller side, the `Call` has the following cycle:

`new` -> `trying` -> `ringing` -> `proceeded` -> `offering` -> `active` -> `releasing` -> `ended`

_Note_: From any state, the `Call` state can move to `ended`.

### Call lifecycle from the callee point of view

On the callee side, the `Call` has the following cycle:

`ringing` -> `proceeded` -> `offering` -> `active` -> `releasing` -> `ended`

_Note_: From any state, the `Call` state can move to `ended`.

## Messages exchanged

This part lists the different messages exchanged during the session lifecycle. This could help you plugging **JSONgle** to your existing transport layer.

Each message is composed of:

-   An **id** which is the unique identifier of the message
-   2 fields: **from** and **to** which represent the caller and the caller. This is where you will map your existing user identifier.
-   A **jsongle** object which contain the **JSONgle** grammar (aka the different kinds of messages)

### session-propose

The **session-propose** message is sent by the issuer to propose a session (a call) to a recipient.

```json
{
    "id": "20229102-7f9f-4ef4-87d8-481dd6ef5f85",
    "from": "70001",
    "to": "70002",
    "jsongle": {
        "sid": "3bf74aa9-f41d-40d5-a1d8-e3e614ba4af2",
        "action": "session-propose",
        "reason": "",
        "initiator": "70001",
        "responder": "70002",
        "description": {
            "initiated": "2020-09-05T19:31:34.186Z",
            "media": "audio"
        }
    }
}
```

### session-info - unreachable

When a call is initiated to a remote peer, the server could answer to the initiator by a message of type **session-info** containing a `reason=unreachable` in order to inform the issuer that the recipient can't be joined.

```json
{
    "id": "001ad89f-43f4-423f-9055-e24813e9c82a",
    "from": "server",
    "to": "70001",
    "jsongle": {
        "sid": "23dfe5aa-3e13-4203-8650-3a11bec2d373",
        "action": "session-info",
        "reason": "unreachable",
        "initiator": "70001",
        "responder": "12334",
        "description": { "ended": "2020-09-15T13:36:31.151Z" }
    }
}
```

### session-info - trying

When a call is initiated to a remote peer, the server could answer to the initiator by a message of type **session-info** containing a `reason=trying` in order to inform the issuer that his call has been successfully handled and is 'routing'.

```json
{
    "id": "3fab1209-fb00-494f-82e4-855185a8cba6",
    "from": "server",
    "to": "70001",
    "jsongle": {
        "sid": "678403f4-7b1f-4ea5-84cb-c6699a91db22",
        "action": "session-info",
        "reason": "trying",
        "initiator": "70001",
        "responder": "70002",
        "description": {
            "tried": "2020-09-10T17:50:26.058Z"
        }
    }
}
```

_Note_: For that specific message, the issuer is the server, not the remote peer.

### session-info - ringing

When the responder receives a **session-propose** message, he starts by answering an acknowledgment to the issuer. For doing that, he sends a message of type **session-info** with a `reason=ringing` in order to inform the issuer that the call has been successfully received and is ringing.

```json
{
    "id": "fdc216f8-3d73-4865-a3c7-b43e1f5338a3",
    "from": "70002",
    "to": "70001",
    "jsongle": {
        "sid": "678403f4-7b1f-4ea5-84cb-c6699a91db22",
        "action": "session-info",
        "reason": "ringing",
        "initiator": "70001",
        "responder": "70002",
        "description": { "rang": "2020-09-10T17:50:26.061Z" }
    }
}
```

### session-retract

The **session-retract** message is sent by the issuer when he wants to cancel the call in progress. This message is only sent if the call is not active. Elsewhere a **session-terminate** message is sent.

```json
{
    "id": "23cf5699-b746-4e21-8698-01e499b946b7",
    "from": "70001",
    "to": "70002",
    "jsongle": {
        "sid": "678403f4-7b1f-4ea5-84cb-c6699a91db22",
        "action": "session-retract",
        "reason": "",
        "initiator": "70001",
        "responder": "70002",
        "description": { "ended": "2020-09-10T17:50:38.071Z" }
    }
}
```

### session-decline

The **session-decline** message is sent by the responder when he wants to decline the incoming call in progress. If the call is active, a **session-terminate** is sent.

```json
{
    "id": "29842334-91b8-4218-ac72-2fee950cf253",
    "from": "70002",
    "to": "70001",
    "jsongle": {
        "sid": "f0f3e269-e887-4d13-b140-3378357e9660",
        "action": "session-decline",
        "reason": "",
        "initiator": "70001",
        "responder": "70002",
        "description": { "ended": "2020-09-14T19:22:48.903Z" }
    }
}
```

### session-proceed

The **session-proceed** message is sent by the responder when he wants to proceed the call.

```json
{
    "id": "9432a7f1-4128-417c-a0da-5b5e8e32623f",
    "from": "70002",
    "to": "70001",
    "jsongle": {
        "sid": "132a7f2b-8188-4225-bf3a-d4ad43654697",
        "action": "session-proceed",
        "reason": "",
        "initiator": "70001",
        "responder": "70002",
        "description": { "proceeded": "2020-09-14T19:25:51.864Z" }
    }
}
```

### session-initiate

The **session-initiate** message is sent by the issuer when he needs to exchange his local description (aka SDP) with the responder.

```json
{
    "id": "434d910c-9f26-4d1e-b8c7-ee16df9da003",
    "from": "70001",
    "to": "70002",
    "jsongle": {
        "sid": "132a7f2b-8188-4225-bf3a-d4ad43654697",
        "action": "session-initiate",
        "reason": "",
        "initiator": "70001",
        "responder": "70002",
        "description": {
            "offering": "2020-09-14T19:25:52.592Z",
            "offer": {
                "type": "offer",
                "sdp": "v=0\r\no=- 4129577743034933870 2 IN IP4 127.0.0.1\r\ns=-\r\nt=0 0\r\na=group:BUNDLE 0 1\r\na=msid-semantic: WMS dezF0xpAJKoFtfz2NsIqMtHikli1ed5O62UV\r\nm=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 110 112 113 126\r\nc=IN IP4 0.0.0.0\r\na=rtcp:9 IN IP4 0.0.0.0\r\na=ice-ufrag:CSM2\r\na=ice-pwd:aKcQpesqTSxTLLSQGAVUrL4V\r\na=ice-options:trickle\r\na=fingerprint:sha-256 CD:94:F5:ED:54:20:1C:B0:D5:12:31:AF:1A:31:60:88:A5:B0:1E:E3:3C:69:13:C0:3D:50:21:B1:C7:56:BE:07\r\na=setup:actpass\r\na=mid:0\r\na=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level\r\na=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time\r\na=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01\r\na=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid\r\na=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id\r\na=extmap:6 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id\r\na=sendrecv\r\na=msid:dezF0xpAJKoFtfz2NsIqMtHikli1ed5O62UV bd6611d0-9852-4001-b262-79e35a89e5c3\r\na=rtcp-mux\r\na=rtpmap:111 opus/48000/2\r\na=rtcp-fb:111 transport-cc\r\na=fmtp:111 minptime=10;useinbandfec=1\r\na=rtpmap:103 ISAC/16000\r\na=rtpmap:104 ISAC/32000\r\na=rtpmap:9 G722/8000\r\na=rtpmap:0 PCMU/8000\r\na=rtpmap:8 PCMA/8000\r\na=rtpmap:106 CN/32000\r\na=rtpmap:105 CN/16000\r\na=rtpmap:13 CN/8000\r\na=rtpmap:110 telephone-event/48000\r\na=rtpmap:112 telephone-event/32000\r\na=rtpmap:113 telephone-event/16000\r\na=rtpmap:126 telephone-event/8000\r\na=ssrc:2689474596 cname:VviiJ7DRjvl0T9bG\r\na=ssrc:2689474596 msid:dezF0xpAJKoFtfz2NsIqMtHikli1ed5O62UV bd6611d0-9852-4001-b262-79e35a89e5c3\r\na=ssrc:2689474596 mslabel:dezF0xpAJKoFtfz2NsIqMtHikli1ed5O62UV\r\na=ssrc:2689474596 label:bd6611d0-9852-4001-b262-79e35a89e5c3\r\nm=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 102 121 127 120 125 107 108 109 124 119 123 118 114 115 116\r\nc=IN IP4 0.0.0.0\r\na=rtcp:9 IN IP4 0.0.0.0\r\na=ice-ufrag:CSM2\r\na=ice-pwd:aKcQpesqTSxTLLSQGAVUrL4V\r\na=ice-options:trickle\r\na=fingerprint:sha-256 CD:94:F5:ED:54:20:1C:B0:D5:12:31:AF:1A:31:60:88:A5:B0:1E:E3:3C:69:13:C0:3D:50:21:B1:C7:56:BE:07\r\na=setup:actpass\r\na=mid:1\r\na=extmap:14 urn:ietf:params:rtp-hdrext:toffset\r\na=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time\r\na=extmap:13 urn:3gpp:video-orientation\r\na=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01\r\na=extmap:12 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay\r\na=extmap:11 http://www.webrtc.org/experiments/rtp-hdrext/video-content-type\r\na=extmap:7 http://www.webrtc.org/experiments/rtp-hdrext/video-timing\r\na=extmap:8 http://www.webrtc.org/experiments/rtp-hdrext/color-space\r\na=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid\r\na=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id\r\na=extmap:6 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id\r\na=sendrecv\r\na=msid:dezF0xpAJKoFtfz2NsIqMtHikli1ed5O62UV 6a0c428a-336f-4fd5-a67b-92dd9308da30\r\na=rtcp-mux\r\na=rtcp-rsize\r\na=rtpmap:96 VP8/90000\r\na=rtcp-fb:96 goog-remb\r\na=rtcp-fb:96 transport-cc\r\na=rtcp-fb:96 ccm fir\r\na=rtcp-fb:96 nack\r\na=rtcp-fb:96 nack pli\r\na=rtpmap:97 rtx/90000\r\na=fmtp:97 apt=96\r\na=rtpmap:98 VP9/90000\r\na=rtcp-fb:98 goog-remb\r\na=rtcp-fb:98 transport-cc\r\na=rtcp-fb:98 ccm fir\r\na=rtcp-fb:98 nack\r\na=rtcp-fb:98 nack pli\r\na=fmtp:98 profile-id=0\r\na=rtpmap:99 rtx/90000\r\na=fmtp:99 apt=98\r\na=rtpmap:100 VP9/90000\r\na=rtcp-fb:100 goog-remb\r\na=rtcp-fb:100 transport-cc\r\na=rtcp-fb:100 ccm fir\r\na=rtcp-fb:100 nack\r\na=rtcp-fb:100 nack pli\r\na=fmtp:100 profile-id=2\r\na=rtpmap:101 rtx/90000\r\na=fmtp:101 apt=100\r\na=rtpmap:102 H264/90000\r\na=rtcp-fb:102 goog-remb\r\na=rtcp-fb:102 transport-cc\r\na=rtcp-fb:102 ccm fir\r\na=rtcp-fb:102 nack\r\na=rtcp-fb:102 nack pli\r\na=fmtp:102 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42001f\r\na=rtpmap:121 rtx/90000\r\na=fmtp:121 apt=102\r\na=rtpmap:127 H264/90000\r\na=rtcp-fb:127 goog-remb\r\na=rtcp-fb:127 transport-cc\r\na=rtcp-fb:127 ccm fir\r\na=rtcp-fb:127 nack\r\na=rtcp-fb:127 nack pli\r\na=fmtp:127 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=42001f\r\na=rtpmap:120 rtx/90000\r\na=fmtp:120 apt=127\r\na=rtpmap:125 H264/90000\r\na=rtcp-fb:125 goog-remb\r\na=rtcp-fb:125 transport-cc\r\na=rtcp-fb:125 ccm fir\r\na=rtcp-fb:125 nack\r\na=rtcp-fb:125 nack pli\r\na=fmtp:125 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=42e01f\r\na=rtpmap:107 rtx/90000\r\na=fmtp:107 apt=125\r\na=rtpmap:108 H264/90000\r\na=rtcp-fb:108 goog-remb\r\na=rtcp-fb:108 transport-cc\r\na=rtcp-fb:108 ccm fir\r\na=rtcp-fb:108 nack\r\na=rtcp-fb:108 nack pli\r\na=fmtp:108 level-asymmetry-allowed=1;packetization-mode=0;profile-level-id=42e01f\r\na=rtpmap:109 rtx/90000\r\na=fmtp:109 apt=108\r\na=rtpmap:124 H264/90000\r\na=rtcp-fb:124 goog-remb\r\na=rtcp-fb:124 transport-cc\r\na=rtcp-fb:124 ccm fir\r\na=rtcp-fb:124 nack\r\na=rtcp-fb:124 nack pli\r\na=fmtp:124 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=4d0032\r\na=rtpmap:119 rtx/90000\r\na=fmtp:119 apt=124\r\na=rtpmap:123 H264/90000\r\na=rtcp-fb:123 goog-remb\r\na=rtcp-fb:123 transport-cc\r\na=rtcp-fb:123 ccm fir\r\na=rtcp-fb:123 nack\r\na=rtcp-fb:123 nack pli\r\na=fmtp:123 level-asymmetry-allowed=1;packetization-mode=1;profile-level-id=640032\r\na=rtpmap:118 rtx/90000\r\na=fmtp:118 apt=123\r\na=rtpmap:114 red/90000\r\na=rtpmap:115 rtx/90000\r\na=fmtp:115 apt=114\r\na=rtpmap:116 ulpfec/90000\r\na=ssrc-group:FID 1184807374 3691452921\r\na=ssrc:1184807374 cname:VviiJ7DRjvl0T9bG\r\na=ssrc:1184807374 msid:dezF0xpAJKoFtfz2NsIqMtHikli1ed5O62UV 6a0c428a-336f-4fd5-a67b-92dd9308da30\r\na=ssrc:1184807374 mslabel:dezF0xpAJKoFtfz2NsIqMtHikli1ed5O62UV\r\na=ssrc:1184807374 label:6a0c428a-336f-4fd5-a67b-92dd9308da30\r\na=ssrc:3691452921 cname:VviiJ7DRjvl0T9bG\r\na=ssrc:3691452921 msid:dezF0xpAJKoFtfz2NsIqMtHikli1ed5O62UV 6a0c428a-336f-4fd5-a67b-92dd9308da30\r\na=ssrc:3691452921 mslabel:dezF0xpAJKoFtfz2NsIqMtHikli1ed5O62UV\r\na=ssrc:3691452921 label:6a0c428a-336f-4fd5-a67b-92dd9308da30\r\n"
            }
        }
    }
}
```

### session-accept

The **session-accept** message is sent by the responder when he needs to exchange his local description (aka SDP) to the issuer in response to the **session-initiate** message.

```json
{
    "id": "1c778346-4a95-42a1-bff9-044e2c6ad584",
    "from": "70002",
    "to": "70001",
    "jsongle": {
        "sid": "132a7f2b-8188-4225-bf3a-d4ad43654697",
        "action": "session-accept",
        "reason": "",
        "initiator": "70001",
        "responder": "70002",
        "description": {
            "offered": "2020-09-14T19:26:31.315Z",
            "answer": {
                "type": "answer",
                "sdp": "v=0\r\no=mozilla...THIS_IS_SDPARTA-80.0.1 1830613524268934482 0 IN IP4 0.0.0.0\r\ns=-\r\nt=0 0\r\na=sendrecv\r\na=fingerprint:sha-256 7E:60:7C:F2:8D:F0:12:28:00:17:47:40:D5:84:BE:8E:E4:ED:0A:15:AB:3F:34:61:64:41:10:E2:5B:B4:4E:E8\r\na=group:BUNDLE 0 1\r\na=ice-options:trickle\r\na=msid-semantic:WMS *\r\nm=audio 9 UDP/TLS/RTP/SAVPF 111 9 0 8 126\r\nc=IN IP4 0.0.0.0\r\na=sendrecv\r\na=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level\r\na=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid\r\na=fmtp:111 maxplaybackrate=48000;stereo=1;useinbandfec=1\r\na=fmtp:126 0-15\r\na=ice-pwd:eb0c3e027c2b3971ee8c15bf18e335a6\r\na=ice-ufrag:08a2a3c7\r\na=mid:0\r\na=msid:{9d7a16b7-92c7-db45-b3e9-2cedcd22e982} {03fc9dcf-7bc7-ac44-8ca3-fa2e0b5078f2}\r\na=rtcp-mux\r\na=rtpmap:111 opus/48000/2\r\na=rtpmap:9 G722/8000/1\r\na=rtpmap:0 PCMU/8000\r\na=rtpmap:8 PCMA/8000\r\na=rtpmap:126 telephone-event/8000\r\na=setup:active\r\na=ssrc:295854526 cname:{7622cfd6-ed87-5149-a19f-39b5595ec332}\r\nm=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 125 107 108 109\r\nc=IN IP4 0.0.0.0\r\na=sendrecv\r\na=extmap:14 urn:ietf:params:rtp-hdrext:toffset\r\na=extmap:2 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time\r\na=extmap:3 http://www.ietf.org/id/draft-holmer-rmcat-transport-wide-cc-extensions-01\r\na=extmap:12 http://www.webrtc.org/experiments/rtp-hdrext/playout-delay\r\na=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid\r\na=fmtp:125 profile-level-id=42e01f;level-asymmetry-allowed=1;packetization-mode=1\r\na=fmtp:108 profile-level-id=42e01f;level-asymmetry-allowed=1\r\na=fmtp:96 max-fs=12288;max-fr=60\r\na=fmtp:97 apt=96\r\na=fmtp:98 max-fs=12288;max-fr=60\r\na=fmtp:99 apt=98\r\na=fmtp:107 apt=125\r\na=fmtp:109 apt=108\r\na=ice-pwd:eb0c3e027c2b3971ee8c15bf18e335a6\r\na=ice-ufrag:08a2a3c7\r\na=mid:1\r\na=msid:{9d7a16b7-92c7-db45-b3e9-2cedcd22e982} {eceb2a57-cb1e-d846-bb47-490f75cf8f4f}\r\na=rtcp-fb:96 nack\r\na=rtcp-fb:96 nack pli\r\na=rtcp-fb:96 ccm fir\r\na=rtcp-fb:96 goog-remb\r\na=rtcp-fb:96 transport-cc\r\na=rtcp-fb:98 nack\r\na=rtcp-fb:98 nack pli\r\na=rtcp-fb:98 ccm fir\r\na=rtcp-fb:98 goog-remb\r\na=rtcp-fb:98 transport-cc\r\na=rtcp-fb:125 nack\r\na=rtcp-fb:125 nack pli\r\na=rtcp-fb:125 ccm fir\r\na=rtcp-fb:125 goog-remb\r\na=rtcp-fb:125 transport-cc\r\na=rtcp-fb:108 nack\r\na=rtcp-fb:108 nack pli\r\na=rtcp-fb:108 ccm fir\r\na=rtcp-fb:108 goog-remb\r\na=rtcp-fb:108 transport-cc\r\na=rtcp-mux\r\na=rtpmap:96 VP8/90000\r\na=rtpmap:97 rtx/90000\r\na=rtpmap:98 VP9/90000\r\na=rtpmap:99 rtx/90000\r\na=rtpmap:125 H264/90000\r\na=rtpmap:107 rtx/90000\r\na=rtpmap:108 H264/90000\r\na=rtpmap:109 rtx/90000\r\na=setup:active\r\na=ssrc:1230489151 cname:{7622cfd6-ed87-5149-a19f-39b5595ec332}\r\na=ssrc:1348497150 cname:{7622cfd6-ed87-5149-a19f-39b5595ec332}\r\na=ssrc-group:FID 1230489151 1348497150\r\n"
            }
        }
    }
}
```

### transport-info

The **transport-info** messages are sent by the issuer and the responder when they want to exchange their ICE candidates

```json
{
    "id": "e1178f09-bc81-4b0f-8979-4fbdc0f223da",
    "from": "70001",
    "to": "70002",
    "jsongle": {
        "sid": "132a7f2b-8188-4225-bf3a-d4ad43654697",
        "action": "transport-info",
        "reason": "",
        "initiator": "70001",
        "responder": "70002",
        "description": {
            "establishing": "2020-09-14T19:25:52.611Z",
            "candidate": {
                "candidate": "candidate:2960972664 1 tcp 1518214911 192.168.1.8 9 typ host tcptype active generation 0 ufrag CSM2 network-id 1 network-cost 10",
                "sdpMid": "1",
                "sdpMLineIndex": 1
            }
        }
    }
}
```

### session-info - active

The **session-info** message with a `reason=active` is sent by the issuer and the responder when they detect that the call is active on their side (aka when the remote media is established).

```json
{
    "id": "1f9aed7c-dd11-42dd-bfe2-87860a5193cd",
    "from": "70002",
    "to": "70001",
    "jsongle": {
        "sid": "132a7f2b-8188-4225-bf3a-d4ad43654697",
        "action": "session-info",
        "reason": "active",
        "initiator": "70001",
        "responder": "70002",
        "description": { "actived": "2020-09-14T19:26:31.560Z" }
    }
}
```

### session-terminate

The **session-terminate** message is sent by the issuer or the responder when they want to terminate the call.

```json
{
    "id": "0d424e84-c3f0-48c4-85e4-1dd5a1922892",
    "from": "70001",
    "to": "70002",
    "jsongle": {
        "sid": "132a7f2b-8188-4225-bf3a-d4ad43654697",
        "action": "session-terminate",
        "reason": "",
        "initiator": "70001",
        "responder": "70002",
        "description": { "ended": "2020-09-14T19:41:35.124Z" }
    }
}
```
