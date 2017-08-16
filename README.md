# Promises

## Promises/A+
+ [CommonJS](https://en.wikipedia.org/wiki/CommonJS) => [Promises/A](http://wiki.commonjs.org/wiki/Promises/A) => [Promises/A+](https://promisesaplus.com/)
+ The best way to learn about promises is to trying to implement them.
+ Look at the compliance tests.
+ *"A promise represents the eventual result of an asynchronous operation."*

## The "then" method
+ Single interface to interact with a promise.
+ Interoperability - Different promise implementation can interact with each other.
+ Thenables
+ Registers two handlers. One for the successful async operation one for failed async operation.
+ Both are optional arguments.
+ If they are not functions they are going to be ignored.
+ They are executed as functions. No `this` in the handlers. (???)
```javascript
const promise = new Promise(function resolver(resolve) {
  resolve("thing");
});

const oj = {
  handler() {
    assert.notEqual(this, oj);
  }
};

promise
  .then(oj.handler, function onRejected() {
    assert.ok(false);
  });
```
+ Both are called asynchronously. As a micro task...
+ The success handler is called with the return value of the async op.
+ The failure handler is called with the reason of why the async op failed.
+ `catch` is syntactic sugar for `then(null, function failureHandler() {})`
+ `then` can be called multiple times on the same promise. In which case the handlers will be executed in the order of the registration.
```javascript
const promise = new Promise(function resolver(resolve, reject) {
  
  asyncOperation({"foo": "bar"}, (err, result) => {
    
    if (err) {
      return reject(err);
    }
    
    resolve(result);
  });
});
promise.then(function successRunsFirst() {}, function failureRunsFirst() {});
promise.then(function successRunsSecond() {}, function failureRunsSecond() {});
```
+ Returns a promise. Chaining...
```javascript
promise2 = promise1.then(onFulfilled, onRejected);
// 1. If either onFullfilled or onRejected returns a value promise2 will resolve with that value.
// 2. If either onFullfilled or onRejected throws an error promise2 will reject with that error.
// 3. If onFullfilled is not a function promise2 will resolve with the value promise1 resolved with.
// 4. If onRejected is not a function promise2 will reject with the error promise1 rejected with.
```

## The Promise Resolution Procedure
```javascript
[[Resolve]](promise, x)
```
+ If x is a promise
```javascript
const assert = require("assert");

const promise1 = new Promise(function resolver(resolve) {
  resolve("thing");
});

const promise2 = new Promise(function resolver(resolve) {
  resolve(promise1);
});

promise2
  .then(function onFullfilled(value) {
    assert.equal(value, "thing");
  }, function onRejected() {
    assert.ok(false);
  });
```
```javascript
const assert = require("assert");

const promise1 = new Promise(function resolver(resolve, reject) {
  reject(new Error("glitch"))
});

const promise2 = new Promise(function resolver(resolve) {
  resolve(promise1);
});

promise2
  .then(function onFullfilled() {
    assert.ok(false);
  }, function onRejected(err) {
    assert.deepEqual(err, new Error("glitch"));
  });
```
+ If x is a thenable
```javascript
const assert = require("assert");

const thenable = {
  then(promiseResolve, promiseReject) {
    promiseResolve("thing");
  }
};

const promise = new Promise(function resolver(resolve) {
  resolve(thenable);
});

promise
  .then(function onFullfilled(value) {
    assert.equal(value, "thing")
  }, function onRejected() {
    assert.ok(false);
  });
```
```javascript
const assert = require("assert");

const thenable = {
  then(promiseResolve, promiseReject) {
    promiseReject(new Error("glitch"));
  }
};

const promise = new Promise(function resolver(resolve) {
  resolve(thenable);
});

promise
  .then(function onFullfilled() {
    assert.ok(false);
  }, function onRejected(err) {
    assert.deepEqual(err, new Error("glitch"));
  });
```
```javascript
const assert = require("assert");

const thenable = {
  then(promiseResolve, promiseReject) {
    promiseResolve("thing");
    promiseResolve("nothing");
  }
};

const promise = new Promise(function resolver(resolve) {
  resolve(thenable);
});

promise
  .then(function onFullfilled(value) {
    assert.equal(value, "thing")
  }, function onRejected() {
    assert.ok(false);
  });
```
```javascript
const assert = require("assert");

const thenable = {
  then(promiseResolve, promiseReject) {
    promiseReject(new Error("glitch"));
    promiseReject(new Error("ooops"));
  }
};

const promise = new Promise(function resolver(resolve) {
  resolve(thenable);
});

promise
  .then(function onFullfilled() {
    assert.ok(false);
  }, function onRejected(err) {
    assert.deepEqual(err, new Error("glitch"));
  });
```
+ Non-promise, non-thenable

When `[[Resolve]](promise, x)` just resolve promise with x
##Patterns
```javascript
// Promisify
const promise = new Promise(function resolver(resolve, reject) {
  
  asyncOperation({"foo": "bar"}, (err, result) => {
    
    if (err) {
      return reject(err);
    }
    
    resolve(result);
  });
});

```
```javascript
const obj = {
  method() {
    
    return asyncOperation({"foo": "bar"})
      .then(result => {
        
        result.decorate = "withSomething";
        
        return result;
      });
  }
};
```
```javascript
const obj = {
  method() {
    
    asyncOperation({"foo": "bar"})
      .then(result => {
        
        result.decorate = "withSomething";
       
        return result;
      }, err => {
        
        logger.error(err);
      });
  }
};
```
```javascript
const obj = {
  method() {
    
    return asyncOperation({"foo": "bar"})
      .then(result => {
        
        result.decorate = "withSomething";
        
        return result;
      }, err => {
        
        const serviceError = new ServiceError("An error occurred accessing the database");
        
        serviceError.innerErr = err;
        throw serviceError;
      });
  }
};
```
```javascript
const obj = {
  method() {
    
    return firstAsyncOp()
      .then(firstResult => secondAsyncOp(firstResult))
      .then(secondResult => thirdAsyncOp(secondResult));
  }
};
```
```javascript
const obj = {
  method() {
    
    firstAsyncOp()
      .then(firstResult => secondAsyncOp(firstResult))
      .then(secondResult => {
        
        secondResult.foo.bar = "magic";
        thirdAsyncOp(secondResult);
      })
      .catch(err => {
        
        logger.error("An error occurred doing magic: ", err);
      });
  }
};
```
```javascript
const obj = {
  method() {
    
    firstAsyncOp()
      .then(firstResult => secondAsyncOp(firstResult), err => handleErrForFirst(err))
      .then(secondResult => thirdAsyncOp(secondResult), err => handleErrForSecond(err));
  }
};
```
```javascript
const obj = {
  method(arr) {
    
    return Promise.all(arr.map(elemnt => {
      
      asyncOperation(elemnt)
        .then(result => {
          
          result.foo = "bar";
          
          return result;
        });
    }));
  }
};
```
## Anti Patterns
```javascript
const obj = {
  method() {
    
    return new Promise(function resolver(resolve, reject) {
      
      asyncOperation()
        .then(result => {
          resolve(result);
        }, err => {
          reject(err);
        })
    });
  }
}
```
```javascript
const obj = {
  method() {
    
      return asyncOperation1()
        .then(result1 => {
          
          return asyncOperation2(result1)
            .then(result2 => {
              
              return asyncOperation3(result2)
                .then(result3 => {
                  
                  result3.foo = "bar";
                  
                  return result3;
                })
            })
        });
  }
}
```
```javascript
const obj = {
  method() {
    
    return new Promise(function resolver() {
    
      return asyncOperation1()
        .then(result1 => {
          
          return asyncOperation2(result1)
            .then(result2 => {
              
              return asyncOperation3(result2)
                .then(result3 => {
                  
                  result3.foo = "bar";
                  resolve(result3);
                })
                .catch(reject);
            })
            .catch(reject);
        })
        .catch(reject);
      });
  }
}
```
```javascript
const obj = {
  method() {
    
    return firstAsyncOp()
      .then(firstResult => secondAsyncOp(firstResult))
      .then(secondResult => {
        
        const mutatedResult = secondResult;
        
        mutatedResult.foo = "bar";
        
        return mutatedResult;
      })
      .then(mutatedResult => thirdAsyncOp(mutatedResult));
  }
}

const obj = {
  method() {
    
    return firstAsyncOp()
      .then(firstResult => secondAsyncOp(firstResult))
      .then(secondResult => {
        
        const mutatedResult = secondResult;
        
        mutatedResult.foo = "bar";
        
        return thirdAsyncOp(mutatedResult)
      });
  }
}
```
## Deferreds
Non-standard... Questionable. Clearer code though.

API:
```
const abstractDeferred = {
  promise: <...>
  resolve() {
    <...>
  },
  resolve() {
    <...>
  }
};
```
```javascript
const obj = {
  method() {
    
    const deferred = defer();
    
    asyncOperation({"foo": "bar"}, (err, result) => {
      
      if (err) {
        return deferred.reject(err);
      }
      
      deferred.resolve(result);
    });

    return deferred.promise;
  }
};
```
vs
```javascript
const obj = {
  method() {
    
    return new Promise(function resolver(resolve, reject) {
      
      asyncOperation({"foo": "bar"}, (err, result) => {
        
        if (err) {
          return reject(err);
        }
        
        resolve(result);
      });
    });
  }
};
```
## Testing Promise Based Code
Don't do real asynchronous operations for unit testing.

```javascript
describe("Promise based method", () => {
  
  describe("when it's called with boolean true", () => {
    
    it("should return with a promise which successfully resolves to string True", () => {
      
      return instance.asyncMethod(true)
        .then(result => expect(result).to.equal("True"));
      
      // return instance.asyncMethod(true)
      //   .then(result => expect(result).to.equal("True"), () => {
      //     throw new Error("The promise should not get rejected");
      //   });
    })
  });
  
  describe("when it's called with non boolean", () => {
      
    it("should return with a promise which rejects with an error", () => {
        
      return instance.asyncMethod("non-boolean")
        .then(() => {
          throw new Error("The promise should not get resolved");
        }, err => expect(err.message).to.equal("Oooops"));
    })
  });
})
```
### Anti-pattern in Testing
```javascript
describe("Promise based method", () => {
  
  describe("when it's called with boolean true", () => {
    
    it("should set the value of instance.someData to True asynchronously", () => {
      
      instance.asyncMethod(true); // modifies instance.someData asynchronously
      
      process.nextTick(() => expect(instance.someData).to.equal("True"));
      
      // setTimeout(() => expect(instance.someData).to.equal("True"), 0);
    })
  });
});
```
