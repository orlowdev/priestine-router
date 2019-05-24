# Grace.js Router

The simplest dependency-free router for Node.js you can think of.

## Installation

```bash
npm install --save @grace-js/router
```

## Description

### Overall

Grace.js Router provides an easy way to declare routes in your application. This library is dependency free and does not rely on `path-to-regexp` as many other similar ones do. To define the routes to match against, you can:

* use strings
* use regular expressions
* create custom matchers that incapsulate required logic for matching, e.g. `path-to-regexp`

```javascript
const myRouter = Router.empty();

myRouter
  .get('/', (req, res) => {})
  .post(/\/api\/v1\/posts\/(\d+)\/?/, (req, res) => {});
```

### Router.withPrefix

The router instance can be created with assigned prefix that will be prepended to each route defined on that router. The prefix can be a string or a regular expression. A static `Router.withPrefix(prefix)` can be used for that puprose.

```javascript
const apiRouter = Router.withPrefix('/api/v1')
  .get('/posts', (req, res) => {});
```

#### Prefix and route path concatenation

* if **prefix** is a `string` and **route path** is a `string`, the final route is a `string`
* if **prefix** is a `string` and **route path** is a `RegExp`, the final route is a `RegExp`
* if **prefix** is a `RegExp` and **route path** is a `string`, the final route is a `RegExp`
* if **prefix** is a `RegExp` and **route path** is a `RegExp`, the final route is a `RegExp`

```javascript
Router.withPrefix('/api').get('/v1/posts', (req, res) => {}); // /api/v1/posts

Router.withPrefix('/api').get(/\/v1\/posts/, (req, res) => {}); // /\/api\/v1\/posts

Router.withPrefix(/\/api/).get('/v1/posts', (req, res) => {}); // /\/api\/v1\/posts

Router.withPrefix(/\/api/).get(/\/v1\/posts/, (req, res) => {}); // /\/api\/v1\/posts
```

### Registering routes

There are helper methods for defining most common HTTP verbs:

* router.get
* router.post
* router.put
* router.patch
* router.delete
* router.head
* router.options
* router.all (registers routes for all the HTTP methods mentioned above)

Each of these helper methods accept two arguments: a **route** (optional, equals *""* if omitted) and a **callback function** (required).

If you need to register another HTTP method (e.g. `LINK`), you can use `router.register`. All the methods mentioned above internally use `router.register`. Unlike those helper methods, `router.register` accepts three arguments:

* a route (optional, equals *""* if omitted)
* an array of HTTP methods to be registered (required)
* a callback function (required)

#### Why route is optional

The main reason for making the route optional is that in many cases there is a prefix already defined on the router itself and there is nothing to be added after the prefix. If you do not feel that it fits the whole system where you sometimes provide it and sometimes you don't, you can simply put an empty string (**""**) as the first argument because this is what it defaults to when the route definition is omitted.

### Router.of and router.concat

Essentially, Grace.js Router is a Monoid with a bunch of helper methods for registering routes. You can create a set of separate declarative routers and concat them until you build one router aware of the whole route map.

Routers can be concatenated any amount of levels deep and prefix of the absorber router will be prepended to each route from absorbed router.

To create an empty router instance, use static `Router.empty`. Empty router prefix is equal to an empty string (**""**).

To absorb another router's route map, use `router1.concat(router2)`. As a result, your router1 will include all the route definitions of router2 as well as its own route definitions.

**NOTE**: In the example above, prefix of the router1 will be prepended to all the routes taken from router2.

