# Bamboo Point-2-Point

A wire format for replicating [bamboo logs](https://github.com/AljoschaMeyer/bamboo) between two nodes that are connected by a reliable, ordered, bidirectional communication link (e.g. a tcp connection).

**Status: Not yet fully stable, but probably already close to the final thing.**

This protocol deals with the setting in which two nodes store local replicas of bamboo logs, and they want to synchronize and keep each other updated. We don't care *how* that situation arose, the nodes might have met on a local network, on a gossip network over the internet, one node might be running on a usb drive, and so on.

## Goals

- correctness and progress even over unstable networks
- simple, but not shying away from optimizations that require only constant space and time
- efficient usage of bandwidth
- efficient usage of computational resources in the nodes (cpu time, memory)
- nodes with vastly different computational resources (think server farm vs microcontroller) can interact meaningfully, no accidental (or deliberate for that matter) DOS attacks
- complete: no feature creep over time, the protocol is a static, self-contained artifact

## High-Level Overview

The protocol assumes two reliable, ordered, bidirectional, byte-oriented communication channels between the two endpoints. If only one such channel is available (e.g. a single tcp connection), the two independent channels can be simulated, for example via the [bymux](https://github.com/AljoschaMeyer/bymux) protocol. One of the channels is the *control* channel, the other one is the *data* channel. The control channel is used to negotiate which parts of which logs to exchange, and the data channel is used to transmit these log entries.

Replication of logs is fundamentally bidirectional: when both endpoints are interested in entries from a specific log, either of them can relay new data to the other. The replication protocol works by letting any endpoints *propose* to synchronize (parts of) a log. Once the other endpoint has *accepted* such a proposal, the endpoints can then use the data channel to bring each other up to speed with both stored data and as live data becomes available.

To keep things simple, synchronization proposals are restricted to ranges within a log, rather than allowing random-access to arbitrary subsets. Random access can be emulated by sending multiple proposals for different (small) ranges. Transmission of data also follows a strict order: the peers essentially send updates in the order in which they occur in the logs. It is possible to skip over some data, but then the peers cannot go back to transmit it at a later time. Retroactive delivery can however be emulated sending dedicated synchronization proposals for the data that had been skipped over previously.

## Session Setup

Before the regular synchronization part of the protocol begins, the two endpoits need to agree on a bit of metadata. The protocol requires a way of breaking symmetry between the endpoints. One of the endpoints is called the *proactive* endpoint, the other one is the *reactive* endpoint. How exactly these roles are assigned is irrelevant, as long as both endpoints agree on which role they hold (and they don't both hold the same role). A simple way of assigning these roles is to let the endpoint that initiated the connection be the *proactive* one and the one that accepted the connection attempt be the *reactive* one.

Next, each endpoint must convey four pieces of information to the other one (their precise meaning will be explained later):

- whether it prefers to perform greedy or safe data transmissions
- whether it is willing to accept greedy data transmissions
- the maximum size of the compressions window for data it sends
- the maximum size of the compression window for data it receives

The precise way of how this data is exchanged is irrelevant for the further proceeding of the protocol. When this information has been both sent and received, each endpoint derives for pieces of state:

- the *send mode*: greedy if the endpoint said it wanted to send greedily and the other endpoint indicated it wanted to receive greedily, safe otherwise
- the *receive mode*: greedy if the endpoint said it wanted to receive greedily and the other endpoint indicated it wanted to send greedily, safe otherwise
- the *compression window*: the minimum of the maximum compression window size this endpoint wanted to send and the maximum compression window size that the other endpoint wanted to receive
- the *decompression window*: the minimum of the maximum compression window size this endpoint wanted to receive and the maximum compression window size that the other endpoint wanted to send

The send mode of one endpoint matches the receive mode of the other endpoint, and vice versa. The same holds for compression window and decompression window.

## The Control Channel

Synchronization proposals are managed via the control channel. Since proposals incur unbounded state on the other endpoints (even if they are rejected immediately, that rejected has to be transmitted, which might involve buffering), endpoints are not allowed to send an unbounded number of proposals. Instead, a credit-based system is used. At any point, an endpoint can use the control channel to give the other endpoint *proposal credit* units, up to a maximum of 2^64-1 units. Each unit of credit can be spent to issue one synchronization proposal on the control channel.

A synchronization proposal contains the following pieces of information:

- shared information for both endpoints:
  - an *synchronization id* (just *id* for short), a number between 0 and 2^64 - 1
  - which *log* to synchronize: its author and its log id
  - the *start* sequence number of the range of entries to synchronize
  - the *lower certpool info*: a sequence number from the lower certificate pool of the *start* of the range. Synchronization begins by transmitting the metadata of the entries of in the lower certificate pool from this number up to the *start* number. See the explanation of the data channel for a more detailed explanation. This is omitted if a starting *offset* is supplied
  - the *expected starting hash* if one exists (see paragraph below for an explanation)
  - optionally: an *offset* into the payload of the start entry (may be zero). If this is present, then synchronization begins directly at that point in the payload, rather than with the metadata of the start entry.
  - optionally: an *end* sequence number for the range of entries to synchronize (inclusive), and an *upper certpool info* sequence number that indicates the smallest number in the upper certificate pool up to which to synchronize metadata after the *end* of the range has been reached (see the explanation of the data channel for a more detailed explanation).
  - a flag to indicate whether to replicate *payload only* or transmit metadata as well
  - a flag to indicate whether replication of the range should be *sparse*. In sparse replication, rather than transmitting all entries between start and end of the range, only the shortest path from end to start (in the graph of lipmaalinks and backlinks) is transmitted.
- information particular to the proposing endpoint:
  - the minimum payload size this endpoint is interested in, payloads below this size will be skipped over rather than transmitted by the other endpoint
  - the maximum payload size this endpoint is interested in, payloads below this size will skipped over rather than transmitted by the other endpoint
  - a flag whether *mandatory payload* is activated, or whether it is ok for the other endpoint to skip over payloads even though their size is between the minimum and maximum payload sizes
  - how *eager* the endpoint is, asking the other endpoint to do one of the following:
    - send all data regarding this range
    - only send metadata and payloads up to a certain size, skip over payloads above that size (but notifying that the payload is available)
    - only notify of metadata and payloads, but don't send it

The *expected starting hash* of a proposal is the target of the lipmaalink of the entry at the lower certpool info of the proposal. If the proposal specifies a starting offset, the expected starting hash is the hash of the starting entry instead. If the lower certpool info is 1, the expected starting hash is omitted (since none existst). The expected starting hash is included with a proposal so that forks can be detected even when neither endpoint has new data for the other.

Since both endpoints can issue proposals concurrently, the proactive endpoint uses even ids exclusively, and the reactive endpoint uses odd ids exclusively. The id for a new proposal must be unused. Initially, all ids are considered unused. Sending a proposal marks its id as used, until the id has been fully cancelled - the details of cancellation are explained later.

After a range has been propopsed for synchronization, both endpoints can *adjust* or *cancel* the proposal. The first adjustment by the non-proposing endpoint serves as a *confirmation* for the proposal, after the confirmation has been sent/received, data can then be transmitted to perform the synchronization.

An adjustment allows to do one or more of the following (see the documentation for proposals for more detaild explanations of the points):

- shared information for both endpoints:
  - decrease the end sequence number of the range
  - decrease the upper certpool info
- information particular to the sending endpoint:
  - set how *eager* the sending endpoint is for this synchronization
  - make payloads non-mandatory for the other endpoint
  - increase the minimum payload size the sending endpoint is interested in
  - decrease the maximum payload size the sending endpoint is interested in

A confirmation must set the eagerness of the confirming endpoint. The endpoint that didn't send the proposal starts out expecting mandatory payloads, having a minimum payload size of zero and a maximum payload size of 2^64 - 1.

It is possible for both endpoitns to concurrently propose overlapping ranges. Whenever an endpoint receives a proposal whose range overlaps with the range of a not-yet-confirmed proposal it has sent, it acts as if the proposal that begins at a larger sequence number had never been sent. In case of equal starting points, it drops the one sent by the reactive endpoint.

In addition to adjusting proposals, it is also possible to cancel them, even before they have been confirmed. When an endpoint cancels a proposal, the other endpoint also cancels it in confirmation. Both endpoints also send a special indicator over the data channel that confirms the cancellation, so there are four messages in total involved in cancellation of a proposal. Once all four of these have been sent/received, all state associated with the proposal can be released and its id can be reused if so desired. The confirmation of cancellation and the messages on the data channel are necessary to ensure correctness in all scenarios, even under concurrent sending by both endpoints over both channels. It doesn't matter whether one endpoint cancelled in confirmation of the other endpoint's confirmation, or whether both endpoints cancel concurrently for their own reasons.

Cancellation can happen for multiple reasons, and endpoints might want to react differently to different ones. The following possibilities can be expressed:

- cancellation for an unspecified reason, but the receiving endpoint should not propose this synchronization again
- cancellation due to hitting a resource limit, the other endpoint is encouraged to cancel some other proposal and then try again
- cancellation because all data in the (finite) range has been synchronized (this is used to "succesfully close" a proposal)
- cancellation because the other endpoint cancelled
- cancellation because the other endpoint expects mandatory payloads but this endpoint is unwilling to "block" on missing data
- cancellation because the expected starting hash didn't match the hash of the corresponding entry at the other endpoint
  - this cancellation also includes the metadata of the entry that didn't hash to the expected value
- cancellation because there is an end-of-log entry prior to the lower certpool info (or start of the range in case of a proposal that includes a starting offset)
  - this cancellation also includes the sequence number of that entry, the other endpoint is then expected to propose synchronizing just that sequence number so that it can learn of (and verify) the end-of-log  
- cancellation because the feed is forked prior to the lower certpool info (or start of the range in case of a proposal that includes a starting offset)
  - this cancellation also includes the fork proof: the metadata of the two entries that demonstrate the fork

This concludes the description of the flow of data on the control channel. Endpoint grant each other proposal credit, send proposals, adjust (and confirm) them and/or cancel them. After a proposal has been confirmed and before it has been fully cancelled, the data channel is used to synchronize the state of the two endpoints regarding that proposal.

## The Data Channel

The data channel is stateful: for each endpoint, there is a *current proposal* to which all transmitted data pertains. An endpoint can set its current proposal at any point.

Each proposal specifies which data needs to be transmitted, beginning with the metadata of entries in the lower certificate pool, then all the data within the range, and then the metadata of the entries in the upper certificate pool. An endpoint maintains a (conceptual) cursor into this sequence of data for each endpoint. When an endpoint sends data, its cursor is moved correspondingly. One piece of data corresponds to either an entry's metadata or its payload. For each piece of payload, the cursor is more fine-grained, it tracks byte-wise progress in the transmission of the payload.

Synchronization works by having the endpoints advance their cursor by sending data, until the end of the range is reached (if an end exists). Moving the cursor can be done in different ways:

- notifying of some number of pieces of data: advances the cursor without actually transmitting the data. This is used when the other endpoints requested notifications rather than actual data transfer in its eagerness. It is also used to move the cursor of one data past data that has already been transmitted by the other endpoint (this must be done explicitly to ensure correctness in case of concurrent data transmission by both endpoints).
- advance the cursor past one payload because it is smaller than the minimum size to be transmitted
- advance the cursor past one payload because it is larger than the maximum size to be transmitted
- advance the cursor past one payload because it is unavailable (only allowed if transmission of payloads is not mandatory)
- advance within a payload by some number of bytes, without actually sending them (this only allowed to allow the cursor of one endpoint to catch up with the cursor of the other endpoint - in such a situation the peers would switch the role of which one is "ahead" within a single payload... such situations seem rather contrived, but the protocol supports them)
- sending the metadata of an entry (the actual encoding omits some of it that can be reconstructed from context)
- sending some number of bytes of a payload

Payloads can be sent in raw form or in compressed form. The compression format is [LZFoo](https://github.com/AljoschaMeyer/lzfoo). When sending data, the maximum size of the compression window that has been negotiated at the beginning of the connection must be respected. Copy instructions may not point further into the path than the window size. The same backlog of decompressed data is shared across all different synchronization proposals, and the window is never reset. This allows to achieve good compression even for short payloads and when payloads from multiple proposals are interleaved. When raw data is transferred, it does *not* contribute to the decompression buffer.

The other options negated during connection setup are whether sending of data is greedy or safe. In safe send mode, an endpoint is only allowd to transmit metadata and payloads that passed verification - invalid data leads to the whole connection being terminated immediately. In greedy mode, data can be forwared immediately, before being verified. This is most beneficial in case of large payloads, where blocking for the full payload to arrive (to be able to verify it) would leed to long idle periods. Even in greedy mode, data must be verified once it has been fully transmitted. If an endpoint detects that it sent invalid data, it must immediately send an *apology*, both endpoints then move the cursor back by one and pretend the invalid data (either metadata or a payload) had never been transmitted. An apology can be sent in the middle of transferring a payload. An apology can not apply retroactively: if an endpoint sends invalid data and then begins sending the next piece of data for the same proposal, then the connection is terminated.

## Encodings

Encoding on the control channel:

- the byte 0 indicates granting proposal credit, followed by the amount of credit units as a VarU64
- a byte >= 1 and <= 127 (first bit is zero but at least one other bit is one): adjust
  - initial byte is followed by the request number as a VarU64
  - remaining seven bits of the first byte are flags, indicating which things to adjust. Some of them add more information by appending some additional bytes, in the order of the flags:
    - 2nd and 3rd bit: set the eagerness the sending endpoint requests:
      - `01`: only notify
      - `10`: send all metadata, and send payload up to the size that is appended as a VarU64
      - `11`: send everything
    - 4th bit: allow the other endpoint to skip payloads for this request (cannot be undone)
    - 5th bit: append the new ending seqnum as a VarU64. It gets ignored if it isn't less than the old one.
    - 6th bit: append the new upper certpool info as a VarU64. It gets ignored if it isn't less than the old one.
    - 7th bit: append the new minimum payload size of interest to the sending endpoint as a VarU64
    - 8th bit: append the new maximum payload size of interest to the sending endpoint as a VarU64
- the byte 128: cancel: don't try this request again
- the byte 129: cancel: cancelled due to resource limit, maybe cancel another request and try again
- the byte 130: cancel: reached end of the range
- the byte 131: cancel: confirming a cancel from the other endpoint
- the byte 132: cancel: cancelling because you set the mandatory payload flag but I want to advance without that payload
- the byte 133: cancel: the expected starting hash didn't match
  - followed by the metadata of the entry that didn't hash to the expected value
- the byte 134: cancel: there's an end-of-log entry prior to the lower certpool info/start of the range
  - followed by the sequence number of that entry as a VarU64
- the byte 135: cancel: feed is forked
  - followed by the metadata of the two entries that demonstrate the fork
- a byte >= 136 and <= 160: unused
- a byte >= 160 (first bit is one, 2nd and 3rd bit are not both zero): request
  - initial byte is followed by:
    - the request number as a VarU64
    - the 32 bytes of the public key of the log that this request is about
    - the log id of the log that this request is about as a VarU64
    - the sequence number at which the request starts as a VarU64
    - if the fifth bit of the initial byte is not set: the lower certpool info as a VarU64
    - the minimum payload size of interest to the sending endpoint as a VarU64
    - the maximum payload size of interest to the sending endpoint as a VarU64
    - the *expected starting hash*:
      - if the range starts with a payload (5th bit flag is set):
        - the hash of the metadata of the starting entry
      - else if the lower certpool info is not sequence number one:
        - the canonic hash of the lipmaalink target of the lower certpool info hash
      - else:
        - nothing
  - remaining seven bits of the first byte are flags. Some of them add more information by appending some additional bytes, in the order of the flags:
    - 2nd and 3rd bit: set the eagerness the sending endpoint requests:
      - `01`: only notify
      - `10`: send all metadata, and send payload up to the size that is appended as a VarU64
      - `11`: send everything
    - 4th bit: the range is finite, append the ending sequence number and the upper certpool info as VarU64s
    - 5th bit: range begins with the payload, at the offset appended as a VarU64
    - 6th bit: the request is for payloads only, no metadata
    - 7th bit: the request is sparse
    - 8th bit: payloads cannot be skipped

Invariants, drop connection if violated:

- outstanding request credit is at most 2^64 - 1
- no requests are sent if there had been no credit for them
- the proactive endpoint only creates even request numbers, the reactive endpoint only odd ones
- only create request with an unused request number
- only cancel and adjust requests that are currently active
- don't adjust or cancel a request after having already cancelled it
- the confirmation adjustment for a proposal must set an eagerness
- a range end (proposed or adjusted) is never strictly less than the start of the range
- an upper certpool info (proposed or adjusted) is never strictly less than the end of the range
- a lower certpool info (proposed or adjusted) is never strictly greater than the start of the range
- a new minimum payload size is never strictly less than the old one
- a new maximum payload size is never strictly legreater than the old one
- sequence numbers are nonzero
- may not send an unused first byte

Encoding on the data channel:

- the byte 0: switch current request number
  - followed by the request number to which to switch as a VarU64
- the byte 1: unused
- the byte 2: notify of one piece of data
- the byte 3: notify of n pieces of data
  - followed by n as a VarU64
- the byte 4: skipping payload because it is too small
- the byte 5: skipping payload because it is too large
- the byte 6: skipping payload because I don't have it
- the byte 7: skipping over n bytes of payload
  - followed by n as a VarU64
- the byte 8: uncompressed payload
  - followed by the number of bytes as a VarU64
  - followed by that many bytes of payload
- the byte 9: compressed payload
  - followed by the length of the uncompressed data as a VarU64
  - followed by the length of the compressed data as a VarU64
  - followed by that many bytes of LZFoo-compressed data
- a byte >= 10 and <= 15: unused
- the byte 16: apology
- a byte >= 17 and <= 31: unused
- the byte 32: a request has been cancelled
  - followed by the request number that has been cancelled as a VarU64
- a byte >= 33 and <= 127: unused
- the byte 128: metadata for a log entry with tag byte 0
  - if the entry has distinct lipmaalink and backlink:
    - the backlink, unless the metadata of its target has already been transmitted as part of this request
  - followed by the size of the payload as a canonical VarU64
  - followed by the hash of the payload
  - followed by the signature of the entry (64 bytes)
- the byte 129: send metadata for a log entry with tag byte 1
  - followed by the same data as the byte 128
- a byte >= 130: unused

Invariants, drop connection if violated:

- only switch to a request number for which there is an active request
- only cancel a request number for which there is an active request
- after cancelling the current request number, only switching the request number is allowed
- only send payload-related and metadata-related packets at the appropriate time (alternating)
- compression must respect the negotiated window size (and cannot be used if the window size is zero)
- all packets that advance in the log (including `notify n`) may not move past the end of the request
- transmission and skipping of payload data may not exceed the payload size specified by the corresponding metadata
- must respect the eagerness setting of the other endpoint (don't end data when only notifications are expected, don't notify when data is expected)
- may not use bytes 4 or 5 (skipping payload because it is too small or large respectively) if the corresponding metadata proves otherwise
- may not use byte 6 (skip payload because it is unavailable) if the request doesn't allow skipping payloads
- may not send an apology for valid data (although it *is* allowed to send an apology after transmitting a proper prefix of a payload, even though that prefix might have been correct)
- may not send more than one apology for the same piece of data
- in safe sending mode, may not transmit invalid (i.e. non-verifying) data
- may not send an unused first byte
- not an invariant, just a friendly reminder: don't blindly allocate memory based on payload sizes claimed by the peer, enforce a maximum allocation amount and split up the processing over time as necessary

## Running Bamboo-Point-2-Point Over Bymux

When running the protocol over [bymux](https://github.com/AljoschaMeyer/bymux), the control channel is a channel without backpressure, the data channel has backpressure. Before running bymux over the connection, the connection setup information is negotiated (greedy or safe sending and receiving, compression window sizes). Each endpoint sends a byte whose first six bits are ignored, whose seventh bit indicates whether the endpoints wants to send safely (zero) or greedily (one), and whose eigths bit indicates whether the endpoints wants to receive safely (zero) or greedily (one), followed by the sending compression window size as a [VarU64](https://github.com/AljoschaMeyer/varu64),  followed by the receiving compression window size as a VarU64. After these bytes, the bymux session then begins. It is not necessary to wait for the setup information of the other endpoint before sending point-2-point control data such as granting proposal credit.
