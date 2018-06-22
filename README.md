# gogs-webhook-handler

[![NPM](https://nodei.co/npm/gogs-webhook-handler.png?downloads=true&downloadRank=true)](https://nodei.co/npm/gogs-webhook-handler/)
[![NPM](https://nodei.co/npm-dl/gogs-webhook-handler.png?months=6&height=3)](https://nodei.co/npm/gogs-webhook-handler/)

## Thanks to [github-webhook-handler](https://github.com/rvagg/github-webhook-handler)

[Gogs](https://gogs.io/) is a painless self-hosted Git service. But unfortunately there is no any webhook handler for Gogs. I modified Rod Vagg [@rvagg](https://twitter.com/rvagg)'s [github-webhook-handler](https://github.com/rvagg/github-webhook-handler) so that this package could do well in Gogs.
There are two main differents with Gihub webhook and Gogs webhook:
- In request header, Gogs uses `x-gogs-signature`, `x-gogs-event`, `x-gogs-delivery` instead of `x-hub-signature`, `x-github-event`, `x-github-delivery`
- Gogs uses `SHA256` to generates a hash with your secret instead of `SHA1`

This library is a small handler (or "middleware" if you must) for Node.js web servers that handles all the logic of receiving and verifying webhook requests from Gogs.

## Tips

In Gogs Webhooks settings, Content type must be `application/json`.

`application/x-www-form-urlencoded` won't work at present.

## Example

```js
const http = require('http')
const createHandler = require('gogs-webhook-handler')
const handler = createHandler({ path: '/webhook', secret: 'myhashsecret' })
const process = require('child_process')

http.createServer((req, res) => {
  handler(req, res, err => {
    res.statusCode = 404
    res.end('webhook is running~~')
  })
}).listen(7777)

handler.on('error', err => {
  console.error('Error:', err.message)
})

handler.on('push', event => {
  try{
    process.execSync('git pull')
  } catch (e) {
    process.execSync('git checkout -- "*"')
    process.execSync('git pull')
  }
  process.execSync('npm i')
  process.execSync('npm stop')
  process.execSync('npm start')
})

handler.on('issues', event => {
  console.log('Received an issue event for %s action=%s: #%d %s',
    event.payload.repository.name,
    event.payload.action,
    event.payload.issue.number,
    event.payload.issue.title)
})
```

## API

gogs-webhook-handler exports a single function, use this function to *create* a webhook handler by passing in an *options* object. Your options object should contain:

 * `"path"`: the complete case sensitive path/route to match when looking at `req.url` for incoming requests. Any request not matching this path will cause the callback function to the handler to be called (sometimes called the `next` handler).
 * `"secret"`: this is a hash key used for creating the SHA-1 HMAC signature of the JSON blob sent by GitHub. You should register the same secret key with GitHub. Any request not delivering a `X-Gogs-Signature` that matches the signature generated using this key against the blob will be rejected and cause an `'error'` event (also the callback will be called with an `Error` object).
 * `"events"`: an optional array of whitelisted event types (see: *events.json*). If defined, any incoming request whose `X-Gogs-Event` can't be found in the whitelist will be rejected. If only a single event type is acceptable, this option can also be a string.

The resulting **handler** function acts like a common "middleware" handler that you can insert into a processing chain. It takes `request`, `response`, and `callback` arguments. The `callback` is not called if the request is successfully handled, otherwise it is called either with an `Error` or no arguments.

The **handler** function is also an `EventEmitter` that you can register to listen to any of the Gogs event types. Note you can be specific in your Gogs configuration about which events you wish to receive, or you can send them all. Note that the `"error"` event will be liberally used, even if someone tries the end-point and they can't generate a proper signature, so you should at least register a listener for it or it will throw.

See the [Gogs Webhooks documentation](https://gogs.io/docs/features/webhook) for more details on the events you can receive.

Included in the distribution is an *events.json* file which maps the event names to descriptions taken from the API:

```js
var events = require('gogs-webhook-handler/events')
Object.keys(events).forEach(function (event) {
  console.log(event, '=', events[event])
})
```

Additionally, there is a special `'*'` even you can listen to in order to receive _everything_.

## License

**gogs-webhook-handler** is Copyright (c) 2018 Chen Dingding and licensed under the MIT License. All rights not explicitly granted in the MIT License are reserved. See the included [LICENSE.md](./LICENSE.md) file for more details.
