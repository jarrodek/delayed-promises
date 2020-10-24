# Delayed promises

## Synopsis

The delayed modules adds a new functionality to the Promises spec allowing to configure to run the Promise after a specified time.

Developers would be able to create a promise with additional configuration argument that specifies the timeout after which the promise runs.

```javascript
function macroTaskFunction() {
  return new Promise((resolve) => {
    // the task
    resolve();
  }, {
    timeout: 0,
  });
}
```

When calling the `macroTaskFunction()` a task is scheduled rather than a micro task. The `timeout` can be any positive integer that is a number of milliseconds after which the task is scheduled to run (just like `setTimeout` function).

On the `async` / `await` side, developers can execute the task or a part of it by calling the `await` with a numeric argument which is the timeout that the async function awaits.

```javascript
async function delayedFunction() {
  // some work
  await 0;
  // some other work
}
```

## Motivation

In the projects I am working usually the application has to process a huge amount of data. This implies using several technics to split the work into chunks, recursive calls, and timeouts. This, however, needs to be done per case and usually ends up being something that is copied from one code base to another.

The following function allow to schedule a task while using `await` operator in another function.

```javascript
function waitUntil(timeout) {
  return new Promise((resolve) => {
    setTimeout(resolve, timeout);
  });
}
```

Now, consider having a expensive task of reading the data from a data store, transforming the data, and putting the data back to the store. If the data is complex and a lot of it this would require using web workers (which is not always possible) or split the task into several chunks. An example of such a function:

```javascript
class TransformData {
  process(ids) {
    this.queue = Array.from(ids);
    return new Promise((resolve) => {
      this.resolve = resolve;
      this.processQueue();
    });
  }

  async processQueue() {
    const id = this.queue.shift();
    if (!id) {
      this.resolve();
      return;
    }
    const data = await getDatastoreEntry(id);
    if (!data) {
      setTimeout(() => this.processQueue());
      return;
    }
    // process data
    setTimeout(() => this.processQueue());
  }
}
```

This class allows to process the data without risking halting the main process for too long. Without `setTimeout()` the main process can stop processing the tasks or even cause `RangeError: Maximum call stack size exceeded` error.
I believe that this could be optimized for the developer experience by allowing to `await` for a specific amount of time rather than a promise:

```javascript
class TransformData {
  async process(ids) {
    this.queue = Array.from(ids);
    await this.processQueue();
  }

  async processQueue() {
    const id = this.queue.shift();
    if (!id) {
      return;
    }
    const data = await getDatastoreEntry(id);
    if (!data) {
      await 0;
      return this.processQueue();
    }
    // process data
    await 0;
    return this.processQueue();
  }
}
```

## Proposed semantics

The `Promise` interface gets the second argument which is a map of configuration options: `PromiseInit`. The default behavior is to schedule the promise as a micro task as it is today.
The `timeout` property of the `PromiseInit` object is a positive integer that tells the engine when to run the promise. When this configuration options is defined then the Promise behaves like setting up a timeout and then running the promise.

The right-hand-side of the `await` operator now can also be a positive number which runs the same algorithm as `setTimeout` with the value passed value. After the timeout the rest of the logic is executed.