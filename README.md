# liferaft

[![Made by unshift](https://img.shields.io/badge/made%20by-unshift-00ffcc.svg?style=flat-square)](http://unshift.io)[![Version npm](http://img.shields.io/npm/v/liferaft.svg?style=flat-square)](http://browsenpm.org/package/liferaft)[![Build Status](http://img.shields.io/travis/unshiftio/liferaft/master.svg?style=flat-square)](https://travis-ci.org/unshiftio/liferaft)[![Dependencies](https://img.shields.io/david/unshiftio/liferaft.svg?style=flat-square)](https://david-dm.org/unshiftio/liferaft)[![Coverage Status](http://img.shields.io/coveralls/unshiftio/liferaft/master.svg?style=flat-square)](https://coveralls.io/r/unshiftio/liferaft?branch=master)[![IRC channel](http://img.shields.io/badge/IRC-irc.freenode.net%23unshift-00a8ff.svg?style=flat-square)](http://webchat.freenode.net/?channels=unshift)

`liferaft` is an JavaScript implementation of the [Raft] consensus algorithm. 

## Installation

The `liferaft` module is distributed through npm and is compatible with
`browserify` as well as `node.js`. You can install this module using:

```
npm install --save liferaft
```

## Usage

In all examples we assume that you've imported the `liferaft` module as
following:

```js
'use strict';

var LifeRaft = require('liferaft')
  , raft = new Raft('name', { /* optional options */});
```

Please note that the instructions for Node.js and browser are exactly the same
as we assume that your code will be compiled using a [Browserify] based system.
The only major difference is that you probably don't need to configure a commit
and append log (but this of course, fully optional).

The LifeRaft library is quite dumb by default. We try to be as lean and
extendible as possible so you, as a developer, have complete freedom on how you
want to implement Raft in your architecture. This also means that we ship this
library without any build in transport. This allows you to use it with your
existing technology stack and environment. If you want to use `SharedWorkers` as
transport in the browser? Awesome, you can do that. Want to use it on node?
There are literally thousands of different transport libraries that you can use.

### Configuration

There are a couple of default options that you can configure in the constructor
of your Raft:

- `name` A unique name of the node that we just created. If none is supplied we
  will generate a random UUID.
- `heartbeat` The heartbeat timeout. Make sure that this value is lower then
  your minimum election timeout and take message latency in consideration when
  specifying this and the minimum election value.
- `election min` Minimum election timeout.
- `election max` Maximum election timeout.
- `threshold` Threshold for when the heartbeat and latency is to close to the
  minimum election timeout.
- `Log`: An Log compatible constructor we which use for state and data
  replication. 

The timeout values can be configured with either a number which represents the
time milliseconds or a human readable time string such as `10 ms`. The heartbeat
timeouts are used to detect a disconnection from the `LEADER` process if no
message has been received within the given timeout we assume its dead that we
should be promoted to master. The election timeout is the time it may take to
reach a consensus about the master election process. If this times out, we will
start another re-election.

### Events

The `liferaft` module is an `EventEmitter` at it's core and is quite chatty
about the events it emits.

Event               | Description
--------------------|------------------------------------------------------
`term change`       | The term has changed.
`leader change`     | We're now following a newly elected leader.
`state change`      | Our state/role changed.
`heartbeat timeout` | Heartbeat timeout, we're going to change to candidate.
`data`              | Emitted by you, so we can work with the data.
`vote`              | We've received a vote request.
`leave`             | Node has been removed from the cluster.
`join`              | Node has been added to the cluster.
`end`               | This Raft instance has ended.
`initialize`        | The node has been fully initialized.
`error`             | An error happened while doing.. Things!
`threshold`         | The heartbeat timeout is getting close to election timeout.
`leader`            | Our state changed to leader.
`follower`          | Our state changed to follower.
`candidate`         | Our state changed to candidate.
`stopped`           | Our state changed to stopped.

### LifeRaft.type(of)

Check the type of the given thing. This returns the correct type for arrays,
objects, regexps and all the things. It's used internally in the library but
might be useful for you as user as well. The function requires one argument
which would be the `thing` who's type you need figure out.

```js
raft.type([]); // array
raft.type({}); // object
```

### LifeRaft.quorum(responses)

Check if we've reached our quorum (a.k.a. minimum amount of votes requires for a
voting round to be considered valid) for the given amount of votes. This depends
on the amount of joined nodes. It requires one argument which is the amount of
responses that have been received.

```js
raft.join('tcp://127.0.0.1');
raft.join('tcp://127.0.0.2');
raft.join('tcp://127.0.0.3');
raft.join('tcp://127.0.0.4');
raft.join('tcp://127.0.0.4');

raft.quorum(5); // true
raft.quorum(2); // false
```

### LifeRaft.majority()

Returns the majority that needs to be reached for our quorum.

```js
raft.majority(); // 4
```

### LifeRaft.indefinitely(attempt, fn, timeout)

According to section x.x of the Raft paper it's required that we retry sending
the RPC messages until they succeed. This function will run the given `attempt`
function until the received callback has been called successfully and within our
given timeout. If this is not the case we will call the attempt function again
and again until it succeeds. The function requires 3 arguments:

1. `attempt`, The function that needs to be called over and over again until he
   calls the receiving callback successfully and without errors as we assume an
   error first callback pattern.
2. `fn`, Completion callback, we've successfully executed the attempt.
3. `timeout`, Time the attempt is allowed to take.

```js
raft.indefinitely(function attemp(next) {
  dosomething(function (err, data) {
    //
    // if there is no error then we wil also pass the data to the completion
    // callback.
    //
    return next(err, data);
  });
}, function done(data) {
  // Successful execution.
}, 1000);
```

### LifeRaft.packet(type, data)

Generate a new packet object that can be transfered to a client. The method
accepts 2 arguments:

1. `type`, Type of packet that we want to transfer.
2. `data`, Data that should be transfered.

```js
var packet = raft.packet('vote', { foo: 'bar' });
```

These packages will contain the following information:

- `state` If we are a `LEADER`, `FOLLOWER` or `CANDIDATE`
- `term` Our current term.
- `name` The name of this node.
- `leader` The name of our leader.
- `last` If logs are enabled we also include the last committed term and index.

And of course also the `type` which is the type you passed this function in and
the `data` that you want to send.

### LifeRaft.join(name, write)

Add a new raft node to your cluster. All parameters are optional but normally
you would pass in the name or address with the location of the server you want
to add. The write method is only optional if you are using a custom instance
that already has the `write` method defined.

```js
var node = raft.join('127.0.0.1:8080', function write(packet) {
  // Write the message to the actual server that you just added.
});
```

As seen in the example above it returns the `node` that we created. This `Node`
is also a Raft instance. When the node is added to the cluster it will emit the
`join` event. The event will also receive a reference to the node that was added
as argument:

```js
raft.on('join', function join(node) {
  console.log(node.name); // 127.0.0.1:8080
});
```

### LifeRaft.leave(name)

Now that you've added a new node to your raft cluster it's also good to know
that you remove them again. This method either accepts the name of the node that
you want to remove from the cluster or the returned `node` that was returned
from the [`LifeRaft.join`](#liferaftjoin) method.

```js
raft.leave('127.0.0.1:8080');
```

Once the node has been removed from the cluster it will emit the `leave` event.
The event will also receive a reference to the node that was removed as
argument:

```js
raft.on('leave', function leave(node) {
  console.log(node.name); // 127.0.0.1:8080
});
```

### LifeRaft.end()

This signals that the node wants to be removed from the cluster. Once it has
successfully removed it self, it will emit the `end` event.

```js
raft.on('end', function () {
  console.log('Node has shut down.');
});

raft.end();
```

### LifeRaft.promote()

**Private method, use with caution**

This promotes the Node from `FOLLOWER` to `CANDIDATE` and starts requesting
votes from other connected nodes. When the majority has voted in favour of this
node, it will become `LEADER`.

```js
raft.promote();
```

## Extending

LifeRaft uses the same pattern as Backbone.js to extend it's prototypes. It
exposes an `.extend` method on the constructor. When you call this method it
will return a fresh LifeRaft constructor with the newly applied prototypes and
properties. So these extends will not affect the default instance. This extend
method accepts 2 arguments.

1. Object with properties that should be merged with the `prototype`.
2. Object with properties that should be merged with the constructor.

```js
var LifeBoat = LifeRaft.extend({
  foo: function foo() {
    return 'bar';
  }
});
```

### Initialization

When you extend the `LifeRaft` instance you can assign a special `initialize`
method. This method will be called when our `LifeRaft` code has been fully
initialized and we're ready to initialize your code. Please bare in mind that
this is a synchronous invocation and that we will start heart beat timeout after
the execution of the function. This method is ideal for implementing your own
transport technology. The function is invoked with one argument, these are the
options that were used to construct the instance. If no options were provided we
will default to empty object so this argument is always an available.

```js
var LifeBoat = LifeRaft.extend({
  socket: null,
  initialize: function initialize(options) {
    this.socket = new CustomTransport(this.name);
  }
});
```

In parallel to the execution of your `initialize` method we also emit an
`initialize` event. This receives the same amount of arguments.

## License

MIT

[Raft]: https://ramcloud.stanford.edu/raft.pdf
[Browserify]: http://browserify.org/
