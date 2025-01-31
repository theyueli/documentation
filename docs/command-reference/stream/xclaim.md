---
description: Change the ownership of a message in a consumer group
---

# XCLAIM

## Syntax

	XCLAIM key group consumer min-idle-time id [id ...] [IDLE ms]
      [TIME unix-time-milliseconds] [RETRYCOUNT count] [FORCE] [JUSTID]
      [LASTID lastid]

**Time Complexity:** O(log N) with N being the number of messages in the PEL of the consumer group.

**ACL categories:** @write, @stream, @fast

In the context of a stream consumer group, this command changes the ownership of a pending message, so that the new owner is the consumer specified as the command argument.
Normally this is what happens:

1. There is a stream with an associated consumer group.
2. Some consumer A reads a message via `XREADGROUP` from a stream, in the context of that consumer group.
3. As a side effect a pending message entry is created in the Pending Entries List (PEL) of the consumer group:
   it means the message was delivered to a given consumer, but it was not yet acknowledged via `XACK`.
4. Then suddenly that consumer fails forever.
5. Other consumers may inspect the list of pending messages, that are stale for quite some time, using the [`XPENDING`](./xpending.md) command.
   In order to continue processing such messages, they use `XCLAIM` to acquire the ownership of the message and continue.
   Consumers can also use the `XAUTOCLAIM` command to automatically scan and claim stale pending messages.

You can learn more about Streams [here](https://redis.io/docs/data-types/streams/).

Note that the message is claimed only if its idle time is greater than the minimum idle time we specify when calling `XCLAIM`.
Because as a side effect, `XCLAIM` will also reset the idle time (since this is a new attempt at processing the message),
two consumers trying to claim a message at the same time will never both succeed: only one will successfully claim the message.
This avoids that we process a given message multiple times in a trivial way (yet multiple processing is possible and unavoidable in the general case).

Moreover, as a side effect, `XCLAIM` will increment the count of attempted deliveries of the message unless the `JUSTID` option has been specified (which only delivers the message ID, not the message itself).
In this way messages that cannot be processed for some reason, for instance because the consumers crash attempting to process them, will start to have a larger counter and can be detected inside the system.

`XCLAIM` will not claim a message in the following cases:

1. The message doesn't exist in the group PEL (i.e. it was never read by any consumer)
2. The message exists in the group PEL but not in the stream itself (i.e. the message was read but never acknowledged, and then was deleted from the stream, either by trimming or by `XDEL`)

In both cases the reply will not contain a corresponding entry to that message (i.e. the length of the reply array may be smaller than the number of IDs provided to `XCLAIM`).
In the latter case, the message will also be deleted from the PEL in which it was found.

## Command Options

The command has multiple options, however most are mainly for internal use in order to transfer the effects of `XCLAIM` or other commands
to the AOF file and to propagate the same effects to the replicas, and are unlikely to be useful to normal users:

1. `IDLE <ms>`: Set the idle time (last time it was delivered) of the message.
   If `IDLE` is not specified, an `IDLE` of 0 is assumed, that is, the time count is reset because the message has now a new owner trying to process it.
2. `TIME <ms-unix-time>` : This is the same as `IDLE` but instead of a relative amount of milliseconds, it sets the idle time to a specific Unix time (in milliseconds).
   This is useful in order to rewrite the AOF file generating `XCLAIM` commands.
3. `RETRYCOUNT <count>`: Set the retry counter to the specified value. This counter is incremented every time a message is delivered again.
   Normally `XCLAIM` does not alter this counter, which is just served to clients when the [`XPENDING`](./xpending.md) command is called:
   this way clients can detect anomalies, like messages that are never processed for some reason after a big number of delivery attempts.
4. `FORCE`: Creates the pending message entry in the PEL even if certain specified IDs are not already in the PEL assigned to a different client.
   However, the message must exist in the stream. Otherwise, the IDs of non-existing messages are ignored. 
5. `JUSTID`: Return just an array of IDs of messages successfully claimed, without returning the actual message.
   Using this option means the retry counter is not incremented.
6. `LASTID` : Update the consumer group last ID with the specified ID if the current last ID is smaller than the provided one.


## Return

[Array reply](https://redis.io/docs/reference/protocol-spec/#arrays), specifically:

- The command returns all the messages successfully claimed, in the same format as [`XRANGE`](./xrange.md).
- However, if the `JUSTID` option was specified, only the message IDs are reported, without including the actual message.

## Examples

Create a stream `mystream` with two messages, then create a consumer group `mygroup` with the ID `0-0` as the last delivered entry:

```shell
dragonfly> XADD mystream * name Alice surname Adams
"1695755830453-0"

dragonfly> XADD mystream * name John surname Doe
"1695755847112-0"

dragonfly> XGROUP CREATE mystream mygroup 0-0
OK
```

Within the consumer group, read one message using a consumer (i.e., `consumer-123`) without acknowledging it:

```shell
dragonfly> XREADGROUP GROUP mygroup consumer-123 COUNT 1 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) "1695755830453-0"
         2) 1) "name"
            2) "Alice"
            3) "surname"
            4) "Adams"
```

We claim the message with ID `1695755830453-0`, only if the message is idle for at least one hour without the original consumer
or some other consumer making progresses (acknowledging or claiming it), and assigns the ownership to the consumer `consumer-456`:

```shell
dragonfly> XCLAIM mystream mygroup consumer-456 3600000 1695755830453-0
1) 1) "1695755830453-0"
   2) 1) "name"
      2) "Alice"
      3) "surname"
      4) "Adams"
```
