# CONCEPTS

A client application interacts with the CPAAS server using _Resources_, _Artifacts_, and _Operations_.

A _Resource_ is a telephony-related object created on the CPAAS platform as a result of an expressed intent of a client application.  Examples of _Resources_ include:
* Calls
* Conferences
* Bridges (i.e., a relation between two calls, or between a call and a conference)
* IVRs (speech or DTMF interaction between a person or endpoint and the CPAAS
* Registered SIP endpoints

Resources are transient - they come and go, and have a lifecycle.  Resources can be created or manipulated by _Operations_.

An _Operation_ represents a command, or more generally an expressed intent of an application.  Operations may result in the production of a new Resource, or the change of state or disappearance of an existing Resource.  Operations have a lifecycle — they are initiated, and will eventually complete with success or failure.  They may also have intermediate states that can be expressed.

An _Artifact_ is a long-lived object that is produced by the CPAAS while processing application requests.  In general, we prefer to ship artifacts off of the system as soon as possible; i.e. to push them into the user’s world to deal with.  Artifacts include:
* Recordings
* Transcripts
* Call Detail Records

Finally, we have _Provisioned Resources_.  Provisioned Resources are long-lived resources that are provisioned to a user’s account, which an application may utilize or interact with.  Provisioned Resources include:
* DIDs
* SIP registration credentials
* Domains
* API Keys
* SIP Trunks



## CLIENT-SERVER INTERACTION

The normative server architecture will consist of an edge / SBC element, and a cluster of one or more servers; each server running drachtio + freeswitch + node.js + and a single app which provides the server CPAAS application.  The reason for drachtio and freeswitch on the same server is both simplicity of operation and scaling (i.e., cookie-cutter) as well as to simplify the use of TTS (i.e. no need to create a freeswitch module).

Additionally, there will be a redis cluster, and possibly a noSQL or SQL database.

The server connects to customer applications via web sockets (ws) or secure web sockets (css).  A customer provisions a web callback URL for each DID (or group of DIDs), such that when a call arrives on that DID, the associated callback is invoked with HTTP(S) and then upgraded to a web socket connection.  (Undetermined at this time if we will use pure web sockets or socket.io).  

Information will be exchanged between the client and server using JSON messages over websockets.  The message formats will be documented so that it is possible for a 3rd party to build a client in any language, but a node.js implementation will be provided out of the box.

 ### Sample interactions
 The brief scenarios below are intended to give a slightly fuller sense of how clients and servers connect and interact.  The descriptions below assume the client app is using the nodejs implementation mentioned above.

 #### Inbound call
 As mentioned above, an inbound call will typically invoke the provisioned HTTP(S) callback that the owner of the DID has configured.  (Note: it shall be possible to select the callback via alternate methods as well; e.g. selection by ANI would be useful on a shared DID used my multiple customers).

 The customer's responsibility is to run an http server at the provisioned URL and to accept an upgrade to websockets.  ngrok.com and zeit.co are example serverless solutions that might be used by customers to achieve this.

 After establishing the connection and upgrading to websockets, the server will send a JSON payload indicating an event: the creation of a new (incoming) call.  Information describing the call will be provided in the payload.  
 
The client is responsible for responding with an operation request of some kind: examples might include:

* say something to the caller
* collect DTMF input
* bridge the call to an outbound destination
* play music to the caller
* connect the caller to a conference
* place the caller in a queue, etc

The server will perform the operation requested, and as a result of this may send additional messages to the client, which may be either:
* events representing the creation of new resources or artifacts, or the change of state of existing resources
* the completion status (or interim completion status) of the requested operation.

In the nodejs world on the client side, receiving a completion status of success will result in the Promise returned by the operation-invoking function to be resolved; receiving a completion status of failure will result in the Promise being rejected.

Client-server interaction will continue in this fashion until the client determines that the session should be ended.  It is up to the client to determine when a session is "over", and to close the websocket connection at that point after doing any desired cleanup.  

Any calls still in progress when the client closes the connection will be allowed to continue until they are hungup by callers (possibly with a max duration applied).  Any artifacts that have not been shipped to the client at this time (i.e. when the client disconnects) will be purged from the system and will no longer be retrievable.  (TBD on what happens to other resources that may have been allocated on behalf of the application, such as conferences).

#### Outbound call
It shall be possible to initiate an outbound call as well.  To do so, a customer must make an HTTP POST request to a specific URL, something like 
```
    /v1/Accounts/{AccountSid}/Calls
```
The POST body may contain a JSON payload with any metadata that the customer would like to receive back in the next step.

Upon receiving this request, the CPAAS server will not yet generate the outbound call attempt.  Instead, it will invoke the web callback that the user has configured for outbound calls.  If a JSON payload was received in the POST request, it is included in the web callback POST request. Once again, the HTTP(S) connection is upgraded to a websocket connection.

In the nodejs world on the client side, an event is triggered indicating an intent to place an outbound call.  The client code is then responsible for responding with a request to actually create the outbound call.  The call will be connected on the near end (i.e. the CPAAS server end) to an IVR endpoint, so that the client can follow a successful connection result by performing media operations (e.g. saying something to the called party).

It is not required that the client follow through and initiate the outbound call upon receiving the callbvack; nor that it do so immediately, if it chooses to so at all.  The invocation of the callback merely represents that an intent to make a call was expressed, but the client code controls what happens next at that point.  The client, for example, may choose to first allocate a Conference Resource, and then outdial from the Conference.