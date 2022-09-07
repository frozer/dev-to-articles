* Preface
Hi Team,

Some time ago I faced an interesting task to solve, and today I'd like to share what I found. Let's imagine that we have a service which needs to load some data (sounds familiar, huh?), 

Also we've got several "customers" which utilize the data retrieval method from this service, and for first look all works fine, but when I opened the Developers Console, I was wondering how much duplicate data requests we got there! 

Here is our initial implementation of that service (I rewrote it to be more general):
```
// let's assume we have some network slowness there...
const delayedDataFetch = () => new Promise((resolve) =>
  setTimeout(() => resolve([1, 2, 3]), 2000)
);

class SomeService {
  async getData() {
    return await delayedDataFetch();
  }
}

// ...
// emulation of parallel requests for the same data
function main() {
  const service = new SomeService();
  await Promise.all([
    service.getData(1),
    service.getData(2)
  ]).then(
    ([res1, res2]) => {
      // receives correct data
      console.log(`1. ${JSON.stringify(res1)}`);
      // receives correct data once again, 
      // from the second call to delayedDataFetch
      // and I want to get rid of that second data call to server
      console.log(`2. ${JSON.stringify(res2)}`);
    }
  );
}
```
Well, my first thought was to implement a some little cache for such data:
```
const CACHE_EXPIRATION_TIME_MS = 5000;

class SomeService {
  constructor() {
    this.cache = {
      isLoading: false,
      data: null,
      // current timestamp plus expiration delay
      expire: 0
    };
  }

  // added sequenceId for more clarity
  async getData(sequenceId) {
    console.log(`Received ${sequenceId} request`);
    
    // if cache is expired and isLoading is false 
    // - initiate data update from server
    if (this.cache.expire < new Date().getTime() && !this.cache.isLoading) {
      this.cache.isLoading = true;

      this.cache = {
        isLoading: false,
        data: await delayedDataFetch(),
        expire: new Date().getTime() + CACHE_EXPIRATION_TIME_MS
      };
    }

    console.log(`Response ${sequenceId} with data`);
    return this.cache.data;
  }
}
```
It works! At least, as I can see from Network Tab the number of networks requests reduced to one. Looks like I can submit this code to repo. 

But... hold on, I just started to receive errors from other parts of applications, and the main issue here, that they are different from time to time. Quick searching gives me a new problem - the new code returns null data from time to time. 

So, this simple approach didn't solve the problem completely. Let's look closely to this code of `getData` method:
```
  // a first request switch isLoading to true, and...
  if (this.cache.expired < new Date().getTime() && !this.cache.isLoading) {
    this.cache.isLoading = true;
    // ... do something
  }

  // the second one simply receives empty data... 
  console.log(`Response ${sequenceId} with data`);
  return this.cache.data;
}
```
Gotcha! I found! But how can we say our "customers" that they need to wait? Of course, I can switch to RxJS f.e., but I don't want to bubble up the size or our bundle, and I was eager to reach the goal using more vanilla approach.

I started about to return an another Promise to the second response:
```
  // a first request switch isLoading to true, and...
  if (this.cache.expire < new Date().getTime() && !this.cache.isLoading) {
    this.cache.isLoading = true;
    // ... do something
  }
  
  if (this.cache.isLoading) {
    // it should return something to client in such cases, 
    // which "something" should be resolved back to refreshed data
    console.log(`Response ${sequenceId} with unresolved promise`);
    return new Promise(resolve => ???);
  }

  // the second one simply receives empty data... 
  console.log(`Response ${sequenceId} with data`);
  return this.cache.data;
}
```
I was thinking about how I can resolve it, and a flash brighten my brain :-) - I need a simple PUB/SUB!
I decided to add an additional subscribers pool into `SomeService` class and renamed it to `SomeServiceWithDataCache`:

```
class SomeServiceWithDataCache {
  constructor() {
    this.cache = {
      isLoading: false,
      expire: 0,
      data: null
    };
    this.cacheSubscriptions = [];
  }
```
and change the code of `getData` method accordingly:
```
  async getData(sequenceId) {
    console.log(`Received ${sequenceId} request`);
    if (this.cache.expire < new Date().getTime() && !this.cache.isLoading) {
      this.cache.isLoading = true;

      this.cache = {
        isLoading: false,
        data: await delayedDataFetch(),
        expire: new Date().getTime() + CACHE_EXPIRATION_TIME_MS
      };

      // once we receive data - iterate over subscribers pool, 
      // and resolve each with received data, and finally drop
      // subscriptions
      await Promise.all(
        this.cacheSubscriptions.map((res) => res(this.cache.data))
      ).then(() => (this.cacheSubscriptions = []));
    }

    if (this.cache.isLoading) {
      console.log(`Response ${sequenceId} with unresolved promise`);
      // push the promise resolve into subscribers pool
      return new Promise((resolve) => this.cacheSubscriptions.push(resolve));
    }

    console.log(`Response ${sequenceId} with data`);
    return this.cache.data;
  }
}
```
Well, how it works - it receives the first data request from client, founds that internal data cache is expired, and initiates data loading. For the second request it returns the unresolved Promise, and put the resolver function into the subscribers pool. Once it receives data from server it iterates through the subscribers pool and call resolver with an actual data. That's it.
