
# API 

**JSONgle** exposes the following API

## Properties

The **JSONgle** object instance offers the following properties:

### currentCall -> Call

Get the current call or null

### id

Get the connected user id

### name -> string

The name of the library

### version -> string

The version of the library

### ticket -> Object

For each call done, a ticket is generated and can be retrieved through the getter `ticket` or by listening to the event `onticket`. The event is fired once the call has ended.

```js
// Got a ticket on a call in progress at any time
const ticket = jsongle.ticket;

jsongle.onticket = (ticket) => {
    //Get the generated ticket once the call has ended
};
```

## Methods

The **JSONgle** object instance offers the following methods:

### call(string: id, (optional)string: type = 'audio')

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

The method accepts a second optional parameter which is the media used. This is used to alert the recipient about the kind of call you want to initiate. If not provided, the default media used is `MEDIA.AUDIO`. There is no check with the media used in the call.

```js
jsongle.call(id, JSONGle.MEDIA.AUDIO);
```

### decline()

When call is ringing (`state` === `ringing`) and initiated from someone else (`direction` === `JSONgle.DIRECTION.INCOMING`), you have the possibility to decline it.

```js
jsongle.oncallended = (hasBeenInitiated) => {
    // Do something when the call has been ended
};

jsongle.decline();
```

A message will be sent to the initiator and the call will be ended (`state` === `ended`).

### proceed()

In the same manner, when the call is ringing (`state`=== `ringing`) and initiated from someone else (`direction` === `JSONgle.DIRECTION.INCOMING`), you have the possibility to proceed it which means you want to answer the call.

```js
jsongle.oncallstatechanged = (call) => {
    // Do something when the call state has changed
};

jsongle.proceed();
```

A message will be sent to the initiator and the call will move to state `proceeded` that will trigger the negotiation step one step further.

### sendOffer(object localDescription)

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

    await pc.createOffer();
    jsongle.sendOffer(pc.localDescription);
};
```

In the same way, when the remote recipient sends his SDP (his local description), **JSONGle** fires an event with that description in order for your application to give it to the WebRTC stack. Here is the minimum to do

```js
jsongle.onofferreceived = (remoteDescription) {
    pc.setRemoteDescription(remoteDescription);
}
```

### sendCandidate(object candidate)

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

### setAsActive()

In order to inform the recipient that everything is ok on your side, send a `session-info` message with a `reason=active`. This can be done by calling the method `setAsActive()` when the WebRTC call is established.

```js
pc.onconnectionstatechange = () => {
    if (pc.connectionState === "connected") {
        jsongle.setAsActive();
    }
};
```

In the same way, your application will receive a `session-info` with a `reason=active` from your recipient. This will trigger the event `oncallstatechanged`.

### mute() & unmute()

When the application mute or unmute the media, a `session-info`message with a `reason=mute` or `reason=unmute` can be sent to your recipient in order to inform him of the nature of the change.

```js
// When muting the audio stream
jsongle.mute()

// When unmuting the audio stream
jsongle.unmute()

json.onlocalcallmuted = (call) => {
    // Do something when the local stream is muted
    ...
}

json.onlocalcallunmuted = (call) => {
    // Do something when the local stream is unmuted
    ...
}
```

The recipient will receive an event in the same way to be informed

```js
json.oncallmuted = (call) => {
    // Do something when the remote stream is muted
    ...
}

json.oncallunmuted = (call) => {
    // Do something when the remote stream is unmuted
    ...
}
```

### end()

At anytime, an initiated call can be ended by the issuer or the responder. When the call not yet active, the issuer can **retract** it. When the call is active, both can **end** that call.

From the application point of view, only one method is provided that retracts or ends the call depending on its internal state.

```js
jsongle.oncallended = (hasBeenInitiated) => {
    // Do something when the call has been ended
};

// End or retract a call
jsongle.end();
```

### send(string: to, object: content)

At anytime, a custom message can be exchanged to specific recipient (identified by his id), a room (identified by a roomid) or to the server directly (identified by the 'sn' property received in the **session-hello** event) by using the **send** method.

**content** is an object that replaces the content of the `description` property in the JSONgle message.

```js
// Custom message
const msgData={
  presence: 'available',
};

jsongle.send('user_4', msgData);

//On 'user_4'
jsongle.ondatareceived = (content, from) => {
  // Do something with the content received 
  // content = { action: session-custom, description: { presence: 'available' } }
}
```

By listening to the event `ondatareceived`, the remote recipient is able to handle the content of that message.

### iq(string: to, string: query, object: content) -> Promise

At anytime, a request can be send to the server to execute an action (eg: registering to a room). For that, JSONgle offers the `iq` method that is a **Promise**.

**content** is an object representing the requested data to send.

**query** is the method that will be executed.

Once the server has performed the request, it sends back an asynchronous answer that is handled by that promise.

```js
// A request
const requestData = {
  rid: '188-4225-bf3a-d4ad43654697',
}

try {
  const result = await jsongle.iq('barracuda', 'session-register', requestData);
  // Do something in case of the request has been successfully executed
} catch(err) {
  // Do something in case the request failed
}
```