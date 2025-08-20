Getting started with peer connections
Peer connections is the part of the WebRTC specifications that deals with connecting two applications on different computers to communicate using a peer-to-peer protocol. The communication between peers can be video, audio or arbitrary binary data (for clients supporting the RTCDataChannel API). In order to discover how two peers can connect, both clients need to provide an ICE Server configuration. This is either a STUN or a TURN-server, and their role is to provide ICE candidates to each client which is then transferred to the remote peer. This transferring of ICE candidates is commonly called signaling.

Signaling
The WebRTC specification includes APIs for communicating with an ICE (Internet Connectivity Establishment) Server, but the signaling component is not part of it. Signaling is needed in order for two peers to share how they should connect. Usually this is solved through a regular HTTP-based Web API (i.e., a REST service or other RPC mechanism) where web applications can relay the necessary information before the peer connection is initiated.

The follow code snippet shows how this fictious signaling service can be used to send and receive messages asynchronously. This will be used in the remaining examples in this guide where necessary.


// Set up an asynchronous communication channel that will be
// used during the peer connection setup
const signalingChannel = new SignalingChannel(remoteClientId);
signalingChannel.addEventListener('message', message => {
    // New message from remote client received
});

// Send an asynchronous message to the remote client
signalingChannel.send('Hello!');
Signaling can be implemented in many different ways, and the WebRTC specification doesn't prefer any specific solution.

Initiating peer connections
Each peer connection is handled by a RTCPeerConnection object. The constructor for this class takes a single RTCConfiguration object as its parameter. This object defines how the peer connection is set up and should contain information about the ICE servers to use.

Once the RTCPeerConnection is created we need to create an SDP offer or answer, depending on if we are the calling peer or receiving peer. Once the SDP offer or answer is created, it must be sent to the remote peer through a different channel. Passing SDP objects to remote peers is called signaling and is not covered by the WebRTC specification.

To initiate the peer connection setup from the calling side, we create a RTCPeerConnection object and then call createOffer() to create a RTCSessionDescription object. This session description is set as the local description using setLocalDescription() and is then sent over our signaling channel to the receiving side. We also set up a listener to our signaling channel for when an answer to our offered session description is received from the receiving side.


async function makeCall() {
    const configuration = {'iceServers': [{'urls': 'stun:stun.l.google.com:19302'}]}
    const peerConnection = new RTCPeerConnection(configuration);
    signalingChannel.addEventListener('message', async message => {
        if (message.answer) {
            const remoteDesc = new RTCSessionDescription(message.answer);
            await peerConnection.setRemoteDescription(remoteDesc);
        }
    });
    const offer = await peerConnection.createOffer();
    await peerConnection.setLocalDescription(offer);
    signalingChannel.send({'offer': offer});
}
On the receiving side, we wait for an incoming offer before we create our RTCPeerConnection instance. Once that is done we set the received offer using setRemoteDescription(). Next, we call createAnswer() to create an answer to the received offer. This answer is set as the local description using setLocalDescription() and then sent to the calling side over our signaling server.


const peerConnection = new RTCPeerConnection(configuration);
signalingChannel.addEventListener('message', async message => {
    if (message.offer) {
        peerConnection.setRemoteDescription(new RTCSessionDescription(message.offer));
        const answer = await peerConnection.createAnswer();
        await peerConnection.setLocalDescription(answer);
        signalingChannel.send({'answer': answer});
    }
});
Once the two peers have set both the local and remote session descriptions they know the capabilities of the remote peer. This doesn't mean that the connection between the peers is ready. For this to work we need to collect the ICE candidates at each peer and transfer (over the signaling channel) to the other peer.

ICE candidates
Before two peers can communitcate using WebRTC, they need to exchange connectivity information. Since the network conditions can vary depending on a number of factors, an external service is usually used for discovering the possible candidates for connecting to a peer. This service is called ICE and is using either a STUN or a TURN server. STUN stands for Session Traversal Utilities for NAT, and is usually used indirectly in most WebRTC applications.

TURN (Traversal Using Relay NAT) is the more advanced solution that incorporates the STUN protocols and most commercial WebRTC based services use a TURN server for establishing connections between peers. The WebRTC API supports both STUN and TURN directly, and it is gathered under the more complete term Internet Connectivity Establishment. When creating a WebRTC connection, we usually provide one or several ICE servers in the configuration for the RTCPeerConnection object.

Trickle ICE
Once a RTCPeerConnection object is created, the underlying framework uses the provided ICE servers to gather candidates for connectivity establishment (ICE candidates). The event icegatheringstatechange on RTCPeerConnection signals in what state the ICE gathering is (new, gathering or complete).

While it is possible for a peer to wait until the ICE gathering is complete, it is usually much more efficient to use a "trickle ice" technique and transmit each ICE candidate to the remote peer as it gets discovered. This will significantly reduce the setup time for the peer connectivity and allow a video call to get started with less delays.

To gather ICE candidates, simply add a listener for the icecandidate event. The RTCPeerConnectionIceEvent emitted on that listener will contain candidate property that represents a new candidate that should be sent to the remote peer (See Signaling).


// Listen for local ICE candidates on the local RTCPeerConnection
peerConnection.addEventListener('icecandidate', event => {
    if (event.candidate) {
        signalingChannel.send({'new-ice-candidate': event.candidate});
    }
});

// Listen for remote ICE candidates and add them to the local RTCPeerConnection
signalingChannel.addEventListener('message', async message => {
    if (message.iceCandidate) {
        try {
            await peerConnection.addIceCandidate(message.iceCandidate);
        } catch (e) {
            console.error('Error adding received ice candidate', e);
        }
    }
});
Connection established
Once ICE candidates are being received, we should expect the state for our peer connection will eventually change to a connected state. To detect this, we add a listener to our RTCPeerConnection where we listen for connectionstatechange events.


