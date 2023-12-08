# Reverse Tunneling Dialer

remotedialer creates a two-way connection between a server and a client, so
that a `net.Dial` can be performed on the server and actually connects to the
client running services accessible on localhost or behind a NAT or firewall.

## Architecture

### Abstractions

remotedialer consists of structs that organize and abstract a TCP connection
between a server and a client. Both client and server use ``Session``s, which
provides a means to make a network connection to a remote endpoint and keep
track of incoming connections. A server uses the ``Server`` object which
contains a ``sessionManager`` which governs one or more ``Session``s, while a
client creates a ``Session`` directly. A ``connection`` implements the
``io.Reader`` and ``io.WriteCloser`` interfaces, so it can be read from and
written to directly. The ``connection``'s internal ``readBuffer`` monitors the
size of the data it is carrying and uses ``backPressure`` to pause incoming
data transfer until the amount of data is below a threshold.

![](./docs/remotedialer.png)

### Data flow

![](./docs/remotedialer-flow.png)

A client establishes a session with a server using the server's URL. The server
upgrades the connection to a websocket connection which it uses to create a
``Session``. The client then also creates a ``Session`` with the websocket
connection.

The client sits in front of some kind of HTTP server, most often a kubernetes
API server, and acts as a reverse proxy for that HTTP resource. When a user
requests a resource from this remote resource, request first goes to the
remotedialer server. The application containing the server is responsible for
routing the request to the right client connection.

The request is sent through the websocket connection to the client and read
into the client's connection buffer. A pipe is created between the client and
the HTTP service which continually copies data between each socket. The request
is forwarded through the pipe to the remote service, draining the buffer. The
service's response is copied back to the client's buffer, and then forwarded
back to the server and copied into the server connection's own buffer. As the
user reads the response, the server's buffer is drained.

The pause/resume mechanism checks the size of the buffer for both the client
and server. If it is greater than the threshold, a ``PAUSE`` message is sent
back to the remote connection, as a suggestion not to send any more data. As
the buffer is drained, either by the pipe to the remote HTTP service or the
user's connection, the size is checked again. When it is lower than the
threshold, a ``RESUME`` message is sent, and the data transfer may continue.

### remotedialer in the Rancher ecosystem

remotedialer is used to connect Rancher to the downstream clusters it manages,
enabling a user agent to access the cluster through an endpoint on the Rancher
server. remotedialer is used in three main ways:

#### Agent config and tunnel server

When the agent starts, it initially makes a client connection to the endpoint
`/v3/connect/register`, which runs an authorizer that sets some initial data
about the node. The agent continues to connect to the endpoint `/v3/connect` on
a loop. On each connection, it runs an OnConnect handler which pulls down node
configuration data from `/v3/connect/config`.

#### Steve Aggregation

The steve aggregation server on the agent establishes a remotedialer Session
with Rancher, making the steve API on the downstream cluster accessible from
the Rancher server and facilitating resource watches.

#### Health Check

The clusterconnected controller in Rancher uses the established tunnel to check
that clusters are still responsive and sets alert conditions on the cluster
object if they are not.

## Running Locally

remotedialer provides an example client and server which can be run in
standalone mode FOR TESTING ONLY. These are found in the `server/` and
`client/` directories.`

### Compile

Compile the server and client:

```
go build -o server/server server/main.go
go build -o client/client client/main.go
```

### Run

Start the server first.

```shell
./server/server
```

The server has debug mode off by default. Enable it with `--debug`.

The client proxies requests from the remotedialer server to a web server, so it
needs to be run somewhere where it can access the web server you want to
target. The remotedialer server also needs to be reachable by the client.

For testing purposes, a basic HTTP file server is provided. Build the server with:

```shell
go build -o dummy/dummy dummy/main.go
```

Create a directory with files to serve, then run the web server from that directory:

```shell
mkdir www
cd www
echo 'hello' > bigfile
/path/to/dummy
```

Run the client with

```shell
./client/client
```

Both server and client can be run with even more verbose logging:

```shell
CATTLE_TUNNEL_DATA_DEBUG=true ./server/server --debug
CATTLE_TUNNEL_DATA_DEBUG=true ./client/client
```

### Usage

If the remotedialer server is running on 192.168.0.42, and the web service that
the client can access is running at address 127.0.0.1:8125, make proxied
requests like this:

```shell
curl http://192.168.0.42:8123/client/foo/http/127.0.0.1:8125/bigfile
```

where `foo` is the hardcoded client ID for this test server.

This test server only supports GET requests.

### Usage Local Demo

Server:

```shell
# CATTLE_TUNNEL_DATA_DEBUG=true go run server/main.go --debug

