LinguaLeo RPC protocol specification
============

### Request
Request should be encoded using json. Format described below:

```json
{
    "id" : "10",
    "v" : "1",
    "method" : "methodName",
    "args": ["array", "of", "args"],
    "reply" : true
}
```

<table>
<tr>
    <th>Parameter</th>
    <th>Format</th>
    <th>Description</th>
    <th>Default</td>
</tr>
<tr>
    <td>id</td>
    <td>Any number</td>
    <td>Unique client identifier (big random)</td>
    <td></td>
</tr>
<tr>
    <td>v</td>
    <td>Any number</td>
    <td>Version of the method</td>
    <td>1</td>
</tr>
<tr>
    <td>method</td>
    <td>String in camelCase</td>
    <td>Method name to be called on server</td>
    <td></td>
</tr>
<tr>
    <td>args</td>
    <td>Associative array, all keys names must be in camelCase</td>
    <td>Arguments will be passed into server's method</td>
    <td><em>(Empty array)</em></td>
</tr>
<tr>
    <td>reply</td>
    <td>Boolean</td>
    <td>Does a request require a reply</td>
    <td>true</td>
</tr>
</table>

### Response
Response should be encoded using json too. Format described below:

```json
{
    "reply" : ["some result"],
    "code" : 1,
    "error": "error occurred"
}
```

<table>
<tr>
    <th>Parameter</th>
    <th>Format</th>
    <th>Description</th>
    <th>Default</th>
</tr>
<tr>
    <td>reply</td>
    <td>Associative array, all keys names must be in camelCase</td>
    <td>Reply data</td>
    <td><em>(Empty array)</em></td>
</tr>
<tr>
    <td>code</td>
    <td>Number</td>
    <td>Reply code</td>
    <td>0</td>
</tr>
<tr>
    <td>error</td>
    <td>String</td>
    <td>Error message</td>
    <td><em>(Empty string)</em></td>
</tr>
</table>

If no error occurred error code must be set to 0 and error message to empty string.

There is common errors codes:
* 1 - `Method not found`
* 2 - `Version not supported`

## Redis implementation details

Implementation uses Redis lists to organize queues.

#### Client flow:

* Client forms request
* Client pushes encoded request (using `LPUSH`) to Redis list, key format must be `server.[endpoint]`
* If reply is needed then client waits (using `BRPOP`) until key with name `client.[client id]` is appeared

#### Server flow:

* Server gets the encoded request from list (using `BRPOP`). List key format is `server.[endpoint]`
* Server processes the request
* If reply is needed then server pushes (using `LPUSH`) message to list. List key name must be `client.[client id]`
* The server should set expiration on the client key for 10 seconds

## Service definition ##

The service should implement special methods.

### discover ###

- - -

The command returns the service definition (description, methods).

#### Request ####

* version: `1`
* method: `discover`
* args: `['methodName1', 'methodNameN']`, _optional_

Example:

```json
{
    "v": 1,
    "method": "discover"
}
```

#### Response ####

The reply has two properties on the root: 
* `service` - description of the service, _optional_
* `methods` - list of existed methods

A method is described by three properties:
* `description` - description of the method, _optional_
* `parameters` - existed parameters (for more information see *Example*), _optional_
* `returns` - result data type, _optional_

A parameter is described by two properties:
* `type` - data type (available types: `string`, `integer`, `float`, `boolean`, `array`, `_schema_`, default `string`)
* `default` - default value, _optional_

Example:

```json
{
    "reply": {
        "service": "Calculator",
        "methods": {
            "add": {
                "parameters": [  // an array indicates positional parameters
                    { "type": "integer", "default": 0 },
                    { "type": "integer", "default": 0 }
                ],
                "returns": "integer"
            },
            "divide": {
                "description": "Do division",
                "parameters": {
                    "divisor": { "type": "integer" },
                    "dividend": { "type": "integer" }
                },
                "returns": "float"
            },
            "simple": {}, // no requirements on parameters or return type
            "getAddress" : {
                "description": "Takes a person and returns an address",
                "parameters": {
                    "person": {
                         // for the type we use a schema
                        "type": { 
                            "firstName": { "type": "string" }, 
                            "lastName": { "type": "string" } 
                        }
                    },
                },
                // use a schema for the return type
                "returns": {
                    "street": { "type": "string" },
                    "zip": { "type": "string" }, 
                    "state": { "type": "string" },
                    "town": { "type": "string" }
                }
            }
        }
    }
}
```

### getInfo ###

- - -

The method returns information and statistics about the service.

#### Request ####

* version: `1`
* method: `getInfo`

Example:

```json
{
    "v": 1,
    "method": "getInfo"
}
```

#### Response ####

* `uptime_in_seconds` - service uptime in seconds
* `uptime_in_days` - service uptime in days
* `used_memory` - used memory in bytes
* `used_memory_human` - used memory for a human (example `1.51M`, `1.3G`)
* `used_memory_peak` - used memory peak in bytes
* `used_memory_peak_human` - used memory peak for a human (see `used_memory_human`)
* `total_connections_received` - total number of connections accepted by the server
* `total_methods_processed` - total number of methods processed by the server
* `connected_redis` - number of connected redis servers
* `redisXXX` - redis server description (example: `ip address:port`)
* `latest_method_usec` - duration of the latest method call in microseconds
* `methods_per_sec` - number of methods processed per second

Example:

```json
{
    "reply": {
        "uptime_in_seconds": 128300441,
        "uptime_in_days": 1484,
        "used_memory": 484211234,
        "used_memory_human": "461M",
        "used_memory_peak": 641233123,
        "used_memory_peak_human": "611M",
        "total_connections_received": 12313,
        "total_methods_processed": 912381,
        "connected_redis": 2,
        "redis1": "127.0.0.1:6379",
        "redis2": "127.0.0.1:6480",
        "latest_method_usec": 123124124124,
        "methods_per_sec": 123
    }
}
```
