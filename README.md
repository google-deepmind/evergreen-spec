# Evergreen Protocol

We are publishing a protocol for bidirectional streaming RPCs, informed by our
experience building GenAI applications. Our goal is to create a protocol that
can support a wide range of existing, planned, and unforeseen use cases in the
long run, while being immediately useful for generative AI use cases today.

We would appreciate any feedback on the design of the protocol, compelling use
cases, and things we haven't considered. To reach out to us, either use the [Github issues](https://github.com/google-deepmind/evergreen-spec/issues/new) tool or send us an email at evergreen-oss-team@google.com.

- [Data model description](#data-model-description)
    - [Nodes, chunks, and actions](#nodes-chunks-and-actions)
    - [Identification](#identification)
    - [Lifetime](#lifetime)
    - [Ordering](#ordering)
    - [Boundaries](#boundaries)
    - [Mime type](#mime-type)
    - [Chunk payloads](#chunk-payloads)
    - [Generating actions](#generating-actions)
    - [Error handling](#error-handling)
    - [Default actions](#default-actions)
    - [Default values](#default-values)
- [Future extensions / decisions](#future-extensions--decisions)
- [Examples](#examples)

## Data model description

### Nodes, chunks, and actions

Evergreen is a protocol representing a logical **session** between a client and
a server. Session is a medium where **nodes** and **actions** are both
**produced** and **consumed**.

Each leaf node (node without children) is a typed sequence of bytes. The bytes
are split up into chunks and can be incrementally delivered over a transport
like bidirectional streaming gRPC. The chunk boundaries here do not have
semantic meaning--a node where all chunks in the same leaf have been
concatenated into a single chunk represents the same data as the original node.

Non-leaf nodes allow us to deliver a sequence of leaf nodes, concurrently and in
an out-of-order fashion. For example, a Wikipedia page can be sent as a root
node, with several text, image and video child nodes. Text and image nodes would
all be leaf nodes containing their respective chunk. In contrast, each video
node is then further transformed into a sequence of leaf nodes, each containing
the chunk for a single video frame.

Conceptually, each input or output is a chunk of data, or a list of chunks.
However, instead of sending chunks over the wire as a simple list, we use a tree
of nodes for input and output to (a) share common content across different
actions, and (b) stream different chunks out of order. The input and output of
an action is the result of flattening the respective tree of nodes into a list
of chunks.

Note that, even though conceptually similar to trees, nodes can form a directed
acyclic graph (DAG) due to aliasing and sharing nodes in the hierarchy. In this
doc, we use the term tree to describe a hierarchy of nodes instead of DAG for
simplicity.

Nodes are **identified using a unique ID** within the session. The ID is
determined by the producer. In addition to ID, each chunk has a **mime type**
indicating how the payload should be interpreted.

Both nodes and chunks can be streamed.

**Actions** are a predefined set of functions (e.g., `GENERATE`) that can be
executed over a set of nodes, and produce nodes. The inputs and outputs are
named, similar to
[named parameters](https://en.wikipedia.org/wiki/Named_parameter) in programming
languages. The acceptable input and output parameters, the type of outputs, and
the configuration fields are defined by the action. We do not impose any
limitations on that.

### Identification

Nodes are **identified using a unique ID** within the session. These IDs are
determined by the producers. Nodes and chunks cannot be shared across sessions.
However, a chunk can reference external storage, which can implicitly result in
sharing state across sessions.

IDs should not bear semantics: requests should remain valid and semantics should
not change if each ID is replaced with a randomly picked unique string.
Input/output names should be used for semantic representation instead.

### Lifetime

Nodes are bound to sessions and they will be deleted as soon as the session
ends. Within a session, nodes and chunks will persist for the length of the
session. As soon as a session expires, all nodes and chunks within that session
are expired too.

### Ordering

Children of a node can be generated and/or received in any order. The ordering
within a node is denoted using a 0-based sequence number. Sequence numbers
should be unique within a node. If a consumer receives more than one
`NodeFragment` message with the same sequence number with the same parent node,
the consumer should accept the first received `NodeFragment` message with that
`seq` number, and ignore the rest. The consumer should not produce an error, to
facilitate easier implementation of retries. If nodes must be processed in an
ordered fashion, it is the responsibility of the consumer to order them upon
receipt.

A node can reference nodes that are not received yet as their children. The
consumer should form proper data dependency barriers to support that. A node can
be a descendant of more than one node in a session, and a named node can be used
as an input to more than one action.

While nodes cannot transitively include themselves, we still support deep
nesting. Implementations should have a reasonable limit for the maximum nesting
level.

Implementations are encouraged but not required to send actions before input nodes, and parent nodes before their children, to enable potential
pipelining.<sup>1</sup> Regardless of when a node is received, the receiver must
hold on to the given node until the end of the session. Note that a session may
be unilaterally aborted by the receiver, which will result in dropping all nodes
in that session.

### Boundaries

Messages for a single node contain a `continued` flag to indicate that the node
is not finalized (there are more fragments of the same node). If the receiver
receives any `NodeFragment` message with a sequence number greater than the one
of the final message of the node (a `NodeFragment` message in which `continued`
is False), the session will be aborted with an error.

Since the `continued` flag is by default false, any `NodeFragment` message
received without this flag is considered final.

### Mime type

Each chunk has a **mime type**, which can inform the consumer how to
interpret/decode the bytes in that chunk. We require the message with `seq=0` to
populate the metadata field, and that no other chunks contain metadata. In the
same node, we assume all messages have the same metadata as the one sent in
sequence number 0. If a message other than sequence 0 contains metadata, the
session will be aborted.

Note that the payload of a chunk can be either an inline data or a reference to
an external source. For external sources, mime type indicates the type of the
content in the external storage.

Non-leaf nodes can directly or transitively include leaf nodes with different
mimetypes. Note that nodes themselves do not have a payload, and hence no mime
type.

### Chunk payloads

Chunks can have inline binary `data` or an external `ref` as their payloads.
External references are URIs. The supported URIs depend on the implementation.
Regardless of having inline or external payload, leaf nodes are built out of
concatenation of chunks, ordered by sequence number.

### Generating actions

For each action, the producer must provide a list of inputs and list of outputs.
Each input and output entry is a pair of (parameter name, node ID). The
parameter name is the name of an input or output parameter of the invoked
action. This is analogous to named arguments in programming languages, without
any particular ordering. For example, to generate a video from text, the user
can invoke `generate(input=[(text=prompt1)], output=[(video=frames1)]` which
uses the `prompt1` node as the text input, and puts the model's output video in
the `frames1` node.

The producer should generate chunks with the root node IDs provided in the
output `NamedParameters`. The provided node IDs in the output must be new and
must not be reused. If the sender does not provide a `NamedParameter`
corresponding to a supported output on the action type, it signals that the
sender will not use that particular field of the output and it need not be
populated by the server.

When an action is received, the consumer can start processing an action at any
time (e.g., it can wait for all input nodes to be received, or can start
processing the action immediately). Outputs from previous actions can be used as
inputs without re-sending them.

### Error handling

Actions may not successfully execute for various reasons, including malformed or
missing input parameters. When an action fails, we abort a session.

There are other cases where the implementation may prefer to abort the session.
For example, when the user sends invalid chunks, or when the implementation
detects abuse.

### Default actions

Note that we expect servers (including models) to provide different actions. The
action name, input names and types, output names and types, and accepted
configurations should be agreed upon between the producer and the consumer.

For existing applications, we believe most generative models will provide a
`GENERATE` method. We envision that, in the near future, agent infrastructure
will provide various actions as a menu of functions.

### Default values

* Sequence number (`seq`) is 0 if sequence number is not provided.
* `continued` is false if not provided.

## Future extensions / decisions

### Libraries

We may provide thick libraries wrapping the bidirectional streaming protocol to
avoid exposing users to the complexity of the base protocol.

### DROP command

We may consider a `DROP(node)` action that will drop the node from the session.

### Per-node TTLs

There may be a `SET_TTL(node)` action, which sets the TTL of node and all
descendent chunks. Then servers must hold on to the given node until the TTL of
the node expires or the session goes away. (Note that a session may be
unilaterally aborted by the receiver.)

### Partial (per-action) failures

In the future, we may extend the protocol to allow partial failures within a
session, where some action failures do not turn into session failures. For now,
the session is aborted upon any action failures.

### Action cancellation

We may provide a way to cancel ongoing actions.

## Examples

### Asking a language model about a long video

Client sends:

```
action {
  name: "GENERATE"
  input {name: "prompt", id: "prompt_1"}
  output {name: "response", id: "response_1"}
}
node_fragment {
  id: "prompt_1"
  child_ids: "question_1"
  child_ids: "video_1"
}
node_fragment {
  id: "question_1"
  chunk_fragment: {
    metadata { mimetype: "text/plain" }
    data: "Write a summary of this video: "
  }
}
node_fragment {
  id: "video_1"
  seq: 0
  continued: true
  chunk_fragment: {
    metadata { mimetype: "video/mp4" }
    ref: "file://path/to/file/part1"
  }
}
node_fragment {
  id: "video_1"
  seq: 1
  chunk_fragment: {
    # NOTE: Metadata in the second chunk can be omitted.
    metadata { mimetype: "video/mp4" }
    # the content of the video at /part2 is concatenated
    # with the content of the video at /part1 to build the
    # full video in the buffer.
    ref: "file://path/to/file/part2"
  }
}
```

Server sends:

```
node_fragment {
  id: "response_1"
  seq: 0
  continued: true
  chunk_fragment {
    metadata { mimetype: "text/plain" }
    data: "It is a translation of an "
  }
}
node_fragment {
  id: "response_1"
  seq: 1
  chunk_fragment {
    metadata { mimetype: "text/plain" }
    data: "F1 race. "
  }
}
```

Client sends:

```
action {
  name: "GENERATE"
  input {name: "prompt", id: "prompt_2"}
  output {name: "response", id: "response_2"}
}
node_fragment {
  id: "prompt_2"
  child_ids: "prompt_1"
  child_ids: "response_1"
  child_ids: "question_2"
}
node_fragment {
  id: "question_2"
  chunk_fragment {
    metadata { mimetype: "text/plain" }
    data: "Who's winning?"
  }
}
```

Server sends:

```
node_fragment {
  id: "response_2"
  chunk_fragment {
    metadata { mimetype: "text/plain" }
    data: "Ayrton Senna."
  }
}
```


### Customized generate action

Client sends:

```
action {
  name: "GENERATE"
  config: [...proto.Any wrapping GenerateConfig...]
  input {name: "text", id: "prompt_1"}
  output {name: "text", id: "response_1"}
}
node_fragment {
  id: "prompt_1"
  child_ids: "prompt_1_text"
  child_ids: "prompt_1_eot"
}
node_fragment {
  id: "prompt_1_text"
  chunk_fragment {
    metadata { mimetype: "text/plain" }
    data: "Write a heroic novel about a half-eaten jam doughnut."
  }
}
node_fragment {
  id: "prompt_1_eot"
  chunk_fragment {
    metadata { mimetype: "application/x-protobuf; type=EndOfTurn" }
  }
}
```

### Streamed chains

Client sends:

```
action {
  name: "GENERATE"
  config: [...proto.Any wrapping GenerateConfig...]
  input {name: "text", id: "prompt_1"}
  output {name: "text", id: "response_1"}
}
node_fragment {
  id: "prompt_1"
  child_ids: "prompt_1_text"
  continued: true
  // `seq: 0` is implicit.
}
node_fragment {
  id: "prompt_1_text"
  chunk_fragment {
    metadata { mimetype: "text/plain" }
    data: "Write a heroic novel about a half-eaten jam doughnut."
  }
}
// `prompt_1` is streamed by sending more chunks belonging to the
// `prompt_1` tree.
node_fragment {
  id: "prompt_1"
  child_ids: "prompt_1_eot"
  seq: 1
  // `continued: false` is implicit.
}
node_fragment {
  id: "prompt_1_eot"
  chunk_fragment {
    metadata { mimetype: "application/x-protobuf; type=EndOfTurn" }
  }
}
```


<sup>1</sup> If a chain or buffer is received before the action, the receiver
has no choice but to buffer it. If however the action is known already, the
buffer can start being processed as its contents arrive.


## Licence

Copyright 2024 DeepMind Technologies Limited.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or  implied.
See the License for the specific language governing permissions and
limitations under the License.
