
# dungeon v0.0.1

Do you send requests to the same service(s) repeatedly? Using `dungeon`
will save you from repeating yourself, and a few epics may even drop!

Requests are sent with [`quest`](https://github.com/aleclarson/quest).

### Features
- Minimal API surface
- Don't repeat yourself (DRY)
- `GET` request batching
- Oh, and it's isomorphic! 😎

### Usage
```js
const Dungeon = require('dungeon')

// Each dungeon represents a single service.
const api = new Dungeon('https://your-api.com/v1')

// Fetch some JSON.
const users = await api.json('/users', query, headers)

// Create an HTTP request.
const req = api.post('/users/alec', query, headers)

// Set a request header.
req.set('keep-alive', false)

// Attach a request body.
const res = await req.send({name: 'Alec', age: 23})

// Stop the request before it receives a response.
// This will reject the returned promise with an `AbortError`.
req.abort()

// Simple response API.
const text = await res.text()
const json = await res.json()
res.headers // => {}
res.status  // => 201
res.ok      // => true

// Set default headers.
api.set('headers', {
  Authorization: 'Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==',
})

// Set default query.
api.set('query', {
  api_key: 'h7hdj6s8j4',
})

// Hook into every request.
api.on('request', (req) => {
  req.method // => 'GET'
  req.path // => '/users'
  req.query // => {}
  req.headers // => {}
})

// Hook into every response.
api.on('response', (res) => {
  res.req // => [object Request]
  res.headers // => {}
  res.status // => 200
  res.ok // => true
})
```

### Loaders
Loaders wrap a specific endpoint and batch all GET requests within
a small timeframe.

Requests that time out or never connect are retried using an
exponential backoff algorithm, and a "retry" event is emitted.

Loaders can also detect when a device reconnects to the internet,
and retry timed out requests immediately.

```js
const users = api.loader('/users/*')

// Fetch a user by their ID.
const user = await users.json('satoshi')

// Listen for retries.
users.on('retry', (req) => {
  req.retries // => 1

  // Prevent the retry.
  return false
})

// Attach custom methods.
users.getName = async function(userId) {
  const res = await this.json(userId, {fields: 'name'})
  return res.name
}
```

Loaders inherit any `api` object configuration declared using the
`set` method, like the default query/headers or request timeout.
Each loader can customize its behavior using its own `set` method.

```js
// Limit the number of requests per batch (defaults to 25).
user.set('batchSize', 10)

// To avoid hanging requests, set a timeout (in seconds).
users.set('timeout', 15)
```

#### Batching
Batched requests are sent to the `/batch` endpoint where the body is
a JSON object like `{path, batch}` where `batch` is an array of IDs
or tuples like `[id, query]` when a query is supplied. The loader's
default query is attached to the `{path, batch}` object as the
`query` property, instead of duplicating it for every request.
