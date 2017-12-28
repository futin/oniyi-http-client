# oniyi-http-client  
[![NPM version][npm-image]][npm-url] [![Dependency Status][daviddm-image]][daviddm-url]
> Adding a plugin interface to [request](https://www.npmjs.com/package/request) that allows modifications of request parameters and response data

## Installation

```sh
$ npm install --save oniyi-http-client
```

## Usage

**Note:** this module does not support streams, yet

```js
const httpClientFactory = require('oniyi-http-client');

const client = httpClientFactory({
  defaults: {
    headers: {
      'Accept-Language': 'en-US,en;q=0.8',
      Host: 'httpbin.org',
      'Accept-Charset': 'ISO-8859-1,utf-8;q=0.7,*;q=0.3',
      Accept: 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8'
    }
  }
});

client.get('http://httpbin.org/headers', {}, (err, response, body) => {
  if (err) {
    logger.warn('got an error');
    if (err.stack) {
      logger.error(err.stack);
    } else {
      logger.error(err);
    }
    process.exit(0);
  }
  if (response) {
    logger.debug('statusCode: %d', response.statusCode);
    logger.debug('headers: ', response.headers);
    logger.debug('body: ', body);
  }
  process.exit(0);
});
```

## Motivation
"Is there really a need for another http library?" you might ask. There isn't. The actual need is for the ability to asynchronously hook into the process of making a http request or receiving a response.

I came across this requirement when working with various REST APIs, making requests with a number of different credentials (representing users logged into my application). Since the flow within my app provided me with an `user` object that has an async method to retrieve this user's credentials (e.g. an oauth access-token), I wanted to follow the `DRY` (don't repeat yourself) pattern and not manually resolve before invoking e.g. [`request`](https://www.npmjs.com/package/request).

Instead I thought it would be much easier to pass the user along with the request options and have some other module take care of resolving and injecting credentials.

Quickly more use-cases come to mind:
* [credentials](https://www.npmjs.com/package/oniyi-http-plugin-credentials)
* async cookie jars
* [caching](https://www.npmjs.com/package/oniyi-cache)
* [throttling](https://www.npmjs.com/package/oniyi-limiter)

Also, use-cases that require to manipulate some options based on other options (maybe even compiled by another plugin) can be solved by this phased implementation. Some REST APIs change the resource path depending on the type of credentials being used. E.g. when using *BASIC* credentials, a path might be `/api/basic/foo` while when using `oauth` the path changes to `/api/oauth/foo`. This can be accomplished by using e.g. [oniyi-http-plugin-format-url-template](https://www.npmjs.com/package/oniyi-http-plugin-format-url-template) in a late phase (`final`) of the onRequest PhaseLists.

## Phases
This HTTP Client supports running multiple plugins / hooks in different phases before making a request as well as after receiving a response. Both [PhaseLists](https://github.com/strongloop/loopback-phase/blob/master/lib/phase-list.js) are initiated with the phases `initial` and `final` and [zipMerged](https://github.com/strongloop/loopback-phase/blob/master/lib/phase-list.js#L164) with `params.requestPhases` and `params.responsePhases` respectively. That means you can add more phases by providing them in the factory params.

```js
const client = httpClientFactory({
  requestPhases: ['early', 'initial', 'middle', 'final'],
  responsePhases: ['initial', 'middle', 'final', 'end'],
});
```

### onRequest
`onRequest` is one of the (currently) two hooks that executes registered plugins in the defined phases. After all phases have run their handlers successfully, the resulting request options from `ctx.options` are used to initiate a new `request.Request`. The return value from `request.Request` (a readable and writable stream) is what the returned `Promise` from any of the request initiating methods from `client` (`makeRequest`, `get`, `put`, `post`, ...) resolves to.

Handlers in this phaseList are invoked with (`ctx`, `next`), where `ctx` has the following members:
* `hookState`
* `options`

### onResponse
`onResponse` is the second hook and executes registered plugins after receiving the response from `request` but before invoking `callback` from the request execution. That means plugins using this hook / phases can work with and modify `err, response, body` before the app's `callback` function is invoked. Here you can do things like validating response's `statusCode`, parsing response data (e.g. xml to json), caching, reading `set-cookie` headers and persist in async cookie jars... the possibilities are wide.

Handlers in this phaseList are invoked with (`ctx`, `next`), where `ctx` has the following members:
* `hookState`
* `options`
* `response`
* `responseError`
* `responseBody`

## Using plugins

Every plugin can register any number of handlers for any of the phases available `onRequest` as well as `onResponse`.

The following example creates a plugin named `plugin-2` which adds a request-header with name and value `plugin-2`.
Also, it stores some data in shared state that is re-read on response and printed.

```js
const plugin2 = {
  name: 'plugin-2',
  onRequest: [{
    phaseName: 'initial',
    handler: (ctx, next) => {
      const { options, hookState } = ctx;
      // store something in the state shared across all hooks for this request
      _.set(hookState, 'plugin-2.name', 'Bam Bam!');

      setTimeout(() => {
        _.set(options, 'headers.plugin-2', 'plugin-2');
        next();
      }, 500);
    },
  }],
  onResponse: [{
    phaseName: 'final',
    handler: (ctx, next) => {
      const { hookState } = ctx;
      // read value from state again
      const name = _.get(hookState, 'plugin-2.name');

      setTimeout(() => {
        logger.info('Name in this plugin\'s store: %s', name);
        next();
      }, 500);
    },
  }],
};

client
  .use(plugin2)
  .get('http://httpbin.org/headers', (err, response, body) => {
    if (err) {
      logger.warn('got an error');
      if (err.stack) {
        logger.error(err.stack);
      } else {
        logger.error(err);
      }
      process.exit(0);
    }
    if (response) {
      logger.debug('statusCode: %d', response.statusCode);
      logger.debug('headers: ', response.headers);
      logger.debug('body: ', body);
    }
    process.exit(0);
  });

```

## License

MIT © [Benjamin Kroeger]()


[npm-image]: https://badge.fury.io/js/oniyi-http-client.svg
[npm-url]: https://npmjs.org/package/oniyi-http-client
[daviddm-image]: https://david-dm.org/benkroeger/oniyi-http-client.svg?theme=shields.io
[daviddm-url]: https://david-dm.org/benkroeger/oniyi-http-client
