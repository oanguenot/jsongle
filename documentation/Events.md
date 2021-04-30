# Events

**JSONgle** fires the following events


| Events                | Description                                                                                                                                                                                                                                 |
| :-------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `oncall`              | Fired when a new call has been received or when a call is initiated.<br>The event contains the `Call`                                                                                                                                       |
| `oncallstatechanged`  | Fired each time there is an update on the current call.<br>The event contains the `Call`                                                                                                                                                    |
| `oncallended`         | Fired when a call has ended.<br>The event contains a boolean indicating if the call has been ended locally (`true`) or from the remote peer (`false`)                                                                                       |
| `onofferneeded`       | Fired when a call needs a SDP offer.<br>The event contains the `Call`<br>The application should get the local description (SDP) and answer as soon as possible by calling the method `sendOffer` with the offer generated from the browser. |
| `onofferreceived`     | Fired when a call received a SDP offer.<br>The event contains the `RTCSessionDescription` received from the recipient.<br>The application should give that offer to the `RTCPeerConnection`.                                                |
| `oncandidatereceived` | Fired when a call received an ICE candidate.<br>The event contains the `RTCIceCandidate` received from the recipient.<br>The application should give that candidate to the `RTCPeerConnection`.                                             |
| `onticket`            | Fired when the call has ended.<br>The event contains a sum-up of all call information.                                                                                                                                                      |
| `oncallmuted`            | Fired when the remote peer has muted the stream on his side.<br>The event contains the `Call` | 
| `oncallunmuted`            | Fired when the remote peer has unmuted the stream on his side.<br>The event contains the `Call` |
| `onlocalcallmuted`            | Fired when the local stream has been muted.<br>The event contains the `Call` | 
| `onlocalcallunmuted`            | Fired when the local stream has been muted.<br>The event contains the `Call` |
| `ondatareceived`  | Fired when a custom message has been received.<br>The event contains an `Object` representing the content and a `string` representing the issuer |

Here is an example of registering to an event

```js
jsongle.oncallstatechanged = (call) => {
    // The call state has changed. Do something if needed
};
```