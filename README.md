# Attempting to write Asynchronous code, Synchronously.
We're going to take a look a few examples of how to chain multiple asynchronys functions, there's many options, we'll start with the basics and explore some of the newer features available in es6/es7. 

The aim of this document is to teach you a few techniques on how to try and acheive a synchronous style of coding in an aysnchonous environment. Synchronous code is much easier to follow and debug, aysnc is generally better in terms of performance and flexibility, (do this, then do that, while doing this etc...).


Table of contents
=================

  * [Structured callback style](#structured-callback-style)
  * [Promises](#promises)
  * [Async function definitions](#using-async-function-definitions)
  * [Generators](#generators)

Let's start with the very basic product example where we need to retreive a price for a product, and then retreive a discount price from a coupon code, and return the final value for the product once ready.


Structured callback style
=========================
`*[much discomfort, max old, mehh]*`

This really is an old pattern and is very outdated, it's not sequential to read and it's easy to lose track when debugging. Using this pattern is not recommended as it creates a [Pyramid of doom](https://en.wikipedia.org/wiki/Pyramid_of_doom_(programming)) effect, which is horrid.

***Note:** For brevity, most actual logic and handling have been kept out of these examples.*
```js
let request = require('request'),
    product = null,
    price = null;
    
getProduct('xyz');
// get our product by id.
function getProduct(id) {
    request(`/product/${id}`, (err, res, body) => {
      product = body;
      getDiscount('code');
    });
}
// request the discount via code.
function getDiscount(code) {
    request(`/coupon/${code}`, (err, res, body) => {
      updatePrice(body);
    });
}
// updates price.
function updatePrice(discount = 0) {
    price = product.price = discount;
    // now we have a final price...
}
```
While the above pattern does work, we can do better...

Promises
=========================
`*[max excite, many happy, yay]*`

Promises are great, they solve the [Pyramid of doom](https://en.wikipedia.org/wiki/Pyramid_of_doom_(programming)) problem and create a more readable code structure.
This assumes you at least have a basic understanding of what promises are, if you don't head over to: [David Walsh's post on promises.][promises]
Promises are more of the norm when it comes to web patterns, so let's explore some of the possiblities by converting the above nested callback hell from above, into a promise pattern
```js
// start call chain, 
// (still horrible pyamid of doom, next example solves this)
getProduct('xyz')
    .then(product => {
        getDiscount('code')
        .then(discount => {
            updatePrice(product, discount);
        });
    });
  
// get our product by id.
function getProduct(id) {
    return new Promise(resolve => {
        request(`/product/${id}`, (err, res, body) => {
          resolve(body);
        });
    });
}
// request the discount via code.
function getDiscount(code) {
    return new Promise(resolve => {
        request(`/coupon/${code}`, (err, res, body) => {
          resolve(body);
        });
    });
}
// updates price.
function updatePrice(product = null, discount = 0) {
    price = product.price = discount;
    // now we have a final price...
}
```
While the above example is much more readable, we can take this further by making it even more syncronous.
Leaving the functions as is, we'll just update the call chain section.

```js
// start call chain
Promise.all([
        getProduct('xyz'),  
        getDiscount('code')
    ])
    .then(results => {
        const product = results[0],
            discount = results[1],
            price = updatePrice(product, discount);
    });
```
The above allows you resolve all the promises first, and then process the data syncronously.
There's a lot more patterns with promises which we've not explored here, we can setup an infrastructure where we can chain promises without nesting, I'm not going to show you how to set this up, but merely showing you that it's possible.

```js
getProduct('xyz')
    .then(getDiscount('code'))
    .then((data) => {
        const product = data.product,
            discount = data.discount,
            price = updatePrice(product, discount);
    });
```
This is much nicer, but we can do better, promises are very powerful, es6/es7 has given us even more power!


Using async function definitions
=========================
`*[as good as jelly babies]*`

Async-await is a godlike new feature, synchronous and asynchronous code can now become indistinguishable. Async-await is a new concept which will push developers into great design patterns, very readable code & easy debugging.
Async function definitions are basically a wrapper for [generators] and [promises], think of each await as an automatic promise binder, which will call the `next` method whenever the promise is resolved.

Now for the magic, again without actually changing our function definitions from above, let's start with how we'd create the call chain using async function definitions.

```js
// IIFE async function expression for the chain
(async function chain() {
    const product = await getProduct('xyz'),
        discount = await getDiscount('code'),
        price = product.price - discount;
})();
```
How awesome is that? The above does the exact same thing as the previous examples with promises or the straight callback style, except it is read and written as synchronous code!

The [`async`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/async_function) function declaration defines an asynchronous function, which returns an AsyncFunction object. When an `async` function is called, it will return a promise, whever that promise is resolved it will be resolved with the value from the resolved await promise.

The [`await`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await) operator is used to wait for a promise to be resolved and it will return the value of that resolved promise, if not a promise it will just return the value. This await expression will cause the `async` function execution to pause, wait and the continue after it's resolved.

This also allows us to easily handle errors, and also success methods, let's say one of the promises is rejected because something went wrong. This is a complete working example, you can copy this code into the chrome console which supports these new features when I wrote this document.

***Note** If you pass `true` to `const product = new Product(true)`; it will show how a rejection is handled.*

```js
class Product {
    constructor(reject) {
        this.reject = reject;
    }
    async getPrice(code) {
		this.product = await this.getProduct(this.id);
        this.discount = await this.getDiscount(code);
        this.price = (this.product.price - this.discount.value) || 0;
        return this;
    }
    // timeout simulates request to server
    getDiscount(x) {
      return new Promise((resolve, reject) => {
        setTimeout(() => {
          const data = {value : 20, code: x, msg : `The code '${x}' is either invalid or does not exist.`};
          this.reject ? reject(data) : resolve(data);
        }, 2000);
      });
    }
    // timeout simulates request to server
    getProduct(x) {
      return new Promise(resolve => {
        setTimeout(() => {
          resolve({price: 40, id : x});
        }, 2000);
      });
    }
}
// grab product api instance, pass true to the constructor 
// to demonstate a rejected promise.
const product = new Product();
product.getPrice('xyz', 'oblong').then(data => {
    console.log(`Product price is: $${data.price}, price was: $${data.product.price}, you saved: $${data.product.price - data.price}! `);
    // will output: Product price is: $20, price was: $40, you saved: $20! 
}, reason => {
    // will output the reason why the promise was rejected, this should be handled yourself.
    console.error(`Couldn't calculate price because: ${reason.msg}`);
    // will output The code 'oblong' is either invalid or does not exist.
});

```

I employ you to play around with these, very handy, I use these when writing asynchronous node tasks within architecture structures, similar to this:
```js
class Chain {
	constructor() {
		
	}

	async start() {
		const pre = await this.process('pre');
	    const scss = await this.process('scss');
	    const js = await this.process('js');
	    const html = await this.process('html');
	    const assets = await this.process('assets');
	    const deploy = await this.process('deploy');
	    const post = await this.process('post');
	    return true;
	}

	// simulate async methods in syncronous pattern.
	process(x) {
	  return new Promise(resolve => {
	    setTimeout(() => {
	      resolve(x);
	    }, 200);
	  });
	}
}
module.exports = Chain;
```





```js
// called from another file
const Chain = require('./_chain.js');
let chain = new Chain();
chain.start().then(() => {
	console.log('dun everytink.');
});
```

Generators
=========================
`*[Much amaze, super wow]*`

While `async` is the perferred pattern for me, there's still more options which give you a similar pattern, this more or less is the same
functionality as Async function definitions, the logic is very similar, just how it's processed is different.

[Generators][generators] allow us to break down the process and do something with each value returned, when it's returned, this is pretty powerful!. An example would be to use the Class above, but convert it to generator syntax and access the values from each promise, outside the main class Chain.

```js
class Chain {
	constructor(...args) {
		this.args = args;
	}

	start(cb) {
		this.cb = cb;
		this.chain = this.asyncChain(this.generator(this));
	}
    // the logic to recurssively and asyncronously process the next method in a generator.
	asyncChain(it, context = undefined) {
		let generator = typeof it === 'function' ? it() : it // Create generator if necessary			
		let { value: promise, done : success } = generator.next();
		if (context) this.cb(context, success);
		if (success) return;
		if ( promise instanceof Promise ) {
			promise.then(resolved => this.asyncChain(generator, resolved))
			.catch(error => generator.throw(error)) // Defer to generator error handling
		}
	}
    // * denotes this function as a generator, yield is similar to await, 
    // although it is only ever 'resolved' when generator.next() is called, it's not relying
    // on the actual promise being resolved.
    
	* generator(self) {
		yield self.process('pre');
	    yield self.process('scss');
	    yield self.process('js');
	    yield self.process('html');
	    yield self.process('assets');
	    yield self.process('deploy');
	    yield self.process('post');
	    return true;
	}

	// simulate async methods in syncronous pattern.
	process(x) {
	  return new Promise(resolve => {
	    setTimeout(() => {
	      resolve(x);
	    }, 200);
	  });
	}
}
```

Now when we request the file, and start the chain, we'll have access to each value resolved from each promise, as they're resolved. 

```js
// Grab chain class from above.
const Chain = require('./_chain.js');
// call chain start method, which accepts a function which will be called
// whenever a promise is resolved, we could have
new Chain().start((processValue, isFinished) => {
    console.log(processValue);
    if (isFinished) console.log('all finished');
});
// will output a new line every 200ms.
// pre
// scss
// js
// html
// assets
// deploy
// post
// all finished
```

There's a million ways to do this, we can replicate the same result above with `async` too, which would be much cleaner, this just demonstrates what's possible with generators when writing async code syncronously.

Any feedback is always welcome, feel free to contact me any time at [www.shannonhochkins.com]('http://www.shannonhochkins.com')!

   [generators]: <https://developer.mozilla.org/en/docs/Web/JavaScript/Guide/Iterators_and_Generators>
   [promises]: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise>
   