// Listen for connectionstatechange on the local RTCPeerConnection
peerConnection.addEventListener('connectionstatechange', event => {
    if (peerConnection.connectionState === 'connected') {
        // Peers connected!
    }
});


Data channels
The WebRTC standard also covers an API for sending arbitrary data over a RTCPeerConnection. This is done by calling createDataChannel() on a RTCPeerConnection object, which returns a RTCDataChannel object.


const peerConnection = new RTCPeerConnection(configuration);
const dataChannel = peerConnection.createDataChannel();
The remote peer can receive data channels by listening for the datachannel event on the RTCPeerConnection object. The received event is of the type RTCDataChannelEvent and contains a channel property that represents the RTCDataChannel connected between the peers.


const peerConnection = new RTCPeerConnection(configuration);
peerConnection.addEventListener('datachannel', event => {
    const dataChannel = event.channel;
});
Open and close events
Before a data channel can be used for sending data, the client needs to wait until it has been opened. This is done by listening to the open event. Likewise, there is a close event for when either side closes the channel.


const messageBox = document.querySelector('#messageBox');
const sendButton = document.querySelector('#sendButton');
const peerConnection = new RTCPeerConnection(configuration);
const dataChannel = peerConnection.createDataChannel();

// Enable textarea and button when opened
dataChannel.addEventListener('open', event => {
    messageBox.disabled = false;
    messageBox.focus();
    sendButton.disabled = false;
});

// Disable input when closed
dataChannel.addEventListener('close', event => {
    messageBox.disabled = false;
    sendButton.disabled = false;
});
Messages
Sending a message on a RTCDataChannel is done by calling the send() function with the data we want to send. The data parameter for this function can be either a string, a Blob, an ArrayBuffer or and ArrayBufferView.


const messageBox = document.querySelector('#messageBox');
const sendButton = document.querySelector('#sendButton');

// Send a simple text message when we click the button
sendButton.addEventListener('click', event => {
    const message = messageBox.textContent;
    dataChannel.send(message);
})
The remote peer will receive messages sent on a RTCDataChannel by listening on the message event.


const incomingMessages = document.querySelector('#incomingMessages');

const peerConnection = new RTCPeerConnection(configuration);
const dataChannel = peerConnection.createDataChannel();

// Append new messages to the box of incoming messages
dataChannel.addEventListener('message', event => {
    const message = event.data;
    incomingMessages.textContent += message + '\n';
});


WebRTC connectivity
This article describes how the various WebRTC-related protocols interact with one another in order to create a connection and transfer data and/or media among peers.

Note: This page needs heavy rewriting for structural integrity and content completeness. Lots of info here is good but the organization is a mess since this is sort of a dumping ground right now.

In this article
Signaling
ICE candidates
When things go wrong
The entire exchange in a complicated diagram
Signaling
Unfortunately, WebRTC can't create connections without some sort of server in the middle. We call this the signal channel or signaling service. It's any sort of channel of communication to exchange information before setting up a connection, whether by email, postcard, or a carrier pigeon. It's up to you.

The information we need to exchange is the Offer and Answer which just contains the SDP mentioned below.

Peer A who will be the initiator of the connection, will create an Offer. They will then send this offer to Peer B using the chosen signal channel. Peer B will receive the Offer from the signal channel and create an Answer. They will then send this back to Peer A along the signal channel.

Session descriptions
The configuration of an endpoint on a WebRTC connection is called a session description. The description includes information about the kind of media being sent, its format, the transfer protocol being used, the endpoint's IP address and port, and other information needed to describe a media transfer endpoint. This information is exchanged and stored using Session Description Protocol (SDP); if you want details on the format of SDP data, you can find it in RFC 8866.

When a user starts a WebRTC call to another user, a special description is created called an offer. This description includes all the information about the caller's proposed configuration for the call. The recipient then responds with an answer, which is a description of their end of the call. In this way, both devices share with one another the information needed in order to exchange media data. This exchange is handled using Interactive Connectivity Establishment (ICE), a protocol which lets two devices use an intermediary to exchange offers and answers even if the two devices are separated by Network Address Translation (NAT).

Each peer, then, keeps two descriptions on hand: the local description, describing itself, and the remote description, describing the other end of the call.

The offer/answer process is performed both when a call is first established, but also any time the call's format or other configuration needs to change. Regardless of whether it's a new call, or reconfiguring an existing one, these are the basic steps which must occur to exchange the offer and answer, leaving out the ICE layer for the moment:

The caller captures local Media via MediaDevices.getUserMedia
The caller creates RTCPeerConnection and calls RTCPeerConnection.addTrack() (Since addStream is deprecating)
The caller calls RTCPeerConnection.createOffer() to create an offer.
The caller calls RTCPeerConnection.setLocalDescription() to set that offer as the local description (that is, the description of the local end of the connection).
After setLocalDescription(), the caller asks STUN servers to generate the ice candidates
The caller uses the signaling server to transmit the offer to the intended receiver of the call.
The recipient receives the offer and calls RTCPeerConnection.setRemoteDescription() to record it as the remote description (the description of the other end of the connection).
The recipient does any setup it needs to do for its end of the call: capture its local media, and attach each media tracks into the peer connection via RTCPeerConnection.addTrack()
The recipient then creates an answer by calling RTCPeerConnection.createAnswer().
The recipient calls RTCPeerConnection.setLocalDescription(), passing in the created answer, to set the answer as its local description. The recipient now knows the configuration of both ends of the connection.
The recipient uses the signaling server to send the answer to the caller.
The caller receives the answer.
The caller calls RTCPeerConnection.setRemoteDescription() to set the answer as the remote description for its end of the call. It now knows the configuration of both peers. Media begins to flow as configured.
Pending and current descriptions
Taking one step deeper into the process, we find that localDescription and remoteDescription, the properties which return these two descriptions, aren't as simple as they look. Because during renegotiation, an offer might be rejected because it proposes an incompatible format, it's necessary that each endpoint have the ability to propose a new format but not actually switch to it until it's accepted by the other peer. For that reason, WebRTC uses pending and current descriptions.

