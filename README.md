# typedjsonrpc
typedjsonrpc is a typed decorator-based json-rpc library for Python. It is influenced by [Flask JSON-RPC](https://github.com/cenobites/flask-jsonrpc) but has some key differences: 

typedjsonrpc...
* allows return type checking
* focuses on easy debugging

## Using typedjsonrpc
### Installation
Clone the repository and install typedjsonrpc:
```
$ git clone https://github.com/palantir/typedjsonrpc.git
$ cd typedjsonrpc
$ pip install .
```
### Project setup
To include typedjsonrpc in your project, use:
```python
from typedjsonrpc.registry import Registry
from typedjsonrpc.server import Server

registry = Registry()
server = Server(registry)
``` 
The registry will keep track of the methods that are available for json-rpc - whenever you annotate a method it will be added to the registry. You can always use the method `rpc.describe()` to get a description of all available methods. `Server` is a uwsgi compatible app that handles the requests. It has a development mode that can be run using `server.run(host, port)`.
### Example usage
Annotate your methods to make them accessible and provide type information:
```python
@registry.method(returns=int, a=int, b=int)
def add(a, b):
    return a + b

@registry.method(returns=str, a=str, b=str)
def concat(a, b):
    return a + b
```
The return type *has* to be declared using the `returns` keyword. For methods that don't return anything, you can use either `type(None)` or just `None`:
```python
@registry.method(returns=type(None), a=str)
def foo(a):
    print(a)
    
@registry.method(returns=None, a=int)
def bar(a):
    print(5 * a)
```

You can use any of the basic json types:

|json type | Python type |
|----------|-------------|
|string    | basestring (Python 2), str (Python 3) |
|number    | int, float  |
|null      | None        |
|boolean   | bool        |
|array     | list        |
|object    | dict        |

Your functions may also accept `*args` and `**kwargs`, but you cannot declare their types. So the correct way to use these would be:
```python
@registry.method(a=str)
def foo(a, *args, **kwargs):
    return a + str(args) + str(kwargs)
```

To check that everything is running properly, try (assuming `add` is declared in your main module):
```
$ curl -XPOST http://localhost:3031/api -d '{"jsonrpc": "2.0", "method":"__main__.add","params": {"a": 5, "b": 7}, "id": "foo"}'
{"jsonrpc": "2.0", "id": "foo", "result": 12}
```
Passing any non-integer arguments into `add` will raise a `TypeError`.

### Batching
You can send a list of json objects as one request and will receive a list of response objects in return. These response objects can be mapped back to the request objects using the `id`. Here's an example of calling the `add` method with two sets of parameters:
```
$ curl -XPOST http://localhost:3031/api -d '[{"jsonrpc": "2.0", "method":"__main__.add","params": {"a": 5, "b": 7}, "id": "foo"}, {"jsonrpc": "2.0", "method":"__main__.add","params": {"a": 42, "b": 1337}, "id": "bar"}]'
[{"jsonrpc": "2.0", "id": "foo", "result": 12}, {"jsonrpc": "2.0", "id": "bar", "result": 1379}]
```

### Debugging
If you create the registry with the parameter `debug=True`, you'll be able to use [werkzeug's debugger](http://werkzeug.pocoo.org/docs/0.10/debug/). In that case, if there is an error during execution - e.g. you tried to use a string as one of the parameters for `add` - the response will contain an error object with a `debug_url`:
```
$ curl -XPOST http://localhost:3031/api -d '{"jsonrpc": "2.0", "method":"__main__.add","params": {"a": 42, "b": "hello"}, "id": "bar"}'
{"jsonrpc": "2.0", "id": "bar", "error": {"message": "Invalid params", "code": -32602, "data": {"message": "Value 'hello' for parameter 'b' is not of expected type <type 'int'>.", "debug_url": "/debug/1234567890"}}}
```
This tells you to find the traceback interpreter at `localhost:3031/debug/1234567890`.

## Dependencies 
* werkzeug
* wrapt
