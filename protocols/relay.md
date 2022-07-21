# Relay Protocol

The `Relay Protocol (RP)` is the base layer communication protocol in the Coalescent Computer stack. It is designed to be a simple and modular protocol that sits directly between the Transport and Application layers. If designed correctly, it should allow for endless rapid evolution of communication protocols without ever having to change the `Relay Protocol`.

It has one purpose: bootstrapping into *other* communication protocols. It does this by establishing a protocol for protocol negotiation and hand-off.

## Design Inspiration

All communication protocols do is layer Signals on a Medium. At each layer, the Signal is a predefined message (sequence of symbols) that both parties have agreed to a meaning for. At all layers but the final layer, there is also a Payload which is passed to the next layer.

In the Internet Protocol Suite, we see these layers pretty clearly. At the bottom we have the physical hardware protocols, sending electrical Signal over wires. These Signals are decoded as binary 1s and 0s on the other side, which then get further decoded into IP messages. Those IP messages contain a header (its Signal) for routing, and a Payload for the next layer of the stack. In a TCP connection, that IP Payload is the TCP Message, again with a header (Signal) and another Payload for the application layer of the stack. The Relay Protocol defines a continuation of this pattern into the Application layer, for an endlessly evolvable, composable, pluggable protcol stack.

## Protocol Definition

> Okay, so we need a way to have the relay protocol know if it accepted a connection or closed a connection from higher level protocols... hmmm... I get messages and forward them based on my state, the only state is "do we have a protocol connection open for this address". so the higher level protocols need to be able to tell me "hey, this connection is good, i'm your boy" and "hey, things didn't work out, this connection is closed". oh, maybe relay receives a message, kicks it to the next layer, but waits for a respons from that layer before processing the next message. the response from a higher layer can either be "connection opened", "cool, thanks, we're good", "sick, also, can you send this back", or "connection closed".

Externally, a Relay Protocol node receives, processes, and sends messages to other Relay Protocol nodes. Internally, a Relay Protocol node forwards message payloads to registered protocol handlers, and expects responses.

### Relay Bootstrap Protocol Messages

#### Connection Request

```
ConnectionRequest {
    proposals: [(bool, ProtocolId)],
    payload: [byte],
}
```
Connection Request includes a list of protocol proposals, in the form of , followed by a payload, in the form of a count-prefixed array of bytes.

* If the Relay node **does not have** an active connection with the sender, it begins protocol negotiation.
    * During protocol negotiation, the Relay node steps through the list of proposals in order.
        * If the Relay node does not support a protocol, it skips it.
        * If the Relay node supports a protocol:
            * **and it is marked as compatible with the payload,** it sends the payload to the protocol handler and waits for a handler response.
            * **and it is not compatible with the payload,** it sends a `ConnectionNegotiationPayloadRequest` message to the sender.
* If the Relay node **does have** an active connection with the sender, it returns a `ConnectionRefused` message.


#### Connection Negotiation Payload Request

```
ConnectionNegotiationPayloadRequest {
    proposal: ProtocolId,
}
```
When trying to initiate a connection, a recipient may select a protocol that is not compatible with the initial payload. In that case, it will respond with a ConnectionNegotiationPayloadRequest, containing the ProtocolId for the selected protocol. The initiator should then respond with a `ConnectionNegotiationPayloadResponse`, to differentiate from an initial `ConnectionRequest` message. If the sender was not expecting a potential negotiation request, or if the ProtocolId in the `proposal` was not sent to recipient making the request, the sender should return a `ConnectionRefused` message.


#### Connection Negotiation Payload Response

```
ConnectionNegotiationPayloadResponse {
    proposal: ProtocolId,
    payload: [byte],
}
```
After asking for a new payload to initiate a connection, a recipient should expect to receive a `ConnectionNegotiationPayloadResponse`, containing the requested ProtocolId in addition to the payload. If the recipient wasn't expecting this message, or if the `proposal` does not match the request, the recipient should return a `ConnectionRefused` message. If the `proposal` is correct, the recipient should forward the `payload` to the specified protocol handler and wait for a handler response.


#### Connection Refused

```
ConnectionRefused {}
```
Connection Refused is an otherwise empty message that is returned when a `ConnectionRequest` is not successful, and does not want the sender to try again.

!!! Perhaps there should be specific alternates of this message so that we can prevent spoofing from interrupting the handshake.


#### Connection Accepted

```
ConnectionAccepted {
    proposal: ProtocolId,
    payload: [byte],
}
```
Once the reciever accepts the connection, it will respond with a `ConnectionAccepted` message, contining the accepted ProtocolId `proposal` and a `payload`. If the sender wasn't expecting this acceptance message, it should respond with a `ConnectionRefusal` message. If the `proposal` checks out, the sender should forward the payload to the specified protocol handler and wait for a handler response. If the response is okay, the sender replies with a final `ConnectionConfirmed` message.

#### Connection Confirmed

```
ConnectionConfirmed {
    payload: [byte],
}
```
This message is only expected by the recipient after the sender has confirmed the connection. The optional payload can be used to verify that the confirmation is coming from the sender, since both sides have had an opportunity to establish some sort of secret.


#### Relay Message

```
RelayMessage {
    payload: [(count) byte],
}
```

Once a connection has been established, all further messages will be sent as a `RelayMessage`.