The current description (which is returned by the RTCPeerConnection.currentLocalDescription and RTCPeerConnection.currentRemoteDescription properties) represents the description currently in actual use by the connection. This is the most recent connection that both sides have fully agreed to use.

The pending description (returned by RTCPeerConnection.pendingLocalDescription and RTCPeerConnection.pendingRemoteDescription) indicates a description which is currently under consideration following a call to setLocalDescription() or setRemoteDescription(), respectively.

When reading the description (returned by RTCPeerConnection.localDescription and RTCPeerConnection.remoteDescription), the returned value is the value of pendingLocalDescription/pendingRemoteDescription if there's a pending description (that is, the pending description isn't null); otherwise, the current description (currentLocalDescription/currentRemoteDescription) is returned.

When changing the description by calling setLocalDescription() or setRemoteDescription(), the specified description is set as the pending description, and the WebRTC layer begins to evaluate whether or not it's acceptable. Once the proposed description has been agreed upon, the value of currentLocalDescription or currentRemoteDescription is changed to the pending description, and the pending description is set to null again, indicating that there isn't a pending description.

Note: The pendingLocalDescription contains not just the offer or answer under consideration, but any local ICE candidates which have already been gathered since the offer or answer was created. Similarly, pendingRemoteDescription includes any remote ICE candidates which have been provided by calls to RTCPeerConnection.addIceCandidate().

See the individual articles on these properties and methods for more specifics, and Codecs used by WebRTC for information about codecs supported by WebRTC and which are compatible with which browsers. The codecs guide also offers guidance to help you choose the best codecs for your needs.

