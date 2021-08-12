---
title: "A Simple Guide to Asynchronous JavaScript: Callbacks, Promises & async/await"
date: 2021-08-12T15:28:02+01:00
draft: false
author: "Mangabo Kolawole"
featured_image: "asynchronous.png"
---

Asynchronous programming in JavaScript is one of the fundamental concepts to grasp to write better JavaScript. 

Today, we'll learn about asynchronous JavaScript, with some real-world examples and some practical examples as well. Along with this article, you'll understand the functioning of:
- Asynchronous Callbacks
- Promises
- Async / Await

## Table of content

- 1 - [Synchronous Programming vs Asynchronous Programming](#1-synchronous-vs-asynchronous)

- 2 -[ Asynchronous Callbacks: I'll call back once I'm done!](#2-asynchronous-callbacks-ill-call-back-once-im-done)

- 3 - [ Promises in JavaScript: I promise a result!](#3-promise-i-promise-a-result)

- 4 - [ Async/Await: I'll execute later! ](#4-asyncawait-ill-execute-when-i-am-ready)

## 1 - Synchronous vs Asynchronous

Before going into asynchronous programming, let's talk about **synchronous programming** first. 

For example, 

```javascript
let greetings = "Hello World.";

let sum = 1 + 10;

console.log(greetings);

console.log("Greetings came first.")

console.log(sum);
```

You'll have an output in this order. 

```javascript
Hello World.
Greetings came first.
11
```
That's *synchronous*. Notice that while each operation happens, nothing else can happen. 

> JavaScript is monothread at its core: when one block of code is being executed, no other block of code will be executed. 

Asynchronous programming is different. To make it simple, when JavaScript identifies asynchronous tasks, it'll simply continue the execution of the code, while waiting for these asynchronous tasks to be completed.

Asynchronous programming is often related to parallelization, the art of performing independent tasks in parallel.

How is it even possible?

Trust me, we do things in an asynchronous way without even realizing it. 

Let's take a real-life example to better understand.  

### Real life example: Coffee shop

*Jack* goes to the coffee shop and goes directly to the first attendant. (Main thread)

- *Jack*: Hi. Please can I have a coffee?  (First asynchronous task)
- *First Attendant*: For sure. Do you want something else? 
- *Jack*: A piece of cake while waiting for the coffee to be ready. (Second asynchronous task)
- *First Attendant*: For sure. ( Launch the preparation of the coffee )
- *First Attendant*: Anything else?
- *Jack*: No. 
- *First Attendant*: 5 dollars, please. 
- *Jack*: Pay the money and take a seat.
- *First Attendant*: Start serving the next customer.
- *Jack*: Start checking Twitter while waiting.
- *Second Attendant*: Here's your cake. (Second asynchronous task call returns)
- *Jack*: Thanks
- *First attendant*: Here's your coffee. (First asynchronous task call returns)
- *Jack*: Hey, thanks! Take his stuff and leave. 

> To make it simple, while you are waiting for someone else to do stuff for you, you can do other tasks or ask others to do more stuff for you. 

Now that you have a clear idea about how asynchronous programming works, let's see how we can write asynchronous with : 
- Asynchronous callbacks
- Promises 
- And the `async/await` syntax.

## 2 - Asynchronous Callbacks: I'll call back once I'm done!

A ***callback*** is a function passed as an argument when calling a function (*high-order function*) that will start executing a task in the background. 
And when this background task is done running, it calls the callback function to let you know about the changes.

```javascript
function callBackTech(callback, tech) {
    console.log("Calling callBackTech!");
    if (callback) {
        callback(tech);
    }
    console.log("Calling callBackTech finished!");
}

function logTechDetails(tech) {
    if (tech) {
        console.log("The technology used is: " + tech);
    }
}

callBackTech(logTechDetails, "HTML5");
```

**Output**

![Output Callback](https://cdn.hashnode.com/res/hashnode/image/upload/v1628603764046/WG7ZO0Sg2j.png)

As you can see here, the code is executed each line after each line: this is an example of **synchronously** executing a callback function.

And if you code regularly in JavaScript, you may have been using callbacks without even realizing it. For example : 
- `array.map(callback)`
- `array.forEach(callback)`
- `array.filter(callback)`

```javascript
let fruits = ['orange', 'lemon', 'banana']

fruits.forEach(function logFruit(fruit){
    console.log(fruit);
});
```

**Output**
```javascript
orange
lemon
banana
```

But callbacks can also be executed **asynchronously**, which simply means that the callback is executed at a later time than the higher-order function. 

Let's rewrite our example using `setTimeout()` function to register a callback to be called asynchronously. 

```javascript
function callBackTech(callback, tech) {
    console.log("Calling callBackTech!");
    if (callback) {
        setTimeout(() => callback(tech), 2000)
    }
    console.log("Calling callBackTech finished!");
}

function logTechDetails(tech) {
    if (tech) {
        console.log("The technology used is: " + tech);
    }
}

callBackTech(logTechDetails, "HTML5");
```

**Output**

![Output Async callbacks](https://cdn.hashnode.com/res/hashnode/image/upload/v1628604316666/YS1IKMEbM.png)

In this asynchronous version, notice that the output of `logTechDetails()` is printed in the last position.

This is because the asynchronous execution of this callback delayed its execution from 2 seconds to the point where the currently executing task is done.

Callbacks are `old-fashioned` ways of writing asynchronous JavaScript because as soon you have to handle multiple asynchronous operations, the callbacks nest into each other ending in `callback hell`.


![Callback Hell](https://cdn.hashnode.com/res/hashnode/image/upload/v1628623467745/L02qUtv0A.jpeg)

 To avoid this pattern happening, we will see now `Promises`.


## 3 - Promise: I promise a result!

*Promises* are used to handle asynchronous operations in JavaScript and they simply represent the fulfillment or the failure of an asynchronous operation. 

Thus, Promises have four states : 
- **pending**: the initial state of the promise
- **fulfilled**: the operation is a success
- **settled**: the operation is a failure
- **rejected**: the operation is either fulfilled or settled, but not pending anymore.

This is the general syntax to create a Promise in JavaScript. 

```javascript
let promise = new Promise(function(resolve, reject) {
    ... code
});
```

`resolve` and `reject` are functions executed respectively when the operation is a success and when the operation is a failure.

To better understand how `Promises` work, let's take an example.

- *Jack's Mom*: Hey Jack! Can you go to the store and get some milk? I need more to finish the cake. 
- *Jack*: For sure, Mom!
- *Jack's Mom*: While you are doing that, I'll be dressing the tools to make the cake. (Async task) In the meantime, let me know if find it. (success callback)
- *Jack*: Great! But what if I don't find the milk? 
- *Jack's Mom*: Then, get some chocolate instead. (Failure callback)

This analogy isn't terribly accurate, but let's go with it. 

Here's what the promise will look like, supposing that Jack has found some milk. 

```javascript
let milkPromise = new Promise(function (resolve, reject) {

    let milkIsFound = true;

    if (milkIsFound) {
        resolve("Milk is found");
    } else {
        reject("Milk is not found");
    }
});
```
Then, this promise can be used like this: 

```javascript
milkPromise.then(result => {
    console.log(result);
}).catch(error => {
    console.log(error);
}).finally(() => {
    console.log("Promised settled.");
});
```
Here : 
- `then()`: takes a callback for success case and executes when the promise is resolved. 
- `catch()`: takes a callback, for failure and executes if the promise is rejected.
- `finally()`: takes a callback and always returns when the premise is settled. It's pretty useful when you want to perform some cleanups. 

Let's use a real-world example now, by creating a promise to fetch some data.

```javascript
let retrieveData = url => {

    return new Promise( function(resolve, reject) {

        let request = new XMLHttpRequest();
        request.open('GET', url);
    
        request.onload = function() {
          if (request.status === 200) {
            resolve(request.response);
          } else {
            reject("An error occured!");
          }
        };
        request.send();
    })
};
```

The `XMLHttpRequest` object can be used to make HTTP request in JavaScript.

Let's use the `retrieveData` to make a request from https://swapi.dev, the Star Wars API.

```javascript
const apiURL = "https://swapi.dev/api/people/1";

retrieveData(apiURL)
.then( res => console.log(res))
.catch( err => console.log(err))
.finally(() => console.log("Done."))
```
Here's what's the output will look like. 

**Output**

![Output](https://cdn.hashnode.com/res/hashnode/image/upload/v1628602217307/Z_SQ2NFqOV.png)

#### Rules for writing promises

- You can't call both `resolve` or `reject` in your code. As soon as one of the two functions gets called, the promise stops and a result is returned.
- If you don’t call any of the two functions, the promise will hang.
- You can only pass one parameter to `resolve` or `reject`. If you have more things to pass, wrap everything in an object.

## 4 - async/await: I'll execute when I am ready!

The `async/await` syntax has been introduced with **[ES2017](https://flaviocopes.com/es2017/)**, to help write better asynchronous code with promises.

Then, what's wrong with promises?
The fact that you can chain `then()` as many as you want makes `Promises` a little bit verbose. 
For the example of Jack buying some milk, he can : 
- call his Mom;
- then buy more milk;
- then buy chocolates;
- and the list goes on. 

```javascript
milkPromise.then(result => {
        console.log(result);
    }).then(result => {
        console.log("Calling his Mom")
    }).then(result => {
        console.log("Buying some chocolate")
    }).then(() => {
        ...
    })
    .catch(error => {
        console.log(error);
    }).finally(() => {
        console.log("Promised settled.");
    });
``` 
Let's see how we can use `async/await` to write better asynchronous code in JavaScript.

### The friend party example

Jack is invited by his friends to a party. 
- *Friends*: When are you ready? We'll pick you. 
- *Jack*: In 20 minutes. I promise.

Well, actually Jack will be ready in 30 minutes. And by the way, his friends can't go to the party without him, so they'll have to wait. 

In a synchronous way, things will look like this. 

```javascript
let ready = () => {

    return new Promise(resolve => {

        setTimeout(() => resolve("I am ready."), 3000);
    })
}
```
The `setTimeout()` method takes a function as an argument (a callback) and calls it after a specified number of milliseconds. 

Let's use this `Promise` in a regular function and see the output. 

```javascript

function pickJack() {

    const jackStatus = ready();
    
    console.log(`Jack has been picked: ${jackStatus}`);

    return jackStatus;
}

pickJack(); // => Jack has been picked: [object Promise]
```

Why this result? The `Promise` hasn't been well handled by the function `pickJack`. 
It considers `jackStatus` like a regular object. 

It's time now to tell our function how to handle this using the `async` and `await` keywords.

First of all, place `async` keyword in front of the function `pickJack()`. 

```javascript
async function pickJack() {
    ...
}
```
By using the `async` keyword used before a function, JavaScript understands that this function will return a `Promise`. 
Even, if we don't explicitly return a `Promise`, JavaScript will automatically wrap the returned object in a Promise. 

And next step, add the `await` keyword in the body of the function.

```javascript
    ...
    const jackStatus = await ready();
    ...
```
`await` makes JavaScript wait until the `Promise` is **settled** and returns a result. 

Here's how the function will finally look.

```javascript
async function pickJack() {

    const jackStatus = await ready();
    
    console.log(`Jack has been picked: ${jackStatus}`);

    return jackStatus;
}

pickJack(); // => "Jack has been picked: I am ready."
```
And that's it for `async/await`. 

This syntax has simple rules: 

- If the function you are creating handles async tasks, mark this function using the `async` keyword.

- `await` keyword pauses the function execution until the promise is settled (fulfilled or rejected).

- An asynchronous function always returns a `Promise`.

Here's a practical example using `async/await` and the `fetch()` method. `fetch()` allows you to make network requests similar to `XMLHttpRequest` but the big difference here is that the Fetch API uses Promises.

This will help us make the data fetching from https://swapi.dev cleaner and simple.

```javascript
async function retrieveData(url) {
    const response = await fetch(url);

    if (!response.ok) {
        throw new Error('Error while fetching resources.');
    }

    const data = await response.json()

    return data;
};
 ```
`const response = await fetch(url);` will pause the function execution until the request is completed.
Now why `await response.json()`? You may be asking yourself. 

After an initial `fetch()` call, only the headers have been read. And as the body data is to be read from an incoming stream first before being parsed as JSON. 

And since reading from a TCP stream (making a request) is asynchronous, the `.json()` operations end up asynchronous.

Then let's execute the code in the browser. 

```javascript
retrieveData(apiURL)
.then( res => console.log(res))
.catch( err => console.log(err))
.finally(() => console.log("Done."));
```
And that's all for `async/await`

### Conclusion

In this article, we learned about callbacks, `async/await` and `Promise` in JavaScript to write asynchronous code. If you want to learn more about these concepts, check these amazing resources.
-  [An Interesting Explanation of async/await in JavaScript](https://dmitripavlutin.com/javascript-async-await) 
-  [Everything About Callback Functions in JavaScript](https://dmitripavlutin.com/javascript-callback/) 
-  [Promises basics](https://javascript.info/promise-basics) 
And as every article can be made better so your suggestion or questions are welcome in the comment section. 😉
