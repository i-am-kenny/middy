# Middlewares documentation

## Available middlewares

 - [cache](#cache)
 - [cors](#cors)
 - [doNotWaitForEmptyEventLoop](#donotwaitforemptyeventloop)
 - [httpErrorHandler](#httperrorhandler)
 - [httpHeaderNormalizer](#httpHeaderNormalizer)
 - [jsonBodyParser](#jsonbodyparser)
 - [s3KeyNormalizer](#s3keynormalizer)
 - [ssm](#ssm)
 - [validator](#validator)
 - [urlEncodeBodyParser](#urlencodebodyparser)
 - [warmup](#warmup)


## [cache](/src/middlewares/cache.js)

Offers a simple but flexible caching layer that allows to cache the response associated
to a given event and return it directly (without running the handler) if such event is received again
in a successive execution.

By default, the middleware stores the cache in memory, so the persistence is guaranteed only for
a short amount of time (the duration of the container), but you can use the configuration
layer to provide your own caching implementation.

### Options

 - `calculateCacheId` (function) (optional): a function that accepts the `event` object as a parameter
   and returns a promise that resolves to a string which is the cache id for the
   give request. By default the cache id is calculated as `md5(JSON.stringify(event))`.
 - `getValue` (function) (optional): a function that defines how to retrieve a the value associated to a given
   cache id from the cache storage. it accepts `key` (a string) and returns a promise
   that resolves to the cached response (if any) or to `undefined` (if the given key
   does not exists in the cache)
 - `setValue` (function) (optional): a function that defines how to set a value in the cache. It accepts
   a `key` (string) and a `value` (response object). It must return a promise that
   resolves when the value has been stored.

### Sample usage

```javascript
// assumes the event contains a unique event identifier
const calculateCacheId = (event) => Promise.resolve(event.id)
// use an in memory storage as example
const myStorage = {}
// simulates a delay in retrieving the value from the caching storage
const getValue = (key) => new Promise((resolve, reject) => {
  setTimeout(() => resolve(myStorage[key]), 100)
})
// simulates a delay in writing the value in the caching storage
const setValue = (key, value) => new Promise((resolve, reject) => {
  setTimeout(() => {
    myStorage[key] = value
    return resolve()
  }, 100)
})

const originalHandler = (event, context, cb) => {
  /* ... */
}

const handler = middy(originalHandler)
  .use(cache({
    calculateCacheId,
    getValue,
    setValue
  }))
```


## [cors](/src/middlewares/cors.js)

Sets CORS headers (`Access-Control-Allow-Origin`), necessary for making cross-origin requests, to response object.

Sets headers in `after` and `onError` phases.

### Options

 - `origin` (string) (optional): origin to put in the header (default: "`*`")

### Sample usage

```javascript
const middy = require('middy')
const { cors } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  cb(null, {})
})

handler.use(cors())

// when Lambda runs the handler...
handler({}, {}, (_, response) => {
  expect(response.headers['Access-Control-Allow-Origin']).toEqual('*')
})
```


## [doNotWaitForEmptyEventLoop](/src/middlewares/doNotWaitForEmptyEventLoop.js)

Sets `context.callbackWaitsForEmptyEventLoop` property to `false`.
This will prevent lambda for timing out because of open database connections, etc.

### Options

By default middleware sets the `callbackWaitsForEmptyEventLoop` property to `false` only in the `before` phase,
meaning you can override it in handler to `true` if needed. You can set it in all steps with the options:

- `runOnBefore` (defaults to `true`) - sets property before running your handler
- `runOnAfter`  (defaults  to `false`)
- `runOnError` (defaults to `false`)

### Sample Usage

```javascript
const middy = require('middy')
const { doNotWaitForEmptyEventLoop } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  cb(null, {})
})

handler.use(doNotWaitForEmptyEventLoop({runOnError: true}))

// When Lambda runs the handler it get context with callbackWaitsForEmptyEventLoop property set to false

handler(event, context, (_, response) => {
  expect(context.callbackWaitsForEmptyEventLoop).toEqual(false)
})
```


## [httpErrorHandler](/src/middlewares/jsonBodyParser.js)

Automatically handles uncatched errors that are created with
[`http-errors`](https://npm.im/http-errors) and creates a proper HTTP response
for them (using the message and the status code provided by the error object).

It should be set as the last error handler.


### Sample usage

```javascript
const middy = require('middy')
const { httpErrorHandler } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  throw new createError.UnprocessableEntity()
})

handler
  .use(httpErrorHandler())

// when Lambda runs the handler...
handler({}, {}, (_, response) => {
  expect(response).toEqual({
    statusCode: 422,
    body: 'Unprocessable Entity'
  })
})
```


## [httpEventNormalizer](/src/middlewares/httpEventNormalizer.js)

If you need to access query string or path parameters in an API Gateway event you
can do so by reading the attributes in the `event.queryStringParameters` and
`event.pathParameters`, for example: `event.pathParameters.userId`. Unfortunately
if there are no parameters for one of this parameters holders, the key `queryStringParameters`
or `pathParameters` won't be available in the object, causing an expression like:
`event.pathParameters.userId` to fail with error: `TypeError: Cannot read property 'userId' of undefined`.

A simple solution would be to add an `if` statement to verify if the `pathParameters` (or `queryStringParameters`)
exists before accessing one of its parameters, but this approach is very verbose and error prone.

This middleware normalizes the API Gateway event, making sure that an object for
`queryStringParameters` and `pathParameters` is always available (resulting in empty objects
when no parameter is available), this way you don't have to worry about adding extra `if`
statements before trying to read a property and calling `event.pathParameters.userId` will
result in `undefined` when no path parameter is available, but not in an error.

### Sample usage

```javascript
const middy = require('middy')
const { httpEventNormalizer } = require('middy/httpEventNormalizer')

const handler = middy((event, context, cb) => {
  console.log('Hello user #{event.pathParameters.userId}') // might produce `Hello user #undefined`, but not an error
  cb(null, {})
})

handler.use(httpEventNormalizer())
```


## [httpHeaderNormalizer](/src/middlewares/httpHeaderNormalizer.js)

Normalizes HTTP header names to their canonical format. Very useful if clients are
not using the canonical names of header (e.g. `content-type` as opposed to `Content-Type`).

API Gateway does not perform any normalization, so the headers are propagated to Lambda
exactly as they were sent by the client.

Other middlewares like [`jsonBodyParser`](#jsonBodyParser) or [`urlEncodeBodyParser`](#urlEncodeBodyParser)
will rely on headers to be in the canonical format, so if you want to support non normalized headers in your
app you have to use this middleware before those ones.

This middleware will copy the original headers in `event.rawHeaders`.

### Options

 - `normalizeHeaderKey` (function) (optional): a function that accepts an header name as a parameter and returns its
   canonical representation.

### Sample usage

```javascript
const middy = require('middy')
const { httpHeaderNormalizer, jsonBodyParser, urlEncodeBodyParser } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  cb(null, {})
})

handler
  .use(httpHeaderNormalizer())
  .use(jsonBodyParser())
  .use(urlEncodeBodyParser())
```


## [jsonBodyParser](/src/middlewares/jsonBodyParser.js)

Automatically parses HTTP requests with JSON body and converts the body into an
object. Also handles gracefully broken JSON as UnprocessableEntity (422 errors)
if used in combination of `httpErrorHanler`.

It can also be used in combination of validator as a prior step to normalize the
event body input as an object so that the content can be validated.

```javascript
const middy = require('middy')
const { jsonBodyParser } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  cb(null, {})
})

handler.use(jsonBodyParser())

// invokes the handler
const event = {
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({foo: 'bar'})
}
handler(event, {}, (_, body) => {
  expect(body).toEqual({foo: 'bar'})
})
```


## [s3KeyNormalizer](/src/middlewares/s3KeyNormalizer.js)

Normalizes key names in s3 events.

S3 events like S3 PUT and S3 DELETE will contain in the event a list of the files
that were affected by the change.

In this list the file keys are encoded [in a very peculiar way](http://docs.aws.amazon.com/AmazonS3/latest/dev/notification-content-structure.html) (urlencoded and
space characters replaced by a `+`). It happens very often that you will use the
key directly to perform operation on the file using the AWS S3 sdk, in such case,
it's very easy to forget to decode the key correctly.

This middleware, once attached, makes sure that every S3 event has the file keys
properly normalized.


### Sample usage

```javascript
const middy = require('middy')
const { s3KeyNormalizer } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  // use the event key directly without decoding it
  console.log(event.Records[0].s3.object.key)

  // return all the keys
  callback(null, event.Records.map(record => record.s3.object.key))
})

handler
  .use(s3KeyNormalizer())
```

## [ssm](/src/middlewares/ssm.js)

Fetches parameters from [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html).
Requires Lambda to have IAM permission for `ssm:GetParameters` action.
Middleware makes 1 API request to fetch all the parameters at once for efficiency.

By default parameters are assigned to `process.env` node.js object.
They can be assigned to function handler's `context` object by setting `setToContext` flag.

It assumes AWS Lambda environment which has `aws-sdk` version `2.176.0` included by default.
If your project which uses this middleware doesn't use `aws-sdk` yet,
you may need to install it as a `devDependency` in order to run tests.  

### Options

- `awsSdkOptions` (object) (optional): Options to pass to AWS.SSM class constructor.
  Defaults to `{ maxRetries: 6, retryDelayOptions: {base: 200} }`
- `params` (object): Map of parameters to fetch from SSM, where key is name of
  parameter middleware will set, and value is param name in SSM.
  Example: `{params: {DB_URL: '/dev/service/db_url''}}`
- `setToContext` (boolean) (optional): This will assign parameters to `context` object
  of function handler.

### Sample Usage

Simplest usage, exports parameters as environment variables.

```javascript
const middy = require('middy')
const { ssm } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  cb(null, {})
})

handler.use(ssm({
  params: {
    SOME_ACCESS_TOKEN: '/dev/service_name/access_token'
  }
}))

// Before running function handler, middleware will fetch SSM params

handler(event, context, (_, response) => {
  expect(process.env.SOME_ACCESS_TOKEN).toEqual('some-access-token')
})
```

Export parameters to `context` object, override AWS region.

```javascript
const middy = require('middy')
const { ssm } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  cb(null, {})
})

handler.use(ssm({
  awsSdkOptions: {region: 'us-west-1'},
  params: {
    SOME_ACCESS_TOKEN: '/dev/service_name/access_token'
  },
  setToContext: true
}))

handler(event, context, (_, response) => {
  expect(context.SOME_ACCESS_TOKEN).toEqual('some-access-token')
})
```

## [validator](/src/middlewares/validator.js)

Automatically validates incoming events and outgoing responses against custom
schemas defined with the [JSON schema syntax](http://json-schema.org/).

If an incoming event failes validation a `BadRequest` error is raised.
If an outgoing response failes validation a `InternalServerError` error is
raised.

This middleware can be used in combination with
[`httpErrorHandler`](#httpErrorHandler) to automatically return the right
response to the user.

### Options

 - `inputSchema` (object) (optional): the JSON schema object that will be used
   to validate the input (`handler.event`) of the Lambda handler.
 - `outputSchema` (object) (optional): the JSON schema object that will be used
   to validate the output (`handler.response`) of the Lambda handler.

### Sample Usage

Example for input validation:

```javascript
const middy = require('middy')
const { validator } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  cb(null, {})
})

const schema = {
  required: ['body', 'foo'],
  properties: {
    // this will pass validation
    body: {
      type: 'string'
    },
    // this won't as it won't be in the event
    foo: {
      type: 'string'
    }
  }
}

handler.use(validator({
  inputSchema: schema
}))

// invokes the handler, note that property foo is missing
const event = {
  body: JSON.stringify({something: 'somethingelse'})
}
handler(event, {}, (err, res) => {
  expect(err.message).toEqual('Event object failed validation')
})
```

Example for output validation:

```javascript
const middy = require('middy')
const { validator } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  cb(null, {})
})

const schema = {
  required: ['body', 'statusCode'],
  properties: {
    body: {
      type: 'object'
    },
    statusCode: {
      type: 'number'
    }
  }
}

handler.use(validator({outputSchema: schema}))

handler({}, {}, (err, response) => {
  expect(err).not.toBe(null)
  expect(err.message).toEqual('Response object failed validation')
  expect(response).not.toBe(null) // it doesn't destroy the response so it can be used by other middlewares
})
```

## [urlEncodeBodyParser](/src/middlewares/urlEncodeBodyParser.js)

Automatically parses HTTP requests with URL encoded body (typically the result
of a form submit).

### Options

 - `extended` (boolean) (optional): if `true` will use [`qs`](https://npm.im/qs) to parse
  the body of the request. By default is set to `false`

### Sample Usage

```javascript
const middy = require('middy')
const { urlEncodeBodyParser } = require('middy/middlewares')

const handler = middy((event, context, cb) => {
  cb(null, event.body) // propagates the body as response
})

handler.use(urlEncodeBodyParser({extended: false}))

// When Lambda runs the handler with a sample event...
const event = {
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: 'frappucino=muffin&goat%5B%5D=scone&pond=moose'
}

handler(event, {}, (_, body) => {
  expect(body).toEqual({
    frappucino: 'muffin',
    'goat[]': 'scone',
    pond: 'moose'
  })
})
```


## [warmup](/src/middlewares/warmup.js)

Warmup middleware that helps to reduce the [cold-start issue](https://serverless.com/blog/keep-your-lambdas-warm/). Compatible by default with the [`serverless-plugin-warmup`](https://www.npmjs.com/package/serverless-plugin-warmup), but it can be configured to suit your implementation.

The idea of this middleware is that you have to keep your important lambdas  (the ones that needs to be always very responsive) running often (e.g. every 5 minutes) on a special schedule that will make the lambda running, by running the handler, but skipping the main business logic to avoid side effects.

If you use [`serverless-plugin-warmup`](https://www.npmjs.com/package/serverless-plugin-warmup) the scheduling part is done by the plugin and you just have to attach the middleware to your "middified" handler. If you don't want to use the plugin you have to create the schedule yourself and define the `isWarmingUp` function to define wether the current event is a warmup event or an actual business logic execution.


### Options

 - `isWarmingUp`: a function that accepts the `event` object as a parameter
   and returns `true` if the current event is a warmup event and `false` if it's a regular execution. The default function will check if the `event` object has a `source` property set to `serverless-plugin-warmup`.
 - `onWarmup`: a function that gets executed before the handler exits in case of warmup. By default the function just prints: `Exiting early via warmup Middleware`.

### Sample usage

```javascript
const isWarmingUp = (event) => event.isWarmingUp === true
const onWarmup = (event) => console.log('I am just warming up', event)

const originalHandler = (event, context, cb) => {
  /* ... */
}

const handler = middy(originalHandler)
  .use(warmup({
    isWarmingUp,
    onWarmup
  }))
```
