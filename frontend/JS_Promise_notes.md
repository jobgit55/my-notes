# Preface
> JS is a single-threaded language that executes in sequential order. But when the browser loads some network requests, you need to wait for the content to be accessed before you can execute the following code. This scenario is a kind of blocking for the program, nothing can be done during this period of time. Therefore, asynchrony comes up to solve this problem. asynchronous makes the program continue execution during the waiting time, and notify us when program has completed execution

# Promise
Promise is a solution to asynchronous programming. It is an object that can retrieve information about asynchronous operations.

## Promise features:
- The state of the object is not affected by the outside. Promise has three states: pending, fulfilled, and rejected. Only the result of the asynchronous operation can determine the current state.
- Once the state changes, it will not change again
- Promise cannot be cancelled. Once created, it will be executed immediately and cannot be cancelled halfway
- If the callback function is not set, the error thrown inside the Promise will not be reflected to the outside
- When it is pending, it is impossible to know its current stage

## Promise usage:
The Promise object is a constructor, used to generate Promise instances
- Create a Promise instance, it will be executed immediately after creation
```
// resolve and reject are functions，provided by JS engine
const promise = new Promise((resolve, reject) => {...})
```

- Use then() method to respectively specify resolved state and rejected state callbacks. It will be executed after all synchronization tasks of the current script finished.
```
promise.then(function(value) {...success}, function(error) {...failure})
```

The reject parameter is generally an instance of the Error object. In addition to the normal value, the parameter of the resolve function may also be another instance, for example promise2. At this time, the state of promise2 determines the state of promise
```
const p1 = new Promise((resolve, reject) => {
    setTimeout(() => reject(new Error('fail')), 3000)
})
// p2 is another instance，p1 determines the state of p2
const p2 = new Promise((resolve, reject) => {
    setTimeout(() => resolve(p1), 1000)
})
p2.then(result => console.log(result), error => console.log(error))
```

- Calling resolve and reject will not terminate the execution of the Promise parameter function, and will continue to execute the following statement, if the return statement is called, it will not be executed
- then method returns a Promise instance (not the original one), so we can use method chaining on it. After the first callback function is completed, the return result will be used as a parameter and passed to the second callback function.
- catch is used to specify the callback function when an error occurs.
	1. error thrown after the Promise state change will not be caught.
	2. The Promise object has a "bubbling" nature and will be passed backwards until it is captured
	3. catch returns a Promise object, we can use then method on it.
- The finally() method is used to specify the operation that will be executed regardless of the final state of the Promise object. finally does not accept any parameters, which means there is no way to know whether the previous Promise status is fulfilled or rejected

## ES6 Promise usage

### Promise.all([p1, p2, p3])
Wrapping multiple Promise instances into a new Promise instance
 - The state of p will become fulfilled when the states of p1, p2, and p3 become fulfilled. At this time, the return values ​​of p1, p2, and p3 form an array passing to the callback function of p
 - If one of p1, p2, and p3 is rejected, the status of p will become rejected, and the return value of the first rejected instance will be passed to the callback function of p
 - If the argument is not a Promise instance, Promise.resolve() will be called
 - If one of p1, p2, and p3 has its own catch method, and once it is rejected, it will trigger Promise.all.then instead of Promise.all.catch

### Promise.race([p1, p2, p3])
Wrapping multiple Promise instances into a new Promise instance
  - As long as one of p1, p2, p3 state changes, the state of p will be changed accordingly.  
  - If the argument is not a Promise instance, Promise.resolve() will be called

### Promise.allSettled([p1, p2, p3])
Wrapping multiple Promise instances into a new Promise instance
  - Wrapped instace will end after all parameter instances returned no mater their states are fulfilled or rejected

Format of the received return value
```
const resolved = Promise.resolve(42);
const rejected = Promise.reject(-1);

const allSettledPromise = Promise.allSettled([resolved, rejected]);

allSettledPromise.then(function (results) {
  console.log(results);
});
// [
//    { status: 'fulfilled', value: 42 },
//    { status: 'rejected', reason: -1 }
// ]
```

### Promise.any([p1, p2, p3])
Wrapping multiple Promise instances into a new Promise instance
 - As long as one of parameters becomes fulfilled, it will become fulfilled. If all parameters become rejected, the wrapped instance will become rejected as well.
 - It is very similar to Promise.race(), but it does not end due to one of the parameters becomes rejected.
 - The error it throws is not a general error, but an instance of AggregateError, which is equivalent to an array, and each member corresponds to an error thrown by a rejected operation.


