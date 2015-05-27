# goczmq [![Build Status](https://travis-ci.org/zeromq/goczmq.svg?branch=master)](https://travis-ci.org/zeromq/goczmq) [![Doc Status](https://godoc.org/github.com/zeromq/goczmq?status.png)](https://godoc.org/github.com/zeromq/goczmq)

## Introduction
A golang interface to the [CZMQ](http://czmq.zeromq.org) ZeroMQ API.

## Installation

```
git clone git@github.com:jedisct1/libsodium.git
cd libsodium
./autogen.sh; ./configure; make; make check
sudo make install
sudo ldconfig
```

```
git clone git@github.com:zeromq/libzmq.git
cd libzmq
./autogen.sh; ./configure --with-libsodium; make; make check
sudo make install
sudo ldconfig
```

```
git clone git@github.com:zeromq/czmq.git
cd czmq
./autogen.sh; ./configure; make; make check
sudo make install
sudo ldconfig
```

```
go get github.com/zeromq/goczmq
```

## Usage
### Direct ZeroMQ API
```go
package main

import (
	"log"

	"github.com/zeromq/goczmq"
)

func main() {
	// Create a router socket and bind it to port 5555.
	router, err := goczmq.NewRouter("tcp://*:5555")
	if err != nil {
		log.Fatal(err)
	}
	defer router.Destroy()

	log.Println("router created and bound")

	// Create a dealer socket and connect it to the router.
	dealer, err := goczmq.NewDealer("tcp://127.0.0.1:5555")
	if err != nil {
		log.Fatal(err)
	}
	defer dealer.Destroy()

	log.Println("dealer created and bound")

	// Send a 'Hello' message from the dealer to the router.
	// Here we send it as a frame ([]byte), with a FlagNone
	// flag to indicate there are now more frames following.
	err = dealer.SendFrame([]byte("Hello"), goczmq.FlagNone)
	if err != nil {
		log.Fatal(err)
	}

	log.Println("dealer sent 'Hello'")

	// Receve the message. Here we call RecvMessage, which
	// will return the message as a slice of frames ([][]byte).
	// Since this is a router socket that support async
	// request / reply, the first frame of the message will
	// be the routing frame.
	request, err := router.RecvMessage()
	if err != nil {
		log.Fatal(err)
	}

	log.Printf("router received '%s' from '%v'", request[1], request[0])

	// Send a reply. First we send the routing frame, which
	// let's the dealer know which client to send the message.
	// The FlagMore flag tells the router there will be more
	// frames in this message.
	err = router.SendFrame(request[0], goczmq.FlagMore)
	if err != nil {
		log.Fatal(err)
	}

	log.Printf("router sent 'World'")

	// Next send the reply. The FlagNone flag tells the router
	// that this is the last frame of the message.
	err = router.SendFrame([]byte("World"), goczmq.FlagNone)
	if err != nil {
		log.Fatal(err)
	}

	// Receive the reply.
	reply, err := dealer.RecvMessage()
	if err != nil {
		log.Fatal(err)
	}

	log.Printf("dealer received '%s'", string(reply[0]))
}
```
### io.ReadWriter support
```go
package main

import (
	"log"

	"github.com/zeromq/goczmq"
)

func main() {
	// Create a router socket and bind it to port 5555.
	router, err := goczmq.NewRouter("tcp://*:5555")
	if err != nil {
		log.Fatal(err)
	}
	defer router.Destroy()

	log.Println("router created and bound")

	// Create a dealer socket and connect it to the router.
	dealer, err := goczmq.NewDealer("tcp://127.0.0.1:5555")
	if err != nil {
		log.Fatal(err)
	}
	defer dealer.Destroy()

	log.Println("dealer created and bound")

	// Send a 'Hello' message from the dealer to the router,
	// using the io.Write interface
	n, err := dealer.Write([]byte("Hello"))
	if err != nil {
		log.Fatal(err)
	}

	log.Printf("dealer sent %d byte message 'Hello'\n", n)

	// Make a byte slice and pass it to the router
	// Read interface. When using the ReadWriter
	// interface with a router socket, the router
	// caches the routing frames internally in a
	// FIFO and uses them transparently when
	// sending replies.
	buf := make([]byte, 16386)

	n, err = router.Read(buf)
	if err != nil {
		log.Fatal(err)
	}

	log.Printf("router received '%s'\n", buf[:n])

	// Send a reply.
	n, err = router.Write([]byte("World"))
	if err != nil {
		log.Fatal(err)
	}

	log.Printf("router sent %d byte message 'World'\n", n)

	// Receive the reply, reusing the previous buffer.
	n, err = dealer.Read(buf)
	if err != nil {
		log.Fatal(err)
	}

	log.Printf("dealer received '%s'", string(buf[:n]))
}
```
### Thread safe channel interface
```go
package main

import (
	"log"

	"github.com/zeromq/goczmq"
)

func main() {
	// Create a router channeler and bind it to port 5555.
	// A channeler provides a thread safe channel interface
	// to a *Sock
	router := goczmq.NewRouterChanneler("tcp://*:5555")
	defer router.Destroy()

	log.Println("router created and bound")

	// Create a dealer channeler and connect it to the router.
	dealer := goczmq.NewDealerChanneler("tcp://127.0.0.1:5555")
	defer dealer.Destroy()

	log.Println("dealer created and bound")

	// Send a 'Hello' message from the dealer to the router.
	dealer.SendChan <- [][]byte{[]byte("Hello")}
	log.Println("dealer sent 'Hello'")

	// Receve the message as a [][]byte. Since this is
	// a router, the first frame of the message wil
	// be the routing frame.
	request := <-router.RecvChan
	log.Printf("router received '%s' from '%v'", request[1], request[0])

	// Send a reply. First we send the routing frame, which
	// let's the dealer know which client to send the message.
	router.SendChan <- [][]byte{request[0], []byte("World")}
	log.Printf("router sent 'World'")

	// Receive the reply.
	reply := <-dealer.RecvChan
	log.Printf("dealer received '%s'", string(reply[0]))
}
```
## GoDoc
[godoc](https://godoc.org/github.com/zeromq/goczmq)

## See Also
* [Peter Kleiweg's](https://github.com/pebbe) [zmq4](https://github.com/pebbe/zmq4) bindings

## License
This project uses the MPL v2 license, see LICENSE
