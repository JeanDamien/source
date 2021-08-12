# `@mswjs/source`

## Install

```sh
$ npm install @mswjs/source -D
# or
$ yarn add @mswjs/source -D
```

## Browser traffic (HAR file)

You can use an [HTTP Archive (HAR)](<https://en.wikipedia.org/wiki/HAR_(file_format)>) file to generate request handlers the following way:

```js
import { fromTraffic } from '@mswjs/source'
import har from './github.com.har'

export const handlers = fromTraffic(har)
```

### How to export a HAR file

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

### Customizing generated handlers

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

> Do not forget to return the `entry` object if you wish to generate a request handler from it.

## Test runtime

```js
// jest.setup.js
import { setupServer } from 'msw/node'
import { fromRuntime } from '@mswjs/source'

const server = setupServer()

beforeAll(() => {
  server.listen()
  fromRuntime(server)
})
```

## OpenAPI (Swagger)

- Explain what spec properties are used as mocked responses.

```js
import { fromOpenApi } from '@mswjs/source'
import apiDocument from 'api.spec.json'

const apiDocument = fs.readFileSync('spec.json')
export const handlers = fromOpenApi(apiDocument)
```