ICE candidates
As well as exchanging information about the media (discussed above in Offer/Answer and SDP), peers must exchange information about the network connection. This is known as an ICE candidate and details the available methods the peer is able to communicate (directly or through a TURN server). Typically, each peer will propose its best candidates first, making their way down the line toward their worse candidates. Ideally, candidates are UDP (since it's faster, and media streams are able to recover from interruptions relatively easily), but the ICE standard does allow TCP candidates as well.

Note: Generally, ICE candidates using TCP are only going to be used when UDP is not available or is restricted in ways that make it not suitable for media streaming. Not all browsers support ICE over TCP, however.

ICE allows candidates to represent connections over either TCP or UDP, with UDP generally being preferred (and being more widely supported). Each protocol supports a few types of candidate, with the candidate types defining how the data makes its way from peer to peer.

UDP candidate types
UDP candidates (candidates with their protocol set to udp) can be one of these types:

host
A host candidate is one for which its ip address is the actual, direct IP address of the remote peer.

prflx
A peer reflexive candidate is one whose IP address comes from a symmetric NAT between the two peers, usually as an additional candidate during trickle ICE (that is, additional candidate exchanges that occur after primary signaling but before the connection verification phase is finished).

srflx
A server reflexive candidate is generated by a STUN/TURN server; the connection's initiator requests a candidate from the STUN server, which forwards the request through the remote peer's NAT, which creates and returns a candidate whose IP address is local to the remote peer. The STUN server then replies to the initiator's request with a candidate whose IP address is unrelated to the remote peer.

relay
A relay candidate is generated just like a server reflexive candidate ("srflx"), but using TURN instead of STUN.

TCP candidate types
TCP candidates (that is, candidates whose protocol is tcp) can be of these types:

active
The transport will try to open an outbound connection but won't receive incoming connection requests. This is the most common type, and the only one that most user agents will gather.

passive
The transport will receive incoming connection attempts but won't attempt a connection itself.

so
The transport will try to simultaneously open a connection with its peer.

Choosing a candidate pair
The ICE layer selects one of the two peers to serve as the controlling agent. This is the ICE agent which will make the final decision as to which candidate pair to use for the connection. The other peer is called the controlled agent. You can identify which one your end of the connection is by examining the value of RTCIceCandidate.transport.role, although in general it doesn't matter which is which.

The controlling agent not only takes responsibility for making the final decision as to which candidate pair to use, but also for signaling that selection to the controlled agent by using STUN and an updated offer, if necessary. The controlled agent just waits to be told which candidate pair to use.

It's important to keep in mind that a single ICE session may result in the controlling agent choosing more than one candidate pair. Each time it does so and shares that information with the controlled agent, the two peers reconfigure their connection to use the new configuration described by the new candidate pair.

Once the ICE session is complete, the configuration that's currently in effect is the final one, unless an ICE reset occurs.

At the end of each generation of candidates, an end-of-candidates notification is sent in the form of an RTCIceCandidate whose candidate property is an empty string. This candidate should still be added to the connection using addIceCandidate() method, as usual, in order to deliver that notification to the remote peer.

When there are no more candidates at all to be expected during the current negotiation exchange, an end-of-candidates notification is sent by delivering a RTCIceCandidate whose candidate property is null. This message does not need to be sent to the remote peer. It's a legacy notification of a state which can be detected instead by watching for the iceGatheringState to change to complete, by watching for the icegatheringstatechange event.

When things go wrong
During negotiation, there will be times when things just don't work out. For example, when renegotiating a connection—for example, to adapt to changing hardware or network configurations—it's possible that negotiation could reach a dead end, or some form of error might occur that prevents negotiation at all. There may be permissions issues or other problems as well, for that matter.

ICE rollbacks
When renegotiating a connection that's already active and a situation arises in which the negotiation fails, you don't really want to kill the already-running call. After all, you were most likely just trying to upgrade or downgrade the connection, or to otherwise make adaptations to an ongoing session. Aborting the call would be an excessive reaction in that situation.

Instead, you can initiate an ICE rollback. A rollback restores the SDP offer (and the connection configuration by extension) to the configuration it had the last time the connection's signalingState was stable.

To programmatically initiate a rollback, send a description whose type is rollback. Any other properties in the description object are ignored.

In addition, the ICE agent will automatically initiate a rollback when a peer that had previously created an offer receives an offer from the remote peer. In other words, if the local peer is in the state have-local-offer, indicating that the local peer had previously sent an offer, calling setRemoteDescription() with a received offer triggers rollback so that the negotiation switches from the remote peer being the caller to the local peer being the caller.


Establishing a connection: The WebRTC perfect negotiation pattern
This article introduces WebRTC perfect negotiation, describing how it works and why it's the recommended way to negotiate a WebRTC connection between peers, and provides sample code to demonstrate the technique.

Because WebRTC doesn't mandate a specific transport mechanism for signaling during the negotiation of a new peer connection, it's highly flexible. However, despite that flexibility in transport and communication of signaling messages, there's still a recommended design pattern you should follow when possible, known as perfect negotiation.

After the first deployments of WebRTC-capable browsers, it was realized that parts of the negotiation process were more complicated than they needed to be for typical use cases. This was due to a small number of issues with the API and some potential race conditions that needed to be prevented. These issues have since been addressed, letting us simplify our WebRTC negotiation significantly. The perfect negotiation pattern is an example of the ways in which negotiation have improved since the early days of WebRTC.

In this article
Perfect negotiation concepts
Create the signaling and peer connections
Connecting to a remote peer
Handling incoming tracks
The perfect negotiation logic
See also
Perfect negotiation concepts
Perfect negotiation makes it possible to seamlessly and completely separate the negotiation process from the rest of your application's logic. Negotiation is an inherently asymmetric operation: one side needs to serve as the "caller" while the other peer is the "callee." The perfect negotiation pattern smooths this difference away by separating that difference out into independent negotiation logic, so that your application doesn't need to care which end of the connection it is. As far as your application is concerned, it makes no difference whether you're calling out or receiving a call.

The best thing about perfect negotiation is that the same code is used for both the caller and the callee, so there's no repetition or otherwise added levels of negotiation code to write.

Perfect negotiation works by assigning each of the two peers a role to play in the negotiation process that's entirely separate from the WebRTC connection state:

A polite peer, which uses ICE rollback to prevent collisions with incoming offers. A polite peer, essentially, is one which may send out offers, but then responds if an offer arrives from the other peer with "Okay, never mind, drop my offer and I'll consider yours instead."
An impolite peer, which always ignores incoming offers that collide with its own offers. It never apologizes or gives up anything to the polite peer. Any time a collision occurs, the impolite peer wins.
This way, both peers know exactly what should happen if there are collisions between offers that have been sent. Responses to error conditions become far more predictable.

How you determine which peer is polite and which is impolite is generally up to you. It could be as simple as assigning the polite role to the first peer to connect to the signaling server, or you could do something more elaborate like having the peers exchange random numbers and assigning the polite role to the winner. However you make the determination, once these roles are assigned to the two peers, they can then work together to manage signaling in a way that doesn't deadlock and doesn't require a lot of extra code to manage.

An important thing to keep in mind is this: the roles of caller and callee can switch during perfect negotiation. If the polite peer is the caller and it sends an offer but there's a collision with the impolite peer, the polite peer drops its offer and instead replies to the offer it has received from the impolite peer. By doing so, the polite peer has switched from being the caller to the callee!

Let's take a look at an example that implements the perfect negotiation pattern. The code assumes that there's a SignalingChannel class defined that is used to communicate with the signaling server. Your own code, of course, can use any signaling technique you like.

Note that this code is identical for both peers involved in the connection.

Create the signaling and peer connections
First, the signaling channel needs to be opened and the RTCPeerConnection needs to be created. The STUN server listed here is obviously not a real one; you'll need to replace stun.my-server.tld with the address of a real STUN server.

js

Copy
const config = {
  iceServers: [{ urls: "stun:stun.my-stun-server.tld" }],
};

const signaler = new SignalingChannel();
const pc = new RTCPeerConnection(config);
This code also gets the <video> elements using the classes "self-view" and "remote-view"; these will contain, respectively, the local user's self-view and the view of the incoming stream from the remote peer.

Connecting to a remote peer
js

Copy
const constraints = { audio: true, video: true };
const selfVideo = document.querySelector("video.self-view");
const remoteVideo = document.querySelector("video.remote-view");

async function start() {
  try {
    const stream = await navigator.mediaDevices.getUserMedia(constraints);

    for (const track of stream.getTracks()) {
      pc.addTrack(track, stream);
    }
    selfVideo.srcObject = stream;
  } catch (err) {
    console.error(err);
  }
}
The start() function shown above can be called by either of the two end-points that want to talk to one another. It doesn't matter who does it first; the negotiation will just work.

This isn't appreciably different from older WebRTC connection establishment code. The user's camera and microphone are obtained by calling getUserMedia(). The resulting media tracks are then added to the RTCPeerConnection by passing them into addTrack(). Then, finally, the media source for the self-view <video> element indicated by the selfVideo constant is set to the camera and microphone stream, allowing the local user to see what the other peer sees.

Handling incoming tracks
We next need to set up a handler for track events to handle inbound video and audio tracks that have been negotiated to be received by this peer connection. To do this, we implement the RTCPeerConnection's ontrack event handler.

js

Copy
pc.ontrack = ({ track, streams }) => {
  track.onunmute = () => {
    if (remoteVideo.srcObject) {
      return;
    }
    remoteVideo.srcObject = streams[0];
  };
};
When the track event occurs, this handler executes. Using destructuring, the RTCTrackEvent's track and streams properties are extracted. The former is either the video track or the audio track being received. The latter is an array of MediaStream objects, each representing a stream containing this track (a track may in rare cases belong to multiple streams at once). In our case, this will always contain one stream, at index 0, because we passed one stream into addTrack() earlier.

We add an unmute event handler to the track, because the track will become unmuted once it starts receiving packets. We put the remainder of our reception code in there.

If we already have video coming in from the remote peer (which we can see if the remote view's <video> element's srcObject property already has a value), we do nothing. Otherwise, we set srcObject to the stream at index 0 in the streams array.

The perfect negotiation logic
Now we get into the true perfect negotiation logic, which functions entirely independently from the rest of the application.

Handling the negotiationneeded event
First, we implement the RTCPeerConnection event handler onnegotiationneeded to get a local description and send it using the signaling channel to the remote peer.

js

Copy
let makingOffer = false;

pc.onnegotiationneeded = async () => {
  try {
    makingOffer = true;
    await pc.setLocalDescription();
    signaler.send({ description: pc.localDescription });
  } catch (err) {
    console.error(err);
  } finally {
    makingOffer = false;
  }
};
Note that setLocalDescription() without arguments automatically creates and sets the appropriate description based on the current signalingState. The set description is either an answer to the most recent offer from the remote peer or a freshly-created offer if there's no negotiation underway. Here, it will always be an offer, because the negotiationneeded event is only fired in stable state.

We set a Boolean variable, makingOffer to true to mark that we're preparing an offer. We set makingOffer immediately before calling setLocalDescription() in order to lock against interfering with sending this offer, and we don't clear it back to false until the offer has been sent to the signaling server (or an error has occurred, preventing the offer from being made). To avoid races, we'll use this value later instead of the signaling state to determine whether or not an offer is being processed because the value of signalingState changes asynchronously, introducing a potential collision of an outgoing and an incoming call ("glare").

Handling incoming ICE candidates
Next, we need to handle the RTCPeerConnection event icecandidate, which is how the local ICE layer passes candidates to us for delivery to the remote peer over the signaling channel.

js

Copy
pc.onicecandidate = ({ candidate }) => signaler.send({ candidate });
This takes the candidate member of this ICE event and passes it through to the signaling channel's send() method to be sent over the signaling server to the remote peer.

Handling incoming messages on the signaling channel
The last piece of the puzzle is code to handle incoming messages from the signaling server. That's implemented here as an onmessage event handler on the signaling channel object. This method is invoked each time a message arrives from the signaling server.

js

Copy
let ignoreOffer = false;
let isSettingRemoteAnswerPending = false;

signaler.onmessage = async ({ data: { description, candidate } }) => {
  try {
    if (description) {
      const readyForOffer =
        !makingOffer &&
        (pc.signalingState === "stable" || isSettingRemoteAnswerPending);
      const offerCollision = description.type === "offer" && !readyForOffer;

      ignoreOffer = !polite && offerCollision;
      if (ignoreOffer) {
        return;
      }
      isSettingRemoteAnswerPending = description.type === "answer";
      await pc.setRemoteDescription(description);
      isSettingRemoteAnswerPending = false;
      if (description.type === "offer") {
        await pc.setLocalDescription();
        signaler.send({ description: pc.localDescription });
      }
    } else if (candidate) {
      try {
        await pc.addIceCandidate(candidate);
      } catch (err) {
        if (!ignoreOffer) {
          throw err;
        }
      }
    }
  } catch (err) {
    console.error(err);
  }
};
Upon receiving an incoming message from the SignalingChannel through its onmessage event handler, the received JSON object is destructured to obtain the description or candidate found within. If the incoming message has a description, it's either an offer or an answer sent by the other peer.

If, on the other hand, the message has a candidate, it's an ICE candidate received from the remote peer as part of trickle ICE. The candidate is destined to be delivered to the local ICE layer by passing it into addIceCandidate().

On receiving a description
If we received a description, we prepare to respond to the incoming offer or answer. First, we check to make sure we're in a state in which we can accept an offer. If the connection's signaling state isn't stable or if our end of the connection has started the process of making its own offer, then we need to look out for offer collision.

If we're the impolite peer, and we're receiving a colliding offer, we return without setting the description, and instead set ignoreOffer to true to ensure we also ignore all candidates the other side may be sending us on the signaling channel belonging to this offer. Doing so avoids error noise since we never informed our side about this offer.

If we're the polite peer, and we're receiving a colliding offer, we don't need to do anything special, because our existing offer will automatically be rolled back in the next step.

Having ensured that we want to accept the offer, we set the remote description to the incoming offer by calling setRemoteDescription(). This lets WebRTC know what the proposed configuration of the other peer is. If we're the polite peer, we will drop our offer and accept the new one.

If the newly-set remote description is an offer, we ask WebRTC to select an appropriate local configuration by calling the RTCPeerConnection method setLocalDescription() without parameters. This causes setLocalDescription() to automatically generate an appropriate answer in response to the received offer. Then we send the answer through the signaling channel back to the first peer.

On receiving an ICE candidate
On the other hand, if the received message contains an ICE candidate, we deliver it to the local ICE layer by calling the RTCPeerConnection method addIceCandidate(). If an error occurs and we've ignored the most recent offer, we also ignore any error that may occur when trying to add the


Lifetime of a WebRTC session
WebRTC lets you build peer-to-peer communication of arbitrary data, audio, or video—or any combination thereof—into a browser application. In this article, we'll look at the lifetime of a WebRTC session, from establishing the connection all the way through closing the connection when it's no longer needed.

This article doesn't get into details of the actual APIs involved in establishing and handling a WebRTC connection; it reviews the process in general with some information about why each step is required. See Signaling and video calling for an actual example with a step-by-step explanation of what the code does.

Note: This page is currently under construction, and some of the content will move to other pages as the WebRTC guide material is built out. Pardon our dust!

In this article
Establishing the connection
ICE restart
Establishing the connection
The internet is big. Really big. It's so big that years ago, smart people saw how big it was, how fast it was growing, and the limitations of the 32-bit IP addressing system, and realized that something had to be done before we ran out of addresses to use, so they started working on designing a new 64-bit addressing system. But they realized that it would take longer to complete the transition than 32-bit addresses would last, so other smart people came up with a way to let multiple computers share the same 32-bit IP address. Network Address Translation (NAT) is a standard which supports this address sharing by handling routing of data inbound and outbound to and from devices on a LAN, all of which are sharing a single WAN (global) IP address.

The problem for users is that each individual computer on the internet no longer necessarily has a unique IP address, and, in fact, each device's IP address may change not only if they move from one network to another, but if their network's address is changed by NAT and/or DHCP. For developers trying to do peer-to-peer networking, this introduces a conundrum: without a unique identifier for every user device, it's not possible to instantly and automatically know how to connect to a specific device on the internet. Even though you know who you want to talk to, you don't necessarily know how to reach them or even what their address is.

This is like trying to mail a package to your friend Michelle by labeling it "Michelle" and dropping it in a mailbox when you don't know her address. You need to look up her address and include it on the package, or she'll wind up wondering why you forgot her birthday again.

This is where signaling comes in.

Signaling
Signaling is the process of sending control information between two devices to determine the communication protocols, channels, media codecs and formats, and method of data transfer, as well as any required routing information. The most important thing to know about the signaling process for WebRTC: it is not defined in the specification.

Why, you may wonder, is something fundamental to the process of establishing a WebRTC connection left out of the specification? The answer is simple: since the two devices have no way to directly contact each other, and the specification can't predict every possible use case for WebRTC, it makes more sense to let the developer select an appropriate networking technology and messaging protocol.

In particular, if a developer already has a method in place for connecting two devices, it doesn't make sense for them to have to use another one, defined by the specification, just for WebRTC. Since WebRTC doesn't live in a vacuum, there is likely other connectivity in play, so it makes sense to avoid having to add additional connection channels for signaling if an existing one can be used.

In order to exchange signaling information, you can choose to send JSON objects back and forth over a WebSocket connection, or you could use XMPP or SIP over an appropriate channel, or you could use fetch() over HTTPS with polling, or any other combination of technologies you can come up with. You could even use email as the signaling channel.

It's also worth noting that the channel for performing signaling doesn't even need to be over the network. One peer can output a data object that can be printed out, physically carried (on foot or by carrier pigeon) to another device, entered into that device, and a response then output by that device to be returned on foot, and so forth, until the WebRTC peer connection is open. It'd be very high latency but it could be done.

Information exchanged during signaling
There are three basic types of information that need to be exchanged during signaling:

Control messages used to set up, open, and close the communication channel, and to handle errors.
Information needed in order to set up the connection: the IP addressing and port information needed for the peers to be able to talk to one another.
Media capability negotiation: what codecs and media data formats can the peers understand? These need to be agreed upon before the WebRTC session can begin.
Only once signaling has been successfully completed can the true process of opening the WebRTC peer connection begin.

It's worth noting that the signaling server does not actually need to understand or do anything with the data being exchanged through it by the two peers during signaling. The signaling server is, in essence, a relay: a common point which both sides connect to knowing that their signaling data can be transferred through it. The server doesn't need to react to this information in any way.

The signaling process
There's a sequence of things that have to happen in order to make it possible to begin a WebRTC session:

Each peer creates an RTCPeerConnection object representing their end of the WebRTC session.
Each peer establishes a handler for icecandidate events, which handles sending those candidates to the other peer over the signaling channel.
Each peer establishes a handler for track event, which is received when the remote peer adds a track to the stream. This code should connect the tracks to its consumer, such as a <video> element.
The caller creates and shares with the receiving peer a unique identifier or token of some kind so that the call between them can be identified by the code on the signaling server. The exact contents and form of this identifier is up to you.
Each peer connects to an agreed-upon signaling server, such as a WebSocket server they both know how to exchange messages with.
Each peer tells the signaling server that they want to join the same WebRTC session (identified by the token established in step 4).
descriptions, candidates, etc. — more coming up
ICE restart
Sometimes, during the lifetime of a WebRTC session, network conditions change. One of the users might transition from a cellular to a Wi-Fi network, or the network might become congested, for example. When this happens, the ICE agent may choose to perform ICE restart. This is a process by which the network connection is renegotiated, exactly the same way the initial ICE negotiation is performed, with one exception: media continues to flow across the original network connection until the new one is up and running. Then media shifts to the new network connection and the old one is closed.

Note: Different browsers support ICE restart under different sets of conditions. Not all browsers will perform ICE restart due to network congestion, for example.

The handling of the failed ICE connection state below shows how you might restart the connection.

js

Copy
pc.oniceconnectionstatechange = () => {
  if (pc.iceConnectionState === "failed") {
    pc.setConfiguration(restartConfig);
    pc.restartIce();
  }
};
The code first calls RTCPeerConnection.setConfiguration() with an updated configuration object. This should be done before restarting ICE if you need to change the connection configuration in some way (such as changing to a different set of ICE servers).

The handler then calls RTCPeerConnection.restartIce(). This tells the ICE layer to automatically add the iceRestart flag to the next createOffer() call, which triggers an ICE restart. It also creates new values for the ICE username fragment (ufrag) and password, which will be used by the renegotiation process and the resulting connection.

The answerer side of the connection will automatically begin ICE restart when new values are detected for the ICE ufrag and ICE password.



A simple RTCDataChannel sample
The RTCDataChannel interface is a feature of the WebRTC API which lets you open a channel between two peers over which you may send and receive arbitrary data. The API is intentionally similar to the WebSocket API, so that the same programming model can be used for each.

In this example, we will open an RTCDataChannel connection linking two elements on the same page. While that's obviously a contrived scenario, it's useful for demonstrating the flow of connecting two peers. We'll cover the mechanics of accomplishing the connection and transmitting and receiving data, but we will save the bits about locating and linking to a remote computer for another example.

In this article
The HTML
The JavaScript code
Next steps
See also
The HTML
First, let's take a quick look at the HTML that's needed. There's nothing incredibly complicated here. First, we have a couple of buttons for establishing and closing the connection:

html

Copy
<button id="connectButton" name="connectButton" class="buttonleft">
  Connect
</button>
<button
  id="disconnectButton"
  name="disconnectButton"
  class="buttonright"
  disabled>
  Disconnect
</button>
Then there's a box which contains the text input box into which the user can type a message to transmit, with a button to send the entered text. This <div> will be the first peer in the channel.

html

Copy
<div class="messagebox">
  <label for="message"
    >Enter a message:
    <input
      type="text"
      name="message"
      id="message"
      placeholder="Message text"
      inputmode="latin"
      size="60"
      maxlength="120"
      disabled />
  </label>
  <button id="sendButton" name="sendButton" class="buttonright" disabled>
    Send
  </button>
</div>
Finally, there's the little box into which we'll insert the messages. This <div> block will be the second peer.

html

Copy
<div class="messagebox" id="receive-box">
  <p>Messages received:</p>
</div>
The JavaScript code
While you can just look at the code itself on GitHub, below we'll review the parts of the code that do the heavy lifting.

Starting up
When the script is run, we set up a load event listener, so that once the page is fully loaded, our startup() function is called.

js

Copy
let connectButton = null;
let disconnectButton = null;
let sendButton = null;
let messageInputBox = null;
let receiveBox = null;

let localConnection = null; // RTCPeerConnection for our "local" connection
let remoteConnection = null; // RTCPeerConnection for the "remote"

let sendChannel = null; // RTCDataChannel for the local (sender)
let receiveChannel = null; // RTCDataChannel for the remote (receiver)

function startup() {
  connectButton = document.getElementById("connectButton");
  disconnectButton = document.getElementById("disconnectButton");
  sendButton = document.getElementById("sendButton");
  messageInputBox = document.getElementById("message");
  receiveBox = document.getElementById("receive-box");

  // Set event listeners for user interface widgets

  connectButton.addEventListener("click", connectPeers, false);
  disconnectButton.addEventListener("click", disconnectPeers, false);
  sendButton.addEventListener("click", sendMessage, false);
}
This is quite straightforward. We declare variables and grab references to all the page elements we'll need to access, then set event listeners on the three buttons.

Establishing a connection
When the user clicks the "Connect" button, the connectPeers() method is called. We're going to break this up and look at it a bit at a time, for clarity.

Note: Even though both ends of our connection will be on the same page, we're going to refer to the one that starts the connection as the "local" one, and to the other as the "remote" end.

Set up the local peer
js

Copy
localConnection = new RTCPeerConnection();

sendChannel = localConnection.createDataChannel("sendChannel");
sendChannel.onopen = handleSendChannelStatusChange;
sendChannel.onclose = handleSendChannelStatusChange;
The first step is to create the "local" end of the connection. This is the peer that will send out the connection request. The next step is to create the RTCDataChannel by calling RTCPeerConnection.createDataChannel() and set up event listeners to monitor the channel so that we know when it's opened and closed (that is, when the channel is connected or disconnected within that peer connection).

It's important to keep in mind that each end of the channel has its own RTCDataChannel object.

Set up the remote peer
js

Copy
remoteConnection = new RTCPeerConnection();
remoteConnection.ondatachannel = receiveChannelCallback;
The remote end is set up similarly, except that we don't need to explicitly create an RTCDataChannel ourselves, since we're going to be connected through the channel established above. Instead, we set up a datachannel event handler; this will be called when the data channel is opened; this handler will receive an RTCDataChannel object; you'll see this below.

Set up the ICE candidates
The next step is to set up each connection with ICE candidate listeners; these will be called when there's a new ICE candidate to tell the other side about.

Note: In a real-world scenario in which the two peers aren't running in the same context, the process is a bit more involved; each side provides, one at a time, a suggested way to connect (for example, UDP, UDP with a relay, TCP, etc.) by calling RTCPeerConnection.addIceCandidate(), and they go back and forth until agreement is reached. But here, we just accept the first offer on each side, since there's no actual networking involved.

js

Copy
localConnection.onicecandidate = (e) =>
  !e.candidate ||
  remoteConnection.addIceCandidate(e.candidate).catch(handleAddCandidateError);

remoteConnection.onicecandidate = (e) =>
  !e.candidate ||
  localConnection.addIceCandidate(e.candidate).catch(handleAddCandidateError);
We configure each RTCPeerConnection to have an event handler for the icecandidate event.

Start the connection attempt
The last thing we need to do in order to begin connecting our peers is to create a connection offer.

js

Copy
localConnection
  .createOffer()
  .then((offer) => localConnection.setLocalDescription(offer))
  .then(() =>
    remoteConnection.setRemoteDescription(localConnection.localDescription),
  )
  .then(() => remoteConnection.createAnswer())
  .then((answer) => remoteConnection.setLocalDescription(answer))
  .then(() =>
    localConnection.setRemoteDescription(remoteConnection.localDescription),
  )
  .catch(handleCreateDescriptionError);
Let's go through this line by line and decipher what it means.

First, we call RTCPeerConnection.createOffer() method to create an SDP (Session Description Protocol) blob describing the connection we want to make. This method accepts, optionally, an object with constraints to be met for the connection to meet your needs, such as whether the connection should support audio, video, or both. In our simple example, we don't have any constraints.
If the offer is created successfully, we pass the blob along to the local connection's RTCPeerConnection.setLocalDescription() method. This configures the local end of the connection.
The next step is to connect the local peer to the remote by telling the remote peer about it. This is done by calling remoteConnection.setRemoteDescription(). Now the remoteConnection knows about the connection that's being built. In a real application, this would require a signaling server to exchange the description object.
That means it's time for the remote peer to reply. It does so by calling its createAnswer() method. This generates a blob of SDP which describes the connection the remote peer is willing and able to establish. This configuration lies somewhere in the union of options that both peers can support.
Once the answer has been created, it's passed into the remoteConnection by calling RTCPeerConnection.setLocalDescription(). That establishes the remote's end of the connection (which, to the remote peer, is its local end. This stuff can be confusing, but you get used to it). Again, this would normally be exchanged through a signalling server.
Finally, the local connection's remote description is set to refer to the remote peer by calling localConnection's RTCPeerConnection.setRemoteDescription().
The catch() calls a routine that handles any errors that occur.
Note: Once again, this process is not a real-world implementation; in normal usage, there's two chunks of code running on two machines, interacting and negotiating the connection. A side channel, commonly called a "signalling server," is usually used to exchange the description (which is in application/sdp form) between the two peers.

Handling successful peer connection
As each side of the peer-to-peer connection is successfully linked up, the corresponding RTCPeerConnection's icecandidate event is fired. These handlers can do whatever's needed, but in this example, all we need to do is update the user interface:

js

Copy
function handleCreateDescriptionError(error) {
  console.log(`Unable to create an offer: ${error.toString()}`);
}

function handleLocalAddCandidateSuccess() {
  connectButton.disabled = true;
}

function handleRemoteAddCandidateSuccess() {
  disconnectButton.disabled = false;
}

function handleAddCandidateError() {
  console.log("Oh noes! addICECandidate failed!");
}
The only thing we do here is disable the "Connect" button when the local peer is connected and enable the "Disconnect" button when the remote peer connects.

Connecting the data channel
Once the RTCPeerConnection is open, the datachannel event is sent to the remote to complete the process of opening the data channel; this invokes our receiveChannelCallback() method, which looks like this:

js

Copy
function receiveChannelCallback(event) {
  receiveChannel = event.channel;
  receiveChannel.onmessage = handleReceiveMessage;
  receiveChannel.onopen = handleReceiveChannelStatusChange;
  receiveChannel.onclose = handleReceiveChannelStatusChange;
}
The datachannel event includes, in its channel property, a reference to a RTCDataChannel representing the remote peer's end of the channel. This is saved, and we set up, on the channel, event listeners for the events we want to handle. Once this is done, our handleReceiveMessage() method will be called each time data is received by the remote peer, and the handleReceiveChannelStatusChange() method will be called any time the channel's connection state changes, so we can react when the channel is fully opened and when it's closed.

Handling channel status changes
Both our local and remote peers use a single method to handle events indicating a change in the status of the channel's connection.

When the local peer experiences an open or close event, the handleSendChannelStatusChange() method is called:

js

Copy
function handleSendChannelStatusChange(event) {
  if (sendChannel) {
    const state = sendChannel.readyState;

    if (state === "open") {
      messageInputBox.disabled = false;
      messageInputBox.focus();
      sendButton.disabled = false;
      disconnectButton.disabled = false;
      connectButton.disabled = true;
    } else {
      messageInputBox.disabled = true;
      sendButton.disabled = true;
      connectButton.disabled = false;
      disconnectButton.disabled = true;
    }
  }
}
If the channel's state has changed to "open", that indicates that we have finished establishing the link between the two peers. The user interface is updated correspondingly by enabling the text input box for the message to send, focusing the input box so that the user can immediately begin to type, enabling the "Send" and "Disconnect" buttons, now that they're usable, and disabling the "Connect" button, since it is not needed when the connection is open.

If the state has changed to "closed", the opposite set of actions occurs: the input box and "Send" button are disabled, the "Connect" button is enabled so that the user can open a new connection if they wish to do so, and the "Disconnect" button is disabled, since it's not useful when no connection exists.

Our example's remote peer, on the other hand, ignores the status change events, except for logging the event to the console:

js

Copy
function handleReceiveChannelStatusChange(event) {
  if (receiveChannel) {
    console.log(
      `Receive channel's status has changed to ${receiveChannel.readyState}`,
    );
  }
}
The handleReceiveChannelStatusChange() method receives as an input parameter the event which occurred; this will be an RTCDataChannelEvent.

Sending messages
When the user presses the "Send" button, the sendMessage() method we've established as the handler for the button's click event is called. That method is simple enough:

js

Copy
function sendMessage() {
  const message = messageInputBox.value;
  sendChannel.send(message);

  messageInputBox.value = "";
  messageInputBox.focus();
}
First, the text of the message is fetched from the input box's value attribute. This is then sent to the remote peer by calling sendChannel.send(). That's all there is to it! The rest of this method is just some user experience sugar — the input box is emptied and re-focused so the user can immediately begin typing another message.

Receiving messages
When a "message" event occurs on the remote channel, our handleReceiveMessage() method is called as the event handler.

js

Copy
function handleReceiveMessage(event) {
  const el = document.createElement("p");
  const textNode = document.createTextNode(event.data);

  el.appendChild(textNode);
  receiveBox.appendChild(el);
}
This method performs some basic DOM injection; it creates a new <p> (paragraph) element, then creates a new Text node containing the message text, which is received in the event's data property. This text node is appended as a child of the new element, which is then inserted into the receiveBox block, thereby causing it to draw in the browser window.

Disconnecting the peers
When the user clicks the "Disconnect" button, the disconnectPeers() method previously set as that button's handler is called.

js

Copy
function disconnectPeers() {
  // Close the RTCDataChannels if they're open.

  sendChannel.close();
  receiveChannel.close();

  // Close the RTCPeerConnections

  localConnection.close();
  remoteConnection.close();

  sendChannel = null;
  receiveChannel = null;
  localConnection = null;
  remoteConnection = null;

  // Update user interface elements

  connectButton.disabled = false;
  disconnectButton.disabled = true;
  sendButton.disabled = true;

  messageInputBox.value = "";
  messageInputBox.disabled = true;
}
This starts by closing each peer's RTCDataChannel, then, similarly, each RTCPeerConnection. Then all the saved references to these objects are set to null to avoid accidental reuse, and the user interface is updated to reflect the fact that the connection has been closed.
