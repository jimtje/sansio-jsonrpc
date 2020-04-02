# JSON-RPC v2.0 Sans I/O

This project provides [a Sans I/O](https://sans-io.readthedocs.io/) implementation of
[JSON-RPC v 2.0](https://www.jsonrpc.org/specification). This means that the library
handles all of the encoding, decoding, framing, and logic required by the protocol
specification, but it does not implement any I/O (input or output). In order to use this
library, you need to either to use a downstream project that wraps I/O around this
project or else write your own I/O wrapper.

## Client Example

This example illustrates what the library does and how it needs to be integrated into
an I/O framework of your choosing.

```python
from sansio_jsonrpc import JsonRpcPeer, JsonRpcException

client = JsonRpcPeer()
bytes_to_send = client.request(
    callback=handle_response,
    method='open_vault_door',
    args={'employee': 'Mark', 'pin': 1234},
)
```

First, we instantiate the client. We call the `request` method to create a new JSON-RPC
request and convert it into a `bytes` representation that is suitable for sending to the
server. The method also takes a callback that will be invoked when the response to this
request is received. The code for the callback is listed below, but first let's see how
to send this request to the server.

```python
connection.send(bytes_to_send)
received_bytes = connection.recv()
```

Remember, this is a Sans I/O library, so it does not do any networking itself. It is the
caller's responsibility to transfer data to and from the remote machine. This block
shows a hypothetical I/O framework being used to send the pending request and receive
the response as `bytes`, but this can be implemented with any I/O framework of your
choosing.

```python
try:
    client.recv(received_bytes)
except JsonRpcException as jre:
    print('Exception from received data!', jre)
```

In this block, we feed the data received from the remote machine into the client
object's `recv()` method. This method parses the data into a `JsonRpcResponse` object
and then passes that object into the callback. Note that `recv()` always needs to be
wrapped in `try/except`. For example, if the remote machine sent an invalid request that
could not be parsed, that would raise an exception here.

```python
def handle_response(response: JsonRpcResponse) -> None:
    if response.is_success:
        print(response.result)     # {"vault_status": "open"}
    else:
        print('Uh oh, JSON-RPC error code:', response.error.code)
        raise JsonRpcException.exc_from_error(response.error)
```

This final block shows how to write a callback. The callback receives one argument, the
response object. It is a good idea to check if it is a success response or an error
response. If it is a success, then the `response.result` will contain a valid JSON
object (i.e. str, int, float, list, dict, or None). If the response is an error, then
the `response.error` object is set to a `JsonRpcError` object that contains information
about the error. You can also

## Server Example

This example shows how to receive a JSON-RPC request and dispatch a response.

```python
from sansio_jsonrpc import JsonRpcPeer, JsonRpcException, JsonRpcMethodNotFoundError

server = JsonRpcPeer()
```

This example starts out the same way as the client: instantiating a `JsonRpcPeer`
object. Note that the client and server implementation are actually in the same class!

This might be surprising at first but it is the most flexible way to implement JSON-RPC.
For example, the specification says that a notification is a special type of request,
and a request can only be sent from client to server. But there are scenarios where it
would be useful to have the server send notifications to the client, e.g. a
publish/subscribe system. The specification says that:

> One implementation of this specification could easily fill both of those roles, even
> at the same time, to other different clients or the same client.

This is why the library implements both roles in a single class. The choice to name the
object `server` is just a convention to help us remember what's going on.

```python
received_bytes = connection.recv()
```

This block shows the hypothetical I/O framework again. As a server, we just want to wait
for a client to send us something.

```python
def handle_request(request):
    # Your application logic goes here.
    if request.method == 'get_foo':
        return {'foo': 1}
    elif request.method == 'get_bar':
        return {'bar': 2}
    else:
        raise JsonRpcMethodNotFoundError()

try:
    request = server.recv(received_bytes)
    result = handle_request(request)
    bytes_to_send = server.respond_with_result(result)
except JsonRpcException as jre:
    print('Exception from received data!', jre)
    bytes_to_send = server.respond_with_error(jre.get_error())
```

We parse the data received from the network. Then we invoke a handler to service the
request and create a response. That's where your application logic comes in: you need to
do something to handle each available method. If you don't have a handler for a given
method, you should raise `JsonRpcMethodNotFoundError`.

The request and response processing should be wrapped in `try/except` in order to
gracefully handle errors, either due to a misbehaving client or due to any errors in
your handler code. The exception handler should also generate a response, since the
client will want to know that an occurred. We call either
`server.respond_with_result(...)` or `server.respond_with_error(...)` to create a
suitable response in network representation.

```python
connection.send(bytes_to_send)
```

Finally, we use our hypothetical I/O framework to send the server's response data.

## Exceptions

The exception system in this library is designed to make error-handling as Pythonic as
possible, abstracting over the details of the JSON-RPC protocol as much as possible. All
exceptions in this module inherit from `JsonRpcException`, so this is a good choice for
a catch-all exception handler. All exceptions have an error code and a message.
Exceptions can also optionally have a `data` field which can be set to any valid JSON
data type.

There are also specialized errors in the library that correspond to specific JSON-RPC
error codes in the specification. For example, if you receive data that cannot be
decoded into a valid JSON-RPC request, the specification calls for a -32700 error, and
this library raises a `JsonRpcParseError` to make it easy to catch this specific
condition. If the server sends

The specification also allows you to define your own error codes. There are

Exceptions are used in two ways in this library. First of all, the library itself can
raise exceptions. For example in `recv()`, it can raise `JsonRpcParseError` as described
above. The second usage is that the remote peer might send you a response that contains
an error object. In this case, you may want to convert that error into an exception so
you can raise it.

The static method `JsonRpcException.exc_from_error(...)` converts an error into an
exception. This method automatically selects the appropriate subclass of
`JsonRpcException`. For example, if the remote peer sent you error code -32700, this
method converts that into a `JsonRpcParseError`. Some error codes are reserved for
implementations to define their own error codes, e.g. -32099 is reserved in the
specification but not assigned any meaning. If the remote peer sends this error code,
then this method will raise `JsonRpcReservedError`.

The specification also has unreserved error codes that are available for defining
application-specific errors. You can use these in your own code in two different ways.
The first approach is to catch/raise `JsonRpcApplicationError` and set/check the error
code to your application-defined value. The second approach is to create your own
subclass of `JsonRpcApplicationError`. Here's an example:

```python
from sansio_jsonrpc import JsonRpcApplicationError


class MyApplicationError1(JsonRpcApplicationError):
    ERROR_CODE = 1
    ERROR_MESSAGE = "My application error type 1"


class MyApplicationError2(JsonRpcApplicationError):
    ERROR_CODE = 2
    ERROR_MESSAGE = "My application error type 2"
```

The `JsonRpcException.exc_from_error(...)` method will automatically select the
appropriate subclass. For example if the server sends error code 1, then the method will
return a `MyApplicationError1` instance, even though that class is defined in _your
code_! The library uses some metaclass black magic to make this work.

## Developing

The project uses MyPy for type checking and Black for code formatting. Poetry is used to
manage dependencies and to build releases.
