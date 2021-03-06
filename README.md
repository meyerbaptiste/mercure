# Mercure, Server-Sent Live Updates
*Protocol and Reference Implementation*

Mercure is a protocol allowing to push data updates to web browsers and other HTTP clients in a convenient, fast, reliable and battery-efficient way.
It is especially useful to publish real-time updates of resources served through web APIs, to reactive web and mobile apps.

The protocol has been published as an Internet Draft that [is maintained in this repository](spec/mercure.md).

A reference, production-grade, implementation of **a Mercure hub** (the server) is also available here.
It's a free software (AGPL) written in Go. It is provided along with a library that can be used in any Go application to implement the Mercure protocol directly (without a hub) and an official Docker image.

In addition, a managed and high-scalability version of Mercure is [available in private beta](mailto:dunglas+mercure@gmail.com?subject=I%27m%20interested%20in%20Mercure%27s%20private%20beta).

## Mercure in a Few Words

* native browser support, no lib nor SDK required (built on top of [server-sent events](https://www.smashingmagazine.com/2018/02/sse-websockets-data-flow-http2/))
* compatible with all existing servers, even those who don't support persistent connections (serverless architecture, PHP, FastCGI...)
* built-in connection re-establishment and state reconciliation
* [JWT](https://jwt.io/)-based authorization mechanism (securely dispatch an update to some selected subscribers)
* performant, leverages [HTTP/2 multiplexing](https://developers.google.com/web/fundamentals/performance/http2/#request_and_response_multiplexing)
* designed with [hypermedia in mind](https://en.wikipedia.org/wiki/HATEOAS), also supports [GraphQL](https://graphql.org/)
* auto-discoverable through [web linking](https://tools.ietf.org/html/rfc5988)
* message encryption support
* can work with old browsers (IE7+) using an `EventSource` polyfill
* [connection-less push](https://html.spec.whatwg.org/multipage/server-sent-events.html#eventsource-push) in controlled environments (e.g. browsers on mobile handsets tied to specific carriers)

The reference hub implementation:

* Fast, written in Go
* Works everywhere: static binaries and Docker images available
* Automatic HTTP/2 and HTTPS (using Let's Encrypt) support
* Cloud Native, follows [the Twelve-Factor App](https://12factor.net) methodoloy
* Open source (AGPL)

# Examples

Example implementation of a client (the subscriber), in JavaScript:

```javascript
// The subscriber subscribes to updates for the https://example.com/foo topic
// and to any topic matching https://example.com/books/{name}
const url = new URL('https://hub.example.com/subscribe');
url.searchParams.append('topic', 'https://example.com/books/{id}');
url.searchParams.append('topic', 'https://example.com/users/dunglas');

const eventSource = new EventSource(url);

// The callback will be called every time an update is published
eventSource.onmessage = e => console.log(e); // do something with the payload
```

Optionaly, the hub URL can be automatically discovered:

```javascript
fetch('https://example.com/books/1') // Has this header `Link: <https://hub.example.com/subscribe>; rel="mercure"`
    .then(response => {
        // Extract the hub URL from the Link header
        const hubUrl = response.headers.get('Link').match(/<(.*)>.*rel="mercure".*/)[1];
        // Subscribe to updates using the first snippet, do something with response's body...
    });
```

To dispatch an update, the application server (the publisher) just need to send a `POST` HTTP request to the hub.
Example using [Node.js](https://nodejs.org/) / [Serverless](https://serverless.com/):

```javascript
// Handle a POST, PUT, PATCH or DELETE request or finish an async job...
// and notify the hub
const https = require('https');
const querystring = require('querystring');

const postData = querystring.stringify({
    'topic': 'https://example.com/books/1',
    'data': JSON.stringify({ foo: 'updated value' }),
});

const req = https.request({
    hostname: 'hub.example.com',
    port: '443',
    path: '/publish',
    method: 'POST',
    headers: {
        Authorization: 'Bearer <valid-jwt-token>', // the JWT key must be shared between the hub and the server
        'Content-Type': 'application/x-www-form-urlencoded',
        'Content-Length': Buffer.byteLength(postData),
    }
}, /* optional response handler */);
req.write(postData);
req.end();

// You'll probably prefer use the request library or the node-fetch polyfill in real projects,
// but any HTTP client, written in any language, will be just fine.
```

Examples in other languages are available in [the `example/` directory](example/).

## Use Cases

### Live Availability

* a Progressive Web App retrieves the availability status of a product from a REST API and displays it: only one is still available
* 3 minutes later, the last product is bought by another customer
* the PWA's view instantly show that this product isn't available anymore

### Asynchronous Jobs

* a Progressive Web App tell the server to compute a report, this task is costly and will some time to finish
* the server delegates the computation of the report on an asynchronous worker (using message queue), and close the connection with the PWA
* the worker sends the report to the PWA when it is computed

### Collaborative Editing

* a webapp allows several users to edit the same document concurently
* changes made are immediately broadcasted to all connected users

**Mercure gets you covered!**

## Protocol Specification

The full protocol specification can be found in [`spec/mercure.md`](spec/mercure.md).
It is also available as an [IETF's Internet Draft](https://www.ietf.org/id-info/),
and is designed to be published as a RFC.

## Hub Implementation

### Usage

### Managed Version

A managed, high-scalability version of Mercure is available in private beta.
[Drop us a mail](mailto:dunglas+mercure@gmail.com?subject=I%27m%20interested%20in%20Mercure%27s%20private%20beta) for details and pricing.

#### Prebuilt Binary

Grab a binary from the release page and run:

    PUBLISHER_JWT_KEY=myPublisherKey SUBSCRIBER_JWT_KEY=mySubcriberKey ADDR=:3000 DEMO=1 ./mercure

The server is now available on `http://localhost:3000`, with the demo mode enabled.

To run it in production mode, and generate automatically a Let's Encrypt TLS certificate, just run the following command as root:

    PUBLISHER_JWT_KEY=myPublisherKey SUBSCRIBER_JWT_KEY=mySubcriberKey ACME_HOSTS=example.com ./mercure

The value of the `ACME_HOSTS` environment variable must be updated to match your domain name(s).
A Let's Enctypt TLS certificate will be automatically generated.
If you omit this variable, the server will be exposed on an (unsecure) HTTP connection.

When the server is up and running, the following endpoints are available:

* `POST https://example.com/publish`: to publish updates
* `GET https://example.com/subscribe`: to subscribe to updates

See [the protocol](spec/mercure.md) for further informations.

To compile the development version and register the demo page, see [CONTRIBUTING.md](CONTRIBUTING.md#hub).

#### Docker Image

A Docker image is available on Docker Hub. The following command is enough to get a working server in demo mode:

    docker run \
        -e PUBLISHER_JWT_KEY=myPublisherKey -e SUBSCRIBER_JWT_KEY=mySubcriberKey \
        -p 80:80 \
        dunglas/mercure

The server, in demo mode, is available on `http://localhost:80`.

In production, run:

    docker run \
        -e PUBLISHER_JWT_KEY=myPublisherKey -e SUBSCRIBER_JWT_KEY=mySubcriberKey -e ACME_HOSTS=example.com \
        -p 80:80 -p 443:443 \
        dunglas/mercure

Be sure to update the value of `ACME_HOSTS` to match your domain name(s), a Let's Encrypt TLS certificate will be automatically generated.

### Environment Variables

* `ACME_CERT_DIR`: the directory where to store Let's Encrypt certificates
* `ACME_HOSTS`: a comma separated list of host for which Let's Encrypt certificates must be issues
* `ADDR`: the address to listen on (example: `127.0.0.1:3000`, default to `:80` or `:http` or `:https` depending if HTTPS is enabled or not)
* `ALLOW_ANONYMOUS`:  set to `1` to allow subscribers with no valid JWT to connect
* `DB_PATH`: the path of the [bbolt](https://github.com/etcd-io/bbolt) database (default to `updates.db` in the current directory)
* `CERT_FILE`: a cert file (to use a custom certificate)
* `CERT_KEY`: a cert key (to use a custom certificate)
* `CORS_ALLOWED_ORIGINS`: a comma separated list of hosts allowed CORS origins
* `DEBUG`: set to `1` to enable the debug mode (prints recovery stack traces)
* `DEMO`: set to `1` to enable the demo mode (automatically enabled when `DEBUG=1`)
* `LOG_FORMAT`: the log format, can be `JSON`, `FLUENTD` or `TEXT` (default)
* `PUBLISHER_JWT_KEY`: must contain the secret key to valid publishers' JWT
* `SUBSCRIBER_JWT_KEY`: must contain the secret key to valid subscribers' JWT

If `ACME_HOSTS` or both `CERT_FILE` and `CERT_KEY` are provided, an HTTPS server supporting HTTP/2 connection will be started.
If not, an HTTP server will be started (**not secure**).

## FAQ

### How to Use Mercure with GraphQL?

Because they are delivery agnostic, Mercure plays particulary well with [GraphQL's subscriptions](https://facebook.github.io/graphql/draft/#sec-Subscription).

In response to the subscription query, the GraphQL server may return a corresponding topic URL.
The client can then subscribe to the Mercure's event stream corresponding to this subscription by creating a new `EventSource` with an URL like `https://hub.example.com?topic=https://example.com/subscriptions/<subscription-id>` as parameter.

Updates for the given subscription can then be sent from the GraphQL server to the clients through the Mercure hub (in the `data` property of the server-sent event).

To unsubscribe, the client just calls `EventSource.close()`.

Mercure can easily be integrated with Apollo GraphQL by creating [a dedicated transport](https://github.com/apollographql/graphql-subscriptions).

### What's the Difference Between Mercure and WebSocket?

[WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) is a low leve and bidirectional protocol. Mercure is a high level and unidirectional protocol (servers-to-clients, but we will come back to that later).
Unlike Mercure (which is built on top of Server-Sent Events), WebSocket [is not designed to leverage HTTP/2](https://www.infoq.com/articles/websocket-and-http2-coexist).

Also, Mercure provides convenient built-in features (authorization, re-connection, state reconciliation...) while with WebSocket, you need to implement them yourself.

HTTP/2 connections are multiplexed and bidirectional by defaul (it was not the case of HTTP/1).
Even if Mercure is unidirectional, when using it over a h2 connection (recommended), your app can receive data through Server-Sent Events, and send data to the server with regular `POST` (or `PUT`/`PATCH`/`DELETE`) requests, with no overhead.

Basically, in most cases Mercure can be used as a modern, easier to use replacement for WebSocket, but it is a higher level protocol.

### What's the Difference Between Mercure and WebSub?

[WebSub](https://www.w3.org/TR/websub/) is a server-to-server protocol while Mercure is mainly a server-to-client protocol (that can also be used for server-to-server communication, but it's not is main interest).

Mercure has been heavily inspired by WebSub, and we tried to make the protocol as close a possible from the WebSub one.

Mercure uses Server-Sent Events to dispatch the updates, while WebSub use `POST` requests. Also, Mercure has an advanced authorization mechanism, and allows to subscribe to several topics with only one connection using templated URIs.

### What's the Difference Between Mercure and Web Push?

The [Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API) is [mainly designed](https://developers.google.com/web/fundamentals/push-notifications/) to send [notifications](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API) to devices currently not connected to the application.
In most implementations, the size of the payload to dispatch is very limited, and the messages are sent through the proprietary APIs and servers of the browsers' and operating systems' vendors.

On the other hand, Mercure is designed to send live updates to devices currently connected to the web or mobile app. The payload is not limited, and the message goes directly from your servers to the clients.

In summary, use the Push API to send notifications to offline users (that will be available in Chrome, Android and iOS's notification centers), and use Mercure to receive live updates when the user is using the app.

## Resources

* [JavaScript library to parse `Link` headers](https://github.com/thlorenz/parse-link-header)
* [JavaScript library to decrypt JWE using the WebCrypto API](https://github.com/square/js-jose)
* [`EventSource` polyfill for old browsers](https://github.com/Yaffle/EventSource)
* [`EventSource` implementation for Node](https://github.com/EventSource/eventsource)
* [Server-Sent Events client for Go](https://github.com/donovanhide/eventsource)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## Credits

Created by [Kévin Dunglas](https://dunglas.fr). Sponsored by [Les-Tilleuls.coop](https://les-tilleuls.coop).
