# `Promise.withResolvers`

Note: this proposal is now [at stage 4](https://github.com/tc39/proposals/commit/8fef293ac43c2ae15b4209ce1fec6347c9a0a583). See the spec PR here: https://github.com/tc39/ecma262/pull/3179

## Status

Stage: 4

Champions:

- Peter Klecha ([@peetklecha](https://github.com/peetklecha))

Authors:

- Peter Klecha ([@peetklecha](https://github.com/peetklecha))

[Stage 3 slides](https://docs.google.com/presentation/d/1KFShqHVFhVBaqZ3anheUGOwtVDrPWCVeFvmaUpwk3AQ)
[Stage 2 slides](https://docs.google.com/presentation/d/1CEh2xgW-KB0Tpz2GQtcJ8nDbWq99d3y8NCwYJw-laSI)
[Stage 1 slides](https://docs.google.com/presentation/d/18CqQc6GfZJBWmT7li2nqfvrSFhpNwtQWPfSXhAwo-Bo)

## Synopsis

When hand-rolling a Promise, the user must pass an executor callback which takes two arguments: a resolve function, which triggers resolution of the promise, and a reject function, which triggers rejection. This works well if the callback can embed a call to an asynchronous function which will eventually trigger the resolution or rejection, e.g., the registration of an event listener.

```js
const promise = new Promise((resolve, reject) => {
  asyncRequest(config, response => {
    const buffer = [];
    response.on('data', data => buffer.push(data));
    response.on('end', () => resolve(buffer));
    response.on('error', reason => reject(reason));
  });
});
```

Often however developers would like to configure the promise's resolution and rejection behavior after instantiating it. Today this requires a cumbersome workaround to extract the resolve and reject functions from the callback scope:

```js
let resolve, reject;
const promise = new Promise((res, rej) => {
  resolve = res;
  reject = rej;
});
asyncRequest(config, response => {
  const buffer = [];
  response.on('callback-request', id => {
    promise.then(data => callback(id, data));
  });
  response.on('data', data => buffer.push(data));
  response.on('end', () => resolve(buffer));
  response.on('error', reason => reject(reason));
});
```

Developers may also have requirements that necessitate passing resolve/reject to more than one caller, so they MUST implement it this way:

```js
let resolve = () => { };
let reject = () => { };

function request(type, message) {
  if (socket) {
    const promise = new Promise((res, rej) => {
      resolve = res;
      reject = rej;
    });
    socket.emit(type, message);
    return promise;
  }

  return Promise.reject(new Error('Socket unavailable'));
}

socket.on('response', response => {
  if (response.status === 200) {
    resolve(response);
  }
  else {
    reject(new Error(response));
  }
});

socket.on('error', err => {
  reject(err);
});
```

This is boilerplate code that is very frequently re-written by developers. This proposal simply seeks to add a static method, tentatively called `withResolvers`, to the `Promise` constructor which returns a promise along with its resolution and rejection functions conveniently exposed.

```js
const { promise, resolve, reject } = Promise.withResolvers();
```

This method or something like it may be known to some committee members under the name `defer` or `deferred`, names also sometimes applied to utility functions in the ecosystem. This proposal adopts a more descriptive name for the benefit of users who may not be familiar with those historical functions.

## Existing implementations

Libraries and applications continually re-invent this wheel. Below are just a handful of examples.

|Library|Example|
|------------|----------|
|React|[inline example](https://github.com/facebook/react/blob/d9e0485c84b45055ba86629dc20870faca9b5973/packages/react-dom/src/__tests__/ReactDOMFizzStaticBrowser-test.js#L95)
|Vue | [inline example](https://github.com/vuejs/core/blob/9c304bfe7942a20264235865b4bb5f6e53fdee0d/packages/runtime-core/src/compat/componentAsync.ts#L32)
|Axios|[inline example](https://github.com/axios/axios/blob/bdf493cf8b84eb3e3440e72d5725ba0f138e0451/lib/cancel/CancelToken.js#L20)
|TypeScript|[utility](https://github.com/microsoft/TypeScript/blob/1d96eb489e559f4f61522edb3c8b5987bbe948af/src/harness/util.ts#L121)
|Vite|[inline example](https://github.com/vitejs/vite/blob/134ce6817984bad0f5fb043481502531fee9b1db/playground/test-utils.ts#L225)
|Deno stdlib | [utility](https://deno.land/std@0.178.0/async/deferred.ts?source)

