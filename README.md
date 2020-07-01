# Bamboo Point-2-Point

This document specifies a protocol for exchanging parts of [bamboo logs](https://github.com/AljoschaMeyer/bamboo) between two endpoints. The protocol aims to be efficient, simple, and robust. It allows to make progress even in the face of frequent connection losses, enforces rigorous backpressure, and allows interleaving of data transmissions so that large payloads do not stall out other communication.

**Status: Not yet stable, some details might still be adjusted. The overall concept feels very solid though, further changes will probably be minor. As such, implementations are very welcome.**

The protocol is first presented as a stateless request-response protocol that is conceptually simple. This description is then extended to a stateful protocol that allows immediat forwarding of newly incoming entries without having to wait for them to be polled for by a new request. This final design is structured such that an endpoint can choose to merely implement the stateless part of the protocol. Any two endpoints can interact meaningfully, regardless of whether they implement the full protocol or the stateless subset.

The protocol is built around a conceptually simple notion, the *interval*. An interval describes a subsequence of the data that makes up a log. A request specifies an interval of interest, and the response transmits the matching data. It starts out at one end of the interval, and progresses through the log entries in ascending or descending order, until either all entries in the interval have been transmitted, or the next piece of data is not available, at which point the response stops. This mechanism of delivering all the data up to the first missing piece and then completely stopping allows to keep the protocol rather simple.

## Notation and Vocabulary

Throughout this text, we will consider two communicating endpoints. We call the endpoint that sends a request `A` and the endpoint responding to it `B`.

We write `m_i` for the metadata of an entry of sequence number `i`, and `p_i` for the payload of an entry of sequence number `i`. An *item* is either a piece of metadata or a payload. We write `i_i` for an item of sequence number `i`. We define a total order on items: `i_i < i_j` if `i < j`, and `m_i < p_i`. Whenever we speak of "greater" or "lesser" items, we always refer to this order, never to anything else (e.g. never to a size in bytes). To talk about an endpoint which has a specific set of items locally available, we write `B = {i_i, i_j, ...}`.

For any natural number `n`, we denote by `v(n)` the highest entry of the certificate pool of `n`, i.e. the smallest natural number greater than or equal to `n` such that there exists a number `k` with `v(n) == ((3^k) - 1) / 2`.

We write `cert_low(n)` for the numbers on the shortest path from `n` to `1` in the graph of backlinks. We write `cert_high(n)` for the numbers on the shortest path from `v(n)` to `n` in the graph of backlinks. When it is clear from context (e.g. we are referring to items), these expressions denote the metadata of the entries of these numbers.

## Intervals

In this section, we define intervals and how to resolve which data to send in response to the request for an interval. We will define intervals step by step, in our initial definition, an interval consists of two sequence numbers, the `start` and `end`. We write such an interval as `(start, end)`.

Upon receiving a request for an interval, `B` computes the set of items that *satisfy* the interval. For our initial definition, this set contains the payloads `p_start`, `p_end` and all payloads in between, the metadata `m_start`, `m_end` and all metadata in between, `cert_low(min(start, end))`, and `cert_high(max(start, end))`.

The *response* consists of `B` sending some of those items to `A`, in a predefined order: if `start < end`, the items are sent sorted from least to greatest (an *ascending* request/response), otherwise from greatest to least (an *descending* request/responds). The response ends if either all the items have been sent, or at the first item that `B` does not have locally available.

With this definition, there is no way to request a single-number interval in ascending order. We thus define an *ascending single-number interval* for the sequence number `n`, written as `(n)`, as working just like `(n, n)` except it is ascending.

Some examples, assuming `B = {m_1, m_4, p_4, m_5, p_5, m_6, m_7, p_7, m_8}`:

| Request | Response |
|---|---|
| `(4, 4)` | `(m_4, p_4, m_1)` |
| `(4)` | `(m_1, m_4, p_4)` |
| `(1, 20)` | `(m_1)` |
| `(4, 7)` | `(m_1, m_4, p_4, m_5, p_5, m_6)` |
| `(4, 5)` | `(m_1, m_4, p_4, m_5, p_5, m_6, m_7, m_8)` |
| `(4, 1)` | `(m_4, p_4)` |
| `(5, 4)` | `()` |

### Limiting Certificate Pool Metadata

In order to prevent duplicate transmission of metadata across multiple requests, we now extend the definition of an interval with a way of limiting the number of items that are transmitted merely because they are part of some certificate pool. In addition to `start` and `end`, an interval now also contains two more natural numbers, `dist_low` and `dist_high`. Whereas before the response included `cert_low(min(start, end))` in full, it now only includes those items whose distance to `min(start, end)` in the graph of backlinks is at most `dist_low`. `cert_high(max(start, end))` is restricted to the items whose distance to `max(start, end)` in the graph of backlinks is at most `dist_high`.

Due to the nature of certificate pools, these numbers need never be larger than 84. By convention, we use 255 to indicate that the full path should be included.

We write our new intervals as `(start<dist_low>, end<dist_high>)` for ascending intervals, or `(start<dist_high>, end<dist_low>)` otherwise. If we omit the angle brackets, the corresponding value should be `255`, i.e. the whole certificate pool is being requested. For ascending single-number intervals we write `(<dist_low>n<dist_high>)`.

Some examples, again assuming `B = {m_1, m_4, p_4, m_5, p_5, m_6, m_7, p_7, m_8}`:

| Request | Response |
|---|---|
| `(6<2>, 7<0>)` | `(m_4, m_5, m_6, p_6, m_7, p_7)` |
| `(7<1>, 6<0>)` | `(m_8, m_7, p_7, m_6, p_6)` |
| `(7<2>, 6<0>)` | `()` |
| `(5<1>, 5)` | `(m_6, m_5, p_5, m_4, m_1)` |
| `(5, 5<1>)` | `()` |

### Metadata-Only Intervals

So far, every interval includes at least one payload, but sometimes it is necessary to fetch merely some part of a certificate pool. An *ascending metadata interval* consists of a `start` sequence number and a `dist_high` number, written as `(m:start<dist_high>)`, and is satisfied by all the items in `cert_high(start)` whose distance to `start` in the graph of backlinks is at most `dist_high`.

A *descending metadata interval* consists of a `start` sequence number and a `dist_low` number, written as `(m:<dist_low>start)`, and is satisfied by all the items in `cert_low(start)` whose distance to `start` in the graph of backlinks is at most `dist_low`.

### Relativity

Sometimes `A` doesn't know the exact sequence numbers of interest, they merely want "some new items" or "some old items". To this end, we allow `start` and `end` to be given as offsets relative to the least or greatest payload that `B` has locally available. `B` resolves these offsets to some absolute numbers, and then proceeds as if those numbers had been supplied for `start` and `end`, with `dist_low` and `dist_high` set to `255`.

We write an offset to the least available payload as `...n`, and it is resolved as follows: let `a` be the least sequence number such that `p_a` is locally available to `B`, and let `b` be the least sequence number greater than `a` such that `p_b` is not locally available to `B`. `...n` resolves to `min(a + n, b)`.

We write an offset to the greatest available payload as `n...`, and it is resolved as follows: let `z` be the greatest sequence number such that `p_z` is locally available to `B`, and let `y` be the greatest sequence number less than `z` such that `y` is not locally available to `B`. `n...` resolves to `max(z - n, y)`.

Just like there are ascending single-number intervals `(n)`, there are also ascending single-offset intervals `(...n)` and descending single-offset intervals `(n...)`.

Returning to our running example of `B = {m_1, m_4, p_4, m_5, p_5, m_6, m_7, p_7, m_8}`, `...0` resolves to `4`, `...1` resolves to `5`, `...2` and `...99` both resolve to `6`. `0...` resolves to `7`, `1...` and `99...` both resolve to `6`.

Offsets enable some useful queries such as "as much as possible" (`(...0, 0...)` for ascending order, `(0..., ...0)` for descending order), "the newest hundred entries" (`(100..., 0...)` ascending, `(0..., 100...)` descending) or "everything newer than `n`" (`(n, 0...)`).

## Further Request Parameters

We now have fully defined intervals, but requests also take some further parameters.

By default, `A` expects `B` to only send pieces of metadata whose signature is correct and only payloads of the correct hash and size as specified in the corresponding metadata. A response containing items that don't fulfill these criteria is considered malicious, usually leading to the connection being terminated. `A` can however specify the request to be *unverified*, in which case `B` is allowed to respond with unverified items. This can speed up the transmission of data, at the risk of wasting bandwidth on invalid data.

A request can specify a *minimum payload size* and a *maximum payload size*. When checking which items are part of the response, entries whose payload is too large or too small are treated as though they were not locally available. In particular, this means that responses end just before the first mismatching payload. These filters do not apply to the resolution of relative offsets.

A request with an absolute `start` value can specify to be an *immediate payload* request. No items from the part of the certificate pool corresponding to the `start` value are transmitted, and neither is `m_start`. Instead, `p_start` is sent immediately. An immediate payload request also specifies an offset into the payload at which to start transmitting. This feature is used to make progress even in the face of connection failures while a large payload is being transmitted.

A request can be *lazy*, in which case the response merely conveys how much data would have been sent rather than sending the data itself. The response consists of the number of complete items that would've been sent, and, if the next item would be a payload of which `B` does not have all bytes available locally, the number of consecutive bytes starting from the beginning of the payload which `B` *does* have available.

The response to a lazy immediate payload request works slightly differently. If the response includes all of the initial payload, it is simply counted as one item. Otherwise, the number of items is set to zero, and the number of bytes is the longest consecutive sequence available to `B` starting at the specified offset.

The response to a lazy request also contains the hash of the last piece of metadata that would have been sent if the response had not been lazy, if there is one.

A request that is not lazy is also called *eager*.

### Stored Fork Proofs

We call two entries from the same log a *fork* if they have identical metadata except for the payload hash or the payload size (and the signature). We call two entries `x` and `y` from the same log a *fork proof* in one of the following cases:

- `x` and `y` are a fork. The *position* of the fork is their sequence number. The *cause* of the fork is `{x, y}`.
- Two of their backlinks point to the same sequence number `n` but have different values. The *position* is `n`. The targets of those links form another fork proof. The *cause* of `x` and `y` is the cause of that fork proof.
- `x` has a backlink pointing at the sequence number of `y` but with the value unequal to the hash of `y`. The *position* is the sequence number of `y`. The targets of the backlink and `y`  form another fork proof. The *cause* of `x` and `y` is the cause of that fork proof.
- `x` is an end-of-log entry and `y` has a greater sequence number than `x`. The *position* is the sequence number of `x`. The *cause* consists of `x` and its successor on the path to `y`.

A regular response may never contain two items that together form a fork proof.

There are two different ways of handling requests for a forked log. By default, `B` may send a fork proof of the log in question at any point of the response, immediately terminating the response. `B` should (but is not obligated to) deliver the fork proof as early as possible. If `B` has multiple fork proofs to choose from, they should send one of minimal position in case of an ascending request, or one of maximal position in case of a descending request.

Alternatively, a request can specify *local* fork handling. With this parameter set, `B` is only allowed to send a fork proof of position `p` if the next item to send has sequence number greater than or equal to `p` in case of an ascending request, or less than or equal to `p` in case of the descending request. A request with local fork handling may also includes a *trust anchor* consisting of a pair of a sequence number `n` and a hash `h`.

A trust anchor limits which forks proofs `B` is allowed to send. If the request is ascending, `B` may only send fork proofs whose cause has a position strictly greater than `n` and whose cause contains an entry from which there is a path of entries to an entry of hash `h` and sequence number `n`. If the request is descending, `B` may only send fork proofs whose cause has a position strictly less than `n` and whose cause contains an entry to which there is a path of entries from an entry of hash `h` and sequence number `n`.

All these restrictions also apply to lazy requests. For a lazy request, the position against which to check whether a fork proof may be sent in local mode is the start of the interval. If there is a fork proof beyond the start of the interval, the lazy response first reports the number of items until that position, the hash of the newest items if required, and then the fork proof.

### Partial Fork Proofs

A request can include up to four *expected hashes* of certain pieces of metadata: the skiplink target of the least piece of metadata that would be part of a full response, the metadata corresponding to the least payload that would be part of a full response, the metadata corresponding to the greatest payload that would be part of a full response, and the greatest piece of metadata that would be part of a full response. These can only be specified in cases where their sequence number is uniquely determined, i.e. if their position does not depend on resolving an offset.

`B` might have an item that would form a fork proof together with an entry of the expected hash at that sequence number. This item is called a *partial fork proof*. Unless a trust anchor specifies to ignore forks at this position, `B` can terminate the response by sending the partial fork proof - immediately in default fork handling, or at the appropriate point in local fork handling. Note how `A` thus ultimately obtains a full fork proof, but `B` doesn't. In fact, `A` might have lied about an expected hash. If `B` wants to know for sure, they can make a request for the item in question.

## Canceling Requests

`A` might lose interest in the response to a request before that response has arrived. To this end, `A` can *cancel* any previously issued request. `A` cannot consider that request to be canceled immediately though, since `B` might have sent the response concurrently. Thus, upon receiving a cancellation, `B` sends the empty response as confirmation. In some sense, `A` issuing a cancellation doesn't really cancel anything, it merely prompts `B` to respond immediately.

## Stateful Synchronization

A request/response protocol as described above is stateless in the sense that a request can be processed and then be forgotten about. Statelessness leads to a problem though: data can only be sent if it has been requested. If an endpoint wants to always keep up to date with a log, it needs to constantly send new requests, a technique called *polling*. It would be nice if an endpoint could ask in advance to receive new items as they become available. This requires to manage state across requests, endpoints need to remember which new items to forward.

Interestingly enough, our protocol can easily be extended to become stateful: whenever `B` would terminate a response because they lack an item, they instead pause the response, resuming it once that item becomes available - possibly to immediately be paused again afterwards. The only other change is to merely pause responses for intervals whose `end` is specified as an offset after the end has been reached, and to reevaluate the offset as new items become available. For example an ascending interval ending at `0...` would pause once the newest item has been sent, and if an even newer item became available, the response would be resumed.

In the stateful case, it can happen that both endpoints have a paused request that would resume with the same item(s). Suppose `A` then obtains the item(s) and sends them to `B`. It would not make sense for `B` to then send them back to `A` again. Instead, `B` has to send a confirmation in place of the item(s) and then advances which item should be sent next. This confirmation works similarly to a lazy response: it contains the number of fully received items (with the first item of an immediate payload request counting as fully received if its last byte has been transmitted), and how many bytes of a partially transmitted last item have been received. The confirmation mechanism is necessary to correctly handle cases where both endpoints send the same item(s) concurrently. In the same setting, if `B`'s request was lazy, `B` would still send a confirmation to `A` and advance which item to send next upon receiving a notification that `A` had new items available. The confirmation would mirror `A`'s notification.

## The Actual Protocol

The protocol assumes a reliable, ordered, byte-oriented channel between the two endpoints. Endpoints send each other messages. An endpoint reads bytes until a full message has been received, checks that it is valid, acts on it, and then waits for the next message. An invalid message is always handled by terminating the connection. We now describe the precise protocol by listing the state maintained by the endpoints, the different kinds of messages, how they are encoded, their validity criteria, and how to react to them.

### Global Connection State

To avoid overloading endpoints, there is a backpressure mechanism in place: some types of messages - requests, adjustments and responses - may only be sent after permission has been granted. This is implemented as a credit system. Sending a request consumes one *request credit*, and may only be done if at least one credit is available. Endpoints start out with zero request credit available, but can grant each other request credit via messages, up to a maximum of `2^64 - 1`.

There is a similar mechanism for responses, but instead of limiting the number of response messages, *response credit* limits the number of metadata and payload bytes that may be sent. Details on credit handling are given in the sections on the messages impacted by it. For now, it suffices to state that each endpoint maintains four 64-bit counters for credit: `credit_request_mine`, `credit_request_yours`, `credit_response_mine` and `credit_response_yours`. All of these are initially set to zero.

Response data is sent in a stateful way. Rather than tagging every item with information on which request it pertains to, there is a *active request* and all incoming response data pertains to that request. An *active request message* changes the currently active request. An endpoint keeps track of `active_request_mine` and `active_ request_yours`. Both are *request ids*, which are simply 64-bit integers, and are initially set to zero. A request id can either be *fresh* or *used* for an endpoint. Initially, all request ids are fresh for both endpoints. The descriptions of the message types below indicate how certain ids change from fresh to used and back again. Certain messages are invalid if an id of the wrong state is supplied.

The remainder of the state of the connection consists of the various requests opened by either endpoint.

### Messages

There are seven different kinds of messages:

- Request messages indicate new interest in a part of a log.
- Response messages transmit items, item sizes, and indicate the end of a response.
- Active request messages change to which request the following response messages pertain.
- Cancellation messages ask the other endpoint to immediately end a particular response.
- Adjustment messages can efficiently update whether a request is lazy or eager.
- Request credit messages allow the other endpoint to send more request messages.
- Response credit messages allow the other endpoint to send more response messages.

In the following definition of the encoding for the various message types, 64-bit integers are always encoded as [VarU64s](https://github.com/AljoschaMeyer/varu64), and hashes are always encoded as [YAMF-Hashes](https://github.com/AljoschaMeyer/yamf-hash). `dist_low` and `dist_high` are always encoded as a single byte.

#### Request Messages

A request message consists of the log id to be requested, the request id, the interval of interest, the fork handling mode (default, local, or local with a trust anchor), an optional minimum payload size, an optional maximum payload size, an optional immediate payload offset, whether responses should be verified, and whether responses should be lazy.

The request id does not necessarily have to be fresh for the sending endpoint (i.e. this is not a validity criterium that must be checked), but sending a used one might lead to receiving garbage responses. If the request id was fresh, it then becomes used.

When sending a request message, an endpoint decrements `credit_request_mine`. If it would fall below zero, the message may not be sent. When receiving a request message, an endpoint decrements `credit_request_yours`. If it would fall below zero, the message is invalid.

The encoding of a request message consists of two bytes of metadata, followed by some *base data*: the request id, followed by the log id, encoded as the raw public key (32 bytes) followed by the log number. Depending on the two metadata bytes, different pieces of data follow the base data. If multiple pieces of data follow, their order is the order in which they are presented in the following list:

- The first bit of the two metadata bytes is always 0, to indicate that the message is a request message.
- The second and third bit indicate the fork handling mode:
  - `00` indicates default fork handling
  - `01` indicates local fork handling without a trust anchor
  - `10` indicates local fork handling with a trust anchor (sequence number and hash), the base data is followed by the sequence number, followed by the hash
  - `11` makes an invalid message
- The fourth bit is set to `1` if there is a minimum payload size for this request, `0` if there isn't. If it is set to `1`, the base data is followed by the minimum payload size.
- The fifth bit is set to `1` if there is a maximum payload size for this request, `0` if there isn't. If it is set to `1`, the base data is followed by the maximum payload size.
- The sixth bit is set to `1` if the request is an immediate payload request, `0` if it isn't. If it is, the base data is followed by the offset at which the transmission of the payload bytes should begin.
- The seventh bit is set to `1` if responses must be verified, `0` if they may be unverified.
- The eighth bit is set to `1` if the request is lazy, `0` if it isn't.
- The second byte of metadata concerns the interval of the request.
  - If the interval is a regular interval (consisting of a start and an end), the ninth and tenth bit are set to `0`.
    - If the start of the interval is an absolute value:
      - The 11th bit is set to `0`, and the base data is followed by the absolute `start` sequence number and either `dist_low` if the interval is ascending or `dist_high` if the interval is descending.
      - If the 12th bit is set to `1`, the base data is followed by a hash. If the interval is ascending, this hash is the expected hash for the skiplink target of the least piece of metadata that would be part of a full response. If the interval is descending, this hash is the expected hash for the greatest piece of metadata that would be part of a full response.
      - If the 13th bit is set to `1`, the base data is followed by a hash. If the interval is ascending, this hash is the expected hash for the metadata corresponding to the least payload that would be part of a full response. If the interval is descending, this hash is the expected hash for the metadata corresponding to the greatest payload that would be part of a full response.
    - If the start of the interval is an offset relative to the least available payload, the 11th bit is set to `1` and the 12th and 13th bit are set to `0`, and the base data is followed by the offset.
    - If the start of the interval is an offset relative to the greatest available payload, the 11th bit is set to `1`, the 12th bit is set to `0`, and the 13th bit is set to `1`, and the base data is followed by the offset.
    - If the end of the interval is an absolute value:
      - The 14th bit is set to `0`, and the base data is followed by the absolute `end` sequence number and either `dist_high` if the interval is ascending or `dist_low` if the interval is descending.
      - If the 15th bit is set to `1`, the base data is followed by a hash. If the interval is descending, this hash is the expected hash for the metadata corresponding to the least payload that would be part of a full response. If the interval is descending, this hash is the expected hash for the metadata corresponding to the greatest payload that would be part of a full response.
      - If the 16th bit is set to `1`, the base data is followed by a hash. If the interval is ascending, this hash is the expected hash for the skiplink target of the least piece of metadata that would be part of a full response. If the interval is ascending, this hash is the expected hash for the greatest piece of metadata that would be part of a full response.
    - If the end of the interval is an offset relative to the least available payload, the 14th bit is set to `1` and the 15th and 16th bit are set to `0`, and the base data is followed by the offset.
    - If the end of the interval is an offset relative to the greatest available payload, the 14th bit is set to `1`, the 15th bit is set to `0`, and the 16th bit is set to `1`, and the base data is followed by the offset.
  - If the interval is an ascending single-number interval:
    - The ninth bit is set to `1` and the 10th bit is set to `0`, and the base data is followed by the sequence number.
    - If the ascending single-number interval is given by an absolute number:
      - The 11th and 12th bit are set to `0`, and the base data is followed by `dist_low` and `dist_high`.
      - The 13th bit is set to `1` if an expected hash for the skiplink target of the least piece of metadata that would be part of a full response is supplied, in which case the base data is followed by that hash. Otherwise, it is set to `0`.
      - The 14th bit is set to `1` if an expected hash for the metadata corresponding to the sequence number is supplied, in which case the base data is followed by that hash. Otherwise, it is set to `0`.
      - The 15th bit is set to `1` if an expected hash for the greatest piece of metadata that would be part of a full response is supplied, in which case the base data is followed by that hash. Otherwise, it is set to `0`.
      - The 16th bit may be set arbitrarily and must be ignored.
    - If the ascending single-number interval is given by an offset relative to the least available payload, the 11th bit is set to `1` and the 12th bit is set to `0`. Bits 13 to 16 may be set arbitrarily and must be ignored.
    - If the ascending single-number interval is given by an offset relative to the greatest available payload, the 11th bit is set to `1` and the 12th bit is set to `1`. Bits 13 to 16 may be set arbitrarily and must be ignored.
    - If the 11th bit is set to `0` and the 12th bit set to `1`, the message is invalid. Bits 13 to 16 may be set arbitrarily and must be ignored.
  - If the interval is a metadata interval:
    - The ninth bit is set to `1` and the 10th bit is set to `1`, and the base data is followed by the sequence number.
    - If it is a descending interval:
      - The 11th bit is set to `0`, and the base data is followed by `dist_low`.
      - The 12th bit is set to `1` if an expected hash for the skiplink target of the least piece of metadata that would be part of a full response is supplied, in which case the base data is followed by that hash. Otherwise, it is set to `0`.
      - The 13th bit is set to `1` if an expected hash for the metadata corresponding to the sequence number is supplied, in which case the base data is followed by that hash. Otherwise, it is set to `0`.
    - If it is an ascending interval:
      - The 11th bit is set to `1`, and the base data is followed by `dist_high`.
      - The 12th bit is set to `1` if an expected hash for the greatest piece of metadata that would be part of a full response is supplied, in which case the base data is followed by that hash. Otherwise, it is set to `0`.
      - The 13th bit is set to `1` if an expected hash for the metadata corresponding to the sequence number is supplied, in which case the base data is followed by that hash. Otherwise, it is set to `0`.
    - Bits 14 to 16 may be set arbitrarily and must be ignored.

#### Response Messages

Response messages are sent to satisfy requests. They come in three variants: *eager response messages* transmit some number of bytes which encode requested items, *lazy response messages* notify of the availability of some items, and *end response messages* signal the end of a response even though more items could have followed.

##### Eager Response Messages

An eager response message is encoded as the byte `0x80`, sometimes followed by a sequence number (see the next paragraph), followed by the number of bytes it contains, and then that many bytes. When sending such a message, an endpoint decreases `credit_response_mine` by that much, and if it would fall below zero, the message may not be sent. When receiving such a message, an endpoint decreases `credit_response_yours` by that much, and if it would fall below zero, the message is invalid. An eager response message is also invalid if it pertains to a lazy request.

If no prior eager or lazy response messages have been sent for the currently active request and the start of the request has been given as a relative offset, then the initial `0x80` byte is followed by the sequence number of the payload to which the start offset has been resolved.

The actual bytes that are being sent depends on the request. The order in which the satisfying items are being sent is predetermined, so there is no need for any more transmission metadata. Endpoints merely need to keep track of the current state of the item transmission for each of their requests. An endpoint is allowed to chop up responses in any way it sees fit, for example it might send the first three bytes of metadata, but then start sending completely unrelated messages, even responding to other requests, before continuing transmission where it left of.

Payloads are transmitted verbatim, but metadata items omit some information which can be reconstructed from context, e.g. the log id. The bytes to transmit for a piece of metadata are defined as follows:

- The tag byte of the entry.
- The value of the skiplink, unless it is equal to the predecessor link or the metadata of its target has already been transmitted as part of this response or its value has been given as an expected hash.
- The value of the predecessor link, unless the metadata of its target has already been transmitted as part of this response or its value has been given as an expected hash.
- The size of the payload.
- The hash of the payload.
- The signature of the entry.

##### Lazy Response Messages

A lazy response message notifies the other endpoint of some number of items. It consists of the number of full items, and the size of the partially available trailing payload (which may frequently be zero). A lazy response message is invalid if it pertains to an eager request, unless the responding endpoint has received the data which the lazy response message indicates from the requesting endpoint itself via another request.

A lazy response message furthermore carries the hash of the last piece of metadata that would have fully been part of the response if it was eager, if there is one.

A lazy response message is encoded as the byte `0x90`, sometimes followed by a sequence number (see the next paragraph), followed by the number of full items and the number of trailing payload bytes, followed by the hash if there is one.

If no prior eager or lazy response messages have been sent for the currently active request and the start of the request has been given as a relative offset, then the initial `0x90` byte is followed by the sequence number of the payload to which the start offset has been resolved.

##### End Response Messages

An end response message terminates a response. When sending such a message, the request id `active_request_mine` becomes fresh for the sending endpoint. When receiving such a message, the request id `active_request_yours` becomes fresh for the sending endpoint.

An end response message contains some information on why the response has been ended. All end response messages begin with a byte whose first four bits are set to `0xa`.

The 5th and 6th bit indicates different reasons for the end response message:

- `00` indicates that the response has been ended because a full fork proof is available. If the fork handling mode of the request does not allow transmitting a fork proof at this position, the message is invalid. The byte is followed by the two pieces of metadata (omitting the log id) that constitute the fork proof, each encoded as follows:
  - The tag byte of the entry.
  - The sequence number of the entry.
  - The value of the skiplink.
  - The value of the predecessor link.
  - The size of the payload.
  - The hash of the payload.
  - The signature of the entry.
- `01` indicates that the response has been ended because a partial fork proof is available. If the fork handling mode of the request does not allow a fork proof at this position, the message is invalid. The byte is followed by the piece of metadata that constitutes the partial fork proof, encoded as above.
- `10` indicates that the response has been ended because of a cancellation or adjustment message.
- `11` indicates that the response has been ended for some other reason.

The 7th bit can modify request credit: If it is set to `1`, the receiving endpoint is being granted one request credit.

If the 8th bit is set to `1`, the message is immediately followed by a request id, which becomes the new active request id. The message is invalid, if the id is fresh.

A request for an interval with an absolute end value is automatically ended by an eager or lazy response message that reaches the end of the interval. The request id becomes fresh automatically, so sending an explicit end message would actually be invalid. An eager or lazy response message which overshoot the end of the interval is also invalid.

#### Request Credit Messages

A request credit message consists of a single 64-bit integer. When sending such a message, the endpoint increases `credit_request_ yours` by the sent amount. When receiving such a message, the endpoint increases `credit_request_mine` by the received amount. If it would overflow (become greater than `2^64 - 1`), the message is invalid.

A request credit message is encoded as the byte `0xb0` followed by the amount of granted credit.

#### Response Credit Messages

A response credit message consists of a single 64-bit integer. When sending such a message, the endpoint increases `credit_response_ yours` by the sent amount. When receiving such a message, the endpoint increases `credit_response_mine` by the received amount. If it would overflow (become greater than `2^64 - 1`), the message is invalid.

A response credit message is encoded as the byte `0xc0` followed by the amount of granted credit.

#### Cancellation Messages

A cancellation message consists of a single 64-bit integer, the request id to be canceled. When receiving such a message, the endpoint should close the corresponding request (details are given in the description of response messages). If the id is currently fresh for the other endpoint, the message is invalid.

A cancellation message is encoded as the byte `0xd0` followed by the request id.

#### Active Request Messages

An active request message consists of a  64-bit integer, the *offset*, and a mode, either addition or subtraction. When sending such a message in addition mode, the endpoint adds the offset to `active_request_mine`, in subtraction mode it is subtracted instead. When receiving such a message in addition mode, the endpoint adds the offset to `active_request_yours`, in subtraction mode it is subtracted instead. If the computation overflows or underflows, the message is invalid. If the id is currently fresh for the other endpoint, the message is invalid.

An active request message in addition mode is encoded as the byte `0xe0` followed by the request id offset. An active request message in subtraction mode is encoded as the byte `0xe8` followed by the request id offset.

#### Adjustment Messages

*Adjustment messages* have not been mentioned in the description of the protocol, this is because they are merely an efficient shortcut for canceling a request and creating a new one that differs only in its laziness.

Each adjustment message specifies two request ids: The *old* one (must be used or the message is invalid), and the *new* (should be fresh, otherwise leads to garbage data). Upon receiving an adjustment message, an endpoint should immediately end the old request, as if it had been canceled. It also acts as if it had just received a request message that would lead to a request identical to the old one, except for the id and the adjustment, as described in the following paragraphs. It may however only set the current request to the new one after it has sent the cancellation for the old one - any message setting the current request to the new id too early is invalid.

If the old request was lazy, the new request is eager and vice versa. The new request begins at exactly the position in the stream of items where the old message was canceled, unless a different position has been specified:

A toggle adjustment message can optionally specify a sequence number `n` and an optional offset `i` into a payload. `n` and `i` specify a position in the stream of item data: If no `i` is supplied, it is the position of the metadata corresponding to the entry of sequence number `n`. If `i` is supplied, it gives an offset into the payload for the entry of sequence number `n`.

In the following description, we compare the supplied position with the current position in the stream of items of the old request. If no such position is known (the request has a relative start offset and no eager or lazy response message has been sent yet), the supplied position counts as greater in case of an ascending request, and as lesser in case of a descending request. Otherwise, the two positions are compared by which item would have come first in an ascending/descending request.

The state for the new request depends on the old one at the time of sending its cancellation:

- If the old request was lazy and the position in the stream of items is less than or equal to the supplied position, the new request starts at the supplied position.
- If the old request was lazy and the position of the stream of items is greater than the supplied position, the new request starts at the same position where the old message left off, but automatically switches to eager at the supplied position.
- If the old request was eager and the position of the stream of items is less than or equal to the supplied position, the new request starts at the the same position where the old message left off.
- If the old request was eager and the position of the stream of items is greater than the supplied position, the new request starts at the same position where the old message left off, but automatically switches to lazy at the supplied position.

If the old request had no known current position, the supplied position also acts as the start position of the new request, with a `dist_low` of zero if it is ascending, or a `dist_high` of zero if it is descending. If the old request had a known position, and the supplied position was not the position of an item in the response item stream, the message is invalid.

An adjustment message is encoded as the byte `0xf0` if it does not specify a position, as `0xf8` if it does but the position does not contain a payload offset, or as `0xfc` if it does. Both are followed by the old request id and the new request id. If the first byte is `0xf8` or `0xfc`, it is then followed by the sequence number. if the first byte is `0xfc`, it is then followed by the offset.
