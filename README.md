# `@mswjs/source`

A library that allows you to generate request handlers for [Mock Service Worker](https://github.com/mswjs/msw) (MSW) from various sources (HAR files, OpenAPI specification, runtime).

## Install

```sh
$ npm install @mswjs/source -D
# or
$ yarn add @mswjs/source -D
```

## Usage

This library is meant to generate [request handlers](https://mswjs.io/docs/basics/request-handler)—functions that describe which requests to intercept and how to respond to them. Once you got the handlers from the [source type](#source-types) of your choice, provide them to Mock Service Worker to set up API mocking.

Here's a quick overview of how to enable API mocking in a Node.js process using the request handlers generated by `@mswjs/source`:

```js
import { setupServer } from 'msw/node'
import { fromTraffic, fromOpenApi } from '@mswjs/source'
import har from './github.com.har'
import petstoreSpec from './petstore.json'

const handlers = [...fromTraffic(har), ...fromOpenApi(petstoreSpec)]

const server = setupServer(...handlers)
server.listen()
```

> Learn more about mocking API in a [browser](https://mswjs.io/docs/getting-started/integrate/browser) and in [Node.js](https://mswjs.io/docs/getting-started/integrate/node). With MSW you can intercept requests using the same request handlers in any environment.

## Source types

There are multiple sources to generate request handlers from. Choose one, or multiple, to get started:

- [HTTP Archive (HAR file)](#http-archive)
- [OpenAPI (Swagger)](#openapi-swagger)

## HTTP archive

Use the `fromTraffic` function to generate request handlers from an [HTTP Archive (HAR)](<https://en.wikipedia.org/wiki/HAR_(file_format)>):

```js
import { fromTraffic } from '@mswjs/source'
import har from './github.com.har'

export const handlers = fromTraffic(har)
```

> Note that `*.har` files are written in JSON so you don't have to process their imports in any way.

### Exporting an HAR file

You can generate and export an HAR file using your browser. Please see the detailed instructions on how to do that below.

<details>
  <summary><strong>Google Chrome</strong></summary>
  <ol>
    <li>Navigate to the desired page;</li>
    <li>Open the Dev Tools (<kbd>⌘ + ⌥ + I</kbd> on MacOS; <kbd>Ctrl + Shift + I</kbd> on Windows/Linux);</li>
    <li>Click on the "<em>Network</em>" tab in the Dev Tools;</li>
    <li>Click the <kbd>⤓</kbd> (<em>Export HAR...</em>) icon;</li>
    <li>Choose where to save the HAR file.</li>
  </ol>
</details>

<details>
  <summary><strong>Firefox</strong></summary>
  <ol>
    <li>Navigate to the desired page;</li>
    <li>Open the Dev Tools (<kbd>⌘ + ⌥ + I</kbd> on MacOS; <kbd>Ctrl + Shift + I</kbd> on Windows/Linux);</li>
    <li>Click on the "<em>⇅ Network</em>" tab in the Dev Tools;</li>
    <li>Click on the <kbd>⚙</kbd> icon next to the throttling settings;</li>
    <li>Choose "<em>Save all as HAR</em>";</li>
    <li>Choose where to save the HAR file.</li>
  </ol>
</details>

<details>
  <summary><strong>Safari</strong></summary>
 <ol>
    <li>Navigate to the desired page;</li>
    <li>Open the Dev Tools (<kbd>⌘ + ⌥ + I</kbd>);</li>
    <li>Click on the "<em>⇅ Network</em>" tab in the Dev Tools;</li>
    <li>Click on the <kbd>Export</kbd> button;</li>
    <li>Choose where to save the HAR file.</li>
  </ol>
</details>

### Features

- [Response timing](#response-timing)
- [Response order sensitivity](#response-order-sensitivity)
- [Customizing generated handlers](#customizing-generated-handlers)

#### Response timing

Generated handlers respect the response timing from the respective HAR entries.

Take a look at this archive describing a request/response entry:

```json
{
  "log": {
    "entries": [
      {
        "request": {
          "method": "GET",
          "url": "https://example.com/settings"
        },
        "response": {
          "content": "hello world"
        },
        "time": 554
      }
    ]
  }
}
```

The request handler for `GET https://example.com/settings` will have a _delayed_ response using the `log.entries[i].time` (ms) as the delay duration. Roughly, it can be represented as follows:

```js
rest.get('https://example.com/settings', (req, res, ctx) => {
  return res(ctx.text('hello world'), ctx.delay(554))
})
```

#### Response order sensitivity

If the same request has multiple responses in the archive, those responses will be used sequentially in the handlers.

> Note that this library does a straightforward request URL matching and disregards any other parameters (like request headers or body) when looking up an appropriate chronological response.

Consider the following HAR that has different responses for the same `GET https://example.com/user` endpoint:

```json
{
  "log": {
    "entries": [
      {
        "request": {
          "method": "GET",
          "url": "https://example.com/user"
        },
        "response": {
          "content": {
            "text": "john"
          }
        }
      },
      {
        "request": {
          "method": "GET",
          "url": "https://example.com/user"
        },
        "response": {
          "status": 404,
          "content": {
            "text": "User Not Found"
          }
        }
      }
    ]
  }
}
```

When translated to request handlers, these are the mocked responses you get:

```js
// The first response was "john", so you receive it
// upon the first request.
fetch('https://example.com/user').then((res) => res.text())
// "john"

// The second response, however, was a 404 error.
fetch('https://example.com/user').then((res) => res.text())
// "User Not Found"
```

Note that any subsequent request to the same endpoint **will receive the _latest_ response** it has in the HAR. In the example above, any subsequent request will receive a mocked `404` response.

#### Customizing generated handlers

The `fromTraffic` function accepts an optional second argument, which is a function that maps each network entry in the archive. You can use this function to modify or skip certain entries when generating request handlers.

Here's an example of ignoring all requests to `https://api.github.com` in the HAR file:

```js
fromTraffic(har, (entry) => {
  if (entry.request.url.startsWith('https://api.github.com')) {
    return
  }

  return entry
})
```

> Return the `entry` object if you wish to generate a request handler for it, or return nothing if the current HAR entry must not have any associated handlers.

---

## OpenAPI (Swagger)

Use the `fromOpenApi` function to generate request handlers from an [OpenAPI specification](https://swagger.io/specification) document.

```js
import { fromOpenApi } from '@mswjs/source'

// Import your OpenAPI (Swagger) speficiation (JSON).
import specification from './v2.json'

const handlers = fromOpenApi(specification)
```

OpenAPI support goes far beyond generating mocks from your schemas. Take a look at the list of features that you can utilize:

- References support;
- [Conditional responses](#conditional-responses), accessible on runtime;
- Randomly generated data based on [JSON Schema types](#json-schema);
- Coercing request paths without responses as `501 Not Implemented` responses;

### Response body

A mocked response body is extracted from various places of your specification.

#### Explicit examples

Whenever a response, or its part, has a designated `"example"` property, that example is used as the mocked response value.

```json
{
  "/user": {
    "get": {
      "responses": {
        "200": {
          "example": {
            "id": "abc-123",
            "firstName": "John"
          }
        }
      }
    }
  }
}
```

```js
fetch('/user').then((res) => res.json())
// { "id": "abc-123", "firstName": "John" }
```

> Note that explicit `"example"` **always takes precedence**, being a more reliable source for the response structure.

#### JSON Schema

We support response structures written with JSON Schema. The schema types are used to generate respective random values.

> See the [JSON Schema tests](./test/oas) to learn more about random data generation.

### Conditional responses

A single request may have multiple responses specified in the documentation, separated by the response status code. You can control which mocked response a particular request receives by providing that response code as the `response` query parameter of the request URL.

For example, consider this OpenAPI document that specifies multiple responses for the same request:

```json
{
  "paths": {
    "/user": {
      "get": {
        "responses": {
          "200": {},
          "500": {
            "description": "Server error",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "code": {
                      "type": "string",
                      "pattern": "^ER-[0-9]+$"
                    },
                    "message": {
                      "type": "string"
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

If you wish to model a scenario when the server responds with the 500 response based on the specification above, include the `response=500` query parameter in your request URL:

```js
// This request receives the "500" response
// based on the specification above.
const res = await fetch('/user?response=500')
const json = await res.json()
// { "code": "ER-1443", "message": "Test message" }
```