```javascript
const mainRouter = Router.empty();

const postRouter = Router.withPrefix('/posts')
  .get((req, res) => {})
  .post((req, res) => {});

const commentsRouter = Router.withPrefix(/\/(\w+)\/comments/)
  .get('', (req, res) => {})
  .get(/(\w+)/, (req, res) => {});

const authRouter = Router.withPrefix('/auth')
  .post('/sign-in', (req, res) => {})
  .post('/sign-up', (req, res) => {});

const apiRouter = Router.withPrefix('/api')
  .concat(
    Router.withPrefix('/v1')
      .concat(authRouter)
      .concat(postRouter.concat(commentsRouter))
  );

// POST /api/v1/auth/sign-in
// POST /api/v1/auth/sign-up
// GET /api/v1/posts
// GET /\/api\/v1\/posts\/(\w+)/
// GET /\/api\/v1\/posts\/(\w+)\/comments/
// POST /\/api\/v1\/posts\/(\w+)\/comments/
```

### Route callback

When a request comes in, it is matched against the route map. When appropriate route was found, its callback is then triggered with Node.js `http.IncomingMessage` and `http.ServerResponse` objects respectively. The Grace.js Router does not amend these objects except for one tiny thing added to the `req` (IncomingMessage) which is the reference to the route matched in the router. To access that value, you will need to import the symbol identifier from Grace.js Router library.

**NOTE**: The matched route is not going to be just string or regular expression - it will assign the wrapper `Matcher` object.

### Wiring up

Grace.js Router is not capable of serving and requires a wrapper function that can will wire it to `http` or `https`. Moreover, the router itself doesn't handle cases when the incoming message cannot be matched against any registered route.

So, the wrapper function should use the route map withing the router **and** handle requests that didn't match any route. Otherwise, you will simply never get response back.

The most simple example for such function is:

```javascript
const Router = require('@grace-js/router');
const http = require('http');
const https = require('https');
const fs = require('fs');
const path = require('path');

const router = Router.withPrefix('/')
    .get(() => {});

// Accept main router as an argument and return a function that will wait for Node.js `IncomingMessage` and `ServerResponse`.
function withRouter(router) {
    // Enclose the router for future reference
    return function matchRoute(req, res) {
        // Match IncomingMessage against the route map
        const matchedRoute = router.routeMap.find(req);
        
        // End response appropriately if the route was not found
        if (!matchedRoute) {
            res.setHeader('Content-Type', 'text/plain');
            res.statusCode = 404;
            return res.end(`Cannot ${req.method} ${req.url}`);
        }
        
        // Execute matched route callback with proper tail call
        return matchedRoute.callback(req, res);
    }
}

http.createServer(withRouter(router)).listen(3000);

const options = {
  key: fs.readFileSync(path.resolve('/path/to/certificate/key.pem')),
  cert: fs.readFileSync(path.resolve('/path/to/certificate/cert.pem')),
};

https.createServer(options, withRouter(router)).listen(3000);
```



### router.sort

By default, Grace.js Router stores route definitions in the order you define them. When incoming message arrives, the router matches the request url and method against registered routes in exact same order they were registered.

There is a built-in mechanism for applying route map sorting to organise matching. This is not something you would normally need except for the case when there is ambiguity in matching RegExp-based routes that go before string-based routes and the latter ones are never reached.

##### Example

```javascript
Router.empty()
    .get(/\/api\/v1\/users\/(\w+)/, (req, res) => {})
    .get('/api/v1/users/sign-in', (req, res) => {});
```

This is probably smelly yet there are edge cases when the ambiguity is required to be in place.

In this case, at the very end of your main router definition you can chain the `router.sort()` method which is executed as follows:

* The complexity of all the routes is calculated:
  * If route path is a string, the amount of slashes is added to the initial complexity (0)
  * If route path is a regular expression, the amount of slashes is subtracted from the initial complexity value (1000000)
* The routes are sorted based on their complexity

This algorithm assures that the simplest route to match against in your application is **"/"** and should be considered the first one to match against. The most complex route in your application would be **/\\/(.\*)/** which should only be referred to as last resort if no other routes match. 

**NOTE**: the initial complexity value for RegExp-based route definitions came up as an arbitrary value high enough to make an assumption that no one would ever register that many routes.
