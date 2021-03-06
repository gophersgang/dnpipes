# dnpipes

Distributed Named Pipes (or: `dnpipes`) are essentially a distributed version of Unix [named pipes](http://en.wikipedia.org/wiki/Named_pipe) comparable to, for example, [SQS](https://aws.amazon.com/sqs/) in AWS or the [Service Bus](https://azure.microsoft.com/en-us/services/service-bus/) in Azure. 

![dnpipes concept](img/concept.png)

Conceptually, we're dealing with a bunch of distributed processes (`dpN` above). These distributed processes may be long-running (such as `dp0` or `dp5`) or batch-oriented ones, for example `dp3` or `dp6`. There are a number of [situations](#use-cases) where you want these distributed processes to communicate, very similar to what [IPC](http://tldp.org/LDP/lpg/node7.html) enables you to do on a single machine. Now, `dnpipes` are a simple mechanism to facilitate IPC between distributed processes. What follows is an [interface specification](#interface-specification) as well as a [reference implementation](#reference-implementation) for `dnpipes`.

## Interface specification

Interpret the key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", "MAY NOT", and "OPTIONAL" in the context of this repo as defined in [RFC 2119](https://tools.ietf.org/html/rfc2119).

A `dnpipes` is a distributed ordered queue (FIFO) of messages available to a number of participating distributed processes. A distributed process is a useful abstraction provided by systems such as DC/OS (for example a Marathon app or a Metronome job) or Kubernetes (ReplicaSet or a Job) that give a user the illusion that a service or application she is executing on a bunch of commodity machines (the cluster) behaves like one global entity while it really is a collection of locally executed processes. In DC/OS this locally executed process would be a [Mesos task](http://mesos.apache.org/documentation/latest/architecture/) and in Kubernetes a [pod](http://kubernetes.io/docs/api-reference/v1/definitions/#_v1_podspec).


A `dnpipes` implementation MUST support the following operations:

- `push(TOPIC, MSG)` … executed by a publisher, this writes the message `MSG` to a `dnpipes` called `TOPIC`.
- `MSG <- pull(TOPIC)` … executed by a subscriber, this reads a message from a `dnpipes` called `TOPIC`.
- `reset(TOPIC)` … executed by either a publisher or consumer, this removes all messages from a `dnpipes` called `TOPIC`.

The following MUST be true at any point in time:

1. After `push` is executed by the publisher `MSG` MUST be available for `pull` to any participant until `reset` is triggered and has completed.
1. A `pull` does not remove a message from a `dnpipes`, it merely delivers its content to the consumer.
1. The way how participants discover a `dnpipes` is outside of the scope of this specification.

Note concerning the last point: since there are many ways to implement service discovery in a distributed system we do not expect that an agreement can be found here hence we leave it up to the implementation how to go about it. The next sections shows an example using Kafka and DNS to achieve this.

## Reference implementation

The reference implementation of `dnpipes` is based on [DC/OS](https://dcos.io) using [Apache Kafka](http://kafka.apache.org/).

### Install

From source:

```bash
$ go get github.com/mhausenblas/dnpipes
$ go build
$ sudo mv dnpipes /usr/local/dnpipes
```

From binaries, for Linux:

```bash
$ curl -s -L https://github.com/mhausenblas/dnpipes/releases/download/0.1.0/linux-dnpipes -o dnpipes
$ sudo mv dnpipes /usr/local/dnpipes
$ sudo chmod +x /usr/local/bin/dnpipes
```

From binaries, for macOS:

```bash
$ curl -s -L https://github.com/mhausenblas/dnpipes/releases/download/0.1.0/macos-dnpipes -o dnpipes
$ sudo mv dnpipes /usr/local/dnpipes
$ sudo chmod +x /usr/local/bin/dnpipes
```

### Example session

To try it out yourself, you first need a to [install a DC/OS cluster](https://dcos.io/install/) and then Apache Kafka like so:

```bash
$ dcos package install kafka
```

Note that if you are unfamiliar with Kafka and its terminology, you can check out the respective [Kafka 101 example](https://github.com/dcos/examples/tree/master/1.8/kafka) now.

Next, figure out where the brokers are (in my case I started Kafka with one broker):

```bash
$ dcos kafka connection

{
  "address": [
    "10.0.2.94:9951"
  ],
  "zookeeper": "master.mesos:2181/dcos-service-kafka",
  "dns": [
    "broker-0.kafka.mesos:9951"
  ],
  "vip": "broker.kafka.l4lb.thisdcos.directory:9092"
}
```
Now, an example session using the `dnpipes` reference implementation looks as follows.
I've set up two terminals, in one I'm starting the `dnpipes` in publisher mode:

```bash
$ ./dnpipes --mode=publisher --broker=broker-0.kafka.mesos:9951 --topic=test
> hello!
> bye
> RESET
2016/12/11 13:38:57 Connected to 10.0.6.130:2181
2016/12/11 13:38:57 Authenticated: id=97087250213175381, timeout=4000
[0] - &{Czxid:295 Mzxid:295 Ctime:1481463523762 Mtime:1481463523762 Version:0 Cversion:1 Aversion:0 EphemeralOwner:0 DataLength:0 NumChildren:1 Pzxid:296}
reset this dnpipes
> ^C
```

The second terminal has `dnpipes` in subscriber mode running:

```bash
$ ./dnpipes --mode=subscriber --broker=broker-0.kafka.mesos:9951 --topic=test 2>/dev/null
hello!
bye
^C
```

And here's a screen shot of the whole thing:

![screen shot of example dnpipes session](img/session.png)

## Use cases

A `dnpipes` can be useful in a number of situations including but not limited to the following:

- To implement a work queue with, for example: [Adron/testing-aws-sqs-worker](https://github.com/Adron/testing-aws-sqs-worker)
- To do event dispatching in microservices, for example: [How we build microservices at Karma](https://blog.karmawifi.com/how-we-build-microservices-at-karma-71497a89bfb4)
- To coordinate Function-as-a-Service executions, for example: [Integrate SQS and Lambda: serverless architecture for asynchronous workloads](https://cloudonaut.io/integrate-sqs-and-lambda-serverless-architecture-for-asynchronous-workloads/)
