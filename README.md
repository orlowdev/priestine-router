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
* router.all (registers routes for all the methods mentioned above)

Each of these helper methods accept two arguments: a **route** (optional, equals *""* if omited) and a **callback function** (required).

If you need to register another HTTP method (e.g. `LINK`), you can use `router.register`. All the methods mentioned above internally use `router.register`. Unlike those helper methods, `router.register` requires three arguments:

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
