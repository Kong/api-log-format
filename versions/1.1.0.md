# Galileo Agents Spec

Agents are libraries that can act as a middlewear / helper utility in capturing incoming requests & outgoing responses data in order to construct [ALF objects][api-log-format] which can be used with the [Galileo][galileo] service.

## Lifecycle

Agents need to be injected at the appropriate point in the request-response lifecycle of the HTTP server, at the start of the request before processing any business logic, and at the end of the response before sending back to the client.

![](https://raw.githubusercontent.com/Mashape/galileo-agent-spec/master/agent-lifecycle.png)

## Configuration

The Agent should expose the following configurations to the user, with fallback to default values when none are provided:

| name                      | type                            | values                        | description                                               | default                          |
| ------------------------- | ------------------------------- | ----------------------------- | --------------------------------------------------------- | -------------------------------- |
| **`SERVICE_TOKEN`**       | `String`                        | `-`                           | **Required**, [Galileo][galileo] Service Token            | `-`                              |
| **`ENVIRONMENT`**         | `String`                        | `-`                           | [Galileo][galileo] Environemnt Slug                       | `-`                              |
| **`LOG_BODIES`**          | `String`                        | `all`, `none`, `request`, `response`  | Capture & send the full bodies of request & response      | `none`                            |
| **`RETRY_COUNT`**         | `Integer`                       | `0-10`                        | Number of retries in case of failures                     | `0`                              |
| **`CONNECTION_TIMEOUT`**  | `Integer`                       | `0-60`                        | Timeout in seconds before aborting the current connection | `30`                              |
| **`FLUSH_TIMEOUT`**       | `Integer`                       | `0-60`                        | Timeout in seconds before flushing the current queue      | `2`                              |
| **`QUEUE_SIZE`**          | `Integer`                       | `0-1000`                      | Total queue size before flushing                          | `1000`                           |
| **`HOST`**                | [`RFC3986`][rfc3986-host]       | `-`                           | DNS Host Address of [Galileo Collector](#collector)       | `collector.galileo.mashape.com`  |
| **`PORT`**                | [`RFC3986`][rfc3986-port]       | `-`                           | Port for Galileo Socket Service                           | `443`                            |
| **`FAIL_LOG`**            | [`RFC3986`][rfc3986-path]       | `-`                           | File system path, storage location for failed requests    | `/dev/null`                      |

## Collector 

The Galileo Collector provides two API endpoints to send data through:

###### `/:version/batch`

> Method: `POST`
> Content-Type: `application/json`

Group multiple [ALF][api-log-format] Objects into an array

```json
[
  {
    "version": "1.1.0",
    "serviceToken": "<my service token>",
    "environment": "PRODUCTION",
    "har": {
      "log": {
        "creator": {
          "name": "HAR Logger",
          "version": "1.0.0"
        },
        "entries": [{...}]
      }
    }
  },
  {
    "version": "1.1.0",
    "serviceToken": "<my service token>",
    "environment": "PRODUCTION",
    "har": {
      "log": {
        "creator": {
          "name": "HAR Logger",
          "version": "1.0.0"
        },
        "entries": [{...}]
      }
    }
  },
  ...
]
```

###### `/:version/single`

> Method: `POST`
> Content-Type: `application/json`

Construct a single [ALF][api-log-format] Object with multiple `entries`.

```json
{
  "version": "1.1.0",
  "serviceToken": "<my service token>",
  "environment": "PRODUCTION",
  "har": {
    "log": {
      "creator": {...},
      "entries": [
        {
          "startedDateTime": "2016-03-13T03:47:16.937Z",
          "serverIPAddress": "10.10.10.10",
          "clientIPAddress": "10.10.10.20",
          "time": 82,
          "request": {...},
          "response": {...},
          "cache": {...},
          "timings": {...}
        },
        {
          "startedDateTime": "2016-03-13T03:47:16.937Z",
          "serverIPAddress": "10.10.10.10",
          "clientIPAddress": "10.10.10.20",
          "time": 82,
          "request": {...},
          "response": {...},
          "cache": {...},
          "timings": {...}
        },
        ...
      ]
    }
  }
}
```

##### Response Types

###### Success: `200 - OK` 

```
TBD
```

###### Partial Success: `207 - Multi-Status`

```
TBD
```

###### Failure: `400 - Bad Request`

###### Failure: `413 - Request Entity Too Large`

The Collector will only accept requests smaller than `500 MB`.

###### Failure: `500 - Server Error`

An un-expected error occurred, please contact [support@mashape.com](mailto:support@mashape.com) if the error continues.

## Agent Behavior

The agent **MUST** follow the following considerations in its operational logic:

### Handling Failure

- On an Interval of `CONNECTION_TIMEOUT` and without a response from the server, the agent should terminate the request.

- On the cases of failure to send data, *(whether through a rejection from [The Collector](#collector), a `CONNECTION_TIMEOUT` event, or otherwise a network failure)*, the agent should retry up to `RETRY_COUNT`, then eventually write to `FAIL_LOG`.

#### `FAIL_LOG`

*TBD*

### Queues

- Collect data and add to a local memory queue before attempting to send to [The Collector](#collector).
- Flush the queue and send to [The Collector](#collector) at:
  - every `FLUSH_TIMEOUT` seconds 
  - every time the queue length reaches `QUEUE_SIZE`
  - when the queue data size reaches [`500 MB`](#failure-413-request-entity-too-large)

### Capturing Data

The Agent will use [API Log Format][api-log-format] to create log entries. Most of the fields in ALF spec are self-explanatory. Check out the [ALF spec][api-log-format] for additional information.

The following rules are beyond the scope of ALF and **MUST** be applied to all agents:

### `clientIPAddress`

- Parse the request headers to obtain the **true** client IP *(see [reference table](#client-ip-headers) below)*.
- fallback to capturing the raw socket client IP if possible.

###### Client IP Headers

| header                | priority | description                                                            |
| --------------------- | -------- | ---------------------------------------------------------------------- |
| `Forwarded`           | 1        | [RFC 7239](https://tools.ietf.org/html/rfc7239) Standard               |
| `X-Real-IP`           | 2        | mostly used in [proxies](http://bit.ly/1Jj9yu6)                        |
| `X-Forwarded-For`     | 3        | common, [non-standard](https://en.wikipedia.org/wiki/X-Forwarded-For)  |
| `Fastly-Client-IP`    | 4        | [Fastly](http://bit.ly/1Rm8pdA)                                        |
| `CF-Connecting-IP`    | 4        | [CloudFlare](http://bit.ly/22ZZ53c)                                    |
| `X-Cluster-Client-IP` | 4        | [Rackspace](http://bit.ly/1KdMKNc), X-Ray                              |
| `Z-Forwarded-For`     | 5        | Z Scaler                                                               |
| `WL-Proxy-Client-IP`  | 5        | Oracle Web Logic                                                       |
| `Proxy-Client-IP`     | 5        | no-references                                                          |

### Request

- Agents cannot obstruct the application natural flow.
  - should not prevent the application from getting the request data for its own processing
  - this is likely framework dependent, or in the case of `PHP`, `Node.js`, the input stream can only be read once, and thus the agent must re-institute the stream so the application logic can continue un-interrupted.

- Agents should attempt to get the **RAW** request as early as possible *(as soon as the last byte is received and before application business logic)*
  - This is to ensure all original headers and body state are captured properly.
  - Body capture should be triggered prior to any processing *(decompression, modification, normalization, etc...)* by the application or application framework.
  - In many languages *(especially: `PHP`, `Node.js`)* reading the input stream is awarded to the **first listener**, the stream is then flushed, thus blocking any following listeners from reading. 
    - The agent should expect this scenario and provide detailed documentation and instructions for proper installment at the appropriate location for capturing the input stream.
    - If the agent is successful in capturing the stream in those scenarios, it should also attempt to redirect the stream for any listeners afterwards, or, provide a raw body property for the application framework to use.

### Response

- The Agent should only attempt to process the response object at the time the application is ready to send it. *(as soon as the last byte is ready to send)*.
  - Just as with the [request](#request) scenario, this is to ensure all possible headers and final modifications to the response objects are captured.
  - Some languages *(such as `PHP`)* would terminate as soon as as the last byte is sent, it is important to trigger the agent logic, before sending the response is started, but not before constructing the response is completed.

### Body Size

Agents **MUST** adhere to the following steps regardless of the `LOG_BODIES` option value:

1. Calculate the request & response body size manually *(in bytes)*
3. Fallback on the `Content-Length` header when manual calculation is not possible.
3. Use `0` when manual calculation is not possible or *response* comes from cache (e.g. `304`)

### Headers

- When not readily available, the agent should attempt to calculate headers sizes: *`ALF.har.log.entries[].request.headersSize`, `ALF.har.log.entries[].response.headersSize`*
  - This can be achieved by reconstructing the [HTTP Message][rfc7230-message] from the start of the HTTP request/response message until (and including) the double `CRLF` before the body.
  - This means calculating the length of the headers as they appeared "on the wire", including the colon, space and `CRLF` between headers.

### Timings

There are 3 mandatory fields that require to be manually calculated *(where applicable)*:

- `ALF.har.log.entries[].timings.send`: duration in milliseconds, between the first byte of request received and *processing time*.
- `ALF.har.log.entries[].timings.wait`: duration in milliseconds, between the *processing time* and the first byte of response sent time.
- `ALF.har.log.entries[].timings.receive`: duration in milliseconds, between the first byte of the response time and the last byte sent time.

###### Note:

The term *"processing time"* can refers to different meanings given the agent type:

- Proxy Agents: sending the last byte to the upstream service
- Native Agents: the start of the application business logic

### Bodies

When the request/response bodies are captured and will be transmitted, they **MUST** be encoded in [base64][rfc3548]:

###### Example

For request bodies:

```js
request.postData = { encoding: 'base64', text: 'BASE64_BODY' }
```

For response bodies:

```js
response.content = { encoding: 'base64', text: 'BASE64_BODY' }
```

[api-log-format]: https://github.com/Mashape/api-log-format
[galileo]: https://getgalileo.io/
[har-spec]: http://www.softwareishard.com/blog/har-12-spec/
[rfc3548]: https://tools.ietf.org/html/rfc3548
[rfc3986-host]: https://tools.ietf.org/html/rfc3986#section-3.2.2
[rfc3986-path]: https://tools.ietf.org/html/rfc3986#section-3.3
[rfc3986-port]: https://tools.ietf.org/html/rfc3986#section-3.2.3
[rfc7230-message]: http://httpwg.github.io/specs/rfc7230.html#http.message
