# go-embryo

> Utilities for creating, parsing, and manipulating embryo nodes on
> IPFS

This package contains a sparse, experimental implementation of
an idea for creating mutable data structures on top of immutable
data stores. This is designed with [IPFS](https://ipfs.io) in mind,
but the protocol is generally agnostic of the underlying storage
network.


# Contents

1. [Introduction](#introduction)
1. [Existing Solutions](#existing-solutions)
1. [Problems](#problems)
1. [Embryo Protocol](#embryo-protocol)
   1. [Basic Data Structure](#basic-data-structure)
   1. [Links](#links)
   1. [Update Channels](#update-channels)
   1. [Update Channels in IPFS](#update-channels-in-ipfs)
   1. [Traversing Timelines](#traversing-timelines)
   1. [Content Verification](#content-verification)
   1. [Dead Streams](#dead-streams)
   1. [Possible Features](#possible-features)
1. [Usage](#usage)
1. [License](#license)


## Introduction

In modern distributed content networks, content-addressability enables
a number of features which dramatically improve both the usability
and the stability of said networks. At the same time, content-
addressing leads to an inherently *immutable* system, at least as
far as individual nodes are concerned.

As many have noted, *mutability* is a requirement of these new
distributed networks if we are to expect a smooth transition into
Web 3.0. Developers need a way to not only reference a static
piece of content, but also a way to point users to the newest update
to that content and to keep them informed of any updates thereafter.


## Existing Solutions

Many projects have their own implementation of a proposed solution
to this issue of immutability. Two promising solutions are found in
the [Dat Project](https://datproject.org) and IPFS-[IPNS](https://github.com/ipfs/go-ipns).

Interestingly, both of these projects rely on a similar mechanism
for creating streams of *mutable* data, though the details of
implementation vary significantly. The main idea behind both is
a data structure and it's updates being signed by a singular private
key and verified by a public key.

In the *Dat project*, this mutable stream takes the form of an
*append-only log* which in turn forms a simple Merkle DAG. In *IPNS*,
the stream is essentially a signed pointer to other mutable records.
(Or thinking about it another way, an *IPNS* record is essentially
a directed graph in which the head node is the public/private key
pair and the various immutable objects are the leaves.)

> TODO Talk about textile.io streams


## Problems

Both of these solutions are recognized as somewhat flawed by
the communities involved in developing and supporting these
systems.

The *Dat project* is currently limited to a single writer for any
given data stream, as there can only be a single public/private
key pair writing to a given stream. Furthermore, even a single
writer can be problematic if multiple concurrent writes occur
from the same key, as is often the case for multiple devices of
a single user given sporadic network connections.

> NOTE: The *Dat project* is working on further enhancements
> to mitigate these problems, including their new *multi-writer*
> protocols and more extensive use of CvRDTs.

*IPNS* also faces this 'single writer' problem, though simply sharing
a single key pair is more feasible in *IPFS*. However, in *IPNS*,
we lose many of the benefits that content-addressability gained
us in the first place, especially if *IPNS* records were to gain
wide use as the primary linking mechanism.

> NOTE: The *IPFS* ecosystem is very wide and diverse. There are
> mentions of other upcoming proposals such as *IPRS*, but I was
> unable to find enough information to remark on them.


## Embryo Protocol

We might be able to mitigate these issues more thoroughly if we
further separate our concerns. In both of the systems discussed
above we are concerned with two distinct issues:

1. The immutable data store
1. The update reference

In the above solutions, the *update reference* system is implemented
as a layer on top of the *immutable data store*. In a sense, the
consumer effectively 'reaches through' the update reference to find
the objects in the data store. This layering of the concerns is
what degrades the benefits of the content addressing -- the content
is distanced from the content itself.

The *embryo* protocol idea centers around the idea of letting the
two concerns exist side-by-side, always favoring the data store layer
so that we can still leverage the benefits of content-addressing.

### Basic Data Structure

The *embryo* protocol is first a wrapper around an object placed
in the immutable data store (e.g. the network's DHT).

In *IPFS*, the standard data object stored in the DHT looks
something like this (from the [white paper](https://ipfs.io/ipfs/QmR7GSQM93Cx5eAg6a6yRzNde1FQv7uL6X1o4k7zrJa3LX/ipfs.draft3.pdf)):

```go
// Basic IPFS object
type IPFSObject struct {
    links []IPFSLink
    data  []byte
}
```

An *embryo* object is a superset of the basic IPFS object,
adding metadata about the update reference **channels**
and the author.

```go
// Embryo object in IPFS
type Embryo struct {
    // Basic IPFS object
    links  []IPFSLink
    data   []byte 
    
    // Required embryo metadata
    authorKey   []byte // Multi-hash of the publisher's public key
    signature   []byte // Author signature of the embryo
    channels    []UpdateChannel // See below
    
    // Optional embryo metadata
    parent      []byte // Multi-hash of the previous published version
    authorInfo  []byte // Multi-hash to information about the publisher
    timestamp   int64 // Unix timestamp
    // Key-Value pairs required for the specified update channel
    // format(s) given. (Could include block-chain type nonce, etc.)
    tags        map[string]string
}

// Update channel object
type UpdateChannel struct {
    format      string // URI of the update format type
    address     string // Multi-address of the update channel
}
```

> NOTE: In the IPFS white paper, a signed object is shown as a wrapper
> object around the raw bytes of the internal message. Here, we are opting
> to keep both properties embedded in the object itself, which means that
> the object must be slightly normalized before the signature if validated.

As noted in the comments, the content itself may either be embedded
directly in the embryo object under the 'content' key or linked to
from the 'contentRef' key. For smaller objects, such as a JSON
document or an IPFS tree object, it makes sense to embed the content.

### Links

In cases where the data is too large to embed directly in the object,
the system must be designed to reference the *embryo* objects
themselves instead of that content.

At first this might seem problematic, but in reality it is
conceptually no different from the basic implementations of data
objects in both *Dat* and *IPFS*. In all cases, a parent node
is inherently able to reference child nodes, and the consumer
applications are expected to treat those children as a *part*
of the larger node.

Linking to embryo nodes directly means that we are able to
maintain the benefits of a content-addressable system. In
addition, the embryo metadata gives us the ability to reference
newer (and possibly older) versions of the data.

### Update Channels

The *update reference* concern is handled by the update channels
listed within the *embryo* itself. Each channel reference must
describe both the address where updates may be found and the
format used for retrieval of those updates. The format documents
themselves would be protocol descriptions dictating the methods
used to retrieve an update via the given address.

This format is something like a *multi-address* in *libp2p*, but
the channel format is explicitly separated out in order to allow
for more varied and free form formats. For instance, it would
be possible in this manner to reference a document describing
a 'one-off' offline update format via standard mail, etc.

### Update Channels in IPFS

In *IPFS*, a naive update channel could run over the pubsub system,
utilizing the hash key of the embryo itself as a channel. In
practice, a system like that would likely be much too chatty
for network-wide implementation.

However, a more optimized system already exists for update channels
which utilizes the XOR properties of the Kademlia DHT -- IPNS!

Using an IPNS record as the 'HEAD' of an update channel makes perfect
sense. We still have the republishing requirements that
we have in the current IPNS implementation, making the data
somewhat ephemeral, but in this case that only affects our update
channel, not the data itself. Thus we maintain the priority of the
data store **over** the update reference concern.

```go
// IPNS update channel record
type IPFSObject struct {
    // May contain a link to the single most recent embryo
    // or up to a full history of embryos from newest to oldest
    links   []IPFSLink
    data    []byte // nil
}
```

### Traversing Timelines

Given the data structures above, the consumer now has the ability
to always find the exact content to which their link refers while
still being able to get newer updates. In the case of IPFS, we
can find that newer record in logarithmic time utilizing IPNS.

If the embryo also includes a reference to a parent record,
or if the update channel includes methods to retrieve a more
full history, we might be able to view past versions of the
data as well.

### Content Verification

Using a system like IPNS as an update channel, we can also
verify, via the publishers public key, that the publisher of the
update is the same as the original publisher of the data we
retrieved.

Our content verification does not end there, however, as we are
not limited to a particular update format protocol. Utilizing
the optional fields on the embryo object as well as the
particulars of the update channel format, it's not hard to envision
encapsulating a block-chain or some other multi-writer data structure
like a CRDT over a distributed network, while still being able to
create permanent links to snapshots of the data at a particular
moment in time.

### Dead Streams

The most obvious and perhaps most critical limitation of this system
is that an update stream might 'die', that is, may no longer be in
operation. In terms of the IPNS system described above, the original
publisher may simply stop publishing to the IPNS address, and the
update channel will no longer be valid.

Although this might be frustrating to a user, it is certainly better
than a dead link for the majority of use cases and more towards the
end goal of a *permanent web*. Furthermore, creating redundancies
for this sort of dead stream is not out of the question (just out
of scope for this README).

### Possible Features

The open ended nature of the update channel format allows for a
wide variety of additional features to be implemented, if one is
so inclined. Below is a partial list -- food for thought:

* Adding publishers to a stream
* Removing publishers from a stream
* Transferring stream 'ownership'
* Forking a stream


## Usage

```go
// TODO
```

## License

Copyright (c) Robert N. Shaw under the **MIT License**. See [LICENSE file](./LICENSE) for details.
