# [Doc] QBit Microservice WebSocket wire protocol draft 01   JSON ASCII method batching RPC


QBit uses JSON and ASCII to form a wire protocol for sending messages in batches.
It is a primarily intended and implemented as the way to invoke methods on services over WebSocket.
The "wire protocol" could be used with other forms like an HTTP POST, Kafka, 0MQ, etc.



# DRAFT

In progress. QBit has a working version of this specification.
This protocol allows for high-speed invocation of methods.
It does not use straight JSON to avoid double
unencoding and double encoding JSON strings which can be CPU intensive.

Early attempts used nested JSON where the message format was itself in JSON as was the arguments, etc.
The issue was the expense of encoding and decoding strings twice.
Also by making the message delimitation ASCII delimiters and not JSON, we can split a large
group of messages into chunks to be parsed, and pre-marshaled in other threads.


## Motivation

QBit desires to be a cross platform microservice lib. This means that clients can/should/will be
written in other languages. QBit supports REST/JSON however, there are speed limitations associated with
request reply protocols like HTTP. Microservice uses REST/JSON as a great cross-platform multi-lingual
service delivery which embraces polylingual/polyglot development.


QBit does not intend on being a Java only solution.
There will/should be JavaScript and Python clients for example.


A good explanation for what we are trying to do is on the [NetFlix API Blog Post](http://techblog.netflix.com/2013/01/optimizing-netflix-api.html). The blog is good fodder for an explanation of what QBit wire protocol for WebSocket (which could also be HTTP posted or sent over Kafka, etc.) is trying to do.  QBit is attempting to collapse many messages into one message (POST or WebSocket call or Kafka) for two reasons:
 1) reduce overhead of thread handoff from queue to queue when going in the actual service implementation
 2) Reduce calls from server to server or device to server.
 3) Reduced IO one HTTP post instead of several

QBit is not done being optimized but it can already send and receive 680K method calls to a Java service a second (1.2 million messages) from a single WebSocket connection. The protocol is ASCII delimiters for messages (which can be events, method calls or returns) with JSON for arguments to service methods.


## Reference implementation

The reference implementation for QBit JSON WebSocket protocol is
`io.advantageous.qbit.spi.BoonProtocolEncoder` and
`io.advantageous.qbit.spi.BoonProtocolParser`.

If this draft specification is vague, the reference implementation is the final word.

As fast as the `BoonProtocolEncoder` and `BoonProtocolParser` is just the tip of the iceberg of what is possible. They can be rewritten to use a full descent parser with an index overlay which should allow
for better separation of marshaling and JSON parsing to happen in other threads.

Also there will be a full binary version for encoding based on BNSF.


## Protocol


### Preamble
***Octet***: 0
Description: Protocol Version Marker
Required Value: 0x1c

This just marks the stream as a QJAC (QBit JSON ASCII Connection protocol)


***Octet***: 1
Description: Protocol Message Type Marker
Supported values: 'm', 'g', 'r'

An 'm' means single method call
A 'g' means a group of messages (events, method calls, and responses)
An 'r' means a single response to a method call
An 'e' means an event message


What comes next depends on if the message is an 'm', 'g', 'e', or 'r'.


### Message Group messages
***Octet***: 2


A group message is a collection of messages.

Each messages has a version since it is only one octet.


***Octet***: 0
Description: Protocol Version Marker
Required Value: 0x1c

This just marks the stream as a QJAC (QBit JSON ASCII Connection protocol)


***Octet***: 1
Description: Protocol Message Type Marker
Supported values:  'r', 'm', or 'e' (response, method call or event).


***Octet***: 2
Description: PROTOCOL_SEPARATOR
Required Value: 0x1d


Beyond the third octet the values are separated by a PROTOCOL SEPARATOR 0x1d.

Messages within a group are delimited with PROTOCOL_MESSAGE_SEPARATOR (0x1f).



##Method Call Message

A method call consists of the following parts described in this JSON representation:


#### JSON like Representation of major parts of a method call
```json

[preamble, messageId, address, returnAddress, headers, parameters,
 objectName, methodName, timestamp, arguments]

```

Just like in the preamble described before delimiters are ASCII characters.


The major parts of the method call are delimited by PROTOCOL_SEPARATOR (0x1d).

The headers are delimited by PROTOCOL_KEY_HEADER_DELIM (0x1a),
PROTOCOL_VALUE_HEADER_DELIM (0x15), PROTOCOL_ENTRY_HEADER_DELIM(0x19).

