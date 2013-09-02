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
