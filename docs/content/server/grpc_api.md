# Server GRPC API

Centrifugo also supports [GRPC](https://grpc.io/) API. With GRPC it's possible to communicate with Centrifugo using more compact binary representation of commands and use the power of HTTP/2 which is the transport behind GRPC.

GRPC API is also useful if you want to publish binary data to Centrifugo channels.

You can enable GRPC API in Centrifugo using `grpc_api` option:

```
./centrifugo --config=config.json --grpc_api
```

As always in Centrifugo you can just add `grpc_api` option to configuration file:

```json
{
    ...
    "grpc_api": true
}
```

By default, GRPC will be served on port `10000` but you can change it using `grpc_api_port` option.

Now as soon as Centrifugo started you can send GRPC commands to it. To do this get our API Protocol Buffer definitions [from this file](https://github.com/centrifugal/centrifugo/blob/master/misc/proto/api.proto).

Then see [GRPC docs specific to your language](https://grpc.io/docs/) to find out how to generate client code from definitions and use generated code to communicate with Centrifugo.

## Example for Python

For example for Python you need to run sth like this according to GRPC docs:

```
python -m grpc_tools.protoc -I../../protos --python_out=. --grpc_python_out=. api.proto
```

As soon as you run command you will have 2 generated files: `api_pb2.py` and `api_pb2_grpc.py`. Now all you need is to write simple program that uses generated code and sends GRPC requests to Centrifugo:

```python
import grpc
import api_pb2_grpc as api_grpc
import api_pb2 as api_pb

channel = grpc.insecure_channel('localhost:10000')
stub = api_grpc.CentrifugoStub(channel)

try:
    resp = stub.Info(api_pb.InfoRequest())
except grpc.RpcError as err:
    # GRPC level error.
    print(err.code(), err.details())
else:
    if resp.error.code:
        # Centrifugo server level error.
        print(resp.error.code, resp.error.message)
    else:
        print(resp.result)
```

Note that you need to explicitly handle Centrifugo API level error which is not transformed automatically into GRPC protocol level error.

## Example for Go

Here is a simple example on how to run Centrifugo with GRPC Go client.

You need `protoc`, `protoc-gen-go` and `protoc-gen-go-grpc` installed.

First start Centrifugo itself:

```bash
centrifugo --config config.json --grpc_api
```

In another terminal tab:

```bash
mkdir centrifugo_grpc_example
cd centrifugo_grpc_example/
touch main.go
go mod init centrifugo_example
mkdir centrifugopb
cd centrifugopb
wget https://raw.githubusercontent.com/centrifugal/centrifugo/master/misc/proto/api.proto -O api.proto
```

Then add manually to api.proto on top:

```
option go_package = "./;centrifugopb";
```

You can choose any other package name. Run `protoc` to generate code:

```
protoc -I ./ api.proto --go_out=. --go-grpc_out=.
```

Put the following code to `main.go` file (created on last step above):

```go
package main

import (
	"context"
	"log"
	"time"

	"centrifugo_example/centrifugopb"

	"google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial("localhost:10000", grpc.WithInsecure())
	if err != nil {
		log.Fatalln(err)
	}
	defer conn.Close()
	client := centrifugopb.NewCentrifugoClient(conn)
	for {
		resp, err := client.Publish(context.Background(), &PublishRequest{
			Channel: "chat:index",
			Data:    []byte(`{"input": "hello from GRPC"}`),
		})
		if err != nil {
			log.Printf("Transport level error: %v", err)
		} else {
			if resp.GetError() != nil {
                respError := resp.GetError()
				log.Printf("Error %d (%s)", respError.Code, respError.Message)
			} else {
				log.Println("Successfully published")
			}
		}
		time.Sleep(time.Second)
	}
}
```

Then run:

```bash
go run main.go
```

The program starts and periodically publishes the same payload into `chat:index` channel.

## API key authorization

Available since v2.6.1.

You can also set `grpc_api_key` (string) in Centrifugo configuration to protect GRPC API with key. In this case you should set per RPC metadata with key `authorization` and value `apikey <KEY>`. For example in Go language:

```go
package main

import (
	"context"
	"log"
	"time"

	"centrifugo_example/centrifugopb"
	
	"google.golang.org/grpc"
)

type keyAuth struct {
	key string
}

func (t keyAuth) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
	return map[string]string{
		"authorization": "apikey " + t.key,
	}, nil
}

func (t keyAuth) RequireTransportSecurity() bool {
	return false
}

func main() {
	conn, err := grpc.Dial("localhost:10000", grpc.WithInsecure(), grpc.WithPerRPCCredentials(keyAuth{"xxx"}))
	if err != nil {
		log.Fatalln(err)
	}
	defer conn.Close()
	client := centrifugopb.NewCentrifugoClient(conn)
	for {
		resp, err := client.Publish(context.Background(), &PublishRequest{
			Channel: "chat:index",
			Data:    []byte(`{"input": "hello from GRPC"}`),
		})
		if err != nil {
			log.Printf("Transport level error: %v", err)
		} else {
			if resp.GetError() != nil {
				respError := resp.GetError()
				log.Printf("Error %d (%s)", respError.Code, respError.Message)
			} else {
				log.Println("Successfully published")
			}
		}
		time.Sleep(time.Second)
	}
}
```

For other languages refer to GRPC docs.
