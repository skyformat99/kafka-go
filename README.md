# kafka-go [![CircleCI](https://circleci.com/gh/segmentio/kafka-go.svg?style=shield)](https://circleci.com/gh/segmentio/kafka-go) [![Go Report Card](https://goreportcard.com/badge/github.com/segmentio/kafka-go)](https://goreportcard.com/report/github.com/segmentio/kafka-go) [![GoDoc](https://godoc.org/github.com/segmentio/kafka-go?status.svg)](https://godoc.org/github.com/segmentio/kafka-go)

## Motivations

We rely on both Go and Kafka a lot at Segment, unfortunately the state of the Go
client libraries for Kafka at the time of this writing was not ideal. The couple
of options available were:

- [sarama](https://github.com/Shopify/sarama) which is by far the most popular
but is quite difficult to work with. It is poorly documented, the API exposes
low level concepts of the Kafka protocol, doesn't support recent Go features
like [contexts](https://golang.org/pkg/context/). It also passes all values as
pointers which causes large numbers of dynamic memory allocations, more frequent
garbage collections, and higher memory usage.

- [confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go) is a
cgo based wrapper around [librdkafka](https://github.com/edenhill/librdkafka),
which means it introduces a dependency to a C library on all Go code that uses
the package. It has much better documentation than sarama but stil lacks support
for Go contexts.

- [goka](https://github.com/lovoo/goka) is a more recent Kafka client for Go
which focuses on a specific usage pattern. It provides abstractions to use Kafka
as a message passing bus between services rather than an ordered log of events,
this is not the typical use case we make of Kafka at Segment. The package also
depends on sarama for all interractions with kafka.

This is where `kafka-go` comes into play, it provides both low and high level
APIs to interract with Kafka, mirroring concepts and implementing interfaces of
the standard library to make it easy to use and integrate with existing
software.

## Connection [![GoDoc](https://godoc.org/github.com/segmentio/kafka-go?status.svg)](https://godoc.org/github.com/segmentio/kafka-go#Conn)

The `Conn` type is the core of the `kafka-go` package, it wraps around a raw
network connection to expose a low-level API to a Kafka server.

Here are examples of how to typically use a connection object:
```go
// to produce messages
conn, _ := kafka.DialContext(ctx, "tcp", "localhost:9092")

conn.SetWriteDeadline(time.Now().Add(10*time.Second))
conn.WriteMessages(
    kafka.Message{Value: []byte("one!")},
    kafka.Message{Value: []byte("two!")},
    kafka.Message{Value: []byte("three!")},
)

conn.Close()
```
```go
// to consume messages
conn, _ := kafka.DialLeader(ctx, "tcp", "localhost:9092")

conn.SetReadDeadline(time.Now().Add(10*time.Second))
batch := conn.ReadBatch(1e6) // fetch 1MB max

b := make([]byte, 10e3) // 10KB max per message
for {
    n, err := batch.Read(b)
    if err != nil {
        break
    }
    fmt.Println(string(b))
}

batch.Close()
conn.Close()
```

Because of its low level, the `Conn` type turns out to be a great building block
for higher level abstractions, like the `Reader` for example.

## Reader [![GoDoc](https://godoc.org/github.com/segmentio/kafka-go?status.svg)](https://godoc.org/github.com/segmentio/kafka-go#Reader)

A `Reader` is another concept exposed by the `kafka-go` package, which intends
to make it simpler to implement the typical use case of consuming from a single
topic-partition pair.  
A `Reader` also automatically handles reconnections and offset management, and
exposes an API that supports asynchronous cancellations and timeouts using Go
contexts.

```go
// make a new reader that consumes from topic-A, partition 0, at offset 42
r := kafka.NewReader(kafka.ReaderConfig{
    Brokers:   []string{"localhost:9092"},
    Topic:     "topic-A",
    Partition: 0,
    MinBytes:  10e3, // 10KB
    MaxBytes:  10e6, // 10MB
})
r.SetOffset(42)

for {
    m, err := r.ReadMessage(ctx)
    if err != nil {
        break
    }
    fmt.Printf("message at offset %d: %s = %s\n", m.Offset, string(m.Key), string(m.Value))
}

r.Close()
```

## Writer [![GoDoc](https://godoc.org/github.com/segmentio/kafka-go?status.svg)](https://godoc.org/github.com/segmentio/kafka-go#Writer)

To produce messages to Kafka, a program may use the low-level `Conn` API, but
the package also provides a higher level `Writer` type which is more appropriate
to use in most cases because it provides additional features:

- Automatic retries and reconnections on errors.
- Configurable distribution of messages across available partitions.
- Synchronously or asynchronously writing messages to Kafka.
- Asynchronous cancellation using contexts.
- Flush pending messages on close to support graceful shutdowns.

```go
// make a writer that produces to topic-A, using the least-bytes distribution
w := kafka.NewWriter(kafka.WriterConfig{
	Brokers: []string{"localhost:9092"},
	Topic:   "topic-A",
	Balancer: &kafka.LeastBytes{},
})

w.WriteMessages(context.Background(),
	kafka.Message{
		Key:   []byte("Key-A"),
		Value: []byte("Hello World!"),
	},
	kafka.Message{
		Key:   []byte("Key-B"),
		Value: []byte("One!"),
	},
	kafka.Message{
		Key:   []byte("Key-C"),
		Value: []byte("Two!"),
	},
)

w.Close()
```
