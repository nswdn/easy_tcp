# easy_tcp

Create this project for creating a tcp server/client faster and process bytes easier.

### Install:
```shell script
go get -u github.com/nswdn/easy_tcp
```

### Usage:
To build a tcp server:
``` go
import (
	"fmt"
	"net"
	"sync"
	"testing"
)

// decode your data in this func
func StringDecoder(b []byte) (interface{}, error) {
	return string(b), nil
}

// encode your data in this func
func StringEncoder(msg interface{}) []byte {
	return []byte(msg.(string))
}

type StringHandler struct {
	mutex sync.Mutex
	times int
}

// process decoded message
func (t *StringHandler) Handle(ctx ConnectionHandler, msg interface{}) {
	t.mutex.Lock()
	t.times++
	t.mutex.Unlock()

	ctx.Write("copy")
	fmt.Println(t.times)
	fmt.Println("read from client: ", msg)
}

func TestServer(t *testing.T) {
	addr := net.TCPAddr{
		IP:   net.ParseIP("0.0.0.0"),
		Port: 3333,
	}

	tcpServer, e := NewTcpServer(&addr)
	if e != nil {
		panic(e)
	}

	tcpServer.AddEncoder(StringEncoder)
	tcpServer.AddDecoder(StringDecoder)
	tcpServer.AddHandler(new(StringHandler))
	tcpServer.Start()
}

```

To build a tcp client:
```go
package goland

import (
	"fmt"
	"math/rand"
	"net"
	"sync"
	"testing"
)

func Decode(data []byte) (interface{}, error) {
	return string(data), nil
}
func Encode(msg interface{}) []byte {
	return []byte(msg.(string))
}

type ClientHandler struct {
	times int
}

func (h *ClientHandler) Handle(ctx ConnectionHandler, msg interface{}) {
	//fmt.Println("response from server: ", msg)
	h.times++
	fmt.Println(h.times)
	fmt.Println(msg)
	if h.times == 1000 {
		wg.Done()
		ctx.Close()
	}
}

var randomSentences = []string{"Bad days will pass", "Your dream is not dre", "the manner in which someone behaves toward or deals with someone or something.", "是啊是啊", "不是不是"}
var serverAddr = net.TCPAddr{
	IP:   net.ParseIP("0.0.0.0"),
	Port: 3333,
}

var wg = sync.WaitGroup{}

func TestMultiClient(t *testing.T) {
	maxClient := 1000
	wg.Add(maxClient)

	for i := 0; i < maxClient; i++ {
		go func() {
			client, e := NewTcpClient(nil, &serverAddr)
			if e != nil {
				panic(e)
			}

			client.AddEncoder(Encode)
			client.AddDecoder(Decode)
			client.AddHandler(new(ClientHandler))
			client.Dial()

			for j := 0; j < 1000; j++ {
				_, e := client.Write(randomSentences[rand.Intn(len(randomSentences))])
				if e != nil {
					e := client.ReConnect()
					if e != nil {
						return
					}
				}
			}
		}()
	}

	wg.Wait()
}

func TestDial(t *testing.T) {

	client, e := NewTcpClient(nil, &serverAddr)
	if e != nil {
		panic(e)
	}

	client.AddEncoder(Encode)
	client.AddDecoder(Decode)
	client.AddHandler(new(ClientHandler))

	client.Dial()

	client.Write("hello")

	select {}
}
```