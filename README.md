# figma-await-call

> A convenient `await`-able interface for messaging between the main and UI threads in a Figma plugin.

Figma plugins typically use `postMessage()` to send messages back and forth between the threads, but managing those asynchronous events can be tedious.  Now you can simply `await` the response to get a synchronous-style flow, like:

```typescript
const title = await call("getProperty", "title");
figma.currentPage.selection[0].characters = title;
```


## Installation

```shell
npm install figma-await-call
```


## Usage

In a Figma plugin, the code that has access to the document API and the code that renders the UI are in different contexts, so you'll typically need to use `postMessage()` to send a request for data from one thread to the other.  The other thread then needs to listen for the `"message"` event and respond by calling `postMessage()` to send back the requested data.  The source thread then *also* needs to listen for the `"message"` event to receive the requested data.

All of this asynchronous event handling is an awkward fit for what is conceptually a synchronous function call.  The `figma-await-call` package wraps all of this event handling in a simpler interface.

The package's `call()` function lets you essentially call a named function in the other thread, while awaiting the promised result.  Any parameters you pass to `call()` after the function name will be passed into the receiving function:

```typescript
// main.ts
import { call } from "figma-await-call";

try {
  const title = await call("getProperty", "title");
  figma.currentPage.selection[0].characters = title;
} catch (error) {
  console.error("Error fetching name property:", error);
}
```

If the called "function" throws an exception in the other thread, that error will be caught and then rethrown in the current thread, so you should wrap the `call()` in a `try/catch` when you know that may happen.

You can also use a standard `then()` method to handle the returned promise:

```typescript
call("getProperty", "title")
  .then((title) => figma.currentPage.selection[0].characters = title)
  .catch(console.error);
```

Of course, making a call from one thread won't do anything if there's nothing in the other thread to receive that call.  So every `call()` to a particular name must be paired with a `receive()` in the other thread to provide a function that will respond to that name:

```typescript
// ui.tsx
import { receive } from "figma-await-call";

const storedProperties = { ... };

receive("getProperty", (key) => {
  if (!(key in storedProperties)) {
    throw new Error(`Unsupported property: ${key}`);
  }

  return storedProperties[key];
});
```

Any number of `call()/receive()` pairs can exist within a plugin.

Note that if your code awaits a call to a name that has no receiver registered in the other thread, then execution will hang at that point.


## API


### `call(name, [...data])`

Makes a call between the main and UI threads.

* `name`: The name of the receiver in the other thread that is expected to respond to this call.
* `...data`: Zero or more parameters to send to the receiver.  They must be of types that can be passed through `postMessage()`.

This returns a promise that can be awaited until the other side responds, and which will be resolved with the return value of the receiver function.

If the receiver function throws an exception, that exception will be rethrown from `call()`, so you should use `try/catch` or `.catch()` to handle that case.


### `receive(name, receiverFn)`

Registers a function to receive calls with a particular name.  It will receive whatever parameters are passed to the `call()` function.

* `name`: The name of the receiver.
* `receiverFn`: The function that will receive calls to the `name` parameter.

The return value from `receiverFn` will be sent back to the caller.  If it returns a promise, then no response will be sent to the caller until the promise resolves.

Only a single function can respond to any given name, so subsequent calls to `receive()` will replace the previously registered function.


### `ignore(name)`

Unregisters the receiver for a given function name.

* `name`: The name of the receiver to unregister.

Subsequent calls to the unregistered name from the other thread will never return (unless a new function is registered for it).


## License

[MIT](./LICENSE) © [John Dunning](https://github.com/fwextensions)