Listening on  :8123
INFO[0084] Handling backend connection request [foo]    
INFO[0096] [001] REQ t=15 http://127.0.0.1:8125/go.mod  
DEBU[0096] CONNECTIONS 8883677790471701682 1            
DEBU[0096] WRITE 417598251398574776 CONNECT      [2]: tcp/127.0.0.1:8125 
DEBU[0096] WRITE 417598251398574777 DATA         [2]: 101 bytes: GET /go.mod HTTP/1.1
Host: 127.0.0.1:8125
User-Agent: Go-http-client/1.1
Accept-Encoding: gzip
 
DEBU[0096] REQUEST 3183994988514175863 DATA         [2]: buffered 
DEBU[0096] ONDATA  [2] 0/1089                           
DEBU[0096] READ    [2] 1089/1089 1089 <nil>             
INFO[0096] [001] REQ OK t=15 http://127.0.0.1:8125/go.mod 
INFO[0096] [001] REQ DONE t=15 http://127.0.0.1:8125/go.mod
```

File Server:

```shell
# go run dummy/main.go

listening  :8125
request 1 4.607209ms
```

Client:

```shell
CATTLE_TUNNEL_DATA_DEBUG=true go run client/main.go --debug

---
INFO[0000] Connecting to proxy                           url="ws://localhost:8123/connect"
DEBU[0005] Wrote ping                                   
DEBU[0010] Wrote ping                                   
DEBU[0011] REQUEST 417598251398574776 CONNECT      [2]: tcp/127.0.0.1:8125 
DEBU[0011] CONNECTIONS 0 1                              
DEBU[0011] REQUEST 417598251398574777 DATA         [2]: buffered 
DEBU[0011] ONDATA  [2] 0/101                            
DEBU[0011] READ    [2] 101/101 101 <nil>                
DEBU[0011] WRITE 3183994988514175863 DATA         [2]: 1089 bytes: HTTP/1.1 200 OK
Accept-Ranges: bytes
Content-Length: 903
Content-Type: text/plain; charset=utf-8
Last-Modified: Fri, 08 Dec 2023 03:21:49 GMT
Date: Fri, 08 Dec 2023 06:09:30 GMT

module github.com/rancher/remotedialer

go 1.17

require (
        github.com/gorilla/mux v1.8.0
        github.com/gorilla/websocket v1.4.1
        github.com/pkg/errors v0.9.1
        github.com/prometheus/client_golang v1.11.1
        github.com/sirupsen/logrus v1.9.0
        github.com/stretchr/testify v1.8.2
)

require (
        github.com/beorn7/perks v1.0.1 // indirect
        github.com/cespare/xxhash/v2 v2.1.1 // indirect
        github.com/davecgh/go-spew v1.1.1 // indirect
        github.com/golang/protobuf v1.4.3 // indirect
        github.com/matttproud/golang_protobuf_extensions v1.0.1 // indirect
        github.com/pmezard/go-difflib v1.0.0 // indirect
        github.com/prometheus/client_model v0.2.0 // indirect
        github.com/prometheus/common v0.26.0 // indirect
        github.com/prometheus/procfs v0.6.0 // indirect
        golang.org/x/sys v0.0.0-20220715151400-c0bba94af5f8 // indirect
        google.golang.org/protobuf v1.26.0-rc.1 // indirect
        gopkg.in/yaml.v3 v3.0.1 // indirect
) 
DEBU[0015] Wrote ping                                   
DEBU[0020] Wrote ping                                   
DEBU[0025] Wrote ping                                   
DEBU[0030] Wrote ping                                   
DEBU[0035] Wrote ping                                   
DEBU[0040] Wrote ping                                   
DEBU[0045] Wrote ping                                   
DEBU[0050] Wrote ping                                   
```

Demo Request:

> Node: the first part `http://127.0.0.1:8123` is the server, `client/foo` is the client, `http/127.0.0.1:8125/go.mod` is the dummy web service.

```shell
# curl http://127.0.0.1:8123/client/foo/http/127.0.0.1:8125/go.mod

---
module github.com/rancher/remotedialer

go 1.17

require (
        github.com/gorilla/mux v1.8.0
        github.com/gorilla/websocket v1.4.1
        github.com/pkg/errors v0.9.1
        github.com/prometheus/client_golang v1.11.1
        github.com/sirupsen/logrus v1.9.0
        github.com/stretchr/testify v1.8.2
)

require (
        github.com/beorn7/perks v1.0.1 // indirect
        github.com/cespare/xxhash/v2 v2.1.1 // indirect
        github.com/davecgh/go-spew v1.1.1 // indirect
        github.com/golang/protobuf v1.4.3 // indirect
        github.com/matttproud/golang_protobuf_extensions v1.0.1 // indirect
        github.com/pmezard/go-difflib v1.0.0 // indirect
        github.com/prometheus/client_model v0.2.0 // indirect
        github.com/prometheus/common v0.26.0 // indirect
        github.com/prometheus/procfs v0.6.0 // indirect
        golang.org/x/sys v0.0.0-20220715151400-c0bba94af5f8 // indirect
        google.golang.org/protobuf v1.26.0-rc.1 // indirect
        gopkg.in/yaml.v3 v3.0.1 // indirect
)
```