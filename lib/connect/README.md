# Consumer / Producer
- these clients were designed for node-kafka-connect, however
than are also exported via sinek an can surely be used as stand-alone
consumers or producers. (in fact; they are easier to setup.)
- clients will automatically re-connect
- consumers will automatically rebalance
- producers will refresh topic meta-data on connect() //if topics are passed in constructor

## Using the consumer

```es6
const {Consumer} = require("sinek");
const consumer = new Consumer("my-topic", config); //checkout config below

// without backpressure
const withBackpressure = false;
consumer.connect(withBackpressure).then(_ => {
    consumer.consume(); //resolves a promise on the first message consumed
});

consumer.on("message", message => console.log(message));
consumer.on("error", error => console.error(error));

// with backpressure
const withBackpressure = true;
consumer.connect(withBackpressure).then(_ => {
    consumer.consume((message, callback) => {
        console.log(message);
        const error = null;
        callback(error); //you must return this callback to receive further messages
    });
});

consumer.on("error", error => console.error(error));

//messages are always objects that look like this:
{
    value: "..",
    key: "..",
    partition: 0,
    offset: 15,
    highWaterOffset: 15,
    topic: "my-topic"
}
```

## Consuming a whole topic once

```es6
const drainThreshold = 10000; //time that should pass with no message being received
const timeout = 0; //time that should maximally pass in total (0 = infinite)
const messageCallback = null; //just like the usual .consume() this supports events and callbacks (for backpressure)

consumer.consumeOnce(messageCallback, drainThreshold = 10000, timeout = 0)
    .then(consumedMessagesCount => console.log(consumedMessagesCount)); //resolves when topic is drained
    .catch(error => console.error(error)); //fires on error or timeout

consumer.on("message", message => console.log(message"));
consumer.on("error", error => console.error(error));
```

## Using the producer

```es6
const {Producer} = require("sinek");
const partitions = 1; //make sure the topic exists and has the given amount of partitions (relates to kafka broker config setup)
const producer = new Producer(config, ["my-topic", partitions]);

//all 3 return promises (buffer wont actually buffer, they will all be sent immediatly per default)
producer.send("my-topic", "my message as string"); //messages will be automatically spread across partitions randomly

const compressionType = 0;
producer.buffer("my-topic", "my-message-key-identifier", {bla: "message as object"}, compressionType);
//this will create a keyed-message (e.g. Kafka LogCompaction on Message-Keys), producer will automatically identfiy
the message-key to a topic partition (idempotent)

//if you do not pass in an identifier, it will be created as uuid.v4()

const version = 1;
producer.bufferFormat("my-topic", "my-message-key-identifier", {bla: "message as object"}, version, compressionType)
//same as .buffer(..) but with the fact that it wraps your message in a certain "standard" message json format e.g.:

{
    payload: {bla: "message as object"},
    key: "my-message-key-identifier",
    id: "my-message-key-identifier",
    time: "2017-05-29T11:58:15.139Z",
    type: "my-topic-published"
}

producer.on("error", error => console.error(error));
```

## Common Methods

```es6
client.getStats() //returns an object with information about the consumer/producer
client.pause() //stops (dont pause continously; the backpressure functionality of the consumer is most likely what you are looking for)
client.resume() //continues
client.close() //close the connection |-> you can pass true/false to the consumer to commit (false per default) the last state before closing
```

## Configuring the kafka connection

```es6
const config = {
    zkConStr: "localhost:2181/",
    logger: {
        debug: msg => console.log(msg),
        info: msg => console.log(msg),
        warn: msg => console.log(msg),
        error: msg => console.log(msg)
    },
    groupId: "kc-sequelize-test",
    clientName: "kc-sequelize-test-name",
    workerPerPartition: 1,
    options: {
        sessionTimeout: 8000,
        protocol: ["roundrobin"],
        fromOffset: "earliest", //latest
        fetchMaxBytes: 1024 * 100,
        fetchMinBytes: 1,
        fetchMaxWaitMs: 10,
        heartbeatInterval: 250,
        retryMinTimeout: 250,
        autoCommit: true, //if you set this to false and run with backpressure the consumer will commit on every successfull batch
        autoCommitIntervalMs: 1000,
        requireAcks: 0,
        ackTimeoutMs: 100,
        partitionerType: 3
    }
};
```

## FAQ

### 1
* Q: My consumer does not receive any messages but no error is logged :(
* A: Most likely the consumer cannot establish a connection to Zookeeper (the module wont log any errors, check the host, port, url setup)

### 2
* Q: I get a lot of leader not available errors when using the producer :(
* A: Check your partition settings, does the topic really have the corresponding amount of partitions available
* A: Otherwise this might related to topic metadata, run `producer.refreshMetadata(["topic"]).then()` (once) first.