### Promise.resolve()
Converting an existing object into a Promise object, and the state is resolved.
```
Promise.resolve('foo')
// equals to
new Promise((resolve) => resolve('foo'))
```
- If the parameter is a Promise, it will be returned without changes
- if the parameter is a thenable object, It will be converted into a Promise object, and the then method of the thenable object will be executed immediately
- If the parameter is an object that does not have a then method, or is not an object, a new Promise object will be returned with resolved state.
- If no parameters passed, directly return a Promise object with resolved state, and execute it at the end of this "event loop"
```
setTimeout(() => {console.log('one')}, 0);
Promise.resolve().then(() => {
    console.log('two')
    setTimeout(() => console.log('three'),0)
})
console.log('four')
// Execution order：four two one three
```

### Promise.reject()
Returing a Promise object with rejected state. The parameter of Promise.reject() will be used as the reason for reject and becomes the parameter of the subsequent method.


### Promise.try()
When the function in then method does not distinguish between synchronous and asynchronous, it will be executed at the end of this "event loop".
```
const f = () => console.log('now');
Promise.resolve().then(f)
console.log('next')
// Execution order：next now
```

#Async / Await
The async function is the syntactic sugar of Generator. Replaced * with async and yield with await.
## async features:
	1. async indicates that there is an asynchronous operation in the function, await indicates that the following expression needs to wait for the result
	2. The return value of the async function is a Promise object. we can use then method to add a callback function. At this time, the parameter of then is the value returned by the return statement inside the function.
	3. Once the executing function encounters `await`, it will pause first and wait until the asynchronous operation completes, then execute the following statement in the function body
	4. The Promise object returned by the async function will change state after all the Promise objects decorated by `await` are executed

## await features:
	1. If a Promise object is behind `await` key word, return the result of this object. If it is neither a Promise object nor a thenable object, return the corresponding valuedirectly.
	2. If the Promise object behind the `await` key word becomes rejected state, it will will interrupt async function execution and be caught by the catch of the async function.
	3. We can use Promise.all to make asynchronous requests execute parallelly at the same time

# Write a Promise that meets the Promises/A+ specification
[Promises/A+ specification](https://promisesaplus.com/)
```
const PENDING = 'pending';
const FULFILLED = 'fulfilled';
const REJECTED = 'rejected';
function Promise(f) {
    this.state = PENDING;
    // when state = fulfilled, result is value
    // when state = rejected, result is reason
    this.result = null;
    // then method can be called multiple times. The callbacks are used to record the onFulfilled and onRejected callbacks registered every time
    this.callbacks = [];
    let onFulfilled = value => transition(this, FULFILLED, value);
    let onRejected = reason => transition(this, REJECTED, reason);
    // use ignore to ensure resolve/reject is only called once
    let ignore = false;
    let resolve = value => {
        if (ignore) return ;
        ignore = true;
        resolvePromise(this, value, onFulfilled, onRejected)
    }
    let reject = reason => {
        if (ignore) return;
        ignore = true;
        onRejected(reason)
    }
    try {
        f(resolve, reject)
    } catch (error) {
        reject(error)
    }
}
// single Promise state transition function
const transition = (promise, state, result) => {
    if (promise.state !== PENDING) return;
    promise.state = state;
    promise.result = result;
    setTimeout(() => handleCallbacks(promise.callbacks, state, result), 0)
}
const handleCallbacks = function(callbacks, state, result) {
    while (callbacks.length) handleCallback(callbacks.shift(), state, result)
}
// passing states between current promise and next promise
const handleCallback = (callback, state, result) => {
    let {onFulfilled. onRejected, resolve, reject} = callback;
    try {
        if (state === FULFILLED) {
            isFunction(onFulfilled) ? resolve(onFulfilled(result)) : resolve(result)
        } else if (state === REJECTED) {
            isFunction(onRejected) ? resolve(onRejected(result)): reject(result)
        }
    } catch (error) {
        reject(error)
    }
}
// must contain then method，accepting onFulfilled and onRejected parameter，returning promise
Promise.prototype.then = function(onFulfilled, onRejected) {
    return new Promise((resolve, reject) => {
        let callback = {onFulfilled, onRejected, resolve, reject};
        if (this.state === PENDING) {
            this.callbacks.push(callback)
        } else {
            setTimeout(() => handleCallback(callback, this.state, this.result), 0)
        }
    })
}
// handling special result
const resolvePromise = (promise, result, resolve, reject) => {
    if (result === promise) {
        let reason = new TypeError('Can not fulfill promise with itself');
        return reject(reason)
    }
    if (isPromise(result)) {
        return result.then(resolve, reject)
    }
    if(isThenable(result)) {
        try {
            let then = result.then;
            if (isFunction(then)) {
                return new Promise(then.bind(result)).then(resilve, reject)
            }
        } catch (error) {
            return reject(error)
        }
    }
    resolve(result)
}
Promise.prototype.catch = function(onRejected) {
    return this.then(null, onRejected)
}
Promise.resolve = value => new Promise(resolve => resolve(value))
Promise.reject = reason => new Promise((_,reject) => reject(reason))
```