The parameters are also delimited by PROTOCOL_KEY_HEADER_DELIM (0x1a),
PROTOCOL_VALUE_HEADER_DELIM (0x15), PROTOCOL_ENTRY_HEADER_DELIM(0x19).

The arguments for the method are delimited by PROTOCOL_ARG_SEPARATOR (0x1e).


Another way to look at a message with a JSON mindset is this:

#### JSON like representation showing headers/params as a map and arguments as a list
```json

    [preamble, messageId, address, returnAddress, {'h1':'v1'}, {'p1':'v1'},
    objectName, methodName, timestamp, [arg0, arg1, arg2]]

```

Headers, parameters, address, objectName, methodName, timestamp, and arguments
are optional.

This is possible:

```json
    [preamble, null, address, returnAddress, {'h1':'v1'}, {'p1':'v1'},
    objectName, methodName, null, null]

```

 As is this:


```json
    [preamble, null, address, returnAddress, {'h1':'v1'}, {'p1':'v1'},
    null, null, null, null]

```




***Value*** 1
Description: Message ID
Supported Value: a numeric usually sequential message identification number.


***Value*** 2
Description: Address
Supported Value: The "to address" where is the message going. Some form of URL.


***Value*** 3
Description: Return address
Supported Value: The "return address" where should responses to this message be sent. Some form of URL.



***Value*** 4:
Description: Headers
Supported Value: Headers can contain additional routing or security tokens.
Note: An empty headers section is just two repeated PROTOCOL SEPARATOR 0x1d.



A header can have multiple values:


Headers consist of key names with many values.

Each header is delimited by PROTOCOL_KEY_HEADER_DELIM (0x1a) and
ends with PROTOCOL_ENTRY_HEADER_DELIM(0x19). Each key can have many values.
Values are delimited by PROTOCOL_VALUE_HEADER_DELIM (0x15).
Headers are like maps of JSON objects.


#### JSON representation of a header.
```json

{
   "Accept": ["jpeg", "gif", "png"],
   "Name" : ["bob", 1]
}

```

***Value*** 5:
Description: Parameter
Supported Values: Parameters are treated just like headers.



ASCII was picked to delimit headers and params so JSON string encoding and decoding would not need to be performed (or rather performed twice if you embed a JSON string inside of a JSON string)
which can be expensive if you are sending millions of messages or reading million of messages from IO.


***Value*** 6:
Description: Object Name
Supported Values: Name of the object we are invoking

This value could be empty if the address had enough information for the
invocation. Or the address could be just the address of the module, and
this is the object (or service name) inside of that module.


***Value*** 7:
Description: Method Name
Supported Values: Name of the method on the object or service that
we are invoking.

This value could be empty if the address had enough information for the
invocation.


***Value*** 8:
Description: Timestamp
Supported Values: Timestamp of when the method was sent

If this value is empty a timestamp will be generated on the server when the method is
received and that will become the timestamp for the method.

***Value*** 9:
Description: Arguments
Supported Values: Arguments to the method invocation.

Each argument is separated by a PROTOCOL_ARG_SEPARATOR (0x1e), and then
converted into JSON copying the fields/properties from Java data value
to a JSON object (map). Jackson, GSON, Boon could be used.
The mapping is defined by the framework and annotations for that framework.

#### JSON Representation of a group full of methods

```json

    [PROTOCOL_PREAMBLE_VERSION, 'g',
      [
        [PROTOCOL_PREAMBLE_VERSION, 'a', {'header1':'value1'}
         {'param1':'pvalue1'}, 'localhost:8080/module1',
         'returnAddress', 'helloService', ["hi", "mom", 1]
        ],
        [PROTOCOL_PREAMBLE_VERSION, 'a', {'header1':'value1'}
             {'param1':'pvalue1'}, 'localhost:8080/module2',
             'returnAddress', 'goodBye', ["goodbye", "mom", 1]
        ]
      ]
    ]
```


Address can be blank if method name, and object name is set.
The returnAddress may be omitted if it is the same as the group ids return address.




##Response Message

A response typically to a method call.

#### JSON like Representation of major parts of a method call
```json

[preamble, messageId, address, returnAddress, headers, parameters,
 objectName, methodName, timestamp, wasErrors, responseMessage]

```

The messageId will match the messageId of the method call.
The address will either be the address of the method call or
the concatenation of "$address.$objectName.$methodName" or blank if the objectName,
methodName are set.

