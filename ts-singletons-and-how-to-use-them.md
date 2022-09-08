|Status|Link|
| ----------- | ----------- |
|In Progress||

# Javascript Singletons And How To Use Them

Hi Team,

Today, I'd like to tell about singletons and how we are using them on daily-basis. 
So, singleton pattern allows us to create single instance of particular class. For 
which purposes  can it be helpful? We are using them for:

* data interchange services, which take care about data transferring to/from underlying services
* data processing services, which can be instantiated only once on application start
* cache services
* testing purposes

For example, we decided to continue with previously described Cache Service 
(https://dev.to/frozer/vanilla-js-data-cache-service-1ei2), for some reasons we don't need multiple 
cache services in our app, so how we are going to create a singletone?

Let's start with very basic code to create a singletone:
```
class SingletoneService {
  static instance;
  
  getInstance(args) {
    if (!SingletoneService.instance) {
      SingletoneService.instance = new SingletoneService(...args);
    }
    
    return SingletoneService.instance;
  }
  
  constructor(args) {
    // do something with args
  }
  
  doSomething() {
    // do something
  }
}
```
So, how does it work? In the main program we can create instance of 
singleton by calling this *getInstance* method with arguments (if we need them):
```
function main() {
  const instance = SingletoneService.getInstance();
  
  instance.doSomething();
}
main();
```
By calling the *getInstance* method it checks for static **instance** field 
value of class SingletoneService, and then instantiate a new object based on that 
class using the *new* keyword. If it exists, it simply returns the existing instance.

Now, once the draft implementation is done, try to move this functionality into our
CacheService:
```
class SomeServiceWithDataCache {
  static instance;
  
  getInstance(args) {
    if (!SomeServiceWithDataCache.instance) {
      SomeServiceWithDataCache.instance = new SomeServiceWithDataCache(...args);
    }
    
    return SomeServiceWithDataCache.instance;
  }
  
  constructor() {
    this.cache = {
      isLoading: false,
      expire: 0,
      data: null
    };
    this.cacheSubscriptions = [];
  }
  ...
}
```
Also, you can use singletones to inject dependency to your class:
```
class DataProcessingService {
  // use existing DataCacheService instance by default
  constructor(cacheService = DataCacheService.getInstance()) {
    // do something in constructor)
  }
  ...
}
```
Which allows you to implement code isolation and Open-Closed principle.
