---
title: "A NATS JetStream Storage Backend using S3-Compatible Object Storage"
date: 2025-10-15T14:05:14-07:00
draft: false
toc: false
images:
tags:
  - untagged
---

## Introduction

[NATS](https://github.com/nats-io/nats-server) is a pub-sub messaging system that enables "simple, secure and performant communications [...] for digital systems, services and devices." [JetStream](https://docs.nats.io/nats-concepts/jetstream) is a persistence feature that builds on top of NATS and "enables messages to be stored and replayed at a later time." JetStream is an optional feature that is embedded into the NATS server. When JetStream is enabled, it allows the creation of "streams" that provide similar functionality to Kafka or Redis Streams. Currently, NATS JetStream supports persisting data to a file-backed store or in-memory.

There is a recent trend of using S3-compatible object for providing data persistence to infrastructure systems including WarpStream and Mimir. The benefits to using object storage over file-based storage or direct block, include:

- Cost: Object storage is significantly cheaper than block storage for data at rest.
- Infinite Scalability: If using an object storage service such as AWS S3, no capacity planning is needed.
- Durability and Availability: Object storages provide durability and availability guarantees. and durability guarantees (For AWS S3, 99.999999999% durability and 99.99% availability over a year).

The major downside of object storage is latency for both reading and writing data.

In this project, a patch to the NATS server is created that adds a new storage backend: "object storage". This allows JetStream assets (streams and consumers) to be created with no dependency on a local file system. The patch can be [viewed on GitHub](https://github.com/nats-io/nats-server/compare/main...sethitow:nats-server:objstore?expand=1). Note, this work is unrelated to the "object store" features of the NATS client, which allows a NATS server to be _used_ as an object storage service.

## Goals and Non-Goals

This is a personal project undertaken for two reasons:

- to learn more about NATS and the JetStream storage layer
- to explore and understand the implementation challenges with storing message logs and time series data in object storage, inspired by existing systems such as AutoMQ, Mimir, and to a lesser extent DuckDB.

This implementation is not suitable for use in production. However, a productionalized implementation could be used for asynchronously archiving stream messages for continuous backups, data retention compliance, or offline analytical processing. This could be realized by using a file-backed stream as the primary online persistence, and creating an object-backed stream to source from the primary.

## References

There is prior discussion of this concept on GitHub

- [Discussion #5486: Support S3 API for JetStream](https://github.com/nats-io/nats-server/discussions/5486)
- [Discussion #6478: S3 next level: offload, backup plus also data-sharing, querying?](https://github.com/nats-io/nats-server/discussions/6478)
- [Issue #4871: S3 Compatibility for NATS Object Store](https://github.com/nats-io/nats-server/issues/4871)

## Running this project

### Server

The server can be built and run with no special considerations. To run it, it can be pointed at an S3 compatible object storage service. During development, a local Minio server was used. Assuming Minio is running locally and has a bucket called "jetstream-data", the following configuration can be used to run the server:

```json
debug: true
http_port: 8080
prof_port = 65432
jetstream {
    enabled: true
    object_store {
        endpoint: "localhost:9000"
        access_key_id: "minioadmin"
        secret_access_key: "minioadmin"
        region: ""
        bucket: "jetstream-data"
        path_prefix: ""
    }
}
```

### NATS CLI

The NATS CLI validates the schema of request and response objects. To use the NATS CLI with this modified server, support for the object storage type must be added to `natscli`, `nats.go`, and `jsm.go`.

First, clone all the repositories.

```bash
git clone --branch objstore https://github.com/sethitow/natscli.git
git clone --branch objstore https://github.com/sethitow/jsm.go.git
git clone --branch objstore https://github.com/sethitow/nats.go.git
```

The following `go.work` file can be used, assuming all three repositories are checked out in the same folder. The file should be placed in the parent directory that contains the three repositories.

```
go 1.24.0

use (
	./natscli
	./jsm.go
	./nats.go
)
```

Then, you can build the NATS CLI like normal

```bash
cd natscli
go run ./nats stream info {...}
```

## Stream Configuration

The recommended stream configuration is as follows:

- `deny_purge` and `deny_delete` must both be true. Object storage is typically immutable, so performant deletion of an individual message is challenging. See notes below on "Message Deletion".
- `persist_mode` is recommended to be `async`. With this setting, puback'ed messages are buffered in memory before being uploaded as a block.

```json
{
  "name": "test",
  "description": "test stream using object storage",
  "subjects": ["test.>"],
  "storage": "object",
  "retention": "limits",
  "deny_delete": true,
  "deny_purge": true,
  "allow_direct": true,
  "persist_mode": "async"
}
```

## Design Discussion

### Message Deletion

Object storage is typically immutable. Editing an uploaded object requires deleting and re-uploading the object in its entirety. This makes "purging" a message very expensive. As such, it is not supported in the current implementation. Tombstones could be implemented to track deletions separately from the blocks themselves. However, if it is required to actually delete the data (i.e. legal reasons), then mutating the block is unavoidable.

### Clustering

The current implementation only supports single node operation. Clustering is untested.

What could clustering look like for object storage? The storage is assumed to be durable, so each node need not maintain its own copy of the data. Writes still need to go to the leader to be assigned a sequence. Reads can be served from any node (i.e. all reads are direct gets). Since no data is stored on disk, there is no concept of placement. Any node could become either a leader or follower.

### Optimizing Blocks for Range Requests

Once a block is uploaded, only the metadata is stored in memory. All other data is retrieved via range requests. The current implementation does no caching of downloaded blocks; caching would be a sensible addition.

scenarios:

- direct get by sequence
  - well suited to range requests
- consumer seek (i.e. `LoadNextMsg`)
  - block metadata can be selectively downloaded
- last per subject
  - crazy expensive in space and time, no idea how to do this efficiently

layout

- fixed size header
  - block-level accounting (msgs, size, first seq/ts, last seq/ts)
  - offsets
    - message index (offset, len)
    - per-subject index (offset, len)
    - messages (offset, len)
- message index
  - list of all messages
    - seq
    - offset
    - len
- per-subject index
  - some sort of representation of the stree contained within the given block
- messages
  - list of all messages
    - subj
    - header
    - data

### Consumers

Currently, consumers are kicked after a message is processed. Because the storage of the message is asynchronous, the message may not be available when the consumer updates its state with the new sequence. This should be re-architected to kick consumers upon upload completion, not when the message is processed.

The above situation creates a problem for push consumers. Consumers won't get kicked until the next message after the last block upload. Currently, push consumers are prohibited on object store streams.

Pull consumers will receive new messages next time they poll.

The current implementation does not support multi-subject consumers. There is no technical reason for this; they could be implemented in the same way that they are for the filestore.

### Breaking Optimizations

When a client reads a message out of the stream, there are two phases:

- Phase 1: figure out which message is relevant by evaluating subject, sequence, delivery policy, etc
- Phase 2: serve the message contents from the storage to the client

The consumer protocol could be reimagined such that the server only sends a message reference to the client. The client can download the message itself via a range request and presigned URL. This has the following benefits:

- Bandwidth between the client and the server is conserved. Only the message reference information needs to be transmitted.
- Latency is potentially reduced from the client's perspective. Since the server is essentially just proxying the data from object storage to the client, allowing the client to download the data directly reduces network hops, and therefore latency.