The timestamp will also match the timestamp of the method call.
wasErrors is true if the responseMessage represents an exception rather
than a normal response.

Just like in the preamble described before delimiters are ASCII characters.


The major parts of the response are delimited by PROTOCOL_SEPARATOR (0x1d).

The headers are delimited by PROTOCOL_KEY_HEADER_DELIM (0x1a),
PROTOCOL_VALUE_HEADER_DELIM (0x15), PROTOCOL_ENTRY_HEADER_DELIM(0x19).

The parameters are also delimited by PROTOCOL_KEY_HEADER_DELIM (0x1a),
PROTOCOL_VALUE_HEADER_DELIM (0x15), PROTOCOL_ENTRY_HEADER_DELIM(0x19).



***Value*** 1
Description: Message ID
Supported Value: a numeric usually sequential message identification number.
This will correlate to the method call for this response.


***Value*** 2
Description: Address
Supported Value: The "to address" where is the message going. Some form of URL.


***Value*** 3
Description: Return address
Supported Value: The "return address" where this responses should be sent.
Can be some form of a URL.


***Value*** 4:
Description: Headers
Supported Value: Headers can contain additional routing or security tokens.
Note: An empty headers section is just two repeated PROTOCOL SEPARATOR 0x1d.



A header can have multiple values:


Headers consist of key names with many values.

Each header is delimited by PROTOCOL_KEY_HEADER_DELIM (0x1a) and
ends with PROTOCOL_ENTRY_HEADER_DELIM(0x19). Each key can have many values.
Values are delimited by PROTOCOL_VALUE_HEADER_DELIM (0x15).
Headers are like maps of JSON objects.


***Value*** 5:
Description: Parameter
Supported Values: Parameters are treated just like headers.



ASCII was picked to delimit headers and params so JSON string encoding and decoding would not need to be performed
which can be expensive if you are sending millions of messages or reading million of messages from IO.


***Value*** 6:
Description: Object Name
Supported Values: Name of the object we are invoking

This value could be empty if the address had enough information for the
invocation. Or the address could be just the address of the module, and
this is the object (or service name) inside of that module.


***Value*** 7:
Description: Method Name
Supported Values: Name of the method on the object or service that
we are invoking.

This value could be empty if the address had enough information for the
invocation.


***Value*** 8:
Description: Timestamp
Supported Values: Timestamp of when the method was sent

If this value is empty a timestamp will be generated on the server when the method is
received and that will become the timestamp for the method.

***Value*** 9:
Description: was errors
Supported Values: Is the body a response or an exception.

1 denotes there are errors
0 denotes that there are no errors.



***Value*** 10:
Description: response body
Supported Values: The return value of the method call encoded as JSON.




##Event Message

An event is an unsolicited message to a consumer or subscriber.


#### JSON like Representation of major parts of a method call
```json

[preamble, messageId, topic, headers, timestamp, messageBody]

```



The major parts of the response are delimited by PROTOCOL_SEPARATOR (0x1d).

The headers are delimited by PROTOCOL_KEY_HEADER_DELIM (0x1a),
PROTOCOL_VALUE_HEADER_DELIM (0x15), PROTOCOL_ENTRY_HEADER_DELIM(0x19).

The parameters are also delimited by PROTOCOL_KEY_HEADER_DELIM (0x1a),
PROTOCOL_VALUE_HEADER_DELIM (0x15), PROTOCOL_ENTRY_HEADER_DELIM(0x19).



***Value*** 1
Description: Message ID
Supported Value: a numeric usually sequential message identification number.
This will correlate to the method call for this response.


***Value*** 2
Description: Topic
Supported Value: Typically some sort of unique string that identifies a topic
that can be consumed or subscribed to.


***Value*** 4:
Description: Headers
Supported Value: Headers can contain additional routing or security tokens.
Note: An empty headers section is just two repeated PROTOCOL SEPARATOR 0x1d.

Each header is delimited by PROTOCOL_KEY_HEADER_DELIM (0x1a) and
ends with PROTOCOL_ENTRY_HEADER_DELIM(0x19). Each key can have many values.
Values are delimited by PROTOCOL_VALUE_HEADER_DELIM (0x15).
Headers are like maps of JSON objects.


***Value*** 5:
Description: Timestamp
Supported Values: Timestamp of when the method was sent

If this value is empty a timestamp will be generated on the server when the method is
received and that will become the timestamp for the method.

***Value*** 6:
Description: event message body
Supported Values: The body of the event encoded as JSON.
