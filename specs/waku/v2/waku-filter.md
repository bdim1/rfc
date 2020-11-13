---
title: Waku
version: 2.0.0-beta2
status: Draft
authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>, Hanno Cornelius <hanno@status.im>
---

# Table of Contents

- [Abstract](#abstract)
- [Content filtering](#content-filtering)
  * [Rationale](#rationale)
  * [Protobuf](#protobuf)
- [Changelog](#changelog)
- [Copyright](#copyright)
- [References](#references)

# Abstract

`WakuFilter` is a protocol that enables subscribing to messages that a peer receives. This is a more lightweight version of `WakuRelay` specifically designed for bandwidth restricted devices. This is due to the fact that light nodes subscribe to full-nodes and only receive the messages they desire.

# Content filtering

**Protocol identifier***: `/vac/waku/filter/2.0.0-beta1`

Content filtering is a way to do [message-based
filtering](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern#Message_filtering).
Currently the only content filter being applied is on `contentTopic`. This
corresponds to topics in Waku v1.

## Rationale

Unlike the `store` protocol for historical messages, this protocol allows for
native lower latency scenarios such as instant messaging. It is thus
complementary to it.

Strictly speaking, it is not just doing basic request response, but performs
sender push based on receiver intent. While this can be seen as a form of light
pub/sub, it is only used between two nodes in a direct fashion. Unlike the
Gossip domain, this is meant for light nodes which put a premium on bandwidth.
No gossiping takes place.

It is worth noting that a light node could get by with only using the `store`
protocol to query for a recent time window, provided it is acceptable to do
frequent polling.

## Protobuf

```protobuf
message FilterRequest {
  bool subscribe = 1;
  string topic = 2;
  repeated ContentFilter contentFilters = 3;

  message ContentFilter {
    repeated string contentTopics = 1;
  }
}

message MessagePush {
  repeated WakuMessage messages = 1;
}

message FilterRPC {
  string requestId = 1;
  FilterRequest request = 2;
  MessagePush push = 3;
}
```

#### FilterRPC

A node MUST send all Filter messages (`FilterRequest`, `MessagePush`) wrapped inside a
`FilterRPC` this allows the node handler to determine how to handle a message as the Waku
Filter protocol is not a request response based protocol but instead a push based system.

The `requestId` MUST be a uniquely generated string. When a `MessagePush` is sent
the `requestId` MUST match the `requestId` of the subscribing `FilterRequest` whose filters
matched the message causing it to be pushed.

#### FilterRequest

A `FilterRequest` contains an optional topic, zero or more content filters and
a boolean signifying whether to subscribe or unsubscribe to the given filters.
True signifies 'subscribe' and false signifies 'unsubscribe'.

A node that sends the RPC with a filter request and `subscribe` set to 'true' 
requests that the filter node SHOULD notify the light requesting node of messages
matching this filter.

A node that sends the RPC with a filter request and `subscribe` set to 'false'
requests that the filter node SHOULD stop notifying the light requesting node
of messages matching this filter if it is currently doing so.

The filter matches when content filter and, optionally, a topic is matched.
Content filter is matched when a `WakuMessage` `contentTopic` field is the same.

A filter node SHOULD honor this request, though it MAY choose not to do so. If
it chooses not to do so it MAY tell the light why. The mechanism for doing this
is currently not specified. For notifying the light node a filter node sends a
MessagePush message.

Since such a filter node is doing extra work for a light node, it MAY also
account for usage and be selective in how much service it provides. This
mechanism is currently planned but underspecified.

#### MessagePush

A filter node that has received a filter request SHOULD push all messages that
match this filter to a light node. These [`WakuMessage`'s](./waku-message.md) are likely to come from the
`relay` protocol and be kept at the Node, but there MAY be other sources or
protocols where this comes from. This is up to the consumer of the protocol.

A filter node MUST NOT send a push message for messages that have not been
requested via a FilterRequest.

If a specific light node isn't connected to a filter node for some specific
period of time (e.g. a TTL), then the filter node MAY choose to not push these
messages to the node. This period is up to the consumer of the protocol and node
implementation, though a reasonable default is one minute.

---

# Changelog

### 2.0.0-beta2

Initial draft version. Released [2020-10-28](https://github.com/vacp2p/specs/commit/5ceeb88cee7b918bb58f38e7c4de5d581ff31e68)
- Fix: Ensure contentFilter and contentTopic are repeated fields, per implementation
- Change: Add ability to unsubscribe from filters. Make `subscribe` an explicit boolean indication. Edit protobuf field order to be consistent with libp2p.

### 2.0.0-beta1

Initial draft version. Released [2020-10-05](https://github.com/vacp2p/specs/commit/31857c7434fa17efc00e3cd648d90448797d107b)

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

1. [Message Filtering (Wikipedia)](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern#Message_filtering)

2. [Libp2p PubSub spec - topic validation](https://github.com/libp2p/specs/tree/master/pubsub#topic-validation)