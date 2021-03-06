---
Title: High Perfmance Browser Networking 读书笔记
date: 2019-04-23 
status: public
---
[High Perfmance Browser Networking](https://book.douban.com/subject/21866396/)
# 1 Latency & Bandwith

![](./_image/2019-04-23-08-54-09.jpg)
`Latency` and `Bandwith` are the critcal components of networ traffic performance:
- `latency`: The time from the source sending the packet to the destination receiving it.
- `bandwidth`: Maximum of throughtput of logical or physical communication path.

## 1.1 Latency 
Components contributing to the latency
1. Propagation delay: time for a message from the sender to the receiver.
2. Transmission delay: time for pushing all packet into the link.
3. Processing delay: time for processing packet header
4. Queuing delay: time for incoming packet for the queue until it is available.

**Speed of Light**
`Propation delay` depends on the speed of light in the medium we use. Light speed in the vacuum is quite fast but copper wire or fibreoptic will slow down the speed. `100-200` millisecond is acceptable and `300` millisecond exceeds the threshold. 

**Last-Mile latency**
Your local ISP (Internet Service Provider) need to route the cable throughtput the neighor, aggerate the singal and forward it to route node. 

## 1.2 Bandwidth
The total bandwidth of fiber link is the multiple of per-channel data rate and the number of the multiplexed channels. 

# 2 Building Block of  TCP
- RFC 791 - Internet Protocol
- RFC 793 - Transmission Control Protocol

`TCP` provides an effective abastraction of a reliable network running over an unreliable channel, hiding most of the complexity of network communication from our appliaction. including
- retransmission of lost data
- in-order delivery
- congestion control  and avoidance
- data integrity
- so on

## 2.1 Three-Way Handshake
All TCP connecton begin with a three-way handshake
- SYN
Client picks a random sequence number `x` an sends a SYN packet, which may also include additional TCP flags and options
- SYN ACK
Server increments `x` by one, pick own random sequence number `y`, append its own a set of flags and options, and dispatches the response.
- ACK
Client increments both `x` and `y` by one and completes the handshake by dispatching the last ACK packet in the handshake. 

![](./_image/2019-04-24-06-11-40.jpg)
The client can send a data packet immedidately after the ACK packet, and the server must wait fro the ACK beforee it can dispatch any data. 

## 2.2 TCP Fast Open
TCP Fast Open(TFO) allows data transfer within the SYN packet, could decrease HTTP transaction network latency by 15%. Both client and server TFO support is avaliable in Linux 3.7+ kernel. 

## 2.2 Congestion Avoidance and Control
When the nodes on the network grew,  a series of congestion collapse incidents swept the throughtput the network. To solve the probelm, TCP govens the rate with which the data can be sent in both direction. 
### 2.2.1 Flow Control
A mechanism to prevent the sender from overwhelming the recevier with data it may not be able to process.  Each side of the TCP connection adverties its own receive window (rwnd), which communicate the size of the available buffer space to hold the incoming data. 
![](./_image/2019-04-24-06-24-20.jpg)
**Window Scaling**
The original TCP specification allocate 16 bits for advertising the receviver window size. which upper bound is $2^{16}$ or 65535 bytes. RFC 1323 which allows to raise the maximum receiver window size from 65535 bytes to 1 gigabyte. 
### 2.2.2 Slow-Start
We start with a small congestion window and double it for every roundtrip. exponentail growth. 
*Time to reach the cwnd size of N*
$$
\text{Time} = \text{RTT} \times \lceil log_2(\frac{N}{\text{initial cwnd}}) \rceil
$$
If we reused TCP conneciton, it could reduces three-way handshake and slow-start time cost. 
### 2.2.3 Congestion Avoidance
Packet loss is indicative of network congestion.  We need to adjust our window to avoid the inducing more packet loss to avoid overwhelming the network.
## 2.3 Bandwidth-Delay Product
The optimal sender and receiver window sizes must vary based on the roundtrip time and the target data rate between them. 
 receve windows (rwnd) are communicated in every ACK, and the congestion widnow (cwnd) is dynamically adjusted by the sender based the congestion cotnrol and avoidance algorithms.
## 2.3 Head-of-Line Blocking
If one of the packet is lost en route to the receiver, then all subsequent packets must be held in the receiver.

## 2.4 Checklist
- Upgrade server kernel to latest version
- Ensure the cwnd size is set to 10
- Disable slow-start after idle
- Ensure that window scaling is enabled
- Eliminate redundant data transfer
- Compress transferred data
- Position servers closer to the user to reduce roundtrip times
- Reused established TCP connections whenever possible.

# 3 Building Block of UDP
UDP: User Datagram Protocol
*datagram* is oftern reserved for packets delivered via an unreliable service. No delivery guarantee, no failure notification. 
Domain Name System (DNS) depends on UDP. 

## 3.1 Null Protocol Services
IP packet
![](./_image/2019-04-25-06-52-36.jpg)

UDP header

![](./_image/2019-04-25-06-53-51.jpg)
both source port and checksum filed are optional.
 - no guarantee of message delivery: No acknowledge, retransmissions, or timeouts
 - no guarantee of order of delivery: no packet sequences number, no reording, no head-of-line blocking
 - no connection state tracking: no connection establishing or teardown state machines
 - no congestion control: no built-in client or network feedback mechanisms. 

## 3.3 Optimizing for UDP
- Application `must` tolerate a wide range  of Internet path conditions
- Application `should` control rate of transmission
- Application `should` perform congestion control over all the traffic
- Application `should` use bandwidth similar to TCP
- Application `should` back off retransmistion counter following loss
- Application `should` not send datagrams that exceed path MTU
- Application `should` handle datagram loss, duplication and reordering
- Application `should` enable IPv4 UDP checksum, and `must` enable IPv6 checksum
- Application `may` used keepalive when needed.
# 4 TSL
SSL protocol was implemented at the application layer, directly on top of TCP. 
![](./_image/2019-04-26-06-42-24.jpg)
## 4.1 Encryption, Authentication, and Integrity
- Encryption: A mechanism to obfuscate what is sent from one computer to another
- Authentication: A mechanism to verify the validity of provided identification meaterial.
- Integrity: A mechanism to delete message tampering and forgery. 

## 4.2 TLS  Handshake
Before exchanging data over TLS between client and server, they must agree on the version of  the TLS protocol. It require new packet roundtrips between client and server. 
![](./_image/2019-04-26-06-55-09.jpg)
- 56ms: The Client send a number of specifications in the plain text, such as the version of TLS protocol it is running, the list of supported ciphersuits, and other TLS options it may want to use.
- 84ms: The Server picks the TLS protocol version for futher communication, decides on a ciphersuite from a list of provided by the client, attaches it ceriticate, and sends the response to the client. 
- 112ms: The client verify the certificate provided by the server, it generates a new symmetric key, encrypts it with the server's public key.
- 140ms: the server decypts the symmetric key sent by the client, check the message integrity by verifying the MAC (message authentication code), and returns an encypted 'Finished' message back to the client. 
- 168ms: The client decypts the message with symmetric key it generated earilier, verifies the MAC. Now the tunnel is established and application data can be exchanged.

**Application Layer Protocol Negotiation**
A TLS extension that introduces support for application protocol negotiation into the TLS handshake. 
- The client appends a new `ProtocolNameList` filed, containing the list of supported application protocols, into the ClientHello message.
- The server inspects `ProtocolNameList` filed and returns a  `ProtocolName` filed indicating the selected protocol a part of ServerHello message.

## 4.3 Chain of Trust and Certificate Authorities
Example:
- Both Alice and Bob generate their own public and privete key
- Both Alice and Bob hide their respective private key
- Alice shares her public key with Bob, so do Bob.
- Alice generate a new message for Bob and signs it with her private key.
- Bob uses Alice's public key to verify the provided message signature.

Now, Alice recevied a message from Charlie, who claims to be a friends of Bob's. To prove his statement, Charlie asked Bob to sign his own public key with Bob's private key. and attached his signature with his message. Alice used Bob's public key to verify Bob did indeed sign Charlie's key and she accepts the message and trust Charlie public key. 
![](./_image/2019-04-26-07-33-44.jpg)
Authentication on the Web in your browser follows the exact same process shown. The browser specifics which certificate authorities (CAs) to truest (root CAs) and the burden is then on the CAs to verify each site they sign. 
![](./_image/2019-04-26-07-36-44.jpg)

## 4.4 TLS Record Protocol
The TLS Record  protocol is responsible for identifying types of message(handshake, alert, or data vai the `Content Type` field), as well as securing and verifying the integrity of each message.
![](./_image/2019-04-26-22-59-36.jpg)
  ## 4.5 Performance Checklist
- Get best performance from TCP
- Upgrade TLS libraries to latest release, and rebuild servers against them
- Enable and configuration session caching and stateless resumption
- Monitor your session cache hit rates and adjust configuration accordingly
- Terminate TLS sessions closer to the user to mininize roundtrip latencies
- Configuration TLS record size to fit into a single TCP segment
- Ensure that your certicicate chain does not overflow the initial congestion window
- Remove unnecessary certificate from your chain.
- Disable TLS compression on your server
- Configure SNI support on your server
- Configure OCSP stapling on your server
- Append HTTP Strcit Transport Security header

# 5 HTTP 
## 5.1 HTTP 0.9
- client request is a single ASCII character line.
- client request is terminated by a carriage return (CRLF)
- sever response is an ASCII character stream
- server response is hypertext makrup language
- connection is terminated after the document transfer is complete.

## 5.2 HTTP 1.0
- request may consist of multiple newline separated header fields
- response object is prefixed with a response status line
- response object has its own set of newline separated header fields
- response object is not limited to hypertext
- the connection between server is closed after every request

## 5.3 HTTP 1.1
- request for HTML file, with encoding, charset  and cookie metadata
- chunked response for original HTML request
- Number of octets in the chunk expressed as an ASCII hexadcimal number
- End of chunked stream response
- Request for an icon file made on same TCP connection
- Inform server that the conneciton will not be reused
- Icon response, followed by connection close


# 6 HTTP 1.1 
## 6.1 Optimizaitons
- reduce DNS lookups
- make fewer HTTP requests
- use a CDN
- add an expires header and configure etags
- Gzip assets
- avoid HTTP redirects

## 6.2 Benefits of Keepalive Connections
![](./_image/2019-05-03-15-43-58.jpg)
With keepalve, the first request incurs two roundtrip, and all following requests incurs just one roundtrip of latency.

## 6.3 HTTP Pipelining
Reuse an existing conneciotn between multiple application request, but it implies a strict first in, first out  queueing order on the client. 
![](./_image/2019-05-03-15-49-00.jpg)
**drawbacks**
- a single slow resposne blocks all requests behind it
- when processing in parallel, server must buffer pipelined resposne, which may exhaust server resource.
- a failed response may terminate the tcp connection.

## 6.4 Using mutlple tcp connection
most modern browsers  open up to six connection per host.
**pros**
- the client can dispatch up to six requests in parallel
- the server can process up to six requests in parallel
- the cumulative numebr of packetcs that can be sent in the first roundtrip is increased by a factor of six
**cons**
- addtional sockets consuming resources on the client
- competition for shared bandwidth between parallel tcp stream
- much higher implementation complexity for hanlding connection of sockets
- limited appliacation parallalism.

## 6.5 Concatenation and Spriting
- Concatentating : multiple JS or CSS files are combined into a single resource
- Spiriting: Multiple images are combined into a larger, composite image

**drawbacks**
- all resource of same type will live under the same url
- combine bundle may contain resource that not required for the current pages
- a single update to any one indivial file will require invaliding and downloading the full resource.
- both js and css are parsed and executed only when the transfer is finished.

## 6.6 Resource inlining
files such as images, audio or pdf can be inlined vai the data URI scheme.

# 7 HTTP 2.0
## 7.1 goals
- reduce latency
    - enabling full request and response multiplexing
    - minimize protocol overhead via efficient compression of http header fields
    - request prioritization
    - server push
- how data is formatted(framed) and transported between the client and server

## 7.2 Design and Technical Goals
### 7.2.1 Binary Framing  Layer
![](./_image/2019-05-04-14-29-45.jpg)
Unlike the newline delimited plaintext HTTP 1.x protocol, all HTTP 2.0 communication is split into smaller messages and frames, each of which is encoded in binary format.
### 7.3.2 Streams, Messages and Frames
- Stream: a bidirectional flow of bytes within a estableished connection
- Message: a complete sequence of frames that map to a logical message
- Frame: The smalles unit of communication in HTTP 2.0, each containing a frame header, which at minimum identifies the stream to which the frame belongs

understanding HTTP 2.0
- all communication is performed with a single TCP connection
- the stream is virtual channel within a connection, which carries bidirectional  message, each stream has a unique integer identifier
- the message is logical HTTP message, such as a request, or response, which consist of one or more frames
- the frame is the smallest unit of communication, which carries a specific type of data. 

![](./_image/2019-05-04-14-43-37.jpg)
the ability to break down an HTTP message into independent frames, interleave them, and then reassemble them on the other end is the single most important enhancement of HTTP 2.0. 

### 7.3.3 Request Prioritization
each stream can be assigned a 31-bit priority value:
- 0 represents the highest  priority stream
- $2^{31}-1$ represents the lowest priority stream

### 7.3.4 One Connection Per Origin
only one conenction is used between the client and server
- constistent priorization between all streams
- better compression through use of a single compression context
- Improved impact on network congestion due to fewer TCP connecitons
- Less time in slow-start and faster congestion and loss recovery

### 7.3.5 Flow Control
- flow control is hop-by-hop, not end-to-end
- flow control is based on window update frame
- frow control window size is updated by `WINDOW_UPDATE` frame, which specifics the stream ID and the window size
- frow control is directional: receiver may choose to set any window size.
- from control can be disabled by a receiver.

### 7.3.6 Server Push
the ability of the server to send mulitple replies for a single client request.
![](./_image/2019-05-04-15-02-36.jpg)

### 7.3.7 Header Compression
in http 1.x, all metadata is always sent as plain text and adds anywhere from 500-800 bytes of overhead per request
- http 2.0 uses `header tables` on both the client and server to track and store previously sent key-value pairs
- header table persist for the entire HTTP 2.0 conneciton and are incrementally updated both key by the client and server
- each new header key-value pair is either appended to the existing table or replaces a previous value in the table.

![](./_image/2019-05-04-15-09-24.jpg)
### 7.3.8 Efficient HTTP 2.0 Upgrade and Discovery
the client will have to use the *HTTP Upgrade* mechanism to negotiate the appropriate protocol
```
GET /page HTTP/1.1
Host: server.example.com
Connection: Upgrade, HTTP2-Settings
Upgrade: HTTP/2.0
HTTP2-Settings: (SETTINGS payload)

HTTP/1.1 200 OK
Conntent-length : 243
Content-type : text/html
(.... HTTP 1.1 response ...)
(or)
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: HTTP/2.0
(... HTTP 2.0 response ...) 
```
## 7.4 Binary Framing
all frames shares a common 8-byte header, which contains the length of the frame, its type, a bit field for flags, and 32-bit stream identifier
![](./_image/2019-05-04-15-19-19.jpg)
- the 16-bit length prefix tells us that a single frame can carry $2^{16}-1$ bytes of data, which excludes the 8-byte header size
- the 8-bit ytep field determines how the rest of the frame is interpreted
- the 8-bit flags field allow different frame types to define frame-specific messaging flags.
- the 1-bit reserved fields
- the 32-bit stream identifier uniquely identifiers the HTTP 2.0 stream. 

# 8 Optimizing Application Delivery
## 8.1 Evergreen Performance Best Practices
- Reduce DNS lookups
- Reuse TCP connecitons
- Minimize number of HTTP redirects
- Use a Content Delivery Network (CDN)
- Eliminate unnecessary resources

**Another methods**
- Cache resource on the client
- Compress assets during transfer
- Eliminate uncessary request bytes
- Parallelize request and response processing
- Apply protocol-specific optimizations

# 9 Primer On Browser Networking
Browser is a platform with specific optimization criteria, APIs, and services
![](./_image/2019-05-06-10-42-04.jpg?r=71)
## 9.1 Connection Management
sockets are organized in pools which are grouped by origin, and each pool enforces its own connection limits and security constraints. 
![](./_image/2019-05-06-10-47-09.jpg?r=70)
- origin: *application protocol*, *domain name* and *port number*
- socket pool: a group of sockets belongs to the same origin

## 9.2 Networking Security and Sandboxing
- connection limits
- request formatting and response processing
- TLS negotiation
- same-origin policy

## 9.3 Resource and Client State Caching
- automatically evaluates caching directive on each resource
- automatically expired resource when possible
- automatically manage the size of cache and resource evication

